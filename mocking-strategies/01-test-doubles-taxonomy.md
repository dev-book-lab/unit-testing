# 01. Test Doubles Taxonomy

> **"Mock 쓰면 되는 거 아닌가요?" — 5가지를 구분하지 못하면 테스트가 실수를 숨긴다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Dummy, Stub, Spy, Mock, Fake는 각각 어떤 상황에서 선택하는가?
- 잘못된 테스트 대역을 선택했을 때 어떤 일이 생기는가?
- "테스트 대역(Test Double)"이라는 용어는 왜 존재하는가?

---

## 🔍 왜 구분이 필요한가

```java
public class OrderService {
    private final InventoryClient inventoryClient;   // 외부 재고 API
    private final OrderRepository orderRepository;   // DB
    private final EventPublisher eventPublisher;     // 이벤트 버스
    private final AuditLogger auditLogger;           // 감사 로그

    public Order place(Cart cart, User user) {
        inventoryClient.reserve(cart);               // 실패하면 재고 없음
        Order order = Order.from(cart, user);
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlaced(order));
        auditLogger.log("ORDER_PLACED", user.id());
        return order;
    }
}
```

이 메서드를 테스트하려면 4개의 의존성을 교체해야 합니다. 그런데 각 의존성을 같은 방식으로 교체하는 것이 맞을까요?

- `inventoryClient`: 재고 예약이 **성공했는가** — 상태가 필요
- `orderRepository`: 저장된 주문을 **나중에 조회할 수 있어야** 함
- `eventPublisher`: 이벤트가 **발행됐는가** — 호출 여부가 관심
- `auditLogger`: 이 테스트에서는 **아무 역할도 하지 않아도 됨**

4개에 같은 Mock을 쓰면 테스트가 필요 이상으로 복잡해지거나, 검증해야 할 것을 검증하지 못하게 됩니다.

---

## 🏛️ Test Double 5가지 분류

Gerard Meszaros가 *xUnit Test Patterns*에서 정의한 분류입니다. 이 5가지는 테스트 대역의 **목적**에 따라 구분됩니다.

### 1. Dummy — 전달만 하고 사용되지 않는다

```java
// 파라미터를 채워야 하는데, 이 테스트에서 실제로 쓰이지 않는 값
User dummyUser = new User("ignored@example.com");
AuditLogger dummyLogger = mock(AuditLogger.class); // 호출 여부 관심 없음

// auditLogger는 place()에서 호출되지만 이 테스트의 관심 대상이 아님
orderService.place(cart, dummyUser);
```

**언제:** 메서드 시그니처를 채우기 위해 필요하지만, 테스트 결과에 영향을 주지 않는 객체.

---

### 2. Stub — 미리 정해진 값을 반환한다

```java
// 재고 확인이 항상 성공하도록 고정
InventoryClient stubInventory = mock(InventoryClient.class);
when(stubInventory.isAvailable("책", 1)).thenReturn(true);

// 또는 람다 Stub
InventoryClient stubInventory = (itemId, qty) -> true;
```

**언제:** 테스트 대상이 의존하는 값이 필요하고, 그 값이 고정된 경우. Stub의 관심사는 "어떤 값을 반환하는가"이고, "몇 번 호출됐는가"는 관심 없습니다.

**주의:** Stub은 **상태 검증**에 쓰입니다. Stub이 반환한 값이 최종 결과에 올바르게 반영됐는지 확인합니다.

---

### 3. Spy — 실제 객체를 감싸고 호출 기록을 남긴다

```java
// 실제 EmailSender를 쓰되, 발송 횟수를 기록
EmailSender realSender = new SmtpEmailSender(config);
EmailSender spySender = spy(realSender);

orderService.place(cart, user);

verify(spySender, times(1)).send(any());  // 실제 발송됐는지 확인
```

**언제:** 실제 구현 동작은 유지하면서, 호출 기록도 확인하고 싶을 때. Spy는 실제 메서드를 실행합니다(Stub은 실행 안 함).

**주의:** Spy가 필요하다면 설계를 의심하라. (→ `04-partial-mocking-with-spy.md`)

---

### 4. Mock — 기대하는 호출을 사전에 정의하고 검증한다

```java
EventPublisher mockPublisher = mock(EventPublisher.class);

orderService.place(cart, user);

// 사후에 "이 메서드가 이 인자로 호출됐는가" 검증
verify(mockPublisher).publish(argThat(e -> e instanceof OrderPlaced));
```

**언제:** 테스트 대상이 협력 객체에게 올바른 메시지를 보내는지 **행동 자체**를 검증할 때. Mock의 관심사는 "어떻게 호출됐는가"입니다.

**주의:** Mock은 **행동 검증**입니다. 상태가 아닌 상호작용을 검증합니다.

---

### 5. Fake — 실제로 동작하는 간소화된 구현체

```java
// 실제 DB 없이 동작하는 메모리 구현
class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    private long seq = 1L;

    @Override
    public Order save(Order order) {
        Order saved = order.withId(seq++);
        store.put(saved.id(), saved);
        return saved;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

**언제:** 외부 시스템(DB, 네트워크)을 교체해야 하고, 그 시스템의 **실제 동작**(저장 → 조회)이 테스트에서도 필요할 때. Fake는 진짜처럼 동작하되 외부 의존 없이 빠릅니다.

---

## 😱 잘못된 선택이 만드는 문제

### Mock을 써야 할 자리에 Stub을 쓰면

```java
// ❌ 이벤트 발행 여부를 Stub으로 검증할 수 없다
EventPublisher stubPublisher = mock(EventPublisher.class);
// when() 설정 없음 — void 메서드라서

orderService.place(cart, user);

// 검증도 없음 — 이벤트가 발행됐는지 아무도 확인하지 않는다
// eventPublisher.publish()가 삭제돼도 이 테스트는 통과한다
```

### Fake를 써야 할 자리에 Mock을 쓰면

```java
// ❌ Repository Mock — save 후 findById를 검증할 수 없다
OrderRepository mockRepo = mock(OrderRepository.class);
when(mockRepo.save(any())).thenReturn(savedOrder);

orderService.place(cart, user);

// mockRepo.findById()는 설정하지 않아서 null 반환
// "저장 후 조회가 가능한가"를 검증할 수 없다
```

```java
// ✅ Fake — save 후 findById가 실제로 동작한다
OrderRepository fakeRepo = new InMemoryOrderRepository();

Order result = orderService.place(cart, user);

// 저장된 것을 실제로 조회해서 검증
assertThat(fakeRepo.findById(result.id())).isPresent();
```

---

## ✨ 선택 기준 정리

```
이 의존성이 테스트에서 어떤 역할을 하는가?

"테스트 결과에 아무 영향 없음"
    → Dummy

"테스트 대상에게 값을 제공해야 함"
    → Stub

"실제 구현 + 호출 기록도 필요"
    → Spy  (설계 의심 먼저)

"올바른 메시지를 보냈는지 검증해야 함"
    → Mock

"실제처럼 동작해야 함 (저장 → 조회 등)"
    → Fake
```

---

## 💻 실전 적용: `OrderService.place()` 예제에 대입

```java
class OrderServiceTest {

    // Fake: 저장 후 조회가 테스트에서도 의미 있어야 함
    private final OrderRepository repository = new InMemoryOrderRepository();

    // Stub: 재고 확인 결과를 테스트마다 다르게 줘야 함
    private final InventoryClient inventoryStub = (itemId, qty) -> true;

    // Mock: 이벤트가 발행됐는지 검증해야 함
    private final EventPublisher eventPublisher = mock(EventPublisher.class);

    // Dummy: 이 테스트에서 감사 로그는 관심 없음
    private final AuditLogger auditLogger = mock(AuditLogger.class);

    private final OrderService orderService =
        new OrderService(inventoryStub, repository, eventPublisher, auditLogger);

    @Test
    void 주문_생성_시_OrderPlaced_이벤트가_발행된다() {
        orderService.place(cart, user);

        // Mock으로 행동 검증
        verify(eventPublisher).publish(argThat(e -> e instanceof OrderPlaced));
    }

    @Test
    void 주문_생성_후_조회가_가능하다() {
        Order order = orderService.place(cart, user);

        // Fake로 상태 검증
        assertThat(repository.findById(order.id())).isPresent();
    }
}
```

---

## 🤔 트레이드오프

### "Mockito에서 mock()으로 만들면 다 Mock 아닌가?"

Mockito의 `mock()`은 Dummy, Stub, Mock을 모두 만들 수 있는 도구입니다. `mock()`으로 만든 객체에 `when()`을 설정하면 Stub이고, `verify()`로 검증하면 Mock입니다. 설정도 검증도 없으면 Dummy입니다. **Mockito는 도구이고, Double 분류는 목적**입니다.

### "Fake를 직접 구현하는 비용이 크다"

맞습니다. 특히 인터페이스에 메서드가 많으면 Fake 구현 비용이 높아집니다. 이 비용이 부담된다면 인터페이스가 너무 크다는 설계 신호이기도 합니다. (Interface Segregation Principle)

---

## 📌 핵심 정리

```
Test Double 5가지

Dummy:  전달은 하지만 사용되지 않는 객체
Stub:   고정된 값을 반환 — 상태 검증에 사용
Spy:    실제 객체 + 호출 기록 — 설계 의심 신호
Mock:   기대 호출을 사전 정의 — 행동 검증에 사용
Fake:   실제처럼 동작하는 간소화된 구현체

선택 기준:
  "테스트에서 이 의존성이 하는 역할은 무엇인가?"

흔한 실수:
  저장 후 조회가 필요한데 Mock 사용 → Fake로
  이벤트 발행 검증이 필요한데 Stub 사용 → Mock으로
  어디서나 Mock → 테스트가 구현 세부사항에 속박됨
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 의존성 각각에 대해 어떤 Test Double을 선택할지, 그 이유와 함께 설명하라.

```java
public class PaymentService {
    private final PaymentGateway gateway;       // 외부 결제 API (charge, refund)
    private final PaymentRepository repository;  // 결제 내역 저장
    private final FraudDetector fraudDetector;  // 사기 거래 감지 (boolean 반환)
    private final SlackNotifier slackNotifier;  // Slack 알림 발송
}
```

**Q2.** Stub과 Mock을 혼동하는 팀원이 이벤트 발행을 검증하면서 아래 코드를 작성했다. 무엇이 문제이고, 어떻게 수정해야 하는가?

```java
EventPublisher stubPublisher = mock(EventPublisher.class);
when(stubPublisher.publish(any())).thenReturn(null); // void 메서드인데 이걸 왜 씀?

orderService.place(cart, user);

assertThat(stubPublisher.publish(any())).isNull(); // 이것도 잘못됨
```

**Q3.** Fake와 통합 테스트(실제 DB)의 차이는 무엇인가? Fake만으로 충분한 경우와 실제 DB가 반드시 필요한 경우를 각각 예시로 설명하라.

> 💡 **해설**
>
> **Q1.** `PaymentGateway`: **Stub** — 결제 성공/실패 시나리오를 미리 정해서 테스트. `charge()`가 성공을 반환하는지 실패를 반환하는지에 따라 `PaymentService`의 동작이 달라진다. 단, "결제가 실제로 처리됐는지"가 테스트의 핵심이라면 **Mock**으로 `verify(gateway).charge()`를 검증한다. `PaymentRepository`: **Fake** — 저장된 결제 내역을 나중에 조회해서 검증해야 하므로. `FraudDetector`: **Stub** — 사기 거래 감지 결과를 시나리오별로 고정해서(`true`/`false`) 분기 동작을 테스트. `SlackNotifier`: **Mock** — "알림이 발송됐는가"는 행동 검증이므로 `verify(slackNotifier).send()`로.
>
> **Q2.** 두 가지 문제: ① `when(stubPublisher.publish(any())).thenReturn(null)` — `publish()`가 void 메서드라면 `thenReturn(null)`은 컴파일 에러가 나거나 의미가 없다. void 메서드는 `doNothing().when(stubPublisher).publish(any())`로 설정한다. ② `assertThat(stubPublisher.publish(any())).isNull()` — 이것은 단언이 아니라 `publish()`를 직접 다시 호출하는 것이다. Mock 검증은 `verify(stubPublisher).publish(any())`로 한다. 수정: `verify(eventPublisher).publish(argThat(e -> e instanceof OrderPlaced))`.
>
> **Q3.** Fake로 충분한 경우: 비즈니스 로직이 저장 → 조회 흐름을 따르는지 검증할 때. Fake의 `save()`와 `findById()`가 올바르게 연결되어 있으면 도메인 로직 검증에는 충분하다. 실제 DB가 필요한 경우: ① JPA 쿼리가 올바른지 (JPQL, 네이티브 쿼리) ② DB 제약 조건 (Unique, Not Null) 이 실제로 동작하는지 ③ 트랜잭션 경계와 커밋이 올바른지 ④ N+1 문제나 인덱스 동작을 확인할 때. 이 케이스들은 `@DataJpaTest` + Testcontainers로 통합 테스트를 별도로 작성한다.

---

<div align="center">

**[다음: Stub vs Mock ➡️](./02-stub-vs-mock.md)**

</div>
