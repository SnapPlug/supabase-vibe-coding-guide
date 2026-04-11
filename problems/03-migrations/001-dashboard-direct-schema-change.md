# [006] Supabase Dashboard에서 직접 스키마 변경

> 대시보드 UI로 테이블/컬럼을 수정하면 git에 이력이 남지 않아 팀 협업, 롤백, 환경 동기화가 불가능해진다.

- **카테고리**: `03-migrations`
- **심각도**: 🟠 High
- **관련 파일**: `supabase/migrations/`

---

## 문제 상황

Supabase 대시보드 → Table Editor에서 컬럼 추가, 타입 변경, 인덱스 추가를 직접 클릭으로 처리.

또는 SQL Editor에서 직접 `ALTER TABLE` 실행.

migration 파일이야말로 git으로 추적 가능한 코드베이스 관리의 정석이다. migration을 사용하지 않고 Dashboard UI에서 클릭으로 세팅하는 것은 사용자 문제다.

## 바이브코딩에서 이 문제가 발생하는 이유

1. AI가 "컬럼 추가해줘"라고 하면 SQL을 알려주거나 직접 실행 지시를 내림
2. 대시보드가 GUI로 편리하게 제공되어 있어 클릭으로 해결하려는 유혹
3. "일단 빠르게" 개발할 때는 migration 생성이 번거롭게 느껴짐
4. 로컬 → 스테이징 → 프로덕션 환경이 있을 때 동기화가 안 됨

## 증상

```bash
# 로컬에서 돌리면 에러
supabase db diff
# → "프로덕션 DB와 로컬 스키마가 다릅니다"

# 새 팀원이 환경 셋업하면
supabase db reset
# → 대시보드에서 만든 컬럼이 없어서 앱이 깨짐

# 배포 후 롤백 필요할 때
# → migration 파일이 없어서 이전 상태로 돌아갈 방법 없음
```

## 해결 방법

### 원칙: 모든 스키마 변경은 migration 파일로

```bash
# 새 migration 파일 생성
supabase migration new add_profile_picture_to_users

# 생성된 파일 편집
# supabase/migrations/20260412_add_profile_picture_to_users.sql
```

```sql
-- supabase/migrations/20260412_add_profile_picture_to_users.sql
ALTER TABLE users ADD COLUMN profile_picture_url TEXT;

-- 롤백도 명시적으로 (선택사항이지만 권장)
-- DOWN:
-- ALTER TABLE users DROP COLUMN profile_picture_url;
```

```bash
# 로컬에 적용
supabase db push

# 프로덕션에 적용
supabase db push --linked
```

### 이미 대시보드에서 변경했다면: diff로 migration 생성

```bash
# 현재 DB 상태와 migration 파일 간의 차이를 migration 파일로 생성
supabase db diff --use-migra -f catch_up_dashboard_changes

# 생성된 파일 검토 후 커밋
cat supabase/migrations/*catch_up_dashboard_changes.sql
git add supabase/migrations/
git commit -m "chore: catch up dashboard schema changes"
```

### Claude Code에게 스키마 변경 요청하는 방법

```
users 테이블에 profile_picture_url TEXT 컬럼을 추가해줘.
- supabase migration new 명령으로 migration 파일 생성
- 대시보드에서 직접 수정하지 말 것
- migration 파일에 변경 내용 작성
- supabase db push로 로컬에 적용
```

## 대시보드 사용 vs migration 파일 비교

| | 대시보드 직접 수정 | Migration 파일 |
|---|---|---|
| git 추적 | ❌ | ✅ |
| 팀 공유 | ❌ (구두로 전달) | ✅ (PR로 리뷰) |
| 롤백 | ❌ | ✅ |
| 환경 동기화 | ❌ 수동 반복 | ✅ 자동화 가능 |
| CI/CD | ❌ | ✅ |

## 예방법

- [ ] 팀 규칙: "대시보드 Table Editor는 조회만, 수정은 migration 파일로"
- [ ] `supabase/migrations/` 폴더를 PR 필수 검토 경로로 설정
- [ ] 주기적으로 `supabase db diff` 실행해서 drift 감지
- [ ] 신규 팀원 온보딩 문서에 "대시보드 직접 수정 금지" 명시

```bash
# CI에서 schema drift 감지
supabase db diff --use-migra 2>&1 | grep -v "^$" && echo "⚠️ Schema drift detected" && exit 1 || echo "✅ No drift"
```

## 권장 로컬 개발 워크플로우

권장 흐름:

```
로컬 Supabase 개발 DB
    ↓ 스키마 변경 시 반드시
supabase migration new [이름]
    ↓ migration 파일 작성 후
supabase db push  (로컬 반영 + 테스트)
    ↓ 검증 완료 후
supabase db push --linked  (운영/스테이징 반영)
```

## 참고 자료

- [Supabase CLI 마이그레이션 가이드](https://supabase.com/docs/guides/cli/local-development)
- [supabase db diff 문서](https://supabase.com/docs/reference/cli/supabase-db-diff)
