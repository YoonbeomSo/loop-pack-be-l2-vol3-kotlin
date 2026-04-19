# Round 7 구현 계획: 이벤트 기반 아키텍처 + Kafka

## Context

지금까지 주문-결제-재고-쿠폰 흐름을 하나의 트랜잭션에서 처리해왔다. 이번 라운드에서는:
1. ApplicationEvent로 핵심/부가 로직의 경계를 나누고
2. Kafka로 시스템 간 이벤트 파이프라인을 구축하며
3. 선착순 쿠폰 발급에 Kafka를 적용한다

**점진적 고도화 전략**: 처음부터 Kafka/Outbox를 도입하지 않고, Step 1에서 ApplicationEvent로 경계 분리의 감각을 익힌 뒤, Step 2에서 시스템 간 전파가 필요한 이벤트만 Kafka로 전환한다. "문제가 생기면 고도화"하는 실무적 사고 흐름을 따른다.

---

## Step 1 — ApplicationEvent로 경계 나누기

### 1.1 이벤트 클래스 생성

`commerce-api/application/event/` 하위에 생성:

| 이벤트 | 발행 시점 | 용도 |
|--------|----------|------|
| `OrderCreatedEvent` | 주문 생성 후 | 유저 행동 로깅, Kafka 전파 |
| `PaymentCompletedEvent` | 결제 성공 후 | 유저 행동 로깅, Kafka 전파 |
| `PaymentFailedEvent` | 결제 실패 후 | 로깅 |
| `LikeToggledEvent` | 좋아요 토글 후 | likeCount 집계 (eventual consistency), 유저 행동 로깅 |
| `ProductViewedEvent` | 상품 조회 시 | 유저 행동 로깅 |

> `UserActivityEvent`는 별도 클래스로 만들지 않는다. `UserActivityEventHandler`가 각 도메인 이벤트를 직접 수신하여 로깅한다.

> **`ProductViewedEvent`는 `@EventListener` + `@Async`를 사용한다**: `ProductService.getProductInfo()`는 readOnly 트랜잭션이므로 보호할 쓰기가 없다. `AFTER_COMMIT`으로 커밋을 기다릴 의미가 없으므로 `@EventListener` + `@Async`로 즉시 비동기 실행한다. 매 조회마다 이벤트가 발생해 부하가 있을 수 있지만, Step 2에서 Kafka 집계로 분산된다.

### 1.2 기존 코드 수정

**LikeService** (핵심 변경):
- `addLike()` / `cancelLike()`에서 `productService.incrementLikeCount()` 직접 호출 제거
- 대신 `LikeToggledEvent` 발행 → 좋아요 자체는 강한 일관성, 집계는 eventual consistency

**OrderFacade.createOrder()**:
- 주문 생성 성공 후 `OrderCreatedEvent` 발행

**PaymentFacade.handleCallback()**:
- 결제 성공/실패 후 `PaymentCompletedEvent` / `PaymentFailedEvent` 발행

**ProductService (상품 조회)**:
- 조회 시 `ProductViewedEvent` 발행

### 1.3 이벤트 핸들러

`commerce-api/application/event/handler/` 하위:

| 핸들러 | 리스너 타입 | 처리 내용 |
|--------|-----------|----------|
| `LikeCountEventHandler` | `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` | LikeToggledEvent → productService.increment/decrementLikeCount() |
| `UserActivityEventHandler` | `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` | 모든 이벤트 → UserActivityLog 저장 |

**이벤트 ↔ 핸들러 매핑**:

| 이벤트 | 핸들러 | 처리 |
|--------|--------|------|
| `OrderCreatedEvent` | `UserActivityEventHandler` | 유저 행동 로깅 (ORDER) |
| `PaymentCompletedEvent` | `UserActivityEventHandler` | 유저 행동 로깅 (PAYMENT) |
| `PaymentFailedEvent` | `UserActivityEventHandler` | 유저 행동 로깅 (PAYMENT_FAILED) |
| `LikeToggledEvent` | `LikeCountEventHandler` | Product.likeCount 업데이트 |
| `LikeToggledEvent` | `UserActivityEventHandler` | 유저 행동 로깅 (LIKE) |
| `ProductViewedEvent` | `UserActivityEventHandler` | 유저 행동 로깅 (VIEW) — `@EventListener` + `@Async` (readOnly 트랜잭션이므로 커밋 대기 불필요) |

### 1.4 새 엔티티

**UserActivityLog** (`domain/activity/`):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK | |
| user_id | BIGINT | |
| activity_type | VARCHAR | VIEW, LIKE, ORDER, PAYMENT |
| target_type | VARCHAR | PRODUCT, ORDER |
| target_id | BIGINT | |
| metadata | TEXT (nullable) | JSON 부가 데이터 |
| created_at | DATETIME | |

### 1.5 AsyncConfig

- `@EnableAsync` + `ThreadPoolTaskExecutor` 빈 등록
- `config/AsyncConfig.kt`

### 1.6 커밋 전략

```
feat: ApplicationEvent 기반 도메인 이벤트 분리
test: ApplicationEvent 도메인 이벤트 테스트 추가
```

---

## Step 2 — Kafka 이벤트 파이프라인

### 2.1 토픽 설계

| 토픽 | Key | 이벤트 |
|------|-----|--------|
| `catalog-events` | productId | PRODUCT_LIKED, PRODUCT_UNLIKED, PRODUCT_VIEWED |
| `order-events` | orderId | ORDER_CREATED, PAYMENT_COMPLETED, PAYMENT_FAILED |
| `coupon-issue-requests` | couponId | COUPON_ISSUE_REQUESTED (Step 3) |

### 2.2 Kafka 메시지 표준 포맷

`modules/kafka/event/KafkaEventMessage.kt`:

```kotlin
data class KafkaEventMessage(
    val eventId: String,          // UUID (멱등성 키)
    val eventType: String,
    val aggregateType: String,
    val aggregateId: String,
    val payload: Map<String, Any?>,
    val version: Long,
    val occurredAt: ZonedDateTime,
)
```

### 2.3 Transactional Outbox Pattern (Polling 방식)

**OutboxEvent 엔티티** (`commerce-api/domain/outbox/`):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK | |
| aggregate_type | VARCHAR | ORDER, PRODUCT, COUPON |
| aggregate_id | VARCHAR | |
| event_type | VARCHAR | ORDER_CREATED 등 |
| topic | VARCHAR | Kafka 토픽명 |
| partition_key | VARCHAR | Kafka 메시지 키 |
| payload | TEXT | JSON 직렬화된 이벤트 |
| status | VARCHAR | PENDING, SENT, FAILED |
| created_at | DATETIME | |
| sent_at | DATETIME (nullable) | |

**발행 흐름 — 쓰기 이벤트 (Outbox)**:
1. 서비스 메서드 내에서 도메인 로직 + `outboxService.save()` → **같은 트랜잭션에서 직접 INSERT**
2. `OutboxEventPublisher` (`@Scheduled(fixedDelay=1000)`) → PENDING 이벤트 조회 → KafkaTemplate.send() → 성공 시 SENT로 업데이트

**발행 흐름 — 조회 이벤트 (직접 발행)**:
- `ProductViewedEvent`는 readOnly 트랜잭션이므로 Outbox 불필요 (보호할 쓰기가 없음)
- `ProductViewedKafkaHandler`(`@EventListener` + `@Async`)에서 직접 `KafkaTemplate.send()` 호출
- 조회 유실이 발생해도 데이터 정합성 문제 없음 (At Least Once 보장 불필요)

> **왜 서비스에서 직접 INSERT하는가?**: `@TransactionalEventListener(BEFORE_COMMIT)` 방식도 가능하지만, Outbox가 어디서 저장되는지 코드에서 바로 보이는 게 디버깅과 가독성에 유리하다. `OutboxService` 헬퍼로 중복 코드를 최소화했다.
>
> **왜 readOnly에는 Outbox를 쓰지 않는가?**: Outbox 패턴의 목적은 쓰기 트랜잭션과 이벤트 발행의 원자성 보장이다. readOnly 조회에는 보호할 쓰기가 없으므로 Outbox가 불필요하다.

**commerce-api에 modules/kafka 의존성 추가 필요** (현재 없음)

### 2.4 Producer 설정 변경

`kafka.yml` Producer 설정:

| 설정 | 값 | 이유 |
|------|-----|------|
| `acks` | `all` | 리더 + 모든 ISR 저장 확인 → 메시지 유실 방지 |
| `retries` | `2147483647` (MAX) | idempotence가 중복 방지하므로 delivery.timeout.ms 내 무한 재시도가 안전 |
| `enable.idempotence` | `true` | 리트라이 시 동일 메시지 중복 저장 방지 |
| `max.in.flight.requests.per.connection` | `5` | idempotence=true 시 최대값, 순서 보장 + 병렬 요청 |
| `delivery.timeout.ms` | `120000` | 발행 최대 대기 120초 — retries와 함께 작동, 초과 시 실패 |
| `linger.ms` | `5` | 5ms 동안 메시지를 모아서 배치 전송 — Outbox Polling 1초 간격이라 5ms 추가 지연은 무시 가능 |

> **retries를 MAX로 설정한 근거**: `enable.idempotence=true`가 재시도 시 중복을 막아주므로 재시도 횟수를 제한할 이유가 없다. `delivery.timeout.ms=120000`이 실제 상한선 역할을 한다. retries=3이면 일시적 네트워크 장애에서 불필요하게 실패할 수 있다.

### 2.5 Consumer 구현 (commerce-streamer)

**새 패키지 구조**:
```
commerce-streamer/
  domain/
    event/EventHandled.kt, EventHandledRepository.kt
    metrics/ProductMetrics.kt, ProductMetricsRepository.kt
  infrastructure/
    event/EventHandledJpaRepository.kt, EventHandledRepositoryImpl.kt
    metrics/ProductMetricsJpaRepository.kt, ProductMetricsRepositoryImpl.kt
  application/
    metrics/MetricsAggregationService.kt
    event/IdempotencyService.kt
  interfaces/consumer/
    CatalogEventConsumer.kt
    OrderEventConsumer.kt
```

**EventHandled 엔티티** (멱등 처리):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| event_id | VARCHAR PK | UUID |
| aggregate_type | VARCHAR | |
| aggregate_id | VARCHAR | |
| event_type | VARCHAR | |
| handled_at | DATETIME | |

**ProductMetrics 엔티티** (집계):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK | |
| product_id | BIGINT UNIQUE | |
| like_count | INT (default 0) | |
| order_count | INT (default 0) | |
| view_count | INT (default 0) | |
| version | BIGINT | 최신 이벤트 판별용 |
| created_at | DATETIME | |
| updated_at | DATETIME | |

> **Exactly Once는 환상이다**: 실제로는 "여러 번 시도 + 중복 방지 로직"으로 Exactly Once처럼 보이게 만드는 것이다. Producer는 At Least Once로 발행하고, Consumer는 EventHandled 테이블로 중복 처리를 방지해서 End-to-End Exactly Once 효과를 달성한다.

**Consumer 처리 흐름**:
1. 메시지 수신
2. `IdempotencyService.isAlreadyHandled(eventId)` → 중복이면 skip
3. version 비교 → 현재 metrics.version 이상일 때만 처리
4. `ProductMetrics` upsert (likeCount, viewCount, orderCount 등)
5. `EventHandled` 기록
6. Manual ACK

### 2.6 Consumer Group 설계

| 토픽 | Consumer Group | 앱 | 용도 |
|------|---------------|-----|------|
| `catalog-events` | `metrics-consumer` | commerce-streamer | ProductMetrics 집계 |
| `order-events` | `metrics-consumer` | commerce-streamer | orderCount 집계 |
| `coupon-issue-requests` | `coupon-issue-consumer` | commerce-streamer | 쿠폰 발급 처리 |
| `coupon-issue-results` | `coupon-result-consumer` | commerce-api | 발급 결과 수신 |

### 2.7 EventHandled vs UserActivityLog — 왜 분리하는가?

| 구분 | EventHandled | UserActivityLog |
|------|-------------|-----------------|
| 목적 | 중복 처리 방지 (멱등성) | 유저 행동 기록 (분석/추적) |
| 데이터 | eventId PK만으로 존재 여부 확인 | userId, activityType, targetId 등 상세 정보 |
| 생명주기 | 일정 기간 후 삭제 가능 (처리 완료된 이벤트) | 장기 보관 (분석 데이터) |
| 위치 | commerce-streamer (Consumer 측) | commerce-api (Producer 측) |
| 조회 패턴 | PK 기반 exists 체크 (빠른 조회) | 유저별/상품별 조회 (인덱스 필요) |

> 하나의 테이블로 합치면 멱등 체크용 빠른 조회와 분석용 상세 기록이 섞여 인덱스/조회 성능이 저하된다. 목적이 다르면 테이블도 분리하는 것이 맞다.

### 2.8 KafkaConfig 확장

- 단건 처리용 `singleListenerContainerFactory` 추가 (Step 3 쿠폰용)

### 2.9 토픽 생성 설정

`KafkaTopicConfig`에서 Spring Boot 시작 시 토픽을 자동 생성한다 (`NewTopic` Bean).
kafka.yml의 `auto.create.topics.enable: false`와 별개로, Spring의 `KafkaAdmin`이 `NewTopic` Bean을 감지해 생성.

| 토픽 | 파티션 수 | replicas | 이유 |
|------|----------|----------|------|
| catalog-events | 3 | 1 | 향후 Consumer 확장 여유, 실무 기준 최소 3개 |
| order-events | 3 | 1 | 동일 |
| coupon-issue-requests | 3 | 1 | 파티션 분산 가능하지만 concurrency=1로 순차 처리 |
| coupon-issue-results | 3 | 1 | 동일 |

> replicas는 Broker 1대(로컬 개발)라 1로 유지. 실무에서는 Broker 3대 + replicas=3이 표준.
> 파티션 3개로 설정하면 향후 concurrency를 3까지 늘릴 수 있고, 파티션은 줄일 수 없으므로 초기에 여유를 두는 게 좋다.

### 2.10 테스트 전략

| 테스트 | 위치 | 검증 내용 |
|--------|------|----------|
| `OutboxEventHandlerTest` → **삭제** | - | BEFORE_COMMIT 방식에서 서비스 직접 INSERT로 변경하면서 제거 |
| `MetricsAggregationServiceTest` | commerce-streamer | stale 이벤트 거부, 신규 생성 시 save 2회 검증 |
| `IdempotencyServiceTest` | commerce-streamer | 중복 체크, 처리 완료 기록 검증 |
| `OutboxEventPublisherIntegrationTest` | commerce-api | **로컬 Docker Kafka 연동** — PENDING Outbox → Kafka 발행 → SENT 전환 검증 (Awaitility) |

> 통합 테스트는 Testcontainers가 아닌 **로컬 Docker Compose Kafka**(localhost:19092)를 사용한다. 기존 MySQL도 동일한 방식.

### 2.11 커밋 전략

```
feat: Transactional Outbox Pattern 및 Kafka Producer 구현
feat: Kafka Consumer 및 메트릭 집계 구현
test: Kafka 이벤트 파이프라인 테스트 추가
```

---

## Step 3 — Kafka 기반 선착순 쿠폰 발급

### 3.1 토픽 설계 (Step 3 전용)

| 토픽 | Key | 방향 | 설명 |
|------|-----|------|------|
| `coupon-issue-requests` | couponId | api → streamer | 발급 요청 |
| `coupon-issue-results` | couponId | streamer → api | 발급 결과 콜백 |

**왜 콜백 토픽인가?**
- 두 앱이 같은 DB를 공유하지만, MSA 관점에서 서비스 간 DB 직접 접근은 결합도를 높임
- 콜백 토픽을 사용하면 commerce-streamer가 결과를 발행하고, commerce-api가 소비해서 자기 DB에 저장
- 향후 서비스 분리 시에도 구조 변경 없이 대응 가능
- 이벤트 기반 비동기 통신의 학습 목적에도 부합

```
[사용자] → POST /api/v1/coupons/{couponId}/fcfs-issue
  → commerce-api:
    1. CouponIssueRequest(PENDING) 저장 (자체 DB)
    2. coupon-issue-requests 토픽에 발행
    3. 202 Accepted + requestId 반환

  → commerce-streamer (Consumer):
    1. coupon-issue-requests 소비
    2. 멱등성/중복/수량 체크
    3. IssuedCoupon 생성 (또는 거절)
    4. coupon-issue-results 토픽에 결과 발행

  → commerce-api (Result Consumer):
    1. coupon-issue-results 소비
    2. CouponIssueRequest 상태 업데이트 (SUCCESS/FAILED)

[사용자] → GET /api/v1/coupons/fcfs-issue/status?requestId={requestId}
  → commerce-api: CouponIssueRequest 조회 (자체 DB)
```

### 3.2 API (commerce-api)

**새 엔드포인트**:
- `POST /api/v1/coupons/{couponId}/fcfs-issue` → Kafka에 발행, 202 Accepted + requestId 반환
- `GET /api/v1/coupons/fcfs-issue/status?requestId={requestId}` → 자체 DB에서 결과 조회

**Coupon 엔티티 변경**:
- `maxIssueCount: Int?` 필드 추가 (null이면 일반 쿠폰, 값 있으면 선착순 쿠폰)

**CouponIssueRequest 엔티티** (commerce-api 측):

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK | |
| request_id | VARCHAR UNIQUE | UUID (API 반환값 & Kafka 메시지 key) |
| coupon_id | BIGINT | |
| user_id | BIGINT | |
| status | VARCHAR | PENDING, SUCCESS, FAILED |
| failure_reason | VARCHAR (nullable) | |
| created_at | DATETIME | |
| updated_at | DATETIME | |

**CouponIssueResultConsumer** (commerce-api 내 새 Consumer):
- `coupon-issue-results` 토픽 리스닝
- 수신한 결과로 CouponIssueRequest 상태 업데이트
- commerce-api가 Kafka Consumer 역할도 수행 → `modules/kafka` 의존성 활용

### 3.3 엔티티 전략 — commerce-streamer에 별도 정의

modules/jpa로 이동하지 않고, commerce-streamer에 **필요한 필드와 행위만 노출하는 별도 엔티티**를 정의한다.
같은 테이블을 읽되, 앱별로 독립적인 도메인 모델을 유지한다.

| 엔티티 (streamer) | 원본 (api) | 테이블 | 용도 |
|-------------------|-----------|--------|------|
| `CouponInfo` (읽기 전용) | `Coupon` | coupons | maxIssueCount, expiredAt 확인 |
| `CouponIssuance` (쓰기 전용) | `IssuedCoupon` | issued_coupons | 쿠폰 발급 INSERT |

> **왜 modules/jpa로 이동하지 않는가?**: 기존 commerce-api 코드의 import 경로, Repository 구현체, 테스트 전체를 변경해야 하는 영향 범위가 크다. 멘토링에서도 "기능 차이가 크면 분리가 바람직, 팀 내 합의에 따른 선택"이라는 피드백이 있었다. 두 앱이 같은 DB를 공유하므로 별도 엔티티로도 충분히 동작한다.

### 3.4 Consumer (commerce-streamer)

**CouponIssueConsumer** (`ORDERED_RECORD_LISTENER`, concurrency=1):
- `coupon-issue-requests` 토픽 리스닝 (단건 순차 처리)
- 파티션 키 = couponId → 같은 쿠폰 요청은 같은 파티션에서 순차 처리 → 동시성 문제 해소

**CouponIssueService** (비즈니스 로직):
- CouponInfo(읽기 전용)로 maxIssueCount, expiredAt 확인
- CouponIssuance(쓰기 전용)로 발급 INSERT

**처리 흐름**:
1. 멱등성 체크 (eventId → IdempotencyService)
2. 쿠폰 존재/만료/삭제 체크 (CouponInfo)
3. 중복 발급 체크 (coupon_id + user_id)
4. 수량 체크: 현재 발급 수 < maxIssueCount
5. 통과 시 → CouponIssuance INSERT + `coupon-issue-results` 토픽에 SUCCESS 발행
6. 실패 시 → `coupon-issue-results` 토픽에 FAILED + reason 발행
7. EventHandled 기록 + Manual ACK

**CouponIssueResultMessage** (콜백 메시지 포맷):
```kotlin
data class CouponIssueResultMessage(
    val requestId: String,
    val couponId: Long,
    val userId: Long,
    val status: String,       // SUCCESS, FAILED
    val failureReason: String?,
    val processedAt: ZonedDateTime,
)
```

### 3.5 Nice-to-Have: Redis Sorted Set 방식 (멘토링 참고)

멘토링에서 실무 대안으로 언급된 구조:
- Redis에 쿠폰 재고를 올려놓고, Sorted Set에 사용자 요청을 밀어넣어 백그라운드 순차 처리
- Redis 초당 수십만 건 처리량 활용 → Kafka보다 실시간성 높음
- API에서 재고 소진 시 즉시 요청 차단 가능
- 단, Redis-DB 간 불일치 보완 로직 필요 (Redis 차감 후 DB 롤백 시)

우리 프로젝트에서는 학습 목적으로 Kafka 방식을 채택하되, 이 대안은 블로그에서 트레이드오프로 다룰 수 있다.

### 3.6 커밋 전략

```
feat: Kafka 기반 선착순 쿠폰 발급 구현
test: 선착순 쿠폰 발급 테스트 추가 (동시성 테스트 포함)
```

---

## 전체 커밋 순서

| # | 커밋 메시지 | 내용 |
|---|-----------|------|
| 1 | `feat: ApplicationEvent 기반 도메인 이벤트 분리` | 이벤트 클래스, 핸들러, LikeService/OrderFacade/PaymentFacade 수정, UserActivityLog |
| 2 | `test: ApplicationEvent 도메인 이벤트 테스트 추가` | 단위/통합/E2E 테스트 |
| 3 | `feat: Transactional Outbox Pattern 및 Kafka Producer 구현` | OutboxEvent, OutboxEventPublisher, Kafka 설정 변경 |
| 4 | `feat: Kafka Consumer 및 메트릭 집계 구현` | EventHandled, ProductMetrics, Consumer들, commerce-streamer 확장 |
| 5 | `test: Kafka 이벤트 파이프라인 테스트 추가` | Testcontainers 기반 통합 테스트 |
| 6 | `feat: Kafka 기반 선착순 쿠폰 발급 구현` | FCFS API, CouponIssueConsumer, CouponIssueRequest |
| 7 | `test: 선착순 쿠폰 발급 테스트 추가` | 동시성 테스트, 수량 초과/중복 방지 검증 |

---

## 주요 수정 대상 파일

### commerce-api
- `application/like/LikeService.kt` — likeCount 직접 호출 제거, 이벤트 발행 + outboxService.save()
- `application/order/OrderFacade.kt` — OrderCreatedEvent 발행 + outboxService.save()
- `application/payment/PaymentFacade.kt` — Payment 이벤트 발행 + outboxService.save()
- `application/product/ProductService.kt` — ProductViewedEvent 발행
- `domain/coupon/Coupon.kt` — maxIssueCount 필드 추가
- `build.gradle.kts` — modules/kafka 의존성 추가
- 새로 추가: `application/coupon/FcfsCouponService.kt` — 선착순 쿠폰 발급 요청/상태 조회
- 새로 추가: `application/outbox/OutboxService.kt` — Outbox INSERT 헬퍼
- 새로 추가: `domain/coupon/CouponIssueRequest.kt` — 발급 요청 상태 추적
- 새로 추가: `interfaces/api/coupon/CouponFcfsV1Controller.kt` — FCFS API 엔드포인트
- 새로 추가: `interfaces/consumer/CouponIssueResultConsumer.kt` — 콜백 토픽 소비

### modules/kafka
- `kafka.yml` — acks=all, retries=MAX, delivery.timeout.ms=120s, linger.ms=5
- `KafkaConfig.kt` — RECORD_LISTENER(concurrency=3), ORDERED_RECORD_LISTENER(concurrency=1), BATCH_LISTENER
- `KafkaTopicConfig.kt` — 토픽 4개 생성 (파티션 3, replicas 1)
- 새로 추가: `event/KafkaEventMessage.kt`, `event/KafkaTopics.kt`, `event/CouponIssueResultMessage.kt`

### commerce-streamer
- `domain/coupon/` — CouponInfo(읽기 전용), CouponIssuance(쓰기 전용), CouponIssuanceRepository
- `domain/event/` — EventHandled, EventHandledRepository
- `domain/metrics/` — ProductMetrics, ProductMetricsRepository
- `application/coupon/CouponIssueService.kt` — 쿠폰 발급 비즈니스 로직
- `application/event/IdempotencyService.kt` — 멱등 처리
- `application/metrics/MetricsAggregationService.kt` — ProductMetrics upsert
- `interfaces/consumer/` — CatalogEventConsumer, OrderEventConsumer, CouponIssueConsumer

---

## Step 4 — Consumer 배치 처리 (Nice-to-Have)

### 4.1 배경

현재 메트릭 집계 Consumer(CatalogEventConsumer, OrderEventConsumer)가 RECORD_LISTENER로 **건별 처리**한다. 이벤트가 대량으로 쌓이면 한 건씩 DB upsert하는 것이 비효율적이다.

### 4.2 변경 내용

- CatalogEventConsumer, OrderEventConsumer를 **BATCH_LISTENER로 전환**
- 배치 단위로 메시지를 모아서 **벌크 upsert** 처리
- 멱등 처리도 배치 단위로 수행 (eventId 목록으로 일괄 체크)

### 4.3 주의사항

- 배치 내 일부 메시지 처리 실패 시 → 배치 전체를 재처리해야 함 (멱등 처리로 이미 성공한 건은 skip)
- 쿠폰 발급 Consumer는 순서 보장 필수이므로 **ORDERED_RECORD_LISTENER 유지** (배치 전환하지 않음)

### 4.4 커밋 전략

```
refactor: 메트릭 집계 Consumer를 배치 처리로 전환
```

---

## Step 5 — DLQ (Dead Letter Queue) 구성 (Nice-to-Have)

### 5.1 배경

현재 Consumer 처리 실패 시 로그만 남기고 ACK을 안 하는 방식이다. 같은 메시지가 반복 실패하면 **해당 파티션이 막혀서** 뒤의 정상 메시지도 처리되지 않는다.

### 5.2 변경 내용

- 처리 실패 메시지를 **DLQ 토픽으로 격리** (예: `catalog-events.DLQ`)
- N회 재시도 후에도 실패하면 DLQ로 이동 + 원본 메시지 ACK
- DLQ 메시지는 운영자가 확인 후 수동 재처리 또는 폐기

### 5.3 구현 방식

Spring Kafka의 `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` 활용:

```kotlin
@Bean
fun errorHandler(kafkaTemplate: KafkaTemplate<Any, Any>): DefaultErrorHandler {
    val recoverer = DeadLetterPublishingRecoverer(kafkaTemplate)
    return DefaultErrorHandler(recoverer, FixedBackOff(1000L, 3L))  // 1초 간격, 3회 재시도
}
```

### 5.4 DLQ 토픽 설계

| 원본 토픽 | DLQ 토픽 | 용도 |
|----------|---------|------|
| catalog-events | catalog-events.DLQ | 메트릭 집계 실패 메시지 격리 |
| order-events | order-events.DLQ | 주문 집계 실패 메시지 격리 |
| coupon-issue-requests | coupon-issue-requests.DLQ | 쿠폰 발급 실패 메시지 격리 |

### 5.5 커밋 전략

```
feat: Consumer DLQ 구성
```

---

---

## Step 6 — E2E Kafka 통합 테스트

### 6.1 배경

현재 단위 테스트(Mock)와 부분 통합 테스트(Outbox → Kafka SENT)만 있다. API → Kafka → Consumer → 콜백 → polling의 **전체 흐름을 검증하는 E2E 테스트**가 없다.

### 6.2 테스트 내용

| 테스트 | 검증 흐름 |
|--------|----------|
| 좋아요 E2E | POST /likes → Outbox PENDING → Kafka 발행 → CatalogEventConsumer → ProductMetrics.likeCount 증가 |
| 선착순 쿠폰 E2E | POST /fcfs-issue → coupon-issue-requests → CouponIssueConsumer → coupon-issue-results → CouponIssueRequest SUCCESS → GET /status |
| 선착순 동시성 E2E | 100명 동시 요청 → 100장 한정 쿠폰 → 초과 발급 없음 검증 |

### 6.3 환경

- 로컬 Docker Compose (Kafka + MySQL) 기반
- commerce-api, commerce-streamer 모두 @SpringBootTest로 띄우거나, 별도 프로세스로 실행 후 HTTP 호출

### 6.4 커밋 전략

```
test: E2E Kafka 통합 테스트 추가
```

---

## Step 7 — Outbox 정리 스케줄러

### 7.1 배경

Outbox 테이블에 SENT/FAILED 상태의 레코드가 계속 쌓인다. 운영 시 테이블 크기가 무한정 증가하므로 오래된 레코드를 정리하는 스케줄러가 필요하다.

### 7.2 변경 내용

- `OutboxEventCleaner` (@Scheduled) — SENT 상태이고 N일(예: 7일) 이상 지난 레코드 삭제
- FAILED 상태 레코드는 삭제하지 않고 운영자가 확인 후 수동 처리

### 7.3 구현

```kotlin
@Scheduled(cron = "0 0 3 * * *")  // 매일 새벽 3시
@Transactional
fun cleanOldSentEvents() {
    val threshold = ZonedDateTime.now().minusDays(7)
    outboxEventRepository.deleteAllSentBefore(threshold)
}
```

### 7.4 커밋 전략

```
feat: Outbox 정리 스케줄러 구현
```

---

## Step 8 — Kafka 설정 검증 테스트 (블로그용)

### 8.1 배경

블로그에서 "이 설정을 안 하면 이런 문제가 생긴다"고 주장하려면 실제 테스트로 검증해야 한다. 시나리오 설명만으로는 예측일 뿐이다.

### 8.2 테스트 내용

| # | 테스트 | 방식 | 검증 |
|---|--------|------|------|
| 1 | 멱등 처리 검증 | 자동화 (@SpringBootTest) | 같은 eventId 2번 발행 → likeCount가 1만 증가 |
| 2 | auto-commit 유실 검증 | 수동 (kafka-ui) | auto-commit=true에서 처리 중 앱 종료 → offset 커밋됨 → 재기동 시 메시지 유실 확인 |
| 3 | manual ACK 안전 검증 | 수동 (kafka-ui) | manual ACK에서 같은 시나리오 → offset 미커밋 → 재기동 시 재수신 확인 |

### 8.3 멱등 처리 테스트 (자동화)

commerce-streamer에서 실제 Kafka 메시지를 발행하고, CatalogEventConsumer가 수신하여 ProductMetrics를 업데이트하는 전체 흐름 검증.

- 정상 발행 → ProductMetrics.likeCount=1 확인
- 같은 eventId 중복 발행 → event_handled에서 skip → likeCount=1 유지 확인

### 8.4 auto-commit vs manual ACK 테스트 (수동)

1. 테스트용 Consumer를 auto-commit=true로 설정하여 앱 기동
2. 메시지 10건 발행
3. Consumer가 처리 중일 때 앱 강제 종료
4. kafka-ui에서 Consumer Group의 committed offset 확인
5. 앱 재기동 후 나머지 메시지 수신 여부 확인
6. manual ACK으로 같은 시나리오 반복 → offset 차이 비교

### 8.5 커밋 전략

```
test: Kafka 설정 검증 테스트 추가 (멱등 처리, 블로그용)
```

---

## 한계와 추후 개선 (구현 범위 밖)

| # | 항목 | 현재 상태 | 개선 방향 |
|---|------|----------|----------|
| 1 | 파티션 키 병목 | couponId 키 → 인기 쿠폰 병목 | userId 키 + DB unique 또는 Redis Sorted Set. 키 변경은 간단하지만 동시성 제어를 DB 락으로 변경해야 함 |
| 2 | @Async 스레드 풀 한계 | Step 1에서 ThreadPoolTaskExecutor 제한 | Step 2 Kafka 전환으로 해소. Step 1 단독 운영 시 대량 조회에서 큐 포화 가능 |

---

## 검증 방법

1. **Step 1**: 좋아요 API 호출 → likeCount가 비동기로 업데이트되는지 확인, UserActivityLog 저장 확인
2. **Step 2**: 좋아요/주문 → Outbox PENDING → SENT 전환 확인, commerce-streamer에서 ProductMetrics 집계 확인, 중복 이벤트 멱등 처리 확인
3. **Step 3**: 선착순 쿠폰 발급 API → 202 응답 확인, polling으로 결과 확인, 100장 제한 쿠폰에 동시 요청 시 초과 발급 없음 확인
4. **Step 4**: 배치 Consumer로 대량 이벤트 처리 시 벌크 upsert 동작 확인
5. **Step 5**: 반복 실패 메시지가 DLQ 토픽으로 격리되는지 확인
6. **Step 6**: E2E 전체 흐름 (API → Kafka → Consumer → 콜백 → polling) 검증
7. **Step 7**: 오래된 Outbox SENT 레코드가 스케줄러로 정리되는지 확인
8. **Step 8**: 멱등 처리 자동화 테스트 + auto-commit vs manual ACK 수동 검증 (kafka-ui)
9. **전체 테스트**: `./gradlew test` (로컬 Docker Compose Kafka + MySQL)
