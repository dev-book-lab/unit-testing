# 02. Property-Based Testing

> **"이 입력으로는 동작한다" 대신 "어떤 입력에서도 이 성질이 성립한다" — 경계를 직접 생각해내지 않아도 된다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 예제 기반 테스트와 속성 기반 테스트는 무엇이 다른가?
- jqwik으로 속성을 어떻게 표현하는가?
- 어떤 종류의 코드에 속성 기반 테스트가 가장 효과적인가?

---

## 🔍 예제 기반 테스트의 한계

```java
// 예제 기반 테스트: 우리가 생각해낸 경우만 검증
@Test
void 배송비_계산() {
    assertThat(calculator.calculate(1_000)).isEqualTo(3_000);   // 소형
    assertThat(calculator.calculate(5_000)).isEqualTo(3_000);   // 소형 경계
    assertThat(calculator.calculate(5_001)).isEqualTo(5_000);   // 중형
    assertThat(calculator.calculate(30_000)).isEqualTo(7_000);  // 대형
}
```

이 테스트는 우리가 생각해낸 4가지 예제만 검증합니다. `2_500`, `15_000`, `29_999` 같은 입력에서 어떻게 동작하는지는 알 수 없습니다.

속성 기반 테스트는 "어떤 특정 입력에 대한 결과"가 아니라 **"모든 유효한 입력에서 항상 성립하는 성질"** 을 표현합니다. 그리고 도구가 무작위 입력을 수백 개 생성해서 그 성질이 깨지는 경우를 찾습니다.

---

## 🏛️ jqwik 설정

```kotlin
// build.gradle.kts
testImplementation("net.jqwik:jqwik:1.8.1")
testImplementation("net.jqwik:jqwik-spring:0.14.0") // Spring 통합
```

```java
// jqwik은 JUnit Platform을 사용 — JUnit 5와 함께 동작
// @Property: 속성 기반 테스트 어노테이션
// @ForAll: 임의 값 생성 파라미터
// @Example: 특정 예제 테스트 (jqwik 방식의 @Test)
```

---

## 🏛️ 속성 표현 방법

### 속성 1: 반올림 없는 범위 기반 분류

```java
// "소형 무게(≤5000g)는 항상 3,000원"이라는 성질
@Property
void 소형_무게는_항상_3000원(@ForAll @IntRange(min = 1, max = 5_000) int weight) {
    ShippingFee fee = calculator.calculate(weight);
    assertThat(fee.amount()).isEqualTo(3_000);
}

@Property
void 중형_무게는_항상_5000원(@ForAll @IntRange(min = 5_001, max = 30_000) int weight) {
    ShippingFee fee = calculator.calculate(weight);
    assertThat(fee.amount()).isEqualTo(5_000);
}
```

jqwik은 `1~5_000` 범위에서 무작위 정수를 100번(기본값) 생성해서 각각 검증합니다.

### 속성 2: 단조성 (monotonicity)

```java
// "무게가 클수록 배송비가 같거나 더 비싸다"
@Property
void 무게가_클수록_배송비가_비싸거나_같다(
        @ForAll @IntRange(min = 1, max = 50_000) int weight1,
        @ForAll @IntRange(min = 1, max = 50_000) int weight2) {

    Assume.that(weight1 <= weight2); // weight1 ≤ weight2인 경우만 검증

    int fee1 = calculator.calculate(weight1).amount();
    int fee2 = calculator.calculate(weight2).amount();

    assertThat(fee1).isLessThanOrEqualTo(fee2);
}
```

### 속성 3: 역방향 속성 (roundtrip)

```java
// "직렬화 후 역직렬화하면 원본과 같다"
@Property
void 직렬화_역직렬화_roundtrip(@ForAll Order order) {
    String json = objectMapper.writeValueAsString(order);
    Order deserialized = objectMapper.readValue(json, Order.class);

    assertThat(deserialized).isEqualTo(order);
}
```

### 속성 4: 불변식 (invariant)

```java
// "할인 후 금액은 원래 금액보다 크지 않다"
@Property
void 할인_후_금액은_원금보다_크지_않다(
        @ForAll @IntRange(min = 1_000, max = 1_000_000) int originalPrice,
        @ForAll @IntRange(min = 0, max = 100) int discountRate) {

    int discountedPrice = calculator.applyDiscount(originalPrice, discountRate);

    assertThat(discountedPrice).isLessThanOrEqualTo(originalPrice);
}

// "할인율 0%는 원금과 같다"
@Property
void 할인율_0은_원금과_같다(
        @ForAll @IntRange(min = 1_000, max = 1_000_000) int originalPrice) {

    int discountedPrice = calculator.applyDiscount(originalPrice, 0);

    assertThat(discountedPrice).isEqualTo(originalPrice);
}
```

---

## ✨ Shrinking — 실패한 최소 예제 자동 탐색

속성 기반 테스트의 강력한 기능 중 하나는 **Shrinking**입니다. 테스트가 실패하면 jqwik은 실패를 재현하는 가장 작은 입력을 자동으로 찾아줍니다.

```
실패 예시:
  입력 weight=23_847 에서 실패 발견
  
  Shrinking 시작:
    weight=11_923 → 여전히 실패
    weight=5_961  → 여전히 실패
    weight=2_980  → 통과 (이 범위는 OK)
    weight=5_001  → 실패
    → 최소 실패 입력: weight=5_001

  리포트:
  Property [중형_무게는_항상_5000원] failed with sample {weight=5_001}
```

`5_001`이 실패한다는 것은 경계값(`5_000 → 5_001`) 처리에 버그가 있다는 것을 명확히 보여줍니다.

---

## 💻 커스텀 Arbitrary — 도메인 객체 생성

jqwik의 기본 타입 생성기(`@ForAll String`, `@ForAll Integer`) 외에, 도메인 객체를 생성하는 커스텀 Arbitrary를 만들 수 있습니다.

```java
// 유효한 Order 객체를 생성하는 Arbitrary
@Provide
Arbitrary<Order> validOrders() {
    Arbitrary<Long> userIds = Arbitraries.longs().between(1L, 1_000_000L);
    Arbitrary<Integer> prices = Arbitraries.integers().between(1_000, 10_000_000);
    Arbitrary<OrderStatus> statuses = Arbitraries.of(OrderStatus.values());

    return Combinators.combine(userIds, prices, statuses)
        .as((userId, price, status) ->
            Order.builder()
                .userId(userId)
                .totalPrice(price)
                .status(status)
                .build()
        );
}

// 사용
@Property
void 주문_직렬화_역직렬화(@ForAll("validOrders") Order order) {
    String json = objectMapper.writeValueAsString(order);
    Order result = objectMapper.readValue(json, Order.class);
    assertThat(result).isEqualTo(order);
}
```

---

## 🏛️ 속성 기반 테스트가 잘 맞는 코드

```
적합한 경우:
  수학적 성질이 있는 계산 (단조성, 교환법칙, 결합법칙)
  직렬화/역직렬화 roundtrip
  인코딩/디코딩 (Base64, URL encoding)
  데이터 변환 파이프라인
  정렬, 중복 제거 같은 알고리즘
  금액 계산 (음수 안 됨, 할인 후 > 원금 안 됨)

덜 적합한 경우:
  특정 입력에 대한 구체적인 결과가 요구사항인 경우
    예: "사용자 ID 1번의 이름은 홍길동이어야 한다"
  외부 시스템 연동 (Mock으로 대체해야 함)
  UI/API 계약 검증 (특정 형식이 고정됨)
```

---

## 🤔 예제 기반 vs 속성 기반 — 함께 쓰기

두 방식은 대체 관계가 아니라 보완 관계입니다.

```java
// 예제 기반: 핵심 시나리오를 명확하게
@Test
void VIP_회원_10퍼센트_할인() {
    assertThat(calculator.applyDiscount(20_000, 10)).isEqualTo(18_000);
}

// 속성 기반: 모든 유효한 입력에서 성질 검증
@Property
void 할인된_금액은_원금보다_크지_않다(
        @ForAll @IntRange(min = 1_000, max = 1_000_000) int price,
        @ForAll @IntRange(min = 0, max = 100) int rate) {

    assertThat(calculator.applyDiscount(price, rate))
        .isLessThanOrEqualTo(price)
        .isGreaterThanOrEqualTo(0);
}

@Property
void 할인율이_같으면_원금이_클수록_할인금액이_크다(
        @ForAll @IntRange(min = 1_000, max = 500_000) int smallPrice,
        @ForAll @IntRange(min = 1_000, max = 500_000) int bigPrice,
        @ForAll @IntRange(min = 1, max = 100) int rate) {

    Assume.that(smallPrice < bigPrice);

    int discountSmall = smallPrice - calculator.applyDiscount(smallPrice, rate);
    int discountBig = bigPrice - calculator.applyDiscount(bigPrice, rate);

    assertThat(discountSmall).isLessThanOrEqualTo(discountBig);
}
```

---

## 🤔 트레이드오프

### "무작위 테스트라서 재현이 안 되는 거 아닌가?"

jqwik은 실패한 경우의 seed 값을 리포트에 포함합니다. 같은 seed로 재실행하면 동일한 입력이 생성됩니다.

```java
@Property(seed = "1234567890") // 특정 seed로 고정 실행 가능
void 재현_가능한_테스트(...) { ... }
```

### "실행 횟수가 많아서 느리지 않은가?"

기본 100회 실행입니다. 복잡한 연산이 아니라면 거의 느껴지지 않습니다. 필요하면 횟수를 조정합니다.

```java
@Property(tries = 500) // 500회 실행
@Property(tries = 10)  // 빠른 피드백이 필요할 때
```

---

## 📌 핵심 정리

```
예제 기반 테스트:
  특정 입력 → 특정 결과
  "이 경우는 이렇게 동작한다"
  사람이 경계를 직접 생각해야 함

속성 기반 테스트:
  임의 입력 → 성질이 항상 성립
  "어떤 입력에서도 이 조건이 유지된다"
  도구가 경계를 자동 탐색

jqwik 핵심:
  @Property: 속성 테스트
  @ForAll: 임의 값 파라미터
  @IntRange, @StringLength 등: 범위 제한
  Assume.that(): 전제 조건 필터링
  @Provide: 커스텀 Arbitrary 등록
  Shrinking: 실패 최소 입력 자동 탐색

표현할 수 있는 속성:
  단조성 (더 크면 결과도 더 크다)
  불변식 (항상 양수, 원금 초과 안 됨)
  Roundtrip (직렬화 → 역직렬화 = 원본)
  대칭성 (A+B = B+A)
  멱등성 (두 번 적용 = 한 번 적용)

함께 쓰기:
  예제 기반: 핵심 시나리오, 비즈니스 규칙
  속성 기반: 수학적 성질, 알고리즘, 변환 로직
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 메서드에 대해 예제 기반 테스트가 잡지 못하는 버그를 속성 기반 테스트로 잡을 수 있는 속성 두 가지를 작성하라.

```java
public int splitEvenly(int totalAmount, int parts) {
    return totalAmount / parts;
}
// 실제 요구사항:
// - 나눗셈 결과의 합이 totalAmount를 초과해서는 안 된다
// - parts가 1이면 전체 금액이 반환되어야 한다
```

**Q2.** `Assume.that()`은 어떤 역할을 하는가? 아래 테스트에서 `Assume.that()`을 제거하면 어떤 문제가 생기는가?

```java
@Property
void 무게가_클수록_배송비가_비싸거나_같다(
        @ForAll @IntRange(min = 1, max = 50_000) int w1,
        @ForAll @IntRange(min = 1, max = 50_000) int w2) {

    Assume.that(w1 <= w2);

    assertThat(fee(w1)).isLessThanOrEqualTo(fee(w2));
}
```

**Q3.** 직렬화/역직렬화 Roundtrip 속성 테스트가 통과해도 보장하지 못하는 것은 무엇인가? 속성 기반 테스트의 한계를 설명하라.

> 💡 **해설**

**Q1.**

속성 1: "나눗셈 결과 × parts ≤ totalAmount" (결과의 합이 원금을 초과하지 않는다)

```java
@Property
void 분할_합계는_원금을_초과하지_않는다(
        @ForAll @IntRange(min = 1, max = 10_000_000) int totalAmount,
        @ForAll @IntRange(min = 1, max = 100) int parts) {

    int perPart = service.splitEvenly(totalAmount, parts);
    assertThat((long) perPart * parts).isLessThanOrEqualTo(totalAmount);
}
```

속성 2: "parts=1이면 결과 = totalAmount"

```java
@Property
void 1로_나누면_전체금액(
        @ForAll @IntRange(min = 1, max = 10_000_000) int totalAmount) {

    assertThat(service.splitEvenly(totalAmount, 1)).isEqualTo(totalAmount);
}
```

예제 기반 테스트에서 `splitEvenly(100, 3)` → `33`을 검증해도, `33 * 3 = 99 ≤ 100`이 항상 성립하는지는 검증하지 못한다. 속성 기반 테스트는 수천 개의 임의 쌍으로 이 성질을 자동 검증한다.

**Q2.**

`Assume.that(w1 <= w2)`의 역할: 생성된 무작위 쌍 중 `w1 > w2`인 경우를 테스트 대상에서 제외한다. "w1이 w2보다 작거나 같을 때만 이 속성을 검증한다"는 전제 조건이다.

제거하면 생기는 문제: `w1 > w2`인 경우에도 `fee(w1) ≤ fee(w2)`를 검증한다. `w1 = 10_000`, `w2 = 1_000`이면 `fee(10_000) = 5_000`, `fee(1_000) = 3_000`으로 `5_000 > 3_000`이 되어 단언이 실패한다. 이것은 실제 버그가 아니라 잘못된 전제 조건 설정이 원인인 거짓 실패(false failure)다.

`Assume.that()`은 전제 조건을 명시적으로 표현해서, 우리가 검증하고자 하는 속성의 적용 범위를 정확히 한정한다.

**Q3.**

Roundtrip 속성 테스트가 보장하지 못하는 것: 직렬화된 형식(JSON 구조)이 올바른지 검증하지 않는다.

예를 들어 `Order`를 직렬화했을 때 JSON이 아래와 같아야 한다는 계약이 있다고 가정한다.

```json
{"order_id": 1, "total_price": 20000}
```

그런데 실제로 다음과 같이 직렬화됐을 때:

```json
{"orderId": 1, "totalPrice": 20000}
```

`objectMapper.readValue(json, Order.class)`는 성공적으로 역직렬화해서 원본과 같은 객체를 반환한다. Roundtrip 테스트는 통과한다. 하지만 클라이언트가 `order_id`를 기대하면 실제로는 연동이 깨진다.

속성 기반 테스트의 한계: "이 성질이 항상 성립한다"는 것은 검증하지만, "이 특정 입력에 대해 이 특정 결과가 나와야 한다"는 계약(Contract)은 검증하지 못한다. 예제 기반 테스트와 속성 기반 테스트가 보완적인 이유다.

---

<div align="center">

**[⬅️ 이전: Mutation Testing](./01-mutation-testing.md)** | **[다음: Architecture Testing ➡️](./03-architecture-testing.md)**

</div>
