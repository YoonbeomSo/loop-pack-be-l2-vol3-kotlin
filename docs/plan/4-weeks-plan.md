# 4주차 구현 계획: 쿠폰 도메인 + 주문 쿠폰 적용 + 동시성 제어

## 목표

3주차까지 Brand, Product, Like, Order 도메인 구현 완료 상태에서:
1. **쿠폰(Coupon) 도메인 신규 구현**
2. **주문에 쿠폰 적용 로직 추가**
3. **동시성 제어 및 테스트**

## 핵심 설계 결정

| 항목 | 결정 | 이유 |
|------|------|------|
| 쿠폰 동시성 제어 | 비관적 락 (`SELECT FOR UPDATE`) | 재고와 동일 패턴, 정합성 보장 |
| 쿠폰 + 부분 주문 | 쿠폰 사용 시 부분 주문 불가 | 할인 금액 재계산 복잡성 회피 |
| IssuedCoupon 엔티티 | BaseEntity 미상속, 독자 구현 | Soft Delete 불필요, status로 생명주기 관리 |
| 좋아요 동시성 검증 | likes 테이블 COUNT 조회 | likeCount 필드 추가 없이 검증 |

---

## Phase 1: Coupon 도메인 엔티티 & 단위 테스트 (TDD) - **완료**

### 구현 파일
- `domain/coupon/CouponType.kt` - FIXED, RATE 열거형
- `domain/coupon/IssuedCouponStatus.kt` - AVAILABLE, USED, EXPIRED 열거형
- `domain/coupon/Coupon.kt` - 쿠폰 엔티티 (BaseEntity 상속, Soft Delete)
- `domain/coupon/IssuedCoupon.kt` - 발급 쿠폰 엔티티 (독자 구현)
- `domain/coupon/CouponRepository.kt` - 쿠폰 리포지토리 인터페이스
- `domain/coupon/IssuedCouponRepository.kt` - 발급 쿠폰 리포지토리 인터페이스

### 테스트 파일
- `CouponTest.kt` - 12개 테스트 (생성, 할인 계산, 만료 확인, 최소 주문 금액)
- `IssuedCouponTest.kt` - 11개 테스트 (사용, 사용 가능 여부, 소유자 검증)

### 주요 도메인 로직
- `Coupon.calculateDiscount()` - FIXED/RATE 할인 금액 계산
- `Coupon.validateMinOrderAmount()` - 최소 주문 금액 검증
- `IssuedCoupon.use()` - AVAILABLE → USED 상태 전환
- `IssuedCoupon.validateOwner()` - 쿠폰 소유자 검증

---

## Phase 2: Coupon 서비스 & 단위 테스트 (TDD) - **완료**

### 구현 파일
- `application/coupon/CouponService.kt` - 쿠폰 CRUD
- `application/coupon/IssuedCouponService.kt` - 쿠폰 발급/조회/사용
- `application/coupon/CouponCriteria.kt` - 생성/수정 요청 DTO
- `application/coupon/CouponInfo.kt` - 쿠폰 응답 DTO
- `application/coupon/IssuedCouponInfo.kt` - 발급 쿠폰 응답 DTO

### 테스트 파일
- `CouponServiceTest.kt` - 7개 테스트 (CRUD)
- `IssuedCouponServiceTest.kt` - 7개 테스트 (발급/중복/만료/사용)

---

## Phase 3: Coupon Infrastructure - **완료**

### 구현 파일
- `infrastructure/coupon/CouponJpaRepository.kt` - JPA 리포지토리
- `infrastructure/coupon/CouponRepositoryImpl.kt` - 리포지토리 구현체
- `infrastructure/coupon/IssuedCouponJpaRepository.kt` - JPA 리포지토리 (비관적 락 쿼리 포함)
- `infrastructure/coupon/IssuedCouponRepositoryImpl.kt` - 리포지토리 구현체

### DDL
> local/test 프로파일은 `ddl-auto: update`이므로 Entity 어노테이션 기반으로 테이블/컬럼이 자동 생성된다. 별도 DDL 실행 불필요.

---

## Phase 4: Coupon API (Controller + ApiSpec + DTO)

### 어드민 API

| Method | URI | 기능 |
|--------|-----|------|
| GET | `/api-admin/v1/coupons` | 쿠폰 템플릿 목록 |
| GET | `/api-admin/v1/coupons/{couponId}` | 쿠폰 템플릿 상세 |
| POST | `/api-admin/v1/coupons` | 쿠폰 템플릿 등록 |
| PUT | `/api-admin/v1/coupons/{couponId}` | 쿠폰 템플릿 수정 |
| DELETE | `/api-admin/v1/coupons/{couponId}` | 쿠폰 템플릿 삭제 |
| GET | `/api-admin/v1/coupons/{couponId}/issues` | 발급 내역 조회 |

### 대고객 API

| Method | URI | 기능 |
|--------|-----|------|
| POST | `/api/v1/coupons/{couponId}/issue` | 쿠폰 발급 (인증 필요) |
| GET | `/api/v1/users/me/coupons` | 내 쿠폰 목록 (인증 필요) |

### 구현 파일
- `interfaces/api/coupon/CouponAdminV1ApiSpec.kt`
- `interfaces/api/coupon/CouponAdminV1Controller.kt`
- `interfaces/api/coupon/CouponAdminV1Dto.kt`
- `interfaces/api/coupon/CouponV1ApiSpec.kt`
- `interfaces/api/coupon/CouponV1Controller.kt`
- `interfaces/api/coupon/CouponV1Dto.kt`

---

## Phase 5: Coupon 통합/E2E 테스트

### 통합 테스트
- `CouponServiceIntegrationTest.kt` - 쿠폰 CRUD → 실제 DB 검증
- `IssuedCouponServiceIntegrationTest.kt` - 발급/중복 방지/만료 거부 검증

### E2E 테스트
- `CouponAdminV1ApiE2ETest.kt` - 어드민 API 전체 E2E
- `CouponV1ApiE2ETest.kt` - 대고객 API E2E

---

## Phase 6: Order 도메인 변경 - 쿠폰 적용

### Order Entity 변경
- `couponId: Long?` 필드 추가 (nullable)
- `originalAmount: BigDecimal` 추가 (쿠폰 적용 전 금액)
- `discountAmount: BigDecimal` 추가 (할인 금액, 기본값 0)

### OrderFacade 변경 (핵심)
```
쿠폰 사용 주문 흐름:
1. 비관적 락으로 상품 조회
2. 전체 재고 확인 (부분 주문 불가)
3. 비관적 락으로 IssuedCoupon 조회
4. 쿠폰 유효성 검증 (소유자, 사용 가능, 만료)
5. 재고 차감
6. 할인 계산 + 최소 주문 금액 검증
7. 쿠폰 사용 처리
8. 주문 생성
```

### 수정 파일
- `domain/order/Order.kt`
- `application/order/OrderFacade.kt`
- `application/order/OrderService.kt`
- `application/order/OrderInfo.kt`
- `interfaces/api/order/OrderV1Dto.kt`
- `interfaces/api/order/OrderV1ApiSpec.kt`

---

## Phase 7: Order+Coupon 통합/E2E 테스트

- `OrderFacadeIntegrationTest.kt` 업데이트 - 쿠폰 적용 주문 검증
- `OrderV1ApiE2ETest.kt` 업데이트 - 쿠폰 E2E

---

## Phase 8: 동시성 테스트

### 테스트 파일
- `CouponConcurrencyTest.kt` - 동일 쿠폰 동시 사용 → 1건만 성공
- `StockConcurrencyTest.kt` - 동시 주문 재고 정합성
- `LikeConcurrencyTest.kt` - 동시 좋아요 정합성

### 검증 포인트
- 비관적 락으로 쿠폰 중복 사용 방지
- 재고 음수 방지
- 좋아요 유니크 인덱스 활용

---

## Phase 9: .http 파일 & 전체 빌드 검증

- `.http/coupon-admin-api.http` - 어드민 쿠폰 API
- `.http/coupon-api.http` - 대고객 쿠폰 API
- `.http/commerce-api.http` 업데이트 - 쿠폰 적용 주문
- `./gradlew ktlintCheck && ./gradlew build`
