# [004] `allow *` 로 시작해서 나중에 좁히려다 결국 구멍이 남

> 개발 편의를 위해 RLS를 전면 허용으로 시작하면, 배포 후에도 좁히는 작업을 못 하고 구멍이 슬금슬금 출시된다.

- **카테고리**: `01-rls`
- **심각도**: 🔴 Critical
- **관련 파일**: `supabase/migrations/*.sql`

---

## 문제 상황

빠른 개발을 위해 이렇게 시작한다:

```sql
-- "일단 전부 열고 나중에 좁히자"
CREATE POLICY "allow_all" ON posts
  FOR ALL USING (true);
```

또는 Supabase 대시보드에서 테이블 생성 시 "Enable RLS" 없이 진행.

그리고 나중에 좁히는 작업은... 대부분 일어나지 않는다.

## 바이브코딩에서 이 문제가 발생하는 이유

실무에서 자주 목격되는 패턴이다. allow *(전면허용)을 디폴트로 설정하고 개발을 시작했다가, 나중에 규칙을 재정의하려 해도 잘 되지 않고 보안 구멍이 슬금슬금 출시까지 따라간다.

왜 이런 흐름이 반복되는가:
1. 개발 초반에는 "기능 먼저, 보안은 나중"이라는 압박
2. AI 도구에 "빠르게 만들어줘"라고 하면 allow * 패턴이 나옴
3. 기능이 동작하면 보안 설정을 건드릴 이유가 없어짐 (문제가 안 보이니까)
4. 배포 후 레트로핏은 기존 코드를 다 뜯어봐야 해서 더 어려워짐

**역설적 사실**: Claude/Codex 같은 AI는 오히려 allow * 를 잘 안 쓴다.
AI 도구를 써서 개발할 때 보안 패턴은 오히려 사람보다 낫다. 시간이 걸리더라도 allow * 를 쓰지 않는 경향이 있다.

문제는 AI 모델이 아니라 **"빨리 만들어줘"라는 프롬프트와 검토 없이 수용하는 개발자**에게 있다.

## 증상

에러 없음. 모든 게 잘 동작하는 것처럼 보임.

```typescript
// 비로그인 유저도 전체 데이터 조회 가능
const { data } = await supabase.from('posts').select('*')
// data = [모든 유저의 posts...]

// 다른 유저 데이터 수정도 가능
const { data } = await supabase
  .from('posts')
  .update({ title: 'hacked' })
  .eq('id', 'other-user-post-id')
// 성공해버림
```

## 해결 방법

### 원칙: 처음부터 deny-by-default

RLS는 deny-by-default 구조라 오히려 더 안전하다. Policy를 명시적으로 열어줘야만 접근이 가능하다. 일반 백엔드는 API endpoint마다 개발자가 직접 인증/인가를 추가해야 하는데, 빠뜨려도 아무 경고가 없다. 선언적 보안 모델이 명령적 보안보다 실수 여지가 적다는 것은 업계 정설이다.

```sql
-- 처음부터 이렇게 시작
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
-- 정책 없음 = 전부 차단

-- 필요한 것만 하나씩 열기
CREATE POLICY "posts_select_own" ON posts
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "posts_insert_own" ON posts
  FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "posts_update_own" ON posts
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "posts_delete_own" ON posts
  FOR DELETE USING (auth.uid() = user_id);
```

### 기존 allow * 정책 정리

```sql
-- 1. 현재 정책 확인
SELECT tablename, policyname, cmd, qual
FROM pg_policies
WHERE schemaname = 'public';

-- 2. allow * 정책 찾기
SELECT tablename, policyname
FROM pg_policies
WHERE qual = 'true' OR with_check = 'true';

-- 3. 교체
DROP POLICY "allow_all" ON posts;
CREATE POLICY "posts_select_own" ON posts
  FOR SELECT USING (auth.uid() = user_id);
-- ... 나머지 CRUD 정책 추가
```

### Claude Code에게 요청하는 방법

```
posts 테이블 RLS 정책을 작성해줘.
규칙:
- 기본 전부 차단
- 인증된 유저는 자신의 posts만 CRUD
- admin role(JWT claim에 role='admin')은 전체 조회 가능
- allow * 나 USING(true) 절대 쓰지 말 것
```

## 검증 방법

```sql
-- allow * 패턴 전체 스캔
SELECT
  tablename,
  policyname,
  cmd,
  qual,
  with_check
FROM pg_policies
WHERE schemaname = 'public'
  AND (qual = 'true' OR with_check = 'true' OR qual IS NULL);
```

```typescript
// 비로그인 상태에서 모든 테이블 접근 시도
const tables = ['posts', 'users', 'orders', 'payments']
for (const table of tables) {
  const { data } = await anonClient.from(table).select('*').limit(1)
  if (data && data.length > 0) {
    console.error(`🚨 ${table}: RLS 없음 - 비로그인 접근 가능`)
  }
}
```

## 예방법

- [ ] 신규 프로젝트 시작 시 마이그레이션 템플릿에 RLS ON + 기본 정책 포함
- [ ] AI에게 "빠르게"가 아닌 "보안 포함해서" 요청
- [ ] `allow *` 또는 `USING(true)` 가 포함된 마이그레이션 파일 CI에서 차단
- [ ] 주기적으로 `pg_policies` 스캔으로 전면허용 정책 감지

```bash
# CI 체크: allow * 패턴 감지
grep -r "USING (true)" supabase/migrations/ && echo "⚠️ allow * 정책 발견" && exit 1 || true
```

## 참고 자료

- [Supabase RLS 공식 문서](https://supabase.com/docs/guides/database/postgres/row-level-security)
