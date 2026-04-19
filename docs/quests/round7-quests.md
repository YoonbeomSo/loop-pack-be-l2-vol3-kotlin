# 📝 Round 7 Quests

---

## 💻 Implementation Quest

> 이벤트 기반 아키텍처의 **Why → How → Scale** 을 한 주에 관통합니다.
Spring `ApplicationEvent`로 **경계를 나누는 감각**을 익히고,
Kafka로 **이벤트 파이프라인**을 구축한 뒤, **선착순 쿠폰 발급**에 적용합니다
>

<aside>
🎯

**Must-Have (이번 주에 무조건 가져가야 좋을 것-**무조건 ****하세요**)**

- Event vs Command
- Application Event
- Kafka Producer / Consumer 기본 파이프라인
- Transactional Outbox Pattern
- Kafka 기반 선착순 쿠폰 발급

**Nice-To-Have (부가적으로 가져가면 좋을 것-**시간이 ****허락하면 ****꼭 ****해보세요**)**

- Consumer Group 분리를 통한 관심사별 처리
- Consumer 배치 처리
- DLQ 구성
</aside>

### 📋 과제 정보

**Step 1 — ApplicationEvent로 경계 나누기**

- **무조건 이벤트 분리**가 아니라, 주요 로직과 부가 로직의 경계를 판단한다.
- 주문–결제 플로우에서 부가 로직(유저 행동 로깅, 알림 등)을 이벤트로 분리한다.
- 좋아요–집계 플로우에서 eventual consistency를 적용한다.
- 트랜잭션 결과와의 상관관계에 따라 적절한 리스너(`@TransactionalEventListener` phase 등)를 활용한다.
- "이걸 이벤트로 분리해야 하는가? 에 대한 **판단 기준** 자체가 학습 포인트다.

**Step 2 — Kafka 이벤트 파이프라인**

- `commerce-api` → Kafka → `commerce-collector` 구조로 확장한다.
- Step 1에서 분리한 이벤트 중, **시스템 간 전파가 필요한 것**을 Kafka로 발행한다.
- Producer는 **Transactional Outbox Pattern**으로 At Least Once 발행을 보장한다.
- Consumer는 이벤트를 수취해 집계(좋아요 수 / 판매량 / 조회 수)를 `product_metrics`에 upsert한다.

**Step 3 — Kafka 기반 선착순 쿠폰 발급**

- Step 2에서 익힌 Kafka를 **실전 시나리오**에 적용한다.
- API는 발급 요청을 Kafka에 발행만 하고, Consumer가 실제 쿠폰을 발급한다.
- 발급 수량 제한(e.g. 선착순 100명)에 대한 **동시성 제어**를 구현한다.

**토픽 설계** (예시)

- `catalog-events` (상품/재고/좋아요 이벤트, key=productId)
- `order-events` (주문/결제 이벤트, key=orderId)
- `coupon-issue-requests` (쿠폰 발급 요청, key=couponId)

**Producer, Consumer 필수 처리**

- **Producer**
    - acks=all, idempotence=true 설정
- **Consumer**
    - **manual Ack** 처리
    - `event_handled(event_id PK)` (DB or Redis) 기반의 멱등 처리
    - `version` 또는 `updated_at` 기준으로 최신 이벤트만 반영

> *왜 이벤트 핸들링 테이블과 로그 테이블을 분리하는 걸까? 에 대해 고민해보자*
>

---

## ✅ Checklist

### 🧾 Step 1 — ApplicationEvent

- [ ]  주문–결제 플로우에서 부가 로직을 이벤트 기반으로 분리한다.
- [ ]  좋아요 처리와 집계를 이벤트 기반으로 분리한다. (집계 실패와 무관하게 좋아요는 성공)
- [ ]  유저 행동(조회, 클릭, 좋아요, 주문 등)에 대한 서버 레벨 로깅을 이벤트로 처리한다.
- [ ]  동작의 주체를 적절하게 분리하고, 트랜잭션 간의 연관관계를 고민해 봅니다.

### 🎾 Step 2 — Kafka Producer / Consumer

- [ ]  Step 1의 ApplicationEvent 중 **시스템 간 전파가 필요한 이벤트**를 Kafka로 발행한다.
- [ ]  `acks=all`, `idempotence=true` 설정
- [ ]  **Transactional Outbox Pattern** 구현
- [ ]  PartitionKey 기반 이벤트 순서 보장
- [ ]  Consumer가 Metrics 집계 처리 (product_metrics upsert)
- [ ]  `event_handled` 테이블을 통한 멱등 처리 구현
- [ ]  manual Ack + `version`/`updated_at` 기준 최신 이벤트만 반영

### 🎫 Step 3 — 선착순 쿠폰 발급

- [ ]  쿠폰 발급 요청 API → Kafka 발행 (비동기 처리)
- [ ]  Consumer에서 선착순 수량 제한 + 중복 발급 방지 구현
- [ ]  발급 완료/실패 결과를 유저가 확인할 수 있는 구조 설계 (polling or callback)
- [ ]  동시성 테스트 — 수량 초과 발급이 발생하지 않는지 검증

---

## ✍️ Technical Writing Quest

> 이번 주에 학습한 내용, 과제 진행을 되돌아보며
**"내가 어떤 판단을 하고 왜 그렇게 구현했는지"** 를 글로 정리해봅니다.
>
>
> **좋은 블로그 글은 내가 겪은 문제를, 타인도 공감할 수 있게 정리한 글입니다.**
>
> 이 글은 단순 과제가 아니라, **향후 이직에 도움이 될 수 있는 포트폴리오** 가 될 수 있어요.
>

### 📚 Technical Writing Guide

### ✅ 작성 기준

| 항목 | 설명 |
| --- | --- |
| **형식** | 블로그 |
| **길이** | 제한 없음, 단 꼭 **1줄 요약 (TL;DR)** 을 포함해 주세요 |
| **포인트** | “무엇을 했다” 보다 **“왜 그렇게 판단했는가”** 중심 |
| **예시 포함** | 코드 비교, 흐름도, 리팩토링 전후 예시 등 자유롭게 |
| **톤** | 실력은 보이지만, 자만하지 않고, **고민이 읽히는 글**예: “처음엔 mock으로 충분하다고 생각했지만, 나중에 fake로 교체하게 된 이유는…” |

---

### ✨ 좋은 톤은 이런 느낌이에요

> 내가 겪은 실전적 고민을 다른 개발자도 공감할 수 있게 풀어내자
>

| 특징 | 예시 |
| --- | --- |
| 🤔 내 언어로 설명한 개념 | Stub과 Mock의 차이를 이번 주문 테스트에서 처음 실감했다 |
| 💭 판단 흐름이 드러나는 글 | 처음엔 도메인을 나누지 않았는데, 테스트가 어려워지며 분리했다 |
| 📐 정보 나열보다 인사이트 중심 | 테스트는 작성했지만, 구조는 만족스럽지 않다. 다음엔… |

### ❌ 피해야 할 스타일

| 예시 | 이유 |
| --- | --- |
| 많이 부족했고, 반성합니다… | 회고가 아니라 일기처럼 보입니다 |
| Stub은 응답을 지정하고… | 내 생각이 아닌 요약문처럼 보입니다 |
| 테스트가 진리다 | 너무 단정적이거나 오만해 보입니다 |

### 🎯 Feature Suggestions

- ApplicationEvent만으로 충분한 경계 vs Kafka가 필요한 경계, 그 기준은?
- 트랜잭션 안에 다 넣을 수 있지만, 굳이 나누는 이유
- 좋아요는 동기, 집계는 비동기 — 상품의 좋아요 수가 바로 반영되어야 할까?
- Outbox Pattern 없이 Kafka만 쓰면 어떤 일이 벌어질까?
- 선착순 쿠폰을 Redis로 처리하는 것과 Kafka로 처리하는 것의 차이
- 100장 한정 쿠폰에 1만 명이 동시에 요청하면?
- 멱등 처리를 DB로 할 때와 Redis로 할 때의 트레이드오프