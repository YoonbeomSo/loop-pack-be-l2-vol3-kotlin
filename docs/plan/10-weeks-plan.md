# 10주차 작업 계획 - Collect · Stack · Zip

## 설계 문서 확인 결과

**대상 도메인**: Ranking 확장 (Batch Aggregation + Materialized View)

### 비즈니스 규칙
- 9주차에서 구축한 실시간 일간 랭킹(Redis ZSET) 기반 위에, **주간/월간 랭킹**을 Spring Batch로 집계
- 데이터 소스: `ranking_event_log` 테이블 (occurred_date + product_id 인덱스, eventType=VIEW/LIKE/ORDER, eventValue=trustScore/1.0/금액)
- 가중치: ranking_score_config 테이블에서 동적으로 읽기 (VIEW=0.1, LIKE=0.2, ORDER=0.6)
- 결과 저장: `mv_product_rank_weekly` (Top 100), `mv_product_rank_monthly` (Top 100)
- API: 기존 GET /api/v1/rankings에 `period` 파라미터 추가 (DAILY/WEEKLY/MONTHLY)

### 기존 인프라 현황
- **commerce-batch**: Spring Batch 의존성 구성 완료, DemoJobConfig 템플릿 존재, JobListener/ChunkListener/StepMonitorListener 구현됨
- **commerce-api**: RankingV1Controller (GET /api/v1/rankings?date&size&page), RankingService, RankingRedisRepository (일간 ZSET 조회)
- **commerce-streamer**: RankingEventLog 엔티티(occurred_date, product_id, event_type, event_value), RankingScoreConfig(가중치), 실시간 스코어링 파이프라인
- **modules/jpa**: BaseEntity, JPA+QueryDSL 설정, Testcontainers 지원
- **modules/redis**: Master-Replica 구성

### 구현 시 주의사항
- `ranking_event_log` 엔티티는 commerce-streamer에 정의 → commerce-batch에서는 **JdbcCursorItemReader**로 직접 SQL 접근 (모듈 간 엔티티 의존 없음)
- MV 엔티티는 commerce-batch(writer)와 commerce-api(reader) **양쪽에 각각 정의** (모듈 간 결합도 제거)
- `ranking_score_config` 가중치도 배치에서 JDBC로 직접 읽기 (streamer 의존 없음)
- Spring Batch 메타 테이블은 local/test에서 `initialize-schema: always`, prod에서 `never`
- **배치 모듈은 Reader/Writer 모두 JDBC 사용** — JPA `saveAll()`은 내부적으로 `merge()` 호출 → 건별 SELECT 발생 → 배치 이점 상실 (멘토링 피드백)
- **MV 정의 기준**: 원본 데이터(ranking_event_log)에서 파생되고, 재생성 가능하며, 조회 성능 최적화 목적으로만 사용. "없으면 시스템이 고장나는 데이터"는 MV가 아님 (멘토링 피드백)

---

## 구현 계획

### Step 1. MV 도메인 엔티티 및 리포지토리 (commerce-batch)

**목표**: Materialized View 테이블에 대응하는 엔티티, 리포지토리, Job 파라미터 DTO 구현

**패키지 구조**:
```
apps/commerce-batch/src/main/kotlin/com/loopers/
├── domain/ranking/
│   ├── ProductRankWeekly.kt             # MV 엔티티 (주간)
│   ├── ProductRankMonthly.kt            # MV 엔티티 (월간)
│   ├── ProductRankWeeklyRepository.kt   # 도메인 인터페이스
│   ├── ProductRankMonthlyRepository.kt  # 도메인 인터페이스
│   └── RankingPeriodType.kt             # Enum: WEEKLY, MONTHLY
└── infrastructure/ranking/
    ├── ProductRankWeeklyJdbcRepository.kt   # JDBC batch insert 구현
    └── ProductRankMonthlyJdbcRepository.kt  # JDBC batch insert 구현
```

**MV 테이블 설계**:

`mv_product_rank_weekly` / `mv_product_rank_monthly` (동일 구조):

| Column | Type | 설명 |
|--------|------|------|
| id | BIGINT (PK, AUTO_INCREMENT) | 서로게이트 키 |
| product_id | BIGINT, NOT NULL | 상품 ID |
| total_score | DOUBLE, NOT NULL | 가중치 적용된 총 점수 |
| view_count | INT, NOT NULL, DEFAULT 0 | 조회 이벤트 수 |
| like_count | INT, NOT NULL, DEFAULT 0 | 좋아요 이벤트 수 |
| order_count | INT, NOT NULL, DEFAULT 0 | 주문 이벤트 수 |
| rank | INT, NOT NULL | 사전 계산된 순위 (1-based) |
| period_start_date | DATE, NOT NULL | 기간 시작일 (주간: 월요일, 월간: 1일) |
| period_end_date | DATE, NOT NULL | 기간 종료일 (주간: 일요일, 월간: 말일) |
| created_at | TIMESTAMP, NOT NULL | 배치 실행 시간 |

**인덱스**:
- `idx_mv_weekly_period_rank` on `(period_start_date, rank)` — 메인 조회 경로
- `uk_mv_weekly_product_period` UNIQUE on `(product_id, period_start_date)` — 멱등성 보장

**핵심 구현**:
- 엔티티는 BaseEntity 상속하지 않음 (updated_at, deleted_at 불필요 — MV는 insert-only, cleanup은 delete-by-period)
- `RankingPeriodType` enum에 기간 시작/종료일 계산 로직 포함

```kotlin
enum class RankingPeriodType {
    WEEKLY, MONTHLY;

    fun periodStartDate(date: LocalDate): LocalDate = when (this) {
        WEEKLY -> date.with(DayOfWeek.MONDAY)
        MONTHLY -> date.withDayOfMonth(1)
    }

    fun periodEndDate(date: LocalDate): LocalDate = when (this) {
        WEEKLY -> date.with(DayOfWeek.SUNDAY)
        MONTHLY -> date.withDayOfMonth(date.lengthOfMonth())
    }
}
```

**Repository 인터페이스**:
- `batchInsert(entities: List<ProductRankWeekly>)` — JDBC `batchUpdate()` 사용
- `deleteByPeriodStartDate(date: LocalDate)` — JDBC 직접 DELETE

**TDD**:
- 단위 테스트: `RankingPeriodTypeTest` — 기간 계산 (월요일-일요일, 1일-말일, 월경계 엣지케이스)
- 통합 테스트: `ProductRankWeeklyRepositoryTest` — saveAll, findByPeriodStartDate, deleteByPeriodStartDate (Testcontainers)

---

### Step 2. Batch Reader / Processor / Writer 구현 (commerce-batch)

**목표**: Chunk-Oriented Processing 핵심 컴포넌트 구현

**패키지 구조**:
```
apps/commerce-batch/src/main/kotlin/com/loopers/
├── batch/job/ranking/
│   ├── step/
│   │   ├── RankingAggregationReader.kt      # JdbcCursorItemReader 기반
│   │   ├── RankingAggregationProcessor.kt   # 순위 부여
│   │   ├── RankingAggregationWriter.kt      # MV 테이블 적재
│   │   └── RankingCleanupTasklet.kt         # 기간별 기존 데이터 삭제 (멱등성)
│   └── param/
│       └── RankingJobParameter.kt           # Job 파라미터 DTO
└── domain/ranking/
    ├── ProductRankAggregation.kt            # Reader → Processor 전달 DTO
    └── ProductRankResult.kt                 # Processor → Writer 전달 DTO
```

**Reader: `RankingAggregationReader`**:
- `JdbcCursorItemReader<ProductRankAggregation>` 사용
- ranking_event_log에서 기간 내 데이터를 GROUP BY product_id로 집계
- ranking_score_config에서 가중치를 읽어 SQL 내에서 가중 합산
- ORDER BY total_score DESC, LIMIT 100 (Top 100만 읽기)
- `@StepScope`로 선언하여 Job Parameter late-binding

**집계 SQL**:
```sql
SELECT
    rel.product_id,
    SUM(CASE WHEN rel.event_type = 'VIEW'  THEN rel.event_value * :viewWeight ELSE 0 END) +
    SUM(CASE WHEN rel.event_type = 'LIKE'  THEN rel.event_value * :likeWeight ELSE 0 END) +
    SUM(CASE WHEN rel.event_type = 'ORDER' THEN rel.event_value * :orderWeight ELSE 0 END) AS total_score,
    COUNT(CASE WHEN rel.event_type = 'VIEW'  THEN 1 END) AS view_count,
    COUNT(CASE WHEN rel.event_type = 'LIKE'  THEN 1 END) AS like_count,
    COUNT(CASE WHEN rel.event_type = 'ORDER' THEN 1 END) AS order_count
FROM ranking_event_log rel
WHERE rel.occurred_date BETWEEN :startDate AND :endDate
GROUP BY rel.product_id
ORDER BY total_score DESC
LIMIT 100
```

**Processor: `RankingAggregationProcessor`**:
- Reader가 score 내림차순으로 전달하므로, 순차적으로 rank 부여 (1, 2, 3, ...)
- `ProductRankAggregation` → `ProductRankResult`(rank 포함) 변환

**Writer: `RankingAggregationWriter`**:
- **JDBC `NamedParameterJdbcTemplate.batchUpdate()` 사용** — JPA `saveAll()`은 `merge()` 내부에서 건별 SELECT 발생하므로 배치에서 부적합 (멘토링 피드백)
- periodType에 따라 `weeklyRepository.batchInsert()` 또는 `monthlyRepository.batchInsert()` 호출
- RankingJobParameter로 period_start_date, period_end_date 계산 후 파라미터 맵 생성

**Writer INSERT SQL**:
```sql
INSERT INTO mv_product_rank_weekly
    (product_id, total_score, view_count, like_count, order_count, rank, period_start_date, period_end_date, created_at)
VALUES
    (:productId, :totalScore, :viewCount, :likeCount, :orderCount, :rank, :periodStartDate, :periodEndDate, NOW())
```

**Cleanup Tasklet: `RankingCleanupTasklet`**:
- 집계 Step 전에 실행 — 대상 기간의 기존 MV 데이터 삭제
- 재실행 시 멱등성 보장 (delete → insert 패턴)

**TDD**:
- 단위 테스트: `RankingAggregationProcessorTest` — rank 순차 부여, 경계값
- 단위 테스트: `RankingAggregationWriterTest` — WEEKLY/MONTHLY 분기, 올바른 repository 호출
- 단위 테스트: `RankingCleanupTaskletTest` — 올바른 기간으로 삭제 호출
- 통합 테스트: `RankingAggregationReaderTest` — SQL 집계 정확성, 가중치 적용, 빈 데이터, LIMIT 100 (Testcontainers)

---

### Step 3. Job Config + Parallel Flow 구현 (commerce-batch)

**목표**: Spring Batch Job/Step 구성, 주간+월간 병렬 실행 Flow

**패키지 구조**:
```
apps/commerce-batch/src/main/kotlin/com/loopers/
├── batch/job/ranking/
│   └── RankingAggregationJobConfig.kt   # Job, Step, Flow 구성
```

**Job 구조 — Parallel Flow**:
```
Job: rankingAggregationJob
  ├── [Parallel Flow via split(SimpleAsyncTaskExecutor)]
  │   ├── Weekly Flow:
  │   │   ├── Step 1-W: weeklyCleanupStep (Tasklet)
  │   │   └── Step 2-W: weeklyAggregationStep (Chunk<ProductRankAggregation, ProductRankResult>)
  │   └── Monthly Flow:
  │       ├── Step 1-M: monthlyCleanupStep (Tasklet)
  │       └── Step 2-M: monthlyAggregationStep (Chunk<ProductRankAggregation, ProductRankResult>)
  └── [End]
```

**Spring Batch 장점 활용 포인트**:
1. **Chunk-Oriented Processing**: Reader→Processor→Writer로 대용량 데이터 처리 구조화, chunk size=100
2. **Parallel Flow**: `FlowBuilder.split(SimpleAsyncTaskExecutor)`로 주간/월간 동시 집계 — 배치 총 실행 시간 단축
3. **Job Parameter**: `requestDate`를 받아 기간 자동 계산, `RunIdIncrementer`로 동일 파라미터 재실행 가능
4. **Restart Strategy**: cleanup → aggregate 패턴으로 실패 후 전체 재실행 시 정합성 보장
5. **Listener**: 기존 JobListener(실행 시간 측정), ChunkListener(chunk 카운트), StepMonitorListener(실패 감지) 활용

**핵심 구현**:
- `@ConditionalOnProperty(name = ["spring.batch.job.name"], havingValue = "rankingAggregationJob")`
- 주간/월간 각각 별도 `@StepScope` Reader/Writer Bean (RankingPeriodType으로 분기)
- Processor는 stateless이므로 공유 가능하나, rank 카운터 상태가 있으므로 각각 인스턴스 생성
- chunk size = 100 (Top 100이므로 1 chunk에 전부 처리)

**실행 명령**:
```bash
java -jar commerce-batch.jar \
  --job.name=rankingAggregationJob \
  --requestDate=2026-04-14
```
→ requestDate 기준으로 해당 주 + 해당 월 동시 집계

**TDD**:
- E2E 테스트: `RankingAggregationJobE2ETest` (DemoJobE2ETest 패턴 참고)
  - ranking_event_log에 테스트 데이터 삽입 → Job 실행 → mv_product_rank_weekly + monthly 검증
  - 파라미터 누락 시 실패 검증
  - 멱등성 검증 (동일 파라미터 2회 실행 → 데이터 정합성 유지)

**배치 테스트 전략** (멘토링 피드백):
- **Input 테이블** (ranking_event_log): 매 테스트마다 적정 개수로 초기화 및 생성
- **Output 테이블** (mv_product_rank_weekly/monthly): 테스트 결과 정확성을 위해 초기화
- **Metadata 테이블** (BATCH_JOB_*): **초기화하지 않음** — JobParameters를 매 테스트마다 다르게 넣어 JobInstance 충돌 방지 (jobName + timestamp)

---

### Step 4. Ranking API 확장 (commerce-api)

**목표**: 기존 일간 랭킹 API에 주간/월간 조회 기능 추가

**패키지 구조 변경**:
```
apps/commerce-api/src/main/kotlin/com/loopers/
├── domain/ranking/
│   ├── RankingRepository.kt              (기존 — Redis 일간)
│   ├── RankingPeriodType.kt              (신규 — enum DAILY/WEEKLY/MONTHLY)
│   ├── ProductRankWeekly.kt              (신규 — 읽기 전용 엔티티)
│   ├── ProductRankMonthly.kt             (신규 — 읽기 전용 엔티티)
│   ├── ProductRankWeeklyRepository.kt    (신규 — 도메인 인터페이스)
│   └── ProductRankMonthlyRepository.kt   (신규 — 도메인 인터페이스)
├── infrastructure/ranking/
│   ├── RankingRedisRepository.kt          (기존)
│   ├── ProductRankWeeklyJpaRepository.kt  (신규)
│   ├── ProductRankWeeklyRepositoryImpl.kt (신규)
│   ├── ProductRankMonthlyJpaRepository.kt (신규)
│   └── ProductRankMonthlyRepositoryImpl.kt(신규)
├── application/ranking/
│   └── RankingService.kt                  (수정 — period 라우팅 추가)
└── interfaces/api/ranking/
    ├── RankingV1Controller.kt             (수정 — period 파라미터 추가)
    ├── RankingV1ApiSpec.kt                (수정 — Swagger 문서 추가)
    └── RankingV1Dto.kt                    (수정 — period 필드 추가)
```

**API 스펙**:
```
GET /api/v1/rankings?period=DAILY&date=20260414&size=20&page=1   (기본값: DAILY)
GET /api/v1/rankings?period=WEEKLY&date=20260414&size=20&page=1
GET /api/v1/rankings?period=MONTHLY&date=20260414&size=20&page=1
```

- `period=DAILY` (기본값): 기존 Redis ZSET 조회 (변경 없음)
- `period=WEEKLY`: date를 해당 주 월요일로 변환 → `mv_product_rank_weekly`에서 조회
- `period=MONTHLY`: date를 해당 월 1일로 변환 → `mv_product_rank_monthly`에서 조회

**RankingService 라우팅 로직**:
```kotlin
fun getRankings(date: LocalDate, page: Int, size: Int, period: RankingPeriodType): RankingPageInfo {
    return when (period) {
        RankingPeriodType.DAILY -> getDailyRankings(date, page, size)     // 기존 Redis 로직
        RankingPeriodType.WEEKLY -> getWeeklyRankings(date, page, size)   // 신규 MV 조회
        RankingPeriodType.MONTHLY -> getMonthlyRankings(date, page, size) // 신규 MV 조회
    }
}
```

**주간/월간 조회 로직**:
- MV 테이블에서 period_start_date + rank 순으로 조회 (이미 rank 사전 계산됨)
- 상품/브랜드 정보 enrichment는 기존 일간과 동일 패턴 (ProductRepository, BrandRepository 조회)
- 페이지네이션은 MV의 100건 내에서 in-memory 처리 (Top 100 고정)

**응답 DTO 변경**:
```kotlin
data class RankingPageResponse(
    val rankings: List<RankingResponse>,
    val date: String,
    val period: String,     // 신규: "DAILY", "WEEKLY", "MONTHLY"
    val page: Int,
    val size: Int,
    val totalCount: Long,
)
```

**TDD**:
- 단위 테스트: `RankingServiceTest` 수정 — period 라우팅 (DAILY→Redis, WEEKLY→weeklyRepo, MONTHLY→monthlyRepo)
- 단위 테스트: `RankingV1ControllerTest` 수정 — period 파라미터 파싱, 기본값 DAILY
- E2E 테스트: `RankingV1E2ETest` — period=WEEKLY/MONTHLY HTTP 요청/응답 검증

---

### Step 5. Nice-to-Have: 모니터링 강화 + 에러 처리 + .http 파일

**목표**: 배치 실행 모니터링, 에러 처리 정책, API 테스트 파일

#### 5-1. 모니터링 리스너 강화 (멘토링 피드백 반영)
배치 모니터링 핵심 지표:
- **실행 시간**: Job 전체 소요 시간 (스케줄링 overlap 방지 기준)
- **처리량 (TPS)**: 초당 처리 건수
- **처리 건수**: read / write / skip count — skip 급증 시 데이터 유실 의심
- **실패/에러 지표**: error rate
- **리소스 사용량**: CPU / Memory (배치 특성상 변동 가능)

리스너별 강화:
- 기존 `JobListener`: 주간/월간 기간 정보 + 전체 실행 시간 로그
- 기존 `StepMonitorListener`: read/write/skip 카운트 상세 로그 + 에러 비율
- 기존 `ChunkListener`: chunk 처리 시간 + TPS 계산 로그

#### 5-2. Skip/Retry 정책 (멘토링 피드백)
- Spring Batch의 부분 실패 처리 기능 활용
- `.faultTolerant().skipLimit(N).skip(Exception::class.java)` 설정으로 일부 데이터 오류 시 전체 실패 방지
- `.retryLimit(N).retry(TransientException::class.java)` 설정으로 일시적 오류 재시도

#### 5-3. 배치 실행 .http 파일
- `docs/http/ranking-batch.http`: 배치 실행 커맨드 예시 문서화

#### 5-4. API .http 파일
- 기존 ranking.http에 period 파라미터 포함 요청 추가

**TDD**: 별도 테스트 불필요 (리스너는 기존 테스트에서 커버, .http는 수동 테스트용)

---

## 커밋 전략

| 순서 | 타입 | 커밋 메시지 | 범위 |
|------|------|-----------|------|
| 0 | docs | `docs: 10주차 구현 계획 작성` | Step 0 (docs/plan/10-weeks-plan.md) |
| 1 | feat | `feat: 랭킹 MV 도메인 엔티티 및 배치 컴포넌트 구현` | Step 1 + Step 2 (엔티티, 리포지토리, Reader/Processor/Writer/Cleanup) |
| 2 | test | `test: 랭킹 배치 컴포넌트 단위/통합 테스트 추가` | Step 1 + Step 2 테스트 |
| 3 | feat | `feat: 랭킹 배치 Job 구성 및 Parallel Flow 적용` | Step 3 (JobConfig, parallel flow) |
| 4 | test | `test: 랭킹 배치 Job E2E 테스트 추가` | Step 3 E2E 테스트 |
| 5 | feat | `feat: 랭킹 API 주간/월간 조회 기능 확장` | Step 4 (API period 파라미터, MV 조회) |
| 6 | test | `test: 랭킹 API 주간/월간 조회 테스트 추가` | Step 4 테스트 |
| 7 | chore | `chore: 배치 모니터링 리스너 강화 및 .http 파일 추가` | Step 5 |

---

## 설계 결정 사항 (확정)

| # | 질문 | 결정 | 핵심 이유 |
|---|------|------|----------|
| 1 | 배치 집계 데이터 소스 | `ranking_event_log` | occurred_date 인덱스 존재, 날짜 기반 집계 가능. product_metrics는 날짜 차원 없음 |
| 2 | MV 엔티티 공유 방식 | 양쪽 모듈에 각각 정의 | 모듈 간 결합도 제거. 엔티티 작고(필드 8개) 변경 빈도 낮음 |
| 3 | Job 구조 | 단일 Job + Parallel Flow | Spring Batch 장점 극대화 (병렬 처리, Flow 조합, 실행 시간 단축) |
| 4 | Reader 타입 | JdbcCursorItemReader | ranking_event_log 엔티티가 streamer 모듈에 있어 JPA 사용 불가. SQL로 직접 집계 |
| 5 | Writer 타입 | JDBC `batchUpdate()` | JPA `saveAll()`은 `merge()` 내부에서 건별 SELECT 발생 → 배치 이점 상실. 실무에서 배치=JDBC, API=JPA (멘토링) |
| 6 | Score 계산 | SQL에서 가중치 적용 (ranking_score_config 읽기) | 동적 가중치 반영, 일간 랭킹과 동일한 가중 공식 사용 |
| 7 | 재시작 전략 | cleanup → aggregate (전체 재실행) | Top 100만 처리하므로 비용 무시 가능. 부분 재실행보다 정합성 보장 |
| 8 | MV PK 전략 | (product_id, period_start_date) UNIQUE + id 서로게이트 PK | 기간별 이력 유지, 멱등성 보장 |
| 9 | API period 기본값 | DAILY | 기존 API 하위 호환성 유지 |
| 10 | 일간/주간·월간 SoT 불일치 | 허용 | 일간=Redis(실시간성 우선, 부정확 허용), 주간·월간=DB MV(정확). "실시간성과 정합성은 트레이드오프" (멘토링) |
| 11 | 데이터 일관성 전략 | 스냅샷(날짜 범위 고정) | `WHERE occurred_date BETWEEN`으로 배치 실행 시점 기준 고정. 실무에서 메모리 충분하면 스냅샷 선호 (멘토링) |

---

## 검증 계획

### 1. 배치 Job 검증
```bash
# 테스트 실행 (Testcontainers)
./gradlew :apps:commerce-batch:test

# 로컬 실행
docker-compose -f docker/infra-compose.yml up -d
./gradlew :apps:commerce-batch:bootRun --args="--job.name=rankingAggregationJob --requestDate=2026-04-14"

# DB 검증
SELECT * FROM mv_product_rank_weekly WHERE period_start_date = '2026-04-13' ORDER BY rank;
SELECT * FROM mv_product_rank_monthly WHERE period_start_date = '2026-04-01' ORDER BY rank;
```

### 2. API 검증
```bash
./gradlew :apps:commerce-api:test

# 로컬 API 호출
GET http://localhost:8080/api/v1/rankings?period=DAILY&date=20260414&size=20&page=1
GET http://localhost:8080/api/v1/rankings?period=WEEKLY&date=20260414&size=20&page=1
GET http://localhost:8080/api/v1/rankings?period=MONTHLY&date=20260414&size=20&page=1
```

### 3. 전체 빌드
```bash
./gradlew build
./gradlew ktlintCheck
```

---

## 주요 참조 파일

| 파일 | 용도 |
|------|------|
| `apps/commerce-batch/src/main/kotlin/com/loopers/batch/job/demo/DemoJobConfig.kt` | Job 구성 패턴 참조 |
| `apps/commerce-batch/src/test/kotlin/com/loopers/job/demo/DemoJobE2ETest.kt` | 배치 E2E 테스트 패턴 참조 |
| `apps/commerce-streamer/src/main/kotlin/com/loopers/domain/ranking/RankingEventLog.kt` | 데이터 소스 테이블 구조 참조 |
| `apps/commerce-api/src/main/kotlin/com/loopers/application/ranking/RankingService.kt` | 기존 일간 랭킹 서비스 (수정 대상) |
| `apps/commerce-api/src/main/kotlin/com/loopers/interfaces/api/ranking/RankingV1Controller.kt` | 기존 API 컨트롤러 (수정 대상) |
| `apps/commerce-batch/src/main/kotlin/com/loopers/batch/listener/` | 기존 리스너 (강화 대상) |
