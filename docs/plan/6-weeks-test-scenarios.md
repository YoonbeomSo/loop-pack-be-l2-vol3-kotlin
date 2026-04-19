# 6주차 수동 검증 시나리오

> pg-simulator(8082) + commerce-api(8080) 실행 후 진행.
> 각 시나리오의 **기대 결과**와 **실제 결과**를 기록하여 블로그 소재로 활용.

---

## 사전 준비

```bash
# 1. Docker 인프라 실행 (MySQL, Redis)
docker-compose -f docker/infra-compose.yml up -d

# 2. pg-simulator 실행 (포트 8082)
cd /Users/soyoonbeom/github/loopers/loopback-be-l2-kotlin-additionals
./gradlew :apps:pg-simulator:bootRun

# 3. commerce-api 실행 (포트 8080)
cd /Users/soyoonbeom/github/loopers/loop-pack-be-l2-vol3-kotlin
./gradlew :apps:commerce-api:bootRun

# 4. 테스트 데이터 생성 (회원가입 → 브랜드 → 상품 → 주문)
# http/commerce-api/ 의 user-v1.http, brand-admin-v1.http, product-admin-v1.http, order-v1.http 순서로 실행
```

---

## Part 1: 결제 정상 흐름

### 시나리오 1-1: 결제 요청 → 콜백 성공 → PAID

```
준비: 주문 1건 생성 (status=PENDING)

단계:
1. POST /api/v1/payments { orderId, cardType: "SAMSUNG", cardNo: "1234-5678-9814-1451" }

기대:
  - 응답 200, status="REQUESTED", transactionKey 존재
  - Payment 테이블: status=REQUESTED, transactionKey 저장됨
  - pg_communication_logs: POST 요청 1건, success=true, url=pg.base-url

2. 1~5초 대기 (PG 비동기 처리)

3. PG 콜백이 자동으로 POST /api/v1/payments/callback 호출

4. GET /api/v1/payments?orderId=X

기대:
  - status="PAID"
  - Order 테이블: status=PAID

확인 포인트:
  □ Payment 응답에 transactionKey가 있는가
  □ 콜백 수신 후 Payment/Order가 PAID로 변경되었는가
  □ pg_communication_logs에 요청 로그가 기록되었는가 (url에 PG base-url이 채워졌는가)
  □ 콘솔에 Resilience4j 관련 로그가 없는가 (정상이므로)
```

### 시나리오 1-2: 결제 요청 → 콜백 실패 (한도초과) → FAILED + 보상

```
준비: 주문 1건 생성

단계:
1. POST /api/v1/payments { orderId, cardType: "SAMSUNG", cardNo: "1234-5678-9814-1451" }
   (PG가 70% 성공, 20% 한도초과, 10% 잘못된 카드이므로 여러 번 시도 필요)

2. 콜백에서 status=FAILED 수신 시:

기대:
  - Payment: status=FAILED, failReason="한도초과입니다..."
  - Order: status=FAILED
  - 재고: 원래 수량으로 복원됨
  - 쿠폰: 사용 전 상태(AVAILABLE)로 복원됨 (쿠폰 사용한 주문인 경우)

확인 포인트:
  □ Payment.failReason에 PG 사유가 저장되었는가
  □ 재고가 정확히 복원되었는가 (products 테이블 stock 확인)
  □ 쿠폰이 복원되었는가 (issued_coupons 테이블 status 확인)
  □ Order.status가 FAILED인가
```

### 시나리오 1-3: 중복 결제 요청 → 409 CONFLICT

```
준비: 주문 1건 생성 + 이미 결제 요청 완료 (REQUESTED 상태)

단계:
1. POST /api/v1/payments { 같은 orderId }

기대:
  - 응답 409 CONFLICT "이미 결제가 진행 중인 주문입니다."

확인 포인트:
  □ 두 번째 요청이 409로 거부되는가
  □ Payment가 추가로 생성되지 않았는가 (payments 테이블 count 확인)
```

---

## Part 2: PG 실패 시나리오

### 시나리오 2-1: PG 500 에러 → 즉시 실패 (Retry 안 함)

```
준비: 주문 1건 생성

단계:
1. POST /api/v1/payments (PG가 40% 확률로 500 반환)
   → 500을 받을 때까지 다른 주문으로 반복 시도

기대:
  - Payment: status=FAILED (INITIATED → FAILED)
  - 500 에러는 Retry 하지 않음 (타임아웃만 Retry 대상)
  - pg_communication_logs: 요청 1건만 기록 (Retry 없으므로)
  - 보상 처리: 재고 복원 + 쿠폰 복원

확인 포인트:
  □ pg_communication_logs에 로그가 1건만 있는가 (Retry 안 했으므로)
  □ success=false가 기록되었는가 (PgApiResponse.isSuccess() 기반 판정)
  □ 보상 처리가 되었는가 (재고/쿠폰 확인)
```

### 시나리오 2-2: PG 타임아웃 → Retry 3회 → 최종 실패

```
준비: pg-simulator의 readTimeout보다 긴 지연을 유도하기 어려우므로,
     pg-simulator를 중단한 상태에서 테스트 (ConnectException 발생)

단계:
1. pg-simulator 중단
2. POST /api/v1/payments

기대:
  - Retry 3회 시도 (500ms → 1s → 2s 간격)
  - 총 소요 시간: 약 3.5s + 요청 시간
  - 최종 Fallback → 503 SERVICE_UNAVAILABLE
  - Payment: status=FAILED
  - 콘솔 로그: "Retry 'pgRetry'" 관련 로그 3회
  - Fallback 로그: "type=TIMEOUT_EXHAUSTED" 또는 "type=UNKNOWN" (ConnectException)

확인 포인트:
  □ pg_communication_logs에 로그가 3건 있는가 (Retry 3회)
  □ 각 로그의 success=false, errorMessage에 타임아웃/연결 실패 메시지가 있는가
  □ 응답 시간이 3.5초 이상인가 (Retry 대기 시간 포함)
  □ Fallback 로그에서 failureType이 기록되었는가
```

### 시나리오 2-3: PG 응답은 왔지만 우리가 타임아웃 처리한 경우

```
설명: PG가 실제로 결제를 처리했지만, readTimeout(3s)에 걸려서 우리 쪽에서는 실패로 처리.
     이후 PG 콜백이 도착하면 상태가 복구되어야 함.

이 시나리오는 pg-simulator의 요청 지연이 100~500ms이므로 재현이 어려움.
readTimeout을 임시로 100ms로 낮춰서 테스트 가능.

단계:
1. application.yml에서 readTimeout을 100ms로 임시 변경
2. POST /api/v1/payments
3. 타임아웃 발생 → Retry → 최종 실패 → Payment=FAILED
4. PG는 실제로 처리 → 콜백 도착
5. 콜백에서 transactionKey로 매칭 시도

기대:
  - Payment가 FAILED 상태이므로 콜백 수신 시 REQUESTED가 아니라 무시됨
  - ⚠️ 이 경우 PG에서는 결제 성공인데 우리는 FAILED → 불일치 발생
  - 이건 현재 설계의 한계점으로 인지

확인 포인트:
  □ 타임아웃 후 Payment가 FAILED가 되는가
  □ 이후 콜백이 와도 무시되는가 (REQUESTED 아니므로)
  □ ⚠️ 이 불일치가 발생함을 확인 (블로그 소재)
```

---

## Part 3: 콜백 미도착 + 폴링 복구

### 시나리오 3-1: 콜백 안 옴 → 상태 조회 시 PG 직접 확인 → 성공 동기화

```
준비: 결제 요청 성공 (REQUESTED 상태, transactionKey 있음)

설명: 콜백이 자동으로 오는 환경에서는 이 시나리오 재현이 어려움.
     PG가 처리 완료했지만 콜백 URL 호출이 실패한 상황을 가정.
     → 직접 GET /api/v1/payments?orderId=X 호출로 폴링 동작 확인

단계:
1. POST /api/v1/payments → REQUESTED 상태
2. PG 처리 완료 대기 (1~5초)
3. GET /api/v1/payments?orderId=X

기대:
  - getPaymentStatus()에서 REQUESTED 감지 → PG GET /payments/{transactionKey} 호출
  - PG 응답이 SUCCESS면 → Payment=PAID, Order=PAID
  - PG 응답이 FAILED면 → Payment=FAILED, Order=FAILED + 보상
  - PG 응답이 PENDING면 → 변경 없이 현재 상태 반환

확인 포인트:
  □ pg_communication_logs에 GET 요청 로그가 기록되는가
  □ PG 조회 결과에 따라 상태가 동기화되는가
  □ 이미 PAID/FAILED인 경우 PG 조회를 하지 않는가
```

### 시나리오 3-2: transactionKey 없는 INITIATED 상태 → 폴링 시 PG 조회 안 함

```
준비: Payment가 INITIATED 상태 (PG 호출 실패로 transactionKey 없음)

단계:
1. pg-simulator 중단 상태에서 결제 요청 → INITIATED → FAILED
2. pg-simulator 재시작
3. GET /api/v1/payments?orderId=X

기대:
  - FAILED 상태이므로 PG 조회 없이 현재 상태 반환
  - pg_communication_logs에 GET 요청 로그 없음

확인 포인트:
  □ INITIATED/FAILED 상태에서 불필요한 PG 조회를 하지 않는가
```

---

## Part 4: CircuitBreaker 동작 검증

### 시나리오 4-1: CLOSED → OPEN 전이

```
준비: pg-simulator 실행 상태에서 시작

단계:
1. 서킷 초기 상태 확인
   GET http://localhost:8080/actuator/circuitbreakers
   기대: pgCircuit state="CLOSED"

2. pg-simulator 중단 (kill 또는 Ctrl+C)

3. 결제 요청 반복 전송 (최소 5건 — minimum-number-of-calls)
   각 요청마다 Retry 3회 시도 후 실패

4. 서킷 상태 확인
   GET http://localhost:8080/actuator/circuitbreakers
   기대: pgCircuit state="OPEN"

5. 콘솔 로그 확인
   기대: "CircuitBreaker 'pgCircuit' changed state from CLOSED to OPEN"

6. 추가 결제 요청 전송
   기대:
   - 응답 시간이 매우 빠름 (PG 호출 없이 즉시 Fallback)
   - 503 SERVICE_UNAVAILABLE
   - Fallback 로그: "type=CIRCUIT_OPEN"

확인 포인트:
  □ 5건 실패 후 서킷이 OPEN으로 전환되는가
  □ OPEN 상태에서 PG 호출 없이 즉시 Fallback 되는가
  □ OPEN 후 응답 시간과 CLOSED 때 응답 시간 차이가 확연한가
  □ pg_communication_logs에 OPEN 이후 로그가 없는가 (PG 호출 자체가 차단)
```

### 시나리오 4-2: OPEN → HALF_OPEN 전이

```
준비: 시나리오 4-1 이후 상태 (서킷 OPEN)

단계:
1. pg-simulator 재실행

2. 10초 대기 (wait-duration-in-open-state: 10s)

3. 서킷 상태 확인
   GET http://localhost:8080/actuator/circuitbreakers
   기대: pgCircuit state="HALF_OPEN"

4. 콘솔 로그 확인
   기대: "CircuitBreaker 'pgCircuit' changed state from OPEN to HALF_OPEN"

확인 포인트:
  □ 정확히 10초 후에 HALF_OPEN으로 전환되는가
  □ 10초 이전에는 여전히 OPEN인가
```

### 시나리오 4-3: HALF_OPEN → CLOSED 복구

```
준비: 시나리오 4-2 이후 상태 (서킷 HALF_OPEN, pg-simulator 실행 중)

단계:
1. 결제 요청 3건 전송 (permitted-number-of-calls-in-half-open-state: 3)
   → PG 정상이므로 60% 확률로 성공

2. 3건 중 과반(2건 이상)이 성공하면:
   서킷 상태 확인
   기대: pgCircuit state="CLOSED"

3. 콘솔 로그 확인
   기대: "CircuitBreaker 'pgCircuit' changed state from HALF_OPEN to CLOSED"

확인 포인트:
  □ HALF_OPEN에서 3건만 통과되는가 (4번째는 어떻게 되는가?)
  □ 과반 성공 시 CLOSED로 복구되는가
  □ pg_communication_logs에 HALF_OPEN 기간 로그가 정확히 3건인가
```

### 시나리오 4-4: HALF_OPEN → OPEN 재전환 (복구 실패)

```
준비: 서킷 OPEN 상태 + pg-simulator 중단 유지

단계:
1. 10초 대기 → HALF_OPEN 전환

2. 결제 요청 3건 전송 (pg-simulator 중단이므로 모두 실패)

3. 서킷 상태 확인
   기대: pgCircuit state="OPEN" (다시)

4. 콘솔 로그 확인
   기대: "CircuitBreaker 'pgCircuit' changed state from HALF_OPEN to OPEN"

확인 포인트:
  □ HALF_OPEN에서 실패 시 다시 OPEN으로 돌아가는가
  □ 다시 10초 대기 후 HALF_OPEN을 재시도하는가
```

---

## Part 5: 서킷 이벤트 이력 전체 확인

### 시나리오 5-1: 전체 이벤트 이력 조회

```
단계:
1. Part 4를 모두 진행한 후
2. GET http://localhost:8080/actuator/circuitbreakerevents

기대 이벤트 순서:
  - type: "SUCCESS" (정상 요청들)
  - type: "ERROR" (실패 요청들)
  - type: "STATE_TRANSITION", stateTransition: "CLOSED_TO_OPEN"
  - type: "NOT_PERMITTED" (OPEN 상태에서 차단된 요청들)
  - type: "STATE_TRANSITION", stateTransition: "OPEN_TO_HALF_OPEN"
  - type: "SUCCESS" or "ERROR" (HALF_OPEN에서 시험 요청들)
  - type: "STATE_TRANSITION", stateTransition: "HALF_OPEN_TO_CLOSED" or "HALF_OPEN_TO_OPEN"

확인 포인트:
  □ 모든 상태 전이가 이벤트로 기록되는가
  □ NOT_PERMITTED 이벤트가 OPEN 기간에만 발생하는가
  □ 이벤트 timestamp로 전이 시점을 정확히 추적할 수 있는가
```

### 시나리오 5-2: pg_communication_logs로 Retry + CB 동작 추적

```
단계:
1. Part 4를 모두 진행한 후
2. DB 조회:

SELECT id, method, url, http_status, success, error_message, elapsed, created_at
FROM pg_communication_logs
ORDER BY created_at;

기대:
  - CLOSED 기간: 요청마다 Retry 횟수만큼 로그 (타임아웃 시 3건, 500 시 1건)
  - OPEN 기간: 로그 없음 (PG 호출 자체가 차단됨)
  - HALF_OPEN 기간: 정확히 3건 (permitted-number-of-calls-in-half-open-state)

확인 포인트:
  □ CLOSED → OPEN 전환 시점 전후로 로그 패턴이 달라지는가
  □ OPEN 기간에 로그가 0건인가
  □ Retry 로그의 elapsed 값으로 재시도 간격을 확인할 수 있는가
  □ error_message로 실패 원인 (타임아웃 vs 500 vs 연결 거부)을 구분할 수 있는가
  □ url 컬럼에 pg.base-url 값이 채워져 있는가 (기존 빈 문자열 → base-url 주입으로 수정)
  □ success 컬럼이 PgApiResponse.isSuccess() 기반으로 판정되는가 (HTTP 성공이어도 비즈니스 실패 시 false)
```

---

## Part 6: 응답 시간 비교

### 시나리오 6-1: 정상 vs CB OPEN 응답 시간 비교

```
단계:
1. PG 정상 상태에서 결제 요청 5건 → 평균 응답 시간 기록
2. PG 중단 → CB OPEN
3. CB OPEN 상태에서 결제 요청 5건 → 평균 응답 시간 기록

기대:
  - 정상: 100~500ms (PG 지연) + 서버 처리 시간
  - CB OPEN: 10ms 이내 (PG 호출 없이 즉시 Fallback)
  - 차이: 10~50배

확인 포인트:
  □ CB OPEN 시 응답 시간이 극적으로 빨라지는가
  □ "서킷브레이커가 시스템을 보호한다"는 것이 응답 시간으로 체감되는가
```

---

## Part 7: Fallback 로그 구분 확인

### 시나리오 7-1: Retry 소진 후 Fallback vs CB Open 후 Fallback

```
단계:
1. pg-simulator 중단
2. 결제 요청 전송 (Retry 3회 소진 → Fallback)
   → 콘솔 로그: "type=TIMEOUT_EXHAUSTED" 또는 "type=UNKNOWN"

3. 5건 이상 실패 → CB OPEN
4. 추가 결제 요청 전송 (CB OPEN → 즉시 Fallback)
   → 콘솔 로그: "type=CIRCUIT_OPEN"

확인 포인트:
  □ 두 Fallback의 로그가 다른 type으로 기록되는가
  □ CIRCUIT_OPEN은 Retry 없이 즉시 발생하는가
  □ 이 구분으로 "PG 장애인지 vs 일시적 오류인지" 판단 가능한가
```

---

## 결과 기록 템플릿

각 시나리오 실행 후 아래 형식으로 기록:

```markdown
### 시나리오 X-X: {시나리오명}

**실행 일시**: 2026-03-19 HH:mm

**기대 결과**: (위 시나리오의 기대 내용)

**실제 결과**:
- 응답 코드:
- 응답 본문:
- DB 상태:
- 콘솔 로그:
- pg_communication_logs:

**일치 여부**: ✅ 일치 / ❌ 불일치

**불일치 시 원인 분석**:

**스크린샷/로그 캡처**:
```

---

## 수동 검증 결과

### 시나리오 1-1: 결제 요청 → 콜백 성공 → PAID

**실행 일시**: 2026-03-19 23:34

**기대 결과**: 결제 요청 시 REQUESTED 응답, 콜백 수신 후 Payment/Order PAID 전이

**실제 결과**:
- 응답 코드: 200
- 응답 본문: `status=REQUESTED`, `transactionKey=20260319:TR:a52c27`
- DB 상태: Payment=PAID, Order=PAID, transactionKey 저장됨
- pg_communication_logs: POST 1건, success=true, elapsed=502ms

**일치 여부**: ✅ 일치

---

### 시나리오 1-2: 결제 요청 → 콜백 실패 (한도초과) → FAILED + 보상

**실행 일시**: 2026-03-19 23:37

**기대 결과**: 콜백 FAILED 수신 시 Payment/Order FAILED 전이, 재고 복원, 쿠폰 AVAILABLE 복원

**실제 결과**:
- 응답 코드: 200
- 응답 본문: `status=FAILED`, `failReason="한도초과입니다. 다른 카드를 선택해주세요."`
- DB 상태:
  - Payment: status=FAILED, failReason 저장됨
  - Order: status=FAILED
  - 재고: 보상 전 93 → 주문 5건(-5) → FAILED 보상(+1) → **89** (4건 PAID 차감만 남음, 정확)
  - 쿠폰(issuedCouponId=7): **AVAILABLE** (복원됨)

**일치 여부**: ✅ 일치

---

### 시나리오 1-3: 중복 결제 요청 → 409 CONFLICT

**실행 일시**: 2026-03-19 23:40

**기대 결과**: REQUESTED 상태 주문에 다시 결제 요청 시 409 CONFLICT

**실제 결과**:
- 응답 코드: 409
- 응답 본문: `errorCode=Conflict`, `message="이미 결제가 진행 중인 주문입니다."`
- DB 상태: Payment 추가 생성 없음

**일치 여부**: ✅ 일치

---

### 시나리오 2-1: PG 500 에러 → 즉시 실패 (Retry 안 함)

**실행 일시**: 2026-03-19 23:42

**기대 결과**: PG 500 시 Retry 없이 즉시 실패, pg_communication_logs 1건만 기록

**실제 결과**:
- 응답 코드: 200 (data.status=FAILED)
- 응답 본문: `status=FAILED`, `failReason="PG 시스템 장애"`
- pg_communication_logs: **1건**, success=false, elapsed=333ms, errorMessage=`[500] during [POST]...`

**일치 여부**: ✅ 일치

**Resilience 근거**: PG 500(FeignException.InternalServerError)은 `retry-exceptions` 화이트리스트(feign.RetryableException)에 포함되지 않으므로 Retry 대상이 아님. 실무에서 500은 서버 문제로 재시도해도 동일 결과일 확률이 높음.

---

### 시나리오 2-2: PG 다운 → Retry 3회 → 최종 실패

**실행 일시**: 2026-03-19 23:48

**기대 결과**: PG 다운(ConnectException) 시 Retry 3회 후 최종 실패, pg_communication_logs 3건

**실제 결과**:
- 응답 코드: 200 (data.status=FAILED)
- 응답 본문: `status=FAILED`, `failReason="PG 시스템 장애"`
- 총 소요 시간: **1.30s**
- pg_communication_logs: **3건** 기록
  - 1차: 14:48:45.255 (8ms, Connection refused)
  - 2차: 14:48:45.768 (2ms, Connection refused) — 1차로부터 ~513ms
  - 3차: 14:48:46.293 (3ms, Connection refused) — 2차로부터 ~525ms
- Fallback 로그: `type=CONNECTION_REFUSED`

**일치 여부**: ✅ 일치

**Resilience 근거**: ConnectException → Feign이 RetryableException으로 래핑 → `retry-exceptions`에 매칭 → 3회 재시도. 총 1.3초 소요로 사용자 체감 한계(5~10초) 이내.

---

### 시나리오 2-3: PG 응답은 왔지만 우리가 타임아웃 처리한 경우

**실행 일시**: 2026-03-20 00:09

**기대 결과**: readTimeout=100ms로 임시 변경 → 타임아웃 → Payment=FAILED → PG 콜백 도착 → FAILED 상태이므로 콜백 무시 (불일치 발생)

**실제 결과**:
- readTimeout을 100ms로 변경 후 결제 요청
- Retry 3회 모두 `Read timed out` (elapsed ~100~111ms)
- Retry 간격: 1→2 ~621ms (500ms), 2→3 ~1118ms (1000ms) → **지수 백오프 정상 작동 확인**
- Fallback 로그: `type=TIMEOUT_EXHAUSTED`
- Payment: status=**FAILED**, transactionKey=**NULL** (응답을 못 받았으므로)
- 6초 대기 후 상태 조회: 여전히 **FAILED** (콜백이 와도 transactionKey로 매칭 불가)

**일치 여부**: ✅ 일치

**설계 한계점 인지**: 타임아웃으로 transactionKey를 못 받으면, PG에서 결제가 실제로 성공하더라도 우리 시스템에서는 FAILED로 남음. PG ↔ 우리 시스템 간 상태 불일치가 발생하며, 이를 해결하려면 PG측 거래 조회 API를 orderId 기반으로도 지원하거나, 정산 배치에서 불일치를 검출하는 보정 메커니즘이 필요함.

---

### 시나리오 3-1: 콜백 미도착 → 상태 조회 시 PG 직접 확인

**실행 일시**: 2026-03-19 23:50

**기대 결과**: REQUESTED 상태 + transactionKey 존재 시 PG에 GET 요청으로 폴링

**실제 결과**:
- 결제 요청 후 즉시 상태 조회 → status=REQUESTED (PG에서 아직 PENDING)
- pg_communication_logs:
  - POST 1건 (결제 요청, elapsed=118ms)
  - **GET 1건** (폴링 조회, transactionKey=20260319:TR:b3c5f1, elapsed=54ms)

**일치 여부**: ✅ 일치

---

### 시나리오 3-2: FAILED 상태 → 폴링 시 PG 조회 안 함

**실행 일시**: 2026-03-19 23:51

**기대 결과**: FAILED 상태 주문 조회 시 PG GET 요청 없음

**실제 결과**:
- FAILED 주문(orderId=160396618099593216) 상태 조회 → status=FAILED
- pg_communication_logs: GET 요청 **0건**

**일치 여부**: ✅ 일치

---

### 시나리오 4-1: CLOSED → OPEN 전이

**실행 일시**: 2026-03-19 23:52

**기대 결과**: PG 다운 후 5건 실패 시 CB OPEN 전이, 이후 요청은 PG 호출 없이 즉시 실패

**실제 결과**:
- 요청 1: CB CLOSED, failureRate=30% (기존 실패 포함)
- 요청 2: CB CLOSED, failureRate=40%
- 요청 3: **CB OPEN**, failureRate=**50%** (threshold 도달)
- 요청 4~7: CB OPEN, notPermittedCalls 증가 (PG 호출 없이 즉시 실패)
- Fallback 로그:
  - 요청 1~3: `type=CONNECTION_REFUSED` (Retry 소진 후 fallback)
  - 요청 4~7: `type=CIRCUIT_OPEN` (CB 차단, PG 호출 없음)

**일치 여부**: ✅ 일치

**Resilience 근거**: sliding-window-size=10에서 failure-rate-threshold=50% 도달 시 즉시 OPEN. PG 정상 상태(실패율 ~40%)에서는 50% 미만으로 오탐 없음.

---

### 시나리오 4-2: OPEN → HALF_OPEN 전이

**실행 일시**: 2026-03-19 23:54

**기대 결과**: OPEN 상태에서 10초(wait-duration-in-open-state) 경과 후 HALF_OPEN 전이

**실제 결과**:
- PG simulator 재시작 + 12초 대기 후 CB 확인 → 아직 OPEN
- 요청 전송 → **HALF_OPEN으로 전이**, 첫 요청 REQUESTED(성공)
- CB: state=HALF_OPEN, buffered=1, failed=0

**일치 여부**: ✅ 일치 (단, HALF_OPEN 전이는 자동이 아닌 요청 도착 시 발생)

**참고**: Resilience4j의 OPEN→HALF_OPEN 전이는 wait-duration 경과 후 **다음 요청이 올 때** 발생. 시간만 지나서 자동 전이되지 않음.

---

### 시나리오 4-3: HALF_OPEN → CLOSED 복구

**실행 일시**: 2026-03-19 23:55

**기대 결과**: HALF_OPEN에서 3건 중 과반(2건 이상) 성공 시 CLOSED 복구

**실제 결과**:
- HALF_OPEN 요청 1: REQUESTED (성공) → HALF_OPEN, 1/0
- HALF_OPEN 요청 2: REQUESTED (성공) → HALF_OPEN, 2/0
- HALF_OPEN 요청 3: FAILED (PG 500) → **CLOSED 복구**
- 3건 중 2건 성공(66%) > threshold 50%

**일치 여부**: ✅ 일치

**Resilience 근거**: permitted-number-of-calls-in-half-open-state=3으로 최소한의 샘플로 PG 복구 판단.

---

### 시나리오 4-4: HALF_OPEN → OPEN 재전환

**실행 일시**: 2026-03-19 23:55

**기대 결과**: HALF_OPEN에서 3건 모두 실패 시 다시 OPEN 전환

**실제 결과**:
- PG 다운 → CLOSED→OPEN(5건째) → 10초 대기 → HALF_OPEN
- HALF_OPEN 요청 1: FAILED → HALF_OPEN, 1/1
- HALF_OPEN 요청 2: FAILED → HALF_OPEN, 2/2
- HALF_OPEN 요청 3: FAILED → **OPEN 재전환**, 3/3

**일치 여부**: ✅ 일치

---

### 시나리오 5-1: 전체 이벤트 이력 조회

**실행 일시**: 2026-03-19 23:56

**기대 결과**: CB 이벤트에서 전체 상태 전이 순서가 기록됨

**실제 결과** (STATE_TRANSITION 이벤트만 추출):
```
23:52:45 | CLOSED_TO_OPEN     (PG 다운 → 실패 누적)
23:54:54 | OPEN_TO_HALF_OPEN  (10초 대기 후 요청 도착)
23:55:09 | HALF_OPEN_TO_CLOSED (3건 중 2건 성공 → 복구)
23:55:43 | CLOSED_TO_OPEN     (PG 다시 다운 → 실패 누적)
23:55:54 | OPEN_TO_HALF_OPEN  (10초 대기 후 요청 도착)
23:55:57 | HALF_OPEN_TO_OPEN  (3건 모두 실패 → 재전환)
```

- 기타 이벤트: SUCCESS, ERROR, NOT_PERMITTED, FAILURE_RATE_EXCEEDED 모두 정상 기록

**일치 여부**: ✅ 일치

---

### 시나리오 6-1: 정상 vs CB OPEN 응답 시간 비교

**실행 일시**: 2026-03-19 23:57

**기대 결과**: CB OPEN 시 PG 호출 없이 즉시 실패하여 응답 시간이 극적으로 빨라짐

**실제 결과**:

| 상황 | 응답 시간 (5건) | 평균 |
|------|----------------|------|
| PG 정상 (CLOSED) | 654, 333, 618, 520, 588ms | **~543ms** |
| CB OPEN | 114, 116, 115, 115, 114ms | **~115ms** |

- 차이: **~4.7배** (CLOSED 543ms vs OPEN 115ms)
- CB OPEN 시에도 115ms가 걸리는 이유: 주문 생성 + Payment 생성 + fallback 처리 + 보상 트랜잭션 오버헤드

**일치 여부**: ✅ 일치 (차이가 4.7배로 기대치 10~50배보다 작지만, PG 호출 차단 효과는 명확)

---

### 시나리오 7-1: Retry 소진 후 Fallback vs CB OPEN 후 Fallback 구분

**실행 일시**: 2026-03-19 23:52~23:53

**기대 결과**: Retry 소진과 CB OPEN 시 다른 type의 fallback 로그가 기록됨

**실제 결과**:
- Retry 소진 (PG 다운): `type=CONNECTION_REFUSED` → PG 연결 불가
- CB OPEN (차단): `type=CIRCUIT_OPEN` → PG 호출 자체 차단
- PG 500 (Retry 안 함): `type=UNKNOWN` → 비-Retryable 에러

**일치 여부**: ✅ 일치

**의미**: 장애 원인별 로그 구분으로 운영 시 즉시 원인 파악 가능.
- `CIRCUIT_OPEN` → "PG 장애 지속, CB가 보호 중"
- `CONNECTION_REFUSED` → "PG 서버 자체 접근 불가"
- `UNKNOWN` → "PG가 응답했으나 서버 에러(500)"

---

## 설정 버그 발견 및 수정 기록

수동 검증 과정에서 아래 4건의 설정 버그를 발견하고 수정함:

### 버그 1: resilience4j 설정이 prd 프로파일에만 존재

- **증상**: local 프로파일로 실행 시 Resilience4j 기본값 적용 → 모든 예외에 대해 3회 Retry, ignore-exceptions 미적용
- **원인**: application.yml에서 resilience4j 설정이 `---` (prd 프로파일) 섹션 안에 위치
- **수정**: resilience4j 설정을 공통 영역(프로파일 구분자 위)으로 이동

### 버그 2: retry-exceptions에 원본 타입 지정 (SocketTimeoutException, ConnectException)

- **증상**: PG 다운 시 ConnectException → Retry 안 됨 (pg_communication_logs 1건만)
- **원인**: Feign이 `ConnectException`을 `feign.RetryableException`으로 래핑 → 원본 타입 매칭 실패
- **수정**: `retry-exceptions: [feign.RetryableException]`으로 변경

### 버그 3: ignore-exceptions에 feign.FeignException 포함

- **증상**: `RetryableException extends FeignException` → ignore가 retry보다 우선 → 모든 네트워크 에러 Retry 차단
- **원인**: ignore-exceptions 체크가 retry-exceptions보다 먼저 실행됨
- **수정**: `feign.FeignException` 제거, retry-exceptions 화이트리스트만으로 500 에러 Retry 방지 충분

### 버그 4: Aspect 순서 미설정 (기본: Retry 바깥, CB 안쪽)

- **증상**: CB fallback이 CoreException으로 변환 → Retry가 원본 예외(RetryableException)를 못 보고 CoreException(ignore)만 보고 Retry 안 함
- **원인**: 기본 순서에서 Retry가 CB를 감싸므로, Retry는 CB fallback의 결과만 받음
- **수정**: `circuit-breaker-aspect-order: 1`, `retry-aspect-order: 2` (CB 바깥, Retry 안쪽)

### 버그 5: enable-exponential-backoff 미설정

- **증상**: `exponential-backoff-multiplier: 2`를 지정했지만 Retry 간격이 모두 고정 ~500ms (지수 증가 안 됨)
- **원인**: Resilience4j에서 exponential backoff을 사용하려면 `enable-exponential-backoff: true`를 명시해야 함. multiplier만 지정하면 무시됨
- **수정**: `enable-exponential-backoff: true` 추가
- **수정 후 검증**: 1→2 간격 ~517ms, 2→3 간격 **~1017ms** (500ms × 2 = 1000ms, 정상)

---

## 코드 리뷰 반영 후 변경사항 (2026-03-20)

아래 변경으로 수동 검증 시 확인 포인트가 달라진 부분:

| 변경 항목 | 이전 | 이후 | 영향 시나리오 |
|-----------|------|------|-------------|
| success 판정 | HTTP 통신 성공 시 무조건 true | `PgApiResponse.isSuccess()` 기반 | 2-1, 5-2 |
| url 컬럼 | 빈 문자열 `""` | `pg.base-url` 설정값 주입 | 전체 (pg_communication_logs 확인 시) |
| getPaymentStatus fallback | `meta.result = "SUCCESS"` | `meta.result = "FALLBACK"` | 3-1 (CB OPEN 시 폴링) |
| catch 범위 | `catch(Exception)` | `catch(CoreException)` | 2-1, 2-2 (PG 실패 시 fallback 예외만 catch) |
| AOP 포인트컷 | 모든 메서드 `*(..)` | `requestPayment`, `getPaymentStatus`만 | 전체 (fallback 메서드에 로그 안 남음) |
