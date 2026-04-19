# Round 8 구현 계획: Redis 기반 주문 대기열 시스템

## Context

블랙 프라이데이 트래픽 폭증(100 req/s → 10,000 req/s) 상황에서 시스템을 보호하면서 유저에게 공정한 대기 경험을 제공하는 대기열 시스템을 구현한다. Redis Sorted Set 기반으로 순서를 보장하고, 입장 토큰과 스케줄러로 처리량을 제어하며, Polling 기반 실시간 순번 조회를 제공한다.

**핵심 전략**: 대기열은 주문 API **앞단의 관문**이고, 주문 이후 흐름(ApplicationEvent → Kafka)은 R7 구조를 그대로 활용한다.

---

## 현재 상태 (AS-IS)

| 항목 | 상태 | 비고 |
|------|------|------|
| 주문 API (POST /api/v1/orders) | ✅ | OrderFacade.createOrder() |
| 주문 후 이벤트 파이프라인 (R7) | ✅ | ApplicationEvent → Kafka |
| Redis 모듈 (Master-Replica) | ✅ | masterRedisTemplate / defaultRedisTemplate |
| 대기열 진입 API | ❌ | 미구현 |
| 입장 토큰 발급/검증 | ❌ | 미구현 |
| 스케줄러 기반 순차 처리 | ❌ | 미구현 |
| 순번 조회 API | ❌ | 미구현 |

---

## Redis Key 설계

| Key | Type | 용도 |
|-----|------|------|
| `queue:waiting:order` | Sorted Set | 대기열 (score=epoch ms, member=userId) |
| `queue:token:{userId}` | String | 입장 토큰 (value=UUID, TTL=5분) |

**Redis Template 사용 규칙**:
- 쓰기 (ZADD, ZPOPMIN, SET, DEL): `masterRedisTemplate` (`@Qualifier("redisTemplateMaster")`)
- 읽기 (ZRANK, ZCARD, GET): `defaultRedisTemplate` (Primary, replica-preferred)

---

## 처리량 설계 기준

### 이론값 산출

```
DB 커넥션 풀: 50
주문 1건 평균 처리 시간: 200ms (가정)
이론적 최대 TPS: 50 / 0.2 = 250 TPS
안전 마진 70%: 175 TPS
스케줄러: 100ms마다 ~18명씩 토큰 발급 (Thundering Herd 완화)
```

### 근거 및 보정 계획

- 200ms는 **이론적 가정**이다. 실제 주문 처리 시간은 재고 락, 결제 연동, 쿠폰 적용 여부에 따라 달라질 수 있음
- **Step 6 (k6 부하 테스트)에서 실측** 후 배치 크기를 보정한다
- 보정 흐름: 이론값(175 TPS) → k6 실측 → 실제 안정 TPS 확인 → 배치 크기 조정
- 예상 대기 시간 공식도 실측 TPS 기반으로 보정: `position / 실측_TPS`

### k6 실측 결과 (2026-04-02, 로컬 환경)

| 항목 | 이론값 | 실측값 |
|------|--------|--------|
| DB 커넥션 풀 | 50 (가정) | 10 (HikariCP 기본값) |
| 주문 처리 시간 | 200ms | 538ms (avg) |
| 최대 TPS | 175 | 11.5 |
| 대기열 진입 P95 | - | 80ms |
| Polling P95 | - | 80ms |
| 주문 에러율 | - | 0% |

**보정 방향**: 배치 크기와 예상 대기시간을 application.yml 설정으로 외부화하여 환경별로 조정 가능하게 변경 필요

---

## Step 1 — 대기열 Infrastructure + Domain + Application

### 1.1 WaitingQueueRedisRepository (infrastructure/queue/)

Redis Sorted Set 기반 대기열 저장소.

**파일**: `apps/commerce-api/src/main/kotlin/com/loopers/infrastructure/queue/WaitingQueueRedisRepository.kt`

```kotlin
@Component
class WaitingQueueRedisRepository(
    @Qualifier(RedisConfig.REDIS_TEMPLATE_MASTER)
    private val masterRedisTemplate: RedisTemplate<String, String>,
    private val redisTemplate: RedisTemplate<String, String>,  // replica-preferred
) {
    companion object {
        private const val QUEUE_KEY = "queue:waiting:order"
        private const val TOKEN_KEY_PREFIX = "queue:token:"
        private val TOKEN_TTL = Duration.ofMinutes(5)
    }

    fun addToQueue(userId: Long): Long          // ZADD → ZRANK 반환 (순번)
    fun getPosition(userId: Long): Long?        // ZRANK (0-based → 1-based 변환)
    fun getTotalCount(): Long                   // ZCARD
    fun popUsers(count: Long): List<Long>       // ZPOPMIN N명 꺼내기
    fun isInQueue(userId: Long): Boolean        // ZSCORE null 체크

    fun saveToken(userId: Long, token: String)  // SET + TTL
    fun getToken(userId: Long): String?         // GET
    fun deleteToken(userId: Long)               // DEL
    fun hasToken(userId: Long): Boolean         // EXISTS
}
```

**주의**: `addToQueue()`에서 ZADD 직후 ZRANK를 master에서 읽어야 replica lag 문제 방지.

### 1.2 QueueService (application/queue/)

**파일**: `apps/commerce-api/src/main/kotlin/com/loopers/application/queue/QueueService.kt`

```kotlin
@Component
class QueueService(
    private val waitingQueueRedisRepository: WaitingQueueRedisRepository,
) {
    fun enterQueue(userId: Long): QueueEntryInfo          // 대기열 진입
    fun getQueuePosition(userId: Long): QueuePositionInfo // 순번 + 예상 대기시간 조회
    fun issueTokens(batchSize: Long): Long                // N명 꺼내서 토큰 발급 (스케줄러 호출용)
    fun validateAndConsumeToken(userId: Long)              // 토큰 검증 + 삭제
}
```

**비즈니스 규칙**:
- 이미 대기열에 있는 유저 → 현재 순번 반환 (중복 진입 방지는 Sorted Set 특성으로 자동)
- 이미 토큰을 보유한 유저 → 토큰 있음 상태 반환 (재진입 불필요)
- 토큰 없이 주문 시도 → `CoreException(ErrorType.FORBIDDEN, "입장 토큰이 필요합니다.")`

### 1.3 DTO (application/queue/)

**QueueEntryInfo**: `position: Long, estimatedWaitSeconds: Long, totalWaiting: Long`
**QueuePositionInfo**: `position: Long, estimatedWaitSeconds: Long, totalWaiting: Long, token: String?`
- position = 0이고 token != null이면 → 입장 가능 상태

**예상 대기 시간 계산**: `position / THROUGHPUT_PER_SECOND` (THROUGHPUT_PER_SECOND = 175)

---

## Step 2 — 스케줄러

### 2.1 QueueScheduler (infrastructure/queue/)

**파일**: `apps/commerce-api/src/main/kotlin/com/loopers/infrastructure/queue/QueueScheduler.kt`

```kotlin
@Component
@ConditionalOnProperty(name = ["queue.scheduler.enabled"], havingValue = "true", matchIfMissing = true)
class QueueScheduler(
    private val queueService: QueueService,
) {
    @Scheduled(fixedRate = 100)  // 100ms마다 실행
    fun issueEntryTokens() {
        queueService.issueTokens(BATCH_SIZE)
    }

    companion object {
        private const val BATCH_SIZE = 18L  // 175 TPS / 10 batches per sec ≈ 18
    }
}
```

### 2.2 SchedulerConfig

**파일**: `apps/commerce-api/src/main/kotlin/com/loopers/support/config/SchedulerConfig.kt`

```kotlin
@Configuration
@EnableScheduling
class SchedulerConfig
```

### 2.3 application.yml 설정 추가

```yaml
queue:
  scheduler:
    enabled: true
```

test 프로파일 (`application-test.yml`):
```yaml
queue:
  scheduler:
    enabled: false
```

---

## Step 3 — API 레이어

### 3.1 QueueV1Controller (interfaces/api/queue/)

| Method | Endpoint | 인증 | 설명 |
|--------|----------|------|------|
| POST | `/api/v1/queue/enter` | O (LoginId + LoginPw) | 대기열 진입 |
| GET | `/api/v1/queue/position` | O (LoginId + LoginPw) | 순번 + 예상 대기시간 조회 |

```kotlin
@RestController
class QueueV1Controller(
    private val userService: UserService,
    private val queueService: QueueService,
) : QueueV1ApiSpec {

    @PostMapping("/api/v1/queue/enter")
    override fun enterQueue(loginId, password): ApiResponse<QueueV1Dto.EnterResponse>

    @GetMapping("/api/v1/queue/position")
    override fun getPosition(loginId, password): ApiResponse<QueueV1Dto.PositionResponse>
}
```

### 3.2 QueueV1ApiSpec + QueueV1Dto

Swagger 어노테이션 분리. 기존 `OrderV1ApiSpec` 패턴을 따름.

**Response DTO**:
- `EnterResponse`: position, estimatedWaitSeconds, totalWaiting
- `PositionResponse`: position, estimatedWaitSeconds, totalWaiting, token (nullable)

---

## Step 4 — 주문 API 토큰 검증 연동

### OrderV1Controller 수정

**파일**: `apps/commerce-api/src/main/kotlin/com/loopers/interfaces/api/order/OrderV1Controller.kt`

```kotlin
@PostMapping("/api/v1/orders")
override fun createOrder(loginId, password, request): ApiResponse<...> {
    val authUser = userService.authenticate(loginId, password)
    queueService.validateAndConsumeToken(authUser.id)  // 토큰 검증 + 소비
    val result = orderFacade.createOrder(authUser.id, request.toCriteria(), request.couponId)
    return ApiResponse.success(OrderV1Dto.CreateOrderResponse.from(result))
}
```

- QueueService 의존성 추가
- 토큰 없거나 만료 → `CoreException(ErrorType.FORBIDDEN, "입장 토큰이 필요합니다.")`

### 토큰 소비 이벤트 (ApplicationEvent 활용)

주문 완료 후 토큰 삭제를 직접 호출하지 않고, **ApplicationEvent로 분리**한다.

**이벤트**: `EntryTokenConsumedEvent(userId, token, orderId)`
- 발행 시점: `OrderFacade.createOrder()` 성공 후
- 핸들러: `@TransactionalEventListener(AFTER_COMMIT)` + `@Async`
  - 토큰 삭제 (`DEL queue:token:{userId}`)
  - 토큰 소비 카운트 증가 (AtomicLong 또는 Micrometer Counter)

**이유**:
- R7에서 구축한 이벤트 패턴과 일관성 유지
- 토큰 삭제가 주문 트랜잭션의 핵심 경로에서 분리됨 (토큰 삭제 실패가 주문 실패로 이어지지 않음)
- Token Conversion Rate, Token Expiry Rate 등 운영 지표 수집 기반 마련

### Token Expiry Rate 지표 수집

Redis TTL 만료는 이벤트를 발생시키지 않으므로, **산술 계산**으로 간접 측정한다.

```
Token Expiry Rate = (발급 수 - 소비 수) / 발급 수
```

**QueueService에 카운터 추가**:
- `tokenIssuedCount`: 토큰 발급 시 +1 (issueTokens에서)
- `tokenConsumedCount`: 토큰 소비 시 +1 (EntryTokenEventHandler에서)
- `getTokenExpiryRate()`: (issued - consumed) / issued 반환

이 지표로 TTL 5분이 적절한지 검증한다:
- 만료율 < 10%: 정상
- 만료율 10~30%: TTL 점검 필요
- 만료율 > 30%: TTL 늘리거나 주문 UX 개선 필요

**파일**:
- `application/queue/event/EntryTokenConsumedEvent.kt` (신규)
- `application/queue/event/EntryTokenEventHandler.kt` (신규)

---

## Step 5 — 테스트

### 5.1 단위 테스트 — QueueServiceTest
- Mockito 기반, 대기열 진입/순번 조회/토큰 검증 로직

### 5.2 통합 테스트 — QueueServiceIntegrationTest
- `@SpringBootTest` + Testcontainers (Redis)
- 실제 Redis 대기열 진입 → 토큰 발급 → 검증 흐름
- 동시 진입 순서 보장 확인
- 토큰 TTL 만료 테스트
- **Token Expiry Rate 검증 테스트**:
  - 10명에게 토큰 발급 → 7명만 주문 완료 → 3명 TTL 만료 대기
  - `getTokenExpiryRate()` = 30% 확인
  - TTL을 짧게 설정(테스트용 1~2초)하여 만료 시나리오 재현
  - 만료율 기준(10%/30%) 임계값 검증

### 5.3 E2E 테스트 — QueueV1ApiE2ETest
- `@SpringBootTest(RANDOM_PORT)` + `TestRestTemplate`
- POST /queue/enter → 순번 응답
- GET /queue/position → 대기시간 포함 응답
- 토큰 발급 후 POST /orders → 주문 성공
- 토큰 없이 POST /orders → 403

### 5.4 기존 테스트 영향
- `OrderV1Controller`에 토큰 검증 추가로 기존 주문 E2E 테스트 실패 예상
- 테스트 헬퍼로 Redis에 직접 토큰 삽입하여 해결

---

## Step 6 — k6 부하 테스트

Gradle 빌드 파이프라인 **밖에서** 실행. 통합 테스트에 포함하지 않으므로 빌드 시간에 영향 없음.

### 실행 방식

```
1. Docker 인프라 기동 (MySQL, Redis, Kafka)
2. ./gradlew :apps:commerce-api:bootRun
3. k6 run k6/queue-load-test.js
```

### k6 스크립트 구성

**파일**: `k6/queue-load-test.js`

| 시나리오 | 내용 |
|---------|------|
| 대기열 진입 부하 | 1,000 VU가 동시에 POST /queue/enter |
| Polling 부하 | 대기열 진입 후 2초마다 GET /queue/position |
| 주문 처리량 측정 | 토큰 발급 후 POST /orders → 실제 TPS 측정 |
| 스파이크 테스트 | 0 → 10,000 VU 급증 시 시스템 안정성 확인 |
| **Jitter A/B 비교** | 동일 부하에서 Jitter 없음 vs Jitter 적용(0~2초 랜덤 딜레이) 비교 |

### 검증 항목

- 실제 주문 처리 TPS (이론값 175와 비교)
- P99 응답 시간
- 에러율
- **측정 결과로 배치 크기(18) 및 예상 대기 시간 공식 보정**
- **Jitter 비교**: 동시 DB 커넥션 피크, P99 응답시간, 에러율 차이

---

## Step 7 — Nice-to-Have (Must-Have 완료 후 순차 적용)

Must-Have(Step 1~5) 완료 및 k6 실측(Step 6) 이후, 데이터 기반으로 필요성을 판단하여 적용한다.

### 7.1 Jitter — 토큰 활성화 시점 분산

**목적**: 같은 배치(18명)가 동시에 주문 API를 호출하는 것을 추가 분산
**구현**: 토큰 발급 시 0~2초 랜덤 딜레이를 토큰 메타데이터로 저장, 클라이언트가 딜레이 후 주문 호출
**적용 기준**: k6 Jitter A/B 비교에서 P99 응답시간이 유의미하게 차이날 때

```kotlin
// PositionResponse에 activateAfterMs 필드 추가
data class PositionResponse(
    val position: Long,
    val estimatedWaitSeconds: Long,
    val totalWaiting: Long,
    val token: String?,
    val activateAfterMs: Long?,  // 0~2000 랜덤, 클라이언트가 이만큼 대기 후 주문
)
```

### 7.2 동적 Polling 주기 — 순번 구간별 조회 간격 조절

**목적**: 대기 인원이 많을 때 Polling 자체의 서버 부하 감소
**구현**: 순번 응답에 `nextPollAfterMs` 필드 추가, 클라이언트가 이 값만큼 대기 후 재조회

```
순번 1~100:    nextPollAfterMs = 1000  (곧 입장, 빠르게 조회)
순번 100~1000: nextPollAfterMs = 3000
순번 1000+:    nextPollAfterMs = 5000  (아직 멀어, 느리게 조회)
```

**적용 기준**: k6 Polling 부하 시나리오에서 순번 조회 API의 P99가 높아질 때

### 7.3 Graceful Degradation — Redis 장애 시 Fallback

**목적**: Redis가 죽어도 서비스가 완전히 멈추지 않도록 대응
**선택지**:

| 전략 | 설명 | 트레이드오프 |
|------|------|-------------|
| 전면 차단 | 대기열 진입 막고 "잠시 후 다시 시도" 안내 | 안전하지만 서비스 중단 |
| 대기열 우회 | 토큰 없이 주문 API 직접 허용 | 서비스 유지하지만 과부하 위험 |
| Fallback 큐 | 로컬 메모리 큐나 Kafka로 임시 전환 | 순번 정확성 떨어짐 |

**구현**: QueueService에서 Redis 호출을 try-catch로 감싸고, 장애 시 설정된 전략으로 분기

```kotlin
fun enterQueue(userId: Long): QueueEntryInfo {
    return try {
        // Redis Sorted Set 진입
    } catch (e: Exception) {
        log.error("Redis 장애 감지", e)
        when (fallbackStrategy) {
            BLOCK -> throw CoreException(ErrorType.SERVICE_UNAVAILABLE, "대기열 서비스 점검 중입니다.")
            BYPASS -> QueueEntryInfo(position = 0, ..., token = issueBypassToken(userId))
        }
    }
}
```

**적용 기준**: 구현 우선순위는 낮지만, 어떤 전략을 쓸지 **사전에 결정해두는 것**이 핵심. k6에서 Redis 장애 시뮬레이션(Redis 컨테이너 kill)으로 검증 가능.

---

## 커밋 전략

| 순서 | 타입 | 메시지 |
|------|------|--------|
| 1 | `feat:` | `feat: 주문 대기열 시스템 구현` |
| 2 | `test:` | `test: 주문 대기열 도메인 테스트 추가` |
| 3 | `chore:` | `chore: k6 부하 테스트 스크립트 추가` |
| 4 | `feat:` | `feat: 대기열 Nice-to-Have 적용` (적용 항목에 따라 분리 가능) |

---

## 생성/수정 파일 목록

### 신규 생성
- `infrastructure/queue/WaitingQueueRedisRepository.kt`
- `application/queue/QueueService.kt`
- `application/queue/QueueEntryInfo.kt`
- `application/queue/QueuePositionInfo.kt`
- `application/queue/event/EntryTokenConsumedEvent.kt`
- `application/queue/event/EntryTokenEventHandler.kt`
- `infrastructure/queue/QueueScheduler.kt`
- `support/config/SchedulerConfig.kt`
- `interfaces/api/queue/QueueV1Controller.kt`
- `interfaces/api/queue/QueueV1ApiSpec.kt`
- `interfaces/api/queue/QueueV1Dto.kt`
- `k6/queue-load-test.js`
- 테스트 3개 (단위/통합/E2E)

### 수정
- `interfaces/api/order/OrderV1Controller.kt` — 토큰 검증 추가
- `application.yml` — 스케줄러 설정
- `application-test.yml` — 스케줄러 비활성화

---

## 검증 방법

1. `./gradlew :apps:commerce-api:test` — 전체 테스트 통과
2. `./gradlew ktlintCheck` — 코드 스타일 확인
3. Docker 기동 후 수동 E2E:
   - POST /queue/enter → 순번 응답
   - GET /queue/position → 순번 + 대기시간
   - 스케줄러 토큰 발급 후 POST /orders → 주문 성공
   - 토큰 없이 POST /orders → 403 에러
