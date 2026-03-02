# 03. Mockito Best Practices

> **Mockito는 강력한 도구다 — 잘못 쓰면 테스트가 통과해도 아무것도 검증하지 않는다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `@Mock`과 `@InjectMocks`의 함정은 무엇인가?
- `when().thenReturn()`과 `doReturn().when()`은 언제 구분해서 쓰는가?
- `@Captor`는 어떤 상황에서 단언을 더 명확하게 만드는가?

---

## 🔍 Mockito를 올바르게 쓰기 위한 전제

Mockito는 선언적이고 읽기 좋은 API를 제공하지만, 그 편리함이 잘못된 사용을 숨기기도 합니다. "컴파일되고 테스트가 통과했다"는 것이 "올바르게 검증했다"를 의미하지 않습니다.

---

## 😱 @Mock + @InjectMocks의 함정

### 함정 1: 어떤 생성자가 선택됐는지 알 수 없다

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderRepository orderRepository;
    @Mock DiscountPolicy discountPolicy;
    @Mock EventPublisher eventPublisher;

    @InjectMocks OrderService orderService;  // 어떻게 주입됐는가?
}
```

`@InjectMocks`는 다음 순서로 주입을 시도합니다.
1. 생성자 주입 (가장 큰 생성자부터)
2. Setter 주입
3. 필드 주입

```java
// OrderService에 이런 생성자가 있다면?
public OrderService(OrderRepository orderRepository, DiscountPolicy discountPolicy) {
    this.orderRepository = orderRepository;
    this.discountPolicy = discountPolicy;
    this.eventPublisher = new KafkaEventPublisher(); // 직접 생성!
}
```

`eventPublisher` Mock은 실제로 주입되지 않았습니다. `KafkaEventPublisher`가 쓰입니다. 하지만 컴파일은 되고, `verify(eventPublisher).publish()`는 아무것도 검증하지 않고 통과합니다.

### 함정 2: 주입 실패가 조용히 넘어간다

```java
@InjectMocks SomeService someService;
```

`SomeService`의 생성자 파라미터 중 하나가 Mock으로 제공되지 않으면, Mockito는 `null`을 주입하거나 기본 생성자를 시도합니다. 예외 없이 조용히 실패하기 때문에 나중에 `NullPointerException`으로 나타납니다.

### 해결: 명시적 생성자 주입

```java
// ✅ 무슨 일이 일어나는지 코드에 명시된다
class OrderServiceTest {

    private final OrderRepository orderRepository = mock(OrderRepository.class);
    private final DiscountPolicy discountPolicy = mock(DiscountPolicy.class);
    private final EventPublisher eventPublisher = mock(EventPublisher.class);

    // 생성자로 직접 주입 — 어떤 Mock이 어디 들어가는지 명확
    private final OrderService orderService =
        new OrderService(orderRepository, discountPolicy, eventPublisher);
}
```

이 방식은:
- 어떤 의존성이 주입됐는지 코드로 명확히 보입니다
- 생성자가 바뀌면 컴파일 에러가 나서 즉시 알 수 있습니다
- `@BeforeEach` 없이도 필드 초기화가 가능합니다

---

## 🏛️ when().thenReturn() vs doReturn().when()

### `when().thenReturn()` — 일반적인 경우

```java
// 반환값이 있는 메서드에 사용
when(discountPolicy.calculate(vipUser)).thenReturn(10);
when(repository.findById(1L)).thenReturn(Optional.of(order));
```

내부적으로 `calculate(vipUser)`를 한 번 실제로 호출해서 캡처합니다. **반환값이 있는 메서드**에는 이 방식이 올바릅니다.

### `doReturn().when()` — void 메서드와 Spy

```java
// ❌ void 메서드에 when().thenReturn() 사용 — 컴파일 에러
when(eventPublisher.publish(any())).thenReturn(null); // publish()는 void

// ✅ void 메서드는 doNothing 또는 doThrow
doNothing().when(eventPublisher).publish(any());
doThrow(new RuntimeException("발행 실패")).when(eventPublisher).publish(any());

// ✅ Spy에서 실제 메서드 호출을 막을 때
// when(spy.method())는 spy.method()를 실제로 호출한 후 캡처 → 부수효과 발생
// doReturn()은 실제 메서드를 호출하지 않음
doReturn(10).when(discountPolicySpy).calculate(any());
```

### 정리

| 상황 | 사용 방식 |
|------|----------|
| 반환값 있는 메서드 | `when(mock.method()).thenReturn(value)` |
| void 메서드 | `doNothing().when(mock).method()` |
| void 메서드 예외 | `doThrow(ex).when(mock).method()` |
| Spy 메서드 덮어쓰기 | `doReturn(value).when(spy).method()` |

---

## ✨ @Captor — 복잡한 인자를 정밀하게 검증한다

### 언제 필요한가

```java
// 이벤트 객체 전체를 verify로 비교하기 어려울 때
verify(eventPublisher).publish(???);
// OrderPlaced 이벤트 안의 orderId, userId, amount를 각각 검증하고 싶다면?
```

### argThat으로 인라인 검증

```java
verify(eventPublisher).publish(argThat(event -> {
    OrderPlaced e = (OrderPlaced) event;
    return e.orderId().equals(savedOrder.id())
        && e.userId().equals(user.id())
        && e.amount() == 18_000;
}));
```

간단한 경우에는 `argThat`으로 충분하지만, 여러 필드를 검증하면 람다가 길어집니다.

### @Captor로 캡처 후 AssertJ로 검증

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Captor ArgumentCaptor<DomainEvent> eventCaptor;

    @Test
    void 주문_생성_시_발행된_이벤트를_검증한다() {
        orderService.place(cart, vipUser);

        // 실제 전달된 인자를 캡처
        verify(eventPublisher).publish(eventCaptor.capture());

        DomainEvent captured = eventCaptor.getValue();

        // AssertJ로 정밀하게 검증
        assertThat(captured).isInstanceOf(OrderPlaced.class);
        OrderPlaced event = (OrderPlaced) captured;
        assertThat(event.orderId()).isNotNull();
        assertThat(event.amount()).isEqualTo(18_000);
        assertThat(event.userId()).isEqualTo(vipUser.id());
    }
}
```

`@Captor`를 쓰면:
- 캡처 로직과 검증 로직이 분리되어 가독성이 높아집니다
- AssertJ의 풍부한 단언을 이벤트 내용에 적용할 수 있습니다
- 실패 메시지가 구체적으로 나옵니다

---

## 💻 실전 적용: thenAnswer로 동적 반환

```java
// 호출 인자에 따라 다른 값을 반환해야 할 때
when(repository.findById(anyLong())).thenAnswer(invocation -> {
    Long id = invocation.getArgument(0);
    if (id == 1L) return Optional.of(new Order(1L, 18_000));
    if (id == 2L) return Optional.of(new Order(2L, 9_000));
    return Optional.empty();
});
```

```java
// 호출 횟수마다 다른 값을 반환 (재시도 로직 테스트)
when(externalApi.call())
    .thenThrow(new TimeoutException())  // 첫 번째 호출: 실패
    .thenThrow(new TimeoutException())  // 두 번째 호출: 실패
    .thenReturn(successResponse);       // 세 번째 호출: 성공

retryService.callWithRetry(externalApi);
assertThat(result).isEqualTo(successResponse);
```

---

## 🤔 트레이드오프

### "@Mock + @InjectMocks를 완전히 쓰지 말아야 하는가?"

절대적으로 금지할 이유는 없습니다. 다만 `@InjectMocks`를 사용한다면 **어떤 생성자로 주입되는지 확인**하고, 예상대로 주입됐는지 간단히 검증하는 것이 좋습니다. 생성자 주입 방식을 명시적으로 쓰는 팀도 많습니다.

### "Captor가 너무 많으면 테스트가 복잡해지지 않는가?"

Captor가 많이 필요하다는 것은 하나의 메서드가 너무 많은 협력 객체에 메시지를 보낸다는 신호일 수 있습니다. `verify`와 `Captor`가 3개 이상 필요하다면 메서드의 책임을 분리하는 것을 고려합니다.

---

## 📌 핵심 정리

```
@InjectMocks 주의사항:
  주입 방식이 암묵적 (생성자, Setter, 필드 순)
  주입 실패가 조용히 넘어감
  → 명시적 생성자 주입이 더 안전하고 명확

when() vs doReturn():
  반환값 있는 메서드: when().thenReturn()
  void 메서드:        doNothing/doThrow().when()
  Spy 덮어쓰기:       doReturn().when()

@Captor 사용 기준:
  argThat 람다가 3줄 이상 → Captor로 분리
  이벤트/DTO의 여러 필드를 검증할 때
  AssertJ와 결합해서 실패 메시지를 명확히 하고 싶을 때

thenAnswer:
  호출 인자에 따라 동적으로 다른 값 반환
  재시도 로직에서 호출 순서마다 다른 결과
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 `@InjectMocks`가 예상대로 동작하지 않을 수 있는 이유를 찾고, 명시적 생성자 주입으로 리팩터링하라.

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock PaymentGateway gateway;
    @Mock PaymentRepository repository;

    @InjectMocks PaymentService paymentService;

    // PaymentService의 실제 생성자:
    // public PaymentService(PaymentGateway gateway) {
    //     this.gateway = gateway;
    //     this.repository = new JpaPaymentRepository(); // 직접 생성!
    // }
}
```

**Q2.** `when(userService.getCurrentUser()).thenReturn(user)`를 설정했는데 `NullPointerException`이 발생한다. 어떤 경우에 이런 일이 생기는가?

**Q3.** 아래 verify는 무엇이 문제인가? 어떻게 수정해야 하는가?

```java
orderService.place(cart, vipUser);
orderService.place(cart, normalUser);

// 두 번 호출됐을 때 첫 번째 호출의 인자를 검증하려 한다
verify(eventPublisher, times(2)).publish(argThat(e ->
    ((OrderPlaced) e).userId().equals(vipUser.id())
));
```

> 💡 **해설**
>
> **Q1.** `@InjectMocks`는 가장 큰 생성자를 먼저 시도한다. `PaymentService(PaymentGateway gateway)` 생성자에 `gateway` Mock만 주입하고, `repository`는 `new JpaPaymentRepository()`로 직접 생성된다. 즉 `repository` Mock은 실제로 주입되지 않는다. `verify(repository).save()`는 아무것도 검증하지 않고 통과한다. 리팩터링: `PaymentRepository mockRepo = mock(PaymentRepository.class); PaymentService service = new PaymentService(mockGateway, mockRepo);` — 생성자가 두 인자를 받도록 수정해야 한다. 이것이 `@InjectMocks`가 숨기는 설계 문제이기도 하다.
>
> **Q2.** 두 가지 원인: ① `userService` 자체가 null인 경우 — `@Mock`이 없거나 `MockitoExtension`이 적용되지 않았다. ② `userService.getCurrentUser()`가 `final` 메서드인 경우 — Mockito는 기본적으로 final 메서드를 Mock할 수 없다(MockMaker 설정 필요). ③ static 메서드를 Mock하려는 경우 — Mockito 기본 설정으로는 static Mock 불가 (MockedStatic 필요). 확인 순서: MockitoExtension 선언 → @Mock 선언 → 메서드가 final/static인지 확인.
>
> **Q3.** `times(2).publish(argThat(...))`는 두 번 모두 같은 matcher를 만족해야 통과한다. `vipUser.id()`와 `normalUser.id()`가 다르므로 두 번째 호출은 argThat을 만족하지 못해 verify가 실패한다. 수정: `InOrder inOrder = inOrder(eventPublisher); inOrder.verify(eventPublisher).publish(argThat(e -> ((OrderPlaced)e).userId().equals(vipUser.id()))); inOrder.verify(eventPublisher).publish(argThat(e -> ((OrderPlaced)e).userId().equals(normalUser.id())));` 또는 `@Captor`로 `getAllValues()`를 사용해 각 호출을 분리 검증한다.

---

<div align="center">

**[⬅️ 이전: Stub vs Mock](./02-stub-vs-mock.md)** | **[다음: Partial Mocking with Spy ➡️](./04-partial-mocking-with-spy.md)**

</div>
