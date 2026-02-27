# 01. What Is a "Unit"?

> **단위(Unit)를 어떻게 정의하느냐가 Mock 전략, 테스트 구조, 설계 피드백 전부를 결정한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- "단위"가 클래스인가, 동작인가 — 왜 이 대답이 팀마다 다른가?
- 같은 코드에 런던파 스타일로 테스트를 짜면 무슨 문제가 생기는가?
- 고전파와 런던파 중 실무에서 어떤 기준으로 선택하는가?

---

## 🔍 왜 이 질문이 중요한가

```java
public class OrderService {
    private final DiscountPolicy discountPolicy;
    private final OrderRepository orderRepository;

    public Order place(Cart cart, User user) {
        int discount = discountPolicy.calculate(user);
        Order order = Order.from(cart, discount);
        orderRepository.save(order);
        return order;
    }
}
```

`OrderService.place()`를 테스트할 때, 아래 두 팀의 접근이 완전히 다릅니다.

**팀 A (런던파):**  
`DiscountPolicy`와 `OrderRepository`를 **Mock으로 교체**한다.  
`OrderService`만 격리해서 검증하는 것이 단위 테스트라고 생각하기 때문이다.

**팀 B (고전파):**  
`DiscountPolicy`는 **실제 구현**을 사용하고, `OrderRepository`만 In-memory Fake로 교체한다.  
`place()` 라는 동작(behavior) 전체가 올바른지 검증하는 것이 단위 테스트라고 생각하기 때문이다.

두 팀 모두 "단위 테스트를 작성하고 있다"고 말하지만, 전혀 다른 테스트를 만들고 있습니다.  
**이 차이가 쌓이면 팀의 테스트 전략 전체가 달라집니다.**

---

## 🏛️ 두 학파의 정의

### 런던파 (London School / Mockist)

> **"단위 = 클래스(Class). 하나의 클래스를 모든 협력자로부터 격리해서 테스트한다."**

- 모든 의존성을 Mock으로 교체
- 한 테스트가 실패하면 정확히 어떤 클래스가 문제인지 알 수 있음
- 대표 저작: *Growing Object-Oriented Software, Guided by Tests* (Freeman, Pryce)

### 고전파 (Classical School / Chicago / Detroit)

> **"단위 = 동작(Behavior). 의미 있는 기능 하나를 검증한다. 협력자는 실제 구현을 쓴다."**

- 외부 시스템(DB, 네트워크)만 교체
- 테스트가 구현이 아닌 결과에 집중
- 대표 저작: *Test-Driven Development: By Example* (Kent Beck)

---

## 😱 런던파를 극단적으로 적용하면

아래는 런던파 스타일로 `OrderService`를 테스트한 코드입니다.

```java
// ❌ 모든 협력자를 Mock으로 교체한 테스트
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock DiscountPolicy discountPolicy;
    @Mock OrderRepository orderRepository;
    @InjectMocks OrderService orderService;

    @Test
    void place_주문을_생성하고_저장한다() {
        // Arrange
        User user = new User("alice");
        Cart cart = new Cart(List.of(new Item("책", 20_000)));
        Order expectedOrder = Order.from(cart, 10);

        when(discountPolicy.calculate(user)).thenReturn(10);
        when(orderRepository.save(any())).thenReturn(expectedOrder);

        // Act
        Order result = orderService.place(cart, user);

        // Assert
        assertThat(result).isEqualTo(expectedOrder);
        verify(orderRepository).save(any(Order.class));  // ← 구현 세부사항 검증
        verify(discountPolicy).calculate(user);           // ← 구현 세부사항 검증
    }
}
```

### 무엇이 문제인가?

**1. 구현을 검증하는 테스트가 됩니다**

`verify(discountPolicy).calculate(user)`는 "할인이 올바르게 적용됐는가?"가 아니라  
"`discountPolicy.calculate()`를 호출했는가?"를 검증합니다.  
내부 구현을 바꾸면 테스트가 깨지지만, 실제 동작은 올바를 수 있습니다.

**2. 테스트가 설계에 속박됩니다**

`OrderService`가 `DiscountPolicy`를 직접 호출하지 않고,  
`PricingEngine`을 통해 간접 호출하도록 리팩터링하면?  
→ 기능은 동일한데 테스트 전부가 깨집니다.

**3. Mock 설정이 프로덕션 로직을 중복합니다**

```java
when(discountPolicy.calculate(user)).thenReturn(10);
```

이 `10`이라는 값은 실제 `DiscountPolicy` 내부 로직의 결과입니다.  
테스트에서 이 값을 하드코딩하는 순간, **실제 할인 계산이 맞는지 아무도 검증하지 않습니다.**

---

## ✨ 고전파 접근: 동작을 검증한다

```java
// ✅ 협력자는 실제 구현 사용, 외부 시스템(DB)만 교체
class OrderServiceTest {

    // DiscountPolicy: 실제 구현 사용
    private final DiscountPolicy discountPolicy = new RateDiscountPolicy(10); // 10% 할인
    // OrderRepository: In-memory Fake 사용 (DB 연결 없이 빠름)
    private final OrderRepository orderRepository = new InMemoryOrderRepository();

    private final OrderService orderService =
        new OrderService(discountPolicy, orderRepository);

    @Test
    void 회원_등급에_따라_할인이_적용된_주문이_저장된다() {
        // Arrange
        User vipUser = new User("alice", Grade.VIP);
        Cart cart = new Cart(List.of(new Item("책", 20_000)));

        // Act
        Order savedOrder = orderService.place(cart, vipUser);

        // Assert — "동작의 결과"를 검증
        assertThat(savedOrder.totalPrice()).isEqualTo(18_000); // 10% 할인 적용
        assertThat(orderRepository.findById(savedOrder.id())).isPresent();
    }
}
```

### 무엇이 달라졌는가?

| 관점 | Mock 남용 | 고전파 접근 |
|------|-----------|-------------|
| 검증 대상 | `calculate()` 호출 여부 | 할인이 실제로 적용된 금액 |
| 리팩터링 내성 | 내부 구조 바꾸면 테스트 깨짐 | 결과가 같으면 테스트 통과 |
| 테스트 실패 의미 | "이 메서드를 호출 안 했어" | "주문 금액이 틀렸어" |
| 발견하는 버그 | 호출 경로 오류 | 실제 비즈니스 로직 오류 |

---

## 💻 실전 적용: In-memory Fake 만들기

고전파 접근에서 핵심은 **외부 시스템을 Fake로 교체**하는 것입니다.  
Fake는 Mock과 달리 실제 동작하는 최소 구현체입니다.

```java
// 프로덕션 인터페이스
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}

// 테스트용 Fake — 실제로 동작하는 In-memory 구현
class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    private long idSequence = 1L;

    @Override
    public Order save(Order order) {
        Order saved = order.withId(idSequence++);
        store.put(saved.id(), saved);
        return saved;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

Fake의 장점:
- Mock 설정 코드가 없어 테스트가 짧아집니다
- 실제 Repository처럼 동작하므로 `save()` 후 `findById()`로 검증 가능합니다
- 여러 테스트에서 재사용할 수 있습니다

---

## 🤔 트레이드오프

고전파가 항상 옳은 것은 아닙니다.

### 런던파가 유리한 경우

**협력 객체의 초기화 비용이 클 때**  
예: `PricingEngine`이 수백 개의 룰을 로드하는 경우.  
실제 구현을 쓰면 테스트가 느려집니다.

**협력 객체가 아직 구현되지 않았을 때**  
인터페이스만 있고 구현체가 없다면 Mock이 유일한 선택입니다.  
TDD로 설계를 먼저 탐색할 때 런던파 방식이 효과적입니다.

**협력 관계 자체가 설계 목표일 때**  
"어떤 메시지를 보내는가"가 핵심인 메시징 시스템에서는  
`verify()`로 메시지 전송 자체를 검증하는 것이 맞습니다.

### 고전파가 유리한 경우

**리팩터링을 자주 하는 코드베이스**  
구현이 바뀌어도 동작이 같다면 테스트가 통과해야 합니다.

**비즈니스 규칙이 여러 클래스에 걸쳐 있을 때**  
할인 계산이 `DiscountPolicy` + `MembershipPolicy` + `CouponPolicy`에 분산되어 있다면  
이 전체 협력 결과를 하나의 테스트에서 검증하는 것이 현실적입니다.

---

## 📌 핵심 정리

```
단위(Unit)의 두 가지 정의

런던파: 단위 = 클래스
  → 모든 협력자를 Mock으로 교체
  → 장점: 실패 위치 정확, 격리 완벽
  → 단점: 구현에 속박, 리팩터링에 취약

고전파: 단위 = 동작 (behavior)
  → 외부 시스템만 교체 (DB, 네트워크)
  → 장점: 리팩터링 내성, 실제 버그 발견
  → 단점: 실패 시 원인 파악이 상대적으로 어려움

실무 기준:
  ✅ 기본은 고전파 — 협력자는 실제 구현 사용
  ✅ 외부 시스템 → Fake 또는 Test Double
  ✅ 협력자 초기화 비용이 크다면 → Mock 고려
  ✅ "호출했는가"가 아닌 "결과가 맞는가"를 검증하라
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 런던파 개발자와 고전파 개발자는 각각 무엇을 Mock으로 교체하겠는가? 그리고 두 접근의 테스트가 **동시에 통과하지만 버그가 있는** 시나리오를 하나 설명하라.

```java
public class NotificationService {
    private final UserRepository userRepository;
    private final EmailSender emailSender;
    private final TemplateEngine templateEngine;

    public void sendWelcome(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        String body = templateEngine.render("welcome", user);
        emailSender.send(user.email(), body);
    }
}
```

**Q2.** 팀에서 고전파 스타일로 테스트를 작성하기로 결정했다. 그런데 `DiscountPolicy`의 실제 구현이 외부 프로모션 API를 호출한다는 것을 나중에 알게 됐다. 이 상황에서 어떻게 경계를 다시 설정하겠는가?

**Q3.** 고전파 방식에서 테스트 하나가 실패했을 때 런던파보다 원인 파악이 어렵다고 했다. 이 단점을 완화하면서도 고전파 스타일을 유지하는 방법은 무엇인가?

> 💡 **해설**
>
> **Q1.** 런던파는 `UserRepository`, `EmailSender`, `TemplateEngine` **셋 다** Mock으로 교체한다. 고전파는 `EmailSender`만 교체하고(실제 메일 발송 방지) 나머지는 실제 구현을 쓴다. 동시 통과하지만 버그가 있는 시나리오: `templateEngine.render()`가 사용자 이름을 HTML 인코딩 없이 삽입해 XSS가 발생하는 경우. 런던파 테스트는 `render()`를 Mock해서 `"<b>Hello</b>"`를 그냥 반환하므로 이 버그를 발견하지 못한다. 고전파 테스트도 Email body 내용을 검증하지 않으면 동일하게 통과한다. 두 테스트 모두 "이메일을 보냈는가"에만 집중하고 있기 때문이다.
>
> **Q2.** 외부 API 호출은 명확한 경계다. `DiscountPolicy` 인터페이스를 두 계층으로 분리한다. `LocalDiscountPolicy`(순수 룰)는 실제 구현 사용, `PromotionApiClient`(외부 호출)는 Fake 또는 WireMock으로 교체한다. 경계 기준은 "내 프로세스 안에서 제어할 수 있는가"이다. 외부 네트워크 호출, 파일 시스템, DB가 경계이고 그 안쪽 협력자는 실제 구현을 쓰는 것이 고전파의 원칙이다.
>
> **Q3.** 테스트 이름을 동작 단위로 잘게 쪼개고, 실패 메시지를 풍부하게 만든다. `assertThat(order.totalPrice()).as("VIP 10% 할인이 적용돼야 한다").isEqualTo(18_000)` 처럼 `as()`로 의도를 명시하면 실패 시 어느 규칙이 깨졌는지 즉시 알 수 있다. 또한 각 테스트가 하나의 시나리오만 검증하도록 분리하면(단일 단언 원칙) 실패한 테스트 이름만 봐도 어떤 동작이 깨졌는지 파악된다.

---

<div align="center">

**[다음: The Test Pyramid ➡️](./02-the-test-pyramid.md)**

</div>
