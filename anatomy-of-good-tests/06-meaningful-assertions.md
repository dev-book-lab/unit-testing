# 06. Meaningful Assertions

> **테스트가 실패했을 때, 코드를 열지 않고도 "무엇이 왜 잘못됐는지" 알 수 있어야 한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- JUnit5의 기본 `assertEquals`와 AssertJ의 `assertThat`은 왜 다른가?
- 실패 메시지를 풍부하게 만드는 AssertJ 기법은 무엇인가?
- 커스텀 단언(Custom Assertion)은 언제 만들어야 하는가?

---

## 🔍 실패 메시지가 중요한 이유

```
테스트 실패 시 개발자의 다음 행동:

좋은 실패 메시지:
  "expected: 18000, but was: 20000"
  → 바로 DiscountPolicy 쪽 코드를 확인한다 (10초)

나쁜 실패 메시지:
  "expected: true, but was: false"
  → 어느 assertTrue가 실패했는지 테스트 코드를 열고
  → 어떤 값이 true여야 했는지 코드를 따라가고
  → 실제 값이 뭔지 디버거를 켜서 확인한다 (10분)
```

실패 메시지의 품질이 디버깅 시간을 결정합니다.

---

## 😱 빈약한 단언이 만드는 나쁜 메시지

### 패턴 1: assertTrue / assertFalse

```java
// ❌ assertTrue — 실패해도 아무 정보가 없다
assertTrue(order.totalPrice() == 18_000);
// 실패 메시지: "expected: true, but was: false"
// → 실제 값이 뭔지 전혀 모름

// ✅ AssertJ — 실패 시 실제 값을 보여준다
assertThat(order.totalPrice()).isEqualTo(18_000);
// 실패 메시지: "expected: 18000, but was: 20000"
```

### 패턴 2: 약한 단언

```java
// ❌ 어떤 값인지 전혀 모른다
assertThat(order.totalPrice()).isPositive();
// 실패 메시지: "expected positive but was: -100"
// → 왜 음수인지 모름

// ✅ 구체적인 기댓값
assertThat(order.totalPrice()).isEqualTo(18_000);
// 실패 메시지: "expected: 18000, but was: -100"
// → 기댓값과 실제값의 차이가 명확
```

### 패턴 3: 구조를 모르는 단언

```java
// ❌ List의 어떤 요소가 다른지 알 수 없다
assertThat(order.items()).isEqualTo(expectedItems);
// 실패 메시지: "expected: [Item{name='책', price=20000}], 
//              but was: [Item{name='책', price=20000}, Item{name='펜', price=3000}]"
// → 다르다는 건 알지만, 어느 요소가 문제인지 파악하기 어렵다

// ✅ 크기 먼저, 내용 검증 분리
assertThat(order.items()).hasSize(1);
assertThat(order.items()).extracting(Item::name).containsExactly("책");
```

---

## ✨ AssertJ 핵심 기법

### 1. `as()` — 단언에 설명 추가

```java
// ❌ 실패 시 어떤 필드가 문제인지 불명확
assertThat(order.totalPrice()).isEqualTo(18_000);
assertThat(order.status()).isEqualTo(PENDING);

// ✅ as()로 맥락 추가
assertThat(order.totalPrice())
    .as("VIP 10%% 할인이 적용된 최종 금액")
    .isEqualTo(18_000);
// 실패 메시지: "[VIP 10% 할인이 적용된 최종 금액] expected: 18000, but was: 20000"

assertThat(order.status())
    .as("신규 주문의 초기 상태")
    .isEqualTo(PENDING);
```

### 2. `extracting()` — 컬렉션 필드 추출

```java
// ✅ 여러 필드를 동시에 검증
List<Order> orders = orderService.findByUser(user);

assertThat(orders)
    .extracting(Order::status)
    .containsExactly(PENDING, COMPLETED, CANCELLED);

// 여러 필드 동시 추출
assertThat(orders)
    .extracting(Order::totalPrice, Order::status)
    .containsExactlyInAnyOrder(
        tuple(18_000, COMPLETED),
        tuple(9_000, CANCELLED)
    );
```

### 3. `usingRecursiveComparison()` — 객체 깊은 비교

```java
// ❌ equals가 구현되지 않은 객체 비교
assertThat(actualOrder).isEqualTo(expectedOrder);
// 실패 이유를 알 수 없음 (참조 비교가 됨)

// ✅ 필드 단위 재귀 비교
assertThat(actualOrder)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")     // 동적 생성 필드 제외
    .isEqualTo(expectedOrder);
// 실패 메시지: "field 'totalPrice': expected: 18000, but was: 20000"
```

### 4. `satisfies()` — 여러 조건을 람다로

```java
// ✅ 복합 조건을 하나의 블록에서 검증
assertThat(order).satisfies(o -> {
    assertThat(o.totalPrice()).isEqualTo(18_000);
    assertThat(o.status()).isEqualTo(PENDING);
    assertThat(o.items()).hasSize(1);
});
```

### 5. 예외 메시지까지 검증

```java
// ❌ 예외 타입만 검증
assertThatThrownBy(() -> orderService.place(emptyCart, user))
    .isInstanceOf(EmptyCartException.class);

// ✅ 예외 메시지도 검증 — 실수로 다른 예외를 던져도 잡힌다
assertThatThrownBy(() -> orderService.place(emptyCart, user))
    .isInstanceOf(EmptyCartException.class)
    .hasMessage("장바구니가 비어있습니다")
    .hasMessageContaining("비어있");

// 또는
assertThatExceptionOfType(EmptyCartException.class)
    .isThrownBy(() -> orderService.place(emptyCart, user))
    .withMessage("장바구니가 비어있습니다");
```

---

## 💻 실전 적용: 커스텀 단언 만들기

같은 검증 로직이 여러 테스트에서 반복된다면 커스텀 단언으로 추출합니다.

```java
// ❌ 반복되는 단언 패턴
@Test void 테스트A() {
    Order order = ...;
    assertThat(order.status()).isEqualTo(COMPLETED);
    assertThat(order.totalPrice()).isGreaterThan(0);
    assertThat(order.completedAt()).isNotNull();
}

@Test void 테스트B() {
    Order order = ...;
    assertThat(order.status()).isEqualTo(COMPLETED);
    assertThat(order.totalPrice()).isGreaterThan(0);
    assertThat(order.completedAt()).isNotNull();
}
```

```java
// ✅ AbstractAssert를 상속한 커스텀 단언
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {

    public OrderAssert(Order actual) {
        super(actual, OrderAssert.class);
    }

    public static OrderAssert assertThat(Order actual) {
        return new OrderAssert(actual);
    }

    public OrderAssert isCompleted() {
        isNotNull();
        if (actual.status() != COMPLETED) {
            failWithMessage("주문 상태가 COMPLETED여야 하지만 <%s>입니다", actual.status());
        }
        if (actual.totalPrice() <= 0) {
            failWithMessage("완료된 주문의 금액은 0보다 커야 합니다");
        }
        if (actual.completedAt() == null) {
            failWithMessage("완료된 주문에는 completedAt이 있어야 합니다");
        }
        return this;
    }

    public OrderAssert hasTotalPrice(int expected) {
        if (actual.totalPrice() != expected) {
            failWithMessage("주문 금액이 <%d>여야 하지만 <%d>입니다", expected, actual.totalPrice());
        }
        return this;
    }
}

// 사용 — 체인으로 읽기 좋고 실패 메시지가 명확하다
@Test void 결제_완료_주문_검증() {
    Order order = paymentService.complete(orderId);
    assertThat(order).isCompleted().hasTotalPrice(18_000);
}
```

---

## 🤔 트레이드오프

### "AssertJ를 처음 쓰는 팀원에게 러닝 커브가 있다"

맞습니다. 하지만 `isEqualTo`, `hasSize`, `contains`, `isInstanceOf`, `extracting` 수준의 기본 API만 익혀도 JUnit5 기본 단언보다 훨씬 나은 실패 메시지를 얻을 수 있습니다. 커스텀 단언은 팀에 충분히 적응된 후 도입하면 됩니다.

### "커스텀 단언 코드도 테스트가 필요한가?"

커스텀 단언 자체는 단순한 `if`와 `failWithMessage`로 구성되므로 별도 테스트 없이도 관리 가능합니다. 다만 커스텀 단언이 복잡해진다면(내부 로직이 있다면) 테스트를 추가하는 것이 맞습니다.

---

## 📌 핵심 정리

```
나쁜 단언의 신호:
  assertTrue/assertFalse → 실제 값이 안 보임
  isNotNull() 외 아무것도 없음 → 무엇을 검증하는지 모름
  isPositive() → 기댓값이 없음

AssertJ 핵심 기법:
  .as("설명")         → 실패 메시지에 맥락 추가
  .extracting()       → 컬렉션 필드 추출 후 검증
  .usingRecursiveComparison() → equals 없어도 깊은 비교
  .satisfies()        → 복합 조건을 한 블록에서
  .hasMessage()       → 예외 메시지까지 검증

커스텀 단언:
  같은 검증 패턴이 3회 이상 반복 → 추출 고려
  AbstractAssert 상속으로 체이닝 가능
  도메인 언어로 단언을 표현할 수 있음

기준:
  실패 메시지만 보고 코드를 열지 않아도
  "무엇이 왜 틀렸는지" 알 수 있어야 한다
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 단언들을 AssertJ로 리팩터링하고, 각각의 실패 메시지가 어떻게 달라지는지 비교하라.

```java
// 리팩터링 대상
assertTrue(order != null);
assertTrue(order.totalPrice() > 0);
assertEquals(order.status(), OrderStatus.PENDING);
assertTrue(order.items().size() == 1);
```

**Q2.** 아래 상황에서 커스텀 단언을 만드는 것이 적절한가? 판단 기준과 함께 설명하라.

```java
// 3개의 테스트 클래스에서 반복되는 패턴
assertThat(response.statusCode()).isEqualTo(200);
assertThat(response.contentType()).isEqualTo("application/json");
assertThat(response.body()).isNotEmpty();
```

**Q3.** 예외를 검증하는 두 스타일 중 어느 것이 더 나은가? 그 이유는 무엇인가?

```java
// 스타일 A
try {
    orderService.place(emptyCart, user);
    fail("예외가 발생해야 합니다");
} catch (EmptyCartException e) {
    assertThat(e.getMessage()).contains("비어있");
}

// 스타일 B
assertThatThrownBy(() -> orderService.place(emptyCart, user))
    .isInstanceOf(EmptyCartException.class)
    .hasMessageContaining("비어있");
```

> 💡 **해설**
>
> **Q1.** `assertTrue(order != null)` → `assertThat(order).isNotNull()` (실패 시: "expected not null, but was null"). `assertTrue(order.totalPrice() > 0)` → `assertThat(order.totalPrice()).isPositive()` 또는 더 명확하게 `.isGreaterThan(0)` (실패 시: "expected positive, but was: -100"). `assertEquals(order.status(), OrderStatus.PENDING)` → `assertThat(order.status()).isEqualTo(OrderStatus.PENDING)` (실패 시: "expected: PENDING, but was: COMPLETED"). `assertTrue(order.items().size() == 1)` → `assertThat(order.items()).hasSize(1)` (실패 시: "expected size: 1, but was: 2 in: [Item1, Item2]").
>
> **Q2.** 적절하다. 판단 기준 충족 여부: ① 3개 이상의 테스트에서 반복 → ✅ "3개의 테스트 클래스"에서 반복. ② 패턴이 동일 → ✅ 동일한 3개 단언. ③ 도메인 의미가 있음 → ✅ "올바른 JSON API 응답" 이라는 개념. 커스텀 단언: `assertThat(response).isValidJsonResponse()` — 내부적으로 세 단언을 포함. 이름이 도메인 언어로 표현되어 테스트 의도가 더 명확해진다.
>
> **Q3.** 스타일 B(assertThatThrownBy)가 낫다. 이유: ① 스타일 A는 예외가 발생하지 않으면 `fail()`을 명시적으로 호출해야 하는데, 이를 빠뜨리면 테스트가 항상 통과한다(테스트 작성 실수). ② 스타일 B는 예외가 발생하지 않으면 자동으로 실패한다. ③ 스타일 B는 체이닝으로 예외 타입과 메시지를 한 표현에서 검증할 수 있어 가독성이 높다. ④ try-catch는 플로우 제어 코드이지 단언 코드가 아니다 — 테스트 안에 try-catch가 있으면 읽는 사람이 예외 처리 의도를 파악하는 데 더 많은 인지 부담이 생긴다.

---

<div align="center">

**[⬅️ 이전: Fixtures & SetUp](./05-fixtures-and-setup.md)**

</div>
