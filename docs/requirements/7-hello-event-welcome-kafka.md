## 🧭 루프팩 BE L2 - Round 7

> 느슨하게, 유연하게, 확장 가능하게!
>
>
> 애플리케이션 이벤트 기반으로 **무거운 동기 연산을 분리**하고, **복잡한 도메인 흐름을 제어**합니다.
>
> Kafka 기반으로 **서비스 경계 밖으로 이벤트를 발행**하고, 별도 Consumer 앱이 **후속 처리와 운영 책임**을 담당합니다.
>
> 메시지 전달 보장(At Least Once, Idempotency, DLQ)을 학습하며 **신뢰 가능한 이벤트 파이프라인**을 구현합니다.
>

<aside>
🎯

**Summary**

</aside>

지금까지 우리는 재고 차감, 포인트 차감, 쿠폰 사용, 결제 처리 등 **모든 흐름을 하나의 트랜잭션 안에서 처리**해왔습니다. 하지만 이 방식은 트랜잭션이 커지고, 실패 포인트가 많아지며, 시스템의 결합도도 높아지는 단점이 있습니다.

이번 라운드에서는 애플리케이션 이벤트를 활용해 **유스케이스의 후속 흐름을 분리**하고, **비동기 트랜잭션 흐름**을 설계하는 방법을 학습합니다. **서비스 경계를 넘는 확장성**을 갖추기 위해서 **Kafka 를 통해 이벤트를 외부로 발행**하고, **별도의 Consumer 앱이 후속 처리를 담당하는 구조**를 학습합니다.

나아가, Kafka를 활용한 **선착순 쿠폰 발급** 시나리오를 통해 대량 요청을 안전하게 처리하는 구조를 직접 구현해봅니다.

<aside>
📌

**Keywords**

</aside>

- 애플리케이션 이벤트 (ApplicationEventPublisher)
- 트랜잭션 분리 & 도메인 decoupling
- 후속 처리 비동기화
- 사후 정합성 처리 전략
- Kafka Producer & Consumer
- At Most Once / At Least Once / Exactly Once
- Idempotency & 멱등 처리
- Dead Letter Queue (DLQ)

<aside>
🧠

**Learning**

</aside>

## 🪓 문제 분석 - 무거워진 트랜잭션과 집중된 관심사

지금까지 우리는 **아래 흐름을 한 호흡에 처리**하도록 했습니다.

```json
createOrder()
 ├── 재고 차감
 ├── 쿠폰 사용
 ├── 결제 요청
 └── 주문 저장
```

모두 하나의 트랜잭션 범위 내에서 처리하려다 보니, 아래 문제가 발생하기 시작했어요.

| **문제점** | **설명** |
| --- | --- |
| 🧨 실패 전파 | PG API가 느려지거나 실패하면 주문 전체가 롤백됩니다 |
| 🧱 높은 결합도 | User, Product, Coupon, Payment 도메인이 모두 한 흐름에 엮입니다 |
| 🔁 재시도 불가 | 롤백은 가능하지만, 어디까지 성공했는지 불확실하여 복구가 어렵습니다 |
| 🐌 성능 저하 | 트랜잭션이 길어질수록 DB 락이 길게 유지되어 TPS가 하락합니다 |

### 🍰 흐름을 나누는 사고

> 이 문제를 해결하기 위한 핵심 전략은 **트랜잭션을 나누는 것**입니다.
>
>
> > ✅ 주문 등록
> ✅ 상품 재고 차감
> ✅ 할인 처리 (쿠폰 기반의 사용)
> ✅ 결제 처리 (카드 등)
> ✅ 데이터 플랫폼에 주문 정보 전송
> >
>
> 모든 처리를 동시에 하지 말고, **지금 꼭 해야 하는 것**과 **조금 나중에 해도 되는 것**을 분리합니다.
>

| **구분** | **하는 일** | **트랜잭션 경계** |
| --- | --- | --- |
| ✅ 핵심 트랜잭션 | 주문 생성, 금액 계산, 유효성 검증 | 반드시 커밋 보장 |
| ✉ 후속 트랜잭션 | 쿠폰 차감, 포인트 적립 기록, PG 호출 | 커밋 이후 실행 |
|  |  |  |

예를 들어, PG 장애가 발생해도 주문은 저장되어야 합니다. 이것이 바로 **트랜잭션의 분리**가 필요한 이유입니다.

### ✉️ Command vs Event

>
>
>
> 트랜잭션을 나누기 위한 도구로 **이벤트**를 사용합니다.
> **커맨드**는 하기 위한 정보를 담고, **이벤트**는 발생한 그 순간의 정보를 담습니다.
>

| **항목** | **Command** | **Event** |
| --- | --- | --- |
| 의미 | “~을 해라” (명령) | “~이 발생했다” (사실 통지) |
| 컨텍스트 | 요청 (request) | 결과 (result) |
| 주체 | 지목된 핸들러에 의해 실행 | 후속 핸들러가 알아서 반응 |
| 흐름 제어 | 호출자가 제어 | 호출자가 제어하지 않음 |

---

## ➗ Spring Application Event

<aside>
💡

**TL;DR**
Spring 은 `ApplicationEvent` 라는 개념을 통해 애플리케이션 내부에서 이벤트 기반의 흐름 제어를 제공합니다.

</aside>

### 🧠 ApplicationEvent

>
>
>
> Spring 은 ApplicationEvent 를 통해 애플리케이션 내부에서 **이벤트 기반의 흐름 제어**를 제공합니다.
> Kafka 같은 메시지 브로커 없이도 **Spring 단에서 후속 흐름을 분리**할 수 있습니다.
>

| **구성 요소** | **설명** |
| --- | --- |
| ApplicationEventPublisher | 이벤트를 발행하는 역할. 발행 시 스프링 내부적으로 @EventListener를 호출함 |
| @EventListener | 이벤트에 대한 처리를 수행하는 역할 |
| @TransactionalEventListener | 트랜잭션이 커밋된 뒤에만 이벤트가 처리되도록 보장 |
| phase = AFTER_COMMIT | 트랜잭션이 성공적으로 커밋된 경우에만 이벤트가 동작 |
| @Async | 이벤트 리스너를 **비동기(별도 스레드)**로 실행함 |

### ✍️ Why use ApplicationEvent?

| **비교 항목** | **설명** |
| --- | --- |
| 단순 호출 vs 이벤트 | 직접 호출은 강하게 결합됨 (OrderService → CouponService 직접 호출)
ApplicationEvent는 흐름을 **통지**만 하고, **처리는 외부에 위임** |
| 유연한 확장 | 새로운 후속 로직이 필요하면, 이벤트 리스너만 추가하면 됨 |
| 추가 기술 의존 없음 | Kafka 같은 메시지 브로커 없이도 **Spring 단에서 구현 가능** |
| 커밋 이후 보장 | @TransactionalEventListener(phase = AFTER_COMMIT)로 안전하게 커밋 후 처리 가능 |

### Java 코드 예시

`OrderApplicationService.java`

```java
// 1. 주문 저장 후 이벤트 발행
@Transactional
void createOrder(...) {
		Order order = orderRepository.save(...);
		eventPublisher.publishEvent(OrderCreatedEvent.from(order));
}

```

`OrderEventHandler.java`

```java
// 2. 커밋 이후 비동기로 후속 로직 처리
@Component
class OrderEventHandler {
		..
    @TransactionalEventListener(phase = AFTER_COMMIT)
    @Async
    void handle(OrderCreatedEvent event) {
        if (event.couponId != null) {
		        couponService.useCoupon(event.couponId);
        }
        pointService.record(event.amount());
        pgClient.requestPayment(PaymentCommand.from(event));
    }
}
```

### Kotlin 코드 예시

`OrderApplicationService.kt`

```kotlin
// 1. 주문 저장 후 이벤트 발행
@Transactional
fun createOrder(...) {
    val order = orderRepository.save(...)
    eventPublisher.publishEvent(OrderCreatedEvent.from(order))
}
```

`OrderEventHandler.kt`

```kotlin
// 2. 커밋 이후 비동기로 후속 로직 처리
@Component
class OrderEventHandler {
		..
    @TransactionalEventListener(phase = AFTER_COMMIT)
    @Async
    fun handle(event: OrderCreatedEvent) {
        event.couponId?.let { couponService.useCoupon(it) }
        pointService.record(event.amount)
        pgClient.requestPayment(PaymentCommand.from(event))
    }
}
```

**ApplicationEventPublisher ← 애플리케이션 레이어에서 발생하는 이벤트**

@Entity

class Xxx  implements AggreagreRoot

**publicsEvents(Events) ← 도메인 레이어에서도 이벤트를 발행할 수 있음.**

우리가 사용하는 JPA 에서는 어떻게 사용하면 좋을지?

---

## 💣 오해 - Silver Bullet ?

>
>
>
> @Async, 이벤트 기반 구조는 확장성과 응답 속도 면에서 매우 유리합니다.
> 하지만 그만큼 **제어 흐름**이 눈에 보이지 않고, **실패 감지**가 어렵습니다
>

### 🔥 발생할 수 있는 문제

| **리스크** | **설명** | **대응 전략** |
| --- | --- | --- |
| ❌ 예외 은닉 | 이벤트 리스너 내 실패는 사용자에게 노출되지 않음 | 로그 적재, 모니터링, 실패 이벤트 보관 필요 |
| ❌ 순서 보장 어려움 | 이벤트 리스너는 병렬 실행될 수 있음 | 업무적으로 순서 의존이 없는 흐름만 분리 |
| ❌ 중복 실행 | 트랜잭션 재시도나 이벤트 중복 발행 시 여러 번 실행될 수 있음 | idempotency 처리: 이벤트 ID 기준 중복 차단 |
| ❌ 장애 누락 | 슬랙 알림, 메일 전송 등 외부 연동 실패 시 조용히 무시될 수 있음 | 예외 발생 시 DLQ (Dead Letter Queue) 등 보완 구조 필요 |

### **☂** 그래서 실무에서는 자연스럽게 이런 질문에 도달합니다.

- 중요 이벤트는 **이벤트 저장소 (Outbox)**에 적재 후 처리하는 게 좋지 않을까?
- 정말 중요한 처리는 결국 **메시지 브로커(Kafka 등)**로 보내야 하지 않을까?

---

## 📮 How About It? Kafka!

<aside>
❓

Kafka 는 흔히 **고성능 메시지 큐**라고 생각하지만, 근본적으로는 **분산 로그 저장소(Distributed Log Store)** 입니다.

- 일반적인 Message Queue 와는 다르게 **디스크에 Log 를 지속적으로 append** 함
- Consumer 는 **각자의 Offset 을 기억하고 필요한 시점부터 읽는 것이** 가능함 (재처리 가능)
</aside>

### Kafka 의 주요 특징

- **고가용성** - Partition + Replica 구조로 브로커 장애 시에도 데이터 유실을 최소화
- **확장성** - Broker, Partition 의 수평 확장으로 처리량의 선형 증가
- **범용성** - 단순 메세징 뿐이 아닌 다음의 용도로도 사용
    - 1️⃣ 로그 수집
    - 2️⃣ 이벤트 소싱
    - 3️⃣ 스트리밍 처리의 기반

### Kafka Components

1. **Broker**
    - 카프카 서버 Unit
    - Producer 의 메세지를 받아 Offset 지정 후 디스크에 저장
    - Consumer 의 파티션 Read 에 응답해 디스크의 메세지 전송
    - `Cluster` 내에서 각 1개씩 존재하는 Role Broker
        - **Controller**

          다른 브로커를 모니터링하고 장애가 발생한 Broker 에 특정 토픽의 Leader 파티션이 존재한다면, 다른 브로커의 파티션 중 Leader 를 재분배하는 역할을 수행

        - **Coordinator**

          컨슈머 그룹을 모니터링하고 해당 그룹 내의 특정 컨슈머가 장애가 발생해 매칭된 파티션의 메세지를 Consume 할 수 없는 경우, 해당 파티션을 다른 컨슈머에게 매칭해주는 역할 수행 (`Rebalance`)

2. **Cluster**
    - 고가용성 (HA) 를 위해 여러 서버를 묶어 특정 서버의 장애를 극복할 수 있도록 구성
    - Broker 가 증가할 수록 메시지 수신, 전달 처리량을 분산시킬 수 있으므로 확장에 유리

      > 동작중인 다른 Broker 에 영향 없이 확장이 가능하므로, 트래픽 양의 증가에 따른 브로커 증설이 손쉽게 가능
>
3. **Topic & Partition**
    - `Topic` 은 메세지를 분류하는 기준이며 N 개의 `Partition` 으로 구성
        - 1개의 `Leader` 와 0..N 개의 `Follower` 파티션으로 구성해 가용성을 높일 수 있음
    - `Partition` 은 서로 다른 서버에 분산시킬 수 있기 때문에 수평 확장이 가능
    - 각 `Topic` 의 메세지 처리순서는 `Partition` 별로 관리됨
4. **Message**
    - 카프카에서 취급하는 데이터의 Unit **( ByteArray )**
5. **Producer & Consumer**
    - `Producer` - 메세지를 특정 Topic 에 생성
        - 저장될 파티션을 결정하기 위해 메세지의 Key 해시를 활용하며 Key 가 존재하지 않을 경우, 균형 제어를 위해 Round-Robin 방식으로 메세지를 기록
        - **Partitioner**

          메세지를 수신할 때, 토픽의 어떤 파티션에 저장될 지 결정하며 Producer 측에서 결정. 메세지에 key 가 없다면 / 특정  메세지에 key 가 존재한다면 key 의 해시값에 매칭되는 파티션에 데이터를 전송함으로써 항상 같은 파티션에 메세지를 적재해 **순서 보장** 이 가능하도록 처리할 수 있음.

    - `Consumer` - 1개 이상의 Topic 을 구독하며 메세지를 순서대로 읽음
        - 메세지를 읽을 때마다 파티션 별로 Offset 을 유지해 읽는 메세지의 위치를 추적할 수 있으며 오프셋은 두가지 종류가 존재
        - `CURRENT-OFFSET`

          컨슈머가 어디까지 처리했는지를 나타내는 offset 이며 메세지를 소비하는 컨슈머가 이를 기록하고 후에 장애가 발생했을 시에 그 뒤부터 이어 처리할 수 있도록 하며 장애 복구 상황을 위해 메세지가 처리된 이후에 반드시 커밋하여야 함

        - 만약 오류가 발생하거나 문제가 발생할 경우, 컨슈머 그룹 차원에서 `--reset-offsets`  옵션을 통해 실패한 시점으로 오프셋을 되돌릴 수 있음
6. **Consumer Group**
    - 메세지를 소비할 때, 토픽의 파티션을 매칭하는 그룹 단위이며 N 개의 컨슈머를 포함
    - 각 파티션은 그룹 내 하나의 컨슈머만 소비할 수 있음
    - 보통 소비 주체인 Application 단위로 Consumer Group 을 생성, 관리함
    - 같은 토픽에 대한 소비주체를 늘리고 싶다면, 별도의 컨슈머 그룹을 만들어 토픽을 구독

      ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0e86a30d-2307-454b-9469-59a3c805b1b1/0453d381-6abe-4bfa-a4ab-5ef2e4a7ef98/Untitled.png)


    > 파티션의 개수가 그룹 내 컨슈머 개수보다 많다면 잉여 파티션의 경우 메세지가 소비될 수 없음을 의미함
    > 
    - **( 참고 )** 토픽의 Partition 개수와 Consumer 개수에 따른 소비
        
        ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0e86a30d-2307-454b-9469-59a3c805b1b1/b6f3777a-44b7-4e85-a0b0-3b708ede0291/Untitled.png)
        
        ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0e86a30d-2307-454b-9469-59a3c805b1b1/ba2876f1-df33-45fc-86b5-85eb462e17ba/Untitled.png)
        
        ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0e86a30d-2307-454b-9469-59a3c805b1b1/74cc946f-e4b3-4f1b-9611-374cd526b410/Untitled.png)

7. **Rebalancing**
    - Consmuer Group 의 **가용성과 확장성**을 확보해주는 개념
    - 특정 컨슈머로부터 다른 컨슈머로 파티션의 소유권을 이전시키는 행위

      e.g. `Consumer Group` 내에 Consumer 가 추가될 경우, 특정 파티션의 소유권을 이전시키거나 오류가 생긴 Consumer 로부터 소유권을 회수해 다른 Consumer 에 배정함


    <aside>
    🚫 **( 주의 )** 리밸런싱 중에는 컨슈머가 메세지를 읽을 수 없음.
    
    </aside>
    
    `Rebalancing Case`
    
    1. Consumer Group 내에 새로운 Consumer 추가
    2. Consumer Group 내의 특정 Consumer 장애로 소비 중단
    3. Topic 내에 새로운 Partition 추가
8. **Replication**
    - Cluster 의 가용성을 보장하는 개념
    - 각 Partition 의 Replica 를 만들어 백업 및 장애 극복
        - Leader Replica

          각 파티션은 1개의 리더 Replica를 가진다. 모든 Producer, Consumer 요청은 리더를 통해 처리되게 하여 일관성을 보장한다.

        - Follower Replica

          각 파티션의 리더를 제외한 Replica 이며 단순히 리더의 메세지를 복제해 백업한다. 만일, 파티션의 리더가 중단되는 경우 팔로워 중 하나를 새로운 리더로 선출한다.

          > Leader 의 메세지가 동기화되지 않은 Replica 는 Leader 로 선출될 수 없다.
          >
            - `In-Sync Replica (ISR)`

              Leader 의 최신 메세지를 계속 요청하는 Follower

            - `Out-Sync Replica (OSR)`

              특정 기준에 의해 Leader 의 메세지를 백업하는 Follower


### Messaging Systems

| 구분 | **Redis (Pub/Sub)** | **RabbitMQ (AMQP)** | **Kafka (Distributed Log)** |
| --- | --- | --- | --- |
| **기반 모델** | Pub/Sub | 메시지 큐 (AMQP 프로토콜) | Pub/Sub + 분산 로그 |
| **메시지 저장** | ❌ 저장 안 함
(채널 자체에 보관 X) | ✅ 일시 저장
Queue 에 보관(서버/Queue 종료 시 삭제) | ✅ 저장
디스크(Log)에 영속 저장(보존 기간까지 유지) |
| **구독 방식** | 채널 기반, subscriber 없으면 메시지 소실 | Exchange → Queue 매핑 후 Consumer 수신 | Topic → Partition, Consumer Group 으로 분배 |
| **순서 보장** | 없음 | Queue 단위 순서 보장 | Partition 단위 순서 보장(Key 로 동일 Partition 강제 가능) |
| **확장성** | 제한적 (단일 서버 메모리 한계) | 브로커 클러스터 구성 가능하지만 Scale-out 제한적 | 고수준 확장성 (Broker/Partition 수평 확장) |
| **재처리 (Replay)** | ❌ 불가
(실시간 전달 전용) | ❌ 불가 
(소비하면 Queue 에서 제거) | ✅ 가능 
(Consumer Offset 조정으로 과거 이벤트 재소비 가능) |
| **메시지 유실** | Subscriber 없으면 유실 | Queue 보관 중 서버 장애 시 유실 가능 | 설정(`acks=all`, Replica)으로 내구성 보장 |
| **활용 사례** | 실시간 알림, 단순 신호 전달 | 트랜잭션 메시징, 업무 프로세스 큐잉 | 로그 수집, 이벤트 소싱, 스트리밍 처리, 대규모 이벤트 파이프라인 |

---

## 🚀 Kafka Essentials

> 카프카 활용의 핵심은 **메세지를 잃지 않고, 단 한번만 처리되게 보장할 수 있는가** 입니다.
>

### 📦 Message Delivery Semantics

**1️⃣ Producer → Broker**

- 🎯 **어떻게든 발행 (At Least Once)**
- `Producer` 는 네트워크 지연, 장애가 있어도 메세지를 최소 한 번은 `Broker` 에 기록되도록 보장해야 합니다.

  ### Producer 측 패턴: **Transactional Outbox**

    ```jsx
    - 도메인 데이터 변경(DB write)과 아웃박스 메시지 기록 → 하나의 DB 트랜잭션으로 묶음
    - Outbox 테이블에 쌓인 메시지를 별도의 릴레이가 Broker로 전달 (Polling 하거나 CDC (ex.데비지움) 사용하거나)
    - 실패 시 재시도 → At Least Once 발행 보장
    ```

    1. 주문 DB insert + Outbox "OrderCreatedEvent" 기록 → 하나의 트랜잭션
    2. 백그라운드 워커(릴레이)가 Outbox → Kafka publish
    3. 네트워크 불안해도 워커가 재시도 → Broker에 최소 1회 기록

```java
**참조
https://microservices.io/patterns/data/transactional-outbox.html**
```

**2️⃣ Consumer ← Broker**

- 🎯 **어떻게든 한 번만 처리 (At Most Once)**
- `Consumer` 는 같은 메세지가 여러 번 오더라도, 멱등하게 처리하여 최종 결과는 단 한번만 반영되도록 보장해야 합니다.

  ### Consumer 측 패턴: **Idempotent Consumer**

    1. 메시지를 받을 때 event_handled 테이블에 메시지 ID 기록
    2. 이미 처리된 메시지인지 검사 → 있으면 skip
    3. 처리 완료 후 상태 업데이트

> **왜 이벤트 핸들링 테이블과 로그 테이블을 분리하는 걸까? 에 대해 고민해보자**
>

### 🛡️ 멱등성 구현 전략

| 전략 | 설명 |
| --- | --- |
| `eventId` PK 테이블 | 중복 메시지 무시 |
| `version` / `updatedAt` 비교 | 최신 이벤트만 반영 |
| **Upsert** | Insert or Update로 동일 결과 유지 |

---

### 🚨 Operation Tips

- **Retry & Backoff**: 일시 장애는 재시도로 복구, 즉시 무한재시도는 금물
- **DLQ (Dead Letter Queue)**: 반복 실패 메시지는 DLQ로 격리, 운영자가 후처리
- **Lag 모니터링**: Consumer가 얼마나 뒤쳐져 있는지 체크 (지연·병목 지표)
- **Partition 순서 보장**: Partition 단위로만 순서가 보장되므로 `partition.key=aggregateId` 설정 필수

---

## 🏗️ 우리 프로젝트에 적용하기

### 아키텍처 구조

```jsx
commerce-api (Producer)
  ├── 도메인 이벤트 발행 (ApplicationEvent)
  ├── Outbox 테이블 기록
  └── Kafka 토픽 발행
				├── catalog-events  (key=productId)
				├── order-events    (key=orderId)
				└── coupon-issue-requests (key=couponId)

commerce-streamer (Consumer)
  ├── Metrics 집계 (product_metrics upsert)
  ├── 멱등 처리 (event_handled 테이블)
  └── 쿠폰 발급 처리
```

### 왜 앱을 분리하는가?

| **단일 앱** | **Producer / Consumer 분리** |
| --- | --- |
| 모든 처리가 한 서버에 집중 | 발행과 소비의 스케일을 독립적으로 조절 |
| Consumer 장애가 API에 영향 | Consumer가 죽어도 API는 정상 동작 |
| 배포 시 전체 재시작 | 각각 독립 배포 가능 |

### Metrics 집계 파이프라인

- `commerce-api`에서 발생하는 유저 행동(상품 조회, 좋아요, 주문 등)을 Kafka로 발행
- `commerce-streamer`가 이벤트를 소비해 product_metrics 테이블에 집계 (upsert)
- 좋아요 수, 판매량, 상세 페이지 조회 수 등을 실시간에 가깝게 반영

## 🎫 선착순 쿠폰 발급

> *선착순 100장 한정 쿠폰 발급에 1만 명이 동시에 요청하면 어떻게 될까요?*
>

```jsx
[사용자] → POST /coupons/issue
   → commerce-api: Kafka에 발급 요청 발행 (빠른 응답)
   → coupon-issue-requests 토픽
   → commerce-streamer: Consumer가 순차 처리
       ├── 수량 확인 → 발급 or 거절
       ├── 중복 발급 방지 (userId 기반)
       └── 발급 결과 저장
[사용자] → GET /coupons/issue/{requestId} (결과 조회)
```

### 왜 Kafka인가?

- API는 요청을 Kafka에 넣기만 하므로 응답이 빠르다
- Consumer가 순차적으로 처리하므로 동시성 문제가 줄어든다
- 트래픽이 몰려도 Kafka가 버퍼 역할을 하며 시스템을 보호한다

💡 **이 구조는 다음 주차에서 다룰 대기열 시스템의 기초가 됩니다.**

---

### **🌾** Summary

| **구분** | **ApplicationEvent (Spring)** | Kafka |
| --- | --- | --- |
| 전송 범위 | 애플리케이션 내부 (단일 JVM) | 애플리케이션 외부 (서비스 간 / 시스템 간) |
| 보존 | 없음 (메모리 기반) | 설정된 기간 동안 메시지 로그로 보존 |
| 신뢰성 | 장애 발생 시 손실 가능 | 브로커에 저장되며 재처리 가능 (at-least-once 등) |
| 속도 | 매우 빠름 | 네트워크 I/O 포함, 상대적으로 느림 |
| 적절한 사용 | 내부 후속 처리 흐름 | 서비스 간 이벤트 전달, 비동기 처리, 데이터 파이프라인 |

| **목적** | **설명** |
| --- | --- |
| 🎯 트랜잭션 최소화 | 핵심 흐름만 빠르게 처리하고, 후속 로직은 별도로 |
| 🎯 결합도 감소 | OrderService는 쿠폰, 포인트, 결제 흐름을 몰라도 됨 |
| 🎯 장애 격리 | 외부 시스템(PG) 실패가 전체 서비스에 영향을 주지 않음 |
| 🎯 유연한 확장 | 이벤트 구독만으로 기능 확장 가능 → 신규 알림, 적립도 쉽게 추가 가능 |

<aside>
📚

**References**

</aside>

| 구분 | 링크 |
| --- | --- |
| 🔍 Event vs Command | [littlemobs - Event 와 Command 의 차이점 쉽게 이해하기](https://littlemobs.com/blog/difference-between-event-and-command/) |
| ⚙ Spring Application Events | [Spring Application Events](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/application-events.html) |
| 📖 Event in Spring | [Baeldung - Event in Spring](https://www.baeldung.com/spring-events) |
| 🔍 Kafka | [Kafka docs](https://kafka.apache.org/documentation/) |
| ⚙ Spring Kafka | [Spring for Apache Kafka](https://spring.io/projects/spring-kafka) |
| 📖 우아콘2023 - 카프카 | [Kafka를 활용한 이벤트 기반 아키텍처 구축](https://www.youtube.com/watch?v=DY3sUeGu74M) |
| 🌟 라인 - 카프카 활용 | [LINE에서 Kafka를 사용하는 방법](https://engineering.linecorp.com/ko/blog/how-to-use-kafka-in-line-1) |

<aside>
🌟

**Next Week Preview**

</aside>

> **만약 사용자가 너무 많으면 어떻게 제어해야 할까?**
>
>
> 이번 주차에는 핵심 로직은 빠르게 끝내고, 후속 로직은 나중에 처리하자는 구조적 분리를 배웠습니다. 카프카를 통해 안정적으로 이벤트를 발행하고 처리하는 파이프라인도 구축해보았습니다.
>
> 다음주에는 블랙 프라이데이 행사를 앞두고, 주문이 몰렸을 때 원활히 제어하기 위한 주문 대기열 시스템을 고민해볼거예요!
>