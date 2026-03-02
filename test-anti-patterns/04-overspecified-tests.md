# 04. Overspecified Tests

> **구현을 검증하는 테스트는 리팩터링할 때마다 깨진다 — 테스트가 동작을 보호해야지 구현을 감시해서는 안 된다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 과잉 명세(Overspecification)란 무엇이고 왜 발생하는가?
- 동작(behavior)을 검증하는 테스트와 구현(implementation)을 검증하는 테스트는 어떻게 다른가?
- `verify()`를 언제 써야 하고 언제 쓰지 말아야 하는가?

---

## 🔍 과잉 명세란 무엇인가

과잉 명세(Overspecification)는 테스트가 **결과(what)**가 아닌 **방법(how)**을 검증할 때 발생합니다.

```java
// 요구사항: VIP 회원에게 10% 할인을 적용한다

// ✅ 동작 검증 — "무엇이 되어야 하는가"
@Test
void VIP_회원_10퍼센트_할인_적용() {
    Order order = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());
    assertThat(order.totalPrice()).isEqualTo(18_000);
}

// ❌ 구현 검증 — "어떻게 계산되는가"
@Test
void VIP_회원_할인_계산_과정() {
    Order order = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());

    verify(discountPolicy).calculate(vipUser);          // 이 객체가 호출됐는가?
    verify(discountPolicy, times(1)).calculate(any());  // 정확히 1번 호출됐는가?
    verify(priceCalculator).apply(20_000, 10);          // 이 파라미터로 호출됐는가?
    assertThat(order.totalPrice()).isEqualTo(18_000);   // 결과도 검증
}
```

두 번째 테스트는 `DiscountPolicy`를 다른 방식으로 구현하거나, 계산 중간 단계를 내부 메서드로 추출하면 `verify()`가 실패합니다. 외부에서 보이는 동작(`order.totalPrice() == 18_000`)은 변하지 않았는데도.

---

## 😱 과잉 명세가 만드는 문제

### 문제 1: 리팩터링이 두렵다

```java
// 리팩터링 전: DiscountPolicy.calculate() 사용
public Order place(PlaceOrderCommand command) {
    int discount = discountPolicy.calculate(command.user());
    int finalPrice = command.price() - discount;
    return Order.of(command.user(), finalPrice);
}

// 리팩터링 후: 계산 로직을 Order.create()로 이동
public Order place(PlaceOrderCommand command) {
    return Order.create(command, discountPolicy); // 내부 구조 변경
}
```

동작은 동일하지만 `verify(discountPolicy).calculate(vipUser)`가 실패합니다. 리팩터링 후 깨진 테스트를 수정하는 데 시간을 씁니다. 점점 "테스트가 있으면 리팩터링이 어렵다"는 인식이 생깁니다.

### 문제 2: verify() 과잉 사용

```java
// ❌ 반환값이 있는 메서드의 결과를 확인할 수 있는데 verify를 씀
@Test
void 할인_계산_테스트() {
    when(discountPolicy.calculate(vipUser)).thenReturn(2_000);

    orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());

    verify(discountPolicy).calculate(vipUser); // 호출 여부 검증
    // 실제로 할인이 적용됐는지(18_000)는 검증 안 함
}
```

`verify()`는 호출 여부만 확인합니다. 실제 결과로 검증할 수 있다면 `verify()`는 불필요합니다.

### 문제 3: 내부 구현 상세를 테스트에 노출

```java
// ❌ 내부 메서드 호출 순서까지 검증
@Test
void 주문_처리_순서() {
    InOrder inOrder = inOrder(inventoryClient, orderRepository, eventPublisher);

    orderService.place(aCommand().build());

    inOrder.verify(inventoryClient).reserve(any());     // 1번째
    inOrder.verify(orderRepository).save(any());        // 2번째
    inOrder.verify(eventPublisher).publish(any());      // 3번째
}
```

재고 확인과 저장 순서가 바뀌어도 동작이 올바르다면, 이 테스트는 의미 없이 실패합니다. 순서 자체가 비즈니스 요구사항이라면 검증해야 하지만, 구현 편의상의 순서라면 검증 대상이 아닙니다.

---

## ✨ 동작 검증으로 전환하기

### 원칙: 결과로 검증할 수 있으면 verify()를 쓰지 않는다

```java
// 판단 기준
// "이 메서드의 호출을 검증하지 않고, 결과/상태로 검증할 수 있는가?"
// → YES: assertThat으로 결과 검증
// → NO (부수효과만 있는 경우): verify()로 행동 검증

// ✅ 결과로 검증
@Test
void VIP_할인_적용() {
    Order order = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());
    assertThat(order.totalPrice()).isEqualTo(18_000); // 결과로 충분
    // verify(discountPolicy) 불필요
}

// ✅ verify()가 적절한 경우 — 부수효과 검증
@Test
void 주문_완료_시_이메일_발송() {
    orderService.place(aCommand().build());
    // 이메일은 반환값이 없는 부수효과 → verify 적절
    verify(emailSender).send(eq("user@example.com"), contains("주문 확인"));
}
```

### 인수 매처: 느슨하게, 단언: 엄격하게

```java
// ❌ Stub에서 과잉 명세
when(orderRepository.save(argThat(o ->
    o.userId().equals(1L) &&
    o.totalPrice() == 18_000 &&
    o.status() == PENDING &&
    o.createdAt() != null     // createdAt은 내부에서 생성 — 테스트가 알 필요 없음
))).thenReturn(savedOrder);

// ✅ Stub은 느슨하게, 결과 단언에서 엄격하게
when(orderRepository.save(any(Order.class))).thenReturn(savedOrder);

Order result = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());
assertThat(result.totalPrice()).isEqualTo(18_000); // 중요한 것만 검증
```

### Fake로 교체 — verify() 필요 없어짐

```java
// ❌ Mock + verify 과잉 조합
@Test
void 주문_저장_후_조회_가능() {
    when(orderRepository.save(any())).thenReturn(savedOrder);
    when(orderRepository.findById(1L)).thenReturn(Optional.of(savedOrder));

    orderService.place(aCommand().build());

    verify(orderRepository).save(any()); // 저장됐는가?
    // findById는 검증 안 함 — 연결이 되는지 알 수 없음
}

// ✅ Fake — verify 없이 상태로 검증
@Test
void 주문_저장_후_조회_가능() {
    InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
    OrderService service = new OrderService(fakeRepo, ...);

    Order placed = service.place(aCommand().build());

    // save()와 findById()의 연결이 실제로 동작하는지 검증
    assertThat(fakeRepo.findById(placed.id())).isPresent();
}
```

---

## 💻 실전 적용: verify() 사용 판단 체크리스트

```
verify()를 쓰기 전에 질문한다:

① "반환값이나 상태로 같은 것을 검증할 수 있는가?"
   → YES: assertThat으로 대체

② "이 호출이 없으면 시스템이 잘못 동작하는가?"
   → YES: verify() 사용 (부수효과)
   → NO: verify() 불필요

③ "이 verify()가 실패했을 때 무엇이 잘못됐는지 바로 알 수 있는가?"
   → NO: 결과 단언으로 대체

verify()가 적절한 경우:
  이메일 발송 (EmailSender.send())
  이벤트 발행 (EventPublisher.publish())
  외부 API 호출 (PaymentGateway.charge())
  캐시 무효화 (CacheManager.evict())
  감사 로그 기록 (AuditLogger.log())

verify()가 불필요한 경우:
  Repository.save() — Fake로 상태 검증
  DiscountPolicy.calculate() — 결과 가격으로 검증
  Validator.validate() — 예외 여부로 검증
```

---

## 🤔 트레이드오프

### "구현이 올바르게 동작하는지 확인하려면 내부 호출도 검증해야 하지 않는가?"

내부 호출 검증은 단위 테스트가 아닌 **화이트박스 테스트**입니다. 구현을 변경하면 테스트도 변경해야 하므로 리팩터링 비용이 높아집니다. 외부에서 관찰 가능한 동작(결과, 부수효과)으로 검증하면 내부 구현을 자유롭게 변경할 수 있습니다.

### "InOrder로 순서 검증이 필요한 경우는 없는가?"

있습니다. "재고를 먼저 차감하고 결제를 시도해야 한다" 같이 순서 자체가 비즈니스 요구사항일 때는 `InOrder` 검증이 의미 있습니다. 순서가 구현 편의사항이 아닌 기능 요구사항인 경우입니다.

---

## 📌 핵심 정리

```
과잉 명세란:
  결과(what)가 아닌 방법(how)을 검증하는 것
  구현 세부사항이 테스트에 하드코딩된 상태

증상:
  verify()가 과도하게 사용됨
  리팩터링 후 테스트가 깨짐 (동작은 같은데)
  InOrder로 내부 호출 순서 검증
  argThat으로 내부 상태까지 매칭

원칙:
  결과로 검증할 수 있으면 verify() 불필요
  부수효과(이메일, 이벤트)만 verify() 사용
  Stub은 느슨하게, 단언은 엄격하게

전환 방법:
  verify(repo.save()) → Fake로 상태 검증
  verify(policy.calculate()) → 결과 가격으로 검증
  InOrder (구현 편의) → 제거
  InOrder (비즈니스 요구) → 유지
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트에서 과잉 명세를 찾고, 동작 검증 방식으로 리팩터링하라.

```java
@Test
void 포인트_적립_테스트() {
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));
    when(pointCalculator.calculate(user, 20_000)).thenReturn(200);

    pointService.accumulatePoints(1L, 20_000);

    verify(userRepository).findById(1L);
    verify(pointCalculator).calculate(user, 20_000);
    verify(pointRepository).save(argThat(p ->
        p.userId().equals(1L) && p.amount() == 200
    ));
}
```

**Q2.** `verify()`를 제거하면 "이 메서드가 실제로 호출됐는지 알 수 없다"고 주장하는 팀원이 있다. 어떻게 설득하는가? Fake Repository를 예시로 사용하라.

**Q3.** 아래 두 테스트 중 어느 쪽이 더 나은가? 이유를 설명하라.

```java
// 테스트 A
@Test
void 주문_이메일_발송() {
    orderService.place(aCommand().withUser(user).build());
    verify(emailSender).send(eq(user.email()), any());
}

// 테스트 B
@Test
void 주문_이메일_발송() {
    orderService.place(aCommand().withUser(user).build());
    verify(emailSender).send(
        eq(user.email()),
        eq("주문이 접수됐습니다. 주문 번호: " + orderId + ", 금액: 20,000원")
    );
}
```

> 💡 **해설**

**Q1.**

과잉 명세:

① `verify(userRepository).findById(1L)` — `findById` 호출 여부가 아닌, 포인트가 올바르게 적립됐는지가 중요하다.

② `verify(pointCalculator).calculate(user, 20_000)` — 계산 메서드 호출 여부가 아닌, 계산 결과가 200포인트인지가 중요하다.

리팩터링:

```java
@Test
void 포인트_적립_테스트() {
    // Fake Repository로 상태 검증
    InMemoryPointRepository fakePointRepo = new InMemoryPointRepository();
    UserRepository stubUserRepo = id -> Optional.of(user);
    PointCalculator realCalculator = new PointCalculator(); // 실제 계산기

    PointService service = new PointService(stubUserRepo, realCalculator, fakePointRepo);

    service.accumulatePoints(1L, 20_000);

    // verify 없이 결과로 검증
    Optional<Point> point = fakePointRepo.findByUserId(1L);
    assertThat(point).isPresent();
    assertThat(point.get().amount()).isEqualTo(200);
}
```

**Q2.**

"이 메서드가 호출됐는지 알 수 없다"는 우려는 타당하지만, Fake Repository로 해결할 수 있다.

Mock + verify 방식:

```java
verify(orderRepository).save(any()); // 호출 여부만 확인, 저장 내용 불명확
```

Fake 방식:

```java
InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
service.place(aCommand().build());

// save()가 실제로 호출됐는지, 그리고 올바른 데이터가 저장됐는지 동시에 검증
assertThat(fakeRepo.findAll()).hasSize(1);
assertThat(fakeRepo.findAll().get(0).status()).isEqualTo(PENDING);
```

Fake는 `save()`가 호출됐는지와 저장된 데이터가 올바른지를 동시에 검증한다. `verify(repo.save())`는 호출 여부만 알고 내용은 모른다. Fake가 더 강력한 검증이다.

**Q3.**

테스트 A가 더 낫다. 이유:

테스트 B는 이메일 본문 전체를 하드코딩한다. "주문이 접수됐습니다"가 "주문이 완료됐습니다"로 바뀌거나, 이메일 템플릿이 변경되면 테스트도 수정해야 한다. 이메일 문구 자체가 비즈니스 요구사항이 아닌 UI 세부사항이라면 이 검증은 과잉이다.

테스트 A는 핵심만 검증한다. 이메일이 올바른 수신자에게 발송됐는가(`eq(user.email())`). 내용은 `any()`로 느슨하게 둔다.

다만 이메일 내용에 반드시 특정 정보(주문 번호, 금액)가 포함되어야 한다는 비즈니스 요구사항이 있다면:

```java
verify(emailSender).send(
    eq(user.email()),
    allOf(
        containsString(orderId.toString()),
        containsString("20,000")
    )
);
```

전체 문자열 일치(`eq(...)`)보다 핵심 정보 포함 여부(`containsString`)로 검증하는 것이 균형점이다.

---

<div align="center">

**[⬅️ 이전: Flickering Tests](./03-flickering-tests.md)** | **[다음: Test Code Duplication ➡️](./05-test-code-duplication.md)**

</div>
