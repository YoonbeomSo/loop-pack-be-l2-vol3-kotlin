# Round 6: PG 연동 결제 기능 + Failure-Ready Systems 구현 계획

## Context

6주차 과제는 외부 시스템(PG)과의 결제 연동을 구현하고, **장애 전파를 방지하는 회복 전략**을 적용하는 것이 목표.
PG 시스템은 비동기 결제를 제공하며, 요청 성공률 60%, 처리 결과(성공 70% / 한도초과 20% / 잘못된 카드 10%)로 다양한 실패 시나리오가 존재한다.
Timeout, Retry, CircuitBreaker, Fallback을 조합하여 PG 장애 시에도 내부 시스템이 정상 응답하도록 보호한다.

---

## 현재 상태 (AS-IS)

| 항목 | 상태 | 비고 |
|------|------|------|
| Payment 도메인 | ❌ 미구현 | 엔티티, 서비스, API 모두 없음 |
| Order 상태 관리 | ❌ 없음 | Order에 status 필드 없음, 생성만 가능 |
| PG 연동 | ❌ 없음 | FeignClient, RestTemplate 등 미설정 |
| Resilience4j | ❌ 없음 | 의존성 미추가 |
| 외부 시스템 호출 | ❌ 없음 | 프로젝트에 외부 HTTP 호출 코드 없음 |

---

## PG Simulator 사양 (별도 제공, 구현 불필요)

- **프로젝트 위치**: `/Users/soyoonbeom/github/loopers/loopback-be-l2-kotlin-additionals`
- **포트**: 8082 (management: 8083)
- **DB**: MySQL `paymentgateway` 스키마

### API 스펙

| Method | Endpoint | Header | 설명 |
|--------|----------|--------|------|
| POST | `/api/v1/payments` | `X-USER-ID` (필수) | 비동기 결제 요청 |
| GET | `/api/v1/payments/{transactionKey}` | `X-USER-ID` (필수) | 결제 상태 확인 |
| GET | `/api/v1/payments?orderId=X` | `X-USER-ID` (필수) | 주문별 결제 조회 |

### 요청/응답 특성

```
요청 성공 확률: 60% (40%는 HTTP 500 반환)
요청 지연: 100ms ~ 500ms
처리 지연: 1s ~ 5s (비동기, 콜백으로 결과 전달)
처리 결과:
  - 성공: 70% → status=SUCCESS, reason="정상 승인되었습니다."
  - 한도 초과: 20% → status=FAILED, reason="한도초과입니다. 다른 카드를 선택해주세요."
  - 잘못된 카드: 10% → status=FAILED, reason="잘못된 카드입니다. 다른 카드를 선택해주세요."
```

### 1. 결제 요청 (POST /api/v1/payments)

**Request:**
```http
POST /api/v1/payments
X-USER-ID: {userId}     ← String 타입
Content-Type: application/json

{
  "orderId": "1351039135",         // String, 6자 이상
  "cardType": "SAMSUNG",           // enum: SAMSUNG | KB | HYUNDAI
  "cardNo": "1234-5678-9814-1451", // 형식: xxxx-xxxx-xxxx-xxxx
  "amount": 5000,                  // Long, 양수
  "callbackUrl": "http://localhost:8080/api/v1/payments/callback"  // http://localhost:8080 으로 시작해야 함
}
```

**Success Response (200):**
```json
{
  "meta": { "result": "SUCCESS", "errorCode": null, "message": null },
  "data": {
    "transactionKey": "20260316:TR:abc123",   // 형식: yyyyMMdd:TR:xxxxxx
    "status": "PENDING",
    "reason": null
  }
}
```

**Error Response (500 — 40% 확률):**
```json
{
  "meta": { "result": "FAIL", "errorCode": "Internal Server Error", "message": "현재 서버가 불안정합니다. 잠시 후 다시 시도해주세요." },
  "data": null
}
```

### 2. 결제 상태 확인 (GET /api/v1/payments/{transactionKey})

**Success Response (200):**
```json
{
  "meta": { "result": "SUCCESS", "errorCode": null, "message": null },
  "data": {
    "transactionKey": "20260316:TR:abc123",
    "orderId": "1351039135",
    "cardType": "SAMSUNG",
    "cardNo": "1234-5678-9814-1451",
    "amount": 5000,
    "status": "PENDING" | "SUCCESS" | "FAILED",
    "reason": "정상 승인되었습니다." | "한도초과입니다. ..." | null
  }
}
```

### 3. 주문별 결제 조회 (GET /api/v1/payments?orderId=X)

**Success Response (200):**
```json
{
  "meta": { "result": "SUCCESS", "errorCode": null, "message": null },
  "data": {
    "orderId": "1351039135",
    "transactions": [
      {
        "transactionKey": "20260316:TR:abc123",
        "status": "SUCCESS",
        "reason": "정상 승인되었습니다."
      }
    ]
  }
}
```

### 4. 콜백 (PG → 우리 서버)

PG가 결제 처리 완료 후 `callbackUrl`로 POST 요청:

```json
{
  "transactionKey": "20260316:TR:abc123",
  "orderId": "1351039135",
  "cardType": "SAMSUNG",
  "cardNo": "1234-5678-9814-1451",
  "amount": 5000,
  "status": "SUCCESS" | "FAILED",
  "reason": "정상 승인되었습니다." | "한도초과입니다. ..." | "잘못된 카드입니다. ..."
}
```

> 콜백 실패 시 PG는 재시도하지 않음 (로그만 남김)

### PG 상태값 정리

| 상태 | 의미 |
|------|------|
| `PENDING` | 결제 요청 접수됨, 아직 처리 중 |
| `SUCCESS` | 결제 승인 완료 |
| `FAILED` | 결제 거부 (한도초과/잘못된 카드) |

---

## 비동기 결제 흐름 설계

```
[Client] → POST /api/v1/payments { orderId, cardType, cardNo }
    ↓
[PaymentV1Controller] → 사용자 인증
    ↓
[PaymentFacade.requestPayment()]
    ├─ 1. Order 검증 (소유자, status=PENDING)
    ├─ 2. 중복 결제 검증 (해당 orderId에 이미 Payment 존재하면 CONFLICT)
    ├─ 3. Payment 생성 (status=REQUESTED)
    ├─ 4. PgPaymentClient로 PG 호출 (CircuitBreaker + Retry 적용)
    │     ← PG 응답: { transactionKey: "yyyyMMdd:TR:xxxxxx", status: "PENDING" }
    ├─ 5. payment.transactionKey 저장
    └─ 6. 응답: "결제 요청이 접수되었습니다"

    ... PG 내부 처리 (1~5초) ...

[PG Simulator] → POST callbackUrl { transactionKey, orderId, status, reason, ... }
    ↓
[PaymentCallbackV1Controller] → [PaymentFacade.handleCallback()]
    ├─ Payment 상태 → PAID / FAILED  (status == "SUCCESS" → PAID, "FAILED" → FAILED)
    └─ Order 상태 → PAID / FAILED

[Client] → GET /api/v1/payments?orderId=X (상태 폴링)
    ↓
[PaymentFacade.getPaymentStatus()]
    ├─ Payment가 PAID/FAILED → 결과 반환
    └─ Payment가 REQUESTED → PG GET /payments/{transactionKey} 직접 호출 → 동기화
```

---

## 장애 시나리오별 대응 전략

### 시나리오 1: PG 타임아웃 (실제로는 결제 처리됨)

```
문제: FeignClient가 3초 타임아웃으로 실패했지만, PG는 실제로 결제를 처리함
대응:
  - Payment는 REQUESTED 상태로 DB에 저장되어 있음
  - PG 콜백이 도착하면 handleCallback()에서 transactionKey로 매칭 → 정상 동기화
  - 콜백 미도착 시 → Client가 GET /payments?orderId=X 호출
    → REQUESTED 상태 감지 → PG GET /payments/{transactionKey}로 직접 조회 → 동기화
```

### 시나리오 2: PG 콜백 미도착 (PG는 콜백 실패 시 재시도하지 않음)

```
문제: PG가 결제를 처리했지만 콜백이 유실됨
대응:
  - getPaymentStatus()에서 REQUESTED 상태인 Payment 발견 시
    PG GET /payments/{transactionKey} API로 직접 상태 확인
  - status가 "SUCCESS"/"FAILED"이면 Payment + Order 상태 동기화
  - status가 아직 "PENDING"이면 "처리 중" 응답
```

### 시나리오 3: PG 완전 장애 (CircuitBreaker OPEN)

```
문제: PG 서버가 완전히 다운되어 연속 실패 발생
대응:
  - 실패율 50% 초과 시 CircuitBreaker OPEN → PG 호출 자체를 차단
  - Fallback 즉시 응답: "PG 시스템 장애로 결제 요청에 실패했습니다"
  - 10초 후 HALF_OPEN → 3건만 시험 호출 → 성공 시 CLOSED 복귀
  - 내부 시스템(주문, 상품 등)은 PG 장애와 무관하게 정상 동작
```

### 시나리오 4: 중복 결제 요청

```
문제: 사용자가 같은 주문에 대해 결제를 두 번 요청
대응:
  - requestPayment()에서 orderId로 기존 Payment 조회
  - 이미 존재하면 CONFLICT (409) 에러 반환
```

### 시나리오 5: Retry 중 성공

```
문제: 첫 번째 PG 요청이 타임아웃, 두 번째 재시도에서 성공
대응:
  - Resilience4j Retry가 자동으로 재시도 (max 3회, 500ms → 1s → 2s 간격)
  - 성공 응답 수신 시 정상 흐름 진행
  - 첫 번째 요청이 PG에서 실제 처리되었다면 중복 결제 위험
    → PG의 orderId 기반 멱등성에 의존 (PG가 동일 orderId 중복 요청 거부)
```

---

## 과제 ① Order.id를 Snowflake ID로 변경 + 상태(OrderStatus) 추가

### 주문번호 설계 — Snowflake ID로 Order.id 통합

> **설계 판단**: 별도 `orderNo` 필드를 두면, 내부에서는 id, PG에는 orderNo,
> ERP 정산에서는 또 다른 키를 쓰게 되어 시스템 간 매핑이 복잡해진다.
>
> **멘토님 피드백**: auto increment는 식별자로서 한계가 있으며,
> 스노우플레이크/TUID 등 시간 기반 분산 ID 생성기 사용을 권장.
>
> **결론**: `Order.id`(Long PK)를 **Snowflake ID**로 생성하여,
> 내부 PK = API 응답 = PG orderId = ERP 정산 키를 **하나의 값으로 통합**.

### Snowflake ID 구조

```
64비트 Long (양수만 사용 → 63비트)

┌──────────────────────┬──────────┬──────────────┐
│  타임스탬프 (41bit)    │ 머신ID   │  시퀀스       │
│  ms 단위, 약 69년     │ (10bit)  │  (12bit)     │
│                      │ 0~1023   │  0~4095/ms   │
└──────────────────────┴──────────┴──────────────┘

예시: 1742313600000001234 (19자리 Long)
```

**장점:**
- `Long` 타입 유지 → 기존 Order.id 타입 변경 불필요
- 시간순 거친 정렬 가능 → 인덱스 효율 좋음 (순차 삽입에 가까움)
- 여러 서버에서 동시 채번해도 충돌 없음 (머신ID로 구분)
- DB/Redis 의존 없이 애플리케이션에서 생성
- PG 6자 이상 조건 충족 (19자리)
- UUID보다 인덱스 성능 좋음 (Long vs String)

### 구현: SnowflakeIdGenerator

#### 신규: `infrastructure/id/SnowflakeIdGenerator.kt`

```kotlin
/**
 * Snowflake ID 생성기
 * - 64비트 Long: 1(부호) + 41(타임스탬프) + 10(머신ID) + 12(시퀀스)
 * - 커스텀 epoch 사용 (2025-01-01 기준)
 * - 단일 머신에서 ms당 4096건까지 생성 가능
 */
@Component
class SnowflakeIdGenerator(
    @Value("\${snowflake.machine-id:1}") private val machineId: Long,
) {
    companion object {
        private const val EPOCH = 1735689600000L          // 2025-01-01 00:00:00 UTC
        private const val MACHINE_ID_BITS = 10L
        private const val SEQUENCE_BITS = 12L
        private const val MAX_MACHINE_ID = (1L shl MACHINE_ID_BITS.toInt()) - 1   // 1023
        private const val MAX_SEQUENCE = (1L shl SEQUENCE_BITS.toInt()) - 1        // 4095
        private const val MACHINE_ID_SHIFT = SEQUENCE_BITS
        private const val TIMESTAMP_SHIFT = SEQUENCE_BITS + MACHINE_ID_BITS
    }

    private var sequence = 0L
    private var lastTimestamp = -1L

    init {
        require(machineId in 0..MAX_MACHINE_ID) {
            "machineId는 0~$MAX_MACHINE_ID 범위여야 합니다."
        }
    }

    @Synchronized
    fun generate(): Long {
        var timestamp = System.currentTimeMillis()

        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) and MAX_SEQUENCE
            if (sequence == 0L) {
                // 같은 ms에 4096건 초과 → 다음 ms까지 대기
                timestamp = waitNextMillis(lastTimestamp)
            }
        } else {
            sequence = 0L
        }

        lastTimestamp = timestamp

        return ((timestamp - EPOCH) shl TIMESTAMP_SHIFT.toInt()) or
            (machineId shl MACHINE_ID_SHIFT.toInt()) or
            sequence
    }

    private fun waitNextMillis(lastTimestamp: Long): Long {
        var timestamp = System.currentTimeMillis()
        while (timestamp <= lastTimestamp) {
            timestamp = System.currentTimeMillis()
        }
        return timestamp
    }
}
```

#### 설정: `application.yml`

```yaml
snowflake:
  machine-id: ${SNOWFLAKE_MACHINE_ID:1}   # 서버별로 다른 값 (0~1023)
```

> 단일 서버 환경에서는 기본값 1로 충분.
> 서버 다중화 시 각 인스턴스에 고유한 machine-id를 환경변수로 주입.

### 변경 범위

#### 수정: `OrderBaseEntity.kt` — auto increment 제거

```kotlin
@MappedSuperclass
abstract class OrderBaseEntity {
    @Id
    // @GeneratedValue 제거 — SnowflakeIdGenerator에서 직접 id 할당
    val id: Long = 0

    @Column(name = "created_at", nullable = false, updatable = false)
    lateinit var createdAt: ZonedDateTime
}
```

#### 수정: `domain/order/Order.kt` — 생성자에서 id를 받도록 변경

```kotlin
class Order(
    id: Long,             // SnowflakeIdGenerator에서 생성한 id를 전달받음
    userId: Long,
    couponId: Long? = null,
) : OrderBaseEntity() {
    // id는 OrderBaseEntity에서 할당
}
```

#### 수정: `application/order/OrderService.kt` — 주문 생성 시 Snowflake ID 채번

```kotlin
@Component
class OrderService(
    private val orderRepository: OrderRepository,
    private val snowflakeIdGenerator: SnowflakeIdGenerator,    // 추가
) {
    @Transactional
    fun createOrder(userId: Long, items: List<OrderItemCommand>, couponId: Long? = null): Order {
        val orderId = snowflakeIdGenerator.generate()   // Snowflake ID 생성
        val order = Order(id = orderId, userId = userId, couponId = couponId)
        // ... 기존 로직 (addItem 등)
        return orderRepository.save(order)
    }
}
```

#### 수정: Payment 관련 — order.id를 그대로 PG에 전달

```kotlin
// PaymentFacade에서 PG 요청 시
val pgRequest = PgPaymentRequest(
    orderId = order.id.toString(),    // Snowflake ID → String으로 변환 (19자리)
    ...
)

// PG 콜백 수신 시
// callbackRequest.orderId == order.id.toString() → 매칭
```

#### 기존 API·DTO·테스트 — 변경 최소

- `OrderInfo.id: Long` → 그대로 (값만 달라짐)
- `OrderV1Dto` 응답 DTOs → `orderId: Long` / `id: Long` 그대로 (값만 달라짐)
- `GET /api/v1/orders/{orderId}` → Long PathVariable 그대로
- 기존 테스트: id 값 형태만 달라지므로 **하드코딩된 id 검증만 수정**

### Snowflake vs 이전 방식 비교

| 항목 | auto increment | yyyyMMdd + 시퀀스 | **Snowflake (채택)** |
|------|---------------|------------------|---------------------|
| 추가 인프라 | 없음 | Redis or DB 시퀀스 | **없음** |
| 다중 서버 | 충돌 없음 (DB 위임) | 채번 서버 필요 | **머신ID로 충돌 방지** |
| 시간순 정렬 | O (순차) | O (날짜 포함) | **O (거친 정렬)** |
| PG 6자 이상 | X (1, 2, 3...) | O (14자리) | **O (19자리)** |
| Long 타입 유지 | O | △ (20자리 초과 위험) | **O (63비트 이내)** |
| 비즈니스 의미 | 없음 | 날짜 포함 | **시간 추출 가능** |

---

### 신규 파일: `domain/order/OrderStatus.kt`

```kotlin
enum class OrderStatus {
    PENDING,    // 주문 생성 완료, 결제 전
    PAID,       // 결제 완료
    FAILED,     // 결제 실패
}
```

> CouponType, IssuedCouponStatus 등 기존 패턴처럼 별도 파일로 분리

### 수정: `domain/order/Order.kt`

```kotlin
// 추가할 필드
@Column(name = "status", nullable = false)
@Enumerated(EnumType.STRING)
var status: OrderStatus = OrderStatus.PENDING
    private set

// 추가할 메서드
fun markPaid() {
    if (status != OrderStatus.PENDING) {
        throw CoreException(ErrorType.BAD_REQUEST, "PENDING 상태에서만 결제 완료 처리가 가능합니다.")
    }
    status = OrderStatus.PAID
}

fun markFailed() {
    if (status != OrderStatus.PENDING) {
        throw CoreException(ErrorType.BAD_REQUEST, "PENDING 상태에서만 결제 실패 처리가 가능합니다.")
    }
    status = OrderStatus.FAILED
}

fun validatePayable() {
    if (status != OrderStatus.PENDING) {
        throw CoreException(ErrorType.BAD_REQUEST, "결제 가능한 상태가 아닙니다. 현재 상태: $status")
    }
}

fun validateOwner(userId: Long) {
    if (this.userId != userId) {
        throw CoreException(ErrorType.FORBIDDEN, "해당 주문에 대한 권한이 없습니다.")
    }
}
```

### 수정: `application/order/OrderInfo.kt`

- `status: OrderStatus` 필드 추가
- `from(order)` 매핑에 status 포함

### 수정: `interfaces/api/order/OrderV1Dto.kt`

- Response DTO에 `status: String` 필드 추가

---

## 과제 ② Payment 도메인 구현

### 2-1. 도메인 레이어

#### 신규: `domain/payment/PaymentStatus.kt`

> 멘토링 피드백: "결제 요청 전에 레코드를 저장하고 상태를 세분화하라"
> INITIATED(생성)와 REQUESTED(PG 응답 수신)를 분리하여
> "PG에 요청은 보냈지만 응답은 못 받은 상태"를 추적 가능하게 함

```kotlin
enum class PaymentStatus {
    INITIATED,  // Payment 생성 완료, PG 요청 전 (TX1에서 커밋됨)
    REQUESTED,  // PG에 요청 전송 완료, transactionKey 수신 (TX2에서 커밋됨)
    PAID,       // PG 승인 완료 (콜백 or 폴링으로 확인)
    FAILED,     // PG 거절 or 최종 실패 확정
}
```

**상태 전이 흐름:**
```
INITIATED → REQUESTED : PG가 200 OK 응답 (transactionKey 수신)
INITIATED → FAILED    : PG 호출 자체 실패 (CB Open, 500 에러 등)
REQUESTED → PAID      : PG 콜백 status=SUCCESS or 폴링으로 확인
REQUESTED → FAILED    : PG 콜백 status=FAILED or 폴링으로 확인
```

#### 신규: `domain/payment/Payment.kt`

```kotlin
@Entity
@Table(
    name = "payments",
    indexes = [
        Index(name = "idx_payments_order_id", columnList = "order_id"),
        Index(name = "idx_payments_transaction_key", columnList = "transaction_key"),
    ],
)
class Payment(
    orderId: Long,
    userId: Long,
    amount: BigDecimal,
    cardType: String,
    cardNo: String,
) : BaseEntity() {

    @Column(name = "order_id", nullable = false)
    val orderId: Long = orderId             // Order.id = 비즈니스 주문번호 (20260316000001)

    @Column(name = "user_id", nullable = false)
    val userId: Long = userId

    @Column(name = "amount", nullable = false, precision = 15, scale = 2)
    val amount: BigDecimal = amount

    @Column(name = "card_type", nullable = false, length = 20)
    val cardType: String = cardType

    @Column(name = "card_no", nullable = false, length = 20)
    val cardNo: String = cardNo   // 마스킹 저장 (예: "****1451")

    @Column(name = "transaction_key", length = 50)
    var transactionKey: String? = null        // PG의 transactionKey (형식: yyyyMMdd:TR:xxxxxx)
        private set

    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    var status: PaymentStatus = PaymentStatus.INITIATED
        private set

    @Column(name = "fail_reason", length = 200)
    var failReason: String? = null
        private set

    fun markRequested(transactionKey: String) {
        if (status != PaymentStatus.INITIATED) {
            throw CoreException(ErrorType.BAD_REQUEST, "INITIATED 상태에서만 PG 요청 완료 처리가 가능합니다.")
        }
        this.transactionKey = transactionKey
        status = PaymentStatus.REQUESTED
    }

    fun markPaid() {
        if (status != PaymentStatus.REQUESTED) {
            throw CoreException(ErrorType.BAD_REQUEST, "REQUESTED 상태에서만 결제 완료 처리가 가능합니다.")
        }
        status = PaymentStatus.PAID
    }

    fun markFailed(reason: String?) {
        if (status !in listOf(PaymentStatus.INITIATED, PaymentStatus.REQUESTED)) {
            throw CoreException(ErrorType.BAD_REQUEST, "INITIATED 또는 REQUESTED 상태에서만 실패 처리가 가능합니다.")
        }
        status = PaymentStatus.FAILED
        failReason = reason
    }

    companion object {
        fun maskCardNo(cardNo: String): String {
            // "1234-5678-9814-1451" → "****-****-****-1451"
            return cardNo.split("-").let { parts ->
                if (parts.size == 4) "****-****-****-${parts.last()}"
                else "****${cardNo.takeLast(4)}"
            }
        }
    }
}
```

#### 신규: `domain/payment/PaymentRepository.kt`

```kotlin
interface PaymentRepository {
    fun save(payment: Payment): Payment
    fun findById(id: Long): Payment?
    fun findByOrderId(orderId: Long): Payment?
    fun findByTransactionKey(transactionKey: String): Payment?
}
```

---

### 2-2. 인프라 레이어

#### 신규: `infrastructure/payment/PaymentJpaRepository.kt`

```kotlin
interface PaymentJpaRepository : JpaRepository<Payment, Long> {
    fun findByOrderId(orderId: Long): Payment?
    fun findByTransactionKey(transactionKey: String): Payment?
}
```

#### 신규: `infrastructure/payment/PaymentRepositoryImpl.kt`

```kotlin
@Repository
class PaymentRepositoryImpl(
    private val paymentJpaRepository: PaymentJpaRepository,
) : PaymentRepository {
    // 기존 패턴(OrderRepositoryImpl 등)과 동일하게 구현
}
```

---

### 2-3. PG 클라이언트 (FeignClient)

#### 신규: `infrastructure/pg/PgClient.kt`

```kotlin
@FeignClient(
    name = "pgClient",
    url = "\${pg.base-url}",
    configuration = [PgClientConfig::class],
)
interface PgClient {

    @PostMapping("/api/v1/payments")
    fun requestPayment(
        @RequestHeader("X-USER-ID") userId: String,        // String 타입 주의
        @RequestBody request: PgPaymentRequest,
    ): PgApiResponse<PgPaymentResponse>                     // ApiResponse 래퍼

    @GetMapping("/api/v1/payments/{transactionKey}")
    fun getPaymentStatus(
        @RequestHeader("X-USER-ID") userId: String,
        @PathVariable transactionKey: String,
    ): PgApiResponse<PgTransactionDetailResponse>

    @GetMapping("/api/v1/payments")
    fun getPaymentsByOrderId(
        @RequestHeader("X-USER-ID") userId: String,
        @RequestParam orderId: String,
    ): PgApiResponse<PgOrderTransactionsResponse>
}
```

#### 신규: `infrastructure/pg/PgClientConfig.kt`

```kotlin
@Configuration
class PgClientConfig {
    @Bean
    fun feignOptions(): Request.Options {
        return Request.Options(
            1000, TimeUnit.MILLISECONDS,  // connectTimeout: 1초
            3000, TimeUnit.MILLISECONDS,  // readTimeout: 3초
            true,                         // followRedirects
        )
    }
}
```

> PG 요청 지연이 100~500ms이므로 readTimeout 3초는 충분한 여유.
> 요청 실패(40%) + 타임아웃 시 Retry가 재시도.

#### 신규: `infrastructure/pg/PgPaymentRequest.kt`

```kotlin
data class PgPaymentRequest(
    val orderId: String,
    val cardType: String,       // "SAMSUNG" | "KB" | "HYUNDAI"
    val cardNo: String,         // 형식: xxxx-xxxx-xxxx-xxxx
    val amount: Long,           // Long 타입 (BigDecimal → Long 변환 필요)
    val callbackUrl: String,    // http://localhost:8080 으로 시작해야 함
)
```

#### 신규: `infrastructure/pg/PgApiResponse.kt`

> PG 시뮬레이터의 응답은 `{ meta: { result, errorCode, message }, data: T }` 래퍼로 감싸져 있음

```kotlin
data class PgApiResponse<T>(
    val meta: PgMeta,
    val data: T?,
) {
    data class PgMeta(
        val result: String,       // "SUCCESS" | "FAIL"
        val errorCode: String?,
        val message: String?,
    )

    fun isSuccess(): Boolean = meta.result == "SUCCESS"
}
```

#### 신규: `infrastructure/pg/PgPaymentResponse.kt`

> POST /api/v1/payments 의 data 부분

```kotlin
data class PgPaymentResponse(
    val transactionKey: String,   // PG 발급 거래 키 (형식: yyyyMMdd:TR:xxxxxx)
    val status: String,           // "PENDING"
    val reason: String?,
)
```

#### 신규: `infrastructure/pg/PgTransactionDetailResponse.kt`

> GET /api/v1/payments/{transactionKey} 의 data 부분

```kotlin
data class PgTransactionDetailResponse(
    val transactionKey: String,
    val orderId: String,
    val cardType: String,
    val cardNo: String,
    val amount: Long,
    val status: String,           // "PENDING" | "SUCCESS" | "FAILED"
    val reason: String?,
) {
    fun isTerminal(): Boolean = status == "SUCCESS" || status == "FAILED"
    fun isSuccess(): Boolean = status == "SUCCESS"
}
```

#### 신규: `infrastructure/pg/PgOrderTransactionsResponse.kt`

> GET /api/v1/payments?orderId=X 의 data 부분

```kotlin
data class PgOrderTransactionsResponse(
    val orderId: String,
    val transactions: List<PgTransactionSummary>,
)

data class PgTransactionSummary(
    val transactionKey: String,
    val status: String,           // "PENDING" | "SUCCESS" | "FAILED"
    val reason: String?,
)
```

#### 신규: `infrastructure/pg/PgCallbackRequest.kt`

> PG가 콜백으로 보내는 요청 형식

```kotlin
data class PgCallbackRequest(
    val transactionKey: String,
    val orderId: String,
    val cardType: String,
    val cardNo: String,
    val amount: Long,
    val status: String,           // "SUCCESS" | "FAILED"
    val reason: String?,
)
```

---

### 2-4. 애플리케이션 레이어

#### 신규: `application/payment/PaymentCriteria.kt`

```kotlin
data class PaymentCriteria(
    val orderId: Long,
    val cardType: String,
    val cardNo: String,
)
```

#### 신규: `application/payment/PaymentInfo.kt`

```kotlin
data class PaymentInfo(
    val id: Long,
    val orderId: Long,
    val amount: BigDecimal,
    val cardType: String,
    val cardNo: String,
    val transactionKey: String?,
    val status: PaymentStatus,
    val failReason: String?,
    val createdAt: LocalDateTime,
) {
    companion object {
        fun from(payment: Payment): PaymentInfo { ... }
    }
}
```

#### 신규: `application/payment/PaymentService.kt`

```kotlin
@Component
class PaymentService(
    private val paymentRepository: PaymentRepository,
) {
    @Transactional
    fun createPayment(userId: Long, orderId: Long, amount: BigDecimal, cardType: String, cardNo: String): Payment {
        val maskedCardNo = Payment.maskCardNo(cardNo)
        val payment = Payment(orderId, userId, amount, cardType, maskedCardNo)
        return paymentRepository.save(payment)
    }

    @Transactional(readOnly = true)
    fun getPaymentByOrderId(orderId: Long): Payment? {
        return paymentRepository.findByOrderId(orderId)
    }

    @Transactional(readOnly = true)
    fun getPaymentByTransactionKey(transactionKey: String): Payment? {
        return paymentRepository.findByTransactionKey(transactionKey)
    }
}
```

#### 신규: `application/payment/PgPaymentClient.kt` — Resilience4j 래퍼

```kotlin
@Component
class PgPaymentClient(
    private val pgClient: PgClient,
) {
    @CircuitBreaker(name = "pgCircuit", fallbackMethod = "requestPaymentFallback")
    @Retry(name = "pgRetry")
    fun requestPayment(userId: String, request: PgPaymentRequest): PgApiResponse<PgPaymentResponse> {
        return pgClient.requestPayment(userId, request)
    }

    // Fallback: CircuitBreaker OPEN 또는 Retry 모두 실패 시 호출
    // 멘토링 피드백: Retry 소진 vs CB Open을 로그에서 구분해야 장애 원인 파악 가능
    fun requestPaymentFallback(userId: String, request: PgPaymentRequest, t: Throwable): PgApiResponse<PgPaymentResponse> {
        val failureType = when (t) {
            is CallNotPermittedException -> "CIRCUIT_OPEN"       // PG에 요청 자체가 안 감
            is SocketTimeoutException -> "TIMEOUT_EXHAUSTED"     // PG에 요청 갔지만 응답 없음
            else -> "UNKNOWN"
        }
        log.warn("PG 결제 요청 fallback: type=$failureType, orderId=${request.orderId}", t)
        throw CoreException(ErrorType.SERVICE_UNAVAILABLE, "PG 시스템 장애로 결제 요청에 실패했습니다. 잠시 후 다시 시도해주세요.")
    }

    @CircuitBreaker(name = "pgCircuit", fallbackMethod = "getPaymentStatusFallback")
    fun getPaymentStatus(userId: String, transactionKey: String): PgApiResponse<PgTransactionDetailResponse> {
        return pgClient.getPaymentStatus(userId, transactionKey)
    }

    // Fallback: PG 상태 조회 실패 시 "아직 처리 중" 응답
    fun getPaymentStatusFallback(userId: String, transactionKey: String, t: Throwable): PgApiResponse<PgTransactionDetailResponse> {
        return PgApiResponse(
            meta = PgApiResponse.PgMeta(result = "SUCCESS", errorCode = null, message = null),
            data = PgTransactionDetailResponse(
                transactionKey = transactionKey,
                orderId = "",
                cardType = "",
                cardNo = "",
                amount = 0,
                status = "PENDING",
                reason = null,
            ),
        )
    }
}
```

> **Resilience4j 어노테이션 적용 순서**: `Retry(CircuitBreaker(fn))`
> → Retry가 먼저 재시도하고, 모든 재시도 실패 시 1건의 실패로 CircuitBreaker에 기록
> → CircuitBreaker의 실패 카운트가 과도하게 증가하지 않음

#### 신규: `application/payment/PaymentFacade.kt`

```kotlin
@Component
class PaymentFacade(
    private val paymentService: PaymentService,
    private val orderService: OrderService,
    private val pgPaymentClient: PgPaymentClient,
    @Value("\${pg.callback-url}") private val callbackUrl: String,
) {
    /**
     * 결제 요청 — 트랜잭션을 2단계로 분리하여 PG 타임아웃 시에도 Payment 레코드 유지.
     *
     * [트랜잭션 1] Order 검증 + Payment 생성 (REQUESTED) → 커밋
     *   → DB에 Payment가 확정되므로, PG 호출이 실패해도 콜백/폴링으로 복구 가능
     *
     * [트랜잭션 2] PG 호출 + transactionKey 저장 → 커밋
     *   → PG 호출 실패 시 Payment는 REQUESTED 상태로 남음 (추후 복구)
     */
    fun requestPayment(userId: Long, criteria: PaymentCriteria): PaymentInfo {
        // ── 트랜잭션 1: Order 검증 + Payment 생성 ──
        val payment = createPaymentInNewTransaction(userId, criteria)

        // ── 트랜잭션 2: PG 호출 + transactionKey 저장 ──
        return callPgAndUpdatePayment(userId, payment, criteria)
    }

    @Transactional
    fun createPaymentInNewTransaction(userId: Long, criteria: PaymentCriteria): Payment {
        val order = orderService.getOrder(criteria.orderId)
        order.validateOwner(userId)
        order.validatePayable()

        paymentService.getPaymentByOrderId(criteria.orderId)?.let {
            throw CoreException(ErrorType.CONFLICT, "이미 결제가 진행 중인 주문입니다.")
        }

        return paymentService.createPayment(
            userId, order.id, order.totalAmount, criteria.cardType, criteria.cardNo,
        )
    }

    @Transactional
    fun callPgAndUpdatePayment(userId: Long, payment: Payment, criteria: PaymentCriteria): PaymentInfo {
        val pgRequest = PgPaymentRequest(
            orderId = payment.orderId.toString(),   // Order.id를 그대로 PG에 전달 (20260316000001)
            cardType = criteria.cardType,
            cardNo = criteria.cardNo,
            amount = payment.amount.toLong(),
            callbackUrl = callbackUrl,
        )
        val pgResponse = pgPaymentClient.requestPayment(userId.toString(), pgRequest)

        if (!pgResponse.isSuccess() || pgResponse.data == null) {
            payment.markFailed("PG 결제 요청 실패")
            throw CoreException(ErrorType.INTERNAL_ERROR, "PG 결제 요청이 실패했습니다.")
        }
        payment.markRequested(pgResponse.data.transactionKey)  // INITIATED → REQUESTED

        return PaymentInfo.from(payment)
    }

    @Transactional
    fun handleCallback(callbackRequest: PgCallbackRequest) {
        val payment = paymentService.getPaymentByTransactionKey(callbackRequest.transactionKey)
            ?: throw CoreException(ErrorType.NOT_FOUND, "결제 정보를 찾을 수 없습니다.")

        // 이미 처리된 결제는 무시 (멱등성)
        if (payment.status != PaymentStatus.REQUESTED) return

        val isSuccess = callbackRequest.status == "SUCCESS"
        if (isSuccess) payment.markPaid() else payment.markFailed(callbackRequest.reason)

        val order = orderService.getOrder(payment.orderId)
        if (isSuccess) order.markPaid() else order.markFailed()
    }

    @Transactional
    fun getPaymentStatus(userId: Long, orderId: Long): PaymentInfo {
        val order = orderService.getOrder(orderId)
        order.validateOwner(userId)

        val payment = paymentService.getPaymentByOrderId(orderId)
            ?: throw CoreException(ErrorType.NOT_FOUND, "결제 정보를 찾을 수 없습니다.")

        // REQUESTED 상태면 PG에 직접 확인하여 동기화 (INITIATED는 PG에 요청 자체가 안 갔으므로 조회 불필요)
        if (payment.status == PaymentStatus.REQUESTED && payment.transactionKey != null) {
            val pgResponse = pgPaymentClient.getPaymentStatus(userId.toString(), payment.transactionKey!!)
            val pgDetail = pgResponse.data
            if (pgDetail != null && pgDetail.isTerminal()) {
                if (pgDetail.isSuccess()) {
                    payment.markPaid()
                    order.markPaid()
                } else {
                    payment.markFailed(pgDetail.reason)
                    order.markFailed()
                }
            }
        }

        return PaymentInfo.from(payment)
    }
}
```

> **트랜잭션 분리 설계**: requestPayment()를 2개의 트랜잭션으로 분리.
> - 트랜잭션 1: Payment 생성 (REQUESTED) → **커밋 확정** → PG 타임아웃이 발생해도 Payment 레코드 유지
> - 트랜잭션 2: PG 호출 + transactionKey 저장 → 실패 시 Payment는 REQUESTED 상태로 남음
> - 이후 콜백 또는 getPaymentStatus() 폴링에서 PG 상태를 확인하여 복구 가능
>
> 만약 트랜잭션 2에서 PG 호출 자체가 실패(500 에러)한 경우:
> - Payment는 REQUESTED + transactionKey=null 상태
> - PG에 결제가 생성되지 않았으므로 데이터 불일치 없음
> - 사용자가 재결제 시 중복 결제 검증에 걸림 → 기존 REQUESTED Payment 처리 방안 필요
>   (예: 일정 시간 경과 후 transactionKey=null인 REQUESTED는 만료 처리)

---

### 2-5. API 레이어

#### 신규: `interfaces/api/payment/PaymentV1Dto.kt`

```kotlin
class PaymentV1Dto {
    data class CreateRequest(
        @Schema(description = "주문 ID", example = "1")
        val orderId: Long,
        @Schema(description = "카드 종류", example = "SAMSUNG")
        val cardType: String,
        @Schema(description = "카드 번호", example = "1234-5678-9814-1451")
        val cardNo: String,
    ) {
        fun toCriteria() = PaymentCriteria(orderId, cardType, cardNo)
    }

    data class PaymentResponse(
        @Schema(description = "결제 ID")
        val id: Long,
        @Schema(description = "주문 ID")
        val orderId: Long,
        @Schema(description = "결제 금액")
        val amount: BigDecimal,
        @Schema(description = "카드 종류")
        val cardType: String,
        @Schema(description = "마스킹된 카드 번호")
        val cardNo: String,
        @Schema(description = "PG 거래 키", example = "20260316:TR:abc123")
        val transactionKey: String?,
        @Schema(description = "결제 상태", example = "REQUESTED")
        val status: String,
        @Schema(description = "실패 사유")
        val failReason: String?,
    ) {
        companion object {
            fun from(info: PaymentInfo) = PaymentResponse(...)
        }
    }

    // 콜백 요청은 infrastructure/pg/PgCallbackRequest 사용 (PG가 보내는 형식 그대로)
}
```

#### 신규: `interfaces/api/payment/PaymentV1ApiSpec.kt`

```kotlin
@Tag(name = "Payment V1 API", description = "결제 API")
interface PaymentV1ApiSpec {

    @Operation(summary = "결제 요청", description = "주문에 대한 PG 결제를 요청합니다")
    @ApiResponses(...)
    fun requestPayment(loginId: String, password: String, request: PaymentV1Dto.CreateRequest): ApiResponse<PaymentV1Dto.PaymentResponse>

    @Operation(summary = "결제 상태 조회", description = "주문의 결제 상태를 조회합니다")
    @ApiResponses(...)
    fun getPaymentStatus(loginId: String, password: String, orderId: Long): ApiResponse<PaymentV1Dto.PaymentResponse>
}
```

#### 신규: `interfaces/api/payment/PaymentV1Controller.kt`

```kotlin
@RestController
class PaymentV1Controller(
    private val userService: UserService,
    private val paymentFacade: PaymentFacade,
) : PaymentV1ApiSpec {

    @PostMapping("/api/v1/payments")
    override fun requestPayment(
        @RequestHeader("X-Loopers-LoginId") loginId: String,
        @RequestHeader("X-Loopers-LoginPw") password: String,
        @RequestBody request: PaymentV1Dto.CreateRequest,
    ): ApiResponse<PaymentV1Dto.PaymentResponse> {
        val user = userService.authenticate(loginId, password)
        val result = paymentFacade.requestPayment(user.id, request.toCriteria())
        return ApiResponse.success(PaymentV1Dto.PaymentResponse.from(result))
    }

    @GetMapping("/api/v1/payments")
    override fun getPaymentStatus(
        @RequestHeader("X-Loopers-LoginId") loginId: String,
        @RequestHeader("X-Loopers-LoginPw") password: String,
        @RequestParam orderId: Long,
    ): ApiResponse<PaymentV1Dto.PaymentResponse> {
        val user = userService.authenticate(loginId, password)
        val result = paymentFacade.getPaymentStatus(user.id, orderId)
        return ApiResponse.success(PaymentV1Dto.PaymentResponse.from(result))
    }
}
```

> 콜백 엔드포인트(`/callback`)는 PG 시스템이 호출하므로 사용자 인증 불필요.
> 같은 `/api/v1/payments` 리소스이므로 별도 컨트롤러로 분리하지 않고 `PaymentV1Controller`에 포함.

```kotlin
// PaymentV1Controller 안에 추가
@PostMapping("/api/v1/payments/callback")
fun handleCallback(
    @RequestBody request: PgCallbackRequest,
): ApiResponse<Unit> {
    paymentFacade.handleCallback(request)
    return ApiResponse.success()
}
```

---

## 과제 ③ Resilience4j 설정

### 의존성 추가: `apps/commerce-api/build.gradle.kts`

```kotlin
// Feign Client
implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
// Resilience4j + AOP
implementation("io.github.resilience4j:resilience4j-spring-boot3")
implementation("org.springframework.boot:spring-boot-starter-aop")
```

### 설정 추가: `application.yml`

```yaml
pg:
  base-url: ${PG_BASE_URL:http://localhost:8082}
  callback-url: ${PG_CALLBACK_URL:http://localhost:8080/api/v1/payments/callback}

resilience4j:
  circuitbreaker:
    instances:
      pgCircuit:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10           # 최근 10건 기준
        minimum-number-of-calls: 5        # 최소 5건 이상 호출 후 실패율 판단 시작
        failure-rate-threshold: 50        # 실패율 50% 초과 시 OPEN
        wait-duration-in-open-state: 10s  # OPEN 유지 시간
        permitted-number-of-calls-in-half-open-state: 3  # HALF_OPEN에서 시험 호출 수
        slow-call-duration-threshold: 3s  # 3초 이상이면 slow call로 판단
        slow-call-rate-threshold: 50      # slow call 비율 50% 초과 시 OPEN
        register-health-indicator: true   # Actuator 헬스체크 연동
  retry:
    instances:
      pgRetry:
        max-attempts: 3                   # 최대 3회 시도
        wait-duration: 500ms              # 첫 번째 대기: 500ms
        exponential-backoff-multiplier: 2 # 대기 시간 2배씩 증가 (500ms → 1s → 2s)
        retry-exceptions:                 # 재시도 대상: 타임아웃만 (500은 즉시 실패)
          - java.net.SocketTimeoutException
          - java.net.ConnectException
        ignore-exceptions:                # 재시도 제외 (비즈니스 예외, PG 500 에러)
          - com.loopers.support.error.CoreException
          - feign.FeignException
        fail-after-max-attempts: true     # 모든 재시도 실패 시 fallback 호출
```

### 설정 근거

| 설정 | 값 | 근거 |
|------|---|------|
| **Feign connectTimeout** | 1s | TCP 연결 수립에 1초면 충분, 초과 시 PG 자체 장애 |
| **Feign readTimeout** | 3s | PG 요청 지연 최대 500ms + 여유 (slow call 기준과 일치) |
| **Retry max-attempts** | 3 | 타임아웃만 재시도. 500 에러는 서버 문제이므로 재시도해도 같은 결과일 가능성 높음 |
| **Retry wait-duration** | 500ms exponential | PG 일시 과부하 시 충분한 회복 시간 부여 |
| **Retry 대상** | 타임아웃만 | SocketTimeoutException, ConnectException만 재시도. FeignException(500)은 즉시 실패 |
| **CircuitBreaker sliding-window** | 10건 | 소규모 샘플로 빠르게 장애 감지 |
| **CircuitBreaker minimum-number-of-calls** | 5건 | 최소 5건 이상 호출이 있어야 실패율 판단 시작. 이 설정이 없으면 서버 시작 직후 첫 2건이 실패했을 때 바로 서킷이 열림 (오탐). 멘토링 피드백: "윈도우 사이즈 10, 미니멈 콜 3일 때 3번째 실패부터 서킷이 열릴 수 있음 — 미니멈 콜에 도달해야 실패율 계산이 시작됨" |
| **CircuitBreaker failure-rate** | 50% | 10건 중 5건 실패 시 OPEN. PG 정상 시에도 500 에러(40%)가 Retry 없이 즉시 CB에 카운트되므로, 정상 상태에서 서킷이 열릴 수 있는 위험 존재. 타임아웃만 Retry 대상이므로 500은 매번 CB 실패로 카운트됨. 필요 시 failure-rate를 60%로 올려 여유를 둘 수 있음 |
| **CircuitBreaker wait-in-open** | 10s | PG 복구 시간 고려, 너무 길면 사용자 경험 저하 |

---

## 과제 ④ ErrorType 추가

### 수정: `support/error/ErrorType.kt`

```kotlin
SERVICE_UNAVAILABLE(HttpStatus.SERVICE_UNAVAILABLE, "Service Unavailable", "외부 서비스가 일시적으로 불가합니다."),
```

---

## 과제 ⑤ PG 통신 로그 (AOP 기반 자동 저장)

> PG와의 모든 요청/응답을 DB에 저장하여 장애 추적, 재시도 이력, 콜백 미도착 원인 파악에 활용.
> **로그 저장 실패 시 결제 흐름에 영향 없음** (try-catch 격리).

### 5-1. 도메인 레이어

#### 신규: `domain/pg/PgCommunicationLog.kt`

```kotlin
@Entity
@Table(
    name = "pg_communication_logs",
    indexes = [
        Index(name = "idx_pg_log_transaction_key", columnList = "transaction_key"),
        Index(name = "idx_pg_log_order_id", columnList = "order_id"),
        Index(name = "idx_pg_log_created_at", columnList = "created_at"),
    ],
)
class PgCommunicationLog(
    method: String,
    url: String,
    orderId: String?,
    transactionKey: String?,
    requestBody: String?,
    responseBody: String?,
    httpStatus: Int?,
    success: Boolean,
    errorMessage: String?,
    elapsed: Long,
) : BaseEntity() {

    @Column(name = "method", nullable = false, length = 10)
    val method: String = method                   // "POST", "GET"

    @Column(name = "url", nullable = false, length = 200)
    val url: String = url                         // "/api/v1/payments", "/api/v1/payments/{key}"

    @Column(name = "order_id", length = 50)
    val orderId: String? = orderId

    @Column(name = "transaction_key", length = 50)
    val transactionKey: String? = transactionKey

    @Column(name = "request_body", columnDefinition = "TEXT")
    val requestBody: String? = requestBody        // JSON 직렬화된 요청 본문

    @Column(name = "response_body", columnDefinition = "TEXT")
    val responseBody: String? = responseBody      // JSON 직렬화된 응답 본문

    @Column(name = "http_status")
    val httpStatus: Int? = httpStatus             // 200, 500 등

    @Column(name = "success", nullable = false)
    val success: Boolean = success                // PG 호출 성공 여부

    @Column(name = "error_message", length = 500)
    val errorMessage: String? = errorMessage      // 예외 메시지 (실패 시)

    @Column(name = "elapsed", nullable = false)
    val elapsed: Long = elapsed                   // 소요 시간 (ms)
}
```

#### 신규: `domain/pg/PgCommunicationLogRepository.kt`

```kotlin
interface PgCommunicationLogRepository {
    fun save(log: PgCommunicationLog): PgCommunicationLog
}
```

### 5-2. 인프라 레이어

#### 신규: `infrastructure/pg/PgCommunicationLogJpaRepository.kt`

```kotlin
interface PgCommunicationLogJpaRepository : JpaRepository<PgCommunicationLog, Long>
```

#### 신규: `infrastructure/pg/PgCommunicationLogRepositoryImpl.kt`

```kotlin
@Repository
class PgCommunicationLogRepositoryImpl(
    private val jpaRepository: PgCommunicationLogJpaRepository,
) : PgCommunicationLogRepository {
    override fun save(log: PgCommunicationLog) = jpaRepository.save(log)
}
```

### 5-3. Feign Client 데코레이터 (HTTP 레벨 로깅)

> Feign의 `Client` 인터페이스를 데코레이트하여 **HTTP 레벨에서** request/response를 캡처.
> `RequestInterceptor`는 요청만 가로채고 응답을 볼 수 없으므로, `Client` 래핑 방식을 사용.
> **별도 트랜잭션**(`REQUIRES_NEW`)으로 저장하여 결제 트랜잭션 롤백 시에도 로그는 유지.

#### 신규: `infrastructure/pg/PgLoggingClient.kt`

```kotlin
/**
 * Feign Client 데코레이터 — 모든 PG HTTP 통신의 request/response를 DB에 자동 저장.
 * Feign이 실제 HTTP 요청을 보내는 Client.execute()를 감싸므로,
 * Retry 각 시도마다 개별 HTTP 요청이 정확히 기록됨.
 */
class PgLoggingClient(
    private val delegate: Client,
    private val pgCommunicationLogRepository: PgCommunicationLogRepository,
    private val transactionTemplate: TransactionTemplate,
) : Client {

    override fun execute(request: Request, options: Request.Options): Response {
        val startTime = System.currentTimeMillis()
        val requestUrl = request.url()
        val httpMethod = request.httpMethod().name
        val requestBody = request.body()?.let { String(it, Charsets.UTF_8) }

        // orderId 추출 (요청 본문 또는 URL 파라미터에서)
        val orderId = extractOrderId(requestUrl, requestBody)

        return try {
            val response = delegate.execute(request, options)
            val elapsed = System.currentTimeMillis() - startTime

            // 응답 본문 읽기 (스트림을 다시 읽을 수 있도록 버퍼링)
            val responseBody = response.body()?.let {
                val bytes = it.asInputStream().readAllBytes()
                response = response.toBuilder().body(bytes).build()  // 재사용 가능하게 복원
                String(bytes, Charsets.UTF_8)
            }
            val transactionKey = extractTransactionKey(responseBody)

            saveLogAsync(httpMethod, requestUrl, orderId, transactionKey,
                requestBody, responseBody, response.status(), true, null, elapsed)

            response
        } catch (e: Exception) {
            val elapsed = System.currentTimeMillis() - startTime

            saveLogAsync(httpMethod, requestUrl, orderId, null,
                requestBody, null, null, false, e.message, elapsed)

            throw e
        }
    }

    /**
     * 별도 트랜잭션(REQUIRES_NEW)으로 로그 저장.
     * 저장 실패 시 결제 흐름에 영향 없음.
     */
    private fun saveLogAsync(
        method: String, url: String, orderId: String?, transactionKey: String?,
        requestBody: String?, responseBody: String?, httpStatus: Int?,
        success: Boolean, errorMessage: String?, elapsed: Long,
    ) {
        try {
            transactionTemplate.execute {
                pgCommunicationLogRepository.save(
                    PgCommunicationLog(
                        method = method,
                        url = url,
                        orderId = orderId,
                        transactionKey = transactionKey,
                        requestBody = requestBody,
                        responseBody = responseBody,
                        httpStatus = httpStatus,
                        success = success,
                        errorMessage = errorMessage?.take(500),
                        elapsed = elapsed,
                    )
                )
            }
        } catch (e: Exception) {
            logger.warn("PG 통신 로그 저장 실패: ${e.message}", e)
        }
    }

    private fun extractOrderId(url: String, requestBody: String?): String? { ... }
    private fun extractTransactionKey(responseBody: String?): String? { ... }
}
```

#### 수정: `infrastructure/pg/PgClientConfig.kt`에 로깅 Client 등록

```kotlin
@Configuration
class PgClientConfig(
    private val pgCommunicationLogRepository: PgCommunicationLogRepository,
) {
    @Bean
    fun feignOptions(): Request.Options {
        return Request.Options(
            1000, TimeUnit.MILLISECONDS,
            3000, TimeUnit.MILLISECONDS,
            true,
        )
    }

    @Bean
    fun pgFeignClient(): Client {
        val transactionTemplate = TransactionTemplate(transactionManager).apply {
            propagationBehavior = TransactionDefinition.PROPAGATION_REQUIRES_NEW
        }
        return PgLoggingClient(
            delegate = Client.Default(null, null),
            pgCommunicationLogRepository = pgCommunicationLogRepository,
            transactionTemplate = transactionTemplate,
        )
    }
}
```

### 5-4. AOP 방식 vs Feign Client 데코레이터 비교

| 항목 | AOP 방식 | Feign Client 데코레이터 (채택) |
|------|---------|------|
| **로깅 레벨** | 서비스 메서드 레벨 | **HTTP 레벨** (raw request/response) |
| **Retry 기록** | PgPaymentClient 메서드 기준 | **실제 HTTP 요청 단위** (더 정확) |
| **캡처 데이터** | 직렬화된 DTO | **raw HTTP body, status code** |
| **비즈니스 코드 영향** | 없음 (AOP) | **없음 (Client 데코레이터)** |
| **타임아웃 기록** | 예외 메시지만 | **소요시간 + 예외 메시지** |
| **FeignClient 설정 연동** | 별도 | **Feign Config에 통합** |

### 5-5. 설계 포인트

| 항목 | 설계 결정 | 이유 |
|------|----------|------|
| **저장 방식** | Feign `Client` 데코레이터 | HTTP 레벨에서 raw request/response를 정확히 캡처 |
| **대상** | PG FeignClient의 모든 HTTP 요청 | Retry 각 시도가 개별 HTTP 요청이므로 자동으로 개별 기록 |
| **트랜잭션** | `REQUIRES_NEW` (TransactionTemplate) | 결제 트랜잭션 롤백 시에도 로그는 유지 |
| **실패 격리** | `try-catch` | 로그 저장 실패가 결제 흐름에 영향 없음 |
| **저장 내용** | raw HTTP body, status, URL, elapsed(ms), error | 장애 분석에 필요한 최소 정보 |

### 5-5. 로그로 확인 가능한 시나리오

```
1. Retry 이력 추적:
   → 같은 orderId로 3건의 로그 (1,2번 실패 + 3번 성공)

2. 타임아웃인데 PG에서는 처리됨:
   → 요청 로그: success=false, errorMessage="Read timed out", elapsed=3000ms
   → 이후 콜백은 정상 도착 → transactionKey로 매칭하여 추적

3. PG 500 에러 패턴 분석:
   → httpStatus=500 로그가 연속 → CircuitBreaker OPEN 시점과 일치하는지 확인

4. 콜백 미도착 원인:
   → 요청 로그에 transactionKey 있고 success=true → PG는 정상 수신
   → 콜백 로그 없음 → PG 측 콜백 발송 실패
```

---

## 수정 대상 파일 종합

### 신규 파일 (27개)

| # | 경로 (apps/commerce-api/src/main/kotlin/com/loopers/ 기준) | 용도 |
|---|------|------|
| 1 | `domain/order/OrderStatus.kt` | 주문 상태 enum |
| 1-1 | `infrastructure/id/SnowflakeIdGenerator.kt` | Snowflake ID 생성기 (DB/Redis 의존 없음) |
| 2 | `domain/payment/Payment.kt` | 결제 엔티티 |
| 3 | `domain/payment/PaymentStatus.kt` | 결제 상태 enum |
| 4 | `domain/payment/PaymentRepository.kt` | Repository 인터페이스 |
| 5 | `infrastructure/payment/PaymentJpaRepository.kt` | JPA Repository |
| 6 | `infrastructure/payment/PaymentRepositoryImpl.kt` | Repository 구현체 |
| 7 | `infrastructure/pg/PgClient.kt` | FeignClient 인터페이스 |
| 8 | `infrastructure/pg/PgClientConfig.kt` | Feign 타임아웃 설정 |
| 9 | `infrastructure/pg/PgPaymentRequest.kt` | PG 요청 DTO |
| 10 | `infrastructure/pg/PgApiResponse.kt` | PG 응답 래퍼 DTO |
| 11 | `infrastructure/pg/PgPaymentResponse.kt` | PG 결제요청 응답 data DTO |
| 12 | `infrastructure/pg/PgTransactionDetailResponse.kt` | PG 상태확인 응답 data DTO |
| 13 | `infrastructure/pg/PgOrderTransactionsResponse.kt` | PG 주문별 조회 응답 data DTO |
| 14 | `infrastructure/pg/PgCallbackRequest.kt` | PG 콜백 요청 DTO |
| 15 | `domain/pg/PgCommunicationLog.kt` | PG 통신 로그 엔티티 |
| 16 | `domain/pg/PgCommunicationLogRepository.kt` | 로그 Repository 인터페이스 |
| 17 | `infrastructure/pg/PgCommunicationLogJpaRepository.kt` | 로그 JPA Repository |
| 18 | `infrastructure/pg/PgCommunicationLogRepositoryImpl.kt` | 로그 Repository 구현체 |
| 19 | `infrastructure/pg/PgLoggingClient.kt` | Feign Client 데코레이터 (HTTP 레벨 통신 로깅) |
| 20 | `application/payment/PaymentCriteria.kt` | 결제 요청 Criteria |
| 21 | `application/payment/PaymentInfo.kt` | 결제 Info DTO |
| 22 | `application/payment/PaymentService.kt` | 결제 CRUD 서비스 |
| 23 | `application/payment/PgPaymentClient.kt` | Resilience4j 래퍼 |
| 24 | `application/payment/PaymentFacade.kt` | 결제 오케스트레이션 |
| 25 | `interfaces/api/payment/PaymentV1ApiSpec.kt` | Swagger 스펙 |
| 26 | `interfaces/api/payment/PaymentV1Controller.kt` | 결제 API 컨트롤러 (콜백 포함) |
| 27 | `interfaces/api/payment/PaymentV1Dto.kt` | Request/Response DTO |

### 수정 파일 (7개)

| # | 경로 | 변경 내용 |
|---|------|----------|
| 1 | `apps/commerce-api/build.gradle.kts` | OpenFeign + Resilience4j + AOP 의존성 추가 |
| 2 | `CommerceApiApplication.kt` | `@EnableFeignClients` 추가 |
| 3 | `domain/order/Order.kt` | status 필드 + markPaid/markFailed/validatePayable/validateOwner |
| 4 | `application/order/OrderInfo.kt` | status 필드 추가 |
| 5 | `interfaces/api/order/OrderV1Dto.kt` | Response에 status 필드 추가 |
| 6 | `support/error/ErrorType.kt` | SERVICE_UNAVAILABLE 추가 |
| 7 | `application.yml` | pg 설정 + resilience4j 설정 |

---

## 테스트 계획

### 단위 테스트

| 대상 | 테스트 내용 |
|------|-----------|
| `Order` (기존 수정) | markPaid(): PENDING → PAID 성공 |
| | markPaid(): PAID/FAILED → 예외 발생 |
| | markFailed(): PENDING → FAILED 성공 |
| | validatePayable(): PENDING이 아닐 때 예외 |
| | validateOwner(): 다른 userId일 때 예외 |
| `Payment` | 엔티티 생성 시 기본 status=REQUESTED |
| | markPaid(): REQUESTED → PAID 성공 |
| | markFailed(): REQUESTED → FAILED, failReason 저장 |
| | markPaid(): PAID/FAILED → 예외 |
| | maskCardNo(): 카드번호 마스킹 검증 |
| | assignTransactionKey(): transactionKey 할당 |
| `PaymentService` (Mockito) | createPayment → save 호출 확인 |
| | getPaymentByOrderId → findByOrderId 호출 확인 |
| | getPaymentByTransactionKey → findByTransactionKey 호출 확인 |
| `PaymentFacade` (Mockito) | 정상 결제 요청 흐름 |
| | 주문 미존재 시 NOT_FOUND |
| | 다른 사용자의 주문 → FORBIDDEN |
| | 이미 결제 진행 중 → CONFLICT |
| | PG 호출 실패 (fallback) → SERVICE_UNAVAILABLE |
| | 콜백 수신 → Payment + Order 상태 업데이트 |
| | 상태 조회 시 REQUESTED → PG 확인 → 동기화 |
| `PgLoggingClient` | PG 호출 성공 시 로그 저장 확인 (success=true, responseBody 포함) |
| | PG 호출 실패 시 로그 저장 확인 (success=false, errorMessage 포함) |
| | 로그 저장 실패해도 원본 예외가 그대로 throw 되는지 확인 |

### 통합 테스트 (Testcontainers + 실제 DB)

| 대상 | 테스트 내용 |
|------|-----------|
| `PaymentService` | Payment 생성/저장/조회 실제 DB 검증 |
| | orderId, transactionKey 기반 조회 검증 |
| `PaymentFacade` | Order 생성 → Payment 생성 → 상태 업데이트 통합 흐름 |
| | 콜백 처리 → Payment + Order 상태 변경 DB 반영 확인 |
| `PgLoggingClient` | PG 호출 시 pg_communication_logs 테이블에 로그 저장 확인 |
| | Retry 3회 시도 시 HTTP 요청 3건 → 로그 3건 저장 확인 |
| | 결제 트랜잭션 롤백 시에도 로그는 유지 (REQUIRES_NEW) |

> 통합 테스트에서 PgClient는 MockBean 처리 (실제 PG 시뮬레이터 의존 제거)

### E2E 테스트 (TestRestTemplate)

| 대상 | 테스트 내용 |
|------|-----------|
| `POST /api/v1/payments` | 200 OK — 결제 요청 성공 |
| | 401 — 인증 실패 |
| | 404 — 존재하지 않는 주문 |
| | 409 — 중복 결제 요청 |
| | 503 — PG 장애 (CircuitBreaker fallback) |
| `POST /api/v1/payments/callback` | 200 OK — 콜백 수신 성공 (status=SUCCESS) |
| | 200 OK — 콜백 수신 성공 (status=FAILED) |
| | 404 — 존재하지 않는 transactionKey |
| `GET /api/v1/payments?orderId=X` | 200 OK — 결제 상태 조회 |
| | 404 — 결제 정보 없음 |
| | 403 — 다른 사용자의 주문 |

> E2E 테스트에서 PgClient는 @MockBean으로 교체

---

## 커밋 계획

| 순서 | 타입 | 메시지 | 포함 내용 |
|------|------|--------|----------|
| 1 | `feat:` | Order 엔티티에 주문 상태 추가 | OrderStatus, Order 수정, OrderInfo/DTO 수정 |
| 2 | `test:` | Order 주문 상태 전이 테스트 추가 | 단위 테스트 |
| 3 | `feat:` | 결제 도메인 및 PG 연동 구현 | Payment 전 레이어 + PG FeignClient + Resilience4j + API |
| 4 | `test:` | 결제 도메인 테스트 추가 | 단위 + 통합 + E2E |

---

## 구현 순서

```
Step 1: 의존성 추가 (build.gradle.kts, @EnableFeignClients)
Step 2: OrderStatus enum + Order 엔티티 수정
         → 커밋 1: feat: Order 엔티티에 주문 상태 추가
Step 3: Order 상태 전이 테스트
         → 커밋 2: test: Order 주문 상태 전이 테스트 추가
Step 4: Payment 도메인 레이어 (Entity, Status, Repository)
Step 5: Payment 인프라 레이어 (JPA, RepositoryImpl)
Step 6: PG 클라이언트 (FeignClient, Config, DTOs)
Step 7: Payment 애플리케이션 레이어 (Service, PgPaymentClient, Facade)
Step 8: Payment API 레이어 (Controller, ApiSpec, DTO)
Step 9: Resilience4j 설정 (application.yml)
Step 10: ErrorType 추가
         → 커밋 3: feat: 결제 도메인 및 PG 연동 구현
Step 11: Payment 단위 테스트
Step 12: Payment 통합 테스트
Step 13: Payment E2E 테스트
         → 커밋 4: test: 결제 도메인 테스트 추가
```

---

## 검증 방법

### 자동 검증

```bash
# 전체 테스트 통과
./gradlew :apps:commerce-api:test

# 코드 스타일 통과
./gradlew ktlintCheck
```

### 수동 검증 (pg-simulator 실행 필요)

```bash
# 1. pg-simulator 실행 (별도 프로젝트, 포트 8082)
cd /Users/soyoonbeom/github/loopers/loopback-be-l2-kotlin-additionals
./gradlew :apps:pg-simulator:bootRun

# 2. commerce-api 실행 (포트 8080)
cd /Users/soyoonbeom/github/loopers/loop-pack-be-l2-vol3-kotlin
./gradlew :apps:commerce-api:bootRun
```

```http
### 1. 주문 생성 후 결제 요청
POST http://localhost:8080/api/v1/payments
X-Loopers-LoginId: testUser
X-Loopers-LoginPw: testPw
Content-Type: application/json

{
  "orderId": 1,
  "cardType": "SAMSUNG",
  "cardNo": "1234-5678-9814-1451"
}

### 2. 결제 상태 조회 (콜백 수신 전 → REQUESTED)
GET http://localhost:8080/api/v1/payments?orderId=1
X-Loopers-LoginId: testUser
X-Loopers-LoginPw: testPw

### 3. 콜백 수신 후 → 상태 조회 (PAID or FAILED)
GET http://localhost:8080/api/v1/payments?orderId=1
X-Loopers-LoginId: testUser
X-Loopers-LoginPw: testPw

### 4. PG 중단 후 결제 요청 → CircuitBreaker 동작 확인 (503)
POST http://localhost:8080/api/v1/payments
X-Loopers-LoginId: testUser
X-Loopers-LoginPw: testPw
Content-Type: application/json

{
  "orderId": 2,
  "cardType": "SAMSUNG",
  "cardNo": "1234-5678-9814-1451"
}
```

---

## CircuitBreaker 동작 검증 방안

> 서킷브레이커가 CLOSED → OPEN → HALF_OPEN → CLOSED 로 전이되는 과정을 3가지 방법으로 확인.

### 사전 설정

#### 1. Actuator 엔드포인트 활성화

```yaml
# application.yml에 추가
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

#### 2. Resilience4j 로그 레벨 설정

```yaml
# application.yml에 추가
logging:
  level:
    io.github.resilience4j: DEBUG
```

#### 3. Grafana 대시보드 (Optional)

- 프로젝트에 monitoring 모듈(Prometheus + Grafana) 이미 존재
- Resilience4j의 `register-health-indicator: true` 설정과 연동하면 서킷 상태를 실시간 그래프로 확인 가능

---

### 테스트 시나리오

#### 시나리오 1: 정상 → PG 중단 → 서킷 OPEN 확인

```
1. pg-simulator 실행 + commerce-api 실행
2. 서킷 상태 확인 (CLOSED)
   GET http://localhost:8080/actuator/circuitbreakers
   → pgCircuit: "CLOSED"

3. pg-simulator 중단 (kill)

4. 결제 요청 10건 연속 전송 (모두 실패)
   → 콘솔 로그에서 Retry 동작 확인:
     "Retry 'pgRetry', waiting 500ms before next attempt..."

5. 서킷 상태 확인 (OPEN)
   GET http://localhost:8080/actuator/circuitbreakers
   → pgCircuit: "OPEN"

   콘솔 로그 확인:
   → "CircuitBreaker 'pgCircuit' changed state from CLOSED to OPEN"

6. 추가 결제 요청 전송
   → PG 호출 없이 즉시 fallback 응답 (503 SERVICE_UNAVAILABLE)
   → 응답 시간이 매우 빠름 (PG 호출 안 함)
```

#### 시나리오 2: 서킷 OPEN → HALF_OPEN → CLOSED 복구 확인

```
1. 시나리오 1 상태에서 이어서 진행 (서킷 OPEN)

2. pg-simulator 재실행

3. 10초 대기 (wait-duration-in-open-state: 10s)

4. 서킷 상태 확인 (HALF_OPEN)
   GET http://localhost:8080/actuator/circuitbreakers
   → pgCircuit: "HALF_OPEN"

   콘솔 로그:
   → "CircuitBreaker 'pgCircuit' changed state from OPEN to HALF_OPEN"

5. 결제 요청 전송 (HALF_OPEN에서 3건만 통과)
   → permitted-number-of-calls-in-half-open-state: 3
   → 3건 중 과반 성공 시 CLOSED 복귀

6. 서킷 상태 확인 (CLOSED)
   GET http://localhost:8080/actuator/circuitbreakers
   → pgCircuit: "CLOSED"

   콘솔 로그:
   → "CircuitBreaker 'pgCircuit' changed state from HALF_OPEN to CLOSED"
```

#### 시나리오 3: 서킷 이벤트 이력 전체 확인

```
GET http://localhost:8080/actuator/circuitbreakerevents

→ 전체 이벤트 이력:
  - type: "SUCCESS", timestamp: ...
  - type: "ERROR", timestamp: ...
  - type: "STATE_TRANSITION", stateTransition: "CLOSED_TO_OPEN", timestamp: ...
  - type: "NOT_PERMITTED", timestamp: ...  (OPEN 상태에서 차단된 요청)
  - type: "STATE_TRANSITION", stateTransition: "OPEN_TO_HALF_OPEN", timestamp: ...
  - type: "STATE_TRANSITION", stateTransition: "HALF_OPEN_TO_CLOSED", timestamp: ...
```

#### 시나리오 4: PG 통신 로그로 Retry + CircuitBreaker 동작 추적

```
서킷 OPEN 전후의 pg_communication_logs 테이블 조회:

SELECT method, url, http_status, success, elapsed, created_at
FROM pg_communication_logs
ORDER BY created_at DESC;

→ 서킷 OPEN 전: 요청마다 로그 3건 (Retry 3회 × 각 HTTP 요청)
  | POST | /api/v1/payments | null  | false | 3001ms |  ← 타임아웃
  | POST | /api/v1/payments | null  | false | 3002ms |  ← 재시도 1
  | POST | /api/v1/payments | null  | false | 3001ms |  ← 재시도 2

→ 서킷 OPEN 후: 로그 없음 (PG 호출 자체가 차단됨)
  → Fallback이 즉시 응답하므로 HTTP 요청 발생 안 함 → 로그 없음
```

### 검증 체크리스트

| # | 확인 항목 | 확인 방법 |
|---|---------|----------|
| 1 | CLOSED → OPEN 전이 | Actuator + 로그 |
| 2 | OPEN 상태에서 PG 호출 차단 (fallback 즉시 응답) | 응답 시간 비교 + pg_communication_logs 로그 없음 확인 |
| 3 | OPEN → HALF_OPEN 전이 (10초 후) | Actuator + 로그 |
| 4 | HALF_OPEN → CLOSED 복귀 | PG 재실행 후 요청 → Actuator 확인 |
| 5 | Retry 동작 | pg_communication_logs에 요청당 최대 3건 로그 |
| 6 | 서킷 이벤트 이력 | `/actuator/circuitbreakerevents` 전체 이력 확인 |
| 7 | Grafana 대시보드 (Optional) | 서킷 상태 실시간 그래프 |
