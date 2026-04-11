# [005] RLS에 복잡한 비즈니스 로직을 넣으려는 시도

> RLS는 "최후의 보안 저지선"으로만 사용해야 한다. 복잡한 권한 로직을 RLS로 처리하려 하면 테스트 불가, 유지보수 불가 상태가 된다.

- **카테고리**: `01-rls`
- **심각도**: 🟠 High
- **관련 파일**: `supabase/migrations/*.sql`

---

## 문제 상황

바이브코딩으로 복잡한 권한 로직을 RLS에 넣으려 하면 이런 상황이 된다.

```sql
-- "이 주문은 주문자이거나, 주문자의 팀장이거나, 해당 지역 관리자이면 볼 수 있음"
CREATE POLICY "orders_complex_access" ON orders
  FOR SELECT USING (
    auth.uid() = user_id
    OR EXISTS (
      SELECT 1 FROM team_members
      WHERE manager_id = auth.uid()
        AND member_id = orders.user_id
    )
    OR EXISTS (
      SELECT 1 FROM regional_admins
      WHERE admin_id = auth.uid()
        AND region = orders.region
    )
  );
```

복잡한 권한 검사를 RLS에 결합하면 이후 곤란한 상황이 생기기 쉽다. 엔드포인트 단위로 미들웨어를 쓰면 처리가 깔끔해지는데, RLS로는 그렇게 하기 어렵다.

## 바이브코딩에서 이 문제가 발생하는 이유

1. AI가 권한 요구사항을 받으면 SQL 로직으로 구현하려 함
2. RLS가 "DB 레벨 보안"이라는 것을 알고 있어서 거기에 모든 권한을 집어넣음
3. 처음엔 동작하지만, 로직이 복잡해질수록 테스트가 불가능해짐
4. 비즈니스 로직이 SQL 안에 숨어들어 디버깅 불가

## 올바른 역할 분리

권장 패턴은 복잡한 권한 검증 로직을 서버에서 수행하고, admin client로 RLS를 우회하는 것이다. 복잡한 비즈니스 로직은 서버에 SSOT로 두어 단위 테스트가 가능하게 유지한다. RLS는 서버를 거치지 않고 들어오는 공격에 대한 최후의 저지선으로만 사용한다.

```
[클라이언트]
    ↓
[서버 (비즈니스 로직 + 권한 검증)]  ← 복잡한 RBAC는 여기서
    ↓ (service_role 또는 검증된 user context)
[Supabase DB]
    ↓ RLS (최후의 저지선: ownerId = userId 수준)
[실제 데이터]

[클라이언트] → [Supabase 직접 접근]
                    ↓ RLS (단순 소유권 체크만)
               [실제 데이터]
```

### 케이스 1: 백엔드 서버가 있는 경우

```typescript
// server/routes/orders.ts
app.get('/orders/:id', authenticate, async (req, res) => {
  const order = await db.from('orders').select('*').eq('id', req.params.id).single()

  // 복잡한 권한 검증은 서버에서
  const canAccess = await checkOrderAccess(req.user.id, order.data)
  if (!canAccess) return res.status(403).json({ error: 'Forbidden' })

  res.json(order.data)
})

// 권한 로직: 테스트 가능한 일반 함수
async function checkOrderAccess(userId: string, order: Order): Promise<boolean> {
  if (order.user_id === userId) return true

  const isTeamManager = await isManagerOf(userId, order.user_id)
  if (isTeamManager) return true

  const isRegionalAdmin = await isAdminForRegion(userId, order.region)
  return isRegionalAdmin
}
```

```sql
-- RLS는 이것만 (최후의 저지선)
CREATE POLICY "orders_basic_protection" ON orders
  FOR SELECT USING (auth.uid() = user_id);
-- 서버는 service_role로 우회해서 위 정책을 신경 안 써도 됨
```

### 케이스 2: 클라이언트 직접 접근 (서버 없음)

RLS만 쓴다면 복잡한 로직은 **포기**하거나 **단순화**해야 한다.

```sql
-- 단순하게 유지: 소유권 + admin 역할만
CREATE POLICY "orders_select" ON orders
  FOR SELECT USING (
    auth.uid() = user_id
    OR auth.jwt() ->> 'role' = 'admin'
  );
```

복잡한 팀장/지역관리자 권한이 필요하다면 → **서버 레이어를 추가**하는 것이 맞다.

RLS의 목적은 DB 레벨에서 서버를 거치지 않는 직접 요청을 방어하는 것이다. VPN, IP block과는 다른 역할로, 하나의 보안 레이어를 덧대는 정도로 사용한다.

## RLS에 적합한 로직 vs 아닌 로직

| 적합 | 부적합 |
|------|--------|
| `auth.uid() = user_id` (소유권) | 여러 테이블 JOIN이 필요한 권한 |
| `auth.jwt() ->> 'role' = 'admin'` (JWT role) | 시간/이벤트 기반 권한 (구독 만료 등) |
| `is_published = true` (공개 여부) | 복잡한 계층적 팀 구조 권한 |
| 단순 FK 관계 기반 접근 | AOP 방식 접근 제어 |

## 예방법

- [ ] RLS 정책이 5줄 넘어가면 서버로 옮기는 것을 검토
- [ ] `EXISTS (SELECT ... FROM other_table ...)` 가 2개 이상이면 서버로 이동
- [ ] RLS 정책은 "이 row의 소유자인가?" 수준으로만 유지
- [ ] 복잡한 권한 로직은 서버의 `checkAccess()` 함수로 작성 → 단위 테스트 가능

## 참고 자료

- [Supabase - 서버사이드 렌더링 패턴](https://supabase.com/docs/guides/auth/server-side/nextjs)
