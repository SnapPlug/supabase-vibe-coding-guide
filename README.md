# Supabase 바이브코딩 문제집

> Claude Code(바이브코딩)로 Supabase 백엔드를 개발할 때 실제 발생하는 문제와 해결방안을 정리한 레포지토리.

## 목적

AI 어시스턴트(Claude Code)와 함께 Supabase로 백엔드를 개발하다 보면 반복적으로 마주치는 문제들이 있다.
단순한 "어떻게 쓰나요?" 수준이 아니라, **바이브코딩 특유의 함정** — AI가 그럴듯하게 코드를 짜줬지만 실제로는 보안 구멍이나 운영 문제를 만드는 패턴들을 집중적으로 다룬다.

각 문제는 다음 형식으로 정리한다:
- 어떤 상황에서 발생하는가
- AI가 왜 이 문제를 만들어내는가 (바이브코딩 특화 원인)
- 실제 해결방법과 코드
- 다시는 안 걸리려면 어떻게 해야 하나

## 문제 카테고리

| # | 카테고리 | 핵심 주제 |
|---|---------|---------|
| 01 | [RLS (Row Level Security)](./problems/01-rls/) | 보안 정책 설정, 바이브코딩 함정 |
| 02 | [인증 (Auth)](./problems/02-auth/) | JWT, OAuth, 세션 관리 |
| 03 | [마이그레이션 (Migrations)](./problems/03-migrations/) | 스키마 변경, 버전 관리 |
| 04 | [스키마 설계 (Schema)](./problems/04-schema/) | 테이블 구조, 관계, 인덱스 |
| 05 | [Edge Functions](./problems/05-edge-functions/) | 서버리스 함수 배포/실행 |
| 06 | [스토리지 (Storage)](./problems/06-storage/) | 파일 업로드, 접근 제어 |
| 07 | [실시간 (Realtime)](./problems/07-realtime/) | 구독, 이벤트 처리 |
| 08 | [TypeScript 타입](./problems/08-typescript/) | 자동 생성 타입, 타입 안전성 |
| 09 | [바이브코딩 패턴](./problems/09-vibe-coding-patterns/) | AI 특화 안티패턴, 검증 방법 |

## 문제 파일 구조

각 문제는 [TEMPLATE.md](./TEMPLATE.md)를 기반으로 작성한다.

```
problems/
└── 01-rls/
    ├── README.md          # 카테고리 개요 + 문제 목록
    └── 001-제목.md        # 개별 문제 파일
```

## 참고 스크린샷

`screenshots/` 폴더에 관련 논의, 에러 메시지, 레퍼런스 이미지를 보관한다.

## 기여 방법

새 문제를 발견하면:
1. 해당 카테고리 폴더에 `NNN-제목.md` 파일 생성
2. [TEMPLATE.md](./TEMPLATE.md) 형식 준수
3. 카테고리 `README.md`의 문제 목록에 추가
4. 최상위 [INDEX.md](./INDEX.md) 업데이트
