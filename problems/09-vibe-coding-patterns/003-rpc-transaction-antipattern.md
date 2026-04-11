# [010] 트랜잭션에 RPC(DB 함수) 남용 → 유지보수 불가

> 트랜잭션 처리를 위해 Supabase RPC(PostgreSQL 함수)에 의존하면, 로직이 DB에 묻혀 테스트·수정이 불가능해진다.

- **카테고리**: `09-vibe-coding-patterns`
- **심각도**: 🟠 High
- **관련 파일**: `supabase/migrations/*.sql`, 백엔드 코드

---

## 문제 상황

트랜잭션이 필요할 때 RPC를 쓰면 로직이 파일이 아닌 DB에 저장되어 유지보수가 힘들다. 중요한 애플리케이션이라면 전통적인 Server ↔ DB client 구조가 권장된다.

Supabase 공식 문서와 AI가 트랜잭션이 필요할 때 RPC를 권장한다:

```typescript
// AI가 생성하는 패턴
const { data, error } = await supabase.rpc('process_order', {
  p_order_id: orderId,
  p_user_id: userId
})
```

```sql
-- DB에 저장된 함수 (migration 파일에는 있지만 실질적 로직은 DB 안에)
CREATE OR REPLACE FUNCTION process_order(p_order_id UUID, p_user_id UUID)
RETURNS JSON AS $$
BEGIN
  -- 재고 확인
  -- 주문 생성
  -- 결제 처리
  -- 알림 발송
  -- 포인트 적립
  RETURN json_build_object('success', true);
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION '%', SQLERRM;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## 왜 문제인가

1. **로직이 DB 안에 묻힘**: IDE에서 일반 코드처럼 수정 불가
2. **테스트 어려움**: 일반 유닛 테스트 프레임워크로 테스트 불가
3. **디버깅 지옥**: `RAISE NOTICE`로만 디버깅, 스택 트레이스 없음
4. **비즈니스 로직 파편화**: 일부는 TypeScript, 일부는 PL/pgSQL
5. **AI가 수정하기 어려움**: 바이브코딩 세션에서 DB 함수 컨텍스트가 없으면 AI가 현재 상태를 모름

## 해결 방법

### 권장: 서버에서 트랜잭션 직접 처리

```typescript
// Next.js API Route or Express
// app/api/orders/process/route.ts

import { createClient } from '@supabase/supabase-js'

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // 서버에서만 사용
)

export async function POST(req: Request) {
  const { orderId, userId } = await req.json()

  // PostgreSQL 트랜잭션을 서버에서 직접 실행
  const { data, error } = await supabaseAdmin.rpc('begin_transaction')
  
  try {
    // 재고 확인
    const { data: stock } = await supabaseAdmin
      .from('inventory')
      .select('quantity')
      .eq('product_id', orderId)
      .single()

    if (!stock || stock.quantity < 1) {
      throw new Error('재고 부족')
    }

    // 주문 생성
    const { data: order } = await supabaseAdmin
      .from('orders')
      .insert({ user_id: userId, status: 'pending' })
      .select()
      .single()

    // 재고 차감
    await supabaseAdmin
      .from('inventory')
      .update({ quantity: stock.quantity - 1 })
      .eq('product_id', orderId)

    return Response.json({ success: true, order })
  } catch (error) {
    // 롤백 필요 시 처리
    return Response.json({ error: error.message }, { status: 400 })
  }
}
```

### Supabase에서 진짜 트랜잭션이 필요하다면: pgTransaction

```typescript
// Supabase JS SDK v2에서 트랜잭션 지원 방법
// BEGIN/COMMIT을 RPC로 감싸는 패턴

// 또는 Postgres.js 직접 연결 (서버사이드)
import postgres from 'postgres'

const sql = postgres(process.env.DATABASE_URL!)

async function processOrder(orderId: string, userId: string) {
  return await sql.begin(async (sql) => {
    const [stock] = await sql`
      SELECT quantity FROM inventory WHERE product_id = ${orderId} FOR UPDATE
    `
    
    if (stock.quantity < 1) throw new Error('재고 부족')
    
    const [order] = await sql`
      INSERT INTO orders (user_id, status) VALUES (${userId}, 'pending')
      RETURNING *
    `
    
    await sql`
      UPDATE inventory SET quantity = quantity - 1 WHERE product_id = ${orderId}
    `
    
    return order
  })
}
```

### RPC가 적합한 경우 (좁게 사용)

| 적합 | 부적합 |
|------|--------|
| 단순 집계 쿼리 (`COUNT`, `SUM` 등) | 복잡한 비즈니스 로직 |
| 데이터 변환 (배열 정렬 등) | 외부 서비스 호출 포함 로직 |
| 성능 최적화가 필요한 단일 쿼리 | 다단계 트랜잭션 |
| PostgreSQL 특화 기능 활용 | 조건 분기가 많은 로직 |

## 테스트 비교

```typescript
// RPC 방식: 테스트 어려움
test('주문 처리', async () => {
  // DB에 실제 함수가 있어야 하고
  // 테스트 DB 상태를 직접 세팅해야 하고
  // 결과를 DB에서 다시 조회해야 함
  const { data } = await supabase.rpc('process_order', { ... })
  // 중간 과정 검증 불가
})

// 서버 함수 방식: 테스트 쉬움
test('주문 처리', async () => {
  // 의존성 모킹 가능
  const mockInventory = jest.fn().mockResolvedValue({ quantity: 5 })
  const result = await processOrder(orderId, userId, { getInventory: mockInventory })
  expect(result.status).toBe('pending')
  expect(mockInventory).toHaveBeenCalledWith(orderId)
})
```

## 예방법

- [ ] 비즈니스 로직이 포함된 DB 함수 작성 전에 "서버에서 처리 가능한가?" 먼저 확인
- [ ] RPC는 "단순 쿼리 최적화"용으로만 제한
- [ ] 복잡한 로직은 TypeScript로 작성 → 테스트 가능하게
- [ ] 트랜잭션이 필요하면 `postgres.js` + `sql.begin()` 사용

## 참고 자료

- [postgres.js 트랜잭션](https://github.com/porsager/postgres#transactions)
- [Supabase RPC 가이드](https://supabase.com/docs/guides/database/functions)
