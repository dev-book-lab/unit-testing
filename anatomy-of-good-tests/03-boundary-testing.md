# 03. Boundary Testing

> **버그는 경계에서 태어난다 — "정상 케이스"만 테스트하는 팀이 놓치는 것은 항상 Off-by-One이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 경계값 분석(BVA)이란 무엇이고, 경계값을 어떻게 체계적으로 찾는가?
- Off-by-One 에러가 발생하는 구조적 이유는 무엇인가?
- `>`, `>=`, `<`, `<=` 중 어느 것이 맞는지 테스트가 어떻게 강제하는가?

---

## 🔍 왜 경계가 위험한가

```java
public class AgeVerifier {
    public boolean isAdult(int age) {
        return age > 18;  // ← 버그: 18세는 성인이 아닌가?
    }
}
```

```java
// 이 테스트로는 버그를 잡을 수 없다
@Test void 성인이면_true를_반환한다() {
    assertThat(verifier.isAdult(30)).isTrue();   // 통과
}

@Test void 미성년자면_false를_반환한다() {
    assertThat(verifier.isAdult(15)).isFalse();  // 통과
}
```

두 테스트 모두 통과합니다. 하지만 `isAdult(18)`은 `false`를 반환합니다. `>` 와 `>=` 중 하나가 틀렸는데, 테스트는 이 차이를 전혀 검증하지 못했습니다.

경계값인 `18`이 "포함인가, 미포함인가"를 명시적으로 테스트하지 않았기 때문입니다.

---

## 🏛️ 경계값 분석(BVA): 체계적으로 경계 찾기

경계값 분석은 입력 도메인을 **동등 분할 구역**으로 나누고, 각 구역의 경계에서 테스트하는 기법입니다.

```
isAdult(age)의 입력 도메인:

구역 1: 미성년자      구역 2: 성인
────────────────────|──────────────────
...  15  16  17  [18]  19  20  30 ...

경계: 18

테스트해야 할 값:
  경계 - 1 = 17  (미성년자 구역의 최댓값)
  경계     = 18  (경계값 자체, 가장 위험)
  경계 + 1 = 19  (성인 구역의 최솟값)
```

```java
// ✅ 경계값 3개를 모두 테스트한다
@Test void 경계_미만_17세는_미성년자다() {
    assertThat(verifier.isAdult(17)).isFalse();
}

@Test void 경계값_18세는_성인이다() {
    assertThat(verifier.isAdult(18)).isTrue();  // ← 이 테스트가 >18 버그를 잡는다
}

@Test void 경계_초과_19세는_성인이다() {
    assertThat(verifier.isAdult(19)).isTrue();
}
```

---

## 😱 Off-by-One: 가장 흔한 경계 버그

Off-by-One은 경계 조건에서 1을 빼거나 더하는 실수입니다. 원인은 자연어와 코드의 표현 방식 차이에서 옵니다.

```
요구사항: "최소 주문 금액은 1,000원 이상이다"

개발자 A의 해석:
  amount >= 1_000  (올바름)

개발자 B의 해석:
  amount > 1_000   (1,000원이 거절됨 — Off-by-One)

개발자 C의 해석:
  amount > 999     (올바름이지만 의도가 불명확)
```

```java
// ❌ 이 테스트로는 A, B, C 중 어느 것이든 통과한다
@Test void 충분한_금액이면_주문이_된다() {
    assertThat(orderService.canPlace(5_000)).isTrue();
}

@Test void 금액이_부족하면_주문이_안된다() {
    assertThat(orderService.canPlace(500)).isFalse();
}

// ✅ 경계값 테스트가 있어야 A/B/C를 구분할 수 있다
@Test void 최소주문금액_999원은_주문이_불가하다() {
    assertThat(orderService.canPlace(999)).isFalse();
}

@Test void 최소주문금액_1000원은_주문이_가능하다() {
    assertThat(orderService.canPlace(1_000)).isTrue();   // B는 여기서 실패
}

@Test void 최소주문금액_1001원은_주문이_가능하다() {
    assertThat(orderService.canPlace(1_001)).isTrue();
}
```

---

## ✨ 경계값을 체계적으로 찾는 체크리스트

### 숫자 범위

```java
// 요구사항: "쿠폰 할인율은 1~100 사이다"
// 찾아야 할 경계값: 0, 1, 100, 101

@Test void 할인율_0은_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(0)).isInstanceOf(InvalidDiscountRateException.class);
}
@Test void 할인율_1은_유효하다() {
    assertThatCode(() -> new Coupon(1)).doesNotThrowAnyException();
}
@Test void 할인율_100은_유효하다() {
    assertThatCode(() -> new Coupon(100)).doesNotThrowAnyException();
}
@Test void 할인율_101은_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(101)).isInstanceOf(InvalidDiscountRateException.class);
}
```

### 컬렉션 크기

```java
// 요구사항: "장바구니에 최대 10개 상품까지 담을 수 있다"
// 경계값: 0개, 1개, 9개, 10개, 11개

@Test void 빈_장바구니는_주문할_수_없다() {
    Cart emptyCart = new Cart();
    assertThatThrownBy(() -> orderService.place(emptyCart, user))
        .isInstanceOf(EmptyCartException.class);
}

@Test void 아이템_1개는_주문_가능하다() {
    Cart cart = aCart().withItemCount(1).build();
    assertThatCode(() -> orderService.place(cart, user)).doesNotThrowAnyException();
}

@Test void 아이템_10개는_주문_가능하다() {
    Cart cart = aCart().withItemCount(10).build();
    assertThatCode(() -> orderService.place(cart, user)).doesNotThrowAnyException();
}

@Test void 아이템_11개는_주문할_수_없다() {
    Cart cart = aCart().withItemCount(11).build();
    assertThatThrownBy(() -> orderService.place(cart, user))
        .isInstanceOf(CartLimitExceededException.class);
}
```

### 문자열 길이

```java
// 요구사항: "비밀번호는 8~20자다"
// 경계값: 7자, 8자, 20자, 21자

@Test void 비밀번호_7자는_유효하지_않다() { ... }
@Test void 비밀번호_8자는_유효하다() { ... }
@Test void 비밀번호_20자는_유효하다() { ... }
@Test void 비밀번호_21자는_유효하지_않다() { ... }
```

### 날짜/시간 경계

```java
// 요구사항: "당일 자정 이전 주문만 익일 배송 처리한다"
// 경계값: 자정 1초 전, 자정, 자정 1초 후

@Test void 자정_1초_전_주문은_익일배송이다() {
    Clock clock = fixedClock("2024-01-15T23:59:59");
    assertThat(deliveryService.isNextDay(order, clock)).isTrue();
}

@Test void 정확히_자정에_주문하면_익일배송이_아니다() {
    Clock clock = fixedClock("2024-01-16T00:00:00");
    assertThat(deliveryService.isNextDay(order, clock)).isFalse();
}
```

---

## 💻 실전 적용: null과 빈 값도 경계다

```java
// String 입력의 경계값: null, "", " ", 정상값, 최대 길이 초과

@ParameterizedTest
@NullAndEmptySource           // null과 "" 자동 제공
@ValueSource(strings = {" "}) // 공백도 경계
void 이메일이_비어있으면_유효하지_않다(String email) {
    assertThat(validator.isValid(email)).isFalse();
}

@Test void 최대_길이_이메일은_유효하다() {
    String maxLength = "a".repeat(50) + "@example.com";  // 정확히 62자
    assertThat(validator.isValid(maxLength)).isTrue();
}

@Test void 최대_길이_초과_이메일은_유효하지_않다() {
    String tooLong = "a".repeat(51) + "@example.com";    // 63자
    assertThat(validator.isValid(tooLong)).isFalse();
}
```

### 정수 오버플로우 경계

```java
// 금액 계산에서 Integer.MAX_VALUE 근처 동작 검증
@Test void 최대값_근처에서_금액_합산이_오버플로우되지_않는다() {
    Order order1 = anOrder().withPrice(Integer.MAX_VALUE - 100).build();
    Order order2 = anOrder().withPrice(200).build();

    // long으로 계산해야 오버플로우 없음
    assertThatCode(() -> summaryService.total(order1, order2))
        .doesNotThrowAnyException();
}
```

---

## 🤔 트레이드오프

### "경계값마다 테스트를 분리하면 테스트가 너무 많아지지 않는가?"

경계값 테스트는 `@ParameterizedTest`와 잘 어울립니다. 경계값들이 같은 구역에 속하면 하나의 파라미터화 테스트로 묶을 수 있습니다.

```java
@ParameterizedTest
@ValueSource(ints = {0, -1, Integer.MIN_VALUE})
void 유효하지_않은_할인율(int rate) {
    assertThatThrownBy(() -> new Coupon(rate))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@ParameterizedTest
@ValueSource(ints = {1, 50, 100})
void 유효한_할인율(int rate) {
    assertThatCode(() -> new Coupon(rate)).doesNotThrowAnyException();
}
```

### "요구사항이 모호할 때 경계값을 어떻게 결정하는가?"

"최소 주문 금액"이라는 요구사항이 있을 때 "이상"인지 "초과"인지 불분명하다면, 경계값 테스트를 먼저 작성하는 것 자체가 요구사항을 명확하게 만드는 활동입니다. 테스트를 쓰면서 "1,000원 이상인가, 초과인가?"를 질문하게 되고, 그 답이 코드에 명시됩니다.

---

## 📌 핵심 정리

```
경계값 분석(BVA):
  입력 도메인을 동등 구역으로 분리
  각 구역 경계에서 3개 테스트: 경계-1, 경계, 경계+1

Off-by-One 방지:
  >  vs >= 를 구분하는 것은 경계값 테스트뿐이다
  자연어 "이상/초과", "이하/미만"을 코드로 명확히 표현

경계값 체크리스트:
  숫자 범위: 최솟값-1, 최솟값, 최댓값, 최댓값+1
  컬렉션: 0개, 1개, 최대-1개, 최대개, 최대+1개
  문자열: null, "", " ", 최대 길이, 최대+1 길이
  날짜/시간: 기준 시각 -1초, 정각, +1초

"정상 케이스만 테스트하는 팀이 놓치는 것":
  항상 경계에서 발생하는 >, >= 구분 버그
  null/빈 문자열 처리
  컬렉션의 빈 상태
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 메서드에서 테스트해야 할 경계값을 모두 찾고, 각각에 대한 테스트를 작성하라.

```java
// 요구사항: 적립 포인트는 구매 금액의 1%이며,
//           최소 10포인트, 최대 5,000포인트가 적립된다.
public int calculatePoints(int purchaseAmount) {
    int points = purchaseAmount / 100;
    return Math.max(10, Math.min(5_000, points));
}
```

**Q2.** 문자열 유효성 검사에서 `null`과 `""`(빈 문자열)을 각각 다른 예외로 처리하는 정책이 있다. 이 두 경계를 어떻게 테스트하겠는가? `@NullAndEmptySource`를 쓰면 충분한가?

**Q3.** 날짜 관련 비즈니스 로직("오늘 자정까지 주문하면 내일 배송")을 테스트할 때, `LocalDate.now()`에 의존하는 코드를 어떻게 경계값 테스트하겠는가? 테스트 가능한 설계를 포함해서 설명하라.

> 💡 **해설**
>
> **Q1.** 경계값 분석: ① `purchaseAmount`가 작아서 `points < 10`인 구역 — 경계: `purchaseAmount = 999`(9포인트), `purchaseAmount = 1_000`(10포인트). ② 정상 범위: `1_000 <= amount <= 500_000`. ③ `points > 5_000`이 되는 구역 — 경계: `purchaseAmount = 500_000`(5,000포인트), `purchaseAmount = 500_001`(5,001포인트 → 5,000 캡). 테스트: `구매금액_999원은_최소_10포인트가_적립된다()`, `구매금액_1000원은_10포인트가_적립된다()`, `구매금액_500000원은_5000포인트가_적립된다()`, `구매금액_500001원은_5000포인트로_상한_적용된다()`.
>
> **Q2.** `@NullAndEmptySource`는 충분하지 않다. 이 애너테이션은 null과 ""를 하나의 파라미터 소스로 묶지만, 서로 다른 예외를 발생시켜야 한다면 분리해서 테스트해야 한다: `void null_입력은_NullArgumentException을_발생시킨다()` — `assertThatThrownBy(() -> validator.validate(null)).isInstanceOf(NullArgumentException.class)` / `void 빈_문자열은_EmptyValueException을_발생시킨다()` — `assertThatThrownBy(() -> validator.validate("")).isInstanceOf(EmptyValueException.class)`. 같은 예외라면 `@NullAndEmptySource`로 묶는 것이 적절하다.
>
> **Q3.** `LocalDate.now()`에 직접 의존하면 경계값 테스트가 불가능하다(자정에만 실행해야 함). 설계: `deliveryService.isNextDay(order, Clock clock)`처럼 `Clock`을 주입받도록 변경한다. 테스트: `Clock.fixed(자정_1초_전)`과 `Clock.fixed(자정_0초)`를 주입해서 각각 true/false를 검증한다. 프로덕션 코드에서는 `Clock.systemDefaultZone()`을 주입한다. 이것이 `FIRST - Repeatable` 원칙과도 연결된다.

---

<div align="center">

**[⬅️ 이전: Test Data Builders](./02-test-data-builders.md)** | **[다음: Parameterized Tests ➡️](./04-parameterized-tests.md)**

</div>
