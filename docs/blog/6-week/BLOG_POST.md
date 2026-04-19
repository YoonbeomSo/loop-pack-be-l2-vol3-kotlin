# 직접 테스트해서 찾아낸 Resilience4j 설정값

*PG 결제 연동에서 Timeout, Retry, CircuitBreaker 값을 실측 데이터로 도출한 과정*

## TL;DR

- Resilience4j 설정값은 문서의 예시를 복사하는 게 아니라, 내 시스템에서 직접 측정한 데이터로 도출해야 한다
- PG 정상 응답 ~413ms를 측정해서 readTimeout=3s를 정했고, Retry 3회 총 소요 ~1.78s로 사용자 체감 한계를 확인했다
- CircuitBreaker 상태 전이를 Actuator로 직접 관찰하니, 문서로만 이해했던 CB 동작이 비로소 체감됐다

---

## 1. 왜 설정값에 근거가 필요했나

커머스 프로젝트에서 PG 결제를 연동하게 됐다. 외부 시스템과 통신하는 만큼, 장애에 대비한 Resilience 패턴이 필요했다.

이번에 사용한 PG 시뮬레이터 스펙은 이렇다:

- 40% 확률로 HTTP 500 에러
- 정상 응답도 100~500ms 랜덤 지연
- 결제 결과는 비동기 콜백으로 전달

40%나 500을 뱉는다. 그러니까 이건 "정상인데 가끔 실패하는 PG"가 아니라, 실무에서 장애가 터진 PG를 시뮬레이션한 환경인 거다.

외부 시스템은 언제든 죽을 수 있다. 빠르게 실패하고, 시스템이 자체적으로 복구할 수 있는 구조를 만들어야 한다. 그리고 서킷브레이커 같은 패턴은 문서로 이해하는 것만으로는 부족하고, 실제로 실패율과 상태 변화를 눈으로 확인하며 조정해야 한다.

그래서 Timeout + Retry + CircuitBreaker + Fallback을 조합하기로 했다.

### CB가 없으면 어떻게 되는가

잠깐, 왜 Timeout + Retry만으로는 부족한 걸까? PG가 다운된 상황을 생각해보자.

CB 없이 Retry만 있으면, 요청이 들어올 때마다 PG에 3번씩 시도한다. PG가 죽어있으니 3번 다 실패한다. 그런데 이 과정에서 **스레드가 잡혀있다.** Retry 3회 x 대기 시간 = 약 1.78초 동안 Tomcat 스레드 하나가 점유된다.

동시에 100명이 결제를 시도하면? 100개 스레드가 각각 1.78초씩 PG에 헛짓거리를 한다. Tomcat 스레드 풀이 200개인데, PG 장애 때문에 스레드가 금방 고갈된다. 결제뿐 아니라 **상품 조회, 주문 목록 같은 PG와 무관한 API까지 응답을 못 하게 된다.** 이게 장애 전파다.

CB가 있으면 다르다. 실패율이 임계치를 넘으면 서킷이 열리고, 이후 요청은 PG를 호출하지 않고 즉시 실패 응답을 반환한다. 스레드 점유 시간이 543ms에서 115ms로 줄어든다. "PG가 죽어있다는 걸 이미 아니까, 더 이상 보내지 않는다."

정리하면 이렇다:

- **Timeout**: PG가 느릴 때 무한 대기를 방지
- **Retry**: 일시적 네트워크 실패 시 재시도로 성공 확률 향상
- **CircuitBreaker**: PG 장애가 지속될 때 우리 서버까지 죽는 장애 전파를 차단
- **Fallback**: 실패 시 사용자에게 의미 있는 응답(결제 실패 안내)을 반환

이 네 가지가 조합되어야 "외부 시스템 장애에도 우리 서비스는 살아있는" 구조가 된다.

그런데 문제가 하나 있었다. **각 설정값을 어떤 근거로 정할 것인가?**

`readTimeout=3s`는 왜 3초인가? `max-attempts=3`은 왜 3회인가? `failure-rate-threshold=50`은 왜 50%인가?

문서의 예시값을 그대로 복사하면 쉽겠지만, 그건 "남의 시스템에 맞춘 값"이지 "내 시스템에 맞는 값"이 아니다. 직접 테스트해서 측정 데이터를 근거로 값을 도출하기로 했다.

---

## 2. 테스트 환경 — pg_communication_logs가 눈이 됐다

설정값을 도출하려면 먼저 "PG가 어떻게 동작하는지" 관찰할 수 있어야 한다.

PG는 우리가 통제할 수 있는 부분이 아니다. 아무리 CB/Retry를 잘 설정해도 외부 시스템에서 장애가 나는 건 우리가 어떻게 할 수 없다. PG의 코드를 고칠 수도, 배포를 앞당길 수도, 서버를 재시작할 수도 없다. 통제할 수 없는 외부 시스템에 대해 우리가 할 수 있는 최대한의 대응은 **"무슨 일이 있었는지 정확히 남기는 것"**이라고 생각했다. 장애를 막을 수는 없지만, 장애가 발생했을 때 "언제, 무엇을 보냈고, 어떤 응답을 받았는지"를 증거로 남겨둘 수는 있다.

운영 중에 고객이 "결제가 안 됐다"고 클레임을 넣으면, "이 주문 건이 PG에 요청이 갔는지, 응답이 왔는지, 몇 ms 걸렸는지"를 추적해야 한다. PG사와 장애 원인을 협의할 때도 "우리 쪽 로그에는 14:23:05에 POST 요청을 보냈고, 3초 후 타임아웃"이라는 객관적 근거가 필요하다. 콘솔 로그는 휘발되고 검색이 어렵지만, DB에 구조화해서 남기면 orderId 하나로 해당 결제의 전체 통신 이력을 즉시 조회할 수 있다.

그래서 PG 통신마다 요청/응답을 기록하는 `pg_communication_logs` 테이블을 만들었다. AOP로 Feign Client 호출을 가로채서 HTTP 메서드, URL, 상태 코드, 소요 시간, 에러 메시지를 자동으로 기록한다.

```sql
SELECT method, url, http_status, success, error_message, elapsed, created_at
FROM pg_communication_logs
ORDER BY created_at;
```

운영 관점에서 만든 로그였는데, 개발 과정에서 Resilience4j 설정 검증 도구로도 위력을 발휘했다. Retry 횟수, 재시도 간격, 에러 타입을 로그로 직접 확인할 수 있었고, Actuator의 CircuitBreaker 엔드포인트와 함께 "설정이 의도대로 동작하는지" 검증하는 핵심 수단이 됐다. 이게 없었으면 "Retry가 진짜 3번 도는 건지" 확인할 방법이 없었을 거다. 실제로 이 로그 덕분에 설정 버그 5건을 발견했다 — "PG 다운인데 로그가 1건만 찍힌다 → Retry가 안 되고 있다" 같은 발견이 가능했다.

검증 환경은 commerce-api(8080) + pg-simulator(8082) + MySQL을 로컬에서 띄운 구성이다.

---

## 3. Timeout — "PG가 얼마나 걸리는지" 먼저 재야 한다

### 처음 생각

connectTimeout과 readTimeout을 설정해야 하는데, 처음에는 "connectTimeout=1s, readTimeout=3s 정도면 적당하겠지"라고 감으로 잡았다.

그런데 타임아웃 설정은 서비스 수준 목표(SLO)와 연계해야 한다. 감이 아니라 측정값이 필요했다. 그래서 PG 정상 상태에서 결제 요청을 여러 건 보내고, `pg_communication_logs`의 elapsed를 확인했다.

### 실측

| 측정 항목 | 값 |
|----------|-----|
| 정상 응답 평균 | ~413ms |
| 정상 응답 최대 | 562ms |
| PG 스펙상 지연 범위 | 100~500ms |
| Connection refused 시간 | 2~9ms (PG 다운 시 즉시 반환) |

### 값 도출

| 설정 | 값 | 근거 |
|------|-----|------|
| connectTimeout | 1s | 정상 연결은 수십ms. 1s는 ~20배 여유. 1s 초과면 PG 서버 자체가 문제인 거다 |
| readTimeout | 3s | 정상 응답 최대 562ms의 ~5.3배. 너무 길면 스레드 점유가 늘어난다 |

"감으로 잡은 값"과 결과적으로 같았지만, 이제는 "왜 이 값인지"를 설명할 수 있게 됐다. 정상 응답이 최대 562ms인데 readTimeout을 10s로 잡으면 PG가 느려질 때 불필요하게 오래 기다리는 거고, 1s로 잡으면 정상 요청도 잘라버리는 거니까.

타임아웃 두 개를 분리한 이유도 있다. connectTimeout은 "PG 서버에 접근이 되느냐", readTimeout은 "PG가 응답을 주느냐"를 따로 판단하는 거다. PG가 아예 죽었을 때(Connection refused)는 2~9ms 만에 알 수 있으니, connectTimeout 1s는 넉넉하다. 반면 PG가 살아는 있는데 느린 상황은 readTimeout 3s로 잡아서, 정상 처리 중인 요청을 섣불리 끊지 않도록 했다.

---

## 4. Retry — "몇 번, 얼마 간격으로" 재시도할 것인가

### 어떤 에러에 재시도할 것인가

PG가 일시적으로 안 될 때 재시도를 하되, 어떤 에러에 재시도할지를 구분해야 했다.

재시도는 네트워크 오류 등 일시적 문제에 한정하는 것이 일반적이다. 이를 기반으로 재시도 전략을 정했다:

- **재시도 O**: 타임아웃, 연결 실패 — 일시적 네트워크 문제이므로 다시 시도하면 성공할 수 있다
- **재시도 X**: PG 500 에러 — 서버 내부 문제이므로 재시도해도 같은 결과일 확률이 높다

PG 시뮬레이터는 40% 확률로 500을 반환하니까, 500도 재시도하면 성공 확률이 올라가긴 한다(3회 시도 시 93.6%). 하지만 실무에서 500은 "서버가 고장난 것"이므로 재시도 의미가 없다. 시뮬레이터 특성에 맞추는 게 아니라 실무 관점을 따르기로 했다.

### 재시도가 안전한 이유 — orderId 기반 멱등성

재시도 전략을 정하면서 한 가지 더 고민한 게 있었다. **"같은 결제를 3번 보내면 3번 결제되는 거 아닌가?"**

결론부터 말하면 안전하다. 이유는 두 가지다.

**첫째, ConnectException은 PG에 요청이 아예 도달하지 않은 것이다.** TCP 연결 자체가 실패한 거니까, PG 측에 아무 흔적도 남지 않는다. 몇 번을 재시도해도 중복 결제 위험이 없다. 수동 검증에서 PG를 다운시킨 후 3회 Retry가 발생했을 때, PG DB에 트랜잭션이 0건인 것을 확인했다.

**둘째, 실무 PG는 orderId(가맹점 주문번호) 기반 멱등성을 보장한다.** 우리가 PG에 보내는 요청에는 `orderId`가 포함되어 있다. 실무 PG(토스페이먼츠, NHN 등)는 같은 orderId로 결제 요청이 오면 새 트랜잭션을 만들지 않고 기존 트랜잭션을 반환한다. 그래서 SocketTimeoutException(요청은 갔는데 응답을 못 받은 경우)으로 재시도해도, PG 측에서 중복 결제가 생기지 않는다.

```
1차 요청: orderId=123 → PG 트랜잭션 생성 (txn-abc) → 응답 타임아웃
2차 요청: orderId=123 → PG가 기존 txn-abc 반환 (새로 생성 안 함) → 응답 수신
```

다만 이번 프로젝트의 PG 시뮬레이터는 orderId 기반 멱등성을 보장하지 않아서, 매 요청마다 새 transactionKey를 생성한다. 이건 시뮬레이터의 한계로, 실무에서는 PG가 멱등성을 보장하므로 Retry가 안전하다.

정리하면 Retry가 안전한 조건은 이렇다:
- **ConnectException**: PG에 안 갔으므로 무조건 안전
- **SocketTimeoutException**: PG에 갔을 수 있지만, orderId 멱등성으로 중복 결제 방지
- **PG 500**: 재시도 자체를 안 하므로 해당 없음

### 실측

PG를 다운시킨 상태에서 결제 요청을 보내 Retry가 동작하는지 확인했다.

| 설정 | 측정 | 결과 |
|------|------|------|
| max-attempts: 3 | PG 다운 시 pg_communication_logs | **3건** 기록 (Retry 정상 동작) |
| 총 소요 시간 | 3회 재시도 완료까지 | **~1.78s** (backoff 포함) |
| 사용자 체감 | 결제 화면에서 허용 가능? | 5~10초 한계 이내, **적절** |

PG 500 에러 시에는 로그가 **1건만** 찍혔다. 재시도 없이 즉시 실패하는 것을 확인했다. 의도한 대로다.

### 지수 백오프 간격 측정

재시도 간격도 `pg_communication_logs`의 `created_at`으로 직접 확인했다. 500ms 대기 후 첫 재시도, 그 다음은 500ms x 2 = 1000ms 대기 후 재시도 — 이걸 지수 백오프(exponential backoff)라고 한다. PG가 일시적으로 과부하일 때, 재시도 간격을 점점 벌려서 PG에게 복구 시간을 주는 효과가 있다.

| 시도 | 시각 | 이전 시도와의 간격 |
|------|------|-------------------|
| 1차 | 14:48:45.255 | — |
| 2차 | 14:48:45.772 | ~517ms (500ms) |
| 3차 | 14:48:46.789 | ~1017ms (500ms x 2) |

로그 타임스탬프로 실제로 지수 증가하는 걸 확인했다.

### 최종 Retry 설정

```yaml
retry:
  instances:
    pgRetry:
      max-attempts: 3                      # 총 소요 ~1.78s, 사용자 체감 이내
      wait-duration: 500ms                 # 첫 대기
      enable-exponential-backoff: true     # 지수 백오프 활성화
      exponential-backoff-multiplier: 2    # 500ms → 1000ms
      retry-exceptions:
        - feign.RetryableException         # 타임아웃/연결 실패만 재시도
      ignore-exceptions:
        - com.loopers.support.error.CoreException  # 비즈니스 에러는 무시
```

---

## 5. CircuitBreaker — 직접 눈으로 상태 전이를 봤다

CB 개념은 문서로 이해했다. "실패율이 임계치를 넘으면 서킷이 열리고, 일정 시간 후 다시 닫힌다." 그런데 실감이 안 났다. "실패율 50%"가 실제로 어떻게 동작하는지, OPEN 상태에서 요청이 들어오면 어떻게 되는지, 다시 CLOSED로 복구되는 과정이 어떤 건지.

직접 PG를 끄고 켜면서 Actuator(`/actuator/circuitbreakers`)로 CB 상태 전이를 관찰했다.

### CLOSED → OPEN: 서킷이 열리는 순간

PG를 다운시키고 요청을 반복 전송했다. 매 요청 후 Actuator로 상태를 확인했다.

- 요청 1: CB CLOSED, failureRate=30% (기존에 쌓인 실패 포함)
- 요청 2: CB CLOSED, failureRate=40%
- 요청 3: **CB OPEN**, failureRate=**50%** (threshold 도달!)
- 요청 4~7: CB OPEN — PG 호출 없이 즉시 실패

서킷이 열린 후에는 PG를 아예 호출하지 않는다. `pg_communication_logs`에 로그가 남지 않는 것으로 확인했다. PG가 죽어있는데 계속 요청을 보내봐야 의미가 없으니까, CB가 "이미 장애인 거 알고 있으니 보내지마"라고 차단하는 거다.

### OPEN → HALF_OPEN: "자동"이 아니었다

10초(`wait-duration-in-open-state`) 대기 후 Actuator로 CB 상태를 확인했는데, 아직 OPEN이었다. 어? 10초 지났는데?

요청을 하나 보내니 그제서야 HALF_OPEN으로 전이됐다.

> Resilience4j의 OPEN→HALF_OPEN 전이는 시간만 지나서 자동으로 되는 게 아니라, **wait-duration 경과 후 다음 요청이 올 때** 전이된다.

문서에도 있는 내용이지만, 직접 겪어보니 체감이 다르다. "10초 지났으니 자동으로 HALF_OPEN이겠지"라고 생각하면 함정에 빠진다.

### HALF_OPEN → CLOSED: 복구 판단

PG를 다시 살린 후 HALF_OPEN 상태에서 3건(`permitted-number-of-calls-in-half-open-state`)을 보냈다:

- 요청 1: 성공 (REQUESTED)
- 요청 2: 성공 (REQUESTED)
- 요청 3: 실패 (PG 500 — 40% 확률에 걸림)
- 결과: 3건 중 2건 성공(66%) > threshold 50% → **CLOSED 복구**

3건만으로 "PG가 살아났는지" 판단하는 거다. 과반이 성공하면 복구, 실패하면 다시 OPEN.

### HALF_OPEN → OPEN: 복구 실패

반대로, PG를 다운 유지한 채 HALF_OPEN에서 3건을 보내면?

- 요청 1~3: 모두 실패 → **다시 OPEN 전환**

또 10초 기다렸다가 다음 요청이 와야 HALF_OPEN을 재시도한다.

### 응답 시간 — CB의 실질적 효과

가장 인상적이었던 건 응답 시간 차이였다.

| 상황 | 응답 시간 (5건 평균) | 비고 |
|------|---------------------|------|
| PG 정상 (CLOSED) | ~543ms | PG 네트워크 지연 포함 |
| CB OPEN | ~115ms | PG 호출 없이 즉시 fallback |
| **차이** | **~4.7배** | |

PG가 죽었을 때 매 요청마다 543ms씩 기다리는 게 아니라 115ms로 즉시 실패시킨다. CB OPEN에서도 115ms가 걸리는 이유는 주문 생성 + Payment 생성 + fallback 처리 + 보상 트랜잭션 오버헤드 때문이다. PG 호출 자체는 완전히 차단된다.

이걸 보고 나서 "CB가 왜 필요한지"가 숫자로 납득됐다. 장애 상황에서 스레드 점유가 4.7배 줄어든다는 건, 같은 스레드 풀로 4.7배 더 많은 요청을 처리할 수 있다는 뜻이다.

### CB 이벤트 이력

Actuator의 `circuitbreakerevents`에서 전체 상태 전이 이력을 확인할 수 있었다:

```
23:52:45 | CLOSED_TO_OPEN     (PG 다운 → 실패 누적)
23:54:54 | OPEN_TO_HALF_OPEN  (10초 대기 후 요청 도착)
23:55:09 | HALF_OPEN_TO_CLOSED (3건 중 2건 성공 → 복구)
23:55:43 | CLOSED_TO_OPEN     (PG 다시 다운 → 실패 누적)
23:55:54 | OPEN_TO_HALF_OPEN  (10초 대기 후 요청 도착)
23:55:57 | HALF_OPEN_TO_OPEN  (3건 모두 실패 → 재전환)
```

한눈에 "언제 열렸고, 언제 닫혔고, 몇 건이 차단됐는지"가 보인다.

### CB 설정값 도출

| 설정 | 값 | 실측 근거 |
|------|-----|----------|
| sliding-window-type | COUNT_BASED | PG 결제는 트래픽이 비교적 일정하고 장애 시 빠른 감지가 중요. TIME_BASED는 트래픽 편차가 큰 서비스(낮 폭주, 새벽 한산)에 적합 |
| failure-rate-threshold | 50% | PG 정상 실패율 ~40% < 50%. 10%p 여유로 오탐 없음 |
| sliding-window-size | 10 | 최근 10건 관찰. 트래픽이 적은 프로젝트에 적합 |
| minimum-number-of-calls | 5 | 서버 시작 직후 첫 2건 실패만으로 서킷 열리는 오탐 방지 |
| slow-call-duration-threshold | 3s | readTimeout(3s)과 동일. 에러는 아니지만 느린 응답도 CB가 감지. failureRateThreshold와 독립 판단되어 느린 응답만으로도 서킷이 열릴 수 있다 |
| slow-call-rate-threshold | 50% | 느린 호출이 50% 넘으면 서킷 OPEN. 에러 없이도 PG가 전반적으로 느려지면 차단 |
| wait-duration-in-open-state | 10s | PG 복구 시간 확보 + 사용자 체감 균형 |
| permitted-calls-in-half-open | 3 | 최소 샘플로 복구 판단. 2/3 성공이면 CLOSED |

`failure-rate-threshold=50%`를 정할 때 제일 고민했다. PG 시뮬레이터가 정상 상태에서도 40% 확률로 500을 반환하는데, 이게 CB에 ERROR로 카운트된다. threshold를 40%로 잡으면 정상 상태에서도 서킷이 열려버리고, 60%로 잡으면 진짜 장애인데 감지가 느려진다. 실측한 정상 실패율(~40%)에 10%p 여유를 둬서 50%로 정했다.

실제로 PG 정상 상태에서 CB가 열리지 않는 것을 반복 테스트로 확인했다. 정상에서는 failureRate가 30~40% 사이에서 왔다갔다하며 50%에 도달하지 않는다. 반면 PG가 다운되면 100% 실패로 즉시 OPEN이 된다.

---

## 6. Fallback — 장애 원인을 로그로 구분하기

CB + Retry를 조합하면 Fallback이 호출되는 경로가 여러 가지다. "PG가 다운돼서 Retry 3회를 소진한 건지", "CB가 열려있어서 PG 호출 자체가 차단된 건지"를 구분해야 운영 시 장애 대응이 가능하다.

예외 타입으로 분기하는 코드를 작성했다:

```kotlin
fun requestPaymentFallback(userId: String, request: PgPaymentRequest, t: Throwable) {
    val failureType = when {
        t is CallNotPermittedException -> "CIRCUIT_OPEN"
        t is RetryableException && t.cause is SocketTimeoutException -> "TIMEOUT_EXHAUSTED"
        t is RetryableException && t.cause is ConnectException -> "CONNECTION_REFUSED"
        else -> "UNKNOWN"
    }
    log.warn("PG 결제 요청 fallback: type=$failureType, orderId=${request.orderId}", t)
}
```

실제 테스트에서 로그를 확인하니 구분이 명확했다:

| 상황 | Fallback type | 의미 |
|------|--------------|------|
| Retry 소진 (PG 다운) | `CONNECTION_REFUSED` | PG 서버 접근 불가 |
| Retry 소진 (타임아웃) | `TIMEOUT_EXHAUSTED` | PG 응답 지연 |
| CB OPEN (차단) | `CIRCUIT_OPEN` | PG 장애 지속, CB가 보호 중 |
| PG 500 (Retry 안 함) | `UNKNOWN` | PG가 응답했으나 서버 에러 |

`CIRCUIT_OPEN`이 급증하면 "PG 장애가 지속되고 있다", `CONNECTION_REFUSED`가 간헐적이면 "PG가 불안정하다"로 판단할 수 있다. Fallback이라는 한 단어로 뭉뚱그리면 장애 원인을 알 수 없지만, 타입을 나누니까 대시보드에서 바로 보인다.

---

## 7. 테스트 과정에서 만난 설정 버그 5건

여기까지가 "이상적인 흐름"이었다. 그런데 실제로 pg-simulator를 연동해서 수동 검증을 돌려보니, 설정이 의도대로 동작하지 않는 경우가 5건이나 있었다. 공식 문서대로 설정했는데 안 되는 것들이었다.

### 버그 1: YAML 프로파일 함정

**증상**: local에서 돌렸는데 Retry가 모든 예외에 3번씩 돌아간다.

**원인**: resilience4j 설정이 `---` 구분자 아래 prd 프로파일 섹션에 위치해 있었다. local 프로파일로 실행하면 Resilience4j 기본값이 적용되는데, 기본값은 모든 예외에 재시도한다.

**수정**: resilience4j 설정을 공통 영역(프로파일 구분자 위)으로 이동. Spring YAML에서 `---` 아래에 넣은 설정은 해당 프로파일에서만 로드된다. 코드 리뷰에서 놓치기 가장 쉬운 부분이다.

### 버그 2: Feign의 예외 래핑

**증상**: PG 다운인데 `pg_communication_logs`에 1건만 찍힌다. Retry가 안 되고 있었다.

**원인**: `retry-exceptions`에 `ConnectException`을 넣었는데, Feign이 `ConnectException`을 `feign.RetryableException`으로 래핑해서 던진다. Resilience4j는 정확한 타입 매칭이라, `ConnectException`을 넣어놔도 `RetryableException`과 매칭되지 않는다.

```
실제 발생: java.net.ConnectException
    ↓ Feign 내부에서 래핑
Resilience4j가 받는 예외: feign.RetryableException (타입 매칭 실패!)
```

**수정**: `retry-exceptions: [feign.RetryableException]`으로 변경. 수정 후 PG 다운 시 로그 **3건** 확인. 프레임워크가 예외를 어떻게 변환하는지 알아야 한다.

### 버그 3: ignore-exceptions 우선순위

**증상**: 버그 2를 수정했는데 여전히 Retry가 안 된다.

**원인**: `ignore-exceptions`에 `feign.FeignException`이 있었다. Resilience4j는 ignore를 retry보다 먼저 평가한다. 그런데 `RetryableException extends FeignException`이다. 상위 클래스인 `FeignException`에 먼저 매칭되어 모든 네트워크 에러의 Retry가 차단된 거다.

```
feign.FeignException          ← ignore-exceptions에 있음 (먼저 평가)
  └─ feign.RetryableException  ← retry-exceptions에 있음 (평가 안 됨!)
```

**수정**: `ignore-exceptions`에서 `FeignException` 제거. PG 500(`FeignException.InternalServerError`)은 `retry-exceptions` 화이트리스트(`RetryableException`)에 포함되지 않으니 자연스럽게 재시도 대상에서 빠진다. 별도로 ignore할 필요가 없었다.

### 버그 4: Aspect 순서 — 이게 제일 중요했다

**증상**: Retry 설정을 다 고쳤는데도 PG 다운 시 재시도가 안 된다.

**원인**: Resilience4j 기본 Aspect 순서에서는 Retry가 바깥, CB가 안쪽이다.

```
기본 순서: 요청 → Retry(바깥) → CB(안쪽) → Feign → PG
```

이 순서에서 PG가 다운되면:

1. Feign → PG 호출 → `RetryableException` 발생
2. CB가 받음 → fallback 호출 → `CoreException`을 throw
3. Retry가 받음 → `CoreException`은 ignore-exceptions → **재시도 안 함!**

CB fallback이 원본 예외(`RetryableException`)를 `CoreException`으로 바꿔버리니까, Retry 입장에서는 비즈니스 예외만 보이는 거다.

**올바른 순서**: CB(바깥) → Retry(안쪽)

```
올바른 순서: 요청 → CB(바깥) → Retry(안쪽) → Feign → PG
```

이러면 Retry가 원본 예외(`RetryableException`)를 직접 받아서 3회 재시도한다. Retry를 다 소진한 후의 최종 실패 1건만 CB에 기록된다. Retry 3회가 CB에서 실패 3건이 아니라 **1건**으로 카운트되는 이점도 있다.

**수정**: `circuit-breaker-aspect-order: 1`, `retry-aspect-order: 2` 설정. 숫자가 작을수록 바깥이다.

### 버그 5: exponential-backoff 플래그 누락

**증상**: `exponential-backoff-multiplier: 2`를 설정했는데 재시도 간격이 모두 고정 ~500ms.

**원인**: Resilience4j에서 exponential backoff를 쓰려면 `enable-exponential-backoff: true`를 명시해야 한다. `multiplier`만 지정하면 무시된다. 이건 솔직히 함정이다.

**수정 전 실측**: 1→2 간격 ~513ms, 2→3 간격 ~525ms (고정)
**수정 후 실측**: 1→2 간격 ~517ms, 2→3 간격 **~1017ms** (지수 증가)

---

## 8. 보상 처리 — 실패해도 시스템은 정합해야 한다

설정값 도출과 별개로, 결제 실패 시 보상 처리가 정확히 동작하는지도 DB에서 직접 확인했다.

결제 실패(콜백 FAILED) 시:

- **재고**: 보상 전 93 → 주문 5건(-5) → FAILED 보상(+1) → **89** (4건 PAID 차감만 남음, 정확)
- **쿠폰**: FAILED 건의 쿠폰이 **AVAILABLE**로 복원됨

PG 500 에러로 즉시 실패한 경우에도, PG 다운으로 Retry 3회를 소진한 후 실패한 경우에도, 보상 처리가 정상 동작하는 것을 확인했다. 결제가 실패했는데 재고가 차감된 채로 남아있으면 안 되니까, 이 부분은 반드시 검증해야 했다.

---

## 9. 최종 설정 — 모든 값에 실측 근거가 있다

```yaml
resilience4j:
  circuitbreaker:
    circuit-breaker-aspect-order: 1    # CB가 바깥 (Retry를 감싸는 쪽)
    instances:
      pgCircuit:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10          # 최근 10건 관찰
        minimum-number-of-calls: 5       # 5건 이후부터 실패율 계산
        failure-rate-threshold: 50       # 정상 실패율(~40%) + 10%p 여유
        wait-duration-in-open-state: 10s
        permitted-number-of-calls-in-half-open-state: 3
        slow-call-duration-threshold: 3s
        slow-call-rate-threshold: 50
        register-health-indicator: true
  retry:
    retry-aspect-order: 2              # Retry가 안쪽 (PG 호출을 직접 감싸는 쪽)
    instances:
      pgRetry:
        max-attempts: 3                  # 총 소요 ~1.78s
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - feign.RetryableException     # Feign이 래핑한 타입
        ignore-exceptions:
          - com.loopers.support.error.CoreException
        fail-after-max-attempts: true

feign:
  connectTimeout: 1s   # 정상 연결 수십ms의 ~20배
  readTimeout: 3s      # 정상 응답 ~413ms의 ~6배
```

### 도출 과정 요약

| 설정 | 초기 생각 | 실측 → 최종값 | 근거 |
|------|----------|-------------|------|
| connectTimeout | 1s (감) | 1s (유지) | 정상 연결 수십ms, ~20배 여유 |
| readTimeout | 3s (감) | 3s (유지) | 정상 응답 최대 562ms, ~5.3배 여유 |
| max-attempts | 3 | 3 (유지) | 총 소요 ~1.78s, 사용자 체감 이내 |
| failure-rate-threshold | 50% | 50% (유지) | 정상 실패율 ~40% < 50%, 오탐 없음 |
| retry-exceptions | ConnectException | **feign.RetryableException** | Feign 예외 래핑 발견 |
| ignore-exceptions | CoreException + FeignException | **CoreException만** | 상속 관계로 Retry 차단 발견 |
| Aspect 순서 | 기본 (Retry 바깥) | **CB(1)→Retry(2)** | fallback이 예외 변환하는 문제 발견 |
| exponential-backoff | multiplier만 설정 | **enable: true 추가** | 플래그 없으면 무시됨 발견 |

값 자체가 바뀐 건 Timeout이나 Retry 횟수가 아니라, **어떤 예외를 어떤 순서로 처리하느냐**에 관한 설정들이었다. 초기에 감으로 잡은 숫자값들은 실측으로 확인한 결과 적절했지만, 프레임워크 내부 동작에 대한 설정 4건은 테스트 없이는 절대 발견할 수 없었을 거다.

---

## 10. 돌아보며

처음에는 Resilience4j 공식 문서의 예시값을 복사해서 넣으면 될 줄 알았다.

그런데 직접 pg-simulator를 연동해서 테스트를 돌려보니 세 가지를 깨달았다.

**첫째, 설정값은 내 시스템에서 측정해야 한다.**

readTimeout=3s가 적절한지는 PG의 정상 응답 시간을 재봐야 알 수 있다. failure-rate-threshold=50%가 적절한지는 정상 상태의 에러율을 측정해봐야 알 수 있다. 문서의 예시값은 "이런 설정이 있다"를 알려줄 뿐, "내 시스템에 이 값이 맞다"를 보장하지 않는다.

**둘째, 프레임워크 내부 동작을 이해해야 한다.**

Feign이 예외를 래핑하는 것, Aspect 순서가 동작을 결정적으로 바꾸는 것, ignore가 retry보다 먼저 평가되는 것 — 이런 건 공식 문서의 설정 예시만으로는 알 수 없다. 5건의 설정 버그 중 4건이 이 "프레임워크 내부 동작"에서 비롯됐다.

**셋째, 눈으로 봐야 비로소 이해된다.**

CB 상태 전이를 Actuator로 직접 관찰하기 전에는 "failureRate 50%면 OPEN" 정도의 이해였다. 직접 PG를 끄고 켜면서 CLOSED → OPEN → HALF_OPEN → CLOSED 전이를 보고, 응답 시간이 543ms에서 115ms로 떨어지는 걸 확인하니까 CB의 가치가 체감됐다. "HALF_OPEN 전이가 자동이 아니다"같은 것도 직접 겪어봐야 안다.

서킷브레이커 동작 확인을 위해 눈으로 보는 테스트가 필요하다 — 이게 이번 경험을 관통하는 메시지였다. 설정은 문서에서 시작하되, 검증은 반드시 내 시스템에서 직접 돌려보고 눈으로 확인해야 한다.
