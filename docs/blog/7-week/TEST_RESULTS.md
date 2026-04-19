# Kafka 설정 검증 테스트 결과

## 테스트 환경
- Docker Compose: Kafka(19092), MySQL(3306), Redis(6379), kafka-ui(9099)
- commerce-api: localhost:8080
- commerce-streamer: localhost:8081 (별도 기동)

---

## 테스트 1: Outbox → Kafka 발행 검증 (acks=all + idempotence)

### 절차
1. 회원가입 + 브랜드 + 상품 생성
2. 좋아요 API 호출
3. Outbox 상태 확인 (PENDING → SENT)
4. Kafka 토픽에 메시지 적재 확인

### 결과

**Outbox 상태:**
```
id  aggregate_type  event_type      status  partition_key  sent_at
3   PRODUCT         PRODUCT_LIKED   SENT    1              2026-03-27 00:15:41
2   PRODUCT         PRODUCT_UNLIKED SENT    1              2026-03-27 00:15:23
1   PRODUCT         PRODUCT_LIKED   SENT    1              2026-03-26 23:59:24
```

**Kafka 토픽 (catalog-events):**
```
Partition 0: offsetMax=203
Partition 1: offsetMax=12
Partition 2: offsetMax=9
```

**likeCount:** 1 (ApplicationEvent 핸들러로 즉시 반영)

**결론:** Outbox PENDING → SENT 전환 정상. acks=all + idempotence=true 설정으로 Kafka에 메시지가 안전하게 적재됨.

---

## 테스트 2: auto-commit 유실 검증

### 절차
(추후 실행)

---

## 테스트 3: manual ACK 안전 검증

### 절차
(추후 실행)

---

## Consumer Group offset 현황

### metrics-consumer (commerce-streamer 미기동 상태)
```
GROUP            TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
metrics-consumer catalog-events  0          195             203             8
metrics-consumer catalog-events  1          12              12              0
metrics-consumer catalog-events  2          8               9               1
metrics-consumer order-events    0          76              76              0
metrics-consumer order-events    1          12              12              0
metrics-consumer order-events    2          8               8               0
```

LAG > 0인 파티션은 commerce-streamer가 꺼져있어서 미소비 상태.
