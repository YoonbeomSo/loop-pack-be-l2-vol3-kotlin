## 📌 Summary
<!--
무엇을/왜 바꿨는지 한눈에 보이게 작성한다.
- 문제(배경) / 목표 / 결과(효과) 중심으로 3~5줄 권장한다.
-->

- 배경:
- 목표:
- 결과:


## 🧭 Context & Decision
<!--
설계 의사결정 기록을 남기는 영역이다.
"왜 이렇게 했는가"가 핵심이다.
-->

### 문제 정의
- 현재 동작/제약:
- 문제(또는 리스크):
- 성공 기준(완료 정의):

### 선택지와 결정
- 고려한 대안:
  - A:
  - B:
- 최종 결정:
- 트레이드오프:
- 추후 개선 여지(있다면):


## 🏗️ Design Overview
<!--
구성 요소와 책임을 간단히 정리한다.
-->

### 변경 범위
- 영향 받는 모듈/도메인:
- 신규 추가:
- 제거/대체:

### 주요 컴포넌트 책임
- `ComponentA`: 
- `ComponentB`: 
- `ComponentC`: 


## 🔁 Flow Diagram
<!--
가능하면 Mermaid로 작성한다. (시퀀스/플로우 중 택1)
"핵심 경로"를 먼저 그리고, 예외 흐름은 아래에 분리한다.
-->

### Main Flow
```mermaid
sequenceDiagram
  autonumber
  participant Client
  participant API
  participant Service
  participant DB
  Client->>API: request
  API->>Service: command/query
  Service->>DB: read/write
  DB-->>Service: result
  Service-->>API: response
  API-->>Client: response
