# 3주차 구현 계획 - Domain Modeling

> 브랜치: `feat/round3-domain-modeling`
> 최종 업데이트: 2026-02-24

---

## 진행 상황 요약

| Phase | 도메인 | 상태 | 비고 |
|:-----:|--------|:----:|------|
| 1 | **Brand** | **완료** | Entity, Facade, Service, Infra, Controller, ApiSpec, Swagger, 인증, E2E |
| 2 | Product | 미시작 | Brand 완료 후 진행 |
| 3 | Like | 미시작 | Product 완료 후 진행 |
| 4 | Order | 미시작 | Product 완료 후 진행 |

---

## 구현 대상 (총 21개 API)

| 도메인 | Customer API | Admin API | 핵심 패턴 |
|--------|:-:|:-:|------|
| Brand | 1개 | 5개 | Soft Delete, 연쇄 삭제 |
| Product | 2개 | 5개 | 비관적 락, 재고 관리 |
| Like | 3개 | - | 멱등성, Soft Delete 복원 |
| Order | 3개 | 2개 | 스냅샷, 부분 주문 |

---

## 아키텍처 의사결정

### 1. Service는 application 레이어, Facade는 2+ 서비스 조합 시에만

- ~~Facade는 처음부터 도입~~ → 3주차 리팩토링으로 변경
- **Service를 `application/` 레이어로 이동**하고 `@Transactional`을 Service에 부여한다.
- **Facade는 2+ 서비스 조합이 필요한 경우에만 유지**한다. (예: BrandFacade.deleteBrand - cascade soft delete, ProductFacade.createProduct - 브랜드 검증)
- 단일 서비스 호출: `Controller → Service (@Transactional) → Repository (인터페이스)`
- 교차 도메인 호출: `Controller → Facade (@Transactional) → Services (@Transactional, REQUIRED 전파로 같은 트랜잭션 참여)`

### 2. protected set → private set

- Kotlin final 클래스에서 `protected set`은 `private set`과 동일
- User, Brand 모두 `private set`으로 통일

### 3. Soft Delete

- `BaseEntity.delete()` / `restore()` 활용 (멱등)
- Repository에서 `deletedAtIsNull` 조건으로 활성 데이터만 조회

### 4. API 공통 규칙 (Brand 구현 시 확립)

- **인증**: 설계 문서의 인증 컬럼에 따라 헤더를 받아 처리
  - 대고객 인증: `X-Loopers-LoginId` + `X-Loopers-LoginPw` → `AuthService.authenticate()`
  - 어드민 인증: `X-Loopers-Ldap: "loopers.admin"` → Controller에서 `validateAdminAuth()` (required = false, 수동 검증)
- **Swagger**: ApiSpec 인터페이스에 어노테이션 분리 (Controller는 Spring 어노테이션만)
- **DTO 차별화**: Customer용 / Admin용 응답 DTO 분리 (Admin에 createdAt, updatedAt 등 추가)
- **테이블명**: ERD 기준 복수형 (brands, products, likes, orders, order_items)

---

## 태스크 목록 (의존성 순서)

### Phase 1: Brand — 완료

| # | 태스크 | 상태 | 생성 파일 |
|---|--------|:----:|-----------|
| 3 | Brand Entity + 도메인 단위 테스트 | **완료** | `Brand.kt`, `BrandTest.kt` (11 tests) |
| 4 | Brand Repository + Service + 서비스 단위 테스트 | **완료** | `BrandRepository.kt`, `BrandCommand.kt`, `BrandService.kt`, `BrandServiceTest.kt` (11 tests) |
| 5 | Brand Infrastructure | **완료** | `BrandJpaRepository.kt`, `BrandRepositoryImpl.kt` |
| 6 | Brand Facade | **완료** | `BrandFacade.kt` (@Transactional 관리) |
| 7 | Brand Controller + ApiSpec + DTO + E2E | **완료** | Controller, ApiSpec, Dto, E2E (16 tests) |

**Brand 구현 시 확립된 패턴:**
- Facade에서 @Transactional 관리 (DIP 준수)
- 어드민 인증: `@RequestHeader("X-Loopers-Ldap", required = false)` + `validateAdminAuth()`
- ApiSpec 인터페이스에 Swagger 분리
- Admin/Customer 응답 DTO 분리 (`BrandAdminResponse` with createdAt/updatedAt)
- 수정 시 브랜드명 중복 검사 (`existsByNameAndIdNot`)

**Brand 테스트 총 38개** (Entity 11 + Service 11 + E2E 16)

### Phase 2: Product — 미시작 (다음 진행)

| # | 태스크 | 상태 | 의존 |
|---|--------|:----:|:----:|
| 8 | Product Entity + 도메인 단위 테스트 | pending | - |
| 9 | Product Repository + Service + 서비스 단위 테스트 | pending | #8 |
| 10 | Product Infrastructure (비관적 락 포함) | pending | #9 |
| 11 | Product Facade + Application DTO | pending | #10 |
| 12 | Product Controller + ApiSpec + DTO + E2E | pending | #11 |
| 12-1 | BrandFacade 수정 — 브랜드 삭제 시 상품 연쇄 soft delete (BR-01) | pending | #10 |

**Product Entity 필드** (03-class-diagram, 04-erd 기준):
- brandId: Long, name: String, price: BigDecimal, stock: Int
- description: String?, imageUrl: String?
- `@Table(name = "products")`

**Product Entity 메서드:**
- `decreaseStock(quantity: Int)` — stock >= 0 보장 (PR-03)
- `hasEnoughStock(quantity: Int): Boolean`
- `softDelete()`, `isDeleted()` — BaseEntity 활용

**비즈니스 규칙:**
- PR-01: 등록 시 브랜드 존재 검증 (활성 브랜드만, Facade에서 BrandService 호출)
- PR-02: 수정 시 브랜드 변경 불가 (Service에서 검증)
- PR-03: 재고 >= 0 (Entity 도메인 레벨 검증)
- BR-01: 브랜드 삭제 시 연관 상품 일괄 soft delete (BrandFacade → ProductService)

**Product API 인증** (01-requirements 기준):
- C05 `GET /api/v1/products`: 인증 X
- C06 `GET /api/v1/products/{productId}`: 인증 X
- A06~A10: 어드민 인증 (`X-Loopers-Ldap`)

**Product DTO 차별화** (01-requirements 섹션 8):
- Customer: 공개 정보 (id, brandId, name, price, description, imageUrl)
- Admin: + stock, createdAt, updatedAt

**Product API 상세:**
- C05: `GET /api/v1/products?brandId={brandId}&sort={sort}&page={page}&size={size}`
  - 정렬: latest(필수), price_asc(선택), likes_desc(선택)
- C06: `GET /api/v1/products/{productId}` — 브랜드 정보 포함
- A06: `GET /api-admin/v1/products?page=0&size=20&brandId={brandId}`
- A07: `GET /api-admin/v1/products/{productId}`
- A08: `POST /api-admin/v1/products` — brandId 필수, 브랜드 존재 검증
- A09: `PUT /api-admin/v1/products/{productId}` — brandId 변경 불가
- A10: `DELETE /api-admin/v1/products/{productId}`

**시퀀스 참조:**
- 02-sequence-diagrams #5: 상품 등록 시 Facade → BrandService.getBrand() → ProductService.create()
- 02-sequence-diagrams #4: 브랜드 삭제 시 BrandFacade → ProductService.softDeleteByBrandId() → BrandService.softDelete()

**인프라 특이사항:**
- 비관적 락: `@Lock(PESSIMISTIC_WRITE)` findAllByIdIn (Order Phase에서 사용, 미리 준비)
- 인덱스: `idx_products_brand_id` (brand_id)

### Phase 3: Like — 미시작

| # | 태스크 | 상태 | 의존 |
|---|--------|:----:|:----:|
| 13 | Like Entity + 도메인 단위 테스트 | pending | #12 |
| 14 | Like Repository + Service + 서비스 단위 테스트 | pending | #13 |
| 15 | Like Infrastructure | pending | #14 |
| 16 | Like Facade + Controller + ApiSpec + DTO + E2E | pending | #15 |

**Like Entity 필드** (03-class-diagram, 04-erd 기준):
- userId: Long, productId: Long
- createdAt, deletedAt (BaseEntity)
- `@Table(name = "likes")`

**Like Entity 메서드:**
- `softDelete()`, `restore()`, `isDeleted()` — BaseEntity 활용

**비즈니스 규칙:**
- LK-01: 중복 좋아요 불가 (UK 인덱스: user_id, product_id)
- LK-02: 이미 좋아요 → 200 OK (멱등성, Soft Delete 상태면 restore)
- LK-03: 좋아요 없는 상태에서 취소 → 200 OK (멱등성, no-op)
- LK-04: 본인 좋아요만 조회 (userId == 로그인 유저 ID)
- LK-05: 삭제된 상품에도 좋아요 가능

**Like API 인증** (01-requirements 기준):
- C07 `POST /api/v1/products/{productId}/likes`: 대고객 인증 O
- C08 `DELETE /api/v1/products/{productId}/likes`: 대고객 인증 O
- C09 `GET /api/v1/users/{userId}/likes`: 대고객 인증 O

**시퀀스 참조:**
- 02-sequence-diagrams #2: 좋아요 등록 — Facade → ProductService.getProductIncludingDeleted() → LikeService
  - 3가지 분기: 이미 좋아요(유지) / Soft Delete 상태(restore) / 없음(신규 생성)
- 02-sequence-diagrams #3: 좋아요 취소 — Facade → LikeService.findActiveLike() → softDelete or no-op

**인프라 특이사항:**
- 유니크 인덱스: `uk_likes_user_product` (user_id, product_id)
- 인덱스: `idx_likes_user_id` (user_id) — 좋아요 목록 조회

**Facade 의존관계:**
- LikeFacade → LikeService + ProductService (상품 존재 검증, deletedAt 무관 조회 필요)

### Phase 4: Order — 미시작

| # | 태스크 | 상태 | 의존 |
|---|--------|:----:|:----:|
| 17 | Order/OrderItem Entity + 도메인 단위 테스트 | pending | #12 |
| 18 | Order Repository + Service + 서비스 단위 테스트 | pending | #17 |
| 19 | Order Infrastructure | pending | #18 |
| 20 | Order Facade (부분 주문) + Facade 단위 테스트 | pending | #19 |
| 21 | Order Controller + ApiSpec + DTO + E2E | pending | #20 |

**Order Entity 필드** (03-class-diagram, 04-erd 기준):
- userId: Long, totalAmount: BigDecimal
- createdAt (삭제 불가이므로 deletedAt 없음)
- `@Table(name = "orders")`

**OrderItem Entity 필드:**
- orderId: Long, productId: Long, quantity: Int
- unitPrice: BigDecimal, productName: String, brandName: String? (스냅샷)
- `@Table(name = "order_items")`

**Order Entity 메서드:**
- `calculateTotalAmount(): BigDecimal` — OrderItem.subtotal() 합산

**OrderItem Entity 메서드:**
- `subtotal(): BigDecimal` — unitPrice × quantity

**비즈니스 규칙:**
- OR-01: 주문 시 상품 정보 스냅샷 저장 (unitPrice, productName, brandName)
- OR-02: 재고 확인 + 차감 필수
- OR-03: 일부 재고 부족 → 부분 주문 (200 OK, orderedItems + excludedItems)
- OR-04: 전체 재고 부족 → 400 Bad Request
- OR-05: 본인 주문만 조회
- OR-06: 주문 삭제 불가 (기록 보존)

**Order API 인증** (01-requirements 기준):
- C10 `POST /api/v1/orders`: 대고객 인증 O
- C11 `GET /api/v1/orders?startAt={startAt}&endAt={endAt}`: 대고객 인증 O
- C12 `GET /api/v1/orders/{orderId}`: 대고객 인증 O
- A11 `GET /api-admin/v1/orders?page=0&size=20`: 어드민 인증
- A12 `GET /api-admin/v1/orders/{orderId}`: 어드민 인증

**Order DTO 차별화** (01-requirements 섹션 8):
- Customer: 본인 주문만 (orderId, totalAmount, items, createdAt)
- Admin: 전체 주문 + 유저 정보

**Order API 상세:**
- C10: 주문 생성 — `{ items: [{ productId, quantity }] }`
- C11: 주문 목록 — `startAt`, `endAt` 필수 (yyyy-MM-dd), **페이징이 아닌 기간 조회**
- C12: 주문 상세 — 본인 주문만 조회 가능
- A11: 어드민 주문 목록 — 페이징
- A12: 어드민 주문 상세 — 모든 주문 조회 가능

**시퀀스 참조:**
- 02-sequence-diagrams #1: 주문 생성 흐름
  - Facade → ProductService.getProductsWithLock() (비관적 락)
  - Facade에서 재고 분류 (orderedItems / excludedItems)
  - 전체 부족 → 400, 부분/전체 가능 → 재고 차감 → 주문 생성
  - 트랜잭션 범위: 락 획득 → 재고 차감 → 주문 생성 전체

**Facade 의존관계:**
- OrderFacade → OrderService + ProductService (비관적 락, 재고 차감)

**인프라 특이사항:**
- 비관적 락: ProductRepository.findAllByIdWithLock() — `@Lock(PESSIMISTIC_WRITE)`
- 인덱스: `idx_orders_user_id` (user_id), `idx_orders_created_at` (created_at), `idx_order_items_order_id` (order_id)
- Order ↔ OrderItem: `@OneToMany` 컴포지션 (생명주기 동일)

### 마무리

| # | 태스크 | 상태 | 의존 |
|---|--------|:----:|:----:|
| 22 | 전체 빌드 + ktLint 검증 | pending | #16, #21 |

---

## 각 도메인 공통 작업 패턴 (TDD)

1. **Entity + 도메인 단위 테스트** (Red → Green → Refactor, Mock 없이)
2. **Repository Interface** (도메인 레이어)
3. **Service + 서비스 단위 테스트** (Mockito)
4. **Repository 구현체** (인프라 레이어)
5. **Facade** (@Transactional, 교차 도메인 조합)
6. **Controller + ApiSpec + DTO** (인증 헤더, Swagger, Admin/Customer DTO 분리)
7. **E2E 테스트** (TestRestTemplate + Docker, 인증 포함)

---

## 생성된 파일 현황

### Brand (Phase 1 완료)

```
apps/commerce-api/src/main/kotlin/com/loopers/
├── application/brand/
│   └── BrandFacade.kt
├── domain/brand/
│   ├── Brand.kt
│   ├── BrandRepository.kt
│   ├── BrandService.kt
│   └── BrandCommand.kt
├── infrastructure/brand/
│   ├── BrandJpaRepository.kt
│   └── BrandRepositoryImpl.kt
└── interfaces/api/brand/
    ├── BrandV1ApiSpec.kt
    ├── BrandAdminV1ApiSpec.kt
    ├── BrandV1Controller.kt
    ├── BrandAdminV1Controller.kt
    └── BrandV1Dto.kt

apps/commerce-api/src/test/kotlin/com/loopers/
├── domain/brand/
│   ├── BrandTest.kt            (11 tests)
│   └── BrandServiceTest.kt     (11 tests)
└── interfaces/api/brand/
    └── BrandV1ApiE2ETest.kt     (16 tests)
```

### 기존 코드 변경

- `User.kt`: `protected set` → `private set` (5곳)

---

## 검증 방법

```bash
# 단위 테스트
./gradlew :apps:commerce-api:test --tests "com.loopers.domain.brand.*"

# E2E 테스트 (Docker 필요)
./gradlew :apps:commerce-api:test --tests "com.loopers.interfaces.api.brand.*"

# 전체 빌드
./gradlew build

# ktLint
./gradlew ktlintCheck
```

---

## 다음 세션에서 이어서 진행하는 방법

1. `docs/plan/3-weeks-plan.md` 읽기
2. Task #8 (Product Entity + 도메인 단위 테스트)부터 시작
3. TDD 순서: 테스트 먼저 → 구현 → 리팩토링
4. Phase 2에서 BrandFacade 수정 필요 (BR-01: 브랜드 삭제 시 연관 상품 연쇄 soft delete)
5. ProductFacade에서 BrandService 의존 (PR-01: 등록 시 브랜드 존재 검증)
6. 각 API의 인증 헤더, Swagger ApiSpec, Admin/Customer DTO 분리를 빠뜨리지 않도록 주의
