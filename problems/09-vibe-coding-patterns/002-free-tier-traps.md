# [009] Free 플랜 3가지 함정: 자동백업 없음 / 1주 미접속 정지 / Queue·Cron 유지보수 지옥

> Free 플랜은 강력하지만 모르면 데이터를 날리거나, 서비스가 멈추거나, 기술 부채를 쌓게 된다.

- **카테고리**: `09-vibe-coding-patterns`
- **심각도**: 🟠 High
- **관련 파일**: 없음 (운영 설정 문제)

---

## 함정 1: Free 플랜 자동백업 없음 → 데이터 날림

Free 플랜은 자동백업을 제공하지 않는다. 직접 dump를 뜨거나 별도 스크립트를 설정해야 한다. Pro 플랜은 하루 한 번 최근 7일간의 DB 데이터를 저장해 복구할 수 있다.

### 증상

- 실수로 데이터 삭제 → 복구 불가
- 잘못된 마이그레이션 실행 → 롤백 불가
- 개발 중 `supabase db reset` 실수로 프로덕션 적용 → 전체 데이터 삭제

### 해결 방법

**옵션 A: Pro 플랜 업그레이드** ($25/월)
- 자동 일일 백업 + 7일 보관
- 중요 프로덕션 데이터라면 이게 가장 간단

**옵션 B: pg_dump 스크립트 + cron**

```bash
#!/bin/bash
# scripts/backup.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="backup_${DATE}.sql"

pg_dump "$DATABASE_URL" \
  --no-owner \
  --no-acl \
  --clean \
  --if-exists \
  > "$BACKUP_FILE"

# S3나 R2에 업로드
aws s3 cp "$BACKUP_FILE" "s3://my-backups/supabase/${BACKUP_FILE}"
rm "$BACKUP_FILE"

echo "Backup completed: ${BACKUP_FILE}"
```

```bash
# crontab -e
# 매일 새벽 3시 백업
0 3 * * * /path/to/scripts/backup.sh >> /var/log/backup.log 2>&1
```

**옵션 C: GitHub Actions로 주기적 백업**

```yaml
# .github/workflows/backup.yml
name: Database Backup

on:
  schedule:
    - cron: '0 18 * * *'  # 매일 UTC 18:00 (KST 03:00)
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup Supabase DB
        run: |
          pg_dump "${{ secrets.DATABASE_URL }}" \
            --no-owner --no-acl --clean --if-exists \
            | gzip > backup_$(date +%Y%m%d).sql.gz
      
      - name: Upload to R2
        run: |
          # Cloudflare R2 or S3 업로드
          aws s3 cp backup_*.sql.gz s3://my-backups/ \
            --endpoint-url ${{ secrets.R2_ENDPOINT }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_KEY }}
```

---

## 함정 2: 1주일 미접속 시 프로젝트 일시정지

무료 플랜은 1주일간 트래픽이 없으면 프로젝트가 일시정지된다. cron으로 자동 정기 호출하는 방식은 Supabase 정책 위반 소지가 있다.

### 증상

```
Error: Connection refused
PostgrestError: "Project is paused"
```

유저가 앱에 접속했을 때 DB가 일시정지 상태여서 에러가 발생.
재활성화까지 수십 초~수 분 소요.

### 해결 방법

**권장: Pro 플랜 또는 실제 유저 유입 확인 후 무료 사용**

개발/스테이징 환경이라면 일시정지는 감수해도 됨. 프로덕션이라면 Pro 플랜.

**임시방편: 외부 cron 서비스 (정책 주의)**

```bash
# Uptime Robot, BetterUptime 등에서 5분마다 헬스체크 등록
# URL: https://xxx.supabase.co/rest/v1/health_check?select=*
# 주의: Supabase TOS 위반 소지 있음 - 프로덕션에서는 Pro 플랜 사용 권장
```

---

## 함정 3: Queue·Cron Plugin 유지보수 지옥

Supabase의 Queue/Cron 플러그인은 바로 쓸 수 있지만 유지보수가 매우 어렵다.

### 왜 문제인가

- Queue/Cron 로직이 DB 함수(PL/pgSQL)로 저장됨
- 코드 에디터에서 수정 불가 → Supabase 대시보드에서만 관리
- 버전 관리 어려움 (migration 파일로 관리 가능하지만 복잡)
- 디버깅이 일반 백엔드 코드보다 훨씬 어려움
- 테스트 작성 어려움

### 해결 방법

```
Supabase Queue/Cron Plugin  →  외부 서비스 사용
```

| 용도 | 대안 |
|------|------|
| Cron 작업 | Vercel Cron, GitHub Actions, Railway Cron |
| Message Queue | BullMQ (Redis), AWS SQS, Trigger.dev |
| 백그라운드 작업 | Inngest, Quirrel |

```typescript
// 권장: Trigger.dev 같은 외부 서비스로 코드베이스에서 관리
// trigger.dev
export const dailyCleanup = task({
  id: 'daily-cleanup',
  cron: '0 0 * * *',
  run: async () => {
    // 일반 TypeScript 코드로 작성 → 테스트 가능
    const { data } = await supabaseAdmin
      .from('sessions')
      .delete()
      .lt('expires_at', new Date().toISOString())
    
    console.log(`Cleaned up ${data?.length} expired sessions`)
  }
})
```

## 요약

| 함정 | 결과 | 해결 |
|------|------|------|
| 자동백업 없음 | 데이터 영구 손실 | Pro 업그레이드 or pg_dump cron |
| 1주 미접속 정지 | 유저가 에러 경험 | Pro 업그레이드 (프로덕션) |
| Queue/Cron Plugin | 기술 부채 누적 | 외부 서비스로 코드베이스 관리 |

## 예방법

- [ ] 프로덕션 서비스라면 처음부터 Pro 플랜 고려 ($25/월)
- [ ] pg_dump 스크립트 + GitHub Actions로 주기적 백업 자동화
- [ ] Queue/Cron은 Trigger.dev, Inngest 등 코드베이스 관리 가능한 서비스 사용

## 참고 자료

- [Supabase 플랜 비교](https://supabase.com/pricing)
- [Supabase pg_dump 가이드](https://supabase.com/docs/guides/database/postgres/pg-dump)
