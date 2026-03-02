# 04. Parameterized Tests

> **같은 테스트 로직에 입력만 다른 케이스가 5개 이상이라면 — 복사-붙여넣기가 아니라 파라미터화가 답이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `@ValueSource`, `@CsvSource`, `@MethodSource`는 각각 언제 쓰는가?
- 파라미터화 테스트의 이름을 어떻게 의미 있게 만드는가?
- 파라미터화로 해결할 수 없는 경우는 어떤 경우인가?

---

## 🔍 복사-붙여넣기 테스트가 만드는 문제

```java
// ❌ 같은 구조, 다른 입력 — 복사-붙여넣기의 냄새
@Test void 할인율_0은_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(0))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@Test void 할인율_음수는_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(-1))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@Test void 할인율_101은_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(101))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@Test void 할인율_200은_유효하지_않다() {
    assertThatThrownBy(() -> new Coupon(200))
        .isInstanceOf(InvalidDiscountRateException.class);
}
```

새 케이스가 생길 때마다 테스트를 하나씩 추가해야 합니다. 검증 로직이 바뀌면 4개를 모두 수정해야 합니다. 테스트 코드의 중복은 프로덕션 코드의 중복만큼 유지보수 비용을 높입니다.

---

## 🏛️ `@ParameterizedTest` 소스별 선택 기준

### `@ValueSource` — 단일 타입의 단순 값 목록

```java
// ✅ 단순한 경우: 하나의 입력값만 변한다
@ParameterizedTest
@ValueSource(ints = {0, -1, -100, 101, 200, Integer.MAX_VALUE})
void 유효_범위_밖의_할인율은_예외가_발생한다(int invalidRate) {
    assertThatThrownBy(() -> new Coupon(invalidRate))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@ParameterizedTest
@ValueSource(strings = {"", " ", "  ", "\t", "\n"})
void 공백_이메일은_유효하지_않다(String blank) {
    assertThat(validator.isValidEmail(blank)).isFalse();
}
```

`@NullSource`, `@EmptySource`, `@NullAndEmptySource`도 함께 조합 가능합니다.

```java
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {" ", "\t"})
void 비어있거나_공백인_이메일은_유효하지_않다(String email) {
    assertThat(validator.isValidEmail(email)).isFalse();
}
```

---

### `@CsvSource` — 입력과 기댓값을 함께 전달

```java
// ✅ 입력 → 기댓값 쌍이 있을 때
@ParameterizedTest(name = "{0}등급 회원의 {1}원 주문 → 할인 후 {2}원")
@CsvSource({
    "VIP,    10000, 9000",
    "GOLD,   10000, 9500",
    "NORMAL, 10000, 10000",
    "VIP,    20000, 18000",
    "GOLD,   20000, 19000",
})
void 등급별_할인_금액이_올바르게_계산된다(String grade, int price, int expected) {
    int result = calculator.calculate(price, Grade.valueOf(grade));
    assertThat(result).isEqualTo(expected);
}
```

`@CsvSource`의 `name` 속성에 `{0}`, `{1}` 플레이스홀더를 쓰면 각 케이스의 이름이 의미 있게 표시됩니다.

```
✅ 실패 메시지:
VIP등급 회원의 10000원 주문 → 할인 후 9000원  FAILED

❌ 이름 없을 때:
등급별_할인_금액이_올바르게_계산된다(1)  FAILED  (어떤 케이스인지 모름)
```

---

### `@MethodSource` — 복잡한 객체나 긴 목록

```java
// ✅ 입력이 객체거나 목록이 길 때
@ParameterizedTest(name = "{0}")
@MethodSource("invalidOrderScenarios")
void 유효하지_않은_주문은_예외가_발생한다(String description, Cart cart, User user) {
    assertThatThrownBy(() -> orderService.place(cart, user))
        .isInstanceOf(InvalidOrderException.class);
}

// 테스트 클래스 안의 static 메서드
static Stream<Arguments> invalidOrderScenarios() {
    return Stream.of(
        Arguments.of("빈 장바구니",
            aCart().empty().build(),
            aUser().build()),
        Arguments.of("재고 없는 상품",
            aCart().withOutOfStockItem().build(),
            aUser().build()),
        Arguments.of("미성년자의 성인 상품 주문",
            aCart().withAdultItem().build(),
            aUser().withBirthDate(LocalDate.of(2010, 1, 1)).build())
    );
}
```

`@MethodSource`는 Test Data Builder와 자연스럽게 조합됩니다.

---

## 😱 파라미터화를 잘못 적용하는 패턴

### 패턴 1: 로직이 다른 케이스를 억지로 묶기

```java
// ❌ 입력과 기댓값뿐 아니라 검증 로직까지 다르다
@ParameterizedTest
@MethodSource("orderScenarios")
void 주문_처리_테스트(Cart cart, User user, Object expectedResult) {
    if (expectedResult instanceof Order) {
        Order result = orderService.place(cart, user);
        assertThat(result).isEqualTo(expectedResult);
    } else if (expectedResult instanceof Class) {
        assertThatThrownBy(() -> orderService.place(cart, user))
            .isInstanceOf((Class<?>) expectedResult);
    }
    // 테스트 안에 if-else → 파라미터화의 잘못된 사용
}
```

**Assert 로직이 다르면 파라미터화하지 않는다.** 정상 케이스와 예외 케이스는 별도 파라미터화 테스트로 분리합니다.

### 패턴 2: 너무 많은 조합을 한 번에

```java
// ❌ 27가지 조합 — 테스트가 실패했을 때 어떤 케이스인지 파악이 어렵다
@ParameterizedTest
@MethodSource("allCombinations")  // grade × coupon × delivery = 27가지
void 모든_주문_조합_테스트(Grade grade, CouponType coupon, DeliveryType delivery) { ... }
```

조합 폭발이 발생할 때는 비즈니스적으로 중요한 조합만 선택하거나, 각 변수를 독립적으로 테스트합니다.

---

## ✨ 파라미터화 테스트 이름 전략

```java
// 기본: 인덱스만 표시 — 실패 시 무의미
@ParameterizedTest
void 테스트(int input) { ... }
// → 테스트[1], 테스트[2], 테스트[3]

// @CsvSource + name: 값이 직접 보임
@ParameterizedTest(name = "입력={0}, 기댓값={1}")
@CsvSource({"10, 9", "20, 18"})
void 테스트(int input, int expected) { ... }
// → 테스트[입력=10, 기댓값=9], 테스트[입력=20, 기댓값=18]

// @MethodSource + Arguments.of(description, ...): 설명 제공
@ParameterizedTest(name = "{0}")
@MethodSource("scenarios")
void 테스트(String description, int input) { ... }
static Stream<Arguments> scenarios() {
    return Stream.of(
        Arguments.of("VIP 최소 주문", 10_000),
        Arguments.of("GOLD 최소 주문", 5_000)
    );
}
// → 테스트[VIP 최소 주문], 테스트[GOLD 최소 주문]
```

---

## 💻 실전 적용: 경계값 테스트와 결합

이전 챕터의 경계값과 파라미터화는 자연스럽게 결합됩니다.

```java
@ParameterizedTest(name = "할인율 {0}%는 유효하지 않다")
@ValueSource(ints = {Integer.MIN_VALUE, -1, 0, 101, Integer.MAX_VALUE})
void 유효_범위_밖_할인율은_예외(int rate) {
    assertThatThrownBy(() -> new Coupon(rate))
        .isInstanceOf(InvalidDiscountRateException.class);
}

@ParameterizedTest(name = "할인율 {0}%는 유효하다")
@ValueSource(ints = {1, 10, 50, 99, 100})
void 유효_범위_내_할인율은_정상_생성(int rate) {
    assertThatCode(() -> new Coupon(rate)).doesNotThrowAnyException();
}
```

이 두 테스트는 경계값(`0`, `1`, `100`, `101`)을 포함하면서 추가 케이스도 자연스럽게 담습니다.

---

## 🤔 트레이드오프

### "파라미터화 테스트가 실패하면 어떤 케이스가 실패했는지 바로 알 수 있는가?"

`name` 속성을 잘 설정했다면 알 수 있습니다. 하지만 케이스가 많을수록 실패한 케이스를 재현하기 위해 특정 파라미터만 단독 실행하는 기능이 필요합니다. IntelliJ에서는 실패한 파라미터 케이스를 클릭해서 단독 실행할 수 있습니다.

### "파라미터 소스가 외부 파일이 되면 테스트 가독성이 떨어지는가?"

`@CsvFileSource`로 CSV 파일을 읽는 방식은 케이스가 수십 개일 때 유용하지만, 테스트 코드만 읽어서는 어떤 케이스가 있는지 알 수 없습니다. 일반적으로 테스트 파일 안에 `@CsvSource` 또는 `@MethodSource`로 케이스를 명시하는 것이 더 읽기 좋습니다.

---

## 📌 핵심 정리

```
소스별 선택 기준:
  @ValueSource      단일 타입 단순 값 목록
  @NullAndEmptySource  null, ""를 함께
  @CsvSource        입력 + 기댓값 쌍, 원시타입
  @MethodSource     복잡한 객체, 긴 목록, Test Data Builder 조합

이름 필수:
  @ParameterizedTest(name = "...")
  {0}, {1}: 파라미터 값
  Arguments.of(description, ...): 설명 직접 제공

파라미터화 금지 케이스:
  Assert 로직이 케이스마다 다를 때
  조합이 폭발적으로 많을 때 (27가지 이상 → 분리)

경계값 + 파라미터화:
  자연스러운 조합
  유효/무효 구역을 각각 @ParameterizedTest로 분리
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 4개 테스트를 적절한 `@ParameterizedTest`로 리팩터링하라. 어떤 소스 타입을 선택할지, 이름은 어떻게 설정할지 포함해서 작성하라.

```java
@Test void 비밀번호_7자는_너무_짧다() {
    assertThat(validator.validate("abc1234")).isFalse();
}
@Test void 비밀번호_8자는_유효하다() {
    assertThat(validator.validate("abc12345")).isTrue();
}
@Test void 비밀번호_20자는_유효하다() {
    assertThat(validator.validate("abcdefghij1234567890")).isTrue();
}
@Test void 비밀번호_21자는_너무_길다() {
    assertThat(validator.validate("abcdefghijk12345678901")).isFalse();
}
```

**Q2.** `@MethodSource`를 사용할 때 소스 메서드를 같은 클래스에 두는 것과 별도 클래스에 두는 것의 장단점은 무엇인가?

**Q3.** 아래 파라미터화 테스트에서 케이스 3이 실패했을 때 어떤 정보가 출력되는가? 더 나은 실패 메시지를 위해 어떻게 수정하겠는가?

```java
@ParameterizedTest
@CsvSource({
    "VIP, 10000, 9000",
    "GOLD, 10000, 9500",
    "NORMAL, 10000, 11000",   // 버그: 10000이어야 함
})
void 등급별_할인_계산(String grade, int price, int expected) {
    assertThat(calculator.calculate(price, grade)).isEqualTo(expected);
}
```

> 💡 **해설**
>
> **Q1.** 두 그룹으로 분리한다(유효/무효). `@ParameterizedTest(name = "비밀번호 \"{0}\"은 유효하지 않다") @ValueSource(strings = {"abc1234", "abcdefghijk12345678901"}) void 유효하지_않은_비밀번호(String pw)` + `@ParameterizedTest(name = "비밀번호 길이 {0}자는 유효하다") @ValueSource(strings = {"abc12345", "abcdefghij1234567890"}) void 유효한_비밀번호(String pw)`. 소스 타입: 단일 String 값이므로 `@ValueSource`. 이름: 값이 직접 보이도록 `{0}` 활용.
>
> **Q2.** 같은 클래스: 테스트와 소스가 가까워 이해하기 쉽다. 단점: 클래스가 커질 수 있다. 별도 클래스: 여러 테스트 클래스에서 공유할 수 있다(`@MethodSource("com.example.OrderScenarios#invalidCases")`). 단점: 테스트와 소스가 분리되어 탐색이 필요하다. 원칙: 한 테스트 클래스에서만 쓰는 소스는 같은 클래스 내 `static` 메서드로, 여러 곳에서 공유하는 픽스처는 별도 클래스로 분리한다.
>
> **Q3.** 현재 출력: `등급별_할인_계산(3)  FAILED` — 세 번째 케이스라는 것만 알 수 있고 어떤 값인지 모른다. 수정: `@ParameterizedTest(name = "{0}등급 {1}원 → 기댓값 {2}원")` 추가. 수정 후 출력: `등급별_할인_계산[NORMAL등급 10000원 → 기댓값 11000원]  FAILED` — 어떤 입력과 기댓값으로 실패했는지 즉시 알 수 있다.

---

<div align="center">

**[⬅️ 이전: Boundary Testing](./03-boundary-testing.md)** | **[다음: Fixtures & SetUp ➡️](./05-fixtures-and-setup.md)**

</div>
