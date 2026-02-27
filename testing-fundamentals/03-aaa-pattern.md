# 03. AAA Pattern

> **Arrange-Act-Assert는 "무엇을 준비하고, 무엇을 실행하고, 무엇을 검증하는가"를 한눈에 읽히게 만드는 구조다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- AAA의 각 단계는 정확히 무엇을 담당하는가?
- AAA가 무너지는 신호는 무엇인가?
- `Given-When-Then`과 AAA는 어떤 관계인가?

---

## 🔍 왜 구조가 필요한가

테스트 코드에 구조가 없으면 이렇게 됩니다.

```java
// ❌ 구조 없는 테스트 — 무엇을 검증하는지 3초 안에 파악할 수 없다
@Test
void 주문_테스트() {
    OrderRepository repo = new InMemoryOrderRepository();
    Order order = new Order();
    order.addItem(new Item("책", 20_000));
    assertThat(order.items()).hasSize(1);
    order.addItem(new Item("펜", 3_000));
    OrderService service = new OrderService(repo, new RateDiscountPolicy(10));
    assertThat(order.totalPrice()).isEqualTo(23_000);
    User user = new User("alice", Grade.VIP);
    Order result = service.place(order, user);
    assertThat(result.totalPrice()).isEqualTo(20_700);
    assertThat(repo.findById(result.id())).isPresent();
}
```

이 테스트는 다음 질문에 바로 답하지 못합니다.
- **무엇을 테스트하고 있는가?** — 상품 추가? 가격 계산? 주문 저장?
- **이 테스트가 실패하면 어디서 실패했는가?** — `assertThat`이 3개 흩어져 있음
- **준비 단계는 어디서 끝나는가?** — 중간에 객체 생성과 검증이 섞임

---

## 🏛️ AAA의 세 단계

### Arrange — 준비

테스트 실행에 필요한 모든 것을 세팅합니다. 협력 객체 생성, 테스트 데이터 구성, Stub 설정이 여기에 속합니다. Arrange 이후에는 테스트 대상 외에 아무것도 바뀌지 않아야 합니다.

### Act — 실행

**딱 하나의 동작**만 실행합니다. 이 단계는 보통 한 줄입니다. 두 줄 이상이 된다면 테스트가 두 가지 동작을 한 번에 검증하고 있다는 신호입니다.

### Assert — 검증

실행 결과가 기대한 것과 일치하는지 확인합니다. 단, Act의 결과만 검증합니다. Act 이전의 상태나 중간 과정을 검증하는 것은 Assert에 속하지 않습니다.

---

## 😱 AAA가 무너지는 패턴

### 패턴 1: Act-Assert-Act-Assert (검증이 중간에 끼어든다)

```java
@Test
void 아이템_추가와_주문_생성_테스트() {
    // Arrange
    Order order = new Order();

    // Act 1
    order.addItem(new Item("책", 20_000));

    // Assert 1 ← 여기서 멈추지 않는다
    assertThat(order.items()).hasSize(1);

    // Act 2
    order.addItem(new Item("펜", 3_000));

    // Assert 2
    assertThat(order.totalPrice()).isEqualTo(23_000);
}
```

**문제:** 두 개의 동작을 하나의 테스트에서 검증하고 있습니다. 첫 번째 Assert가 실패하면 두 번째 Act는 실행조차 안 됩니다. 이 테스트는 2개로 분리해야 합니다.

### 패턴 2: Arrange가 Act 안에 숨어있다

```java
@Test
void 할인_적용된_주문_생성() {
    // ❌ service 생성이 Act처럼 보이는 Arrange 안에 있다
    Order result = new OrderService(
        new InMemoryOrderRepository(),
        new RateDiscountPolicy(10)      // ← 이게 테스트 핵심인데 눈에 안 띔
    ).place(
        new Order(List.of(new Item("책", 20_000))),
        new User("alice", Grade.VIP)
    );

    assertThat(result.totalPrice()).isEqualTo(18_000);
}
```

**문제:** "10% 할인 정책이 적용된다"는 전제가 Act 안에 숨어있어서 읽는 사람이 맥락을 파악하기 어렵습니다.

### 패턴 3: Assert가 없는 테스트

```java
@Test
void 주문을_저장한다() {
    Order order = new Order(List.of(new Item("책", 20_000)));
    orderService.place(order, user);
    // Assert 없음 — 예외만 안 나면 통과
}
```

**문제:** 예외가 발생하지 않는다는 것 외에 아무것도 검증하지 않습니다. 저장이 실제로 됐는지, 저장된 값이 맞는지 아무도 검증하지 않습니다.

---

## ✨ 올바른 AAA 패턴

```java
@Test
void VIP_회원_주문_시_10퍼센트_할인이_적용된다() {

    // Arrange ─────────────────────────────────────
    RateDiscountPolicy discountPolicy = new RateDiscountPolicy(10);
    OrderRepository repository = new InMemoryOrderRepository();
    OrderService service = new OrderService(repository, discountPolicy);

    User vipUser = new User("alice", Grade.VIP);
    Order order = new Order(List.of(new Item("책", 20_000)));

    // Act ──────────────────────────────────────────
    Order result = service.place(order, vipUser);

    // Assert ───────────────────────────────────────
    assertThat(result.totalPrice()).isEqualTo(18_000);
}
```

구분선(`// Arrange`, `// Act`, `// Assert`) 또는 빈 줄로 세 단계를 시각적으로 분리합니다. 취향의 문제지만, 팀 내 한 가지 방식으로 통일하는 것이 중요합니다.

### 각 단계별 크기 기준

| 단계 | 적정 크기 | 너무 길어지면? |
|------|----------|--------------|
| Arrange | 3~7줄 | Test Data Builder 도입 고려 |
| Act | 1줄 (거의 항상) | 테스트를 2개로 분리 |
| Assert | 1~3줄 | 테스트가 너무 많은 것을 검증 중 |

---

## 💻 실전 적용: Given-When-Then과의 관계

`Given-When-Then`은 BDD(Behavior-Driven Development)에서 온 표현이고, `Arrange-Act-Assert`는 xUnit 계열에서 온 표현입니다. 둘은 같은 구조를 다르게 부르는 것입니다.

```java
// JUnit5 + 한국어 테스트 이름에서 자주 쓰는 방식
@Test
void VIP_회원_주문_시_10퍼센트_할인이_적용된다() {

    // given
    User vipUser = new User("alice", Grade.VIP);
    Order order = new Order(List.of(new Item("책", 20_000)));

    // when
    Order result = service.place(order, vipUser);

    // then
    assertThat(result.totalPrice()).isEqualTo(18_000);
}
```

팀에서 BDD 언어(시나리오 중심 설명)를 선호한다면 `given/when/then`, 전통적인 유닛 테스트 스타일이라면 `Arrange/Act/Assert`를 사용합니다. 어느 쪽이든 **구조 자체가 중요**하지, 주석 단어가 중요한 것은 아닙니다.

---

## 🤔 트레이드오프

### "Act이 꼭 한 줄이어야 하는가?"

꼭 그렇지는 않습니다. 단, **하나의 동작을 위한 준비가 Act에 들어가면 안 됩니다.**

```java
// Act이 두 줄이어도 괜찮은 경우 — 두 줄이 하나의 동작을 구성할 때
Order placed = service.place(order, user);
PaymentResult result = paymentService.pay(placed, card);
```

```java
// ❌ Act이 두 줄이면 안 되는 경우 — 두 개의 독립적 동작
service.place(order, user);                    // ← 동작 1
Order saved = repository.findById(order.id()); // ← 이건 Assert 준비 (Arrange에 해당)
assertThat(saved.status()).isEqualTo(PENDING);
```

두 번째 케이스에서 `findById()`는 Act가 아닌 Assert를 위한 조회이므로, 주석 없이 바로 assert에서 쓰거나 Assert 바로 위에 명시합니다.

---

## 📌 핵심 정리

```
AAA = Given-When-Then (같은 구조, 다른 단어)

Arrange (Given):
  테스트 실행에 필요한 모든 것을 세팅
  테스트 대상 외에는 아무것도 바뀌지 않는 상태로 끝냄

Act (When):
  딱 하나의 동작 실행
  거의 항상 한 줄

Assert (Then):
  Act의 결과만 검증
  중간 과정이나 Act 이전 상태는 검증하지 않음

AAA가 무너지는 신호:
  Act-Assert-Act-Assert → 테스트를 분리하라
  Arrange가 10줄 이상 → Test Data Builder를 도입하라
  Assert가 없다 → 무엇을 검증하는지 다시 생각하라
  Act가 두 줄 이상 → 두 동작을 한 테스트에서 검증하는 건 아닌지 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트에서 AAA 구조가 깨진 부분을 찾고, 어떻게 분리하거나 수정하겠는가?

```java
@Test
void 장바구니와_주문_테스트() {
    Cart cart = new Cart();
    cart.addItem(new Item("책", 20_000));
    assertThat(cart.totalPrice()).isEqualTo(20_000);
    cart.addItem(new Item("펜", 3_000));
    User user = new User("alice", Grade.VIP);
    Order order = orderService.place(cart, user);
    assertThat(order.totalPrice()).isEqualTo(20_700);
    assertThat(order.status()).isEqualTo(OrderStatus.PENDING);
}
```

**Q2.** Arrange 단계에서 Stub 설정을 할 때 아래 두 스타일 중 어느 것이 더 나은가? 이유를 AAA 관점에서 설명하라.

```java
// 스타일 A
@BeforeEach
void setUp() {
    when(discountPolicy.calculate(any())).thenReturn(10);
}

@Test
void 주문_생성_테스트() {
    Order result = service.place(order, user);
    assertThat(result.totalPrice()).isEqualTo(18_000);
}

// 스타일 B
@Test
void 주문_생성_테스트() {
    // given
    when(discountPolicy.calculate(user)).thenReturn(10);
    // when
    Order result = service.place(order, user);
    // then
    assertThat(result.totalPrice()).isEqualTo(18_000);
}
```

**Q3.** "Assert 없는 테스트는 의미가 없다"고 했지만, 반드시 예외가 발생하는지를 검증하는 테스트는 어떻게 써야 하는가? JUnit5 스타일로 작성하라.

> 💡 **해설**
>
> **Q1.** 두 가지 문제가 있다. ① `cart.addItem()` 이후 `assertThat(cart.totalPrice())` — 이것은 `장바구니 아이템 추가` 테스트이고, 아래 `orderService.place()` 테스트와 분리해야 한다. ② 하나의 테스트에서 3개의 독립적 동작(카트 생성, 주문 생성, 상태 확인)을 검증하고 있다. 분리 방법: `장바구니에_아이템을_추가하면_총가격이_합산된다()` + `VIP_회원_주문_시_할인이_적용된다()` + `주문_생성_직후_상태는_PENDING이다()` 3개로 나눈다.
>
> **Q2.** 스타일 B가 낫다. AAA 관점에서 Stub 설정(`when(discountPolicy...)`)은 명백히 Arrange의 일부다. `@BeforeEach`에 넣으면 어떤 Stub이 이 테스트에 영향을 미치는지 파악하기 위해 다른 메서드로 이동해야 한다(컨텍스트 스위칭). 테스트 하나만 읽었을 때 Arrange-Act-Assert가 완결되어야 이해하기 쉽다. 단, 모든 테스트에 공통으로 필요한 최소 Setup(객체 생성 등)은 `@BeforeEach`가 적절하다.
>
> **Q3.** `assertThatThrownBy()`를 사용한다. 이것 자체가 Assert다: `assertThatThrownBy(() -> service.place(invalidOrder, user)).isInstanceOf(MinimumAmountException.class).hasMessage("최소 주문 금액은 1,000원입니다")`. 람다 안의 `service.place()`가 Act이고, `assertThatThrownBy`가 Assert이므로 AAA 구조가 유지된다.

---

<div align="center">

**[⬅️ 이전: The Test Pyramid](./02-the-test-pyramid.md)** | **[다음: Test Naming Conventions ➡️](./04-test-naming-conventions.md)**

</div>
