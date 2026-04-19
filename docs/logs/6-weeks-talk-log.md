# 6주차 대화 로그

## 2026-03-16: PG 통신 로그를 DB에 저장할 때 AOP vs Feign Interceptor 중 어떤 방식이 적합한가

### 고민한 부분
- PG 통신 시 request/response를 별도 테이블에 저장하려는데, AOP 기반으로 캡처할지 Feign Interceptor 기반으로 캡처할지 고민

### 선택지
1. **AOP** — Service/Facade 메서드 레벨에서 파라미터/리턴값(Java 객체)을 캡처. 어노테이션 하나로 간단 적용 가능하지만, 직렬화 전 객체라 실제 wire 데이터와 다를 수 있고, HTTP 메타데이터(URL, status code, headers) 접근 불가. 스코프가 넓어 의도치 않은 메서드에도 적용될 수 있음
2. **Feign Interceptor (Logger)** — HTTP 클라이언트 레벨에서 실제 request/response(raw body, headers, status code)를 캡처. PG Feign Client에만 한정되어 스코프가 명확하고, 네트워크 에러도 ErrorDecoder로 캡처 가능. 재시도 시 각 HTTP call 단위로 자연스럽게 분리됨

### 선택한 답
- 2번 Feign Logger/Interceptor 방식
- 이유: PG 로그의 핵심은 "실제 전송된 데이터"이므로 통신 레이어에서 잡는 것이 원칙. AOP는 직렬화 전 객체를 캡처하므로 장애 대응 시 신뢰할 수 없고, HTTP 메타데이터 접근이 안 됨. Feign 레벨이 스코프도 명확하고 데이터 정확성도 보장됨

### 느낀 점
- "로그를 어디서 잡느냐"가 데이터 정확성을 좌우한다는 걸 느꼈을 것. 비즈니스 로직 레벨이 아니라 실제 통신이 일어나는 레이어에서 잡아야 PG 장애 대응 시 신뢰할 수 있는 근거가 된다
- AOP가 범용적이라 습관적으로 선택하기 쉽지만, 목적에 맞는 레이어를 고르는 게 더 중요하다는 점을 체감했을 것

---

## 2026-03-16: 주문번호(orderNo) 설계 — PG용 별도 키 vs 통합 키

### 고민한 부분
- PG 시뮬레이터의 orderId가 6자 이상 문자열을 요구하는데, 우리 Order.id는 auto increment Long (1, 2, 3...)이라 조건 미충족
- 주문번호를 새로 만들어야 하는데, Order.id와 별개의 `orderNo`를 만들 것인지, 아니면 통합할 것인지

### 선택지
1. **PG 전용 orderNo 추가** — Order.id(Long)는 그대로 두고, PG 전송 시에만 쓰는 orderNo 필드를 별도 추가. 기존 API에 영향 없음. 단, 내부에서는 id, PG에서는 orderNo, 나중에 ERP 정산에서는 또 다른 키를 쓰게 되어 시스템 간 매핑이 복잡해짐
2. **orderNo를 대표 비즈니스 키로 통합** — Order.id는 JPA 내부/FK 용으로만 남기고, API 응답·PG 연동·정산 모두 orderNo로 통일. 기존 API 응답 변경 필요하지만, 모든 외부 시스템이 같은 키로 매칭됨
3. **Order.id 자체를 문자열 PK로 변경** — 가장 깔끔하지만 기존 FK 관계, JPA 설정, 전체 테스트에 대규모 변경 필요

### 선택한 답
- 2번: orderNo를 대표 비즈니스 키로 통합
- 형식: `yyyyMMdd-{Order.id 6자리 패딩}` (예: 20260316-000001)
- 생성: Order save() 후 id 확정 시 assignOrderNo() 호출. DB auto increment를 시퀀스로 활용하므로 별도 채번 테이블 불필요
- 이유: 실무에서 정산할 때 우리 DB의 주문번호 = PG의 orderId = ERP의 주문번호가 1:1로 매칭되어야 조회·대사·환불 처리가 수월함. 별도 키를 두면 "이 PG 거래가 우리 시스템의 어떤 주문인지" 찾을 때 항상 매핑 테이블을 거쳐야 하는 번거로움 발생

### 추가 고민: 주문번호 채번 방식
- 헥토 프로젝트에서는 `yyyyMMddHHmmss + 5자리 시퀀스` 방식 사용 (별도 order_sequence 테이블 + 매일 TRUNCATE)
- 처음에는 "Order.id(auto increment)에 날짜를 붙여서 orderNo 필드를 추가하자"는 방향이었으나, 그러면 id와 orderNo 두 개의 식별자를 관리해야 함
- 최종 결정: **Order.id 자체를 `yyyyMMdd + 6자리 시퀀스` Long으로 직접 채번**
  - auto increment를 포기하고, Redis INCR로 일별 시퀀스를 원자적으로 생성
  - `INCR order:seq:20260316` → seq 획득 → `20260316 * 1_000_000 + seq` = `20260316000001`
  - TTL 2일로 설정하면 별도 TRUNCATE 불필요
  - 별도 orderNo 필드가 필요 없고, 기존 API/DTO/테스트 변경도 최소화

### 멘토링 후 방향 전환: Snowflake ID 채택
- 멘토님 피드백: "auto increment는 식별자로서 한계가 있고, 스노우플레이크/TUID 등 시간 기반 분산 ID 생성기를 권장"
- `yyyyMMdd + 시퀀스` 방식은 시퀀스 채번을 위해 Redis나 DB 시퀀스 테이블이 필요했는데, Snowflake ID는 **추가 인프라 없이 애플리케이션에서 독립적으로 생성** 가능
- 64비트 Long: 타임스탬프(41bit) + 머신ID(10bit) + 시퀀스(12bit) → ms당 4096건, 약 69년 사용 가능
- Long 타입 유지 → 기존 코드 변경 최소, PG 19자리로 6자 이상 조건 충족, 시간순 거친 정렬 가능

### 느낀 점
- 식별자 설계는 단순히 "유니크한 값 만들기"가 아니라, "이 값이 어디까지 흘러가느냐"를 먼저 생각해야 함. PG, 정산, 고객 문의 대응까지 같은 키로 추적할 수 있으면 운영 비용이 크게 줄어듦
- 처음에는 `yyyyMMdd + 시퀀스`를 직접 만들려고 했는데, 채번 방식(Redis vs DB)에서 계속 돌았음. 멘토님이 Snowflake를 추천해주셨고, 실제로 보니 추가 인프라 없이 해결되는 가장 깔끔한 방식이었음
- "바퀴를 재발명하지 말 것" — ID 생성은 이미 잘 풀린 문제(Snowflake, TSID 등)가 있으니 가져다 쓰는 게 맞음

---

## 2026-03-16: 트랜잭션 경계 분리 — PG 호출을 트랜잭션 안에 둘 것인가

### 고민한 부분
- PaymentFacade.requestPayment()에서 Payment 생성과 PG 호출이 하나의 @Transactional 안에 있으면, PG 타임아웃 시 트랜잭션 전체가 롤백되어 Payment 레코드가 사라짐
- PG는 실제로 결제를 처리했을 수 있는데 우리 DB에 기록이 없으면 콜백이 와도 매칭 불가

### 선택지
1. **하나의 트랜잭션** — 단순하지만, PG 타임아웃 시 Payment 롤백 → 콜백 수신 불가 → 데이터 불일치
2. **트랜잭션 2단계 분리** — 트랜잭션1(Payment 생성 커밋) → 트랜잭션2(PG 호출 + transactionKey 저장). PG 실패해도 Payment는 DB에 남아있어 복구 가능

### 선택한 답
- 2번: 트랜잭션 분리
- 트랜잭션 1에서 Payment(REQUESTED)를 확정 커밋한 뒤, 트랜잭션 2에서 PG를 호출
- PG 타임아웃 시 Payment는 REQUESTED + transactionKey=null 상태로 남음 → 이후 PG orderId 기반 조회로 복구 가능

### 느낀 점
- 외부 시스템 호출을 트랜잭션 안에 넣으면 "내부 상태의 원자성"은 보장되지만 "외부와의 정합성"이 깨질 수 있음
- 트랜잭션 경계를 어디에 두느냐가 장애 복구 가능성을 결정한다는 걸 느꼈을 것

---

## 2026-03-18: Retry 대상을 500 에러 포함 vs 타임아웃만으로 제한할 것인가

### 고민한 부분
- Resilience4j Retry 설정에서 PG 500 에러도 재시도할지, 타임아웃만 재시도할지 고민
- PG 시뮬레이터는 40% 확률로 500을 반환하므로 재시도하면 성공 확률이 올라감 (3회 시도 시 93.6%)
- 하지만 실무에서 500은 "서버 문제"이므로 재시도해도 같은 에러일 가능성이 높음

### 선택지
1. **500도 Retry** — PG 시뮬레이터 환경에서는 효과적 (랜덤 실패이므로). 최종 실패율 6.4%
2. **타임아웃만 Retry** — 실무 관점에서 정확. 500은 서버 문제이므로 재시도 의미 없음. 불필요한 PG 부하 방지

### 선택한 답
- 2번: 타임아웃(SocketTimeoutException, ConnectException)만 Retry
- 500(FeignException)은 재시도 없이 즉시 실패 처리 → CircuitBreaker에 카운트
- 6주차 멘토링에서 안유진님의 "에러 타입별 재시도 대상 구분" 질문에 대해 멘토님이 "재시도는 네트워크 오류 등 일시적 문제에 한정"이라고 답변한 것과 일치

### 느낀 점
- PG 시뮬레이터의 특성(랜덤 40% 실패)에 맞추면 500도 재시도하는 게 합리적이지만, 이번 과제의 목적이 "실무 Resilience 설계 학습"이므로 실무 관점을 따르는 게 맞음
- "재시도할 수 있는 에러"와 "재시도해도 의미 없는 에러"를 구분하는 게 Retry 설계의 핵심

---

## 2026-03-19: 결제 실패 시 보상 로직 — 재고/쿠폰 복원 방식

### 고민한 부분
- 결제가 최종 실패하면 주문 생성 시 차감했던 재고와 사용한 쿠폰을 복원해야 함
- 처음에 비관적락으로 재고를 조회 후 복원하려 했는데, connection pool 고갈 위험이 지적됨
- 기존 쿠폰에 `@Version` 낙관적 락이 걸려있는데, native SQL atomic update를 쓰면 version 체크가 무시됨

### 선택지
1. **둘 다 atomic update** — 재고: `stock = stock + ?`, 쿠폰: `UPDATE WHERE status='USED'`. 단순하지만 쿠폰의 `@Version`을 우회
2. **재고는 atomic update, 쿠폰은 엔티티 기반** — 재고는 단순 증가라 atomic이 적합, 쿠폰은 기존 낙관적 락 유지
3. **둘 다 엔티티 기반** — 일관성 있지만 재고 복원에 비관적락 사용 필요

### 선택한 답
- 2번: 재고는 atomic update, 쿠폰은 엔티티 기반 `restore()`
- 재고: `UPDATE products SET stock = stock + ? WHERE id = ?` — DB 레벨에서 원자적, 동시성 안전
- 쿠폰: 엔티티 조회 → `restore()` → dirty checking — `@Version`이 동시성 보호, 기존 시스템과 일관성 유지
- 비관적락으로 재고를 조회하는 동안 다른 주문의 재고 차감이 블로킹되면 connection pool이 빠르게 고갈될 수 있음

### 느낀 점
- 같은 "복원"이라도 재고와 쿠폰의 동시성 특성이 다르므로 한 가지 방식으로 통일하는 게 항상 정답은 아님
- 기존 시스템에 이미 적용된 락 전략(@Version 등)을 무시하는 코드를 추가하면 "왜 락을 걸었는지" 의미가 희석됨

---

## 2026-03-19: CircuitBreaker 인스턴스 분리 — POST/GET을 나눌 것인가

### 고민한 부분
- 결제 요청(POST)과 상태 조회(GET)가 같은 `pgCircuit` CircuitBreaker를 공유하고 있음
- POST 장애 시 서킷이 열리면 GET(폴링 복구)까지 막혀서 콜백 미도착 시 복구가 불가능해질 수 있음

### 선택지
1. **분리** — `pg-request`(POST)와 `pg-query`(GET) 별도 인스턴스. POST 장애 시에도 GET 폴링 가능
2. **공유 유지** — PG가 장애면 어차피 POST/GET 둘 다 안 됨. 코드 단순

### 선택한 답
- 2번: 현재 구조(공유) 유지
- PG 서버가 POST만 죽고 GET은 살아있는 상황이 현실적으로 거의 없음
- 이번 과제 핵심은 Resilience 패턴 이해이지 인스턴스 세밀한 분리가 아님
- 코드 단순성 우선

### 느낀 점
- "분리하면 더 안전하다"는 맞지만, 실질적인 이득이 없는 분리는 복잡도만 올림
- 과제 범위와 핵심을 벗어나지 않는 판단이 중요

---

## 2026-03-19: 콜백이 60초 걸리면 우리 서버에 영향이 있는가?

### 고민한 부분
- PG 콜백이 비정상적으로 늦게 (예: 60초) 도착하면 우리 시스템에 문제가 생기는지 궁금
- 테스트에서 콜백 도착까지 약 5초 걸린 것을 확인한 후, 만약 이게 훨씬 길어지면 어떻게 되는지

### 내용 정리
- **서버에 직접적인 영향 없음** — 결제 요청 API는 REQUESTED 응답 후 즉시 스레드를 반환
- 콜백은 PG가 새로운 HTTP 요청으로 보내는 것이라 별도 스레드에서 처리
- 60초 동안 우리 서버는 아무것도 대기하지 않음 (비동기의 핵심)
- **영향이 있는 건 사용자 경험** — "결제 처리 중" 화면에서 60초 대기, 클라이언트 폴링으로 상태 확인 필요
- 동기 방식이었다면 60초 동안 스레드가 점유되어 다른 사용자의 요청이 처리 안 됐을 것

### 느낀 점
- 비동기 결제의 핵심 이점이 바로 이것: "외부 시스템의 처리 시간이 우리 서버의 스레드 점유와 분리된다"
- 동기 vs 비동기의 차이를 콜백 지연 시나리오로 체감할 수 있었음

---

## 2026-03-19: 수동 검증에서 발견한 버그 2건 — 설정 누락 + 예외 catch 범위

### 고민한 부분
- pg-simulator 연동 수동 테스트 중 PG 500 에러 시 기대와 다른 동작 발견
- "Retry 안 해야 하는데 Retry 됨" + "Payment가 FAILED로 안 바뀜"

### 발견한 버그

**버그 1: Retry가 500 에러에서도 동작**
- 원인: `application.yml`의 `ignore-exceptions`에 `feign.FeignException` 누락
- 계획에서는 포함시켰지만 실제 코드에 반영 안 됨
- 결과: PG 500 에러 시 불필요한 재시도 3회 발생 → pg_communication_logs에 로그 3건 기록
- 수정: `ignore-exceptions`에 `feign.FeignException` 추가

**버그 2: PG 호출 실패 시 Payment가 INITIATED 상태로 남음**
- 원인: `PaymentFacade.callPgAndUpdatePayment()`에서 `catch(e: CoreException)`만 잡고, `FeignException`은 uncaught
- 결과: FeignException이 catch 안 됨 → `payment.markFailed()` 미호출 → Payment=INITIATED, Order=PENDING 그대로 → 보상 처리도 안 됨
- 수정: `catch(e: Exception)`으로 변경하여 모든 예외에서 실패 처리 + 보상 실행

### 느낀 점
- 단위 테스트에서는 PgPaymentClient를 Mock으로 처리했기 때문에 FeignException이 실제로 throw되는 상황을 테스트하지 못했음
- "계획에 적어놓고 코드에 반영 안 한" 실수 → 설정 파일은 코드보다 검증이 어려워서 놓치기 쉬움
- **수동 검증(눈으로 보는 테스트)의 가치**: 자동 테스트에서 Mock으로 가린 부분을 실제 연동에서 발견할 수 있음
- 멘토님이 "눈으로 보는 테스트가 중요하다"고 강조한 이유를 체감

---

## 2026-03-19: PG 실패 시 Payment가 INITIATED로 남는 버그 — 트랜잭션 롤백 문제

### 고민한 부분
- 버그 수정(`catch(e: Exception)`) 후에도 PG 500 실패 시 Payment가 여전히 INITIATED 상태로 남음
- `payment.markFailed()`를 호출했는데 왜 DB에 반영이 안 되는가?

### 원인 분석
- `TransactionTemplate.execute` 블록 안에서 `payment.markFailed()` 호출 후 예외를 throw
- 예외가 발생하면 TransactionTemplate이 **트랜잭션 전체를 롤백**
- `markFailed()`의 변경도 같은 트랜잭션이므로 함께 롤백됨
- 결과: Payment는 INITIATED 상태 그대로, 보상 처리도 롤백됨

```
TransactionTemplate.execute {
  payment.markFailed()   ← dirty checking으로 변경 예약
  compensateOrder()      ← 재고/쿠폰 복원 예약
  throw FeignException   ← 트랜잭션 롤백 → 위 변경 모두 취소!
}
```

### 선택지
1. **catch 안에서 throw 안 함** — 실패 처리 후 결과를 반환하고 Fallback 응답은 상위에서 처리
2. **실패 처리를 별도 트랜잭션으로 분리** — 새 TransactionTemplate(REQUIRES_NEW)로 markFailed + 보상 처리
3. **catch 안에서 실패 처리 후 다른 예외로 래핑** — 하지만 근본 원인(같은 트랜잭션)은 해결 안 됨

### 선택한 답
- 1번: catch 안에서 예외를 throw하지 않고, 실패 상태를 반환. 상위에서 SERVICE_UNAVAILABLE 응답 처리
- 이유: PG 호출 실패는 "에러"가 아니라 "결제 실패라는 비즈니스 결과"로 처리하는 게 맞음

### 느낀 점
- 트랜잭션 안에서 "상태 변경 + 예외 throw"를 같이 하면 상태 변경이 롤백되는 함정
- 외부 시스템 호출 실패는 예외로 전파하기보다 결과값으로 처리하는 게 트랜잭션 관리에 안전
- 이 버그는 Mock 기반 단위 테스트에서는 절대 발견 못 함 — 실제 DB + 트랜잭션이 있어야 보임

---

## 2026-03-19: Feign 자체 Retryer가 Resilience4j와 별개로 동작하는 문제

### 고민한 부분
- PG 500 에러 시 pg_communication_logs에 같은 orderId로 3건이 기록됨
- Resilience4j Retry 설정에서 500은 재시도 안 하도록 했는데 왜 3번 호출되는가?

### 원인 분석
- Feign에는 자체 Retryer가 있음 (Resilience4j와 별개)
- PgClientConfig에서 `Retryer.NEVER_RETRY`를 명시하지 않으면 기본 Retryer가 활성화될 수 있음
- Feign 파이프라인: Client.execute() → ErrorDecoder (500→FeignException) → Feign Retryer
- Resilience4j 파이프라인: @Retry → @CircuitBreaker → PgClient (Feign)
- 두 Retry가 중첩되어 동작: Resilience4j가 1회 호출 → Feign 내부에서 3회 재시도
- PgLoggingClient는 Client 레벨이라 Feign 내부 재시도도 전부 기록

### 추가 발견
- PgLoggingClient가 HTTP 500 응답도 `success=true`로 기록하고 있었음
- HTTP status 기준이 아니라 "예외 발생 여부"만으로 success를 판단했기 때문
- 로깅 클라이언트가 비즈니스에 영향은 주지 않지만, 잘못된 데이터를 남기면 장애 분석 시 혼란

### 수정 내용
1. `PgClientConfig`에 `Retryer.NEVER_RETRY` 빈 등록 → Feign 자체 재시도 비활성화
2. `PgLoggingClient`에서 success 판정을 HTTP status 200~299 기준으로 변경

### 느낀 점
- Resilience4j와 Feign 모두 Retry 기능이 있어서, 명시적으로 하나를 끄지 않으면 중첩 동작
- "Retry를 Resilience4j에서 제어한다"면 Feign의 Retry는 반드시 꺼야 함
- 로깅 레이어가 비즈니스에 영향을 주면 안 된다는 원칙 — success 판정이 잘못되면 장애 분석이 틀어짐

---

## 2026-03-19: PG 통신 로그 방식 변경 — 커스텀 Client에서 AOP로

### 고민한 부분
- 커스텀 Feign Client(PgLoggingClient)가 Feign 내부 파이프라인(ErrorDecoder, Retryer)과 충돌
- Feign Retryer를 NEVER_RETRY로 설정해도 적용이 안 되는 문제
- PgClientConfig에 @Configuration을 붙이면 글로벌 빈으로 등록되어 Feign 자식 컨텍스트에서 override 안 됨

### 선택지
1. **AOP on PgPaymentClient** — Feign 파이프라인 무관, 최종 결과만 기록
2. **Feign Logger 확장** — Feign 공식 확장 포인트, raw HTTP 캡처 가능하지만 파싱 복잡
3. **서비스 레이어에서 직접 저장** — 가장 단순하지만 비즈니스 코드에 로깅 섞임

### 선택한 답
- 1번: AOP
- `@Around("execution(* PgPaymentClient.*(..))")`로 메서드 호출을 가로채서 입력/출력/예외 자동 기록
- Feign 파이프라인과 완전 분리 → Retryer, ErrorDecoder에 영향 없음
- REQUIRES_NEW 트랜잭션 유지
- 단점: raw HTTP가 아닌 DTO 레벨 캡처 — 하지만 장애 분석에는 충분

### 느낀 점
- "HTTP 레벨에서 캡처해야 정확하다"는 생각이 있었지만, 프레임워크 파이프라인을 건드리면 예상치 못한 부작용 발생
- 로깅은 비즈니스에 절대 영향을 주면 안 된다는 원칙을 지키려면, 비즈니스 파이프라인과 분리된 레이어(AOP)에서 처리하는 게 안전
- 처음부터 AOP로 했으면 Feign Retryer 충돌, Client 래핑 문제를 전부 겪지 않았을 것 — 단순한 방법부터 시도하는 게 맞음
