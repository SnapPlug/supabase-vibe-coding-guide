# 08. TypeScript 타입

`supabase gen types` 로 자동 생성되는 DB 타입과 실제 사용 패턴. AI는 오래된 타입이나
부정확한 타입을 기반으로 코드를 생성해 런타임 에러를 만든다.

## 문제 목록

| # | 제목 | 심각도 | 상태 |
|---|------|--------|------|
| - | (문제 발견 시 추가) | - | - |

## 자주 발생하는 패턴

- 스키마 변경 후 타입 재생성 누락 → 타입과 실제 DB 불일치
- `Database['public']['Tables']['users']['Row']` 타입 직접 사용 vs 헬퍼 타입
- `null` vs `undefined` 처리 불일치
- Supabase SDK 버전에 따른 타입 API 변경 처리
