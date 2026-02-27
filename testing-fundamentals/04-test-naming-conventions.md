# 04. Test Naming Conventions

> **테스트 이름은 문서다 — 코드를 보지 않고 이름만 읽어도 무슨 동작이 검증되는지 알 수 있어야 한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 테스트 이름에 구현 세부사항이 들어가면 왜 문제가 되는가?
- "동작 중심 이름"이란 무엇이며, 어떻게 만드는가?
- 테스트 실패 메시지만 보고 원인을 파악하려면 이름을 어떻게 써야 하는가?

---

## 🔍 왜 이름이 중요한가

CI가 빨간불이 됐을 때 팀원이 처음 보는 것은 코드가 아닙니다.

```
FAILED tests:

❌ OrderServiceTest > test1
❌ OrderServiceTest > testOrderPlacement
❌ OrderServiceTest > place_WhenCalledWithValidInput_ShouldReturnOrder
```

세 실패 이름을 보고 무엇이 잘못됐는지 파악할 수 있습니까?  
`test1`은 아무 정보도 없습니다. `testOrderPlacement`는 "주문 생성을 테스트한다"는 것만 압니다. 세 번째는 그나마 낫지만 "어떤 상황에서 어떻게 잘못됐는가"는 여전히 모릅니다.

좋은 테스트 이름은 **실패했을 때 코드를 열어보지 않아도 무슨 일이 벌어졌는지** 알 수 있어야 합니다.

---

## 😱 나쁜 이름 패턴과 그 이유

### 패턴 1: 메서드 이름 그대로

```java
@Test void testPlace() { ... }
@Test void testCalculate() { ... }
@Test void testSave() { ... }
```

"테스트한다"는 것 외에 아무 정보가 없습니다. `place()`는 어떤 상황에서, 어떤 결과를 내야 통과하는 테스트인가요?

### 패턴 2: 구현 세부사항이 이름에 들어간다

```java
@Test void place_callsDiscountPolicyAndSavesToRepo() { ... }
@Test void calculate_invokesRateMultiplier() { ... }
```

이 이름은 "어떻게 구현됐는가"를 설명합니다. `DiscountPolicy`를 호출하지 않는 방식으로 리팩터링하면 이름 자체가 거짓말이 됩니다. 테스트 이름은 구현이 아닌 **동작**을 설명해야 합니다.

### 패턴 3: 조건이 없는 기대 결과

```java
@Test void place_returnsOrder() { ... }
@Test void calculate_returnsDiscount() { ... }
```

항상 주문을 반환하는가? 어떤 조건에서 어떤 주문을 반환하는가? 컨텍스트가 없습니다.

### 패턴 4: Should / Must 접두사

```java
@Test void should_applyDiscount_when_userIsVip() { ... }
@Test void should_throwException_when_stockIsZero() { ... }
```

`should`는 불필요한 소음입니다. "테스트는 당연히 어떤 동작이 일어나야 한다는 것을 확인한다"는 것을 이름에서 다시 말할 필요가 없습니다.

---

## ✨ 좋은 이름 패턴

### 핵심 공식: [상황] + [동작/결과]

```java
// 상황(when/given) + 결과(then/expected)
@Test void VIP_회원_주문_시_10퍼센트_할인이_적용된다() { ... }
@Test void 재고가_0이면_주문_생성이_실패한다() { ... }
@Test void 만료된_쿠폰으로는_할인을_받을_수_없다() { ... }
@Test void 동일한_이메일로_두_번_가입하면_중복_예외가_발생한다() { ... }
```

이름만 읽어도 "어떤 상황에서, 무엇이 일어나야 하는가"를 알 수 있습니다.

### 영어를 써야 한다면: 언더스코어로 단어 구분

```java
// 언더스코어가 가독성을 크게 높인다
@Test void placeOrder_whenUserIsVip_applies10PercentDiscount() { ... }
@Test void placeOrder_whenStockIsZero_throwsOutOfStockException() { ... }
@Test void register_whenEmailDuplicated_throwsDuplicateEmailException() { ... }

// 카멜케이스 (읽기 어렵다)
@Test void placeOrderWhenUserIsVipApplies10PercentDiscount() { ... }
```

### 중첩 클래스로 컨텍스트 분리 (@Nested)

테스트가 많아지면 이름에 컨텍스트를 매번 넣는 게 중복됩니다. `@Nested`로 컨텍스트를 클래스 이름에 올리면 각 테스트 이름이 짧아집니다.

```java
class OrderServiceTest {

    @Nested
    @DisplayName("VIP 회원 주문")
    class WhenUserIsVip {

        @Test
        @DisplayName("10% 할인이 적용된다")
        void applies10PercentDiscount() { ... }

        @Test
        @DisplayName("무료 배송이 적용된다")
        void appliesFreeShipping() { ... }
    }

    @Nested
    @DisplayName("재고 없는 상품 주문")
    class WhenStockIsZero {

        @Test
        @DisplayName("OutOfStockException이 발생한다")
        void throwsOutOfStockException() { ... }
    }
}
```

실패 메시지:
```
OrderServiceTest > VIP 회원 주문 > 10% 할인이 적용된다  FAILED
```

컨텍스트와 결과가 계층적으로 표현됩니다.

---

## 💻 실전 적용: 이름 리팩터링 연습

아래 이름들을 리팩터링합니다.

```java
// Before → After

@Test void testDiscount()
// → 어떤 상황의 할인? 무엇이 기대 결과?
// → 정회원_쿠폰_적용_시_5퍼센트_추가_할인된다()

@Test void saveUser_works()
// → "작동한다"는 기준이 무엇?
// → 유효한_사용자_정보로_가입하면_DB에_저장된다()

@Test void calculateTax_negative()
// → 음수 세금? 음수 입력?
// → 가격이_음수이면_세금_계산_시_예외가_발생한다()

@Test void checkInventory_returnsTrue()
// → 어떤 상황에서 true를 반환?
// → 재고가_1개_이상_남아있으면_주문_가능_상태를_반환한다()
```

---

## 🤔 트레이드오프

### 한국어 vs 영어 이름

한국어 테스트 이름은 도메인 개념을 그대로 표현할 수 있어서 비개발자(기획자, QA)도 읽을 수 있습니다. 다만 IDE나 빌드 툴에 따라 이름에 특수문자가 들어갈 경우 렌더링 이슈가 생길 수 있습니다.

영어 이름은 팀이 국제화되어 있거나, 영어 코드베이스 일관성을 유지해야 할 때 적합합니다. 언더스코어 구분(`place_whenVip_applies10PercentDiscount`)이 가독성을 크게 높입니다.

**실무 기준:** 팀 내에서 한 가지로 통일하는 것이 중요합니다. 절반은 한국어, 절반은 영어인 것이 가장 나쁩니다.

### `@DisplayName`을 써야 하는가?

`@DisplayName`은 이름과 분리된 "설명"을 추가할 수 있지만, 메서드 이름이 이미 명확하다면 중복입니다. `@Nested` 클래스에서 계층적 설명을 위해 쓰거나, 한국어 표현이 메서드 이름으로는 제약이 있을 때 사용합니다.

---

## 📌 핵심 정리

```
테스트 이름 = 문서

좋은 이름의 공식:
  [상황] + [동작/결과]
  "VIP 회원 주문 시 10% 할인이 적용된다"

피해야 할 패턴:
  test + 메서드명 → 아무 정보 없음
  구현 세부사항 → 리팩터링 시 이름이 거짓말 됨
  should / must 접두사 → 불필요한 소음
  결과만 있고 조건 없음 → 언제 이 결과가 나오는가?

@Nested 활용:
  컨텍스트 → 중첩 클래스 이름
  구체 행동 → 메서드 이름
  실패 메시지가 계층적으로 표시됨

기준:
  CI가 빨간불일 때 이름만 보고 원인 추정 가능한가?
  → 가능하면 좋은 이름
  → 코드를 열어봐야 한다면 이름을 수정하라
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트들의 이름을 리팩터링하라. 각 이름에서 무엇이 빠져있는지 먼저 분석하고, 더 나은 이름을 제안하라.

```java
@Test void testPasswordValidation() { ... }
@Test void save_returnsSavedEntity() { ... }
@Test void getUser_throwsException() { ... }
@Test void order_shouldBeCreated_whenItemsAreAdded() { ... }
```

**Q2.** 팀원이 "테스트 이름이 길어지면 가독성이 오히려 떨어진다. 짧게 유지해야 한다"고 주장한다. 이 주장의 타당한 부분과 잘못된 부분은 무엇인가?

**Q3.** 아래 `@Nested` 구조가 있을 때, 테스트 실패 시 출력되는 메시지는 어떤 형태인가? 그리고 이 구조에서 더 개선할 수 있는 점은 무엇인가?

```java
class UserServiceTest {
    @Nested class 이메일_검증 {
        @Test void 빈문자열_실패() { ... }
        @Test void 골뱅이_없음_실패() { ... }
        @Test void 정상이메일_성공() { ... }
    }
}
```

> 💡 **해설**
>
> **Q1.** `testPasswordValidation()` → 어떤 비밀번호, 어떤 결과인지 없음 → `비밀번호가_8자_미만이면_유효성_검증이_실패한다()` / `비밀번호에_특수문자가_없으면_유효성_검증이_실패한다()` (케이스를 분리해야 함). `save_returnsSavedEntity()` → "저장하면 저장된 엔티티를 반환한다"는 당연한 것 → `새_사용자를_저장하면_ID가_부여된_엔티티를_반환한다()`. `getUser_throwsException()` → 어떤 상황에서 예외? → `존재하지_않는_ID로_조회하면_UserNotFoundException이_발생한다()`. `order_shouldBeCreated_whenItemsAreAdded()` → should 제거, 조건과 결과 명확히 → `아이템을_추가하면_주문이_생성된다()`.
>
> **Q2.** 타당한 부분: 진짜로 과도하게 긴 이름은(`placeOrderFromCartWhenUserIsVipMemberWithActiveSubscriptionAndCouponCode()`) 오히려 핵심을 잃는다. 잘못된 부분: "짧다 = 좋다"는 잘못된 기준이다. 이름은 "그 자체로 문서"가 되어야 한다. `@Nested`로 컨텍스트를 계층화하면 각 메서드 이름은 짧아지면서도 계층 전체가 긴 설명이 된다. 기준은 길이가 아니라 "이름만 보고 동작을 이해할 수 있는가"이다.
>
> **Q3.** 실패 메시지: `UserServiceTest > 이메일_검증 > 빈문자열_실패 FAILED`. 개선할 점: 메서드 이름이 여전히 결과(성공/실패)만 말하고 왜 그런 결과인지 설명이 없다. `빈_이메일은_유효하지_않다()`, `골뱅이_기호가_없는_이메일은_형식_오류를_반환한다()`, `올바른_형식의_이메일은_통과한다()`처럼 동작 중심으로 수정하는 것이 더 낫다.

---

<div align="center">

**[⬅️ 이전: AAA Pattern](./03-aaa-pattern.md)** | **[다음: Coverage Myths ➡️](./05-coverage-myths.md)**

</div>
