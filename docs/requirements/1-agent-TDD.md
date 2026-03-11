# 🧭 루프팩 BE L2 - Round 1

> 단순히 기능을 구현하는 게 아니라, **의도를 설계한다.**
>
>
> 그리고 그 의도는 이제, **AI와 함께 구현된다.**
>

<aside>
🎯

**Summary**

</aside>

- 기능 구현보다 먼저 테스트 코드를 작성해본다.
- 테스트 가능한 구조란 무엇인지 체감해본다.
- 유저 등록/조회/비밀번호 변경 등을 테스트 주도로 구현해본다.

<aside>
📌

**Keywords**

</aside>

- 단위 테스트 vs 통합 테스트
- 테스트 더블(Mock, Stub, Fake 등)
- 테스트 가능한 코드 구조
- 테스트 주도 개발 (TDD)

<aside>
🧠

**Learning**

</aside>

# 🌍 **개발 환경의 변화**

## 🤖 최근 소프트웨어 개발 풍조: LLM · Agent-based 개발

<aside>
💡 최근 개발 현장에서는 더 이상 **“코드를 직접 얼마나 잘 치느냐”**가 생산성의 병목이 아닙니다.
LLM 은 이미 기계적으로 해석 가능한, 반복적인 행동들을 사람보다 빠르게 만들어내고 있습니다.
Agent 기반 도구들은 `요구사항` 을 작업 단위로 나누고, 실패하면 원인을 추론해 다시 시도하려고 하며 테스트 결과를 기준으로 코드를 수정하고 있습니다.

**사람이 AI 의 보조를 받아 직접 구현하던 시대**는 빠르게 지나고, **사람이 의도를 정의하고 AI 가 구현하는 시대**로 넘어왔습니다.

</aside>

### ⚠️ 문제는 속도 가 아니라 통제 다

<aside>
💡 AI 는 코드를 빠르게 생성할 수 있고, 다량의 문서를 빠르게 읽어낼 수 있습니다.
하지만 **우리 비즈니스의 특성, 코드의 의도, 변경영향과 책임** 등에 대해서는 지속적으로 컨텍스트를 유지할 수 없습니다.

이에, 명확하게 코드에 대한 의도를 분명히 하고 달리는 **자동차의 바퀴를 갈아끼울 수 있도록** 안내하고 조율할 수 있어야 합니다.

</aside>

### 🧠 개발자의 역할은 어떻게 바뀌고 있는가

- Agent 를 더 효율적인 생산성과 광범위한 역할의 확장을 위해 잘 활용해 비즈니스의 성장을 가속화
- 사내의 인프라 및 필요한 기술에 대한 구현 비용을 축소, 가능성을 확장하고 이를 잘 통제

### 참고

["그만 쓸게요" 카카오도 '결단'…AI 등장에 비명 쏟아진 곳](https://n.news.naver.com/article/015/0005240907?sid=105)

### 🤖 Claude Code

<aside>
💡

**Loopers** 에서는 **Claude Code** 를 기반으로 과정을 진행합니다.

</aside>

- 많은 AI 도구들이 존재하지만, [Claude Code](https://claude.com/product/claude-code) 는 근래 가장 많이 사용되고 있는 Antrophic 이 개발한 터미널 기반의 **에이전트 코딩 도구**
- CLI 환경에서 동작하므로 bash / git / project 등에도 제약없이 접근이 가능
- Claude Code 가 제공하는 기능
    - **개발자가 제공한 프롬프트** → 해석 → 계획 수립 → 코드 작성 → 동작 확인
    - **버그에 관련한 프롬프트** → 코드 베이스 분석 및 문제 식별, 수정
    - **코드베이스 탐색** → 전체 프로젝트 구조를 인식하고, 탐색 가능
    - **MCP 활용** → 필요한 도구들과의 통합을 제공하고, 원하는 동작을 자동화

---

# 💯 테스트

## 🧪 테스트 피라미드

> 테스트는 아래처럼 **범위에 따라 역할과 책임이 나뉘며**,
하단일수록 빠르고 많이, 상단일수록 느리지만 신중하게 구성됩니다.
>

![Untitled](attachment:54f631d6-538a-44fa-8358-026c73efed68:Untitled.png)

### 🧱 1. **단위 테스트 (Unit Test)**

- **대상:** 도메인 모델 (Entity, VO, Domain Service)
- **목적:** 순수 로직의 정합성과 규칙 검증
- **환경:** Spring 없이 순수 JVM에서 실행 (JVM 단위 테스트) / **테스트 대역** 을 활용해 모든 의존성을 대체
- **기술:** JUnit5, Kotest, AssertJ 등

> 💬 예: 포인트 충전 시 최대 한도 초과 여부를 검증하는 테스트
>

### 🔁 2. **통합 테스트 (Integration Test)**

- **대상:** 애플리케이션의 Service, Facade 등 계층 로직
- **목적:** 여러 컴포넌트(Repo, Domain, 외부 API Stub)가 연결된 상태에서 **비즈니스 흐름 전체를 검증**
- **환경:** `@SpringBootTest`, 실제 Bean 구성, Test DB
- **기술:** SpringBootTest + H2 + TestContainers 등

> 💬 예: 실제 포인트가 충전되고, DB에 반영되며, 이벤트가 발행되는 전 과정을 검증
>

### 🌐 3. **E2E 테스트 (End-to-End Test)**

- **대상:** 전체 애플리케이션 (Controller → Service → DB)
- **목적:** 실제 HTTP 요청 단위 시나리오 테스트
- **환경:** `MockMvc` 또는 `TestRestTemplate`을 통해 실제 API 요청 시뮬레이션
- **기술:** SpringBootTest + `@AutoConfigureMockMvc`, `WebTestClient` 등

> 💬 예: 사용자가 회원가입 → 포인트 충전 → 주문 흐름을 HTTP 요청으로 수행했을 때의 결과 확인
>

---

## 🔧 테스트 더블(Test Doubles)

> 테스트 대상이 의존하는 외부 객체의 동작을 **빠르고 안전하게 흉내 내는 대역 객체** 입니다.
느리고 불안정한 실제 구현 대신, 테스트 환경에 맞는 **‘조용한 대역’** 을 세워줍니다.
>

### 🧩 테스트 더블은 역할, `mock()`과 `spy()`는 도구

- `Stub`, `Mock`, `Spy`, `Fake` 는 **테스트 목적 (역할)**
- `mock()`, `spy()`는 **객체 생성 방식 (도구)**

e.g.

```kotlin
val repo = mock<UserRepository>() // 도구: mock()
whenever(repo.findById(1L)).thenReturn(User(...)) // 역할: Stub
verify(repo).findById(1L) // 역할: Mock
```

> ✅ mock 객체에 stub + mock 역할을 동시에 부여할 수 있습니다.
>

### 📚 TestDouble 역할별 정리

| 역할 | 목적 | 사용 방식 | 예시 |
| --- | --- | --- | --- |
| **Dummy** | 자리만 채움 (사용되지 않음) | 생성자 등에서 전달 | `User(null, null)` |
| **Stub** | 고정된 응답 제공 (상태 기반) | `when().thenReturn()` | `repo.find()` → 항상 특정 유저 반환 |
| **Mock** | 호출 여부/횟수 검증 (행위 기반) | `verify(...)` | 함수가 실행되었는지 검증 |
| **Spy** | 진짜 객체 감싸기 + 일부 조작 | `spy()` + `doReturn()` | 진짜 서비스 감싸고 일부만 stub |
| **Fake** | 실제처럼 동작하는 가짜 구현체 | 직접 클래스 구현 | **InMemoryUserRepository** |

### 🔁 TestDouble 실전 예제

### 📦 Stub 예제

```kotlin
val userRepo = mock<UserRepository>()
whenever(userRepo.findById(1L)).thenReturn(User("alen"))
```

- 흐름만 통제하고 싶은 경우
- “이렇게 호출하면, 이렇게 응답해줘”

### 📬 Mock 예제

```kotlin
val speaker = mock<Speaker>()
speaker.say("hello")
verify(speaker, times(1)).say("hello")
```

- 호출 여부가 검증 대상
- “너 이렇게 동작했니?”

### 🕵️ Spy 예제

```kotlin
val friend = Friend()
val spyFriend = spy(friend)
spyFriend.hangout()
verify(spyFriend).hangout()
```

- 진짜 객체처럼 동작하면서 일부만 조작
- "로직은 그대로 쓰고, 특정 동작만 덮어씌우고 / 검증하고 싶다"

### 🧪 Fake 예제

```kotlin
class InMemoryUserRepository : UserRepository {
    private val data = mutableMapOf<Long, User>()

    override fun save(user: User) { data[user.id] = user }
    override fun findById(id: Long): User? = data[user.id]
}
```

- 실제 DB 없이 테스트 가능한 저장소 구현
- "완전히 독립적인 테스트 환경이 필요할 때”

---

## 🧱 테스트 가능한 구조

> **검증하고 싶은 로직을, 외부 의존성과 격리된 상태에서 단독으로 검증할 수 있는 구조**입니다.
>
>
> 테스트 가능한 구조란, 검증하고 싶은 코드만 정확히 꺼내서 **조용하고 단단하게 확인할 수 있는 구조**다.
>

### ❌ 테스트하기 어려운 구조의 특징

| 문제 | 설명 |
| --- | --- |
| **내부에서 의존 객체 직접 생성 (`new`)** | 테스트 대역으로 대체 불가 → 테스트 격리 불가능 |
| **하나의 함수가 너무 많은 책임** | 테스트 대상이 모호해짐 → 실패 원인 추적 어려움 |
| **외부 API 호출, DB 접근 등이 하드코딩** | 실제 환경 없이 테스트 불가능 → 느리고 불안정 |
| **private 로직, static 메서드 남용** | 외부에서 로직 분리 불가 → 단위 테스트 불가 |

### ✅ 테스트 가능한 구조로 변경

| 포인트 | 설명 |
| --- | --- |
| **외부 의존성 분리** | 인터페이스화 + 생성자 주입(DI) |
| **비즈니스 로직 분리** | 도메인 엔티티 or 전용 Service에서 책임 분산 |
| **책임 단일화** | 한 함수는 한 역할만 (e.g. 결제만, 재고만 등) |
| **상태 중심 설계** | “입력 → 상태 변화 → 결과” 구조로 정리 |

### 🔍 사례로 살펴보기

```kotlin
class OrderService {
    fun completeOrder(userId: Long, productId: Long) {
        val user = UserJpaRepository().findById(userId)
        val product = ProductJpaRepository().findById(productId)

        if (product.stock <= 0) throw IllegalStateException()
        product.stock--

        if (user.point < product.price) throw IllegalStateException()
        user.point -= product.price

        OrderRepository().save(Order(user, product))
    }
}
```

- 외부 의존성 직접 생성 → Mock/Fake 불가
- 도메인 로직, 상태변경, 외부 호출이 한 곳에 몰려 있음
- `OrderServiceTest` 하나로 모든 케이스 커버해야 함 → 실패 시 어디서 잘못됐는지 추적 불가

---

```kotlin
class OrderService(
    private val userReader: UserReader,
    private val productReader: ProductReader,
    private val orderRepository: OrderRepository,
) {
    fun completeOrder(command: OrderCommand) {
        val user = userReader.get(command.userId)
        val product = productReader.get(command.productId)

        product.decreaseStock()
        user.pay(product.price)

        orderRepository.save(Order(user, product))
    }
}
```

- 외부는 인터페이스로 주입 → Fake/Mock 가능
- 로직은 `user.pay()`, `product.decreaseStock()` 처럼 도메인으로 위임
- 테스트 단위별로 나눌 수 있음 → `UserTest`, `ProductTest`, `OrderServiceTest`

---

## 🔁 TDD (Test-Driven Development)

> TDD는 테스트의 순서보다
**”설계 단위를 잘게 쪼개고, 그것이 검증 가능하게 구현되었는가”**가 핵심이다.
>

### 🔄 3단계 루프: Red → Green → Refactor

```
< 반복 >
1. 실패하는 테스트 작성 (Red)
2. 통과할 최소한의 코드 작성 (Green)
3. 구조 개선 및 리팩토링 (Refactor)
```

### 🧠 그런데 꼭 테스트를 먼저 써야 할까?

| **전략** | **이름** | **설명** |
| --- | --- | --- |
| 🧪 TFD (Test First Development) | 테스트 먼저 작성 → 코드를 맞춰 구현 | 도메인/로직 중심에 적합 |
| 🏗 TLD (Test Last Development) | 코드를 먼저 작성 → 테스트는 나중에 작성 | API/계층 설계가 먼저 필요한 상황에 적합 |

### 🟢 TDD가 필요한 이유

- **요구사항을 먼저 정리할 수 있다**
- **작게 쪼개고 점진적으로 설계하게 된다**
- **인터페이스 설계가 자연스럽게 나온다**
- **리팩토링이 가능해진다**

<aside>
📚

**References**

</aside>

| 구분 | 링크 |
| --- | --- |
| 🔢 테스트 피라미드 | [Testing Pyramid - Martin Fowler](https://martinfowler.com/bliki/TestPyramid.html) |
| 🧪 JUnit5 | [JUnit5 공식 문서](https://junit.org/junit5/docs/current/user-guide/) |
| ⚙️ Mockito | [Mockito 공식 문서](https://site.mockito.org/) |
| 🧰 Mockito-Kotlin | [GitHub: mockito-kotlin](https://github.com/mockito/mockito-kotlin) |
| 🧵 Spring 테스트 | [Spring Boot Testing Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing) |

> 본 과정에서는 원활한 멘토링을 위해  `JUnit5 + Mockito` 기반으로 진행합니다.
>

<aside>
🌟

**Next Week Preview**

</aside>

> 다음 주에는 본격적으로 우리만의 e-commerce 시스템을 **설계** 해봅니다.
>