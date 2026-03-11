# Round 5: 실전 읽기 최적화 구현 계획

## Context

5주차 과제는 실제 트래픽에서 자주 발생하는 **조회 병목 문제**를 해결하는 것이 목표.
상품 목록 조회 시 브랜드 필터 + 좋아요순 정렬, 상품 상세 조회에서 성능 저하가 발생할 수 있으며,
이를 **인덱스, 비정규화, Redis 캐시**로 구조적으로 해결한다.

---

## 현재 상태 (AS-IS)

| 항목 | 상태 | 비고 |
|------|------|------|
| `Product.likeCount` 비정규화 | ✅ 완료 | Atomic Update (native SQL) |
| 좋아요 등록/취소 → likeCount 동기화 | ✅ 완료 | `LikeService` → `ProductService.increment/decrementLikeCount()` |
| 인덱스 | ⚠️ 부분 | `idx_products_brand_id` (brand_id 단일)만 존재 |
| 정렬 기능 (sort 파라미터) | ❌ 미구현 | 컨트롤러에 `// todo` 주석만 존재 |
| Redis 인프라 | ✅ 완료 | Master-Replica 구성, Lettuce 클라이언트, Testcontainers 지원 |
| Redis 캐시 로직 | ❌ 미구현 | RedisTemplate 빈만 존재, 실제 캐시 사용 코드 없음 |
| 대량 테스트 데이터 | ❌ 없음 | |

---

## 과제 ① 상품 목록 조회 성능 개선 (Index + 정렬)

### 1-1. 정렬 기능 구현

#### 새 파일: `ProductSortType.kt`

- **위치**: `apps/commerce-api/src/main/kotlin/com/loopers/domain/product/ProductSortType.kt`
- 기존 프로젝트 enum 패턴(`CouponType`, `IssuedCouponStatus`)과 동일하게 심플한 enum

```kotlin
enum class ProductSortType(val fieldName: String, val direction: Sort.Direction) {
    LATEST("createdAt", Sort.Direction.DESC),
    PRICE_ASC("price", Sort.Direction.ASC),
    LIKES_DESC("likeCount", Sort.Direction.DESC),
    ;

    fun toSort(): Sort = Sort.by(direction, fieldName)
}
```

> **설계 의도**: enum에 Sort 변환 책임을 캡슐화. 컨트롤러는 enum만 받고, Pageable에 Sort를 주입하면 기존 JpaRepository 메서드를 변경 없이 재사용 가능.

#### 수정: `ProductV1ApiSpec.kt`

```kotlin
fun getAllProducts(
    @Parameter(description = "브랜드 ID (필터)")
    brandId: Long?,
    @Parameter(description = "정렬 기준", example = "LATEST")
    sort: ProductSortType?,   // 추가
    @ParameterObject pageable: Pageable,
): ApiResponse<Page<ProductV1Dto.ProductResponse>>
```

#### 수정: `ProductV1Controller.kt`

```kotlin
@GetMapping
override fun getAllProducts(
    @RequestParam(required = false) brandId: Long?,
    @RequestParam(required = false, defaultValue = "LATEST") sort: ProductSortType,
    @PageableDefault(size = 20) pageable: Pageable,  // sort 기본값 제거
): ApiResponse<Page<ProductV1Dto.ProductResponse>> {
    // sort enum → Pageable에 Sort 주입
    val sortedPageable = PageRequest.of(pageable.pageNumber, pageable.pageSize, sort.toSort())
    return productService.getAllProducts(brandId, sortedPageable)
        .map { ProductV1Dto.ProductResponse.from(it) }
        .let { ApiResponse.success(it) }
}
```

> **포인트**: `@PageableDefault`에서 sort 기본값을 제거하고, `ProductSortType` enum의 `toSort()`로 대체. Spring Data의 Pageable 정렬을 활용하므로 Repository 레이어 변경 없음.

#### 변경 없음 확인

- `ProductService.getAllProducts()` — Pageable을 그대로 전달하므로 변경 불필요
- `ProductJpaRepository.findAllByDeletedAtIsNull(pageable)` — Pageable에 Sort가 포함되어 있으면 자동으로 ORDER BY 생성
- `ProductRepositoryImpl` — 변경 불필요

---

### 1-2. 복합 인덱스 추가

#### 수정: `Product.kt` Entity (`@Table` 어노테이션)

```kotlin
@Entity
@Table(
    name = "products",
    indexes = [
        Index(name = "idx_products_brand_id", columnList = "brand_id"),                        // 기존
        Index(name = "idx_products_brand_created", columnList = "brand_id, created_at DESC"),   // 추가
        Index(name = "idx_products_brand_price", columnList = "brand_id, price ASC"),           // 추가
        Index(name = "idx_products_brand_like_count", columnList = "brand_id, like_count DESC"),// 추가
    ],
)
```

| 인덱스 | 커버하는 쿼리 | 용도 |
|--------|-------------|------|
| `idx_products_brand_created` | `WHERE brand_id = ? AND deleted_at IS NULL ORDER BY created_at DESC` | 브랜드 + 최신순 |
| `idx_products_brand_price` | `WHERE brand_id = ? AND deleted_at IS NULL ORDER BY price ASC` | 브랜드 + 가격순 |
| `idx_products_brand_like_count` | `WHERE brand_id = ? AND deleted_at IS NULL ORDER BY like_count DESC` | 브랜드 + 좋아요순 |

> **주의**: `ddl-auto: none`이므로 Entity 어노테이션만으로 실제 DB에 인덱스가 생성되지 않음. 로컬 Docker MySQL에 직접 DDL 실행 필요. 테스트 환경(Testcontainers)에서는 `ddl-auto: create`라면 자동 생성됨.

#### DDL 스크립트 (로컬 실행용)

```sql
-- 복합 인덱스 추가
CREATE INDEX idx_products_brand_created ON products(brand_id, created_at DESC);
CREATE INDEX idx_products_brand_price ON products(brand_id, price ASC);
CREATE INDEX idx_products_brand_like_count ON products(brand_id, like_count DESC);
```

---

### 1-3. 10만건 테스트 데이터 (SQL 스크립트)

#### 새 파일: `data/seed-products.sql`

- 브랜드 20개 먼저 INSERT
- 상품 10만건 INSERT (브랜드별 5000개, 가격 1000~500000 랜덤, likeCount 0~10000 랜덤)
- MySQL의 `RAND()` 함수 활용하여 다양한 분포 생성

```sql
-- 브랜드 20개 생성
INSERT INTO brands (name, description, created_at, updated_at) VALUES
('Nike', '나이키', NOW(), NOW()),
('Adidas', '아디다스', NOW(), NOW()),
-- ... 총 20개

-- 상품 10만건 생성 (프로시저 또는 반복 INSERT)
-- brand_id: 1~20 랜덤, price: 1000~500000 랜덤, like_count: 0~10000 랜덤
```

---

### 1-4. EXPLAIN 분석

인덱스 적용 전/후로 아래 쿼리들의 EXPLAIN 결과를 비교:

```sql
-- 쿼리 1: 브랜드 필터 + 최신순
EXPLAIN SELECT * FROM products
WHERE brand_id = 1 AND deleted_at IS NULL
ORDER BY created_at DESC LIMIT 20;

-- 쿼리 2: 브랜드 필터 + 가격순
EXPLAIN SELECT * FROM products
WHERE brand_id = 1 AND deleted_at IS NULL
ORDER BY price ASC LIMIT 20;

-- 쿼리 3: 브랜드 필터 + 좋아요순
EXPLAIN SELECT * FROM products
WHERE brand_id = 1 AND deleted_at IS NULL
ORDER BY like_count DESC LIMIT 20;

-- 쿼리 4: 전체 + 좋아요순 (brandId 없을 때)
EXPLAIN SELECT * FROM products
WHERE deleted_at IS NULL
ORDER BY like_count DESC LIMIT 20;
```

**확인 포인트**: `key`에 인덱스명이 뜨는지, `Extra`에 `Using filesort`가 사라졌는지, `rows` 수가 줄었는지

---

## 과제 ② 좋아요 수 정렬 구조 개선 (비정규화) — 이미 완료

| 항목 | 상태 | 위치 |
|------|------|------|
| `Product.likeCount` 필드 | ✅ | `Product.kt:49-51` |
| Atomic increment | ✅ | `ProductJpaRepository.kt:31-33` — `UPDATE products SET like_count = like_count + 1` |
| Atomic decrement | ✅ | `ProductJpaRepository.kt:35-40` — `GREATEST(like_count - 1, 0)` |
| 좋아요 등록 시 동기화 | ✅ | `LikeService.addLike()` → `productService.incrementLikeCount()` |
| 좋아요 취소 시 동기화 | ✅ | `LikeService.cancelLike()` → `productService.decrementLikeCount()` |
| 동시성 테스트 | ✅ | `LikeConcurrencyTest.kt` — 10 스레드 경쟁 테스트 |

**추가 작업**: 과제 ①의 `LIKES_DESC` 정렬이 이 과제의 최종 결과물. 블로그에 비정규화 선택 이유(JOIN + GROUP BY vs likeCount 컬럼), trade-off 정리.

---

## 과제 ③ Redis 캐시 적용

### 3-1. 캐시 전략 설계

| 대상 | 캐시 키 | TTL | 무효화 조건 |
|------|---------|-----|-----------|
| 상품 상세 | `product:detail:{productId}` | 10분 | 상품 수정, 삭제 |
| 상품 목록 | `product:list:{brandId}:{sort}:{page}:{size}` | 5분 | 상품 생성, 수정, 삭제, 좋아요 변경 |

> brandId가 null일 때 키의 brandId 부분은 `"all"` 사용

#### 캐시 흐름도

```
[클라이언트] → GET /api/v1/products/{id}
                    ↓
            [ProductService.getProductInfo()]
                    ↓
            [ProductCacheRepository.getProductDetail(id)]
                    ↓
               Redis에 키 있음? ──YES──→ JSON → ProductInfo 반환
                    │
                   NO
                    ↓
            [productRepository.findById(id)]
                    ↓
              DB 조회 → ProductInfo 변환
                    ↓
            [ProductCacheRepository.setProductDetail(id, info)]
                    ↓
              Redis에 저장 (TTL 10분)
                    ↓
                반환
```

---

### 3-2. 캐시 인프라 구현

#### 새 파일: `ProductCacheRepository.kt`

- **위치**: `apps/commerce-api/src/main/kotlin/com/loopers/infrastructure/product/ProductCacheRepository.kt`
- `RedisTemplate<String, String>`을 주입받아 사용 (기존 `defaultRedisTemplate`)
- `ObjectMapper`로 직렬화/역직렬화 (기존 jackson 모듈의 빈 재사용)

```kotlin
@Repository
class ProductCacheRepository(
    private val redisTemplate: RedisTemplate<String, String>,
    private val objectMapper: ObjectMapper,
) {
    companion object {
        private const val PRODUCT_DETAIL_PREFIX = "product:detail:"
        private const val PRODUCT_LIST_PREFIX = "product:list:"
        private val DETAIL_TTL = Duration.ofMinutes(10)
        private val LIST_TTL = Duration.ofMinutes(5)
    }

    // === 상품 상세 ===
    fun getProductDetail(productId: Long): ProductInfo? {
        return try {
            val key = "$PRODUCT_DETAIL_PREFIX$productId"
            val json = redisTemplate.opsForValue().get(key) ?: return null
            objectMapper.readValue(json, ProductInfo::class.java)
        } catch (e: Exception) {
            null  // Redis 장애 시 null → DB fallback
        }
    }

    fun setProductDetail(productId: Long, info: ProductInfo) {
        try {
            val key = "$PRODUCT_DETAIL_PREFIX$productId"
            val json = objectMapper.writeValueAsString(info)
            redisTemplate.opsForValue().set(key, json, DETAIL_TTL)
        } catch (e: Exception) {
            // Redis 장애 시 무시 (캐시 실패가 서비스 장애로 이어지지 않도록)
        }
    }

    fun evictProductDetail(productId: Long) {
        try {
            redisTemplate.delete("$PRODUCT_DETAIL_PREFIX$productId")
        } catch (e: Exception) { /* 무시 */ }
    }

    // === 상품 목록 ===
    fun getProductList(brandId: Long?, sort: String, page: Int, size: Int): Page<ProductInfo>? {
        return try {
            val key = buildListKey(brandId, sort, page, size)
            val json = redisTemplate.opsForValue().get(key) ?: return null
            objectMapper.readValue(json, object : TypeReference<RestPageImpl<ProductInfo>>() {})
        } catch (e: Exception) {
            null
        }
    }

    fun setProductList(brandId: Long?, sort: String, page: Int, size: Int, data: Page<ProductInfo>) {
        try {
            val key = buildListKey(brandId, sort, page, size)
            val json = objectMapper.writeValueAsString(data)
            redisTemplate.opsForValue().set(key, json, LIST_TTL)
        } catch (e: Exception) { /* 무시 */ }
    }

    fun evictAllProductLists() {
        try {
            val keys = redisTemplate.keys("$PRODUCT_LIST_PREFIX*")
            if (!keys.isNullOrEmpty()) {
                redisTemplate.delete(keys)
            }
        } catch (e: Exception) { /* 무시 */ }
    }

    private fun buildListKey(brandId: Long?, sort: String, page: Int, size: Int): String {
        val brand = brandId?.toString() ?: "all"
        return "$PRODUCT_LIST_PREFIX$brand:$sort:$page:$size"
    }
}
```

> **Page 직렬화 문제**: Spring의 `PageImpl`은 기본 역직렬화가 안 됨. `RestPageImpl` (커스텀 Page 구현체) 을 만들어서 JSON 직렬화/역직렬화를 지원해야 함.

#### 새 파일: `RestPageImpl.kt`

- **위치**: `apps/commerce-api/src/main/kotlin/com/loopers/infrastructure/product/RestPageImpl.kt`
- `PageImpl`을 상속하여 Jackson 역직렬화 가능하게 만든 클래스

```kotlin
class RestPageImpl<T>(
    @JsonProperty("content") content: List<T>,
    @JsonProperty("number") number: Int,
    @JsonProperty("size") size: Int,
    @JsonProperty("totalElements") totalElements: Long,
) : PageImpl<T>(content, PageRequest.of(number, size), totalElements)
```

---

### 3-3. ProductService 캐시 통합

#### 수정: `ProductService.kt`

```kotlin
@Component
class ProductService(
    private val productRepository: ProductRepository,
    private val brandRepository: BrandRepository,
    private val productCacheRepository: ProductCacheRepository,  // 추가
) {
    @Transactional(readOnly = true)
    fun getProductInfo(productId: Long): ProductInfo {
        // 1. 캐시 조회
        productCacheRepository.getProductDetail(productId)?.let { return it }

        // 2. DB 조회
        val info = ProductInfo.from(getProduct(productId))

        // 3. 캐시 저장
        productCacheRepository.setProductDetail(productId, info)

        return info
    }

    @Transactional(readOnly = true)
    fun getAllProducts(brandId: Long?, pageable: Pageable): Page<ProductInfo> {
        val sort = pageable.sort.toString()  // "createdAt: DESC" 같은 문자열
        val page = pageable.pageNumber
        val size = pageable.pageSize

        // 1. 캐시 조회
        productCacheRepository.getProductList(brandId, sort, page, size)?.let { return it }

        // 2. DB 조회
        val products = if (brandId != null) {
            productRepository.findAllByBrandId(brandId, pageable)
        } else {
            productRepository.findAll(pageable)
        }
        val result = products.map { ProductInfo.from(it) }

        // 3. 캐시 저장
        productCacheRepository.setProductList(brandId, sort, page, size, result)

        return result
    }

    @Transactional
    fun updateProduct(productId: Long, criteria: UpdateProductCriteria): ProductInfo {
        // ... 기존 로직 ...
        productCacheRepository.evictProductDetail(productId)  // 캐시 무효화
        productCacheRepository.evictAllProductLists()          // 목록 캐시도 무효화
        return ProductInfo.from(savedProduct)
    }

    @Transactional
    fun deleteProduct(productId: Long) {
        // ... 기존 로직 ...
        productCacheRepository.evictProductDetail(productId)  // 캐시 무효화
        productCacheRepository.evictAllProductLists()          // 목록 캐시도 무효화
    }

    @Transactional
    fun createProduct(criteria: CreateProductCriteria): ProductInfo {
        // ... 기존 로직 ...
        productCacheRepository.evictAllProductLists()  // 새 상품 추가 시 목록 캐시 무효화
        return ProductInfo.from(product)
    }

    @Transactional
    fun incrementLikeCount(productId: Long) {
        productRepository.incrementLikeCount(productId)
        productCacheRepository.evictProductDetail(productId)  // 좋아요 수 변경
        productCacheRepository.evictAllProductLists()          // 좋아요순 정렬 영향
    }

    @Transactional
    fun decrementLikeCount(productId: Long) {
        productRepository.decrementLikeCount(productId)
        productCacheRepository.evictProductDetail(productId)
        productCacheRepository.evictAllProductLists()
    }
}
```

---

## 단계별 부하 테스트 — "언제 어떤 최적화가 필요한가"

### 목적

트래픽을 단계적으로 늘려가면서, 각 최적화(인덱스 → 비정규화 → 캐시)가 **언제 필요해지는지** 체감하는 테스트.
"10만건에서 인덱스 없이 몇 명까지 버티는가?" → "인덱스 넣으면 얼마나 좋아지는가?" → "그래도 안 되면 캐시가 얼마나 효과 있는가?"를 숫자로 확인한다.

### 테스트 인프라

- `@SpringBootTest` + Testcontainers (MySQL, Redis)
- 기존 동시성 테스트 패턴(CountDownLatch + ExecutorService) 활용
- 테스트 파일: `ProductReadPerformanceTest.kt` (통합 테스트)

### 측정 지표

| 지표 | 설명 | 측정 방법 |
|------|------|----------|
| **평균 응답 시간** | 전체 요청의 평균 소요 시간 | `System.nanoTime()` 차이 |
| **P95 응답 시간** | 상위 5%를 제외한 최대 응답 시간 | 응답 시간 리스트 정렬 후 95번째 백분위 |
| **처리량 (TPS)** | 초당 처리된 요청 수 | 성공 건수 / 총 소요 시간 |
| **실패율** | 타임아웃 또는 에러 비율 | 실패 건수 / 전체 건수 |

### 공통 테스트 코드 구조

```kotlin
@SpringBootTest
class ProductReadPerformanceTest @Autowired constructor(
    private val productService: ProductService,
    private val productRepository: ProductRepository,
    private val brandRepository: BrandRepository,
    private val databaseCleanUp: DatabaseCleanUp,
    private val redisCleanUp: RedisCleanUp,
) {
    /**
     * 동시 요청을 시뮬레이션하고 성능 지표를 측정하는 헬퍼
     */
    private fun measureConcurrentReads(
        threadCount: Int,
        requestsPerThread: Int,
        operation: () -> Unit,
    ): PerformanceResult {
        val latch = CountDownLatch(1)
        val executorService = Executors.newFixedThreadPool(threadCount)
        val responseTimes = ConcurrentLinkedQueue<Long>()
        val failCount = AtomicInteger(0)

        val totalStart = System.nanoTime()

        repeat(threadCount) {
            executorService.submit {
                latch.await()
                repeat(requestsPerThread) {
                    try {
                        val start = System.nanoTime()
                        operation()
                        responseTimes.add(System.nanoTime() - start)
                    } catch (e: Exception) {
                        failCount.incrementAndGet()
                    }
                }
            }
        }

        latch.countDown()
        executorService.shutdown()
        executorService.awaitTermination(60, TimeUnit.SECONDS)

        val totalTime = System.nanoTime() - totalStart
        return PerformanceResult(responseTimes.toList(), failCount.get(), totalTime)
    }

    data class PerformanceResult(
        val responseTimes: List<Long>,
        val failCount: Int,
        val totalTimeNanos: Long,
    ) {
        val avgMs get() = responseTimes.average() / 1_000_000
        val p95Ms get() = responseTimes.sorted()[(responseTimes.size * 0.95).toInt()] / 1_000_000
        val tps get() = responseTimes.size.toDouble() / (totalTimeNanos / 1_000_000_000.0)
        val failRate get() = failCount.toDouble() / (responseTimes.size + failCount)
    }
}
```

---

### Phase 1: Baseline — 인덱스 없이 어디까지 버티나

> **가설**: 10만건에서 인덱스 없이 동시 사용자가 늘어나면 응답 시간이 급격히 느려질 것

#### 사전 조건
- 10만건 상품 데이터 (브랜드 20개, 브랜드별 5,000건)
- 복합 인덱스 없음 (`idx_products_brand_id` 단일 인덱스만 존재)
- 캐시 비활성화
- 비정규화 미사용 (좋아요순 정렬 시 `LEFT JOIN + GROUP BY` 사용)

#### 테스트 시나리오

```kotlin
@Nested
@DisplayName("Phase 1: 인덱스 없이 동시 조회 부하 테스트")
inner class Phase1_Baseline {

    @Test
    @DisplayName("동시_10명_인덱스없이_브랜드필터_좋아요순_조회")
    fun baseline_10threads() {
        val result = measureConcurrentReads(threadCount = 10, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)  // LIKES_DESC
        }
        println("Phase1 - 10명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
        // 기록만 하고, 기준치 없이 결과 관찰
    }

    @Test
    @DisplayName("동시_50명_인덱스없이_브랜드필터_좋아요순_조회")
    fun baseline_50threads() {
        val result = measureConcurrentReads(threadCount = 50, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase1 - 50명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }

    @Test
    @DisplayName("동시_100명_인덱스없이_브랜드필터_좋아요순_조회")
    fun baseline_100threads() {
        val result = measureConcurrentReads(threadCount = 100, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase1 - 100명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }
}
```

#### 예상 결과

| 동시 사용자 | 예상 avg 응답 | 예상 상황 |
|------------|-------------|----------|
| 10명 | 50~100ms | 느리지만 동작 |
| 50명 | 200~500ms | 체감 가능한 지연 |
| 100명 | 1000ms+ | 커넥션 풀 대기, 타임아웃 발생 가능 |

> **판단 포인트**: Full Scan + filesort가 발생하면서 응답 시간이 선형 이상으로 증가하는 지점 확인

---

### Phase 2: 인덱스 적용 — 어디까지 개선되나

> **가설**: 복합 인덱스만 추가해도 응답 시간이 크게 개선되고, 더 많은 동시 사용자를 감당 가능할 것

#### 사전 조건
- Phase 1과 동일한 10만건
- 복합 인덱스 3개 추가 (`brand_created`, `brand_price`, `brand_like_count`)
- 캐시 비활성화
- 비정규화 사용 (`likeCount` 컬럼으로 JOIN 없이 정렬)

#### 테스트 시나리오

```kotlin
@Nested
@DisplayName("Phase 2: 인덱스 + 비정규화 적용 후 동시 조회 부하 테스트")
inner class Phase2_IndexApplied {

    @Test
    @DisplayName("동시_50명_인덱스적용_브랜드필터_좋아요순_조회")
    fun indexed_50threads() {
        val result = measureConcurrentReads(threadCount = 50, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase2 - 50명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }

    @Test
    @DisplayName("동시_100명_인덱스적용_브랜드필터_좋아요순_조회")
    fun indexed_100threads() {
        val result = measureConcurrentReads(threadCount = 100, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase2 - 100명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }

    @Test
    @DisplayName("동시_200명_인덱스적용_브랜드필터_좋아요순_조회")
    fun indexed_200threads() {
        val result = measureConcurrentReads(threadCount = 200, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase2 - 200명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }
}
```

#### Phase 1 vs Phase 2 비교 기대치

| 동시 사용자 | Phase 1 (인덱스 없음) | Phase 2 (인덱스 있음) | 개선 비율 |
|------------|---------------------|---------------------|----------|
| 50명 | 200~500ms | 10~30ms | 10x~20x |
| 100명 | 1000ms+ | 20~50ms | 20x+ |
| 200명 | 타임아웃 | 50~100ms | 동작 가능 |

> **판단 포인트**: 인덱스만으로 200명까지 버틴다면, "아직 캐시는 필요 없다"고 판단 가능.
> 200명에서도 응답이 100ms 이하면 캐시 없이 인덱스만으로 충분한 규모.

---

### Phase 3: 캐시 적용 — 인덱스로도 부족할 때

> **가설**: 동시 사용자가 더 늘어나면 DB 커넥션 풀이 병목. 캐시로 DB 자체를 안 치면 극적으로 개선될 것

#### 사전 조건
- Phase 2와 동일 (인덱스 + 비정규화)
- Redis 캐시 활성화 (상품 상세 TTL 10분, 목록 TTL 5분)
- 캐시 워밍업: 테스트 전 1회 조회로 캐시 적재

#### 테스트 시나리오

```kotlin
@Nested
@DisplayName("Phase 3: 캐시 적용 후 동시 조회 부하 테스트")
inner class Phase3_CacheApplied {

    @BeforeEach
    fun warmUp() {
        // 캐시 워밍업 — 1회 조회로 캐시에 적재
        productService.getAllProducts(brandId = 1L, pageable)
        productService.getProductInfo(targetProductId)
    }

    @Test
    @DisplayName("동시_200명_캐시적용_브랜드필터_좋아요순_조회")
    fun cached_200threads() {
        val result = measureConcurrentReads(threadCount = 200, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase3 - 200명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }

    @Test
    @DisplayName("동시_500명_캐시적용_브랜드필터_좋아요순_조회")
    fun cached_500threads() {
        val result = measureConcurrentReads(threadCount = 500, requestsPerThread = 50) {
            productService.getAllProducts(brandId = 1L, pageable)
        }
        println("Phase3 - 500명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }

    @Test
    @DisplayName("동시_200명_캐시적용_상품상세_조회")
    fun cached_detail_200threads() {
        val result = measureConcurrentReads(threadCount = 200, requestsPerThread = 50) {
            productService.getProductInfo(targetProductId)
        }
        println("Phase3 상세 - 200명: avg=${result.avgMs}ms, p95=${result.p95Ms}ms, tps=${result.tps}")
    }
}
```

#### Phase 2 vs Phase 3 비교 기대치

| 동시 사용자 | Phase 2 (인덱스만) | Phase 3 (인덱스+캐시) | 이유 |
|------------|-------------------|---------------------|------|
| 200명 | 50~100ms | 1~5ms | DB 안 침, Redis 메모리 접근 |
| 500명 | 커넥션 풀 한계 | 5~10ms | DB 커넥션 불필요 |

> **핵심 체감**: 캐시 hit 시 DB를 아예 안 타기 때문에, 동시 사용자가 아무리 늘어도 DB 커넥션 풀과 무관.

---

### Phase 4: 캐시 miss 상황 — 캐시 뒤에도 인덱스가 필요한 이유

> **가설**: 캐시가 있어도 miss가 발생하면 DB를 침. 이때 인덱스가 없으면 다시 느려진다

#### 테스트 시나리오

```kotlin
@Nested
@DisplayName("Phase 4: 캐시 miss 시 인덱스 유무에 따른 차이")
inner class Phase4_CacheMissWithAndWithoutIndex {

    @Test
    @DisplayName("동시_100명_캐시miss_인덱스있음")
    fun cacheMiss_withIndex() {
        // 매 요청마다 다른 brandId로 조회 → 캐시 miss 유도
        val result = measureConcurrentReads(threadCount = 100, requestsPerThread = 10) {
            val randomBrandId = ThreadLocalRandom.current().nextLong(1, 21)
            productService.getAllProducts(brandId = randomBrandId, pageable)
        }
        println("Phase4 인덱스O: avg=${result.avgMs}ms, p95=${result.p95Ms}ms")
    }

    @Test
    @DisplayName("동시_100명_캐시miss_인덱스없음")
    fun cacheMiss_withoutIndex() {
        // 인덱스 DROP 후 동일 테스트 → Full Scan 발생
        // (Testcontainers에서 DDL로 인덱스 제거)
        val result = measureConcurrentReads(threadCount = 100, requestsPerThread = 10) {
            val randomBrandId = ThreadLocalRandom.current().nextLong(1, 21)
            productService.getAllProducts(brandId = randomBrandId, pageable)
        }
        println("Phase4 인덱스X: avg=${result.avgMs}ms, p95=${result.p95Ms}ms")
    }
}
```

> **판단 포인트**: 캐시가 있어도 인덱스는 필수. 캐시 miss, TTL 만료, Redis 장애 시 DB가 버텨야 하니까.
> 이 테스트로 "인덱스 + 캐시 = 보험 + 성능, 둘 다 있어야 안전하다"는 걸 숫자로 확인.

---

### 전체 비교 요약표 (테스트 후 채울 템플릿)

| Phase | 최적화 | 10명 avg | 50명 avg | 100명 avg | 200명 avg | 500명 avg |
|-------|--------|---------|---------|----------|----------|----------|
| 1 | 없음 (Full Scan) | _ms | _ms | _ms | 타임아웃? | - |
| 2 | 인덱스 + 비정규화 | _ms | _ms | _ms | _ms | 커넥션 한계? |
| 3 | 인덱스 + 비정규화 + 캐시 | _ms | _ms | _ms | _ms | _ms |
| 4 | 캐시만 (인덱스 없음, miss 시) | - | - | _ms | - | - |

### 최적화 도입 판단 기준 (테스트 결과로 도출)

```
Q1. 인덱스가 필요한 시점은?
→ Phase 1에서 "avg 응답 시간이 100ms를 넘는 동시 사용자 수"
→ 이 지점부터 인덱스 필수

Q2. 인덱스만으로 충분한 범위는?
→ Phase 2에서 "avg 응답 시간이 100ms 이하인 최대 동시 사용자 수"
→ 이 범위 안이면 캐시 불필요

Q3. 캐시가 필요한 시점은?
→ Phase 2에서 "커넥션 풀 대기 또는 타임아웃이 발생하는 동시 사용자 수"
→ 이 지점부터 캐시 도입 고려

Q4. 캐시만으로 인덱스를 대체할 수 있는가?
→ Phase 4에서 "캐시 miss 시 인덱스 유무에 따른 응답 시간 차이"
→ 차이가 크면 인덱스 + 캐시 둘 다 필요하다는 결론
```

---

## 구현 순서

```
Step 1: ProductSortType enum 생성 + Controller 정렬 파라미터 추가
        - ProductSortType.kt 생성
        - ProductV1ApiSpec.kt에 sort 파라미터 추가
        - ProductV1Controller.kt에서 sort → Pageable 변환

Step 2: 복합 인덱스 추가
        - Product.kt @Table indexes에 3개 추가
        - DDL 스크립트 작성
        - 설계 문서(04-erd.md) 인덱스 목록 업데이트

Step 3: 정렬 기능 테스트 작성 (단위 + 통합 + E2E)
        → 커밋: feat: 상품 목록 정렬 기능 및 복합 인덱스 구현
        → 커밋: test: 상품 목록 정렬 테스트 추가

Step 4: 10만건 테스트 데이터 SQL 스크립트 생성
        - data/seed-products.sql 작성

Step 5: EXPLAIN 분석 (인덱스 전/후 비교)
        - 로컬에서 실행, 결과를 블로그에 기록

Step 6: Redis 캐시 구현 — 상품 상세 + 목록
        - RestPageImpl.kt 생성
        - ProductCacheRepository.kt 생성
        - ProductService 캐시 연동 + 무효화

Step 7: 캐시 테스트 작성 (단위 + 통합 + E2E)
        → 커밋: feat: 상품 API Redis 캐시 적용
        → 커밋: test: 상품 캐시 테스트 추가

Step 8: 설계 문서 업데이트
        → 커밋: docs: 설계 문서에 복합 인덱스 추가
```

---

## 테스트 계획

### 단위 테스트

| 대상 | 테스트 내용 |
|------|-----------|
| `ProductSortType` | 각 enum의 `toSort()` 결과가 올바른 Sort 객체인지 |
| `ProductService` (Mock) | 캐시 hit 시 DB 미호출 확인, 캐시 miss 시 DB 호출 + 캐시 저장 확인 |
| `ProductService` (Mock) | 수정/삭제 시 `evictProductDetail` + `evictAllProductLists` 호출 확인 |

### 통합 테스트

| 대상 | 테스트 내용 |
|------|-----------|
| 정렬 기능 | `LATEST` / `PRICE_ASC` / `LIKES_DESC` 각각 올바른 순서 반환 |
| 브랜드 필터 + 정렬 | brandId 지정 + 정렬 조합 |
| 캐시 hit/miss | 첫 조회 → DB hit, 재조회 → 캐시에서 반환 |
| 캐시 무효화 | 상품 수정 후 재조회 시 DB에서 새 데이터 반환 |
| Redis 장애 fallback | Redis 비가용 시에도 DB에서 정상 조회 |

### E2E 테스트

| 대상 | 테스트 내용 |
|------|-----------|
| `GET /api/v1/products?sort=LATEST` | 200 OK, 최신순 정렬 확인 |
| `GET /api/v1/products?sort=PRICE_ASC` | 200 OK, 가격 낮은순 확인 |
| `GET /api/v1/products?sort=LIKES_DESC` | 200 OK, 좋아요 많은순 확인 |
| `GET /api/v1/products?brandId=1&sort=LIKES_DESC` | 200 OK, 브랜드 필터 + 좋아요순 |
| `GET /api/v1/products/{id}` | 200 OK, 상품 상세 (캐시 동작) |

---

## 수정 대상 파일 종합

| 파일 | 변경 내용 |
|------|----------|
| `domain/product/Product.kt` | 복합 인덱스 3개 추가 (`@Table indexes`) |
| `domain/product/ProductSortType.kt` | **신규** — 정렬 enum |
| `interfaces/api/product/ProductV1ApiSpec.kt` | sort 파라미터 추가 + Swagger |
| `interfaces/api/product/ProductV1Controller.kt` | sort → Pageable 변환 로직 |
| `application/product/ProductService.kt` | 캐시 조회/저장/무효화 로직 통합 |
| `infrastructure/product/ProductCacheRepository.kt` | **신규** — Redis 캐시 구현 |
| `infrastructure/product/RestPageImpl.kt` | **신규** — Page JSON 직렬화용 |
| `docs/design/04-erd.md` | 인덱스 목록에 복합 인덱스 3개 추가 |
| `data/seed-products.sql` | **신규** — 10만건 테스트 데이터 |

---

## 설계 문서 확인 결과

**대상 도메인**: Product, Like

### 비즈니스 규칙 (변경 없음)
- PR-01: 상품 등록 시 브랜드 필수
- PR-02: 상품 수정 시 브랜드 변경 불가
- LK-01~05: 좋아요 규칙 (기존 유지)

### 시퀀스 흐름
- 기존 상품 조회 흐름에 캐시 레이어 추가 (Controller → Service → Cache → DB)

### DB 스키마 변경
- 테이블 스키마 변경 없음
- 인덱스 3개 추가

### 구현 시 주의사항
- `ddl-auto: none`이므로 인덱스는 로컬 DB에 직접 DDL 실행
- Redis 장애 시 서비스 중단 방지 (모든 캐시 접근에 try-catch)
- `keys()` 명령은 프로덕션에서 성능 이슈 가능 — 현재는 학습 목적이므로 사용하되, 실무에서는 SCAN 또는 별도 키 관리 전략 필요
- Page 직렬화를 위한 `RestPageImpl` 커스텀 클래스 필요
