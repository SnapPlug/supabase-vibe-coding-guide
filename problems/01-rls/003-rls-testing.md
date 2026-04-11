# [003] RLS 정책 테스트 방법

> RLS는 "설정했다"고 끝이 아니다. 실제로 의도대로 동작하는지 검증하는 방법.

- **카테고리**: `01-rls`
- **심각도**: 🟠 High
- **관련 파일**: `supabase/tests/`, `test/rls/`

---

## 문제 상황

RLS 테스트가 어려운 이유:
1. 다양한 역할(anon, authenticated, admin)로 각각 테스트해야 함
2. 테스트 DB 상태를 매 테스트마다 격리해야 함
3. `service_role`로 테스트하면 RLS가 우회되어 실제 문제를 못 잡음
4. AI가 테스트 코드를 생성할 때 `service_role`을 쓰는 경우가 많음

## 방법 1: pgTAP (PostgreSQL 내장 테스트)

Supabase가 공식 지원하는 방법. SQL로 DB 레벨 테스트 작성.

```bash
# supabase/tests/rls_test.sql 생성
```

```sql
-- supabase/tests/rls/posts_rls_test.sql
BEGIN;

SELECT plan(6);

-- 테스트 유저 생성
SELECT tests.create_supabase_user('user1', 'user1@test.com');
SELECT tests.create_supabase_user('user2', 'user2@test.com');

-- user1으로 post 생성
SELECT tests.authenticate_as('user1');
INSERT INTO posts (title, user_id) VALUES ('user1 post', tests.get_supabase_uid('user1'));

-- user1은 자신의 post 볼 수 있음
SELECT results_eq(
  'SELECT count(*) FROM posts',
  ARRAY[1::bigint],
  'user1은 자신의 post 1개를 볼 수 있다'
);

-- user2로 전환
SELECT tests.authenticate_as('user2');

-- user2는 user1 post 못 봄
SELECT results_eq(
  'SELECT count(*) FROM posts',
  ARRAY[0::bigint],
  'user2는 user1의 post를 볼 수 없다'
);

-- user2가 user1 post 수정 시도 → 0 rows affected
SELECT results_eq(
  'UPDATE posts SET title = ''hacked'' WHERE user_id = tests.get_supabase_uid(''user1'') RETURNING *',
  'SELECT NULL::text WHERE false',
  'user2는 user1의 post를 수정할 수 없다'
);

-- anon으로 전환
SELECT tests.authenticate_as_anon();

-- anon은 아무것도 못 봄
SELECT results_eq(
  'SELECT count(*) FROM posts',
  ARRAY[0::bigint],
  'anon은 posts에 접근할 수 없다'
);

SELECT * FROM finish();
ROLLBACK;
```

```bash
# 실행
supabase test db
```

## 방법 2: JavaScript/TypeScript 통합 테스트

실제 Supabase 클라이언트를 써서 E2E 방식으로 검증.

```typescript
// test/rls/posts.rls.test.ts
import { createClient } from '@supabase/supabase-js'

const SUPABASE_URL = process.env.SUPABASE_URL!
const SUPABASE_ANON_KEY = process.env.SUPABASE_ANON_KEY!

// anon 클라이언트 (service_role 절대 금지)
const anonClient = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

async function createAuthenticatedClient(email: string, password: string) {
  const client = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)
  await client.auth.signInWithPassword({ email, password })
  return client
}

describe('posts RLS', () => {
  let user1Client: ReturnType<typeof createClient>
  let user2Client: ReturnType<typeof createClient>
  let user1PostId: string

  beforeAll(async () => {
    user1Client = await createAuthenticatedClient('user1@test.com', 'test1234')
    user2Client = await createAuthenticatedClient('user2@test.com', 'test1234')
  })

  beforeEach(async () => {
    // user1으로 테스트 post 생성
    const { data } = await user1Client
      .from('posts')
      .insert({ title: 'user1 post' })
      .select()
      .single()
    user1PostId = data.id
  })

  afterEach(async () => {
    // 테스트 데이터 정리 (service_role로만 가능)
    const admin = createClient(SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY!)
    await admin.from('posts').delete().eq('id', user1PostId)
  })

  it('user1은 자신의 post를 볼 수 있다', async () => {
    const { data } = await user1Client.from('posts').select('*').eq('id', user1PostId)
    expect(data).toHaveLength(1)
  })

  it('user2는 user1의 post를 볼 수 없다', async () => {
    const { data } = await user2Client.from('posts').select('*').eq('id', user1PostId)
    expect(data).toHaveLength(0)
  })

  it('anon은 posts에 접근할 수 없다', async () => {
    const { data } = await anonClient.from('posts').select('*')
    expect(data).toHaveLength(0)
  })

  it('user2는 user1의 post를 수정할 수 없다', async () => {
    const { data, error } = await user2Client
      .from('posts')
      .update({ title: 'hacked' })
      .eq('id', user1PostId)
      .select()
    expect(data).toHaveLength(0)
  })
})
```

## 방법 3: Supabase SQL Editor로 빠른 검증

개발 중 빠르게 확인할 때.

```sql
-- 1. 특정 유저로 로컬라이즈
SET LOCAL role TO 'authenticated';
SET LOCAL request.jwt.claims TO '{"sub": "user1-uuid", "role": "authenticated"}';

-- 2. 쿼리 실행
SELECT * FROM posts;
-- → user1의 posts만 나와야 함

-- 3. anon으로 확인
SET LOCAL role TO 'anon';
SELECT * FROM posts;
-- → 빈 결과여야 함
```

## 예방법

- [ ] 새 테이블 RLS 정책 추가할 때마다 테스트 파일도 함께 작성
- [ ] CI 파이프라인에 `supabase test db` 포함
- [ ] 테스트 시드 데이터는 별도 파일로 관리 (`supabase/seed.sql`)
- [ ] 테스트용 유저 계정 미리 준비 (user1, user2, admin 역할별)

## 참고 자료

- [Supabase pgTAP 공식 가이드](https://supabase.com/docs/guides/database/extensions/pgtap)
- [supabase-test-helpers](https://github.com/supabase-community/supabase-test-helpers)
