# 4주차 대화 로그

## 2026-03-04: IssuedCoupon 엔티티 설계 — BaseEntity 상속 vs 독자 구현

### 고민한 부분
- 쿠폰 도메인을 구현하면서, `Coupon` 엔티티는 soft delete가 필요하므로 `BaseEntity`(id, createdAt, updatedAt, deletedAt) 상속이 자연스러웠음
- 그런데 `IssuedCoupon`은 성격이 다름: soft delete가 필요 없고, `status` 필드로 상태를 관리하며, `usedAt` 같은 고유 타임스탬프가 있음
- `BaseEntity`를 상속하면 불필요한 `deletedAt`이 생기고, `OrderBaseEntity`를 상속하면 `updatedAt`이 없음
- 기존 프로젝트에는 `BaseEntity`(CRUD용)와 `OrderBaseEntity`(생성 후 불변)만 있어서 IssuedCoupon에 딱 맞는 베이스가 없었음

### 선택지
1. **BaseEntity 상속** — deletedAt이 생기지만 일관성 유지, 코드 재사용
2. **OrderBaseEntity 상속** — updatedAt이 없어서 상태 변경 추적 불가
3. **독자 구현** — id, createdAt, updatedAt만 직접 선언, deletedAt 없이 깔끔하게 맞춤

### 선택한 답
- 3번: 독자 구현
- 이유: IssuedCoupon은 "발급 → 사용/만료"라는 고유 생명주기를 가짐. soft delete 대상이 아니고(사용된 쿠폰도 기록으로 남아야 함), status 필드가 상태를 표현. 불필요한 deletedAt 컬럼을 추가하는 것보다 필요한 컬럼만 가진 독자 구현이 더 명확함

### 느낀 점
- 베이스 엔티티 상속은 편리하지만, 도메인 성격과 맞지 않으면 오히려 혼란을 줌. "이 엔티티는 삭제되는가?"라는 질문 하나로 BaseEntity 상속 여부를 판단할 수 있음
- 기존 프로젝트에 2종류의 BaseEntity가 있다고 해서 모든 엔티티가 둘 중 하나에 속해야 하는 건 아님. 도메인 성격에 맞게 새로운 구조를 만드는 것도 좋은 선택

---

## 2026-03-04: DB 스키마 관리 — ddl-auto: none 환경에서 새 테이블 추가

### 고민한 부분
- 프로젝트가 `ddl-auto: none`으로 설정되어 있어서, JPA 엔티티를 만들어도 테이블이 자동 생성되지 않음
- Flyway나 Liquibase 같은 마이그레이션 도구도 없고, SQL 파일도 없음
- 테스트는 로컬 MySQL Docker 컨테이너(`localhost:3306/loopers`)에서 실행됨
- 새로운 `coupons`, `issued_coupons` 테이블과 `orders` 테이블 컬럼 추가를 어떻게 적용할지

### 내용 정리
- 프로젝트는 테스트 시 `test` 프로파일을 사용하지만, DB 스키마는 수동 관리 방식
- `MySqlTestContainersConfig`은 존재하지만 `ConditionalOnProperty`로 비활성화 상태 (`test.mysql.testcontainers.enabled=true` 설정 없음)
- 결국 로컬 Docker MySQL에 직접 DDL을 실행해야 함
- MySQL 8.0에서 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`가 지원되지 않아서 별도 명령으로 분리 실행함

### 느낀 점
- `ddl-auto: none`은 운영 안전성을 위한 설정이지만, 개발 환경에서는 새 테이블/컬럼 추가 시 수동 작업이 필요한 번거로움이 있음
- Flyway 같은 마이그레이션 도구가 없으면 "어떤 DDL이 언제 적용되었는지" 추적이 어려움. 프로젝트가 커지면 마이그레이션 도구 도입을 고려할 만함
- `build.gradle.kts`의 `systemProperty("spring.profiles.active", "test")` 설정을 발견하면서 테스트 실행 환경을 이해하게 됨. Gradle 설정도 꼼꼼히 봐야 전체 그림이 보임

---

## 2026-03-04: 동시성·멱등성·일관성 전략 분석 — 내 코드는 어떤 고민을 했나

### 고민한 부분
- 과제에서 "동시성, 멱등성, 일관성, 느린 조회, 동시 주문 등 실제 서비스 문제를 해결하라"는 요구사항이 있음
- 낙관적 락과 비관적 락 중 어떤 전략을 왜 선택했는지, 트랜잭션 경계를 어떻게 설계했는지 현재 코드를 점검하고 싶었음

### 선택지 — 락 전략
1. **낙관적 락(Optimistic Lock)** — `@Version` + 충돌 시 재시도 루프
2. **비관적 락(Pessimistic Lock)** — `SELECT ... FOR UPDATE`로 줄 세우기
3. **혼합** — 도메인별로 다르게 적용

### 선택한 답
- 비관적 락 단일 전략 (Product 재고, IssuedCoupon 사용에 `PESSIMISTIC_WRITE`)
- 이유: 재고와 쿠폰은 충돌 확률이 높고, 주문 플로우 중간에 재시도하면 복잡도가 급증함. "줄 세우기"로 단순하게 정합성 보장

### 현재 코드에서 확인한 전략들

**1) 트랜잭션 경계 — Facade 단일 트랜잭션**
- `OrderFacade.createOrder()`에 `@Transactional` 하나로 락 획득 → 재고 차감 → 주문 생성을 원자적으로 묶음
- `reserveStock()`에 `@Transactional`이 없는 이유: 이미 Facade의 트랜잭션 안이므로 별도 경계 불필요
- 장점: 원자성 보장, 구현 단순 / 단점: 락 보유 시간이 길어질 수 있음

**2) 멱등성 — 비즈니스 규칙 기반 중복 방지**
- 쿠폰 발급: `existsByCouponIdAndUserId()` 선검사 → 같은 유저에게 같은 쿠폰 중복 발급 차단 (409 CONFLICT)
- Soft Delete: `deletedAt ?: run { deletedAt = now() }` → 이미 삭제된 엔티티 재삭제 시 무시 (멱등)
- 좋아요: 삭제된 좋아요 → restore / 활성 → no-op
- 한계: HTTP 요청 레벨의 idempotency key는 없어서 네트워크 재시도 중복 주문은 방지 못함

**3) 일관성 — 스냅샷 패턴**
- 주문 생성 시 상품명, 브랜드명, 단가를 스냅샷으로 복사 → 주문 후 상품 가격 변경/브랜드 삭제에도 이력 유지
- `getBrandsIncludingDeleted()` 사용: 삭제된 브랜드의 주문도 브랜드명 보존

**4) 쿠폰 동시성 인프라 — 준비 완료, 조립 미완**
- `getIssuedCouponWithLock()`, `validateOwner()`, `validateUsable()`, `use()` 등 인프라는 구현됨
- `OrderFacade`에 쿠폰 사용 로직이 아직 조립되지 않음 (다음 단계)

**5) 읽기 최적화**
- `@EntityGraph(["_orderItems"])`: Order + OrderItem N+1 방지
- `default_batch_fetch_size: 100`: Lazy 로딩 시 IN절 배치
- Soft delete 필터링을 DB SQL에서 처리
- 아직 없는 것: Redis 캐시, Read Replica 라우팅

### 느낀 점
- 비관적 락을 선택한 것은 "재고/쿠폰은 충돌 빈도가 높다"는 판단에 기반함. 실제로 이커머스에서 인기 상품 주문 시 동시 요청이 몰리기 때문에 맞는 선택
- 트랜잭션 하나에 모든 걸 묶으면 단순하지만, 트랜잭션 시간이 길어지면 커넥션 풀 고갈 위험이 있음 → 추후 최적화 포인트
- 멱등성을 비즈니스 규칙으로만 보장하는 건 기본이고, 실서비스에서는 idempotency key 패턴도 고려해야 함
- 쿠폰을 주문에 조립할 때 락 획득 순서(Product 먼저? Coupon 먼저?)가 데드락 방지의 핵심이 될 것

---

## 2026-03-04: Atomic Update를 왜 안 썼나 — 멘토링 후 코드 재점검

### 고민한 부분
- 멘토님(Devin)이 동시성 해결의 우선순위를 알려줌: (1) Atomic Update → (2) 낙관적 락 → (3) 비관적 락
- 내 코드는 1순위(Atomic Update)를 검토하지 않고 바로 3순위(비관적 락)로 갔음
- Atomic Update가 정확히 뭔지, 왜 내 코드에서는 사용하지 않았는지(혹은 못했는지) 분석해봄

### Atomic Update의 정확한 의미
- "조회 없이 DB 한 문장으로 값을 변경"하는 것
- 예: `UPDATE products SET stock = stock - 3 WHERE id = 1 AND stock >= 3`
- DB 왕복 1회, 락 보유 시간 거의 0 → 가장 부하가 적은 동시성 해결 방식
- vs Read-Modify-Write(현재 방식): SELECT FOR UPDATE → 메모리에서 계산 → UPDATE (DB 왕복 2회, 락 보유 시간 = 트랜잭션 전체)

### 도메인별 분석 — Atomic Update가 적합했는가?

**1) 재고 차감 — Atomic Update 부적합 (현재 비관적 락이 합리적)**
- 부분 주문 로직: 상품 A 성공, 상품 B 재고 부족 → A만 주문. 이 "조회 후 조건 분기"는 단일 UPDATE로 불가
- 여러 상품 동시 처리: 5개 상품을 한 트랜잭션에서 각각 성공/실패 구분해야 함
- 스냅샷 수집: 락 잡은 상태에서 productName, price를 읽어 주문에 복사 → 어차피 SELECT 필요
- 결론: Atomic Update(`stock = stock - qty`)는 가능하지만, 실패 이유 구분 불가 + 스냅샷 위한 SELECT가 따로 필요해서 이점이 줄어듦

**2) 좋아요 — Atomic Update가 가장 적합한데 미적용**
- 현재: Product에 likeCount 필드 자체가 없음. Like 테이블의 행 존재 여부로만 관리
- 멘토님 추천: `UPDATE products SET like_count = like_count + 1 WHERE id = :productId`
- 좋아요는 단순 +1/-1이고 조건 분기가 필요 없으므로 Atomic Update의 교과서적 케이스
- 개선 포인트: Product에 likeCount 추가 → Atomic Update 적용

**3) 쿠폰 상태 변경 — Atomic Update 가능하지만 도메인 검증이 이유**
- 이론적으로: `UPDATE issued_coupons SET status = 'USED', used_at = NOW() WHERE id = :id AND status = 'AVAILABLE'`
- 하지만: `validateOwner()`, `validateUsable()` 등 도메인 검증 후 구체적 에러 메시지 반환이 필요
- `affected rows = 0`만으로는 "이미 사용됨" vs "본인 쿠폰 아님" 구분 불가
- 결론: 도메인 규칙이 복잡하면 Read-Modify-Write가 합리적

### 선택지
1. **Atomic Update 전면 도입** — 단순 연산(좋아요 등)에 적용
2. **현행 유지** — 비관적 락으로 통일
3. **혼합** — 도메인 특성에 따라 Atomic/비관적 분리

### 선택한 답
- 3번: 혼합 전략이 이상적
- 좋아요 카운트 → Atomic Update (단순 +1/-1, 조건 분기 불필요)
- 재고/쿠폰 → 비관적 락 유지 (복잡한 도메인 검증 + 부분 성공 처리 필요)

### 느낀 점
- 멘토님이 "우선순위"를 준 이유를 이해함: 가장 가벼운 해결책부터 검토하고, 그것으로 안 될 때 다음 단계로 가야 함. 처음부터 비관적 락을 쓴 건 "무거운 도구를 먼저 꺼낸" 셈
- Atomic Update는 "조건 분기가 필요 없는 단순 연산"에 최적. 반대로 "조회 후 판단 → 수정" 패턴에서는 한계가 있어서 락이 필요
- 좋아요 도메인에 likeCount를 추가하고 Atomic Update를 적용하면 멘토링 내용을 코드에 반영한 좋은 개선 사례가 될 것
- "왜 이 기술을 선택했는지"를 설명할 수 있어야 한다는 멘토님 말씀이 핵심. Atomic Update를 검토한 뒤 "이 케이스에서는 부적합해서 비관적 락을 선택했다"고 말할 수 있어야 함

---

## 2026-03-04: 재고 차감에서 Atomic Update, 낙관적 락을 안 쓴 이유 정리

### 고민한 부분
- 멘토님 우선순위대로라면 Atomic Update → 낙관적 락 → 비관적 락 순으로 검토해야 함
- 재고 차감에서 왜 1순위, 2순위를 건너뛰고 3순위(비관적 락)를 선택했는지 근거를 명확히 정리하고 싶었음

### Atomic Update를 안 쓴 이유

**핵심: 재고 차감이 "단순 연산"이 아니라 "조회 → 판단 → 분리 → 저장"이기 때문**

1) **부분 성공 처리**: 주문에 상품 3개를 담았는데 A 성공, B 재고 부족, C 성공이면 A+C만 주문 처리하고 B는 제외 목록으로 반환함. for 루프 안에서 상품별로 성공/실패를 분기하는데, `UPDATE stock = stock - qty WHERE stock >= qty`만으로는 이 분기를 표현할 수 없음
2) **실패 이유 구분 불가**: Atomic Update의 `affected rows = 0`만으로는 "재고 부족"인지 "상품이 삭제됨"인지 "존재하지 않는 상품"인지 알 수 없음. 현재 코드는 `product == null` → "존재하지 않는 상품", `!product.reserve(qty)` → "재고 부족, 현재 재고: N" 처럼 구체적 에러 메시지를 제공
3) **스냅샷 수집**: 재고 차감과 동시에 productName, price, brandId를 읽어서 주문에 스냅샷으로 복사해야 함. Atomic Update는 UPDATE만 하므로 이 데이터를 위해 결국 별도 SELECT가 필요 → DB 왕복이 줄어드는 이점이 사라짐

### 낙관적 락을 안 쓴 이유

**핵심: 이커머스 재고는 충돌 빈도가 높고, 주문 플로우 중간 재시도가 복잡함**

1) **충돌 빈도**: 인기 상품에 동시 주문이 몰리면 충돌이 거의 매번 발생. 낙관적 락은 충돌 시 `OptimisticLockException` → 재시도 루프가 필요한데, 대부분 실패하고 재시도하면 오히려 부하가 커짐
2) **재시도 복잡도**: 주문 플로우는 "재고 차감 → 브랜드 조회 → 주문 생성"이 하나의 트랜잭션. 중간에 버전 충돌이 나면 전체 플로우를 처음부터 재시도해야 함. 부분 성공 상태에서의 재시도 로직이 매우 복잡해짐
3) **공정성 문제**: 멘토님이 언급한 것처럼, 1만 건 동시 요청 시 낙관적 락은 단 하나만 성공하고 나머지는 계속 재시도 → 먼저 요청한 사용자가 오히려 늦게 처리될 수 있음. 비관적 락은 줄 세우기이므로 선착순 보장

### 그래서 비관적 락을 선택한 이유

- 여러 상품을 한번에 `SELECT FOR UPDATE`로 락 → 부분 성공 분기 + 스냅샷 수집 + 재고 차감을 하나의 트랜잭션에서 원자적으로 처리
- 중간에 실패 시 전체 자동 롤백 (보상 로직 불필요)
- 트레이드오프 인식: 락 보유 시간이 트랜잭션 전체이므로 동시 처리량이 떨어질 수 있음 → 트랜잭션을 최대한 짧게 유지하는 것이 중요

### 느낀 점
- 기술 선택은 "이게 좋다/나쁘다"가 아니라 "이 상황에서 왜 이걸 선택했는가"를 설명할 수 있어야 함
- Atomic Update가 1순위인 건 "부하가 가장 적어서"이지, "항상 써야 해서"가 아님. 비즈니스 요구사항(부분 성공, 에러 구분, 스냅샷)이 복잡하면 Read-Modify-Write + 비관적 락이 합리적
- 반대로 좋아요 카운트처럼 단순 +1/-1은 Atomic Update가 확실한 정답. 도메인 특성에 따라 달라야 함

---

## 2026-03-04: "미리 재고를 예약하는 설계"는 왜 R-M-W를 선택한 건가

### 고민한 부분
- 재고 차감에 비관적 락을 쓴 이유는 정리했지만, 더 근본적인 질문이 남음
- 애초에 왜 `reserveStock()`이라는 "미리 예약" 패턴을 만들어서 R-M-W가 발생하게 설계한 건지
- R-M-W 없이 Atomic Update만으로 처리할 수 있는 구조는 없었는지

### 핵심 원인: "부분 주문" 비즈니스 요구사항

현재 주문 로직은 상품 A, B, C를 담아서 주문했을 때 B만 재고 부족이면 A+C만 주문 생성하고 B는 "제외 목록"으로 알려주는 구조임

```
for (item in criteria) {
    if (product == null) → failedReservations ("존재하지 않는 상품")
    if (!product.reserve(qty)) → failedReservations ("재고 부족, 현재 재고: N")
    else → reservedProducts (성공)
}
→ 성공한 것만 모아서 주문 생성
→ 실패한 것은 제외 사유와 함께 반환
```

이 "상품별로 성공/실패를 판단하고 분기"하는 로직이 Read를 먼저 해야 하는 이유임

### Atomic Update로 했다면 어떻게 되나

```sql
UPDATE products SET stock = stock - 3 WHERE id = 1 AND stock >= 3;  -- affected=1
UPDATE products SET stock = stock - 5 WHERE id = 2 AND stock >= 5;  -- affected=0
UPDATE products SET stock = stock - 2 WHERE id = 3 AND stock >= 2;  -- affected=1
```

- 상품 2가 왜 실패? → `affected=0`만으로는 재고 부족/삭제됨/존재 안 함 구분 불가
- 성공한 상품 1, 3의 이름과 가격은? → UPDATE만 했으니 모름, 스냅샷을 위해 SELECT 별도 필요
- 상품 2의 현재 재고가 몇 개? → 모름, 에러 메시지를 위해 SELECT 별도 필요
- 결국 Atomic Update를 써도 SELECT가 따로 필요 → DB 왕복 줄어드는 이점 사라짐

### 만약 부분 주문이 없었다면?

"하나라도 재고 부족이면 전체 주문 거부"라는 요구사항이었다면:
- Atomic Update로 처리 가능 (`affected=0`이면 즉시 전체 롤백)
- 에러 메시지도 "재고 부족으로 주문이 불가합니다" 하나면 충분
- 스냅샷도 주문 생성 직전에 SELECT 한 번이면 됨
- 이 경우에는 Atomic Update가 적합했을 것

### 결론

```
부분 주문 요구사항
  → 상품별 성공/실패 판단 + 실패 이유 구분 필요
  → 현재 상태를 먼저 읽어야 판단 가능 (Read)
  → 읽은 데이터로 분기 처리 (Modify)
  → 성공한 것만 모아서 저장 (Write)
  → Read-Modify-Write 불가피
  → R-M-W의 동시성 보장을 위해 FOR UPDATE
```

### 느낀 점
- 기술 선택(R-M-W vs Atomic Update)이 먼저가 아니라, 비즈니스 요구사항(부분 주문)이 기술을 결정한 것
- "왜 R-M-W를 선택했나?"에 대한 답은 "비관적 락이 좋아서"가 아니라 "부분 주문이라는 요구사항이 Read를 강제했기 때문"
- 요구사항이 "전체 성공 or 전체 실패"였다면 Atomic Update가 가능했고 더 효율적이었을 것. 설계는 항상 비즈니스 맥락에서 판단해야 함

---

## 2026-03-04: 데드락 방지 설계 원칙 — 멘토링 내용을 내 코드에서 확인

### 고민한 부분
- 멘토님이 데드락 방지 원칙 3가지를 알려줌: (1) 락 순서 통일 (2) 애그리것 루트에서 락 (3) FK 지양
- 내 코드에서 이 원칙들이 어떻게 적용되어 있는지 확인해봄

### 1) 락 순서 통일 — IN절로 한번에 락 획득

**데드락 시나리오 (개별 락 방식)**
```
Thread A: SELECT ... WHERE id = 1 FOR UPDATE → 상품1 락
Thread B: SELECT ... WHERE id = 2 FOR UPDATE → 상품2 락
Thread A: SELECT ... WHERE id = 2 FOR UPDATE → B가 잡고 있어 대기
Thread B: SELECT ... WHERE id = 1 FOR UPDATE → A가 잡고 있어 대기
→ 서로 기다리며 영원히 멈춤 = 데드락
```

**우리 코드의 해결 — IN절로 한 쿼리에서 동시 락 획득**
```kotlin
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id IN :ids AND p.deletedAt IS NULL")
fun findAllByIdWithLock(ids: List<Long>): List<Product>
```
- 쿼리가 하나이므로 DB가 내부적으로 락 순서를 관리 → 데드락 불가
- 개별 `findByIdWithLock()`을 for 루프로 도는 것보다 안전

**앞으로 주의할 점**: 쿠폰 적용 주문을 구현하면 Product 락 + IssuedCoupon 락을 동시에 잡아야 함. 이때 "항상 Product 먼저 → IssuedCoupon 다음" 같은 순서 규칙을 정해야 데드락 방지 가능

### 2) 애그리것 루트에서 락 — 부모에서 잠그기

- 재고 차감 시 Product(루트)에 락을 잡음. stock이라는 값을 별도 테이블로 분리해서 그쪽에 락을 잡는 게 아님
- Order ↔ OrderItem도 Order(루트)를 통해서만 OrderItem에 접근. OrderItem을 직접 조회하거나 락을 잡을 필요 없음
- `cascade = ALL, orphanRemoval = true`로 루트가 자식의 생명주기를 완전히 관리

### 3) DB 레벨 FK 지양 — 애플리케이션 레벨에서만 관계 매핑

**중요 구분: `@ManyToOne` ≠ DB FK**

우리 프로젝트는 `ddl-auto: none`이라서 Hibernate가 테이블을 생성하지 않음. DDL SQL 파일도 없음. 따라서 DB에 `FOREIGN KEY` 제약조건이 걸려있지 않음

```kotlin
// OrderItem.kt — @ManyToOne은 JPA 레벨의 객체 매핑일 뿐
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
val order: Order = order
// → DB에는 order_id BIGINT 컬럼만 존재, FK 제약 없음

// Product.kt — 아예 Long 값만 저장
@Column(name = "brand_id", nullable = false)
val brandId: Long = brandId
// → 다른 애그리것이므로 @ManyToOne 자체를 안 씀

// IssuedCoupon.kt — 마찬가지로 Long 값만 저장
@Column(name = "coupon_id", nullable = false)
val couponId: Long = couponId
```

**FK가 DB에 있으면 왜 위험한가**
- 자식 INSERT 시 DB가 자동으로 부모 행에 공유 락을 걸어 참조 무결성 검증 → 의도하지 않은 락 전파
- 부모를 동시에 수정하려는 트랜잭션과 충돌 → 데드락 가능성 증가
- 우리 코드는 DB FK 없이 애플리케이션에서 관계를 관리하므로 이 문제 자체가 발생하지 않음

**Order ↔ OrderItem에 `@ManyToOne`을 쓴 이유**
- 같은 애그리것이라 항상 함께 생성/조회/삭제됨
- JPA cascade + `@EntityGraph`로 N+1 방지에 필요
- DB FK가 아닌 애플리케이션 레벨 매핑이므로 데드락 리스크 없음

### 느낀 점
- `@ManyToOne`과 DB FK는 다른 것. JPA 어노테이션은 객체 그래프 매핑이고, DB FK는 제약조건. `ddl-auto: none`이면 DB에 FK가 자동 생성되지 않음
- 같은 애그리것 내부(Order ↔ OrderItem)에서는 `@ManyToOne`으로 객체 탐색의 편의를 가져가고, 다른 애그리것 간(Product → Brand, IssuedCoupon → Coupon)에는 Long ID만 저장해서 결합도를 낮추는 전략
- 데드락 방지의 핵심은 "락 잡는 순서를 예측 가능하게 만드는 것". IN절, 순서 규칙, FK 제거 모두 이 원칙의 구체적 실천

---

## 2026-03-04: 데드락이 발생하면 어떻게 되나

### 고민한 부분
- 데드락 방지 전략은 정리했는데, 만약 실제로 데드락이 발생하면 DB와 애플리케이션에서 어떤 일이 벌어지는지 궁금했음

### DB의 데드락 처리 흐름

```
Thread A: 상품1 락 → 상품2 락 시도 (대기...)
Thread B: 상품2 락 → 상품1 락 시도 (대기...)

→ MySQL InnoDB가 ~1초 내에 데드락 감지
→ 비용이 적은 쪽(변경 행이 적은 트랜잭션)을 "희생자"로 선택
→ 희생자 트랜잭션 강제 롤백 + 에러 반환
→ 희생되지 않은 쪽은 대기가 풀리고 정상 진행 → 커밋 성공
```

- 희생자가 롤백되면서 잡고 있던 락이 해제됨 → 나머지 쪽이 락 획득 → 정상 완료
- 즉 데드락이 발생해도 DB가 영원히 멈추지는 않음. 한쪽이 강제 실패할 뿐

### 애플리케이션에서 어떻게 보이나

희생된 쪽에서 예외 발생:
```
CannotAcquireLockException
  → MySQLTransactionRollbackException
    → "Deadlock found when trying to get lock; try restarting transaction"
```

우리 코드에는 이 예외를 잡는 재시도 로직이 없으므로 → 500 Internal Server Error가 사용자에게 반환됨

### 그래서 방지가 중요한 이유
- 데드락이 발생하면 한쪽 사용자는 주문 실패 + 불친절한 에러 메시지
- 재시도 로직도 없으므로 사용자가 직접 다시 시도해야 함
- 발생 후 대응보다 애초에 발생하지 않도록 설계하는 게 훨씬 중요

### 느낀 점
- 데드락은 DB가 자동 감지해서 한쪽을 죽이므로 시스템이 멈추지는 않지만, 사용자 경험이 나빠짐
- "방지"가 핵심이고, IN절 한번에 락 획득이나 락 순서 통일 같은 설계 원칙이 중요한 이유를 이해함

---

## 2026-03-04: 쿠폰 만료 처리 — 스케줄러 vs 사용 시점 시간 비교

### 고민한 부분
- 쿠폰 만료를 어떻게 처리할지: 스케줄러로 주기적으로 상태를 EXPIRED로 변경할지, 사용 시점에 expiredAt과 현재 시간을 비교할지

### 선택지
1. **스케줄러 방식** — 주기적으로 만료된 쿠폰의 status를 EXPIRED로 UPDATE
2. **사용 시점 시간 비교 방식** — `expiredAt.isBefore(ZonedDateTime.now())`로 필요한 시점에 판단

### 선택한 답
- 2번 선택 (사용 시점 시간 비교)
- 이유: 스케줄러는 주기 사이 불일치 문제(만료됐는데 아직 AVAILABLE), 스케줄러 장애 시 만료 처리 누락, 불필요한 DB UPDATE 부하가 발생함. 시간 비교는 항상 정확하고 추가 인프라 없이 동작함

### 현재 구현
- `Coupon.isExpired()` — `expiredAt.isBefore(ZonedDateTime.now())`로 만료 판단
- `CouponService.issueCoupon()` — 발급 시점에 `coupon.isExpired()` 호출하여 만료된 쿠폰 거부
- `IssuedCouponStatus.EXPIRED` — enum 값은 존재하지만 실제로 이 상태로 변경하는 코드는 없음
- 상태 전이는 `AVAILABLE → USED`만 존재

### 느낀 점
- 상태 필드를 직접 관리하는 것 자체가 버그와 불일치의 원인이 될 수 있음
- "상태를 저장하지 않고 계산으로 판단한다"는 원칙이 불필요한 복잡성을 줄여줌
- 멘토가 말한 핵심이 이것 — 스케줄러 같은 외부 프로세스에 의존하면 장애 포인트가 늘어남

---

## 2026-03-04: 비관적 락과 DB Connection Pool 고갈 위험

### 고민한 부분
- 비관적 락(`SELECT ... FOR UPDATE`)으로 주문을 처리할 때, 락을 기다리는 동안 DB Connection을 계속 점유하게 됨
- 스프링의 HikariCP Connection Pool이 40개인데, 동시 주문이 몰리면 모든 커넥션이 락 대기로 묶여서 풀이 고갈될 수 있지 않은지

### 내용 정리

**왜 고갈이 발생하는가**
- `@Transactional` 진입 시 커넥션 획득 → `SELECT FOR UPDATE` → 다른 트랜잭션이 같은 행을 잡고 있으면 대기
- 대기 중에도 커넥션은 반환되지 않음 → 동시 요청 40개가 모두 대기하면 41번째부터는 커넥션 자체를 못 얻어서 `ConnectionTimeoutException`
- 인기 상품에 동시 주문이 몰리면 현실적으로 발생 가능한 시나리오

**현재 설정에서 괜찮은 이유**
- HikariCP maximum-pool-size: 40 (현재 프로젝트 설정)
- MySQL `innodb_lock_wait_timeout`: 기본 50초 (락 대기 최대 시간)
- 현재 트래픽 수준에서는 40개 커넥션으로 충분
- 비관적 락은 순차 처리이므로 한 트랜잭션이 끝나면 다음이 바로 진행 → 실제 대기 시간은 수십 ms 수준

**대규모 트래픽에서의 대안들**
1. **Atomic Update** — `UPDATE stock = stock - qty WHERE stock >= qty`, 락 없이 DB 한 문장으로 처리. 가장 가벼움
2. **Redis 분산 락** — DB 커넥션을 잡지 않고 Redis에서 락 관리. DB 풀 고갈 위험 제거
3. **큐 기반 처리** — 주문 요청을 큐에 넣고 순차 처리. 커넥션 점유 시간 최소화
4. **별도 커넥션 풀** — 주문 처리 전용 풀을 분리하여 다른 API에 영향 방지

### 느낀 점
- 비관적 락은 정합성은 확실하지만, 커넥션을 오래 잡는다는 트레이드오프가 있음
- 현재 규모에서는 문제없지만, "이 설계가 어디까지 버티는지"를 인지하고 있어야 함
- 대안을 알고 있다는 것 자체가 면접이나 멘토링에서 중요한 포인트. "비관적 락을 선택했지만, 트래픽이 커지면 Redis 분산 락이나 큐 기반으로 전환할 수 있다"고 말할 수 있어야 함

---

## 2026-03-05: 주문 생성 — 부분 주문 허용 vs 전체 실패

### 고민한 부분
- 기존 주문 로직은 "부분 주문"을 허용했음. 상품 A, B를 주문했는데 B 재고가 부족하면 A만 주문하고 B는 `excludedItems`로 반환
- 쿠폰 적용 시에는 부분 주문 불가, 쿠폰 미적용 시에는 부분 주문 허용 — 두 갈래 분기가 존재
- 사용자가 A, B를 함께 주문한 건 "둘 다 원해서"인데, 동의 없이 일부만 결제하는 게 맞는지 의문

### 선택지
1. **부분 주문 유지** — 매출 관점에서 가능한 것만이라도 주문시키는 게 유리. 품절 상품은 응답에 포함시켜 안내
2. **전체 성공 or 전체 실패** — 하나라도 재고 부족이면 전체 주문 거부. 어떤 상품이 부족한지 에러 메시지로 안내

### 선택한 답
- 2번 선택 (전체 성공 or 전체 실패)
- 이유: 사용자 의도 존중. 일반적인 이커머스에서도 동의 없이 일부만 결제하지 않음. 쿠폰 유무와 무관하게 동일한 플로우가 되어 코드가 단순해지는 부수 효과도 있음

### 느낀 점
- 처음에 부분 주문을 "매출 극대화"라는 관점에서 설계했지만, 실제 사용자 경험을 생각하면 전체 실패가 더 자연스러움
- 기술적으로도 `excludedItems`, `OrderResultInfo`, `FailedReservation` 같은 중간 구조체가 전부 사라지고, 쿠폰/비쿠폰 분기가 단일 플로우로 통합됨. 비즈니스 정책이 단순해지면 코드도 단순해진다는 걸 체감
- "부분 주문"이라는 요구사항이 R-M-W 패턴을 강제했었는데, 그 요구사항 자체가 바뀌니 코드 복잡도가 크게 줄어듦. 기술 선택보다 비즈니스 정책이 먼저라는 걸 다시 느낌

---

## 2026-03-05: 최소 주문 금액 미달 시 — 주문 거부 vs 쿠폰만 제외

### 고민한 부분
- 쿠폰 적용 주문에서 최소 주문 금액에 미달하면 어떻게 할지
- "쿠폰을 못 쓰는 것"이지 "주문 자체가 불가능한 것"은 아닌데, 현재는 전체 롤백됨

### 선택지
1. **전체 실패 (주문 거부)** — 사용자가 쿠폰 적용을 "의도"한 것이므로, 할인 못 받는 주문을 강행하지 않음
2. **쿠폰만 거부, 주문은 정가로 진행** — 쿠폰 못 쓰는 건 쿠폰 문제일 뿐, 주문까지 막을 이유 없음. 다만 사용자가 할인 기대하고 결제했는데 정가로 나가면 CS 이슈 가능

### 선택한 답
- 1번 선택 (전체 실패)
- 이유: 사용자가 쿠폰을 명시적으로 요청한 상황. 기대한 할인 없이 정가 결제가 진행되면 오히려 불만. 실패시키고 "최소 주문 금액이 부족합니다"라고 안내하는 게 사용자 입장에서 더 나음

### 느낀 점
- "기술적으로 가능한 것"과 "사용자가 원하는 것"은 다름. 쿠폰 빼고 주문 진행은 기술적으로 쉽지만, UX 관점에서는 예상치 못한 정가 결제가 더 큰 문제
- 에러를 던지는 게 항상 나쁜 게 아니라, 사용자에게 재시도 기회를 주는 셈. 금액 더 채워서 쿠폰 쓸지, 쿠폰 빼고 주문할지 선택권을 돌려주는 것

---

## 2026-03-05: 동시성 테스트를 왜 작성했나 — 비관적 락이 실제로 동작하는지 증명

### 고민한 부분
- 비관적 락(`PESSIMISTIC_WRITE`)을 재고, 쿠폰에 적용했고, 좋아요에는 유니크 제약을 걸었음
- 하지만 "락을 걸었다"는 것과 "동시 요청에서 정합성이 보장된다"는 건 다른 이야기
- 코드 리뷰나 면접에서 "이 락이 실제로 동작하는 걸 어떻게 아느냐"고 물으면 테스트로 증명해야 함
- 단위 테스트나 통합 테스트는 단일 스레드라서 동시성 문제를 재현할 수 없음

### 내용 정리

**재고 동시성 (StockConcurrencyTest)**
- 재고 5개 상품에 10개 스레드가 동시에 주문 → 비관적 락 덕분에 정확히 5개만 성공, 재고 0
- 만약 락이 없었다면 10개 스레드가 모두 "재고 5개 남았네" 읽고 동시에 차감 → 재고가 음수가 됨
- `OrderFacade.createOrder()` → `getProductsWithLock()` → `PESSIMISTIC_WRITE`로 직렬화되는 것을 검증

**쿠폰 동시성 (CouponConcurrencyTest)**
- 동일 발급 쿠폰으로 10개 스레드가 동시에 주문 → 비관적 락 덕분에 1개만 성공, 쿠폰 USED
- 만약 락이 없었다면 여러 스레드가 모두 "AVAILABLE이네" 읽고 동시에 사용 → 쿠폰 1장으로 여러 주문 할인
- `getIssuedCouponWithLock()` → `PESSIMISTIC_WRITE` → `validateUsable()` 순서가 핵심

**좋아요 동시성 (LikeConcurrencyTest)**
- 동일 (userId, productId)에 10개 스레드가 동시에 좋아요 → 유니크 제약 덕분에 레코드 1개만 존재
- 좋아요는 애플리케이션 레벨 락이 없음. DB의 `uk_likes_user_product` 유니크 제약이 중복을 방지
- 일부 스레드에서 `DataIntegrityViolationException`이 발생하지만, 데이터 정합성은 보장됨

**세 테스트의 동시성 보장 메커니즘 차이**
| 도메인 | 보장 방식 | 이유 |
|--------|----------|------|
| 재고 | 비관적 락 | 조회 → 판단 → 차감 R-M-W 패턴이라 락 필요 |
| 쿠폰 | 비관적 락 | 조회 → 검증 → 상태변경 R-M-W 패턴이라 락 필요 |
| 좋아요 | DB 유니크 제약 | 단순 INSERT, 중복만 막으면 되므로 제약조건으로 충분 |

### 느낀 점
- "락을 걸었다"는 설계 의도일 뿐이고, 실제로 멀티 스레드에서 정합성이 깨지지 않는다는 걸 테스트로 보여줘야 설득력이 있음
- `CountDownLatch`로 스레드를 동시에 출발시키는 패턴이 동시성 테스트의 기본 도구. 없으면 스레드가 순차적으로 실행돼서 동시성 문제를 재현할 수 없음
- 좋아요처럼 단순한 경우에는 락 대신 DB 제약조건만으로 충분함. "모든 곳에 비관적 락"이 아니라 도메인 특성에 맞는 전략을 선택하는 게 중요하다는 걸 테스트를 통해 확인함
- 이전에 멘토님이 알려준 우선순위(Atomic Update → 낙관적 락 → 비관적 락)와 연결됨. 좋아요는 가장 가벼운 방식(유니크 제약)으로 해결한 셈이고, 재고/쿠폰은 비즈니스 복잡도 때문에 비관적 락이 합리적이라는 결론을 테스트로 뒷받침함

---

## 2026-03-05: 4주차 보고 준비 — 비관적 락 선택의 근본 원인 재분석

### 고민한 부분
- 멘토링 청강에서 atomic update → 낙관적락 → 비관적락 순으로 우선순위를 두라고 하셨는데, 설계 단계에서 1순위(Atomic Update)를 깊이 검토하지 않고 바로 3순위(비관적 락)로 갔음
- 부분 주문 처리 로직을 삭제한 지금, 낙관적 락이 더 좋은 선택이 아닌지 재검토

### 선택지
1. **낙관적 락으로 전환** — 부분 주문 삭제로 도메인이 단순해졌으니 낙관적 락이 적합할 수도
2. **비관적 락 유지** — 재고는 충돌 빈도가 높고, 상품이 여러 개일수록 낙관적 락의 충돌 확률이 곱으로 증가

### 선택한 답
- 2번: 비관적 락 유지
- 이유: 부분 주문 삭제는 Atomic Update 가능성을 열어준 것이지, 낙관적 락을 유리하게 만든 건 아님. 낙관적 락은 충돌 시 전체 롤백 + 재시도가 필요한데, 주문에 상품이 여러 개 담기면 충돌 확률이 곱으로 올라가고 재시도 폭풍 가능. 비관적 락의 커넥션 풀 고갈 문제는 인지하되, 현재 규모에서는 락 타임아웃 설정이나 향후 Redis 분산 락 전환으로 대응 가능

### 깨달은 핵심 — 도메인 모델 설계가 전략을 결정했다
- `product.reserve()`, `issuedCoupon.validateUsable()` 같은 Rich Domain Model 메서드를 먼저 설계한 시점에서, 이미 "엔티티를 메모리에 올려서 메서드를 호출한다"는 Read-Modify-Write 패턴이 확정됨
- Atomic Update는 엔티티를 거치지 않는 쿼리이므로, 풍부한 도메인 모델과 정면 충돌
- 동시성 전략은 독립적인 선택이 아니라 도메인 설계의 결과물이었는데, 설계할 때는 동시성을 아직 고민하기 전이니까 자연스럽게 놓침
- 우선순위를 제대로 적용하려면, 도메인 모델 설계 단계에서부터 "이 로직이 엔티티 안에 있어야 하는가, 쿼리 한 방으로 해결 가능한가?"를 함께 고민했어야 함

### 다른 수강생들의 접근 비교
- **안유진**: 단순한 도메인에서 락 없음 → 낙관적 → 비관적 → Atomic Update 순으로 사다리를 직접 밟아 올라감. 도메인이 단순하면 1순위 전략이 그대로 적용 가능하다는 사례
- **서태수**: Atomic Update 적용 후 JPA 1차 캐시와 DB 불일치 문제를 발견. `clearAutomatically=true`로 방어할지 낙관적 락으로 전환할지 고민 중. Atomic Update도 트레이드오프가 있다는 사례

### 느낀 점
- 내가 비관적 락으로 직행한 건 "무거운 도구를 먼저 꺼낸" 것이 아니라, Rich Domain Model이라는 설계 결정이 이미 전략을 좁혀놓은 것이었음
- 안유진 님처럼 도메인이 단순하면 Atomic Update가 바로 먹히고, 서태수 님처럼 Atomic Update를 쓰면 JPA 캐시 불일치라는 새로운 문제가 생김. 어떤 전략이든 트레이드오프가 있음
- 결국 "왜 이 전략을 선택했는가"를 도메인 특성과 연결해서 설명할 수 있느냐가 핵심. 비관적 락의 커넥션 풀 고갈 문제까지 인지하고, 향후 대안까지 제시할 수 있으면 충분히 좋은 판단

---

## 2026-03-05: 상품 좋아요 수(likeCount) Atomic Update 구현 — 설계 고려사항

### 고민한 부분
- 멘토링에서 동시성 해결 우선순위가 Atomic Update → 낙관적 락 → 비관적 락 순이라는 피드백을 받았고, 좋아요 카운트가 Atomic Update의 교과서적 케이스라는 결론이 나왔음
- 실제로 구현하면서 6가지 기술적 고려사항이 있었음

### 내용 정리

**1) 왜 엔티티 메서드가 아닌 SQL로 처리하는가**
- 일반적인 JPA 패턴이라면 `product.incrementLikeCount()`를 만들고 dirty checking으로 UPDATE하는 게 자연스러움
- 하지만 이 방식은 read-modify-write 패턴: Thread A가 0을 읽고, Thread B도 0을 읽고, 둘 다 1로 UPDATE → Lost Update 발생
- `UPDATE products SET like_count = like_count + 1`은 DB가 row-level lock을 잡고 현재 값 기준으로 연산하므로, 애플리케이션이 현재 값을 읽을 필요 없이 원자적으로 처리됨
- 그래서 Product 엔티티에는 `private set`으로 읽기 전용만 두고, 쓰기는 `@Modifying @Query`로만 수행

**2) `clearAutomatically = true`가 필요한 이유**
- `@Modifying` 쿼리는 JPA 1차 캐시를 우회해서 DB를 직접 UPDATE함
- Atomic Update로 DB의 like_count가 1이 되었지만, 영속성 컨텍스트에는 여전히 0으로 남아있어 stale read 발생 가능
- `clearAutomatically = true`로 UPDATE 실행 후 영속성 컨텍스트를 자동 클리어
- 추가 고려: 클리어 시 LikeService에서 이미 저장한 Like 엔티티도 detach되지만, `LikeInfo.from(saved)`는 이미 인메모리 참조의 값을 복사하는 것이고 Like에 lazy 연관이 없으므로 안전

**3) `GREATEST(like_count - 1, 0)` — 음수 방지**
- 정상 흐름에서는 likeCount가 음수가 될 일이 없지만, 데이터 마이그레이션 오류나 수동 DB 조작으로 불일치가 생길 수 있음
- DB 레벨에서 방어하여 데이터가 의미 없는 상태(음수)로 빠지지 않도록 함

**4) 멱등성 보장 — 어떤 경우에 count를 바꾸고, 어떤 경우에 안 바꾸는가**
- addLike에는 3가지 분기: 신규(+1), 이미 활성(변화 없음), 복원(+1)
- 중복 요청 시에도 increment를 호출하면 likeCount가 실제 좋아요 수와 불일치
- 실제로 좋아요 상태가 변경되는 경우에만 increment/decrement를 호출

**5) 낙관적 락/비관적 락 대신 Atomic Update를 선택한 이유**
- 좋아요는 단순 +1/-1 연산이므로 현재 값을 읽을 필요가 없음
- Atomic Update가 가장 단순하고 성능이 좋음
- 낙관적 락은 충돌 시 재시도 코드가 필요하고, 비관적 락은 불필요하게 동시 처리량을 제한

**6) `nativeQuery = true`를 사용한 이유**
- JPQL로는 `SET like_count = like_count + 1` 같은 DB 레벨 산술 연산을 직접 표현하기 어려움
- `nativeQuery = true`로 MySQL SQL을 직접 작성해서 DB 엔진이 row-level lock + 산술 연산을 원자적으로 처리

### 느낀 점
- 이전 대화 로그에서 "좋아요 카운트는 Atomic Update의 교과서적 케이스"라고 분석했던 것을 실제로 구현에 옮겼음. 분석만 하는 것과 실제 코드를 작성하는 것은 다름 — clearAutomatically 같은 JPA 캐시 문제는 구현해봐야 깨달음
- Rich Domain Model과 Atomic Update의 충돌이라는 이전 고민이 여기서 구체화됨. likeCount는 엔티티 메서드가 아닌 SQL로 처리하되, 읽기용 필드는 엔티티에 둠으로써 두 패러다임을 공존시킴
- 멘토님이 말한 우선순위(Atomic Update → 낙관적 락 → 비관적 락)를 실제로 적용한 첫 사례. 재고/쿠폰은 도메인 복잡도 때문에 비관적 락이 합리적이었고, 좋아요는 단순해서 Atomic Update가 딱 맞았음. "도메인 특성에 따라 전략을 달리한다"는 원칙을 코드로 증명함

---

## 2026-03-05: likes 테이블과 like_count 비정규화 — 싱크 불일치 대비 전략

### 고민한 부분
- likes 테이블(누가 어떤 상품에 좋아요했는지)과 products.like_count(조회용 비정규화 컬럼)를 분리해서 설계했음
- like_count는 Atomic Update로 실시간 동기화하고 있지만, 애플리케이션 장애로 likes만 저장되고 like_count 업데이트가 누락되는 경우 싱크가 어긋날 수 있음
- 이런 불일치를 배치로 보정하는 게 일반적인 방법인지, 아니면 이 설계 자체가 적절하지 않은 건지 궁금

### 멘토님께 질문 (예정)
> 좋아요 기능을 설계할 때, likes 테이블(누가 어떤 상품에 좋아요했는지)과 products 테이블의 like_count(조회용 비정규화 컬럼)를 분리했습니다.
> like_count는 좋아요 등록/취소 시 Atomic Update(`like_count = like_count + 1`)로 실시간 동기화하고 있는데, 이 둘의 싱크가 어긋나는 경우(예: 애플리케이션 장애로 likes만 저장되고 like_count 업데이트가 누락)를 대비해서 배치로 보정하는 게 일반적인 방법인지 궁금합니다.

### 느낀 점
- 비정규화는 조회 성능을 위한 트레이드오프인데, 도입하면 반드시 "원본과 복사본의 싱크 문제"가 따라온다는 걸 체감
- 현재는 같은 트랜잭션 안에서 Like 저장과 like_count 업데이트가 함께 일어나므로 정상 흐름에서는 불일치가 없음. 하지만 "정상 흐름에서 안전하다"와 "장애 상황에서도 안전하다"는 다른 이야기
- 멘토님 답변을 듣고 배치 보정, 이벤트 소싱 등 실무에서 쓰는 전략을 파악해볼 예정
