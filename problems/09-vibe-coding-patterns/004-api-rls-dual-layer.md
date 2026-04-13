# [011] 클라이언트에서 Supabase 직접 호출 — API 레이어 없이 RLS만 믿는 구조

> 바이브코딩의 기본 패턴은 클라이언트에서 Supabase를 직접 호출한다. RLS 하나만 보안 레이어로 쓰면, 정책 실수 한 번에 데이터 전체가 노출된다.

- **카테고리**: `09-vibe-coding-patterns`
- **심각도**: 🟠 High
- **관련 파일**: `app/api/`, `CLAUDE.md`

---

## 문제 상황

Claude Code에게 "orders 목록 가져와줘"라고 하면 기본으로 이렇게 만든다:

```typescript
// app/orders/page.tsx — Claude 기본 패턴
'use client'

export default function OrdersPage() {
  const [orders, setOrders] = useState([])

  useEffect(() => {
    supabase.from('orders').select('*').then(({ data }) => setOrders(data))
  }, [])
}
```

브라우저에서 Supabase에 직접 요청. API 레이어가 없다.

이 구조의 문제:
- RLS 정책 하나만 잘못 써도 데이터 전체 노출
- 비즈니스 로직이 클라이언트 코드에 섞임
- Supabase REST API를 직접 호출하면 API Route를 우회 가능
- 나중에 다른 DB로 이전할 때 클라이언트 코드 전체를 다시 써야 함

## 바이브코딩에서 이 문제가 발생하는 이유

1. Claude는 "빠른 구현"을 우선시해서 중간 레이어 없이 직접 연결
2. 클라이언트에서 직접 호출하는 게 코드가 적어서 AI 입장에서 "간단한 답"으로 보임
3. `CLAUDE.md`에 아키텍처 규칙이 없으면 매번 다른 패턴이 나옴
4. 개발 중 동작하니까 문제를 발견 못 하고 그대로 배포

## 올바른 구조: API Route + RLS 이중 보안

```
[브라우저]
    ↓ fetch('/api/orders')
[Next.js API Route]  ← 1차: 세션 확인, 비즈니스 로직
    ↓ user JWT로 Supabase 호출
[Supabase DB]
    ↓ RLS              ← 2차: DB 레벨 최후 방어선
[실제 데이터]
```

두 레이어가 각자 독립적으로 방어한다:
- API Route 인증 버그 → RLS가 잡음
- RLS 정책 실수 → API Route가 잡음

## 해결 방법

### 1. CLAUDE.md에 아키텍처 규칙 명시 (프로젝트 시작 전)

```markdown
# 아키텍처 규칙

## DB 접근 원칙
- 클라이언트 컴포넌트에서 supabase.from() 직접 호출 금지
- 모든 DB 접근은 반드시 /app/api/** Route Handler를 통해서만
- API Route에서 반드시 세션 확인 후 user JWT로 Supabase 클라이언트 생성
- service_role은 관리자 전용 API에서만, 절대 클라이언트 노출 금지
- RLS는 추가 안전망으로 항상 활성화 유지
```

### 2. API Route 패턴 (권장)

```typescript
// app/api/orders/route.ts
import { getServerSession } from 'next-auth'
import { createClient } from '@supabase/supabase-js'

export async function GET() {
  // 1차: 서버에서 세션 확인
  const session = await getServerSession()
  if (!session) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  // user JWT로 Supabase 클라이언트 생성 → RLS 적용됨
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      global: {
        headers: { Authorization: `Bearer ${session.access_token}` }
      }
    }
  )

  // 2차: RLS가 auth.uid() = user_id 조건으로 본인 데이터만 반환
  const { data, error } = await supabase.from('orders').select('*')
  if (error) return Response.json({ error: error.message }, { status: 500 })

  return Response.json(data)
}
```

```typescript
// 클라이언트 — Supabase를 전혀 모름
export default function OrdersPage() {
  const [orders, setOrders] = useState([])

  useEffect(() => {
    fetch('/api/orders')
      .then(r => r.json())
      .then(setOrders)
  }, [])
}
```

### 3. Claude Code에게 요청하는 방법

```
orders 목록을 가져오는 기능을 만들어줘.
규칙:
- 클라이언트에서 supabase 직접 호출 금지
- /app/api/orders/route.ts API Route 생성
- API Route에서 getServerSession()으로 인증 확인
- user JWT를 Authorization 헤더에 담아 Supabase 호출
- RLS는 orders 테이블에 auth.uid() = user_id 정책 유지
- 클라이언트는 fetch('/api/orders')만 호출
```

### 4. 기존 프로젝트에서 이전하기

```
현재 클라이언트 컴포넌트에서 supabase.from()을 직접 호출하는
모든 코드를 API Route 패턴으로 이전해줘.
- 각 호출마다 /app/api/[리소스명]/route.ts 파일 생성
- 클라이언트는 fetch()로만 호출하도록 변경
- 기존 RLS 정책은 그대로 유지
```

## 두 구조 비교

| | 클라이언트 직접 호출 | API Route + RLS |
|---|---|---|
| 보안 레이어 | RLS 1개 | API 인증 + RLS 2개 |
| RLS 실수 시 | 💀 데이터 노출 | ✅ API가 막음 |
| 직접 REST 호출 시도 | ❌ 우회 가능 | ✅ RLS가 막음 |
| 비즈니스 로직 위치 | 클라이언트 (노출) | 서버 (보호) |
| DB 이전 난이도 | 높음 (클라이언트 전체 수정) | 낮음 (API Route만 수정) |
| 코드 복잡도 | 낮음 | 약간 높음 |

## 예방법

- [ ] 프로젝트 시작 전 `CLAUDE.md`에 "클라이언트 직접 호출 금지" 규칙 명시
- [ ] PR 리뷰 체크리스트: 클라이언트 컴포넌트에 `supabase.from()` 있는지 확인
- [ ] CI에서 클라이언트 코드의 직접 호출 감지

```bash
# 클라이언트 컴포넌트에서 직접 호출 감지
grep -r "supabase\.from" app/ --include="*.tsx" \
  | grep -v "api/" \
  | grep -v "server" \
  && echo "⚠️ 클라이언트 직접 호출 발견" && exit 1 || true
```

## 참고 자료

- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Supabase Auth with Next.js](https://supabase.com/docs/guides/auth/server-side/nextjs)
- 관련 문제: [005 - RLS 올바른 역할](../01-rls/005-rls-correct-role.md), [008 - DB 마이그레이션 락인](../01-rls/006-rls-direct-client-migration-lockin.md)
