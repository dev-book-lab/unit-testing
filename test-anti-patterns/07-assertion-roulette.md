# 07. Assertion Roulette

> **테스트가 실패했는데 어떤 단언이 틀렸는지 모른다 — 디버깅이 테스트보다 오래 걸린다면 테스트가 제 역할을 못하는 것이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Assertion Roulette이란 무엇이고 왜 발생하는가?
- 실패 메시지를 명확하게 만드는 방법은 무엇인가?
- `SoftAssertions`는 언제 사용하는가?

---

## 🔍 Assertion Roulette이란

Assertion Roulette은 테스트가 실패했을 때 어떤 단언이 틀렸는지, 왜 실패했는지 즉시 파악하기 어려운 상태입니다.

```java
// ❌ 실패 시 메시지: "expected: 18000 but was: 20000"
// → 어떤 주문의, 어떤 필드가, 왜 틀렸는지 알 수 없다
@Test
void 주문_전체_검증() {
    Order order = orderService.place(aCommand().build());

    assertThat(order.id()).isNotNull();
    assertThat(order.userId()).isEqualTo(1L);
    assertThat(order.totalPrice()).isEqualTo(18_000);  // 여기서 실패
    assertThat(order.status()).isEqualTo(PENDING);
    assertThat(order.createdAt()).isNotNull();
}
```

세 번째 단언에서 멈추기 때문에 `status`와 `createdAt`이 올바른지는 알 수 없습니다. 모든 문제를 한 번에 파악하려면 수정과 재실행을 반복해야 합니다.

---

## 😱 문제 1: 메시지 없는 단언

```java
// ❌ 실패 메시지만으로 문맥을 알 수 없다
assertThat(result).isTrue();
// 실패: "expected: true but was: false"
// → result가 무엇을 의미하는지 전혀 알 수 없음

assertThat(list.size()).isEqualTo(3);
// 실패: "expected: 3 but was: 1"
// → 어떤 리스트인지, 왜 3을 기대했는지 알 수 없음
```

---

## 😱 문제 2: 첫 번째 실패에서 멈추는 단언

```java
// ❌ totalPrice가 틀리면 status, createdAt은 검증 안 됨
assertThat(order.totalPrice()).isEqualTo(18_000);  // 실패 → 이후 실행 안 됨
assertThat(order.status()).isEqualTo(PENDING);
assertThat(order.createdAt()).isNotNull();
```

할인 계산 버그로 `totalPrice`가 틀렸을 때, `status`나 `createdAt`도 문제가 있을 수 있지만 알 수 없습니다.

---

## 😱 문제 3: 검증 대상이 불명확한 단언

```java
// ❌ 무엇을 검증하는지 코드만으로 알기 어렵다
assertThat(result.get(0).get("amount")).isEqualTo("20000");
assertThat(result.get(1).get("status")).isEqualTo("COMPLETED");
assertThat(result.size()).isGreaterThan(0);
```

---

## ✨ 해결 1: as()로 단언에 컨텍스트 추가

```java
// ✅ as()로 무엇을 검증하는지 명시
assertThat(order.totalPrice())
    .as("VIP 10% 할인 후 금액")
    .isEqualTo(18_000);

assertThat(isEligible)
    .as("VIP 등급 사용자는 할인 대상이어야 한다 (userId=%d, grade=%s)",
        user.id(), user.grade())
    .isTrue();

assertThat(orders)
    .as("사용자 %d의 PENDING 주문 목록", userId)
    .hasSize(2);
```

실패 시:

```
[VIP 10% 할인 후 금액]
expected: 18000
 but was: 20000
```

---

## ✨ 해결 2: SoftAssertions — 모든 단언을 한 번에 실행

`SoftAssertions`는 실패하는 단언이 있어도 멈추지 않고 모든 단언을 실행한 후, 전체 실패 목록을 한 번에 보고합니다.

```java
// ✅ SoftAssertions — 모든 실패를 한 번에 확인
@Test
void 주문_전체_검증() {
    Order order = orderService.place(aCommand().build());

    SoftAssertions softly = new SoftAssertions();
    softly.assertThat(order.id()).as("주문 ID").isNotNull();
    softly.assertThat(order.userId()).as("사용자 ID").isEqualTo(1L);
    softly.assertThat(order.totalPrice()).as("할인 후 금액").isEqualTo(18_000);
    softly.assertThat(order.status()).as("초기 상태").isEqualTo(PENDING);
    softly.assertThat(order.createdAt()).as("생성 시각").isNotNull();
    softly.assertAll(); // 여기서 모든 실패를 한 번에 보고
}
```

실패 시:

```
Multiple Failures (2 failures)
-- failure 1 --
[할인 후 금액] expected: 18000 but was: 20000
-- failure 2 --
[초기 상태] expected: PENDING but was: null
```

두 가지 문제를 동시에 파악합니다.

### assertSoftly — 람다 스타일

```java
// ✅ assertSoftly: assertAll() 호출 불필요
@Test
void 주문_전체_검증() {
    Order order = orderService.place(aCommand().build());

    assertSoftly(softly -> {
        softly.assertThat(order.id()).as("주문 ID").isNotNull();
        softly.assertThat(order.userId()).as("사용자 ID").isEqualTo(1L);
        softly.assertThat(order.totalPrice()).as("할인 후 금액").isEqualTo(18_000);
        softly.assertThat(order.status()).as("초기 상태").isEqualTo(PENDING);
        softly.assertThat(order.createdAt()).as("생성 시각").isNotNull();
    });
}
```

### @ExtendWith(SoftAssertionsExtension.class) — 주입 스타일

```java
// ✅ JUnit 5 Extension으로 SoftAssertions 자동 주입
@ExtendWith(SoftAssertionsExtension.class)
class OrderServiceTest {

    @InjectSoftAssertions
    SoftAssertions softly;

    @Test
    void 주문_전체_검증() {
        Order order = orderService.place(aCommand().build());

        softly.assertThat(order.id()).as("주문 ID").isNotNull();
        softly.assertThat(order.totalPrice()).as("할인 후 금액").isEqualTo(18_000);
        softly.assertThat(order.status()).as("초기 상태").isEqualTo(PENDING);
        // assertAll() 불필요 — Extension이 자동 처리
    }
}
```

---

## ✨ 해결 3: 커스텀 단언으로 의도 명확화

```java
// ❌ 무엇을 의미하는지 알기 어려운 단언
assertThat(order.status().code()).isEqualTo(2);
assertThat(order.totalPrice() > order.originalPrice()).isTrue();

// ✅ 의도를 드러내는 커스텀 단언 (05 문서에서 소개한 AbstractAssert)
assertThatOrder(order)
    .isInStatus(COMPLETED)
    .hasDiscountApplied();
```

```java
// ✅ extracting으로 컬렉션 검증 명확화
assertThat(orders)
    .extracting(Order::status)
    .as("모든 주문이 PENDING 상태여야 한다")
    .containsOnly(PENDING);

assertThat(orders)
    .extracting(Order::userId, Order::totalPrice)
    .as("사용자별 주문 금액")
    .containsExactlyInAnyOrder(
        tuple(1L, 18_000),
        tuple(2L, 50_000)
    );
```

---

## 💻 실전 적용: 단언 품질 체크리스트

```
단언을 작성할 때 확인한다:

① "이 단언이 실패했을 때 메시지만 보고 원인을 알 수 있는가?"
   → NO: as()로 컨텍스트 추가

② "이 테스트에 단언이 여러 개인데, 하나가 실패하면 나머지도 확인하고 싶은가?"
   → YES: SoftAssertions 사용

③ "같은 객체의 여러 필드를 검증하는가?"
   → YES: SoftAssertions 또는 커스텀 단언

④ "단언이 boolean isXxx() 메서드를 검증하는가?"
   → isTrue()/isFalse() 대신 조건을 명확히 드러내는 방식 검토

단언 당 검증:
  단순 조회/계산 테스트 → 단언 1~2개, 일반 assertThat
  객체 생성/변환 결과 → SoftAssertions 또는 커스텀 단언
  컬렉션 검증 → extracting + containsExactly 조합
```

---

## 🤔 SoftAssertions를 언제 쓰고 언제 쓰지 않는가

```
SoftAssertions 적합:
  객체의 여러 필드를 동시에 검증할 때
  하나의 실패로 다른 검증이 가려지면 디버깅이 어려울 때
  주문 생성 결과, API 응답 구조 검증

SoftAssertions 불필요:
  단언이 1~2개인 단순한 테스트
  앞 단언이 실패하면 뒷 단언이 의미 없을 때
    예: assertThat(result).isNotNull(); 이 실패하면
        assertThat(result.id())는 NullPointerException 발생
        → 이 경우엔 순차 실행이 맞다
```

---

## 🤔 트레이드오프

### "SoftAssertions를 항상 쓰면 되지 않는가?"

항상 쓰면 테스트 의도가 흐려집니다. 단언이 1개인 테스트에 `assertSoftly` 람다를 쓰면 코드가 불필요하게 복잡해집니다. 또한 앞선 단언의 실패가 뒷 단언의 전제 조건일 때 `SoftAssertions`를 쓰면 NPE 등 예상치 못한 오류가 발생할 수 있습니다. 여러 필드를 동시에 검증하는 경우에 선택적으로 사용합니다.

### "as()를 모든 단언에 붙여야 하는가?"

명확한 단언에는 불필요합니다. `assertThat(user.email()).isEqualTo("hong@example.com")`는 실패 메시지만으로 충분히 이해됩니다. 컨텍스트가 부족해서 "왜 이 값을 기대했는가"가 불명확할 때만 `as()`를 추가합니다.

---

## 📌 핵심 정리

```
Assertion Roulette의 증상:
  실패 메시지만으로 원인 파악 불가
  첫 번째 실패 후 나머지 검증 안 됨
  어떤 조건을 검증하는지 불명확

as()로 컨텍스트 추가:
  assertThat(value).as("설명 %s", context).isEqualTo(expected)
  실패 시: [설명] expected X but was Y

SoftAssertions:
  모든 단언을 실행 후 전체 실패 목록 보고
  assertSoftly(softly -> { ... })
  @InjectSoftAssertions로 주입

적합한 케이스:
  객체 여러 필드 동시 검증
  API 응답 구조 검증

extracting으로 컬렉션 검증:
  .extracting(Order::status)
  .containsOnly(PENDING)
  .extracting(Order::userId, Order::price)
  .containsExactlyInAnyOrder(tuple(1L, 18_000))

단언 1개 → assertThat (단순)
여러 필드 검증 → SoftAssertions 또는 커스텀 단언
컬렉션 → extracting + containsExactly
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트를 `SoftAssertions`와 `as()`를 사용해서 실패 시 원인을 즉시 파악할 수 있도록 개선하라.

```java
@Test
void API_응답_검증() {
    OrderResponse response = orderApi.place(request);

    assertThat(response.id()).isNotNull();
    assertThat(response.status()).isEqualTo("PENDING");
    assertThat(response.totalPrice()).isEqualTo(18_000);
    assertThat(response.userId()).isEqualTo(1L);
    assertThat(response.items()).hasSize(2);
    assertThat(response.items().get(0).quantity()).isEqualTo(1);
    assertThat(response.createdAt()).isNotNull();
}
```

**Q2.** 아래 단언이 실패했을 때 메시지만으로 원인을 파악하기 어렵다. 어떻게 개선하는가?

```java
assertThat(discountApplied).isTrue();
assertThat(items.stream().allMatch(i -> i.price() > 0)).isTrue();
assertThat(result).isEqualTo(expected);
```

**Q3.** 아래 두 테스트 중 어느 쪽이 `SoftAssertions`가 더 필요한가? 이유를 설명하라.

```java
// 테스트 A
@Test
void 사용자_존재_확인() {
    User user = userService.findById(1L);
    assertThat(user).isNotNull();
    assertThat(user.name()).isEqualTo("홍길동");
}

// 테스트 B
@Test
void 주문_생성_결과_검증() {
    Order order = orderService.place(aCommand().build());
    assertThat(order.id()).isNotNull();
    assertThat(order.totalPrice()).isEqualTo(18_000);
    assertThat(order.status()).isEqualTo(PENDING);
    assertThat(order.items()).hasSize(2);
    assertThat(order.createdAt()).isNotNull();
}
```

> 💡 **해설**

**Q1.**

```java
@Test
void API_응답_검증() {
    OrderResponse response = orderApi.place(request);

    assertSoftly(softly -> {
        softly.assertThat(response.id())
            .as("주문 ID는 서버에서 생성되어야 한다").isNotNull();
        softly.assertThat(response.status())
            .as("신규 주문의 초기 상태").isEqualTo("PENDING");
        softly.assertThat(response.totalPrice())
            .as("VIP 10%% 할인 적용 후 금액 (원래 20,000원)").isEqualTo(18_000);
        softly.assertThat(response.userId())
            .as("요청한 사용자 ID").isEqualTo(1L);
        softly.assertThat(response.items())
            .as("주문 아이템 수").hasSize(2);
        softly.assertThat(response.items().get(0).quantity())
            .as("첫 번째 아이템 수량").isEqualTo(1);
        softly.assertThat(response.createdAt())
            .as("생성 시각은 null이 아니어야 한다").isNotNull();
    });
}
```

모든 필드에서 동시에 실패가 발생해도 한 번의 실행으로 전체 실패 목록을 확인할 수 있다.

**Q2.**

```java
// ① 컨텍스트가 없는 boolean
// 개선
assertThat(discountApplied)
    .as("VIP 회원(userId=%d)은 할인 대상이어야 한다", user.id())
    .isTrue();

// ② 스트림 결과 boolean
// 개선 — boolean 대신 컬렉션 직접 검증
assertThat(items)
    .as("모든 아이템의 가격은 0보다 커야 한다")
    .extracting(Item::price)
    .allMatch(price -> price > 0, "price > 0");
// 또는 실패한 아이템을 보여주는 방식
assertThat(items).as("양수 가격 아이템")
    .allSatisfy(item ->
        assertThat(item.price()).as("item[%s] 가격", item.name()).isGreaterThan(0)
    );

// ③ result == expected 비교
// 개선 — result와 expected가 무엇인지 명시
assertThat(calculatedTax)
    .as("10만원 상품의 부가세 (10%%)")
    .isEqualTo(expectedTax);
```

**Q3.**

테스트 B가 `SoftAssertions`가 더 필요하다.

테스트 A: `user.isNotNull()` 이 실패하면 `user.name()` 접근 시 NPE가 발생한다. 첫 번째 단언이 두 번째 단언의 전제 조건이다. 이 경우엔 순차 실행이 맞다. `SoftAssertions`를 쓰면 NPE라는 예상치 못한 오류가 발생한다.

테스트 B: `order.id()`, `order.totalPrice()`, `order.status()`, `order.items()`, `order.createdAt()`은 모두 독립적인 검증이다. `totalPrice`가 틀렸다고 해서 `status`나 `createdAt` 검증에 영향을 주지 않는다. `SoftAssertions`로 묶으면 한 번 실행에 모든 문제를 파악할 수 있어 디버깅 효율이 높아진다.

---

<div align="center">

**[⬅️ 이전: Hidden Test Dependencies](./06-hidden-test-dependencies.md)** | **[홈으로 🏠](../README.md)**

</div>
