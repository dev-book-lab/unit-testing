# 06. Verify Wisely

> **`verify()`는 "이 호출이 일어났는가"를 검증한다 — 일어났는가보다 결과가 올바른가가 더 중요한 경우가 많다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `verify()`가 꼭 필요한 경우와 쓰면 안 되는 경우는 어떻게 구분하는가?
- 과도한 verify가 테스트를 어떻게 취약하게 만드는가?
- `verify(mock, never())`와 `verifyNoInteractions()`는 언제 가치 있는가?

---

## 🔍 verify()의 본질

`verify()`는 행동 검증(behavior verification) 도구입니다. "이 Mock 객체의 이 메서드가 이 인자로 호출됐는가"를 사후에 확인합니다.

이것이 필요한 순간은 정확히 하나입니다. **반환값도 없고 상태도 남기지 않는 부수효과를 검증할 때.**

그 외의 경우에 `verify()`를 쓰면 테스트가 구현 세부사항을 검증하게 됩니다.

---

## 😱 verify() 과잉 사용이 만드는 문제

### 패턴 1: 결과로 확인할 수 있는 것을 verify로 확인

```java
@Test
void 할인이_적용된_주문이_생성된다() {
    when(discountPolicy.calculate(vipUser)).thenReturn(10);

    Order order = orderService.place(cart, vipUser);

    // ❌ 이것들은 verify가 필요 없다
    verify(discountPolicy).calculate(vipUser);       // 호출됐는지
    verify(orderRepository).save(any(Order.class));  // 저장 호출됐는지

    // ✅ 결과로 직접 확인하면 충분하다
    assertThat(order.totalPrice()).isEqualTo(18_000);
    assertThat(fakeRepository.count()).isEqualTo(1);
}
```

`discountPolicy.calculate()`가 호출됐는지 확인하는 것이 목적인가, 아니면 할인이 올바르게 적용됐는지 확인하는 것이 목적인가?

후자가 목적이라면 `assertThat(order.totalPrice()).isEqualTo(18_000)`으로 충분합니다. `calculate()`가 호출됐다는 사실은 결과가 올바르면 자명합니다.

### 패턴 2: 모든 협력 객체 호출을 verify로 고정

```java
@Test
void 주문_처리_전체_흐름() {
    orderService.place(cart, user);

    // ❌ 구현 흐름 전체를 verify로 고정 — 구현의 그림자
    verify(inventoryClient).reserve(cart);
    verify(discountPolicy).calculate(user);
    verify(orderRepository).save(any());
    verify(eventPublisher).publish(any());
    verify(auditLogger).log(anyString(), anyLong());
}
```

이 테스트는 `orderService.place()` 내부 구현 순서를 그대로 verify로 베껴놓은 것입니다. `auditLogger.log()`를 `EventListener`로 이동하는 내부 리팩터링을 하면, 기능은 동일한데 이 테스트가 깨집니다.

---

## ✨ verify()가 필요한 경우

### 케이스 1: 부수효과 — 반환값도 상태도 없는 호출

```java
// ✅ 이메일 발송 검증 — 반환값 없음, 상태 없음
@Test
void 주문_확인_이메일이_발송된다() {
    orderService.place(cart, user);

    verify(emailSender).send(
        eq(user.email()),
        argThat(body -> body.contains("주문이 접수됐습니다"))
    );
}

// ✅ 외부 API에 올바른 페이로드 전송 검증
@Test
void 결제_요청이_올바른_금액으로_전송된다() {
    paymentService.pay(order, card);

    ArgumentCaptor<PaymentRequest> captor = ArgumentCaptor.forClass(PaymentRequest.class);
    verify(paymentGateway).charge(captor.capture());
    assertThat(captor.getValue().amount()).isEqualTo(18_000);
}

// ✅ 이벤트 발행 검증
@Test
void OrderPlaced_이벤트가_발행된다() {
    orderService.place(cart, user);
    verify(eventPublisher).publish(any(OrderPlaced.class));
}
```

### 케이스 2: 호출되지 않아야 하는 것 검증

```java
// ✅ 재고 부족 시 결제를 시도하지 않아야 한다
@Test
void 재고_부족_시_결제_요청이_전송되지_않는다() {
    when(inventoryClient.isAvailable(any())).thenReturn(false);

    assertThatThrownBy(() -> orderService.place(cart, user))
        .isInstanceOf(OutOfStockException.class);

    verify(paymentGateway, never()).charge(any());
}

// ✅ 마케팅 수신 거부 회원에게는 프로모션 이메일을 보내지 않아야 한다
@Test
void 마케팅_수신_거부_회원에게는_프로모션_이메일을_보내지_않는다() {
    User optOutUser = aUser().withMarketingOptOut().build();
    notificationService.sendPromotion(optOutUser, promotion);

    verify(emailSender, never()).send(eq(optOutUser.email()), any());
}
```

`never()`와 `verifyNoInteractions()`는 "일어나서는 안 되는 것이 일어나지 않았는가"를 확인하므로 가치 있습니다.

---

## 💻 실전 적용: verify 횟수 옵션

```java
verify(emailSender).send(any(), any());             // 정확히 1번 (기본값)
verify(emailSender, times(2)).send(any(), any());   // 정확히 2번
verify(emailSender, atLeastOnce()).send(any(), any()); // 최소 1번
verify(emailSender, never()).send(any(), any());    // 한 번도 없어야 함

verifyNoInteractions(auditLogger);    // 이 Mock에 어떤 호출도 없어야 함
verifyNoMoreInteractions(emailSender); // verify한 것 외에 추가 호출 없어야 함
```

`verifyNoMoreInteractions()`는 남용을 경계합니다. 모든 호출을 verify해야 하는 부담을 만들고 구현을 통째로 고정시킵니다.

---

## 🤔 트레이드오프

### "verify()를 전혀 쓰지 않아도 되는가?"

부수효과가 핵심인 로직에서는 verify가 유일한 검증 수단입니다. "이메일을 보냈는가", "이벤트를 발행했는가" 같은 것은 결과 상태가 남지 않으므로 verify 없이는 검증이 불가능합니다. 전혀 쓰지 않으면 이런 부수효과 버그를 놓칩니다.

### "`verifyNoMoreInteractions()`를 쓰면 안 되는가?"

권장하지 않습니다. 새로운 기능을 추가할 때마다 기존 테스트를 업데이트해야 하는 부담이 생깁니다. 다만 요금 청구나 보안 경계처럼 "정확히 이것만 호출해야 한다"는 강한 계약이 있는 경우에는 선택적으로 쓸 수 있습니다.

---

## 📌 핵심 정리

```
verify()가 필요한 경우:
  반환값도 없고 상태도 남지 않는 부수효과
  이메일, 알림, 이벤트 발행, 외부 API 전송
  "호출되지 않아야 한다" (never())

verify()가 필요 없는 경우:
  반환값이 있는 계산 → assertThat으로 결과 검증
  Repository 저장 → Fake로 상태 검증
  협력 객체 호출 흐름 전체 → 구현의 그림자

verify 과잉의 신호:
  모든 협력자 호출을 verify함
  리팩터링할 때마다 verify가 깨짐

유용한 verify 옵션:
  times(n)              정확히 n번
  never()               한 번도 없어야 함
  atLeastOnce()         최소 한 번
  verifyNoInteractions  어떤 호출도 없어야 함

판단 기준:
  "이 verify가 없으면 잡지 못하는 버그가 있는가?"
  → YES: 유지
  → NO: assertThat으로 교체 또는 제거
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트에서 verify를 제거해도 되는 것과 반드시 유지해야 하는 것을 구분하라.

```java
@Test
void 주문_생성_테스트() {
    when(discountPolicy.calculate(vipUser)).thenReturn(10);

    Order order = orderService.place(cart, vipUser);

    verify(discountPolicy).calculate(vipUser);              // A
    verify(orderRepository).save(any(Order.class));         // B
    verify(eventPublisher).publish(any(OrderPlaced.class)); // C
    verify(auditLogger).log(anyString(), anyLong());         // D

    assertThat(order.totalPrice()).isEqualTo(18_000);
}
```

**Q2.** 팀원이 "모든 테스트 끝에 `verifyNoMoreInteractions()`를 추가하자"고 제안했다. 이 제안의 장단점은 무엇인가?

**Q3.** `verify(mock, never()).method(args)`와 `verifyNoInteractions(mock)`의 차이는 무엇인가? 각각 어떤 상황에서 쓰는가?

> 💡 **해설**
>
> **Q1.** A (`verify(discountPolicy).calculate(vipUser)`): 제거 가능. `order.totalPrice() == 18_000`이 통과하면 할인이 올바르게 적용됐다는 것이 보장된다. `calculate()`가 호출됐는지 별도로 확인할 필요가 없다. B (`verify(orderRepository).save()`): 제거 가능. Fake Repository를 쓴다면 `fakeRepository.findById(order.id())`로 상태를 검증하면 된다. C (`verify(eventPublisher).publish()`): 유지. 이벤트 발행은 반환값도 없고 상태도 남지 않는다. `assertThat`으로 대체할 수 없다. D (`verify(auditLogger).log()`): 상황에 따라. 감사 로그가 명시적 비즈니스 요구사항이면 유지, 이 테스트의 관심사가 아니라면 제거한다.
>
> **Q2.** 장점: 예상치 못한 추가 Mock 호출(버그로 인한 호출)을 발견할 수 있다. 단점: ① 기능을 추가할 때마다 기존 테스트를 모두 업데이트해야 한다. 내부 로그 호출 하나 추가해도 모든 관련 테스트가 깨진다. ② 테스트가 구현의 복사본이 된다. ③ 유지보수 비용이 크게 높아진다. 결론: 보안이나 요금 청구처럼 "정확히 이것만 호출해야 한다"는 강한 계약이 있는 곳에만 선택적으로 사용한다.
>
> **Q3.** `verify(mock, never()).method(args)`: 특정 메서드의 특정 인자로의 호출이 없어야 한다. 다른 메서드는 호출될 수 있다. 예: "마케팅 수신 거부 사용자에게는 `send(optOutUser.email(), any())`가 호출되면 안 된다. 다른 사용자에게 발송하는 건 허용된다." `verifyNoInteractions(mock)`: 이 Mock 객체에 어떤 호출도 없어야 한다. 예: "결제 실패 시 `auditLogger`에 아무런 로그도 기록되면 안 된다." 선택 기준: 특정 메서드만 금지 → `never()`. 객체 전체에 어떤 호출도 금지 → `verifyNoInteractions()`.

---

<div align="center">

**[⬅️ 이전: Argument Matchers](./05-argument-matchers.md)** | **[다음: Fakes over Mocks ➡️](./07-fakes-over-mocks.md)**

</div>
