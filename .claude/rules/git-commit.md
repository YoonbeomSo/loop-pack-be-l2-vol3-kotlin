---
description: git commit, git add 등 커밋 관련 작업 시 자동 적용
globs:
  - "**/*"
---

# Git 커밋 규칙

## 커밋 메시지 형식

```
<type>: <subject>

- <변경 내용 1>
- <변경 내용 2>
```

- `type`: `feat`, `fix`, `docs`, `refactor`, `test`, `build`, `chore`
- `subject`: 한글, 변경 대상과 행위를 간결하게 요약
- 본문: 빈 줄 후 구체적인 변경 내용을 `-`로 나열 (3개 이상 변경 시)
- 본문 변경 내용 작성 시 변경 된 코드를 디테일하게 작성하지말것, 함축된 의미로 간략하게 설명할 것

## 커밋 메시지 금지 항목

- `Co-Authored-By:` 절대 포함 금지
- 영어 subject 금지 (한글로 작성)
- `feat: 구현`, `refactor: 리팩토링` 같은 의미 없는 subject 금지
- 파일명 나열 금지 (변경의 의도를 적을 것)

## 커밋 단위

### `feat:` — 도메인 전 레이어를 하나의 커밋

- 하나의 도메인에 대해 domain + application + infrastructure + interfaces를 한 커밋에 포함
- 기존 도메인을 확장하는 경우 별도 커밋 가능
- 예시:
  - `feat: 쿠폰 도메인 구현`
  - `feat: 주문 생성을 위한 상품·브랜드 서비스 확장`

### `test:` — 도메인 전체 테스트를 하나의 커밋

- 단위 테스트 + 통합 테스트 + E2E 테스트를 한 커밋에 포함
- `feat:` 커밋 이후에 `test:` 커밋이 온다
- 예시:
  - `test: 쿠폰 도메인 테스트 추가`
  - `test: 주문 도메인 테스트 추가`

### `refactor:` — 하나의 리팩토링 의도를 하나의 커밋

- 변경의 목적이 명확히 드러나도록 subject 작성
- 예시:
  - `refactor: ProductFacade 제거 및 ProductService로 통합`
  - `refactor: Info/Criteria DTO 도입으로 레이어 간 도메인 격리`

### `docs:` — 설계 문서 변경만 커밋

- `docs/design/` 파일만 커밋 대상
- 예시:
  - `docs: 설계 문서에 쿠폰 도메인 추가`
  - `docs: 클래스 다이어그램을 실제 코드 구조에 맞게 수정`

### `fix:` — 버그 수정 단위로 커밋

- 예시: `fix: 전체 재고 예약 실패 시 에러 메시지 개선`

## 커밋 제외 대상

다음 파일/디렉토리는 절대 커밋하지 않는다:

- `.claude/` (settings, skills, rules 등)
- `docs/plan/`
- `docs/logs/`
- `docs/pr/`
- `docs/blog/`
- `docs/quests/`
- `docs/requirements/`
- `.codeguide/`

## 커밋 전 확인 절차

1. `git diff --cached --name-only`로 staging된 파일 목록을 확인한다
2. 커밋 제외 대상이 staging에 포함되어 있으면 `git reset HEAD -- <file>`로 제거한다
3. 이전에 staging된 파일이 남아있을 수 있으므로 반드시 확인 후 커밋한다
