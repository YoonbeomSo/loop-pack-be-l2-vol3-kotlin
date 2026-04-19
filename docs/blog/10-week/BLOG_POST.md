# Spring Batch가 뭘까? 주기적으로? "탈락"

> 스케줄러가 배치라고 생각했던 내가, Spring Batch를 만나고 깨달은 것들

## TL;DR
- 스케줄러는 "언제 실행할지"를 정하는 것이고, 배치는 "대량 데이터를 안전하게 처리하는 것"이다. 둘은 독립적인 개념이다.
- Spring Batch가 제공하는 청크 트랜잭션, Skip/Retry, 커서 스트리밍, 메타데이터를 스케줄러에서 직접 구현하면 결국 프레임워크를 재발명하는 꼴이 된다.
- 주간/월간 랭킹 배치를 직접 구현하면서 각 기능이 왜 필요한지 확인했다.

---

## 1. 스케줄러든 Spring Batch든, 나한텐 다 같았다

회사에서는 주기적으로 도는 작업을 "스케줄러"라고 부르고, Jenkins에서 시간을 설정해서 Spring Batch Task를 실행시키는 걸 "배치"라고 불렀습니다. 스케줄러는 5분마다 캐시를 갱신하거나 상태를 체크하는 용도로, 배치는 매일 새벽에 통계를 집계하는 용도로 쓰이고 있었습니다.

저는 솔직히 이렇게 이해하고 있었습니다.

- **스케줄러** = 주기별로 계속 도는 것 (5분마다, 1시간마다)
- **Spring Batch** = 정해진 시간에 한 번 돌리는 것 (매일 새벽 2시)

둘 다 결국 "정해진 시간에 코드를 실행하는 것" 아닌가? 돌리는 주기가 다를 뿐이지.

그런데 배치를 직접 구현해보니 이 이해가 틀렸습니다.

> "Spring Batch가 뭘까? 주기적으로? — 탈락."

배치의 핵심은 "언제 돌리느냐"가 아니었습니다.

> **Batch Processing** = 모아놓고 한번에 처리하는 작업. 일괄처리.
> **Scheduling** = 언제 실행하냐를 정하는 것.
> 배치는 **"무엇을 어떻게"**, 스케줄링은 **"언제"**의 문제.

직접 만들어보니 이 구분이 체감됩니다.

Jenkins가 매일 새벽 2시에 배치를 트리거하는 건 스케줄링이고,
그 안에서 100만 건을 청크 단위로 읽고 집계해서 저장하는 건 배치 처리입니다.

Jenkins를 Cron으로 바꿔도 배치 로직은 그대로지만,
Spring Batch를 단순 스케줄러로 바꾸면 대량 처리 안전장치가 전부 사라집니다.

| 개념 | 역할 | 예시 |
|------|------|------|
| 배치(Batch) | 무엇을, 어떻게 처리할지 | 100만 건 이벤트 로그를 집계해서 랭킹 생성 |
| 스케줄링(Scheduling) | 언제 실행할지 | 매일 새벽 2시, 매주 월요일 01:00 |

---

## 2. 스케줄러로 대량 데이터를 처리하면 생기는 문제

데이터가 수백 건이면 스케줄러로 충분합니다. 문제는 데이터가 커질 때 시작됩니다.

100만 건의 이벤트 로그를 집계해서 랭킹을 만드는 시나리오를 생각해보겠습니다.

```kotlin
@Scheduled(cron = "0 0 2 * * *")
fun aggregateRanking() {
    val events = eventLogRepository.findAll() // 100만 건 전부 메모리에
    val grouped = events.groupBy { it.productId }
    val rankings = grouped.map { (productId, events) ->
        // 집계 로직...
    }
    rankingRepository.saveAll(rankings) // 한 트랜잭션에 전부
}
```

이 코드가 50만 건째에서 OOM으로 터지면:

1. **재시작 불가** — 어디까지 처리했는지 기록이 없으므로 처음부터 다시
2. **전체 롤백** — `saveAll()`이 하나의 트랜잭션이므로 전부 롤백
3. **부분 실패 처리 불가** — 100건의 데이터 오류 때문에 999,900건도 같이 실패
4. **실행 이력 없음** — 성공/실패, 처리 건수, 소요 시간을 확인할 방법이 없음

이걸 하나씩 해결하려고 체크포인트 테이블을 만들고, 트랜잭션을 분리하고, try-catch로 에러를 잡고, 실행 이력 테이블을 설계하다 보면... 결국 Spring Batch를 재발명하게 됩니다.

---

## 3. 주간/월간 랭킹 배치를 만들면서 확인한 것들

매일 쌓이는 이벤트 로그(조회/좋아요/주문)를 주간/월간 단위로 합산해서 Top 100 랭킹을 사전 계산하는 배치를 만들었습니다. 각 단계에서 Spring Batch가 어떤 역할을 하는지, 스케줄러로는 왜 안 되는지를 함께 정리합니다.

### 3-1. 커서 스트리밍 — findAll()은 안 된다

Reader의 핵심은 ranking_event_log에서 상품별 가중치 점수를 집계하는 SQL입니다.

```sql
SELECT
    rel.product_id,
    SUM(CASE WHEN rel.event_type = 'VIEW'  THEN rel.event_value * ? ELSE 0 END) +
    SUM(CASE WHEN rel.event_type = 'LIKE'  THEN rel.event_value * ? ELSE 0 END) +
    SUM(CASE WHEN rel.event_type = 'ORDER' THEN rel.event_value * ? ELSE 0 END) AS total_score,
    COUNT(CASE WHEN rel.event_type = 'VIEW'  THEN 1 END) AS view_count,
    COUNT(CASE WHEN rel.event_type = 'LIKE'  THEN 1 END) AS like_count,
    COUNT(CASE WHEN rel.event_type = 'ORDER' THEN 1 END) AS order_count
FROM ranking_event_log rel
WHERE rel.occurred_date BETWEEN ? AND ?
GROUP BY rel.product_id
ORDER BY total_score DESC
LIMIT 100
```

가중치(VIEW=0.1, LIKE=0.2, ORDER=0.6)는 `ranking_score_config` 테이블에서 읽어오므로 코드 변경 없이 동적으로 조정 가능합니다. `WHERE occurred_date BETWEEN`으로 날짜 범위를 고정하는 건 스냅샷 전략으로, 배치 실행 중에 새로 유입되는 이벤트는 다음 배치에서 처리됩니다.

`findAll()`은 결과를 `List`로 한꺼번에 메모리에 올립니다. 100만 건이면 OOM입니다.

`JdbcCursorItemReader`는 DB 커서(`ResultSet`)를 열어두고 한 건씩 가져오되, `fetchSize` 단위로 네트워크 전송을 묶어서 효율적으로 스트리밍합니다.

처음에 `List`로 전체를 올리는 Reader를 만들었다가 전환했습니다. Chunk가 500건씩 끊어서 처리하는 구조인데, Reader가 이미 전부 올려버리면 청크의 의미가 사라지기 때문입니다.

```kotlin
// Before: 전체 메모리 적재 — Chunk의 의미가 없음
val results = jdbcTemplate.query(sql, params) { rs, _ -> ... }

// After: 커서 스트리밍 — 메모리에는 청크 크기만큼만 존재
JdbcCursorItemReaderBuilder<ProductRankAggregation>()
    .dataSource(dataSource)
    .sql(sql)
    .fetchSize(500)
    .rowMapper { rs, _ -> ... }
    .build()
```

### 3-2. 청크 트랜잭션 — 500건씩 끊어서 커밋

100만 건을 한 트랜잭션으로 처리하면 실패 시 전부 롤백됩니다. DB는 롤백을 위해 undo 로그를 보관해야 하므로 트랜잭션이 길어질수록 부담도 커집니다.

Spring Batch는 청크 단위(500건)로 읽고 → 가공하고 → 저장하고 → 커밋합니다. 100만 건이면 2,000번의 커밋이 발생하고, 1,500번째에서 실패해도 이전 1,499번의 커밋은 이미 완료되어 있으므로 롤백 범위는 마지막 500건뿐입니다.

실제로 chunk size를 500으로 설정하고 돌려보니 `ChunkListener`에서 매 청크마다 `readCount`, `writeCount`가 500씩 증가하는 로그가 찍히는 걸 확인했습니다.

### 3-3. JPA가 아닌 JDBC — merge의 함정

JPA `saveAll()`은 내부적으로 `save()`를 반복 호출하고, `save()`는 `isNew()` 상태에 따라 `persist()` 또는 `merge()`를 실행합니다. ID가 할당된 엔티티면 `merge()` 경로를 타서 건별 SELECT가 먼저 발생합니다.

100건이면 SELECT 100번 + INSERT 100번 = 200번 쿼리.
JDBC `batchUpdate()`는 같은 100건을 1번의 네트워크 전송으로 처리합니다.

| 구분 | JPA saveAll() | JDBC batchUpdate() |
|------|--------------|-------------------|
| 내부 동작 | merge() → 건별 SELECT + INSERT | INSERT N건을 한 번에 전송 |
| 100건 기준 쿼리 수 | ~200회 | ~1회 (batch) |
| 1차 캐시 / 더티 체킹 | 활성 (불필요한 오버헤드) | 없음 |
| 용도 | API 레이어 (단건 CRUD) | 배치 레이어 (대량 처리) |

### 3-4. Parallel Flow — 주간 + 월간 동시 집계

주간 집계와 월간 집계는 서로 다른 테이블(`mv_product_rank_weekly`, `mv_product_rank_monthly`)에 쓰기 때문에 데이터 의존성이 없습니다. `FlowBuilder.split(SimpleAsyncTaskExecutor())`로 두 작업을 동시에 실행했습니다.

```
Job: rankingAggregationJob
  [Parallel Flow]
  ├── Weekly: Cleanup → Reader → Processor → Writer
  └── Monthly: Cleanup → Reader → Processor → Writer
```

각 Flow의 Reader/Writer가 서로의 상태를 공유하면 안 되므로, `@StepScope` Bean을 Flow별로 각각 생성해서 분리했습니다.

### 3-5. Cleanup → Aggregate — 멱등성 확보

배치가 중간에 실패하고 다시 실행하면 데이터가 중복될 수 있습니다. INSERT만 하는 구조에서 50건을 저장한 뒤 실패하면, 재실행 시 UNIQUE 제약조건 위반이 발생하거나, 제약이 없다면 150건이 됩니다.

집계 전에 해당 기간의 기존 데이터를 먼저 DELETE하면 해결됩니다.

```
1차 실행: DELETE (0건) → INSERT 50건 → 실패!
2차 실행: DELETE (50건 삭제) → INSERT 100건 → 성공!
```

Spring Batch는 이 패턴을 Tasklet Step으로 자연스럽게 지원합니다. 체크포인트 기반 재시작(실패 지점부터 이어서 처리)도 제공하지만, 랭킹처럼 전체 데이터를 합산해야 결과가 의미 있는 작업에서는 Cleanup → Aggregate가 정합성이 더 확실합니다.

E2E 테스트에서 동일 파라미터로 2회 실행해도 데이터가 정확히 유지되는 걸 확인했습니다.

### 3-6. Skip/Retry — 부분 실패 처리

1000만 건 중 100건이 데이터 오류여도 전체가 죽지 않게 하려면, 스케줄러에서는 건별 try-catch에 재시도 로직까지 직접 짜야 합니다. Spring Batch는 설정 몇 줄로 됩니다.

```kotlin
.faultTolerant()
.skipLimit(100)
.skip(DataIntegrityViolationException::class.java)
.retryLimit(3)
.retry(TransientDataAccessException::class.java)
```

스킵된 건수는 `BATCH_STEP_EXECUTION`의 `skip_count`에 자동 기록됩니다.

### 3-7. 메타데이터 — 실행 이력 자동 기록

Spring Batch는 실행할 때마다 `BATCH_JOB_EXECUTION`(Job 시작/종료/상태/파라미터)과 `BATCH_STEP_EXECUTION`(Step별 read/write/skip count)에 자동으로 기록합니다.

스케줄러에서 이런 이력을 남기려면 테이블 설계 + 시작/종료 기록 + 카운팅 + 에러 상태 업데이트를 전부 직접 짜야 합니다.

---

## 4. 스케줄러가 나쁜 게 아니다

`@Scheduled`는 스케줄링 도구로서 충분합니다. 캐시 갱신, 상태 체크, 만료 데이터 삭제처럼 데이터 건수가 적고 실패해도 다음 주기에 다시 처리되는 작업에는 `@Scheduled`가 맞습니다.

문제는 대량 데이터 처리까지 스케줄러에 맡기는 것입니다.

| 상황 | 선택 | 이유 |
|------|------|------|
| 5분마다 캐시 갱신 (수십 건) | @Scheduled | 실패해도 다음 주기에 재갱신 |
| 매일 만료 쿠폰 삭제 (수백 건) | @Scheduled | 건수가 적고 단순 DELETE |
| 매일 100만 건 이벤트 로그 집계 | Spring Batch | 메모리 관리, 청크 트랜잭션 필요 |
| 중간 실패 시 재시작이 필요 | Spring Batch | 체크포인트 / Cleanup-Aggregate |
| 처리 이력과 모니터링이 필요 | Spring Batch | 메타데이터 테이블 자동 기록 |
| 부분 실패(Skip/Retry)가 필요 | Spring Batch | faultTolerant 설정 |

Spring Batch가 필요한 기준은 명확합니다. **데이터가 많거나, 실패 복구가 필요하거나, 실행 이력을 추적해야 할 때.**

---

## 마무리

이전에는 스케줄러와 배치의 차이를 "돌리는 주기"로 구분했는데, 직접 Spring Batch로 랭킹 집계를 구현하면서 그 구분이 틀렸다는 걸 확인했습니다.

스케줄러는 "언제", 배치는 "어떻게"의 문제입니다. 그리고 Spring Batch가 제공하는 청크 트랜잭션, 커서 스트리밍, 실패 복구, 메타데이터는 대량 데이터를 다루는 순간 직접 만들기엔 부담이 큰 것들이었습니다.

다음에 대량 데이터 처리를 만나면 "이거 스케줄러로 충분한가, 배치 프레임워크가 필요한가"를 먼저 따져볼 수 있게 됐습니다.
