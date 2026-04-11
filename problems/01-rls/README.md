# 01. RLS (Row Level Security)

Supabase 바이브코딩에서 가장 위험한 영역. RLS는 PostgreSQL 기능이지 Supabase만의 것이 아닌데,
AI는 이 맥락 없이 코드를 생성하기 때문에 보안 구멍이 생기기 쉽다.

## 왜 특히 위험한가

- 클라이언트가 DB에 직접 접근하는 Supabase 아키텍처에서 RLS는 유일한 데이터 보호 장치
- AI는 RLS 정책을 "있으면 좋은 것" 수준으로 생성하거나, 아예 누락시킴
- 에러 없이 조용히 데이터 전체가 노출됨 (증상이 안 보임)
- 테스트 환경에서 `service_role` 키를 쓰면 RLS를 우회해서 문제를 발견 못 함

## 문제 목록

| # | 제목 | 심각도 | 상태 |
|---|------|--------|------|
| [001](./001-rls-disabled-by-default.md) | RLS 비활성화 상태로 테이블 생성 | 🔴 Critical | 정리됨 |
| [002](./002-rls-vibe-coding-antipatterns.md) | 바이브코딩 AI가 만드는 RLS 안티패턴 | 🔴 Critical | 정리됨 |
| [003](./003-rls-testing.md) | RLS 정책 테스트 방법 | 🟠 High | 정리됨 |

## 핵심 원칙

1. **테이블 생성 즉시 RLS 활성화** — 나중에 켜려고 하면 잊어버림
2. **`anon` 키로 테스트** — `service_role`은 RLS를 우회하므로 테스트 의미 없음
3. **기본은 전부 차단, 필요한 것만 허용** — `USING (false)` → 하나씩 열기
4. **AI가 생성한 RLS 정책은 반드시 직접 검토** — 그럴듯해 보여도 구멍이 있음
