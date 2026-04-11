# [001] RLS 비활성화 상태로 테이블 생성

> 바이브코딩으로 테이블을 만들면 기본적으로 RLS가 꺼져 있어, 모든 데이터가 누구에게나 노출된다.

- **카테고리**: `01-rls`
- **심각도**: 🔴 Critical
- **관련 파일**: `supabase/migrations/*.sql`

---

## 문제 상황

Claude Code에게 "users 테이블 만들어줘"라고 하면 이런 코드가 나온다:

```sql
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Supabase 클라이언트에서 `anon` 키로 접근하면 **모든 사용자의 데이터를 읽을 수 있다.**
에러도 없고, 경고도 없다.

## 바이브코딩에서 이 문제가 발생하는 이유

AI는 SQL 테이블 생성 문법은 정확히 알고 있지만, Supabase의 보안 모델을 이해하지 못한다.

- **일반 백엔드 지식**: Express/NestJS에서는 미들웨어가 인증을 처리하므로 DB 레벨 보안 정책이 없어도 됨
- **Supabase 특수성**: 클라이언트가 DB에 직접 접근하므로 RLS가 유일한 보호 장치
- AI는 "빠른 구현"을 우선시해서 보안 설정을 "나중에 추가할 것"으로 처리함
- 테스트 환경에서 `service_role` 키를 쓰면 RLS 우회 → 문제를 발견 못 하고 지나침

## 증상

에러가 없다. 오히려 잘 동작하는 것처럼 보인다.

```typescript
// 이게 성공해버림 (로그인 안 해도)
const { data } = await supabase.from('users').select('*')
// data = [{ id: '...', email: 'user1@...' }, { id: '...', email: 'user2@...' }, ...]
// 전체 유저 목록이 노출
```

Supabase 대시보드에서 확인:
```
Table: users
RLS enabled: false  ← 이게 문제
```

## 해결 방법

### 방법 1: 테이블 생성과 동시에 RLS 활성화 (권장)

```sql
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 즉시 RLS 활성화
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- 기본 정책: 자신의 데이터만 조회 가능
CREATE POLICY "users_select_own" ON users
  FOR SELECT
  USING (auth.uid() = id);

-- 자신의 데이터만 수정 가능
CREATE POLICY "users_update_own" ON users
  FOR UPDATE
  USING (auth.uid() = id);

-- INSERT는 auth.uid()와 id가 일치할 때만
CREATE POLICY "users_insert_own" ON users
  FOR INSERT
  WITH CHECK (auth.uid() = id);
```

### 방법 2: 기존 테이블에 RLS 추가

```sql
-- 이미 만들어진 테이블이라면
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- 정책 없으면 아무도 접근 못 함 (기본 차단)
-- 필요한 정책만 하나씩 추가
```

### Claude Code에게 요청하는 방법

"테이블 만들어줘" 대신:

```
users 테이블을 생성해줘. 반드시:
1. RLS를 활성화하고
2. 자신의 row만 SELECT/UPDATE 가능한 정책을 추가하고
3. 관리자(role = 'admin')는 전체 조회 가능한 정책도 추가해줘
```

## 검증 방법

```typescript
// anon 키로 테스트 (service_role 절대 금지)
const supabase = createClient(url, process.env.SUPABASE_ANON_KEY)

// 로그인 없이 조회 → 빈 배열이어야 함
const { data, error } = await supabase.from('users').select('*')
console.log(data)   // []
console.log(error)  // null (에러 없이 빈 결과)

// 다른 유저로 로그인 후 타인 데이터 조회 시도 → 빈 배열이어야 함
await supabase.auth.signInWithPassword({ email: 'user2@test.com', password: '...' })
const { data: otherData } = await supabase
  .from('users')
  .select('*')
  .eq('id', 'user1-uuid')
console.log(otherData)  // [] (user1 데이터 접근 불가)
```

## 예방법

- [ ] **마이그레이션 파일 템플릿에 RLS 활성화 포함** — 새 테이블은 무조건 RLS ON
- [ ] **CI에서 RLS 없는 테이블 감지** — `SELECT tablename FROM pg_tables WHERE schemaname = 'public' AND rowsecurity = false`
- [ ] **테스트는 항상 `anon` 키로** — `service_role`은 마이그레이션/시드 데이터 용도만
- [ ] **Claude Code에게 보안 검토 별도 요청** — "이 스키마의 RLS 구멍을 찾아줘"

## RLS가 없는 테이블 찾기

```sql
-- 프로젝트의 RLS 미적용 테이블 목록
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND rowsecurity = false;
```

## 참고 자료

- [Supabase RLS 공식 문서](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase Auth + RLS 가이드](https://supabase.com/docs/guides/auth/row-level-security)
