# [008] RLS 직접 연결 구조로 인한 DB 마이그레이션 락인

> 클라이언트 ↔ Supabase 직접 연결 구조로 RLS에 의존하면, 나중에 다른 DB로 옮기거나 백엔드를 추가할 때 엄청나게 고통받는다.

- **카테고리**: `01-rls`
- **심각도**: 🟠 High
- **관련 파일**: 클라이언트 코드 전반

---

## 문제 상황

나중에 다른 DB로 마이그레이션 계획이 있다면, RLS로 클라이언트 ↔ Supabase 직접 연결하는 구조는 나중에 엄청난 고통을 준다. wrapping 서버를 띄우고 service role로 권한 체크하는 전통적인 백엔드 방식으로 구성하면 추후 마이그레이션이 가능하다.

MVP 단계에서 Supabase 직접 연결로 빠르게 만들었다가, 이후 다음 상황에서 막힌다:
- PostgreSQL → MySQL/MongoDB 이전 검토
- Supabase 비용 증가로 자체 DB 서버로 이전 고려
- 백엔드 서버 추가 (비즈니스 로직 서버화)
- 다른 서비스 제공자로 변경

## 왜 이 구조가 문제인가

클라이언트에서 Supabase SDK를 직접 쓰면:

```typescript
// 클라이언트 코드 전반에 이런 패턴이 박힘
const { data } = await supabase.from('orders').select('*, items(*)')
const { error } = await supabase.from('posts').insert({ title, user_id: user.id })
await supabase.storage.from('avatars').upload(path, file)
```

이 코드는 Supabase SDK에 **강하게 결합**되어 있다:
- Supabase PostgREST 쿼리 문법
- Supabase Storage API
- Supabase Auth 방식 (`auth.uid()`, JWT 처리)
- RLS 정책이 비즈니스 로직을 담당

다른 DB로 이전하려면 **클라이언트 코드 전체**를 다시 써야 한다.

## 해결 방법

### 방법 1: API 레이어로 추상화 (권장)

백엔드 서버를 얇게라도 두고, 클라이언트는 REST API만 호출.

```
클라이언트
    ↓ fetch('/api/orders')
서버 (Next.js API Routes / Express 등)
    ↓ supabaseAdmin.from('orders')...
Supabase DB
```

```typescript
// 클라이언트: Supabase를 모름
const orders = await fetch('/api/orders').then(r => r.json())

// 서버: Supabase 직접 접근 (service_role)
// app/api/orders/route.ts
export async function GET(req: Request) {
  const session = await getServerSession()
  const { data } = await supabaseAdmin
    .from('orders')
    .select('*')
    .eq('user_id', session.user.id)
  return Response.json(data)
}
```

이렇게 하면 나중에 Supabase → 다른 DB로 바꿔도 **서버 코드만 수정**, 클라이언트는 그대로.

### 방법 2: 저장소 패턴 (Repository Pattern)

```typescript
// lib/repositories/orders.repository.ts
export class OrdersRepository {
  async findByUserId(userId: string): Promise<Order[]> {
    const { data } = await supabase
      .from('orders')
      .select('*')
      .eq('user_id', userId)
    return data ?? []
  }

  async create(order: CreateOrderDto): Promise<Order> {
    const { data } = await supabase
      .from('orders')
      .insert(order)
      .select()
      .single()
    return data
  }
}

// 클라이언트는 repository만 사용
const repo = new OrdersRepository()
const orders = await repo.findByUserId(userId)
```

Supabase → 다른 DB로 바꿀 때 `OrdersRepository` 내부만 교체.

### 어떤 경우에 직접 연결이 괜찮은가

- MVP/프로토타입: 빠른 검증이 목표, 이전 계획 없음
- 명확하게 Supabase에 장기간 락인 OK
- 개인 프로젝트 / 사이드 프로젝트

장기 운영 서비스이거나 투자 유치 후 스케일업 계획이 있다면 초기부터 추상화 레이어 두는 것이 낫다.

## 예방법

- [ ] 프로젝트 시작 시 "나중에 DB 바꿀 가능성이 있는가?" 질문
- [ ] 있다면 → 처음부터 API 레이어 or Repository Pattern
- [ ] 없다면 → 직접 연결 OK, 단 RLS에 비즈니스 로직 최소화
- [ ] 클라이언트 코드에서 `supabase.from()`이 10곳 이상이면 추상화 검토

## 참고 자료

- 관련 문제: [005 - RLS 올바른 역할](./005-rls-correct-role.md)
