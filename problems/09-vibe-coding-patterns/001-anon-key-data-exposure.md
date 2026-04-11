# [007] anon 키로 개인정보 노출 + 콘솔로그에 API 키/데이터 찍힘

> RLS 미설정 + anon 키 = 비로그인 유저가 타인 개인정보 조회 가능. 네트워크 탭이나 콘솔로그에 민감한 데이터가 노출된다.

- **카테고리**: `09-vibe-coding-patterns`
- **심각도**: 🔴 Critical
- **관련 파일**: 프론트엔드 코드, `supabase/migrations/*.sql`

---

## 문제 상황 1: 네트워크 탭 개인정보 노출

네트워크 탭에서 개인정보가 노출된다면, 이건 Supabase라서 생긴 문제가 아니다. 보안 설정을 안 해서 생긴 문제다. REST API 서버에서 인증 없이 SELECT * 날리는 것과 본질적으로 동일하다.

브라우저 Network 탭에서 `https://xxx.supabase.co/rest/v1/users?select=*` 요청을 보면:
```json
[
  { "id": "...", "email": "user1@gmail.com", "phone": "010-...", "address": "..." },
  { "id": "...", "email": "user2@gmail.com", "phone": "010-...", "address": "..." }
]
```

비로그인 상태에서도 이게 보인다면 RLS가 없는 것.

## 문제 상황 2: 콘솔로그에 민감한 데이터/키 노출

바이브코딩으로 만든 웹앱에서 API 응답이나 환경변수가 콘솔로그에 그대로 찍히는 경우가 자주 발생한다.

```typescript
// 바이브코딩에서 자주 나오는 패턴
const { data, error } = await supabase.from('users').select('*')
console.log('users data:', data)  // 개인정보 전체가 콘솔에
console.log('supabase response:', { data, error })  // 토큰 포함 가능

// 또는 환경변수 디버깅하다가
console.log('env:', process.env)  // SUPABASE_SERVICE_ROLE_KEY 포함!
```

## 바이브코딩에서 이 문제가 발생하는 이유

1. AI가 디버깅을 위해 `console.log(data)` 추가 → 개인정보 노출
2. 개발 중에는 편의를 위해 모든 데이터 조회 → 프로덕션에 그대로 배포
3. `process.env` 전체를 로그하는 습관 → 시크릿 키 노출
4. RLS 없는 테이블에서 anon 키로 SELECT → 전체 데이터 공개

## 해결 방법

### 1. RLS 설정 (근본 해결)

[001번 문제](../01-rls/001-rls-disabled-by-default.md) 참고.

### 2. console.log 제거/마스킹

```typescript
// 잘못된 패턴
const { data } = await supabase.from('users').select('*')
console.log(data)  // 전체 데이터 노출

// 올바른 패턴: 필요한 정보만 로그
console.log(`Fetched ${data?.length ?? 0} users`)
// 또는 개발 환경에서만
if (process.env.NODE_ENV === 'development') {
  console.log(`users count: ${data?.length}`)
}
```

```typescript
// 환경변수 로그 절대 금지
// console.log(process.env)  // ← 절대 금지

// 특정 값만 확인
console.log('SUPABASE_URL configured:', !!process.env.NEXT_PUBLIC_SUPABASE_URL)
```

### 3. 네트워크 탭 노출 방지

RLS를 설정해도 anon 키로 직접 REST API 호출이 가능하다. 추가로:

```typescript
// select('*') 대신 필요한 컬럼만
const { data } = await supabase
  .from('users')
  .select('id, name, avatar_url')  // email, phone 등 민감 정보 제외
```

```sql
-- 민감한 컬럼은 별도 테이블로 분리
-- users: 공개 가능한 정보
-- user_private_info: 민감 정보 (더 엄격한 RLS)
CREATE TABLE user_private_info (
  user_id UUID REFERENCES users(id) PRIMARY KEY,
  phone TEXT,
  address TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE user_private_info ENABLE ROW LEVEL SECURITY;

-- 자신의 정보만 접근 가능 (제3자 접근 불가)
CREATE POLICY "private_info_own_only" ON user_private_info
  FOR ALL USING (auth.uid() = user_id);
```

### 4. anon 키 노출 자체는 괜찮다

중요한 사실: `anon` 키는 공개되어도 된다. 이건 공개 키다.
진짜 위험한 건 `service_role` 키.

```typescript
// 이건 괜찮음 (anon 키는 public)
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY)

// 이건 절대 클라이언트 코드에 있으면 안 됨
// const supabase = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY)
// service_role은 RLS를 우회하고 모든 데이터에 접근 가능
```

## 프로덕션 전 체크리스트

```bash
# 1. console.log 제거 확인
grep -r "console\.log" src/ --include="*.ts" --include="*.tsx" | grep -v "//.*console"

# 2. process.env 전체 로그 확인
grep -r "console\.log.*process\.env" src/

# 3. service_role 키가 클라이언트 코드에 없는지 확인
grep -r "SERVICE_ROLE" src/ --include="*.ts" --include="*.tsx"
# → 결과가 없어야 함 (서버사이드 코드 제외)

# 4. select('*') 사용 현황
grep -r "\.select\('\*'\)" src/ --include="*.ts" --include="*.tsx"
# → 민감 정보 테이블에서는 특정 컬럼만 조회로 변경 검토
```

## 예방법

- [ ] ESLint rule 추가: `no-console` (프로덕션 빌드에서)
- [ ] 민감 정보(이메일, 폰번호, 주소)는 별도 테이블로 분리 + 엄격한 RLS
- [ ] Claude Code 프롬프트에 "console.log는 개발 환경에서만" 규칙 포함
- [ ] PR 리뷰 체크리스트에 `console.log` 및 `process.env` 전체 로그 항목 추가

## 참고 자료

- [Supabase anon vs service_role](https://supabase.com/docs/guides/api/api-keys)
