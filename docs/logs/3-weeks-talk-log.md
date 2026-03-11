# 3주차 대화 로그

## 2026-02-24: Domain Service에 @Transactional을 붙여야 하는가?

### 고민한 부분
- Brand 도메인 개발 중, BrandService에 `@Transactional`이 직접 붙어있었음
- 설계 문서에서는 "트랜잭션 경계는 Facade의 책임"이라고 정의했는데, 실제 구현에서는 Facade 없이 Service에 트랜잭션을 붙여놓은 상태
- `// todo domain service는 트랜잭션을 관리하지 않는다?` 라는 주석이 남아있었음
- DIP 관점에서 이게 맞는 건지 의문

### 선택지
1. **Service에 @Transactional 유지** — Facade가 없으니 현실적으로 Service에서 관리 (임시 타협)
2. **BrandFacade 도입 후 트랜잭션 이전** — 도메인별로 커밋하려면 지금 정리하는 게 맞음

### 선택한 답
- 2번 선택: BrandFacade를 지금 도입
- 이유: 도메인별로 커밋할 건데, 나중에 Facade 추가하면 Brand 커밋이 불완전해짐

### 느낀 점
- `@Transactional`은 Spring 프레임워크의 인프라 관심사인데, 이걸 Domain Service에 붙이면 DIP 위반이 됨
- DIP의 핵심은 "의존 방향의 역전":
  - 일반적: 도메인 → 인프라 (상위가 하위를 의존)
  - DIP 적용: 인프라 → 도메인 인터페이스 (하위가 상위를 구현)
- 이 프로젝트에서 DIP가 적용된 곳:
  - `BrandService` → `BrandRepository` (인터페이스, 도메인 레이어)
  - `BrandRepositoryImpl` → `BrandRepository` 구현 (인프라 레이어가 도메인을 의존)
- `@Transactional` 제거도 같은 맥락: 도메인 레이어에서 프레임워크 의존을 걷어내면 순수 비즈니스 로직만 남음
- 최종 의존 방향: `Controller → Facade (@Transactional) → Service (순수 로직) → Repository (인터페이스)`

---

## 2026-02-24: 의존 방향의 역전이란?

### 고민한 부분
- DIP(Dependency Inversion Principle)에서 "역전"이 정확히 뭘 의미하는지 이해하고 싶었음
- 이 프로젝트의 `BrandRepository` 구조가 왜 DIP인지

### 내용 정리

#### DIP가 없는 일반적인 구조
```
BrandService → BrandJpaRepository (Spring Data JPA)
```
- Domain Service가 JPA라는 인프라 기술을 직접 의존
- JPA를 Redis로 바꾸면? Service 코드를 전부 수정해야 함
- 상위 모듈(도메인)이 하위 모듈(인프라)에 의존하는 것 → 의존 방향이 위에서 아래로

#### DIP가 적용된 이 프로젝트의 구조
```
domain/BrandService → domain/BrandRepository (인터페이스)
                              ↑ 구현
               infrastructure/BrandRepositoryImpl → BrandJpaRepository
```
- BrandService는 같은 도메인 레이어에 있는 BrandRepository **인터페이스**에만 의존
- BrandRepositoryImpl(인프라)이 BrandRepository(도메인 인터페이스)를 **구현**
- 인프라 레이어가 도메인 레이어의 인터페이스에 의존 → 의존 방향이 아래에서 위로 **역전**됨

#### "역전"이 일어난 포인트
- 일반적 의존 방향: `도메인 → 인프라` (상위가 하위를 의존)
- DIP 적용 후: `인프라 → 도메인` (하위가 상위의 인터페이스를 구현)
- 인터페이스의 위치가 핵심: **인터페이스가 도메인 레이어에 있기 때문에** 의존 방향이 역전됨
- 만약 인터페이스가 인프라 레이어에 있었다면 DIP가 아님

#### 이게 왜 좋은가?
- JPA → Redis로 바꿀 때: `BrandRepositoryImpl`만 새로 만들면 됨, Service는 수정 없음
- 도메인 레이어가 프레임워크에 의존하지 않으니 단위 테스트가 쉬움 (Mockito로 인터페이스만 Mock)
- 비즈니스 로직이 기술 구현과 분리되어 변경에 강함

### 느낀 점
- DIP의 핵심은 "인터페이스를 어느 레이어에 두느냐"임
- 인터페이스가 도메인에 있으면 DIP, 인프라에 있으면 그냥 추상화
- `@Transactional`을 Facade로 올린 것도 같은 원리: 도메인이 프레임워크를 몰라야 함

---

## 2026-02-24: domain-modeling skill에 빠진 내용이 없는지 점검

### 고민한 부분
- `docs/requirements/domainmodeling_developement.md` 학습 자료와 현재 `domain-modeling/SKILL.md`를 비교해봄
- skill 파일이 원문의 의도를 제대로 담고 있는지, 빠진 부분은 없는지 확인

### 발견한 3가지 이슈

#### 1. Manager/doer 뉘앙스가 원문과 달랐음

- **기존 skill**: "행위자(doer)나 Manager 같은 이름이 붙으면 Domain Service일 가능성이 높다"
- **원문 뉘앙스**: "이런 객체는 고유한 상태나 도메인적 의미가 없고, 기능 하나만 수행. 도메인 개념이 아니다. 다만 로직 분리 용도로 유용하다"
- **문제**: skill대로라면 Manager 클래스를 도메인 객체로 오인할 수 있음

#### 2. VO의 Context 의존성이 빠져있었음

- **원문 퀴즈**: "Address는 항상 VO일까? → Context에 따라 다르다. 우체국에서는 추적 대상이므로 Entity가 될 수 있다"
- **문제**: VO 도입 여부를 판단할 때 "이 Context에서 식별/추적이 필요한가?"라는 핵심 질문이 skill에 없었음

#### 3. Anti-pattern 예시가 없었음

- **원문**: 좋아요를 Product 안에 `likedUserIds`로 넣는 잘못된 구조 → 응집도 낮고 확장 어려움
- **문제**: "비즈니스 의미가 커질 수 있는 개념은 별도 도메인으로 분리"라는 원칙은 있었지만, 구체적으로 어떤 게 잘못된 건지 보여주는 예시가 없었음

### 선택지
1. **CLAUDE.md에 반영** — 매번 로딩되는 문서에 넣기
2. **domain-modeling/SKILL.md 보완** — 도메인 모델링할 때만 참조되는 skill에 넣기

### 선택한 답
- 2번: `domain-modeling/SKILL.md` 보완
- 이유: CLAUDE.md에는 이미 충분히 정리되어 있고, 도메인 모델링 시에만 필요한 판단 기준이므로 skill에 넣는 게 맞음

### 느낀 점
- 학습 자료의 "톤"이 중요함. 같은 개념이라도 skill에 옮길 때 뉘앙스가 바뀌면 AI가 다르게 해석할 수 있음
- Manager/doer 같은 경우, "Domain Service일 가능성이 높다" vs "도메인 개념이 아니다"는 완전히 다른 방향의 판단을 유도함
- VO는 절대적인 게 아니라 Context에 따라 달라진다는 걸 기억해야 함. 우리 프로젝트에서도 Product.price를 Money VO로 만들지 말지는 "이 프로젝트에서 금액에 대한 도메인 규칙이 충분한가?"로 판단해야 함

---

## 2026-02-25: 레이어 구조에서 Service, Facade, @Transactional 관계 총정리

### 고민한 부분

Brand 구현을 마치고, 레이어 구조에서 각 구성요소의 역할과 관계에 대해 연쇄적으로 질문이 이어짐. 하나하나 짚어보면서 전체 그림을 이해하고 싶었음.

### Q&A 정리

#### Q1. Domain Service, Application Service, Facade 중 @Transactional은 어디에?

**답: Application 레이어 (우리 프로젝트에서는 Facade)**

```
Controller → Facade (@Transactional) → Domain Service (순수 로직) → Repository
```

- `@Transactional`은 Spring 프레임워크의 인프라 관심사
- Domain Service에 붙이면 도메인 레이어가 Spring에 의존 → DIP 위반
- Application 레이어(Facade)가 트랜잭션 경계를 관리하는 것이 적절

#### Q2. Application Service가 Application Facade야?

**답: 우리 프로젝트에서는 같은 것**

| 역할 | Application Service | 우리 Facade |
|------|-------------------|-------------|
| 트랜잭션 관리 | O | O |
| 여러 도메인 서비스 조합 | O | O |
| 유스케이스 완성 | O | O |
| 상태 없음 (stateless) | O | O |

- 이름만 다를 뿐 책임이 동일
- 일부 아키텍처에서는 Facade = 여러 Application Service를 묶는 상위 레이어로 구분하기도 하지만, 우리 프로젝트 규모에서는 동일하게 취급

#### Q3. 간단한 프로젝트에서 Application Service를 Facade와 분리하는 건 오버엔지니어링?

**답: 맞다**

- 도메인당 유스케이스 3~7개, 교차 도메인도 1개 서비스 정도
- 분리하면 `Controller → Facade → Application Service → Domain Service → Repository` 5단계가 됨
- 위임만 하는 패스스루 코드가 늘어남
- **분리가 필요한 시점**: Facade가 10개 이상 메서드를 갖거나, 주문+결제+알림 같은 복잡한 조합이 생길 때

#### Q4. Facade와 Application Service가 분리된 경우 호출 구조는?

**답:**

```
# Application Service가 없을 때 (우리 프로젝트)
Controller → Facade → Domain Service (여러 개 조합 가능)

# Application Service가 있을 때 (대규모 프로젝트)
Controller → Facade → Application Service → Domain Service
```

- Facade만 있으면: Facade가 직접 Domain Service들을 호출해서 유스케이스를 완성
- Application Service가 있으면: Facade는 Application Service를 호출하고, Application Service가 Domain Service를 조합
- 핵심: **Facade는 항상 가장 바깥의 오케스트레이터**

#### Q5. Domain Service에 @Transactional을 걸면 단위 테스트에 문제가 생겨?

**답: 직접적인 문제는 없지만, 구조적 문제가 생김**

- 단위 테스트 자체는 Mockito로 가능 (Repository를 Mock하니까)
- 하지만 Spring Context 없이 테스트하려면 `@Transactional`이 동작 안 함 → 통합 테스트와 동작이 달라질 수 있음
- 근본적 문제는 테스트보다 **설계**: 도메인 레이어가 프레임워크를 알게 됨

#### Q6. Domain Service에 @Transactional 다는 게 DIP 위반이지?

**답: 맞다**

```
@Transactional  ← Spring 프레임워크 (인프라 기술)
class BrandService  ← 도메인 레이어
```

- 도메인 레이어가 Spring이라는 인프라 기술에 의존
- DIP는 "상위 모듈(도메인)이 하위 모듈(인프라)에 의존하면 안 된다"
- `@Transactional`을 Facade(Application 레이어)로 올리면 도메인은 순수해짐

#### Q7. 의존 방향이 Infrastructure → Application → Domain 인가, Infrastructure → Domain → Application 인가?

**답: 둘 다 맞지 않다. 정확한 의존 방향:**

```
Interfaces → Application → Domain ← Infrastructure
```

- Interfaces(Controller)는 Application(Facade)을 의존
- Application(Facade)은 Domain(Service, Repository 인터페이스)을 의존
- Infrastructure(RepositoryImpl)는 Domain(Repository 인터페이스)을 **구현** (역전!)
- **Domain이 중심**이고, 다른 레이어가 Domain을 향해 의존

#### Q8. Application에서는 Infrastructure에 있는 JpaRepository를 사용 안 하는데?

**답: 그게 DIP가 제대로 적용된 것**

```
BrandFacade → BrandService → BrandRepository (인터페이스, 도메인 레이어)
                                      ↑ 구현
                              BrandRepositoryImpl (인프라 레이어)
```

- Application 레이어(Facade)는 Infrastructure를 직접 알 필요가 없음
- Domain 레이어의 인터페이스만 알면 됨
- Spring이 런타임에 인터페이스와 구현체를 연결해줌 (DI)
- "Application이 Infrastructure를 모른다" = DIP가 잘 적용되었다는 증거

### 선택지
- (의사결정이 아닌 개념 이해 과정이므로 선택지 없음)

### 느낀 점

- 레이어 아키텍처에서 **의존 방향의 핵심은 "Domain이 중심"**이라는 것
  ```
  Interfaces → Application → Domain ← Infrastructure
  ```
- `@Transactional`이 단순한 어노테이션 하나지만, 이걸 어디에 붙이느냐가 아키텍처의 순수성을 결정함
- "Application이 Infrastructure를 모른다"는 게 처음엔 이상했지만, 인터페이스 기반으로 보면 당연한 구조
- Facade = Application Service라는 걸 이해하니까 레이어 구조가 명확해짐
- 프로젝트 규모에 맞는 구조를 선택하는 것이 중요. 오버엔지니어링도 언더엔지니어링도 아닌 **적정 수준**을 찾는 게 설계의 핵심

---

## 2026-02-25: 이번 주차 핵심 개념 정리

### 고민한 부분
- Product 도메인까지 구현을 마치고, 이번 주차에서 반복적으로 나온 두 가지 핵심 개념을 명확하게 정리하고 싶었음

### 1. @Transactional은 왜 Application Service(Facade)에 있어야 하는가

`@Transactional`은 Spring 프레임워크의 기능이다. Domain Service에 붙이면 **도메인 레이어가 Spring을 알게 된다.**

```
// ❌ 도메인이 인프라 기술에 의존
@Transactional  ← Spring (인프라)
class BrandService  ← 도메인 레이어

// ✅ Application 레이어가 인프라 기술을 담당
@Transactional  ← Spring (인프라)
class BrandFacade  ← Application 레이어
```

- Domain Service는 **순수한 비즈니스 로직**만 담당해야 한다
- 트랜잭션 관리는 **비즈니스 로직이 아니라 기술적 관심사**이다
- Application Service(Facade)는 여러 Service를 조합하고 유스케이스를 완성하는 역할이니, 트랜잭션 경계도 여기서 잡는 게 자연스럽다

### 2. 의존 방향 `Interfaces → Application → Domain ← Infrastructure`의 이점과 단점

핵심은 **화살표 방향**이다. Domain을 향해 모두가 의존하고, Domain은 아무도 의존하지 않는다.

**이점:**
- **Domain이 순수하다.** 프레임워크, DB 기술을 몰라서 단위 테스트가 쉽고, 비즈니스 로직이 기술 변경에 영향받지 않는다
- **Infrastructure 교체가 쉽다.** JPA → Redis로 바꿀 때 `RepositoryImpl`만 새로 만들면 되고, Service는 수정 없다
- **변경 영향이 격리된다.** 각 레이어가 자기 책임만 갖고 있어서 한 곳을 바꿔도 다른 레이어로 전파되지 않는다

**단점:**
- **코드량이 늘어난다.** Repository 인터페이스(도메인) + 구현체(인프라) + JpaRepository를 각각 만들어야 한다
- **간단한 CRUD도 레이어를 다 거친다.** Controller → Facade → Service → Repository, 위임만 하는 패스스루 코드가 생긴다
- **프로젝트 규모가 작으면 오버엔지니어링이다.** 교체할 일 없는 인프라를 위해 추상화 비용을 지불하는 셈이다

### 느낀 점
- 이번 주차 고민은 전부 "경계를 어디에 둘 것인가"에 대한 것이었음
- `@Transactional` 위치 하나에서 시작해서 DIP, 의존 방향, Facade 역할까지 연쇄적으로 이해하게 됨
- 이점과 단점을 둘 다 알아야 "왜 이 구조를 선택했는지" 설명할 수 있음. 무조건 좋은 구조는 없고, 트레이드오프를 이해한 위에서 선택하는 것이 중요함

---

## 2026-02-26: Facade에 @Transactional이 항상 필요한가?

### 고민한 부분
- Like 도메인의 `LikeFacade.addLike()`에 `@Transactional`이 붙어있었음
- 이 메서드가 하는 일:
  1. `productService.getProductIncludingDeleted(productId)` — 상품 존재 여부 검증 (읽기 전용)
  2. `likeService.addLike(userId, productId)` — 좋아요 등록 (자체 `@Transactional` 보유)
- "이 두 작업이 하나의 트랜잭션으로 묶여야 하는가?"라는 의문이 생김

### 선택지
1. **@Transactional 유지** — Facade는 항상 트랜잭션 경계를 담당한다는 일관성 유지
2. **@Transactional 제거** — 원자성이 필요 없는 경우 불필요한 트랜잭션을 걸지 않음

### 선택한 답
- 2번: `@Transactional` 제거
- 이유:
  - 상품 조회는 **읽기 전용** 검증이라 롤백할 게 없음
  - LK-05 규칙상 soft delete된 상품도 좋아요 가능이므로, 조회와 등록 사이에 상품이 삭제되어도 문제없음
  - 좋아요 등록이 실패해도 상품 조회를 "되돌릴" 필요가 없음
  - `likeService.addLike()`가 이미 자체 `@Transactional`로 원자성을 보장함
  - **두 작업 사이에 원자성이 필요한 시나리오가 없다**

### 느낀 점
- 이전에 "트랜잭션 경계는 Facade의 책임"이라고 정리했는데, 이걸 "Facade에는 항상 @Transactional"로 잘못 일반화할 뻔했음
- **@Transactional을 붙이는 기준은 "Facade냐 아니냐"가 아니라 "원자성이 필요한가"**임:
  - `BrandFacade.deleteBrand()` — 브랜드 삭제 + 상품 cascade 삭제가 함께 성공/실패해야 함 → 필요
  - `ProductFacade.createProduct()` — 브랜드 검증 + 상품 저장이 함께 성공/실패해야 함 → 필요
  - `LikeFacade.addLike()` — 상품 읽기 검증 + 좋아요 등록, 실패해도 서로 영향 없음 → 불필요
- Facade의 역할은 **서비스 조합/조정**이지, 반드시 트랜잭션을 거는 것이 아님. 트랜잭션은 원자성이 필요할 때만 걸어야 함
- "Facade = @Transactional"이 아니라 "Facade = 조합, @Transactional = 원자성 필요 시"로 분리해서 생각해야 함

---

## 2026-02-26: Facade는 도메인마다 만들어야 하는가? (LikeFacade vs ProductFacade)

### 고민한 부분
- Like 도메인 구현 시 `LikeFacade`를 만들었는데, "상품에 좋아요하다"라는 유스케이스가 `ProductFacade`에 있어야 하는 건 아닌지 의문이 생김
- `LikeFacade.addLike()`는 `ProductService` + `LikeService` 조합인데, 이걸 어느 Facade가 소유해야 하는가?

### 선택지
1. **LikeFacade 유지** — 쓰기 대상이 Like이니 Like 쪽이 소유. 의존 방향이 단방향(Like → Product)으로 유지됨
2. **ProductFacade로 이동** — "상품에 좋아요하다"는 유스케이스 관점에서 상품이 중심. API도 `/api/v1/products/{productId}/likes`

### 선택한 답
- 2번: `ProductFacade`로 이동, `LikeFacade` 삭제
- 이유:
  - 유스케이스 관점에서 "상품에 좋아요를 누르는 것"이므로 상품이 중심
  - 이미 application service가 존재하고, 추후 domain service도 만들 계획이라 Facade 클래스가 도메인마다 생기면 클래스가 과도하게 늘어남
  - 도메인마다 Facade를 만들면 LikeFacade, ReviewFacade, InquiryFacade 등이 각각 생기는데, 이들은 모두 "상품에 X하다"라는 유스케이스 → ProductFacade에서 조합하는 것이 자연스러움

### 반대 의견도 있었음
- LikeFacade를 유지하면 Like → Product 단방향 의존이 유지되고, ProductFacade가 비대해지지 않음
- ProductFacade에 넣으면 Product가 Like를 알게 되어 결합이 생김
- 하지만 **Facade 자체가 유스케이스 조합자**이므로, 교차 도메인 의존은 Facade의 본래 역할

### 느낀 점
- Facade를 "도메인 소유" 관점으로 보면 도메인마다 Facade가 생기고, "유스케이스 소유" 관점으로 보면 유스케이스 중심 객체에 모임
- 이 프로젝트에서는 유스케이스 중심이 더 적합: application service + (미래의) domain service + facade로 3종류 클래스가 생기는데, facade까지 도메인마다 만들면 클래스 폭발이 일어남
- **Facade의 소유 기준은 "누가 쓰는가(쓰기 대상)"가 아니라 "어떤 유스케이스인가"**로 판단하는 게 이 프로젝트에서는 맞음
- 결국 정답은 없고, 프로젝트 규모와 향후 계획에 따라 달라짐. 우리 프로젝트에서는 유스케이스 중심 + 클래스 수 억제가 더 중요했음

---

## 2026-02-26: BrandService가 ProductRepository를 주입받는 건 DDD 위반인가?

### 고민한 부분
- `BrandService`(Application Service)가 `ProductRepository`를 직접 주입받아 사용하고 있었음
- "BrandFacade를 만들어서 BrandService + ProductService를 조합해야 하지 않을까?"라는 의문이 생김
- Application Service가 다른 도메인의 Repository를 직접 사용하는 게 DDD 관점에서 괜찮은 건지 검토

### 선택지
1. **BrandFacade 도입** — BrandService + ProductService를 Facade에서 조합. 각 Service는 자기 도메인 Repository만 사용
2. **현재 구조 유지** — BrandService가 ProductRepository를 직접 주입받아 데이터를 조회하고, 비즈니스 규칙은 BrandDomainService가 캡슐화

### 선택한 답
- 2번: 현재 구조 유지 (코드 변경 없음)
- 이유:
  - "브랜드 삭제 시 상품도 cascade 삭제"는 **비즈니스 규칙**이며, `BrandDomainService`가 이를 명시적으로 캡슐화하고 있음
  - `ProductRepository` 주입은 비즈니스 로직이 아닌 **데이터 조회를 위한 인프라 조율**
  - Facade 방식은 오히려 비즈니스 규칙을 Facade 호출 순서로 분산시켜 DDD에 더 안 맞음

### Facade 방식이 더 나쁜 이유

```kotlin
// 현재 구조 — 비즈니스 규칙이 BrandDomainService에 캡슐화됨
BrandService (Application Service)          BrandDomainService (Domain Service)
├── brandRepository.findById()              ├── brand.delete()
├── productRepository.findAllByBrandId()    └── products.forEach { it.delete() }
├── brandDomainService.deleteBrand(...)        ↑ 비즈니스 규칙 캡슐화
├── brandRepository.save()
└── productRepository.save()
    ↑ 인프라 조율 (fetch, save, transaction)
```

```kotlin
// Facade 방식 — 비즈니스 규칙이 호출 순서로 흩어짐
class BrandFacade(brandService, productService) {
    fun deleteBrand(brandId) {
        brandService.deleteBrand(brandId)              // Brand만 삭제
        productService.deleteProductsByBrandId(brandId) // Product만 삭제
    }
}
// → "cascade 삭제"라는 도메인 지식이 Facade 메서드 안의 호출 순서로만 표현됨
// → BrandDomainService라는 명시적 도메인 규칙 표현이 사라짐
```

### 핵심 판단 기준
- **Application Service의 역할**: 여러 Repository에서 데이터를 가져와 Domain Service에 넘기고, 결과를 저장하는 **인프라 조율자**
- **Repository 주입 ≠ 도메인 침범**: Repository는 인터페이스이고, 데이터를 가져오는 것은 비즈니스 로직이 아님
- **Facade가 필요한 시점**: 두 개 이상의 **Application Service**를 조합해야 할 때 (지금은 BrandService 하나로 충분)

### 느낀 점
- "다른 도메인의 Repository를 주입받으면 DDD 위반"이라고 단순하게 생각했는데, 실제로는 **비즈니스 규칙이 어디에 캡슐화되어 있는가**가 핵심이었음
- Application Service가 여러 Repository를 사용하는 것은 자연스러운 일. 중요한 건 비즈니스 규칙이 Application Service에 누출되지 않고 Domain Service에 있는 것
- Facade를 도입하면 무조건 깨끗해지는 게 아니라, 오히려 도메인 지식이 분산될 수 있음. 구조적 분리가 항상 좋은 것은 아님
- **"이 구조가 맞는가?"를 판단하는 기준은 "비즈니스 규칙이 명시적으로 표현되어 있는가"**임

---

## 2026-02-27: Criteria와 Command의 차이와 역할

### 고민한 부분
- 재고 정책을 Product Aggregate로 이동하면서 `OrderItemCriteria`, `OrderItemCommand` 등 여러 DTO가 생겼음
- 이름이 비슷한데 역할이 다른 것 같아서, 각각의 책임과 쓰임새를 명확히 하고 싶었음

### 정리한 내용

| 구분 | Criteria | Command |
|------|----------|---------|
| 위치 | Application 레이어 | Domain 레이어 |
| 역할 | 외부 요청을 Application으로 전달하는 **입력 DTO** | 도메인 로직 실행에 필요한 **풍부한 도메인 정보** |
| 포함 정보 | 사용자 입력 최소값 (productId, quantity) | 도메인 판단에 필요한 모든 값 (productName, brandName, unitPrice 등) |
| 생성 시점 | Controller → Service 호출 시 | Service가 조회/가공 후 Domain으로 전달 시 |

```
Controller → [Criteria] → Facade/Service → 조회/가공 → [Command] → Domain Service
```

- Criteria: "뭘 주문할래?" (productId=1, quantity=2)
- Command: "이걸로 주문 만들어" (productId=1, productName="에어맥스90", brandName="나이키", quantity=2, unitPrice=129000)

### 느낀 점
- Criteria는 API 레이어의 DTO와 도메인을 분리하기 위한 **격리 장벽**, Command는 도메인 로직이 외부 서비스 없이 동작할 수 있게 **필요한 정보를 다 담는 봉투**
- 이름 자체에 의미가 있음: Criteria는 "기준/조건", Command는 "명령". 기준만 주면 Application이 나머지를 채워서 명령으로 만드는 구조

---

## 2026-02-27: OrderResult는 필요한가? (도메인 DTO 정리)

### 고민한 부분
- 재고 정책 이동 후 `OrderResult`(domain)와 `OrderResultInfo`(application)가 거의 동일한 구조로 남아있었음
- `OrderResult`는 `Order` + `List<ExcludedItem>`을 감싸는 wrapper인데, 이제 Facade가 직접 조립하는 흐름으로 바뀌면서 중간 전달 용도가 사라짐

### 선택지
1. **OrderResult 유지** — 도메인 레이어에 주문 결과 표현 객체를 남겨둠
2. **OrderResult 제거** — Facade에서 `OrderResultInfo.of(order, excludedItems)`로 직접 조립

### 선택한 답
- 2번: `OrderResult` 제거, `ExcludedItem`(domain)도 함께 제거
- 이유:
  - `OrderResult`는 이제 Facade에서 한 번 감싸고 바로 `OrderResultInfo`로 변환하는 **패스스루 wrapper**에 불과
  - `ExcludedItem`도 도메인 레이어에 있을 이유가 없음. 재고 예약 실패 정보는 `FailedReservation`(application/product)에서 이미 표현하고, Facade가 이를 `ExcludedItemInfo`(application/order)로 변환
  - 불필요한 도메인 DTO를 제거하면 변환 체인이 짧아짐

### 느낀 점
- 리팩토링 과정에서 역할이 사라진 DTO는 과감하게 제거해야 함. 남겨두면 "이게 왜 있지?"라는 혼란만 생김
- 도메인 레이어에 있다고 해서 무조건 유지해야 하는 건 아님. **현재 흐름에서 의미 있는 역할을 하는지**가 기준

---

## 2026-02-27: 재고 부족 시 에러 메시지가 사용자에게 오해를 줌

### 고민한 부분
- 재고보다 많은 수량을 주문했더니 `"주문 항목이 비어있습니다"` 라는 에러가 나왔음
- 실제 원인은 "재고 부족으로 예약 실패"인데, 에러 메시지는 "주문 항목이 비어있다"로 표시 → 사용자가 원인을 알 수 없음

### 원인 분석
- `ProductService.reserveStock()` → 재고 부족으로 전체 예약 실패 → `reservedProducts` 빈 리스트
- `OrderDomainService.buildOrder()` → 빈 리스트 받으면 `"주문 항목이 비어있습니다"` 예외
- **문제**: `OrderDomainService`는 "왜 비었는지" 모름. 빈 리스트만 받으니까 자기가 아는 메시지를 던짐

### 해결
- `OrderFacade`에서 `reservedProducts.isEmpty()` 체크를 **먼저** 수행
- 실패 사유를 포함한 메시지 반환: `"주문 가능한 상품이 없습니다. (재고가 부족합니다. 현재 재고: 0)"`
- `OrderDomainService.buildOrder()`의 빈 리스트 검증은 방어 코드로 유지 (직접 호출될 수도 있으니)

### 느낀 점
- 에러 메시지는 **사용자 관점에서 원인을 알 수 있어야** 함. 내부 구조상의 메시지("항목이 비었다")가 아니라 비즈니스 맥락의 메시지("재고가 부족하다")를 줘야 함
- Facade가 흐름을 조율하는 역할이니, **실패 원인을 아는 시점에서 빨리 예외를 던지는 것**이 맞음. 하위 레이어까지 내려가면 맥락이 사라짐
- 도메인 서비스의 방어 코드와 Facade의 비즈니스 검증은 역할이 다름. 둘 다 있어도 괜찮음

---

## 2026-02-27: 주문 목록 조회 날짜 포맷 — ISO vs 커스텀

### 고민한 부분
- 주문 목록 조회 API의 날짜 파라미터가 ISO 8601 형식(`2026-01-01T00:00:00+09:00`)이었음
- 실제 서비스에서 이 포맷이 일반적인지, 더 간결한 포맷이 낫지 않은지 고민

### 선택지
1. **ISO 8601 유지** — 국제 표준, 타임존 정보 포함
2. **`yyyyMMdd HH:mm:ss` 변경** — 한국 서비스에서 일반적으로 쓰이는 포맷

### 선택한 답
- 2번: `yyyyMMdd HH:mm:ss` 포맷으로 변경
- 이유:
  - 한국 대상 서비스에서 ISO 8601은 사용자에게 불필요하게 복잡함
  - 타임존은 서버에서 `Asia/Seoul`로 고정 처리하면 됨
  - 실제 커머스 서비스(쿠팡, 배민 등)에서도 커스텀 포맷을 사용하는 경우가 많음

### 구현 시 겪은 이슈
- Swagger UI에서 `LocalDateTime` 파라미터를 보내면 "Value must be a DateTime" 검증 에러가 남
- 원인: Swagger가 `LocalDateTime`을 ISO DateTime으로 강제 검증
- 해결: `@Parameter`에 `schema = Schema(type = "string")`을 추가하여 Swagger의 타입 검증을 우회

### 느낀 점
- 날짜 포맷은 "표준 vs 실용"의 트레이드오프. 글로벌 서비스라면 ISO 8601이 맞지만, 한국 단일 서비스라면 실용적인 포맷이 나음
- Swagger와 Spring의 타입 시스템이 충돌하는 경우가 있음. 커스텀 포맷을 쓸 때는 Swagger 설정도 함께 맞춰야 함
