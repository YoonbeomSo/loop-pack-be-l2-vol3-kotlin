# Kafka 메시지는 어디서 사라지는가? (Producer부터 Consumer까지, 유실을 막는 설정들)

부제: acks=all만 설정하면 끝인 줄 알았습니다..

## TL;DR

- Kafka 메시지는 Producer → Broker → Consumer, 3개 구간 어디서든 유실될 수 있다
- 각 구간마다 acks=all, idempotence, manual ACK 등 설정이 필요하고, 하나라도 빠지면 구멍이 뚫린다
- "유실은 복구 불가, 중복은 멱등 처리로 해결" — 그래서 ACK은 항상 마지막이어야 한다

---

## 메시지가 사라지는 3가지 구간

Kafka를 처음 도입하면서 "메시지 유실을 막아야 한다"는 건 알았는데, 정확히 어디서 유실되는지는 생각해보지 않았습니다.

정리해보니 3개 구간이었습니다.

| 구간 | 유실 시나리오 | 결과 |
|------|-------------|------|
| Producer → Broker | 발행했는데 Broker가 저장 못 함 | 이벤트 자체가 사라짐 |
| Broker 내부 | Leader에만 저장되고 Follower 복제 전 Leader 죽음 | 저장된 줄 알았는데 없음 |
| Broker → Consumer | Consumer가 가져갔는데 처리 전에 죽음 | 처리 안 됐는데 메시지 사라짐 |

각 구간별로 어떤 설정이 이걸 막는지, 안 했으면 어떤 일이 벌어지는지 살펴보겠습니다.

---

## 구간 1: Producer → Broker — acks 설정

### acks에는 3가지 옵션이 있다

Producer가 메시지를 보낸 후 "성공"으로 판단하는 기준이 `acks`입니다.

| acks | 동작 | 속도 | 안전성 |
|------|------|------|--------|
| `0` | Broker 응답을 기다리지 않음. 보내고 끝 | 가장 빠름 | 유실 가능 (Broker가 받았는지조차 모름) |
| `1` | Leader 파티션에 저장되면 성공 | 보통 | Leader 죽으면 유실 가능 |
| `all` | Leader + 모든 Follower(ISR)에 저장되면 성공 | 가장 느림 | 유실 없음 |

### acks=1일 때 무슨 일이 벌어지는가?

기본값 `acks=1`이면 Leader 파티션에 저장되는 순간 성공 응답을 보냅니다.

문제는 Leader에만 저장된 상태에서 Leader가 죽으면, Follower로 승격된 새 Leader에는 그 메시지가 없다는 겁니다.

```
Producer → "주문 생성 이벤트 발행"
Broker Leader: 저장 완료 → Producer에 "성공" 응답
Broker Follower: 아직 복제 안 됨
Leader 죽음 → Follower가 새 Leader
→ 메시지 없음. 주문은 됐는데 이벤트는 사라짐.
```

`acks=0`은 더 심합니다. Broker가 메시지를 받았는지조차 확인하지 않으니, 네트워크 장애면 Producer는 성공으로 알고 있는데 Broker에는 아무것도 없는 상태가 됩니다. 로그 수집처럼 일부 유실이 괜찮은 경우에만 쓸 수 있습니다.

### acks=all로 설정하면?

모든 ISR(In-Sync Replica)이 저장을 확인한 후에만 성공 응답을 보냅니다.

```yaml
producer:
  acks: all  # 모든 ISR 저장 확인 후 성공
```

```
Producer → "주문 생성 이벤트 발행"
Broker Leader: 저장 완료
Broker Follower 1: 복제 완료
Broker Follower 2: 복제 완료
→ 이제서야 Producer에 "성공" 응답
→ Leader가 죽어도 Follower에 이미 있음. 유실 없음.
```

Leader가 죽어도 Follower에 이미 복제되어 있으니 메시지가 유실되지 않습니다. 대신 Leader 혼자 저장하면 끝나는 걸 모든 Follower 복제까지 기다려야 하니, 그만큼 응답이 느려지는 트레이드오프가 있습니다.

우리 프로젝트에서는 주문/결제/쿠폰 이벤트가 유실되면 후속 처리가 안 되므로 `acks=all`을 선택했습니다. 속도보다 안전이 중요한 경우입니다.

---

## 구간 1 보완: enable.idempotence=true — 리트라이 시 중복 방지

`acks=all`로 유실은 막았는데, 또 다른 문제가 있었습니다.

### 리트라이가 만드는 중복

네트워크 타임아웃으로 Producer가 성공 응답을 못 받으면 리트라이합니다. 그런데 실제로는 Broker에 이미 저장된 상태라면?

```
Producer → Broker: "주문 이벤트 발행"
Broker: 저장 완료, 응답 전송 중...
네트워크 타임아웃 → Producer: "실패로 판단, 리트라이"
Producer → Broker: "주문 이벤트 발행" (같은 메시지)
→ Broker에 동일 메시지 2건 저장
```

Consumer 입장에서는 같은 주문 이벤트를 2번 받게 됩니다. 좋아요 수가 2번 올라가거나, 쿠폰이 2장 발급될 수 있습니다.

### idempotence가 이걸 어떻게 막는가?

`enable.idempotence=true`를 설정하면 Producer가 메시지에 **PID(Producer ID) + 시퀀스 번호**를 붙입니다. Broker는 이 조합을 보고 이미 저장된 메시지인지 판단합니다.

```
메시지 1: PID=1, seq=0 → Broker 저장 ✓
메시지 1: PID=1, seq=0 → Broker: "이미 있네, 무시" (중복 방지)
메시지 2: PID=1, seq=1 → Broker 저장 ✓ (새 메시지)
```

시퀀스 번호는 Producer가 자동으로 관리하므로 우리가 신경 쓸 건 설정을 켜는 것뿐입니다.

```yaml
producer:
  properties:
    enable.idempotence: true              # 리트라이 시 중복 저장 방지
    max.in.flight.requests.per.connection: 5  # idempotence 시 5 이하 필수
```

### max.in.flight.requests.per.connection=5는 왜?

이건 "Broker 응답을 기다리지 않고 동시에 보낼 수 있는 요청 수"입니다. idempotence를 사용하면 Broker가 시퀀스 번호로 순서를 추적하는데, 동시 요청이 너무 많으면 순서가 꼬일 수 있습니다.

Kafka 공식 문서에서 idempotence 사용 시 5 이하를 권장합니다. 5개까지는 Broker가 시퀀스 번호로 순서를 보장할 수 있고, 그 이상이면 보장할 수 없습니다.

```
max.in.flight=1: 한 번에 하나만 → 순서 확실하지만 느림
max.in.flight=5: 동시 5개 → 속도와 순서 보장의 균형 (idempotence 시 최대)
max.in.flight=10: idempotence 순서 보장 깨짐
```

### retries를 MAX로 설정한 이유

처음에 `retries=3`으로 했다가 MAX(2147483647)로 바꿨습니다.

idempotence가 중복을 막아주니까, 재시도를 많이 해도 부작용이 없습니다. 오히려 retries=3이면 일시적 네트워크 장애에서 불필요하게 실패합니다.

그런데 "무한 재시도면 영원히 안 끝나는 거 아닌가?"라는 의문이 들었습니다. 여기서 `delivery.timeout.ms`가 실질적 상한 역할을 합니다.

```yaml
producer:
  retries: 2147483647  # 횟수 제한 없음
  properties:
    delivery.timeout.ms: 120000  # 120초가 지나면 아무리 재시도해도 실패 처리
```

동작을 정리하면:
- `retries` = "몇 번까지 시도할지" (횟수)
- `delivery.timeout.ms` = "최대 몇 초까지 시도할지" (시간)
- 둘 중 하나라도 걸리면 실패

retries=3이면 3번 만에 안 되면 포기하지만, retries=MAX이면 120초 동안 계속 시도합니다. 네트워크가 5초 후에 복구되면 retries=3은 이미 포기했지만, retries=MAX는 성공합니다.

설정값은 단독으로 보면 안 되고, 다른 설정과의 조합으로 봐야 합니다.

---

## 구간 3: Broker → Consumer — auto-commit의 함정

이게 가장 함정이었습니다.

### auto-commit은 정확히 언제 커밋하는가?

`enable.auto.commit=true`(기본값)이면 Consumer가 `poll()`을 호출할 때마다 이전 poll의 offset을 자동 커밋합니다. 정확히는 `auto.commit.interval.ms`(기본 5초) 간격으로 커밋합니다.

문제는 **"가져온 시점"과 "처리 완료 시점"이 다르다**는 겁니다.

```
t=0s: poll() → 메시지 10건 가져옴
t=1s: 3건 처리 완료
t=5s: auto-commit 타이머 동작 → offset 10으로 커밋 (10건 전부 커밋됨!)
t=6s: 4번째 메시지 처리 중 서버 죽음
→ 재기동: offset 10부터 시작 → 4~10번 메시지는 다시 안 옴
→ 7건 유실
```

auto-commit은 "처리했는지"가 아니라 "가져갔는지"를 기준으로 커밋합니다. 이 차이가 유실을 만듭니다.

### `auto.commit.interval.ms`를 줄이면 해결되나?

간격을 1초로 줄이면 유실 범위가 줄어들긴 하지만, 근본적으로 "가져간 것 = 처리한 것"이라는 가정이 틀린 거라서 해결이 안 됩니다. 1초 안에 서버가 죽으면 여전히 유실됩니다.

### enable-auto-commit=false + ack-mode=manual로 바꾸면?

처리가 완전히 끝난 후에만 `acknowledgment.acknowledge()`를 호출하여 offset을 커밋합니다.

```yaml
consumer:
  properties:
    enable-auto-commit: false  # 자동 커밋 끄기
listener:
  ack-mode: manual  # 수동 ACK
```

```kotlin
fun consume(message: KafkaEventMessage, acknowledgment: Acknowledgment) {
    // 1. 멱등 체크
    if (idempotencyService.isAlreadyHandled(message.eventId)) {
        acknowledgment.acknowledge()
        return
    }

    // 2. 비즈니스 로직
    metricsAggregationService.incrementLikeCount(productId, message.version)

    // 3. 처리 완료 기록
    idempotencyService.markHandled(message.eventId, ...)

    // 4. 여기서만 offset 커밋
    acknowledgment.acknowledge()
}
```

이제 offset 커밋 시점 = 처리 완료 시점이 됩니다. 처리가 끝나지 않으면 절대 커밋하지 않습니다.

---

## ACK은 왜 항상 마지막인가?

manual ACK을 쓰면 "처리 완료 후 ACK"이 되는데, 그러면 비즈니스 로직은 성공했는데 ACK 전에 서버가 죽으면 어떻게 될까요? 같은 메시지를 다시 받게 됩니다. 이때 `event_handled` 테이블이 필요합니다.

### event_handled 테이블이 없으면?

```
메시지 수신 → 비즈니스 로직 (likeCount +1) → ACK 전에 서버 죽음
재기동 → 같은 메시지 재수신 → 비즈니스 로직 다시 실행 (likeCount +1)
→ 좋아요가 2번 올라감 (중복 처리)
```

### event_handled 테이블이 있으면?

```
메시지 수신 → 멱등 체크 (없음) → 비즈니스 로직 (likeCount +1) → event_handled INSERT → ACK 전에 서버 죽음
재기동 → 같은 메시지 재수신 → 멱등 체크 (있음!) → skip → ACK
→ 좋아요는 1번만 올라감 (중복 방지)
```

`event_handled` 테이블은 eventId를 PK로 가진 단순한 테이블입니다. 메시지를 처리할 때마다 eventId를 INSERT하고, 재수신 시 PK 존재 여부만 체크합니다.

### 각 단계 사이 서버 죽으면?

이 순서에서 각 단계 사이에 서버가 죽으면 어떻게 되는지 생각해봤습니다.

| 장애 시점 | 결과 | 대응 |
|----------|------|------|
| 비즈니스 로직 처리 전 | offset 미커밋 → 재수신 → 멱등 체크에서 없으니 정상 처리 | 자동 복구 |
| 비즈니스 로직 성공, ACK 전 | offset 미커밋 → 재수신 → 멱등 체크에서 있으니 skip | 멱등 처리 |
| ACK 성공 | 완료 | - |

만약 ACK을 비즈니스 로직 **전에** 보냈다면?

```
ACK 먼저 → offset 커밋됨 → 비즈니스 로직 실행 중 서버 죽음
→ 재기동 시 offset이 이미 커밋 → 이 메시지는 다시 안 옴
→ 처리 안 됐는데 메시지 유실. 복구 불가.
```

정리하면:
- ACK을 먼저 보내면 → **유실** (복구 불가)
- ACK을 나중에 보내면 → **중복** (멱등 처리로 해결 가능)

유실은 되돌릴 수 없지만, 중복은 `event_handled` 테이블로 걸러낼 수 있습니다. 그래서 ACK은 항상 마지막입니다.

이게 바로 "At Least Once 수신 + 멱등 처리 = Exactly Once 효과"입니다. 진짜 Exactly Once는 존재하지 않고, "여러 번 받되 한 번만 처리되도록" 만드는 겁니다.

---

## 설정이 정말 동작하는지, 테스트로 증명하기

앞에서 `enable-auto-commit: false`, `ack-mode: manual`, `event_handled` 멱등 처리 — 이런 설정을 했습니다. 그런데 한 가지 찝찝한 게 있었습니다.

> *"이 설정들이 정말로 의도대로 동작하고 있는 건 맞나?"*

블로그에 "이 설정을 하면 안전합니다"라고 쓰려면, 실제로 동작한다는 증거가 있어야 합니다. 시나리오만 나열하면 그건 예측이지 검증이 아닙니다.

그래서 실제로 Kafka 메시지를 발행하고, Consumer가 처리하고, DB에 반영되는 전체 흐름을 통합 테스트로 검증하기로 했습니다.

### 검증 1: 멱등 처리가 진짜 동작하는가?

검증하고 싶었던 핵심 질문은 이겁니다:

- 같은 `eventId`로 메시지를 2번 보내면, `likeCount`가 1만 올라가는가?
- 아니면 `event_handled` 체크를 빠져나가서 2번 올라가는가?

```kotlin
@SpringBootTest
class CatalogEventConsumerIntegrationTest {

    @Test
    fun likeCountRemainsOne_whenDuplicateEventIdPublished() {
        // arrange
        val eventId = UUID.randomUUID().toString()
        val duplicateMessage = createLikedMessage(productId = 2, version = 1L, eventId = eventId)
        val checkMessage = createViewedMessage(productId = 2, version = 2L)

        // act — 1차 발행
        kafkaTemplate.send(KafkaTopics.CATALOG_EVENTS, "2", duplicateMessage)

        // assert — 1차 처리 대기
        await atMost Duration.ofSeconds(15) untilAsserted {
            val metrics = productMetricsJpaRepository.findByProductId(2)
            assertThat(metrics!!.likeCount).isEqualTo(1)
        }

        // act — 같은 eventId로 2차 발행(중복) + 확인용 VIEWED 이벤트
        kafkaTemplate.send(KafkaTopics.CATALOG_EVENTS, "2", duplicateMessage)
        kafkaTemplate.send(KafkaTopics.CATALOG_EVENTS, "2", checkMessage)

        // assert — VIEWED가 처리됐으면, 중복도 이미 처리(skip)된 것
        await atMost Duration.ofSeconds(15) untilAsserted {
            val metrics = productMetricsJpaRepository.findByProductId(2)
            assertThat(metrics!!.viewCount).isEqualTo(1) // 확인용 메시지 처리됨
            assertThat(metrics.likeCount).isEqualTo(1)   // 중복은 skip됨
        }
    }
}
```

같은 `eventId`를 2번 보내고, 중복 메시지 뒤에 `PRODUCT_VIEWED` 이벤트를 하나 더 보냅니다. 같은 파티션 키(`"2"`)이므로 순서가 보장됩니다. `viewCount=1`이 되었다는 건, 그 앞의 중복 메시지도 Consumer가 이미 받았다(그리고 skip했다)는 증거입니다.

### 검증 2: 테스트가 발견한 진짜 버그

그런데 테스트를 처음 돌렸을 때, 전부 실패했습니다.

```
java.lang.IllegalArgumentException:
  Cannot construct instance of `KafkaEventMessage`:
  no String-argument constructor/factory method to deserialize
  from String value ('{"eventId":"...","eventType":"PRODUCT_LIKED",...}')
```

Consumer가 메시지를 받긴 받았는데, 역직렬화에서 터진 겁니다. 원인을 추적해보니 **두 가지 설정 오류**가 숨어 있었습니다.

**오류 1: kafka.yml 오타**

```yaml
# 잘못된 설정 — consumer에 "value-serializer"라는 속성은 없음
consumer:
  value-serializer: org.apache.kafka.common.serialization.ByteArrayDeserializer

# 올바른 설정
consumer:
  value-deserializer: org.apache.kafka.common.serialization.ByteArrayDeserializer
```

`value-serializer`는 Consumer에 존재하지 않는 속성입니다. Spring Boot가 이걸 무시하고 기본값인 `StringDeserializer`를 사용합니다. 그래서 메시지가 `byte[]`가 아닌 `String`으로 들어온 겁니다.

**오류 2: Consumer의 역직렬화 방식**

```kotlin
// 잘못된 코드 — convertValue는 Map→Object 변환용
val message = objectMapper.convertValue(record.value(), KafkaEventMessage::class.java)

// 올바른 코드 — byte[]를 직접 역직렬화
val message = objectMapper.readValue(record.value() as ByteArray, KafkaEventMessage::class.java)
```

`convertValue()`는 이미 역직렬화된 객체(Map 등)를 타입 변환할 때 쓰는 메서드입니다. `ByteArrayDeserializer`로 받은 raw byte[]를 역직렬화하려면 `readValue()`를 써야 합니다.

두 가지를 수정하고 테스트를 다시 돌리니 전부 통과했습니다.

### 검증 3: manual ACK이 정말 "처리 후에만" offset을 커밋하는가?

멱등 처리 검증은 끝났지만, 하나 더 확인하고 싶은 게 있었습니다. 앞에서 "auto-commit은 처리 여부와 무관하게 커밋하고, manual ACK은 처리 완료 후에만 커밋한다"고 설명했는데 — 이것도 진짜 그런지 확인해봐야 합니다.

Kafka AdminClient로 Consumer Group의 committed offset을 직접 조회해서 검증했습니다.

```kotlin
@Test
fun committedOffsetAdvances_afterMessageProcessed() {
    // arrange
    val adminClient = AdminClient.create(mapOf("bootstrap.servers" to "localhost:19092"))
    val message = createLikedMessage(productId = 100, version = 1L)

    // act — 메시지 발행, 어느 파티션의 몇 번 offset에 들어갔는지 기록
    val sendResult = kafkaTemplate.send(KafkaTopics.CATALOG_EVENTS, "100", message).get()
    val sentPartition = sendResult.recordMetadata.partition()
    val sentOffset = sendResult.recordMetadata.offset()

    // 처리 완료 대기 (DB 반영 확인)
    await atMost Duration.ofSeconds(15) untilAsserted {
        val metrics = productMetricsJpaRepository.findByProductId(100)
        assertThat(metrics!!.likeCount).isEqualTo(1)
    }

    // assert — committed offset이 발행된 메시지 이후로 이동했는지 확인
    val tp = TopicPartition(KafkaTopics.CATALOG_EVENTS, sentPartition)
    await atMost Duration.ofSeconds(10) untilAsserted {
        val afterOffsets = adminClient
            .listConsumerGroupOffsets("metrics-consumer")
            .partitionsToOffsetAndMetadata().get()
        assertThat(afterOffsets[tp]?.offset()).isGreaterThan(sentOffset)
    }
}
```

이 테스트는 이런 흐름입니다:

1. 메시지 발행 전 committed offset을 기록
2. 메시지를 보내고, 어느 파티션의 몇 번 offset에 들어갔는지 확인
3. Consumer가 처리 완료한 후, committed offset이 그 뒤로 이동했는지 확인

"처리 완료 → offset 커밋" 순서가 보장되는지를 AdminClient로 직접 확인하는 겁니다.

추가로, **메시지를 발행하지 않으면 offset이 변하지 않는다**는 것도 테스트했습니다. auto-commit이었다면 `poll()` 할 때마다 offset이 커밋되지만, manual ACK에서는 `acknowledge()`를 호출하지 않는 한 offset이 그대로입니다.

```kotlin
@Test
fun committedOffsetUnchanged_whenNoMessagePublished() {
    val beforeOffsets = getCommittedOffsets("metrics-consumer")
    Thread.sleep(5000)  // auto-commit 간격(5초)만큼 대기
    val afterOffsets = getCommittedOffsets("metrics-consumer")

    // offset 변화 없음 — manual ACK이므로 acknowledge() 없이는 커밋 안 됨
    for ((tp, beforeMeta) in beforeOffsets) {
        assertThat(afterOffsets[tp]?.offset()).isEqualTo(beforeMeta.offset())
    }
}
```

두 테스트 모두 통과했습니다. manual ACK 설정이 의도대로 동작하고 있다는 증거입니다.

### 이 경험에서 느낀 것

솔직히 이 오류들은 설정 시점에는 절대 발견할 수 없는 종류였습니다. 컴파일 에러도 안 나고, 앱 기동도 정상이고, Kafka Consumer가 파티션 할당까지 잘 받습니다. **실제 메시지가 흘러야만** 터지는 문제입니다.

"설정했으니 될 거야"라고 생각했던 것들이 실은 하나도 동작하지 않고 있었다는 게, 테스트를 안 쓰면 이런 일이 벌어진다는 걸 직접 경험한 순간이었습니다.

테스트를 3가지로 나눠서 검증한 걸 정리하면:

| 검증 | 질문 | 결과 |
|------|------|------|
| 멱등 처리 | 같은 eventId 2번 보내면 likeCount가 1인가? | 통과 — event_handled에서 skip |
| 버그 발견 | 설정이 진짜 동작하고 있었나? | yaml 오타 + 역직렬화 버그 발견 |
| manual ACK | offset이 처리 후에만 커밋되나? | 통과 — AdminClient로 확인 |

---

## 전체 설정 정리 — 한 눈에 보기

| 구간 | 설정 | 값 | 안 하면? |
|------|------|-----|---------|
| Producer → Broker | acks | all | Leader에만 저장 → Leader 죽으면 유실 |
| Producer → Broker | enable.idempotence | true | 리트라이 시 동일 메시지 중복 저장 |
| Producer → Broker | max.in.flight | 5 | idempotence 시 5 초과하면 순서 깨짐 |
| Producer → Broker | retries | MAX | 3회면 일시 장애에서 불필요하게 실패 |
| Producer → Broker | delivery.timeout.ms | 120000 | retries=MAX와 세트, 120초가 실제 상한 |
| Broker → Consumer | enable-auto-commit | false | 처리 전에 offset 커밋 → 유실 |
| Broker → Consumer | ack-mode | manual | 자동 커밋과 동일한 문제 |
| Consumer 내부 | event_handled 테이블 | PK 중복 체크 | 재수신 시 중복 처리 |

이 설정들은 하나만 빠져도 구멍이 뚫립니다. `acks=all`로 Broker까지 안전하게 보내도, Consumer에서 auto-commit이면 처리 전에 유실될 수 있습니다. Producer부터 Consumer까지 **전 구간을 관통하는 설계**가 필요합니다.

---

## 마무리

> "유실은 복구 불가, 중복은 멱등 처리로 해결 가능. 그래서 모든 설정은 유실 방지 쪽으로 기울어야 한다."

Kafka 설정을 하면서 가장 크게 느낀 건, 설정값 하나하나가 독립적이지 않다는 겁니다. `acks=all`은 `enable.idempotence=true`와 세트이고, `retries=MAX`는 `delivery.timeout.ms`와 세트이며, `manual ACK`은 `event_handled` 멱등 처리와 세트입니다.

설정값을 외우는 게 아니라, **"이 구간에서 무슨 일이 벌어질 수 있고, 그걸 어떻게 막는가"**라는 질문으로 접근하면 왜 이 조합이 필요한지 자연스럽게 이해됩니다.
