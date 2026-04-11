# [002] 바이브코딩 AI가 만드는 RLS 안티패턴

> AI가 생성하는 RLS 정책은 그럴듯해 보이지만 실제로는 보안 구멍이 있는 패턴들이 있다.

- **카테고리**: `01-rls`
- **심각도**: 🔴 Critical
- **관련 파일**: `supabase/migrations/*.sql`

---

## 문제 상황

백엔드 지식 없이 바이브코딩으로 RLS를 구현하면, AI가 생성한 정책이 "있는 것처럼 보이지만 구멍이 있는" 상태가 된다.

구체적으로 AI가 자주 만드는 잘못된 패턴들:

## 안티패턴 1: 서버 레이어 WHERE절로 RLS를 대체

AI는 PostgreSQL RLS 대신 애플리케이션 레이어에서 필터링하는 코드를 자주 생성한다.

```typescript
// AI가 생성한 코드 (잘못됨)
async function getUserPosts(userId: string) {
  const { data } = await supabase
    .from('posts')
    .select('*')
    .eq('user_id', userId)  // 이걸로 보안을 처리하려 함
  return data
}
```

이 코드는:
- `userId`를 클라이언트에서 조작 가능
- 다른 API 경로에서 같은 테이블을 쓸 때 WHERE 절 빠뜨리면 전체 노출
- RLS가 없으므로 직접 API 호출로 우회 가능

**올바른 방법**:
```sql
-- DB 레벨에서 강제하기
CREATE POLICY "posts_select_own" ON posts
  FOR SELECT
  USING (auth.uid() = user_id);
```

```typescript
// 그러면 이렇게 단순하게 써도 안전
async function getUserPosts() {
  const { data } = await supabase.from('posts').select('*')
  // RLS가 자동으로 현재 유저 것만 반환
  return data
}
```

## 안티패턴 2: 인증 미들웨어만 믿고 RLS 생략

AI가 Express/NestJS 패턴으로 생각할 때 발생한다.

```typescript
// AI가 생성한 서버 코드 (잘못됨)
app.get('/posts', authenticate, async (req, res) => {
  // authenticate 미들웨어가 통과했으니 괜찮다고 생각
  const { data } = await supabaseAdmin.from('posts').select('*')
  // supabaseAdmin은 service_role → RLS 우회
  // 모든 유저의 posts가 반환됨
  res.json(data)
})
```

**올바른 방법**:
```typescript
app.get('/posts', authenticate, async (req, res) => {
  // 사용자의 JWT로 Supabase 클라이언트 생성
  const userSupabase = createClient(url, anonKey, {
    global: { headers: { Authorization: `Bearer ${req.user.token}` } }
  })
  // 이제 RLS가 적용됨
  const { data } = await userSupabase.from('posts').select('*')
  res.json(data)
})
```

## 안티패턴 3: 불완전한 CRUD 정책

AI는 SELECT 정책만 만들고 INSERT/UPDATE/DELETE를 빠뜨리는 경우가 많다.

```sql
-- AI가 생성한 정책 (불완전)
CREATE POLICY "posts_select" ON posts
  FOR SELECT USING (auth.uid() = user_id);

-- INSERT, UPDATE, DELETE 정책 없음
-- → 아무도 글을 못 씀 (INSERT 막힘)
-- → 또는 누구나 글을 쓸 수 있음 (정책 해석 방식에 따라)
```

**올바른 방법**:
```sql
-- 완전한 CRUD 정책 세트
CREATE POLICY "posts_select" ON posts
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "posts_insert" ON posts
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "posts_update" ON posts
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "posts_delete" ON posts
  FOR DELETE USING (auth.uid() = user_id);
```

## 안티패턴 4: RBAC를 RLS로 구현할 때 role 컬럼 신뢰

```sql
-- 잘못된 패턴: users 테이블의 role 컬럼을 직접 조회
CREATE POLICY "admin_all" ON posts
  FOR ALL USING (
    (SELECT role FROM users WHERE id = auth.uid()) = 'admin'
  );
```

이 패턴의 문제:
- N+1 쿼리 (매 row마다 users 테이블 조회)
- users 테이블 RLS가 없으면 보안 문제

**올바른 방법**:
```sql
-- JWT claim에 role 넣기 (Supabase Auth Hook 활용)
CREATE POLICY "admin_all" ON posts
  FOR ALL USING (
    auth.jwt() ->> 'role' = 'admin'
  );
```

또는 별도 함수로 분리:
```sql
CREATE OR REPLACE FUNCTION is_admin()
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_id = auth.uid() AND role = 'admin'
  )
$$ LANGUAGE sql SECURITY DEFINER;

CREATE POLICY "admin_all" ON posts
  FOR ALL USING (is_admin());
```

## 검증 방법

다른 유저로 로그인해서 타인 데이터 접근 시도:

```typescript
// test/rls.test.ts
describe('RLS 검증', () => {
  it('다른 유저의 post에 접근 불가', async () => {
    // user2로 로그인
    const { data: { session } } = await supabase.auth.signInWithPassword({
      email: 'user2@test.com',
      password: 'test1234'
    })

    // user1이 만든 post 조회 시도
    const { data, error } = await supabase
      .from('posts')
      .select('*')
      .eq('user_id', 'user1-uuid')

    expect(data).toHaveLength(0)  // 빈 배열이어야 함
  })

  it('비로그인 상태에서 post 접근 불가', async () => {
    await supabase.auth.signOut()

    const { data } = await supabase.from('posts').select('*')
    expect(data).toHaveLength(0)
  })
})
```

## 예방법

- [ ] AI에게 RLS 정책 생성 요청 시 "SELECT, INSERT, UPDATE, DELETE 모두 포함해서" 명시
- [ ] 생성된 RLS 정책은 반드시 다른 역할(anon, authenticated, 다른 유저)로 테스트
- [ ] `service_role`로 테스트하는 습관 제거 — 항상 `anon` 또는 실제 유저 JWT 사용
- [ ] RBAC 구현 전 Supabase Custom Claims 문서 먼저 읽기

## 참고 자료

- [Supabase RLS 정책 예시](https://supabase.com/docs/guides/database/postgres/row-level-security#examples)
- [Custom Claims & RBAC](https://supabase.com/docs/guides/auth/custom-claims-and-role-based-access-control-rbac)
