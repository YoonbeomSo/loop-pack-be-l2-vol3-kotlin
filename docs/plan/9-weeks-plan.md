# 9주차 작업 계획 - Show Me The Ranking

## 설계 문서 확인 결과

**대상 도메인**: Ranking (신규)

### 비즈니스 규칙
- 기존 설계 문서(01-requirements.md)에 랭킹 도메인은 없음 (Round 9 신규 과제)
- Round 7에서 구축한 Kafka → commerce-streamer 파이프라인의 product_metrics 데이터를 기반으로 ZSET 랭킹 반영
- 가중치: view=0.1, like=0.2, order=0.6 (order score = price * amount 또는 정규화)

### 기존 인프라 현황
- **commerce-streamer**: CatalogEventConsumer(PRODUCT_VIEWED, PRODUCT_LIKED), OrderEventConsumer(ORDER_CREATED) 배치 리스너 존재
- **MetricsAggregationService**: viewCount, likeCount, orderCount 증감 처리 중
- **Redis**: Master-Replica 구성, `@Qualifier(REDIS_TEMPLATE_MASTER)` write 전용 템플릿 존재
- **ZSET 경험**: Round 8 대기열에서 WaitingQueueRedisRepository에 ZADD, ZRANK, ZPOPMIN 사용 경험 있음
- **Kafka**: 배치 리스너(MAX_POLLING_SIZE=3000), 멱등성(EventHandled), 3파티션 구성

### 구현 시 주의사항
- ZSET 적재는 commerce-streamer 모듈에서 수행 (Kafka Consumer 확장)
- Ranking API는 commerce-api 모듈에서 수행
- Redis ZSET 키/TTL 전략 잘 설계해야 함 (일별 키 분리, TTL 2일)
- 상품 상세 조회 응답에 랭킹 정보 추가 시 기존 캐시 전략과의 호환 고려

---

## 구현 계획

### Step 1. 랭킹 ZSET 키 전략 및 Redis Repository (commerce-streamer)

**목표**: ZSET 키 관리 유틸 + Redis 적재 Repository 구현

**패키지 구조**:
```
apps/commerce-streamer/src/main/kotlin/com/loopers/
├── domain/ranking/
│   └── RankingScorePolicy.kt          # 이벤트 타입별 가중치/점수 계산 정책
├── infrastructure/ranking/
│   └── RankingRedisRepository.kt      # ZSET ZINCRBY, ZREVRANGE, ZREVRANK, TTL 관리
```

**핵심 구현**:
- ZSET Key: `ranking:all:{yyyyMMdd}` (일간 키)
- TTL: 2일 (48시간)
- `ZINCRBY` 로 점수 누적 (member = productId)
- 점수 계산:
  - PRODUCT_VIEWED → `1 * 0.1 = 0.1`
  - PRODUCT_LIKED → `1 * 0.2 = 0.2`
  - ORDER_CREATED → `price * quantity * 0.6` (주문 금액 기반, 정규화 필요 시 log 적용 검토)

**TDD**:
- 단위 테스트: RankingScorePolicy 점수 계산 검증
- 통합 테스트: RankingRedisRepository ZINCRBY/ZREVRANGE 동작 검증 (Testcontainers Redis)

---

### Step 2. Kafka Consumer에서 ZSET 적재 연동 (commerce-streamer)

**목표**: 기존 Consumer에서 이벤트 소비 시 ZSET 점수 반영

**패키지 구조**:
```
apps/commerce-streamer/src/main/kotlin/com/loopers/
├── interfaces/consumer/
│   ├── RankingCatalogEventConsumer.kt   # 신규: 별도 consumer-group으로 catalog-events 소비
│   └── RankingOrderEventConsumer.kt     # 신규: 별도 consumer-group으로 order-events 소비
├── application/ranking/
│   └── RankingAggregationService.kt     # 신규: 랭킹 점수 집계 서비스
```

**핵심 구현**:
- **별도 Consumer Group 분리** (확정): 같은 토픽을 다른 consumer-group으로 소비
  - 기존 Metrics Consumer와 완전 독립 → 장애 격리, Kafka 설계 철학과 일관
- RankingAggregationService 신규 생성 → RankingRedisRepository 의존
- RankingCatalogEventConsumer: PRODUCT_VIEWED, PRODUCT_LIKED 이벤트 소비 → 점수 적재
- RankingOrderEventConsumer: ORDER_CREATED 이벤트 소비 → 금액 기반 점수 적재

**TDD**:
- 단위 테스트: RankingAggregationService 가중치 적용 검증
- 통합 테스트: 이벤트 소비 → ZSET 점수 반영 E2E (Kafka + Redis Testcontainers)

---

### Step 3. Ranking API 구현 (commerce-api)

**목표**: 랭킹 Page 조회 API 구현

**패키지 구조**:
```
apps/commerce-api/src/main/kotlin/com/loopers/
├── interfaces/api/ranking/
│   ├── RankingV1ApiSpec.kt            # Swagger 어노테이션
│   ├── RankingV1Controller.kt         # GET /api/v1/rankings
│   └── RankingV1Dto.kt                # Request/Response DTO
├── application/ranking/
│   ├── RankingService.kt              # 랭킹 조회 비즈니스 로직
│   └── RankingInfo.kt                 # Application DTO
├── domain/ranking/
│   └── RankingRepository.kt           # Repository 인터페이스
├── infrastructure/ranking/
│   └── RankingRedisRepository.kt      # ZSET 조회 구현체
```

**API 스펙**:
```
GET /api/v1/rankings?date=yyyyMMdd&size=20&page=1
- 인증: X (인증 불필요)
- date: 조회 대상 날짜 (기본값: 오늘)
- size: 페이지 크기 (기본값: 20)
- page: 페이지 번호 (기본값: 1)
```

**응답 구조**:
```json
{
  "meta": { "result": "SUCCESS" },
  "data": {
    "rankings": [
      {
        "rank": 1,
        "productId": 101,
        "productName": "상품명",
        "brandName": "브랜드명",
        "price": 10000,
        "imageUrl": "...",
        "score": 85.5
      }
    ],
    "date": "20260407",
    "page": 1,
    "size": 20,
    "totalCount": 100
  }
}
```

**핵심 구현**:
- `ZREVRANGE` 로 Top-N 조회 (offset 기반 페이징: start=(page-1)*size, end=page*size-1)
- `ZCARD` 로 전체 멤버 수 조회 (totalCount)
- 상품 ID 목록으로 Product 정보 일괄 조회 (ProductService or ProductRepository)
- 상품 정보 Aggregation하여 응답 구성

**TDD**:
- 단위 테스트: RankingService 페이징, 상품 정보 조합 로직 검증
- 통합 테스트: Redis ZSET → API 응답 E2E
- E2E 테스트: HTTP 요청/응답 전체 흐름 검증

---

### Step 4. 상품 상세 조회에 랭킹 정보 추가 (commerce-api)

**목표**: GET /api/v1/products/{productId} 응답에 순위 정보 추가

**변경 대상**:
```
apps/commerce-api/src/main/kotlin/com/loopers/
├── interfaces/api/product/ProductV1Dto.kt        # ProductDetailResponse에 rank 필드 추가
├── interfaces/api/product/ProductV1Controller.kt  # Controller에서 상품 + 순위 조합
```

**핵심 구현** (Controller 레벨 조합 — 확정):
- **기존 ProductInfo/ProductCacheService는 변경 없음**
- Controller에서 ProductService(캐시된 상품) + RankingService(Redis 순위) 각각 조회 후 응답에 합성
- `ZREVRANK` 로 해당 상품의 순위 조회 (0-based → 1-based 변환)
- 순위에 없으면 null 반환
- `ZSCORE` 로 해당 상품의 점수 조회 (선택)

**TDD**:
- 단위 테스트: 순위 존재/미존재 케이스 검증
- E2E 테스트: 상품 상세 조회 시 rank 포함 응답 검증

---

### Step 5. (Nice-to-Have) 콜드 스타트 해결 - Score Carry-Over

**목표**: 일자 변경 시 전일 점수 일부를 새 키에 복사

**핵심 구현**:
- `ZUNIONSTORE ranking:all:{today} 1 ranking:all:{yesterday} WEIGHTS 0.1 AGGREGATE SUM`
- 23:50 스케줄러로 다음 날 키에 carry-over 실행
- `@Scheduled` 또는 별도 Scheduler 클래스

**TDD**:
- 통합 테스트: carry-over 후 새 키에 전일 점수 * 0.1 반영 확인
- 경계값: 전일 키가 없는 경우 (첫 날) 정상 동작 확인

---

### Step 6. (Nice-to-Have) 시간 단위 랭킹

**목표**: 1시간 단위 실시간 랭킹

**핵심 구현**:
- 추가 ZSET Key: `ranking:all:{yyyyMMddHH}` (시간 단위 키)
- TTL: 3시간
- API 파라미터 확장 또는 별도 엔드포인트

---

## 커밋 전략

| 순서 | 타입 | 커밋 메시지 | 범위 |
|------|------|-----------|------|
| 1 | feat | `feat: 랭킹 ZSET 적재 기능 구현 (commerce-streamer)` | Step 1 + Step 2 |
| 2 | test | `test: 랭킹 ZSET 적재 테스트 추가` | Step 1 + Step 2 테스트 |
| 3 | feat | `feat: 랭킹 조회 API 구현 (commerce-api)` | Step 3 |
| 4 | test | `test: 랭킹 조회 API 테스트 추가` | Step 3 테스트 |
| 5 | feat | `feat: 상품 상세 조회에 랭킹 순위 추가` | Step 4 |
| 6 | test | `test: 상품 상세 조회 랭킹 정보 테스트 추가` | Step 4 테스트 |
| 7 | feat | `feat: 콜드 스타트 Score Carry-Over 구현` | Step 5 (Nice-to-Have) |
| 8 | test | `test: 콜드 스타트 Carry-Over 테스트 추가` | Step 5 (Nice-to-Have) |

---

## 설계 결정 사항 (확정)

| # | 질문 | 결정 | 핵심 이유 |
|---|------|------|----------|
| 1 | Consumer 결합 방식 | **별도 Consumer Group 분리** | Kafka 사용 이유(결합도 감소)와 일관성 유지, 대규모 시스템 패턴 학습 |
| 2 | 주문 점수 계산 | **`price * quantity * 0.6` (금액 기반)** | 과제 가이드 준수, 점수 계산 메서드 분리로 교체 용이하게 설계 |
| 3 | 상품 상세 + 랭킹 캐시 전략 | **Controller 레벨에서 조합** | 기존 캐시 구조 무변경, 관심사 분리 |
| 4 | Ranking API 인증 | **인증 불필요** | 공개 정보, 상품 조회와 동일 성격 |
| 5 | Nice-to-Have 범위 | **둘 다 진행 (콜드 스타트 → 시간 단위 랭킹)** | 학습 목적, 콜드 스타트가 선행 필수 |
