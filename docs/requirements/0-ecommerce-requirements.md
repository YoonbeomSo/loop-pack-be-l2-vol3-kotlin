주의: 이 md 파일은 초기 프로젝트 요구사항을 정리해둔 파일이기 때문에 절대 수정하지 않는다.  

# 이커머스 도메인 요구사항 정의서

## 1. 서비스 개요

### 1.1 배경

**좋아요** 누르고, **쿠폰** 쓰고, 주문 및 **결제**하는 **감성 이커머스**.

내가 좋아하는 브랜드의 상품들을 한 번에 담아 주문하고, 유저 행동은 랭킹과 추천으로 연결돼요.

우린 이 흐름을 하나씩 직접 만들어갈 거예요.

### 1.2 서비스 흐름 예시

1. 사용자가 **회원가입**을 하고
2. 여러 브랜드의 상품을 둘러보고, 마음에 드는 상품엔 **좋아요**를 누르죠.
3. 사용자는 **쿠폰을 발급**받고, 여러 상품을 **한 번에 주문하고 결제**합니다.
4. 유저의 행동은 모두 기록되고, 그 데이터는 이후 다양한 기능으로 확장될 수 있어요.

---

## 2. API 구조

### 2.1 대고객 API

- **Base Path**: `/api/v1`
- **인증 헤더** (유저 로그인이 필요한 기능):
  - `X-Loopers-LoginId`: 로그인 ID
  - `X-Loopers-LoginPw`: 비밀번호

> 인증/인가는 주요 스코프가 아니므로 구현하지 않습니다.
> 유저는 타 유저의 정보에 직접 접근할 수 없습니다.

### 2.2 어드민 API

- **Base Path**: `/api-admin/v1`
- **인증 헤더**:
  - `X-Loopers-Ldap`: `"loopers.admin"` <- 임시 텍스트 고정

> LDAP (Lightweight Directory Access Protocol): 중앙 집중형 사용자 인증, 정보 검색, 액세스 제어 → 회사 사내 어드민

---

## 3. 도메인별 API 명세

### 3.1 유저 (Users)

#### 대고객 API

| Method | Endpoint | user_required | 설명 |
|--------|----------|:-------------:|------|
| POST | `/api/v1/users` | X | 회원가입 |
| GET | `/api/v1/users/me` | O | 내 정보 조회 |
| PUT | `/api/v1/users/password` | O | 비밀번호 변경 |

---

### 3.2 브랜드 & 상품 (Brands / Products)

#### 대고객 API

| Method | Endpoint | user_required | 설명 |
|--------|----------|:-------------:|------|
| GET | `/api/v1/brands/{brandId}` | X | 브랜드 정보 조회 |
| GET | `/api/v1/products` | X | 상품 목록 조회 |
| GET | `/api/v1/products/{productId}` | X | 상품 정보 조회 |

#### 상품 목록 조회 쿼리 파라미터

| 파라미터 | 예시 | 설명 |
|----------|------|------|
| `brandId` | `1` | 특정 브랜드의 상품만 필터링 |
| `sort` | `latest` / `price_asc` / `likes_desc` | 정렬 기준 |
| `page` | `0` | 페이지 번호 (기본값 0) |
| `size` | `20` | 페이지당 상품 수 (기본값 20) |

> 정렬 기준은 선택 구현입니다. 필수는 `latest`, 그 외는 `price_asc`, `likes_desc` 정도로 제한해도 충분합니다.

#### 어드민 API

| Method | Endpoint | ldap_required | 설명 |
|--------|----------|:-------------:|------|
| GET | `/api-admin/v1/brands?page=0&size=20` | O | 등록된 브랜드 목록 조회 |
| GET | `/api-admin/v1/brands/{brandId}` | O | 브랜드 상세 조회 |
| POST | `/api-admin/v1/brands` | O | 브랜드 등록 |
| PUT | `/api-admin/v1/brands/{brandId}` | O | 브랜드 정보 수정 |
| DELETE | `/api-admin/v1/brands/{brandId}` | O | 브랜드 삭제 |
| GET | `/api-admin/v1/products?page=0&size=20&brandId={brandId}` | O | 등록된 상품 목록 조회 |
| GET | `/api-admin/v1/products/{productId}` | O | 상품 상세 조회 |
| POST | `/api-admin/v1/products` | O | 상품 등록 |
| PUT | `/api-admin/v1/products/{productId}` | O | 상품 정보 수정 |
| DELETE | `/api-admin/v1/products/{productId}` | O | 상품 삭제 |

#### 비즈니스 규칙

- **브랜드 삭제 시**: 해당 브랜드의 상품들도 삭제되어야 함
- **상품 등록 시**: 상품의 브랜드는 이미 등록된 브랜드여야 함
- **상품 수정 시**: 상품의 브랜드는 수정할 수 없음

> 상품, 브랜드 정보 중 고객과 어드민에게 제공되어야 할 정보에 대해 고민해보세요.

---

### 3.3 좋아요 (Likes)

#### 대고객 API

| Method | Endpoint | user_required | 설명 |
|--------|----------|:-------------:|------|
| POST | `/api/v1/products/{productId}/likes` | O | 상품 좋아요 등록 |
| DELETE | `/api/v1/products/{productId}/likes` | O | 상품 좋아요 취소 |
| GET | `/api/v1/users/{userId}/likes` | O | 내가 좋아요 한 상품 목록 조회 |

---

### 3.4 주문 (Orders)

#### 대고객 API

| Method | Endpoint | user_required | 설명 |
|--------|----------|:-------------:|------|
| POST | `/api/v1/orders` | O | 주문 요청 |
| GET | `/api/v1/orders?startAt=2026-01-31&endAt=2026-02-10` | O | 유저의 주문 목록 조회 |
| GET | `/api/v1/orders/{orderId}` | O | 단일 주문 상세 조회 |

#### 주문 요청 예시

```json
{
  "items": [
    { "productId": 1, "quantity": 2 },
    { "productId": 3, "quantity": 1 }
  ]
}
```

#### 비즈니스 규칙

- **주문 정보**에는 당시의 상품 정보가 스냅샷으로 저장되어야 합니다.
- **주문 시에 다음 동작이 보장되어야 합니다**: 상품 재고 확인 및 차감

> **결제**는 과정 진행 중, **추가로 개발**하게 됩니다!

#### 어드민 API

| Method | Endpoint | ldap_required | 설명 |
|--------|----------|:-------------:|------|
| GET | `/api-admin/v1/orders?page=0&size=20` | O | 주문 목록 조회 |
| GET | `/api-admin/v1/orders/{orderId}` | O | 단일 주문 상세 조회 |

---

## 4. 나아가며

> ⚙️ **모든 기능의 동작을 개발한 후에 동시성, 멱등성, 일관성, 느린 조회, 동시 주문 등 실제 서비스에서 발생하는 문제들을 해결하게 됩니다.**

---

## 5. API 요약

### 5.1 대고객 API 요약

| 도메인 | Method | Endpoint | 설명 |
|--------|--------|----------|------|
| 유저 | POST | `/api/v1/users` | 회원가입 |
| 유저 | GET | `/api/v1/users/me` | 내 정보 조회 |
| 유저 | PUT | `/api/v1/users/password` | 비밀번호 변경 |
| 브랜드 | GET | `/api/v1/brands/{brandId}` | 브랜드 정보 조회 |
| 상품 | GET | `/api/v1/products` | 상품 목록 조회 |
| 상품 | GET | `/api/v1/products/{productId}` | 상품 정보 조회 |
| 좋아요 | POST | `/api/v1/products/{productId}/likes` | 좋아요 등록 |
| 좋아요 | DELETE | `/api/v1/products/{productId}/likes` | 좋아요 취소 |
| 좋아요 | GET | `/api/v1/users/{userId}/likes` | 내 좋아요 목록 |
| 주문 | POST | `/api/v1/orders` | 주문 요청 |
| 주문 | GET | `/api/v1/orders` | 주문 목록 조회 |
| 주문 | GET | `/api/v1/orders/{orderId}` | 주문 상세 조회 |

**총 12개 API**

### 5.2 어드민 API 요약

| 도메인 | Method | Endpoint | 설명 |
|--------|--------|----------|------|
| 브랜드 | GET | `/api-admin/v1/brands` | 브랜드 목록 조회 |
| 브랜드 | GET | `/api-admin/v1/brands/{brandId}` | 브랜드 상세 조회 |
| 브랜드 | POST | `/api-admin/v1/brands` | 브랜드 등록 |
| 브랜드 | PUT | `/api-admin/v1/brands/{brandId}` | 브랜드 수정 |
| 브랜드 | DELETE | `/api-admin/v1/brands/{brandId}` | 브랜드 삭제 |
| 상품 | GET | `/api-admin/v1/products` | 상품 목록 조회 |
| 상품 | GET | `/api-admin/v1/products/{productId}` | 상품 상세 조회 |
| 상품 | POST | `/api-admin/v1/products` | 상품 등록 |
| 상품 | PUT | `/api-admin/v1/products/{productId}` | 상품 수정 |
| 상품 | DELETE | `/api-admin/v1/products/{productId}` | 상품 삭제 |
| 주문 | GET | `/api-admin/v1/orders` | 주문 목록 조회 |
| 주문 | GET | `/api-admin/v1/orders/{orderId}` | 주문 상세 조회 |

**총 12개 API**

---

## 6. 변경 이력

| 버전 | 날짜 | 변경 내용 |
|------|------|-----------|
| 2.0 | 2026-02-12 | 원본 요구사항 기준 재작성 (API 규격 준수) |
