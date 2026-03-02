# 01. Single Assert Principle

> **하나의 테스트가 하나의 이유로만 실패해야 한다 — 단언의 수가 아니라 검증 목적의 수가 기준이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- "단일 단언"의 진짜 의미는 물리적인 `assertThat` 한 줄인가, 아닌가?
- 여러 단언이 한 테스트에 있어도 되는 경우는 언제인가?
- 단언이 많아지는 것이 어떤 설계 문제를 드러내는가?

---

## 🔍 왜 이 원칙이 필요한가

```java
// 이 테스트가 실패하면, 어디서 왜 실패했는가?
@Test
void 주문_생성_테스트() {
    Order order = orderService.place(cart, vipUser);

    assertThat(order.id()).isNotNull();
    assertThat(order.status()).isEqualTo(PENDING);
    assertThat(order.totalPrice()).isEqualTo(18_000);
    assertThat(order.items()).hasSize(1);
    assertThat(order.createdAt()).isBeforeOrEqualTo(LocalDateTime.now());
}
```

`totalPrice`가 `20_000`으로 잘못 계산됐다면 세 번째 `assertThat`에서 멈춥니다. 네 번째, 다섯 번째 단언이 통과하는지 실패하는지 알 수 없습니다. "이 테스트에서 몇 개의 것이 동시에 잘못됐는가?"를 파악하려면 첫 번째 실패를 고친 후 다시 실행해야 합니다.

더 근본적인 문제는 이름입니다. `주문_생성_테스트`가 실패했을 때 **무엇이** 잘못됐는지 이름만 보고 알 수 없습니다.

---

## 🏛️ 단일 단언 원칙의 진짜 의미

단일 단언 원칙을 "assertThat을 하나만 써라"로 오해하면 안 됩니다.

**원칙의 실제 의미:** 하나의 테스트는 **하나의 동작(behavior)** 만 검증해야 한다.

```java
// ✅ assertThat이 3개지만 단일 동작을 검증한다 — 올바름
@Test
void VIP_회원_주문_시_할인이_적용된다() {
    Order order = orderService.place(cart, vipUser);

    // 세 단언이 모두 "할인 적용"이라는 하나의 동작을 다각도로 검증한다
    assertThat(order.totalPrice()).isEqualTo(18_000);          // 금액이 맞는가
    assertThat(order.discountAmount()).isEqualTo(2_000);        // 할인액이 맞는가
    assertThat(order.appliedPolicy()).isEqualTo("VIP_10%");     // 정책이 맞는가
}

// ❌ assertThat이 2개지만 두 동작을 검증한다 — 분리해야 함
@Test
void 주문_생성_후_알림_발송() {
    Order order = orderService.place(cart, vipUser);

    assertThat(order.status()).isEqualTo(PENDING);              // 주문 생성 검증
    assertThat(emailSender.sentCount()).isEqualTo(1);           // 알림 발송 검증 (다른 동작!)
}
```

**분리 기준:** 테스트 이름 하나로 모든 단언을 설명할 수 있는가?
- 설명 가능하면 → 단일 동작을 검증하는 것
- 설명에 "그리고", "또한"이 필요하면 → 두 개의 테스트로 분리

---

## 😱 단언이 많아지는 흔한 패턴

### 패턴 1: 여러 시나리오를 한 테스트에 우겨넣기

```java
@Test
void 할인_정책_테스트() {
    // 시나리오 1: VIP
    assertThat(calculator.calculate(10_000, "VIP")).isEqualTo(9_000);

    // 시나리오 2: GOLD
    assertThat(calculator.calculate(10_000, "GOLD")).isEqualTo(9_500);

    // 시나리오 3: NORMAL
    assertThat(calculator.calculate(10_000, "NORMAL")).isEqualTo(10_000);
}
```

VIP 계산이 틀리면 GOLD, NORMAL 계산이 맞는지 알 수 없습니다. 이 패턴은 `@ParameterizedTest`로 해결합니다. (→ `04-parameterized-tests.md`)

### 패턴 2: 상태 변화 전후를 모두 한 테스트에서 확인

```java
@Test
void 재고_차감_테스트() {
    int before = inventory.stock("책");
    assertThat(before).isEqualTo(10);          // 사전 상태 검증

    orderService.place(cart, user);

    int after = inventory.stock("책");
    assertThat(after).isEqualTo(9);            // 사후 상태 검증
    assertThat(before - after).isEqualTo(1);   // 차이 검증
}
```

사전 상태 단언(`before == 10`)은 테스트의 **전제 조건**이지 검증 대상이 아닙니다. 이것은 `assertThat` 대신 `assumeThat`이나 단순한 변수 선언으로 처리합니다.

```java
// ✅ 검증 목적에 맞게 정리
@Test
void 주문_시_재고가_1개_차감된다() {
    // Arrange — 사전 상태는 검증 대상이 아닌 설정
    inventory.setStock("책", 10);

    // Act
    orderService.place(cartWith("책"), user);

    // Assert — 변화의 결과만 검증
    assertThat(inventory.stock("책")).isEqualTo(9);
}
```

---

## ✨ 단언을 올바르게 분리하는 방법

### 테스트를 쪼개는 기준: "이 단언이 실패했을 때 테스트 이름이 설명하는가?"

```java
// ❌ Before: 하나의 테스트, 세 가지 다른 동작
@Test
void 주문_생성_테스트() {
    Order order = orderService.place(cart, vipUser);

    assertThat(order.totalPrice()).isEqualTo(18_000);    // 동작 1: 할인 적용
    assertThat(order.status()).isEqualTo(PENDING);        // 동작 2: 초기 상태
    assertThat(emailSender.sentCount()).isEqualTo(1);     // 동작 3: 알림 발송
}

// ✅ After: 세 개의 테스트, 각각 하나의 동작
@Test
void VIP_회원_주문_시_10퍼센트_할인이_적용된다() {
    Order order = orderService.place(cart, vipUser);
    assertThat(order.totalPrice()).isEqualTo(18_000);
}

@Test
void 주문_생성_직후_상태는_PENDING이다() {
    Order order = orderService.place(cart, vipUser);
    assertThat(order.status()).isEqualTo(PENDING);
}

@Test
void 주문_생성_시_확인_이메일이_발송된다() {
    orderService.place(cart, vipUser);
    assertThat(emailSender.sentCount()).isEqualTo(1);
}
```

이제 세 번째 테스트가 실패하면 "주문 생성 시 확인 이메일이 발송돼야 하는데 안 됐다"는 것을 이름만 보고 알 수 있습니다.

### 여러 단언이 있어도 괜찮은 경우: SoftAssertions

한 동작의 결과가 여러 필드에 걸쳐 있을 때, 하나가 실패해도 나머지를 모두 확인하고 싶다면 `SoftAssertions`을 사용합니다.

```java
@Test
void 결제_완료_주문의_전체_상태를_검증한다() {
    Order order = orderService.place(cart, vipUser);

    // SoftAssertions: 모든 단언을 실행한 후 한 번에 실패를 보고한다
    SoftAssertions.assertSoftly(softly -> {
        softly.assertThat(order.totalPrice()).isEqualTo(18_000);
        softly.assertThat(order.discountAmount()).isEqualTo(2_000);
        softly.assertThat(order.appliedPolicy()).isEqualTo("VIP_10%");
    });
}
```

일반 `assertThat`은 첫 번째 실패에서 멈추지만, `SoftAssertions`는 모든 단언을 실행하고 실패한 것 전부를 한꺼번에 보고합니다.

---

## 💻 실전 적용: 단언 개수로 테스트 건강도 확인

```bash
# 테스트당 평균 단언 개수 확인 (간단한 스크립트)
grep -c "assertThat\|assertThatThrownBy\|verify" src/test/java/**/*Test.java \
  | awk -F: '{sum += $2; count++} END {print "평균 단언 수:", sum/count}'
```

경험적 기준:
- **1~3개:** 건강한 테스트
- **4~6개:** 검토 필요 — 단일 동작인지 확인
- **7개 이상:** 거의 확실히 분리 대상

---

## 🤔 트레이드오프

### "테스트를 너무 잘게 쪼개면 설정 코드가 반복된다"

맞습니다. `orderService.place(cart, vipUser)` 호출이 세 테스트 모두에서 반복됩니다. 이 문제는 `@BeforeEach`로 공통 Arrange를 올리거나, Test Data Builder로 설정 코드를 간결하게 만들어서 해결합니다.

반복이 불편하더라도, 테스트가 독립적으로 의미를 갖는 것이 우선입니다. 반복 코드 제거는 그 다음 단계입니다.

### "DTO 전체 필드를 검증해야 할 때는?"

```java
// 모든 필드를 검증해야 할 때 — usingRecursiveComparison 활용
@Test
void 주문_응답_DTO_전체_필드를_검증한다() {
    OrderResponse response = orderService.place(cart, vipUser);

    OrderResponse expected = OrderResponse.builder()
        .totalPrice(18_000)
        .discountAmount(2_000)
        .status(PENDING)
        .build();

    assertThat(response)
        .usingRecursiveComparison()
        .ignoringFields("id", "createdAt")  // 동적 필드 제외
        .isEqualTo(expected);
}
```

이 방식은 "DTO가 올바르게 구성된다"는 하나의 동작을 검증하면서 모든 필드를 커버합니다.

---

## 📌 핵심 정리

```
단일 단언 원칙의 실제 의미:
  assertThat 한 줄 = 아님
  하나의 동작(behavior)만 검증 = 맞음

분리 기준:
  테스트 이름 하나로 모든 단언을 설명할 수 있는가?
  → YES: 그대로 유지
  → NO: 테스트를 분리하라

여러 단언이 허용되는 경우:
  같은 동작의 결과를 다각도로 검증할 때
  SoftAssertions로 전체 실패를 한 번에 확인할 때

단언이 많아지는 신호:
  "그리고", "또한"이 이름에 필요함 → 분리
  서로 다른 협력 객체 상태를 검증 → 분리
  시나리오가 여러 개 → @ParameterizedTest

경험적 기준:
  1~3개: 건강한 테스트
  7개 이상: 거의 확실히 분리 대상
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트를 단일 단언 원칙에 따라 분리하라. 분리 기준을 먼저 설명하고, 분리된 각 테스트의 이름도 작성하라.

```java
@Test
void 회원_가입_테스트() {
    UserService service = new UserService(repo, emailSender, passwordEncoder);

    User user = service.register("alice@example.com", "password123!");

    assertThat(user.id()).isNotNull();
    assertThat(user.email()).isEqualTo("alice@example.com");
    assertThat(user.password()).isNotEqualTo("password123!");  // 암호화 확인
    assertThat(emailSender.lastSentTo()).isEqualTo("alice@example.com");
    assertThat(repo.count()).isEqualTo(1);
}
```

**Q2.** 다음 두 테스트 중 어느 것이 단일 단언 원칙을 더 잘 따르는가? 그 이유는 무엇인가?

```java
// 테스트 A
@Test
void 할인_계산() {
    assertThat(calculator.calculate(10_000, "VIP")).isEqualTo(9_000);
    assertThat(calculator.calculate(10_000, "GOLD")).isEqualTo(9_500);
}

// 테스트 B
@Test
void VIP_할인_적용_결과의_금액_할인액_정책명이_올바르다() {
    DiscountResult result = calculator.calculateDetail(10_000, "VIP");
    assertThat(result.finalPrice()).isEqualTo(9_000);
    assertThat(result.discountAmount()).isEqualTo(1_000);
    assertThat(result.policyName()).isEqualTo("VIP_10%");
}
```

**Q3.** `SoftAssertions`와 일반 `assertThat` 중 무엇을 기본으로 사용해야 하는가? 각각의 장단점과 선택 기준을 설명하라.

> 💡 **해설**
>
> **Q1.** 분리 기준: "그리고"가 필요한 지점마다 분리한다. `user.id()` + `user.email()`은 "사용자가 저장된다"는 하나의 동작. `user.password()`는 "비밀번호가 암호화된다"는 별개의 동작. `emailSender.lastSentTo()`는 "가입 확인 메일이 발송된다"는 별개의 동작. `repo.count()`는 첫 번째 그룹에 통합 가능. 분리 결과: `유효한_이메일과_비밀번호로_가입하면_사용자가_저장된다()` / `가입_시_비밀번호는_암호화되어_저장된다()` / `가입_완료_시_확인_이메일이_발송된다()`.
>
> **Q2.** 테스트 B가 더 잘 따른다. 테스트 A는 VIP와 GOLD 두 시나리오를 한 테스트에서 검증한다(다른 입력, 다른 결과). 하나가 실패하면 다른 하나의 결과를 알 수 없고, 이름("할인_계산")이 어떤 시나리오가 실패했는지 알려주지 않는다. 이것은 `@ParameterizedTest`로 변환해야 한다. 테스트 B는 VIP 할인이라는 하나의 시나리오에서 발생하는 결과(금액, 할인액, 정책명)를 다각도로 검증한다. 세 단언이 모두 "VIP 할인 계산"이라는 하나의 동작을 설명한다.
>
> **Q3.** 기본은 일반 `assertThat`을 사용한다. 첫 번째 실패에서 멈추는 것이 대부분의 경우 더 명확한 피드백을 준다. 테스트를 제대로 분리했다면 단언이 1~3개이므로 첫 번째 실패에서 멈춰도 정보 손실이 적다. `SoftAssertions`는 하나의 동작 결과가 여러 필드에 걸쳐 있고, 그 필드들이 독립적으로 실패할 수 있을 때(예: DTO 전체 검증, 페이지 렌더링 결과) 사용한다. 남용하면 "모든 것을 한 테스트에 넣어도 된다"는 잘못된 습관을 만든다.

---

<div align="center">

**[다음: Test Data Builders ➡️](./02-test-data-builders.md)**

</div>
