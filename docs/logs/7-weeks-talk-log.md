# 7주차 대화 로그

## 2026-03-24: 선착순 쿠폰 발급 결과 조회 방식 — DB 직접 조회 vs Kafka 콜백 토픽

### 고민한 부분
- 선착순 쿠폰 발급 결과를 사용자가 polling할 때, commerce-api가 어떻게 결과를 알 수 있는가?
- 두 앱이 같은 DB를 공유하므로 commerce-api가 직접 조회하면 간단하지만, MSA 관점에서 적절한가?

### 선택지
1. **commerce-api에서 직접 DB 조회** — 두 앱이 같은 DB를 공유하므로 coupon_issue_requests 테이블을 직접 읽는다. 단순하고 빠르다.
2. **Kafka 콜백 토픽으로 결과 전달** — commerce-streamer가 처리 결과를 coupon-issue-results 토픽에 발행하고, commerce-api가 소비해서 자체 DB에 저장. MSA에 더 가깝지만 복잡도 증가.

### 선택한 답
- 2번 선택
- 이유: MSA 관점에서 서비스 간 DB 직접 접근은 결합도를 높이고, 향후 서비스 분리 시 구조 변경이 필요해진다. Kafka 콜백 토픽을 사용하면 이벤트 기반 비동기 통신의 학습 목적에도 부합한다.

### 느낀 점
- 학습 프로젝트라 DB 공유가 간단한 해법이지만, 실무에서의 확장성을 고려한 설계를 선택했다. "지금 편한 것"보다 "나중에 덜 바꿀 것"을 기준으로 판단하는 연습이 됐다.

---

## 2026-03-24: likeCount 관리 — ProductMetrics만 vs 둘 다 업데이트

### 고민한 부분
- 좋아요 집계를 eventual consistency로 바꿀 때, commerce-api의 Product.likeCount와 commerce-streamer의 ProductMetrics 중 어디서 관리할 것인가?

### 선택지
1. **둘 다 업데이트** — Product.likeCount는 ApplicationEvent 핸들러로 업데이트(Step 1), ProductMetrics는 Kafka Consumer로 업데이트(Step 2). 기존 조회 API 호환성 유지.
2. **ProductMetrics만 사용** — Product.likeCount 업데이트를 제거하고 ProductMetrics로 일원화. 기존 상품 조회 API에서 ProductMetrics를 JOIN해서 읽도록 변경 필요.

### 선택한 답
- 1번 선택
- 이유: 기존 상품 조회 API의 호환성을 유지하면서, Kafka 기반 집계 데이터도 별도로 관리할 수 있다.

### 느낀 점
- 두 데이터의 목적이 다르다는 걸 인식했다. Product.likeCount는 상품 조회 응답용, ProductMetrics는 분석/집계용. 같은 숫자라도 사용 맥락이 다르면 이원화가 자연스럽다.

---

## 2026-03-24: UserActivityEvent 별도 클래스 불필요

### 고민한 부분
- 유저 행동 로깅을 위해 `UserActivityEvent`라는 별도 이벤트 클래스를 만들어야 하는가?

### 내용 정리
- `UserActivityEventHandler`가 이미 `OrderCreatedEvent`, `LikeToggledEvent`, `ProductViewedEvent` 등 각 도메인 이벤트를 직접 수신해서 로깅할 수 있다. 별도 `UserActivityEvent`를 만들면 이벤트 발행이 중복된다.
- 결론: 도메인 이벤트를 그대로 수신하는 핸들러만 있으면 되고, 별도 이벤트 클래스는 제거했다.

### 느낀 점
- 이벤트 설계에서 "하나의 사실에 하나의 이벤트" 원칙을 지키는 게 중요하다. 같은 행위를 두 개의 이벤트로 발행하면 관리 포인트만 늘어난다.

---

## 2026-03-24: ProductViewedEvent 리스너 타입 — @TransactionalEventListener vs @EventListener

### 고민한 부분
- `ProductViewedEvent`는 readOnly 트랜잭션에서 발행되는데, `@TransactionalEventListener(AFTER_COMMIT)`을 써야 하는가?

### 선택지
1. **@TransactionalEventListener(AFTER_COMMIT) + @Async** — 다른 이벤트와 통일. readOnly 트랜잭션도 커밋은 발생하므로 동작은 한다.
2. **@EventListener + @Async** — readOnly 트랜잭션에는 보호할 쓰기가 없으므로 커밋 대기가 무의미. 즉시 비동기 실행.

### 선택한 답
- 2번 선택
- 이유: `AFTER_COMMIT`의 목적은 "쓰기 트랜잭션이 확실히 커밋된 후 부가 로직을 실행"하는 것이다. 조회만 하는 readOnly 트랜잭션에서는 보호할 쓰기가 없으므로 커밋을 기다릴 이유가 없다. 리스너 타입은 통일보다 트랜잭션 성격에 맞게 선택해야 한다.

### 느낀 점
- "통일"이 항상 좋은 건 아니다. 왜 그 어노테이션을 쓰는지 이해하고 상황에 맞게 선택하는 게 핵심이다. readOnly에서 AFTER_COMMIT을 쓰면 동작은 하지만 의도가 맞지 않는 코드가 된다.

---

## 2026-03-24: @Async 스레드 풀 한계 — 조회 이벤트가 폭발하면?

### 고민한 부분
- `ProductViewedEvent`를 `@EventListener` + `@Async`로 처리하면, 조회 요청이 대량으로 들어올 때 비동기 스레드 풀이 감당할 수 있는가?

### 내용 정리
- Spring `@Async`의 기본 설정(`SimpleAsyncTaskExecutor`)은 요청마다 새 스레드를 생성해서 제한이 없다 → OOM 위험
- `ThreadPoolTaskExecutor`로 설정하면 `corePoolSize`, `maxPoolSize`, `queueCapacity`로 제한 가능
  - 예: maxPoolSize=50, queueCapacity=500이면 → 최대 50개 스레드 + 500개 큐 대기
  - 550개 초과 시 `RejectedExecutionException` 발생
- 초당 조회가 수백 건이면 큐가 빠르게 차고, 스레드 풀이 병목이 된다
- **하지만 이건 Step 1의 한계이다.** Step 2에서 Kafka로 전환하면 commerce-api는 Outbox에 INSERT 한 줄만 하고 끝 → 비동기 스레드 풀 문제가 사라진다

### 느낀 점
- ApplicationEvent + @Async만으로는 대량 트래픽에 한계가 있다는 걸 구체적인 숫자로 이해했다. Step 1에서 이벤트 분리의 개념을 익히고, Step 2에서 Kafka로 확장하면서 이 한계를 자연스럽게 해소하는 흐름이 "Why → How → Scale"이라는 이번 주차 학습 방향과 맞닿아 있다.

---

## 2026-03-24: Step 2 Outbox 기록 방식 — 서비스에서 직접 save vs 이벤트 핸들러

### 고민한 부분
- Transactional Outbox Pattern에서 Outbox 레코드를 어떻게 생성할 것인가?
- Step 1에서 만든 서비스 코드를 Step 2에서 다시 수정해야 하는가?

### 선택지
1. **서비스에서 직접 outboxEventRepository.save()** — 명시적이고 테스트하기 쉽지만, Step 2에서 서비스 코드를 다시 수정해야 한다.
2. **OutboxEventHandler(@TransactionalEventListener(BEFORE_COMMIT))** — Step 1의 이벤트에 핸들러만 추가. 서비스 코드 수정 없이 이벤트 기반 확장.

### 선택한 답
- 2번 선택
- 이유: Step 1에서 만든 이벤트 구조를 활용해 핸들러 추가만으로 Kafka 전파를 확장할 수 있다. 이게 이벤트 기반 아키텍처의 본래 의도 — "새 기능은 리스너 추가로 확장"에 부합한다. BEFORE_COMMIT으로 같은 트랜잭션에 포함되어 원자성도 보장된다.

### 느낀 점
- 이벤트 기반 아키텍처의 확장성을 직접 체감했다. Step 1 → Step 2 전환 시 기존 코드를 건드리지 않고 핸들러만 추가하는 게 OCP(개방-폐쇄 원칙)와도 일맥상통한다.

---

## 2026-03-24: readOnly 트랜잭션에 Outbox가 필요한가?

### 고민한 부분
- Step 2에서 `OutboxEventHandler(BEFORE_COMMIT)`로 Outbox를 저장하는데, `ProductViewedEvent`는 readOnly 트랜잭션에서 발행된다. readOnly 안에서 INSERT가 가능한가?

### 내용 정리
- 근본적으로 Outbox 패턴의 목적은 **쓰기 트랜잭션과 이벤트 발행의 원자성 보장**이다
- readOnly 조회에는 보호할 쓰기가 없으므로 Outbox 자체가 불필요
- 결론: 쓰기 이벤트는 Outbox, 조회 이벤트(`ProductViewedEvent`)는 `@EventListener` + `@Async`에서 직접 `KafkaTemplate.send()` 호출

### 느낀 점
- 패턴을 일괄 적용하려다 보면 "왜 이 패턴을 쓰는가?"를 놓치기 쉽다. Outbox는 쓰기의 원자성을 위한 것인데, 읽기에 적용하려 한 건 목적과 수단이 뒤바뀐 것이었다.

---

## 2026-03-24: 멘토링에서 확인할 질문 정리

---

## 2026-03-24: AFTER_COMMIT 핸들러에서 @Async 없이 @Transactional 호출하면?

### 고민한 부분
- `LikeCountEventHandler`가 `@TransactionalEventListener(AFTER_COMMIT)`으로 `productService.incrementLikeCount()`를 호출할 때, 기존 트랜잭션은 이미 커밋된 상태인데 새 트랜잭션이 자동으로 열리는가?

### 내용 정리
- Spring 내부 동작: `AFTER_COMMIT`은 `doCommit()` 이후, `cleanupAfterCompletion()` 이전에 실행
- **`@Async` 있을 때**: 별도 스레드 → 트랜잭션 컨텍스트 없음 → `REQUIRED`가 새 트랜잭션 자동 생성 → 문제없음
- **`@Async` 없을 때**: 같은 스레드 → `TransactionSynchronizationManager`에 트랜잭션 컨텍스트가 아직 남아있음 → `REQUIRED`가 이미 커밋된 트랜잭션에 참여 시도 → 쓰기 미반영 또는 예외 가능

### 선택지
1. **`@Async` 추가** — 비동기 실행, `REQUIRED` 그대로 사용. 실패 시 사용자 모름
2. **`REQUIRES_NEW` 명시** — 동기 실행, 명시적으로 새 트랜잭션. 응답에 업데이트 시간 포함

### 선택한 답
- 1번 선택 (`@Async` 추가)
- 이유: 좋아요 집계를 eventual consistency로 설계한 이상, 동기로 기다릴 이유가 없다. 비동기 실행이 설계 의도에 부합하고, 별도 스레드에서 새 트랜잭션이 자연스럽게 열린다.

### 느낀 점
- `@TransactionalEventListener`와 `@Async`의 조합이 단순한 편의가 아니라 트랜잭션 컨텍스트 관리와 직결된다는 걸 알게 됐다. "AFTER_COMMIT이니까 트랜잭션 끝난 거 아니야?"라고 생각하기 쉽지만, Spring 내부에서는 정리가 아직 안 끝난 상태라 주의가 필요하다.

---

## 2026-03-24: Outbox INSERT 실패 시 비즈니스 로직도 롤백 — 이게 맞나?

### 고민한 부분
- Transactional Outbox Pattern에서 비즈니스 로직과 Outbox INSERT가 같은 트랜잭션이므로, Outbox INSERT 실패 시 비즈니스 로직도 롤백된다
- 이게 의도된 동작인가? Outbox 실패를 무시하고 비즈니스만 성공시켜야 하는 건 아닌가?

### 내용 정리
- Outbox 패턴을 쓰는 이유 자체가 **"비즈니스 상태 변경과 이벤트 발행을 원자적으로 묶겠다"**는 것
- "원자적(Atomic)"이란: 여러 작업이 **하나의 단위로 동작** — 전부 성공하거나 전부 실패, 중간 상태가 없음. 쪼갤 수 없는(atom) 단위라는 뜻
- 가능한 시나리오:
  - 둘 다 성공 → 정상
  - 둘 다 실패 → 재시도하면 됨
  - 비즈니스만 성공, Outbox 실패 → **이벤트 유실** → 주문은 생겼는데 후속 처리(결제, 알림)가 안 됨 → 더 위험
- 결론: Outbox INSERT 실패 시 비즈니스 롤백이 **패턴의 의도에 맞는 동작**이다

### 느낀 점
- "실패하면 전부 롤백"이 손해처럼 보이지만, "절반만 성공"하는 게 훨씬 위험하다. 원자성의 가치는 "중간 상태를 만들지 않는 것"에 있다. 이건 DB 트랜잭션의 ACID에서 A(Atomicity)와 같은 개념이고, Outbox 패턴은 이걸 이벤트 발행까지 확장한 것이다.

---

## 2026-03-24: Kafka 파티션 병목 — 동일 쿠폰에 트래픽이 몰리면?

### 고민한 부분
- 선착순 쿠폰 발급에서 couponId를 파티션 키로 사용하면, 동일 쿠폰 요청이 하나의 파티션에 몰린다
- 파티션을 늘려도 같은 키는 같은 파티션으로 가니까 분산이 안 됨

```
key = couponId=100
Partition 0: [쿠폰100] [쿠폰100] [쿠폰100] [쿠폰100]  ← 몰림
Partition 1: (비어있음)
Partition 2: (비어있음)
```

### 선택지
1. **키를 userId로 변경** — 파티션에 골고루 분산. 같은 쿠폰에 대한 순서 보장이 안 되지만, 쿠폰 발급은 유저별 독립 작업이라 순서가 필요 없음. 중복 방지는 DB unique 제약으로 처리. 가장 현실적.
2. **키에 suffix 붙이기** — `coupon_100_ + (userId % N)`으로 N개 파티션에 분산. 부분적 순서 보장 가능하지만 복잡도 증가.
3. **Consumer 내부 병렬 처리** — 단일 파티션에서 Consumer가 스레드풀로 병렬 처리. Kafka 분산 대신 애플리케이션 레벨에서 해결.

### 내용 정리
- 현재 계획은 couponId를 키로 사용해 순차 처리로 동시성 문제를 해소하는 구조
- 이 방식은 트래픽이 적을 때는 문제없지만, 특정 쿠폰에 대량 요청이 몰리면 파티션 병목 발생
- 실무에서는 보통 1번(userId 키) + DB unique 제약이 가장 현실적
- 우리 프로젝트에서는 학습 목적으로 couponId 키를 유지하되, 이 트레이드오프를 인지하고 있기

### 느낀 점
- 파티션 키 선택이 단순한 설정이 아니라 "순서 보장 vs 분산 처리" 사이의 트레이드오프라는 걸 체감했다. couponId 키는 순서 보장에 유리하지만 병목을 만들고, userId 키는 분산에 유리하지만 순서를 포기한다. 정답은 없고, 비즈니스 요구사항(순서가 필요한가? 중복 방지를 어디서 할 것인가?)에 따라 달라진다.

---

## 2026-03-25: AsyncConfig 스레드 풀 설정값의 의미

### 내용 정리
- `corePoolSize = 5`: 항상 유지되는 기본 스레드 수. 요청 없어도 5개 유지
- `maxPoolSize = 20`: 큐가 꽉 찼을 때 최대로 늘어날 수 있는 스레드 수
- `queueCapacity = 200`: 기본 스레드가 바쁠 때 대기하는 작업 큐 크기
- 동작 순서: 요청 1~5 → core 처리 → 6~205 → 큐 대기 → 206~225 → max까지 확장 → 226~ → 거부(RejectedExecutionException)
- 학습 프로젝트 기준 넉넉한 값, Step 2에서 Kafka 전환하면 부하 자체가 줄어듦

### 느낀 점
- 스레드 풀 설정은 숫자 외우기가 아니라, "요청이 몰렸을 때 어떤 순서로 처리/대기/거부되는지"의 흐름을 이해하는 게 중요하다.

---

## 2026-03-25: Kafka Consumer 설정값 — 처리 특성에 따른 선택 기준

### 고민한 부분
- Kafka Consumer 설정을 어떤 기준으로 잡아야 하는가? AsyncConfig의 스레드 풀처럼 정해진 값이 있는 건지?

### 내용 정리
- Kafka 설정은 **Consumer의 처리 특성**에 따라 달라진다

| 설정 | 메트릭 집계 Consumer | 쿠폰 발급 Consumer | 선택 기준 |
|------|-------------------|------------------|----------|
| 처리 방식 | 배치 (여러 건) | 단건 (한 건씩) | 집계는 모아서 upsert 효율적, 쿠폰은 순서 보장 필수 |
| concurrency | 3 | 1 | 집계는 병렬 OK, 쿠폰은 순차 처리 |
| max.poll.records | 500 | 1 | 집계는 벌크, 쿠폰은 한 건 확실하게 |
| session.timeout | 60s | 30s | 배치는 처리 시간 길 수 있어 넉넉하게 |

- 핵심 기준: **"순서 보장이 필요한가?"** vs **"처리량이 중요한가?"**
  - 순서 필요 → 단건 + 단일 스레드 (쿠폰)
  - 처리량 중요 → 배치 + 병렬 (메트릭)
- KafkaConfig에 각각 다른 `ListenerContainerFactory`로 등록하여 분리

### 느낀 점
- 하나의 Kafka 설정으로 모든 Consumer를 처리하는 게 아니라, 비즈니스 특성에 맞게 Consumer별로 설정을 나누는 것이 중요하다. 이건 AsyncConfig에서 단일 스레드 풀을 쓰는 것과 대비되는 접근 — Kafka에서는 Consumer 특성별 분리가 자연스럽다.

---

## 2026-03-26: Kafka Consumer 메시지 처리 4단계 — 왜 이 순서인가?

### 고민한 부분
- Consumer가 메시지를 수신하고 처리하는 흐름에서, 멱등 체크 → 비즈니스 로직 → ACK 순서가 왜 중요한가?

### 내용 정리
4단계 순서:
1. **메시지 수신** — Kafka 브로커에서 메시지 poll
2. **멱등 체크** — `event_handled` 테이블에서 eventId 중복 확인. 있으면 skip
3. **ProductMetrics upsert** — 비즈니스 로직 실행 + `event_handled`에 처리 완료 기록
4. **Manual ACK** — Kafka에 "처리 끝" 알림 (오프셋 커밋)

핵심은 3번과 4번 사이의 간극:
- 3번(비즈니스 로직) 성공 후, 4번(ACK) 전에 서버가 죽으면?
- ACK 안 했으니 Kafka가 같은 메시지를 다시 보냄
- 재수신 → 2번(멱등 체크)에서 이미 처리됐으니 skip
- 중복 처리 없이 안전하게 복구

이게 "At Least Once 수신 + 멱등 처리 = Exactly Once 효과"

### 느낀 점
- ACK을 비즈니스 로직 전에 보내면 "처리 안 됐는데 메시지 사라짐" (유실), 비즈니스 로직 후에 보내면 "이미 처리했는데 다시 받음" (중복). 중복은 멱등 처리로 해결 가능하지만 유실은 복구 불가. 그래서 ACK은 항상 마지막이어야 한다.

---

## 2026-03-26: 파티션 수를 1개에서 3개로 변경 — 왜?

### 고민한 부분
- 처음에 파티션 1개로 설정했는데, SINGLE_LISTENER(concurrency=1)라면 파티션을 늘려도 1개 스레드만 소비하니 의미 없지 않나?
- 파티션은 줄일 수 없으니 처음 결정이 중요한데, 적정 수는?

### 선택지
1. **파티션 1개** — 현재 concurrency=1이니 딱 맞음. 하지만 나중에 확장하려면 파티션을 추가해야 함.
2. **파티션 3개** — 현재는 1개 스레드만 쓰지만, 향후 concurrency를 3까지 늘릴 수 있는 여유. 실무에서도 최소 3개로 시작하는 게 일반적.

### 선택한 답
- 2번 선택 (파티션 3개)
- 이유:
  - 파티션은 **늘리기만 가능하고 줄일 수 없다** — 기존 메시지 재배치, offset 무효화, key 라우팅 깨짐 때문
  - 실무에서 3대가 기본인 이유: **1대가 죽어도 2대가 살아있으면 서비스 유지** (과반수 투표 원리)
  - Broker 3대 + replicas=3 + min.insync.replicas=2가 최소 비용으로 장애 허용 1대를 확보하는 최적 지점
  - 현재 로컬은 Broker 1대라 replicas=1이지만, 파티션 수는 실무 기준으로 맞춰두면 나중에 변경 불필요

### 느낀 점
- "지금 쓰는 만큼만 설정"이 아니라 "확장 가능성을 열어두는 설정"이 분산 시스템에서는 중요하다. 특히 줄일 수 없는 설정(파티션 수)은 처음에 여유를 둬야 한다. 이건 DB 인덱스 설계와도 비슷 — 나중에 추가는 가능하지만 운영 중 변경은 부하가 크다.

---

## 2026-03-26: Outbox 방식 변경 — BEFORE_COMMIT에서 서비스 직접 INSERT로

### 고민한 부분
- `OutboxEventHandler(@TransactionalEventListener(BEFORE_COMMIT))`로 Outbox를 저장하는 게 서비스 코드를 안 건드려서 좋지만, 디버깅이 어렵고 복잡도가 올라감

### 선택지
1. **BEFORE_COMMIT 핸들러** — 서비스 코드 깔끔, 이벤트 기반 확장. 하지만 흐름을 따라가야 해서 디버깅 어려움.
2. **서비스에서 직접 INSERT** — Outbox가 어디서 저장되는지 코드에서 바로 보임. 명시적.

### 선택한 답
- 2번 선택 (서비스에서 직접 INSERT)
- 이유: 학습 프로젝트에서는 명시적인 게 낫다. 코드를 읽는 사람이 "아, 여기서 Outbox에 저장하는구나"를 바로 알 수 있다.
- OutboxService 헬퍼를 만들어 중복 코드 최소화

### 느낀 점
- "우아한 설계"보다 "읽기 쉬운 코드"가 학습 단계에서는 더 가치 있다. BEFORE_COMMIT은 Spring 내부 동작을 알아야 이해할 수 있지만, 직접 INSERT는 코드만 보면 된다.

---

## 2026-03-26: Kafka Producer 설정 튜닝 — 웹서핑 조사 후 적용

### 고민한 부분
- 현재 kafka.yml 설정이 실무 기준에 맞는지? 초기 설정에서 놓친 것이 있는지?

### 내용 정리
웹서핑으로 조사한 결과, 2가지를 변경:

**1. `retries: 3` → `retries: 2147483647` (Integer.MAX_VALUE)**
- 이전: 3번 재시도 후 포기
- 변경: `delivery.timeout.ms`(120초) 내에서 무한 재시도
- 이유: `enable.idempotence=true`가 재시도 시 중복 저장을 막아주므로, 재시도를 넉넉하게 해도 안전하다. 오히려 3번만 재시도하면 일시적 네트워크 장애에서 불필요하게 실패할 수 있다.
- 핵심: **retries는 "몇 번 시도할지"**, **delivery.timeout.ms는 "최대 몇 초까지 시도할지"** — 둘이 같이 작동해서, 120초 안에 성공하면 OK, 120초 넘으면 실패.

**2. `linger.ms: 5` 추가**
- 이전: 미설정 (기본 0 = 메시지 생기자마자 즉시 전송)
- 변경: 5ms 동안 메시지를 모아서 배치 전송
- 이유: Outbox Publisher가 1초 간격으로 최대 100건을 발행하는데, 각 건을 개별 네트워크 요청으로 보내면 비효율적. 5ms 동안 모아서 한 번에 보내면 네트워크 왕복 횟수가 줄어든다.
- 트레이드오프: 최대 5ms 지연이 추가되지만, Outbox Polling이 이미 1초 간격이라 5ms는 무시할 수준.

**변경하지 않은 것과 이유:**
- `acks=all` — 이미 최적 (메시지 유실 방지)
- `max.in.flight.requests.per.connection=5` — idempotence=true 시 최대값, 이미 최적
- `session.timeout.ms=30000` — 단건 처리에 적합한 범위
- `heartbeat.interval.ms=10000` — session의 1/3 규칙 준수

### 느낀 점
- `retries=3`이 "안전한 값"이라고 생각했는데, idempotence와 함께 쓰면 오히려 무한 재시도가 더 안전하다는 게 직관에 반한다. 중복 방지가 보장되니까 "많이 시도해도 부작용이 없다"는 전제가 성립하는 것. 설정값은 단독으로 보면 안 되고, 다른 설정과의 조합으로 봐야 한다.
- `linger.ms`는 "지연 vs 효율"의 트레이드오프인데, 이미 Outbox Polling이 1초 간격이라 5ms 추가 지연은 의미 없다. 기존 아키텍처의 지연 특성을 고려해서 설정해야 한다.

### 참고 자료
- [Confluent - Kafka Producer Configuration](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html)
- [Baeldung - Kafka Idempotence Performance](https://www.baeldung.com/java-kafka-idempotence-performance)
- [Conduktor - Consumer Important Settings](https://learn.conduktor.io/kafka/kafka-consumer-important-settings-poll-and-internal-threads-behavior/)
- [Strimzi - Optimizing Kafka Consumers](https://strimzi.io/blog/2021/01/07/consumer-tuning/)

---

## 2026-03-26: Polling이란? — 비동기 처리 결과를 클라이언트가 확인하는 방법

### 고민한 부분
- 선착순 쿠폰 발급이 비동기(Kafka)로 처리되면, 사용자는 결과를 어떻게 알 수 있는가?
- Polling이 정확히 뭐고, 누가 API를 호출하는 건가?

### 내용 정리
- **Polling = 클라이언트(프론트엔드)가 "결과 나왔어?" 하고 주기적으로 서버에 물어보는 것**
- 서버가 클라이언트에게 직접 알려주는 게 아니라, 클라이언트가 반복 호출한다

흐름:
```
사용자 → POST /fcfs-issue → 202 + requestId (즉시 응답)

프론트엔드:
  1초차: GET /status?requestId=abc → PENDING
  2초차: GET /status?requestId=abc → PENDING
  3초차: GET /status?requestId=abc → SUCCESS → "발급 완료!" 표시
```

비동기 결과 전달 방식 비교:
| 방식 | 구현 | 실시간성 | 복잡도 |
|------|------|---------|--------|
| Polling | 클라이언트가 주기적 HTTP 요청 | 낮음 (1~2초 지연) | 낮음 |
| SSE | 서버가 단방향 push | 높음 | 중간 |
| WebSocket | 양방향 실시간 통신 | 최고 | 높음 |

선착순 쿠폰은 Polling으로 충분 — 1~2초 지연이 사용자 경험에 큰 영향 없음.

### 느낀 점
- "비동기 처리"라는 건 서버 내부의 이야기이고, 사용자 입장에서는 결과를 알 방법이 필요하다. 서버가 알려주지 않으면 클라이언트가 물어봐야 한다. Polling은 가장 단순한 방법이지만, 요청이 많아지면 서버 부하가 늘 수 있다 — 이건 트래픽이 커지면 SSE나 WebSocket으로 전환하면 된다.

---

## 2026-03-26: 엔티티 공유 전략 변경 — modules/jpa 이동 대신 별도 정의

### 고민한 부분
- commerce-streamer가 쿠폰 발급을 위해 Coupon, IssuedCoupon에 접근해야 하는데, 이 엔티티를 어떻게 공유할 것인가?

### 선택지
1. **modules/jpa로 이동** — 정석이지만 기존 commerce-api 코드 전체의 import/테스트 변경 필요
2. **commerce-streamer에 별도 엔티티 정의** — 같은 테이블을 읽되 앱별 독립 모델. 엔티티 중복이지만 심플

### 선택한 답
- 2번 선택
- 구현: `CouponInfo`(읽기 전용 — maxIssueCount, expiredAt 확인), `CouponIssuance`(쓰기 전용 — 발급 INSERT)
- 이유: 기존 commerce-api 코드를 건드리지 않아도 된다. 멘토링에서도 "기능 차이가 크면 분리가 바람직, 배치는 JDBC 사용하는 경우도 있다"는 피드백이 있었다. 두 앱이 같은 DB를 공유하므로 같은 테이블에 다른 엔티티 클래스를 매핑해도 문제없다.

### 느낀 점
- "엔티티 공유 = 모듈 공유"가 유일한 방법이 아니다. 같은 테이블을 각 앱에서 필요한 관점으로 따로 매핑하면, 앱 간 결합도를 낮추면서도 데이터를 공유할 수 있다. 이건 CQRS에서 읽기 모델과 쓰기 모델을 분리하는 것과도 맥이 닿는다.
