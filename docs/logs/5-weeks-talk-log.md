# 5주차 대화 로그

## 2026-03-09: 5주차 블로커 분석 — Redis 캐시 구현

### 고민한 부분
- 5주차 과제(실전 읽기 최적화)에서 인덱스, 비정규화, Redis 캐시 3가지를 해야 하는데, Redis를 안 써봐서 캐시 구현(직렬화, 무효화, Page 처리)을 어떻게 접근해야 할지 감이 안 잡힘

### 선택지
1. 한꺼번에 Redis 캐시 전체를 구현하면서 배우기
2. 블로커가 아닌 것(인덱스 + 정렬)부터 끝내고, Redis는 단계별로 쪼개서 접근

### 선택한 답
- 2번: 인덱스/정렬 먼저 하고, Redis는 단건 캐시(상품 상세) → 목록 캐시(Page 직렬화) 순서로 쪼개서 접근
- 실제 블로커는 Page 직렬화(RestPageImpl + TypeReference) 하나이고, 나머지는 RedisTemplate의 set/get/delete 패턴을 따라가면 됨

### 느낀 점
- Redis가 어렵게 느껴졌지만, 본질은 "빠른 Map<String, String>"이라는 설명에 감이 잡힘. 모르는 기술을 쪼개서 보면 실제 블로커는 생각보다 적음

---

## 2026-03-09: LIKE '%keyword%' Full Scan 문제 — 해결책 탐색

### 고민한 부분
- 중간 문자열 검색(`LIKE '%keyword%'`)은 인덱스를 타지 못하고 Full Scan이 발생한다는 걸 알게 됨
- 해결책으로 MySQL Full-Text Index와 Elasticsearch 두 가지가 있다는데, 각각 뭔지 감이 안 잡힘

### 내용 정리
- **MySQL Full-Text Index**: DB 자체 기능 확장. 텍스트를 토큰(단어) 단위로 쪼개 역인덱스를 만듦. 도입 비용 낮고 SQL만 추가하면 되지만, 한글은 ngram 파서가 필요하고 검색 품질에 한계가 있음
- **Elasticsearch**: 별도 검색 전용 서버. 형태소 분석(한글은 nori), 오타 보정, 유사어, 자동완성 등 고급 검색 가능. 대신 서버 운영 + 데이터 동기화 비용이 큼
- 핵심 차이: Full-Text는 "DB에 검색 기능 얹기", ES는 "검색 전용 DB를 따로 두기"

### 느낀 점
- 검색이 핵심 기능이 아니면 Full-Text로 충분하고, 검색이 서비스의 핵심이면 ES 도입을 고려해야 한다는 판단 기준이 생김. 기술 선택은 항상 "규모와 요구사항"에 맞춰야 함

---

## 2026-03-09: 캐시의 3가지 핵심 구성요소 학습

### 고민한 부분
- Redis 캐시를 구현하기 전에 캐시의 기본 개념(삽입, TTL, Evict)이 뭔지 정리하고 싶었음

### 내용 정리
- **삽입(Cache Write)**: 캐시에 데이터를 언제 넣을지. Cache-Aside(조회 시 없으면 DB에서 가져와 저장)가 가장 일반적. 지금 프로젝트가 이 방식
- **TTL(Time To Live)**: 캐시 데이터의 유통기한. 짧으면 정합성↑ 성능↓, 길면 정합성↓ 성능↑. 데이터 성격에 따라 조절하는 "트레이드오프 다이얼"
- **Evict(캐시 무효화)**: 데이터가 변경됐을 때 TTL 만료 전에 강제로 캐시를 지우는 것. 단건은 쉽지만 목록은 어려움(어떤 페이지가 영향받는지 모르니까)
- TTL은 보험, Evict는 능동적 대응. 둘 다 있어야 안전

### 느낀 점
- 캐시라는 게 결국 "넣고, 얼마나 두고, 언제 빼느냐" 세 가지 판단의 조합이라는 게 정리됨. 복잡해 보이지만 각각을 분리해서 보면 판단 기준이 명확해짐

---

## 2026-03-12: 복합 인덱스 3개 추가에 대한 의문 — 정말 필요한가?

### 고민한 부분
- 상품 목록 정렬 기능 구현 시, 정렬 옵션(최신순/가격순/좋아요순)마다 `(brand_id, 정렬컬럼)` 복합 인덱스를 3개 추가했는데, 이게 과도한 건 아닌지 의문이 들었음
- 인덱스를 추가하면 INSERT/UPDATE 비용이 늘어나는데, 현재 프로젝트 규모에서 꼭 필요한지 판단 근거가 부족했음

### 선택지
1. 복합 인덱스 3개 전부 추가 (정렬별 filesort 완전 제거)
2. 기존 단일 인덱스(`brand_id`)만 유지 (filesort 허용)
3. 주요 정렬(최신순) 1개만 복합 인덱스 추가

### 논의한 내용
- "브랜드당 상품 수백 건 이하면 filesort 무시할 만하다"는 말에 대해, 그 기준의 근거가 뭐냐고 질문함
- 결론: 행 크기, sort_buffer_size, 동시 요청 수, 페이지 크기 등 변수가 많아서 **경험적 추정만으로 판단할 수 없고, EXPLAIN으로 실측해야 함**
- filesort의 정확한 의미도 확인함: 인덱스로 정렬을 해결 못 할 때 메모리(sort buffer)에서 별도 정렬하는 것. buffer 초과 시 디스크 임시 파일 사용

### EXPLAIN 예상 결과 (brand_id=1 상품 10,000건 기준)
- **단일 인덱스**: rows=10,000, Extra=`Using where; Using filesort` → LIMIT 20이어도 전체 정렬 필요
- **복합 인덱스**: rows=약 20, Extra=`Using where` → 인덱스 순서대로 20건 채우면 중단 (LIMIT 최적화 가능)
- 핵심 차이: 복합 인덱스는 LIMIT과 결합 시 필요한 만큼만 읽고 멈출 수 있음

### 느낀 점
- 인덱스 추가 여부를 "감"이 아니라 EXPLAIN ANALYZE로 실측해서 판단해야 한다는 걸 배움. 특히 "수백 건이면 괜찮다"는 식의 경험적 판단은 근거가 약하고, 실제 환경 변수에 따라 달라질 수 있음
- filesort + LIMIT 조합에서 복합 인덱스의 진짜 이점은 "전체 정렬 없이 필요한 만큼만 읽을 수 있다"는 점이었음

---

## 2026-03-12: Redis 캐싱 시 Spring PageImpl 직렬화 문제 — CachedPage DTO 선택

### 고민한 부분
- Redis에 상품 목록을 캐시하려면 `Page<ProductInfo>`를 저장해야 하는데, Spring의 `PageImpl`은 Jackson 역직렬화가 불가능함
- 기본 생성자가 없고, 내부에 `Pageable`, `Sort` 같은 인터페이스 타입이 있어서 Jackson이 객체를 복원할 수 없음

### 선택지
1. **RestPageImpl** — `PageImpl`을 상속한 커스텀 클래스에 `@JsonCreator`를 달아 역직렬화 지원
   - 장점: `Page` 인터페이스 그대로 사용 가능
   - 단점: 불필요한 상속 클래스 추가, Spring 의존
2. **CachedPage DTO** — 순수 data class로 캐시 전용 DTO 사용, `toPage()`로 변환
   - 장점: Spring 의존 없는 순수 data class, Kotlin data class라 Jackson 직렬화/역직렬화가 자연스러움
   - 단점: 변환 메서드(`toPage()`, `from()`) 필요
3. **content + totalElements 분리 저장** — Page 자체를 캐시하지 않고 필드별 분리
   - 장점: 가장 단순
   - 단점: 캐시 키/값 구조가 복잡해짐

### 선택한 답
- 2번: CachedPage DTO
- 이유: Kotlin data class라 직렬화 문제가 없고, 상속도 불필요. DDD 관점에서도 인프라(Spring) 의존도가 낮음. 변환 메서드가 필요하지만 `companion object`의 `from()`과 `toPage()`로 깔끔하게 처리 가능

### 느낀 점
- PageImpl의 직렬화 문제는 Spring + Redis 조합에서 자주 나오는 문제인데, "상속으로 해결"보다 "캐시 전용 DTO 분리"가 더 깔끔함
- 캐시에 저장하는 데이터는 프레임워크에 의존하지 않는 순수한 구조가 좋음. 역직렬화 문제의 근본 원인이 프레임워크 클래스를 그대로 저장하려 한 것이었으니까

---

## 2026-03-12: Redis 캐시 TTL 설정 기준 — 상세 10분, 목록 5분의 근거

### 고민한 부분
- 상품 상세 캐시 TTL을 10분, 목록 캐시 TTL을 5분으로 설정했는데, 이 값의 명확한 근거가 없었음
- TTL을 판단하는 기준식이나 원칙이 있는지 궁금했음

### 내용 정리

**TTL 결정 공식 (경험적)**
```
TTL = 평균 데이터 변경 주기 × 허용 가능한 stale 비율
```
- 예: 상품 정보가 평균 1시간에 1번 변경, 10% stale 허용 → TTL = 60분 × 0.1 = 6분

**TTL에 영향을 주는 요소**

| 요소 | TTL 짧게 | TTL 길게 |
|------|---------|---------|
| 데이터 변경 빈도 | 높을 때 | 낮을 때 |
| 실시간성 요구 | 높을 때 | 낮을 때 |
| DB 부하 | 낮을 때 | 높을 때 |
| 캐시 히트율 목표 | 낮아도 될 때 | 높여야 할 때 |

**상세 > 목록으로 잡은 이유**
- 목록은 어떤 상품이든 변경(생성/수정/삭제/좋아요)되면 전체 목록 캐시가 무효화됨 → 변경 빈도가 더 높음
- 상세는 해당 상품만 변경될 때 무효화되므로 상대적으로 안정적

**좋아요에 캐시 무효화를 넣은 이유**
- 좋아요는 `incrementLikeCount()`/`decrementLikeCount()`로 상품 수정/삭제와 별도 경로로 `likeCount`가 변경됨
- 캐시의 `ProductInfo`에 `likeCount` 필드가 포함되어 있으므로, 무효화하지 않으면 TTL 만료까지 이전 좋아요 수가 반환되는 문제 발생

### 느낀 점
- 솔직히 10분/5분은 명확한 근거 없이 일반적으로 많이 쓰는 값으로 설정한 것. 공식은 있지만 "평균 데이터 변경 주기"를 모르는 상태에서는 적용할 수 없음
- 실제로는 부하 테스트(Step 7) 이후에 캐시 히트율과 DB 쿼리 수를 측정하고, 그 결과를 바탕으로 TTL을 조정하는 게 맞는 순서
- TTL은 "처음에 정확히 맞추는 것"보다 "측정 후 조정하는 것"이 현실적. 지금 값은 부하 테스트용 초기값일 뿐

---

## 2026-03-12: EXPLAIN ANALYZE 결과 — 복합 인덱스 전/후 비교

### 테스트 환경
- MySQL 8.0.45, 상품 100,000건, 브랜드 20개 (브랜드당 약 5,000건)
- 복합 인덱스: `(brand_id, created_at DESC)`, `(brand_id, price ASC)`, `(brand_id, like_count DESC)`

### 쿼리별 결과 비교

#### 쿼리 1: 브랜드별 최신순 정렬
```sql
SELECT * FROM products WHERE brand_id = 1 AND deleted_at IS NULL ORDER BY created_at DESC LIMIT 20;
```

| 구분 | 인덱스 있음 | 인덱스 없음 |
|------|-----------|-----------|
| 사용 인덱스 | `idx_products_brand_created` | `idx_products_brand_id` (단일) |
| 실행 방식 | Index lookup → Filter → Limit | Index lookup → Filter → **Sort** → Limit |
| 실제 읽은 행 | **20행** | **5,000행** |
| 실행 시간 | **0.08ms** | **7.58ms** |
| filesort | 없음 | 있음 (5,000건 전체 정렬) |

#### 쿼리 2: 브랜드별 가격순 정렬
```sql
SELECT * FROM products WHERE brand_id = 1 AND deleted_at IS NULL ORDER BY price ASC LIMIT 20;
```

| 구분 | 인덱스 있음 | 인덱스 없음 |
|------|-----------|-----------|
| 사용 인덱스 | `idx_products_brand_price` | `idx_products_brand_id` (단일) |
| 실제 읽은 행 | **20행** | **5,000행** |
| 실행 시간 | **0.06ms** | **13.9ms** |
| filesort | 없음 | 있음 |

#### 쿼리 3: 브랜드별 좋아요순 정렬
```sql
SELECT * FROM products WHERE brand_id = 1 AND deleted_at IS NULL ORDER BY like_count DESC LIMIT 20;
```

| 구분 | 인덱스 있음 | 인덱스 없음 |
|------|-----------|-----------|
| 사용 인덱스 | `idx_products_brand_like_count` | `idx_products_brand_id` (단일) |
| 실제 읽은 행 | **20행** | **5,000행** |
| 실행 시간 | **0.07ms** | **5.34ms** |
| filesort | 없음 | 있음 |

#### 쿼리 4: 전체 좋아요순 정렬 (brand_id 필터 없음)
```sql
SELECT * FROM products WHERE deleted_at IS NULL ORDER BY like_count DESC LIMIT 20;
```

| 구분 | 인덱스 있음 | 인덱스 없음 |
|------|-----------|-----------|
| 사용 인덱스 | 없음 (Full Table Scan) | 없음 (Full Table Scan) |
| 실제 읽은 행 | **100,000행** | **100,000행** |
| 실행 시간 | **44.2ms** | **40.1ms** |
| filesort | 있음 | 있음 |

### 분석 요약

- **브랜드별 정렬 쿼리 (1~3)**: 복합 인덱스로 읽은 행이 5,000 → 20으로 **250배 감소**, 실행 시간 **76~231배 개선**
- **전체 좋아요순 (4)**: brand_id 필터가 없어서 복합 인덱스를 활용할 수 없음. 두 경우 모두 Full Table Scan + Sort 발생
- 복합 인덱스의 핵심 이점: `LIMIT`과 결합 시 인덱스 순서대로 필요한 만큼만 읽고 중단 (Early Termination)

---

## 2026-03-12: K6 부하 테스트 결과 — 캐시 모드별 비교

### 테스트 환경
- 100 VUs, 30초, 워밍업 10초 선행
- 70% 목록 조회 (브랜드별, 랜덤 정렬/페이지) + 30% 상세 조회 (랜덤 상품 ID)
- 상품 100,000건, 브랜드 20개

### 모드별 결과 비교

| 지표 | Redis | Local (Caffeine) | Layered (Local+Redis) |
|------|-------|-------------------|----------------------|
| **http_req_duration avg** | 6.42ms | 5.60ms | **5.03ms** |
| **http_req_duration p95** | 11.77ms | 10.43ms | **8.80ms** |
| **http_req_duration p99** | - | - | - |
| **http_reqs (rate)** | 937/s | 944/s | **950/s** |
| **http_reqs (total)** | 28,156 | 28,400 | **28,555** |
| **product_list_latency avg** | 5.98ms | 5.40ms | **4.35ms** |
| **product_list_latency p95** | 10.86ms | 10.22ms | **7.02ms** |
| **product_detail_latency avg** | 7.47ms | 6.09ms | **6.61ms** |
| **product_detail_latency p95** | 13.53ms | 10.81ms | **10.74ms** |
| **errors** | 0.00% | 0.00% | 0.00% |

### 분석 요약

1. **Layered가 전체 응답 시간에서 가장 우수**: avg 5.03ms (Redis 대비 22% 개선)
2. **Local이 상세 조회에서 가장 빠름**: avg 6.09ms — 네트워크 없이 메모리 직접 접근
3. **Layered의 목록 조회가 압도적**: avg 4.35ms, p95 7.02ms — Local 캐시 히트 시 Redis 접근 불필요
4. **Redis 단독은 네트워크 오버헤드**: 로컬 캐시 없이 매번 Redis 왕복 발생
5. **모든 모드에서 에러 0%**: 안정적으로 동작

### 캐시 모드별 특성 정리

| 모드 | 장점 | 단점 |
|------|------|------|
| Redis | 서버 간 공유 가능, 데이터 일관성 | 네트워크 레이턴시 |
| Local | 가장 낮은 레이턴시 (메모리 직접 접근) | 서버별 캐시 불일치, 메모리 사용 |
| Layered | Local 히트 시 최고 성능, Redis로 일관성 보완 | 구현 복잡도, 이중 무효화 필요 |


## 상품 목록 캐시 페이지 제한 (첫 3페이지만 캐싱)

### 고민한 부분
- 좋아요(`incrementLikeCount`/`decrementLikeCount`)가 발생할 때마다 `evictAllProductLists()`로 `brandId × sortType × page × size` 전체 조합의 목록 캐시를 날리고 있었다.
- 캐시 키가 많을수록 evict 비용이 크고, evict 직후 캐시가 비어있는 상태에서 동시 요청이 몰리면 Cache Stampede 위험도 있다.
- 그렇다고 캐싱을 안 하면 매번 DB를 때려야 하고, 전부 캐싱하면 evict 비용이 문제다.
- "어디까지 캐싱하는 게 적절한가?"가 핵심 고민이었다.

### Claude가 준 선택지
1. **페이지 제한 캐싱**: 앞쪽 N페이지만 캐싱하고, 뒷페이지는 DB 직접 조회
2. **비동기 캐시 갱신 (write-behind)**: evict 대신 백그라운드에서 캐시를 갱신
3. **TTL 기반 자연 만료**: evict를 하지 않고 TTL로 일정 시간 후 자동 만료

### 내가 고른 답
- **첫 3페이지(page 0~2)만 캐싱**하는 방식을 선택했다.
- `ProductCacheService`에 `MAX_CACHED_PAGE = 3` 상수를 두고, `getProductList()`/`setProductList()`에서 page >= 3이면 캐시를 skip하도록 guard를 적용했다.
- `ProductService`는 변경하지 않았다. 캐시 miss 시 DB fallback이 이미 구현되어 있어서, 캐시 정책은 `ProductCacheService`에서만 관리하는 구조를 유지했다.

### 트레이드오프 정리

| 얻은 것 | 잃은 것 |
|---------|---------|
| 캐시 키 수 대폭 감소 | page 3+ 요청은 항상 DB 직접 조회 |
| evict 비용 감소 (삭제 대상 키가 적음) | 뒷페이지 탐색이 많으면 DB 부하 집중 가능 |
| 캐시 히트율 안정화 (evict 후 빠른 warm-up) | MAX_CACHED_PAGE = 3이라는 가정에 의존 |
| Redis/Local 메모리 절약 | 트래픽 패턴이 바뀌면 재조정 필요 |

### 느낀 점
- 캐시는 "전부 캐싱 vs 안 하기"가 아니라, 트래픽 분포를 기반으로 범위를 정하는 게 중요하다는 걸 체감했다.
- 단순한 상수 하나(`MAX_CACHED_PAGE`)로 캐시 키 폭발과 evict 비용 문제를 상당 부분 해소할 수 있다는 점이 인상적이었다.
- 다만 이 "3"이라는 숫자가 실제 서비스 트래픽에서 유효한지는 모니터링으로 검증해야 한다. 운영 환경에서는 접근 로그 기반으로 적정 페이지 수를 조정하는 것도 고려해볼 만하다.
- 정책(페이지 제한)을 `ProductCacheService`에만 두고 `ProductService`는 건드리지 않은 덕에, 캐시 전략 변경 시 영향 범위가 최소화되는 구조가 됐다.