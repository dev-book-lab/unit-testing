# 05. Argument Matchers

> **`any()`는 "무엇이든 괜찮다"는 뜻이다 — 검증해야 할 것을 포기하는 순간 테스트는 거짓말을 한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `any()` 남용이 어떤 버그를 테스트가 못 잡게 만드는가?
- `ArgumentCaptor`는 `argThat`과 어떤 상황에서 구분해서 쓰는가?
- Matcher를 조합할 때 지켜야 하는 규칙은 무엇인가?

---

## 🔍 Matcher가 필요한 이유

```java
// Mock에 Stub을 설정할 때 인자를 어떻게 명시하는가?
when(repository.findById(1L)).thenReturn(Optional.of(order));       // 정확한 값
when(repository.findById(anyLong())).thenReturn(Optional.of(order)); // 모든 Long
```

Matcher는 "어떤 인자가 왔을 때 이 Stub이 동작하는가" 또는 "어떤 인자로 호출됐는지 검증한다"를 표현하는 도구입니다. 너무 느슨하면 거짓 통과, 너무 엄격하면 불필요한 취약성이 생깁니다.

---

## 😱 `any()` 남용이 만드는 거짓 통과

### 케이스 1: 잘못된 인자로 호출해도 통과

```java
// ❌ 어떤 인자로 호출해도 통과하는 verify
paymentService.charge(order, card);

verify(paymentGateway).charge(any(), any());
// 실제로 잘못된 금액이나 잘못된 카드 정보로 호출해도 통과
// paymentGateway.charge(wrongOrder, nullCard) 도 통과
```

```java
// ✅ 실제로 검증해야 하는 것을 명시
verify(paymentGateway).charge(
    argThat(req -> req.amount() == order.totalPrice()),
    argThat(c -> c.number().equals(card.number()))
);
```

### 케이스 2: Stub이 예상치 않은 호출에도 반응

```java
// ❌ 어떤 userId가 와도 같은 User를 반환
when(userRepository.findById(any())).thenReturn(Optional.of(vipUser));

// 실수로 잘못된 ID를 넘겨도 vipUser가 반환됨
// userId=2L (일반 회원)인데 VIP 할인이 적용되는 버그를 잡지 못한다
Order order = orderService.place(cart, 2L);
assertThat(order.discountRate()).isEqualTo(10); // 통과 — 하지만 틀린 결과
```

```java
// ✅ 의도한 userId에만 반응
when(userRepository.findById(1L)).thenReturn(Optional.of(vipUser));
when(userRepository.findById(2L)).thenReturn(Optional.of(normalUser));
```

---

## 🏛️ Matcher 종류별 선택 기준

### 정확한 값 — 가장 강한 검증

```java
when(repository.findById(1L)).thenReturn(...);
verify(emailSender).send("alice@example.com", "환영합니다!");
```

정확한 값을 쓸 수 있다면 항상 정확한 값을 씁니다.

### `any()` 계열 — 타입만 확인

```java
any()             // null 포함 모든 값
any(Order.class)  // Order 타입 (null 제외)
anyLong()         // long/Long 타입
anyString()       // String 타입 (null 제외)
anyList()         // List 타입
isNull()          // null만
isNotNull()       // null이 아닌 값
```

```java
// 적절한 any() 사용: 타입만 중요하고 값은 무관한 경우
verify(auditLogger).log(anyString(), anyLong());
// auditLogger는 Dummy 역할 — 무슨 값이든 로그만 남기면 됨
```

### `argThat` — 조건을 람다로 표현

```java
// 특정 조건을 만족하는 인자
verify(paymentGateway).charge(argThat(req ->
    req.amount() > 0 && req.currency().equals("KRW")
));
```

### `eq()` — 다른 Matcher와 혼합할 때 정확한 값 지정

**Mockito 규칙:** 하나의 메서드 호출에서 Matcher를 하나라도 쓰면, **모든 인자에 Matcher를 써야 한다.** 정확한 값은 `eq()`로 감쌉니다.

```java
// ❌ 런타임 에러: order는 Matcher가 아닌데 anyInt()는 Matcher
verify(service).process(order, anyInt());

// ✅ eq()로 감싸서 Matcher로 변환
verify(service).process(eq(order), anyInt());
```

---

## ✨ ArgumentCaptor — 복잡한 객체를 캡처해서 검증

`argThat`은 인라인 검증에 적합하지만, 검증할 필드가 많거나 검증 로직이 복잡하면 람다가 길어집니다. `ArgumentCaptor`는 인자를 캡처한 후 AssertJ로 검증합니다.

### argThat vs ArgumentCaptor 비교

```java
// argThat — 조건이 단순할 때
verify(eventPublisher).publish(argThat(e ->
    e instanceof OrderPlaced && ((OrderPlaced) e).orderId() != null
));

// ArgumentCaptor — 여러 필드를 검증할 때
ArgumentCaptor<DomainEvent> captor = ArgumentCaptor.forClass(DomainEvent.class);
verify(eventPublisher).publish(captor.capture());

OrderPlaced event = (OrderPlaced) captor.getValue();
assertThat(event.orderId()).isNotNull();
assertThat(event.userId()).isEqualTo(vipUser.id());
assertThat(event.amount()).isEqualTo(18_000);
assertThat(event.occurredAt()).isBeforeOrEqualTo(LocalDateTime.now());
```

### 여러 번 호출된 경우 — `getAllValues()`

```java
orderService.place(cart1, vipUser);
orderService.place(cart2, normalUser);

ArgumentCaptor<DomainEvent> captor = ArgumentCaptor.forClass(DomainEvent.class);
verify(eventPublisher, times(2)).publish(captor.capture());

List<DomainEvent> events = captor.getAllValues();
assertThat(events).hasSize(2);
assertThat(events.get(0)).isInstanceOf(OrderPlaced.class);

OrderPlaced first = (OrderPlaced) events.get(0);
assertThat(first.userId()).isEqualTo(vipUser.id());
assertThat(first.amount()).isEqualTo(18_000); // VIP 10% 할인
```

---

## 💻 실전 적용: Stub에서 느슨하게, verify에서 엄격하게

```java
@Test
void VIP_회원_주문_시_결제_요청이_올바르게_전송된다() {

    // Stub: 어떤 PaymentRequest든 성공 반환
    when(paymentGateway.charge(any(PaymentRequest.class)))
        .thenReturn(PaymentResult.success());

    // Act
    Order order = orderService.place(cart, vipUser);

    // verify: 실제로 올바른 금액으로 요청됐는지 Captor로 검증
    ArgumentCaptor<PaymentRequest> captor =
        ArgumentCaptor.forClass(PaymentRequest.class);
    verify(paymentGateway).charge(captor.capture());

    PaymentRequest request = captor.getValue();
    assertThat(request.amount()).isEqualTo(18_000);     // VIP 할인 적용됐는가
    assertThat(request.currency()).isEqualTo("KRW");
    assertThat(request.orderId()).isEqualTo(order.id());
}
```

Stub에서는 `any()`로 느슨하게 설정하고, verify에서는 Captor로 엄격하게 검증합니다. 역할에 맞게 분리하는 것이 핵심입니다.

---

## 🤔 트레이드오프

### "argThat 람다 안에서 예외가 나면 알기 어렵다"

맞습니다. `argThat` 람다 내부에서 `ClassCastException`이나 NPE가 발생하면 Mockito가 matcher 실패로 처리해서 실제 원인이 숨겨집니다. 복잡한 객체 검증은 Captor + AssertJ가 더 안전합니다.

### "Captor가 많아지면 테스트가 길어진다"

Captor가 3개 이상 필요하다면, 하나의 메서드가 너무 많은 협력 객체에 복잡한 메시지를 보내는 설계 신호일 수 있습니다. 메서드 분리를 고려합니다.

---

## 📌 핵심 정리

```
Matcher 선택 기준:
  정확한 값을 알 수 있다 → 정확한 값
  타입만 중요하다       → any(Type.class)
  조건이 있다 (단순)    → argThat(람다)
  조건이 있다 (복잡)    → ArgumentCaptor + AssertJ

Mockito Matcher 혼합 규칙:
  한 메서드에 Matcher가 하나라도 있으면
  모든 인자에 Matcher 사용
  정확한 값은 eq()로 감싸기

any() 남용 경고:
  verify(mock).method(any(), any(), any())
  → 잘못된 인자로 호출해도 통과한다
  → 검증할 가치 있는 인자는 명시한다

ArgumentCaptor 패턴:
  .capture()      → 인자 캡처 (verify에서 사용)
  .getValue()     → 단일 호출의 캡처값
  .getAllValues()  → 여러 호출의 캡처값 목록
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 verify에서 `any()`를 쓰는 것이 적절한지 판단하고, 부적절하다면 수정하라.

```java
// 케이스 A: (String action, Long userId)
verify(auditLogger).log(any(), any());

// 케이스 B: (PaymentRequest request)
verify(paymentGateway).charge(any());

// 케이스 C: (String to, String body)
verify(emailSender).send(eq("alice@example.com"), any());
```

**Q2.** 아래 코드는 Mockito 규칙을 위반한다. 무엇이 문제이고 어떻게 수정하는가?

```java
// order는 정확한 값, anyInt()는 Matcher
verify(service).process(order, anyInt());
```

**Q3.** `ArgumentCaptor`로 캡처한 값이 예상과 다를 때 실패 메시지는 어떻게 나오는가? `argThat`과 비교해서 어느 쪽이 디버깅에 더 유리한가?

> 💡 **해설**
>
> **Q1.** 케이스 A: 적절하다. `auditLogger`는 Dummy 역할이므로 어떤 값으로 호출됐는지는 이 테스트의 관심사가 아니다. 케이스 B: 부적절하다. 결제 게이트웨이에 전달되는 금액, 통화, 주문 ID가 올바른지 검증하지 않으면 테스트 가치가 없다. `ArgumentCaptor<PaymentRequest>`로 캡처 후 `assertThat(request.amount()).isEqualTo(18_000)` 등으로 검증해야 한다. 케이스 C: 절반만 적절하다. 수신자는 검증하지만 본문은 `any()`로 포기했다. "환영합니다" 메시지가 누락되는 버그를 잡을 수 없다. `argThat(body -> body.contains("환영"))`으로 최소한의 내용 검증을 추가한다.
>
> **Q2.** 문제: `order`는 일반 객체(Matcher 아님), `anyInt()`는 Matcher다. Mockito는 Matcher와 비Matcher를 혼합하면 `InvalidUseOfMatchersException`을 던진다. 수정: `verify(service).process(eq(order), anyInt())` — `order`를 `eq()`로 감싸서 Matcher로 만든다.
>
> **Q3.** `argThat` 실패 메시지: `Wanted but not invoked: eventPublisher.publish(<lambda condition>)`. 람다 조건식 자체가 출력되지 않아 어떤 값이 실제로 전달됐는지 알 수 없다. 디버거를 켜야 한다. `ArgumentCaptor` 실패 메시지: Captor는 항상 캡처에 성공한다. 이후 `assertThat(event.amount()).isEqualTo(18_000)` 실패 시 `expected: 18000, but was: 20000` 처럼 AssertJ의 명확한 메시지가 출력된다. 결론: 복잡한 객체 검증은 `ArgumentCaptor`가 압도적으로 유리하다. `argThat`은 실패 이유를 숨긴다.

---

<div align="center">

**[⬅️ 이전: Partial Mocking with Spy](./04-partial-mocking-with-spy.md)** | **[다음: Verify Wisely ➡️](./06-verify-wisely.md)**

</div>
