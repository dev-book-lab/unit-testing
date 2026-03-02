# 05. Test Transaction Management

> **`@Transactional` 테스트는 거짓말을 한다 — 롤백되는 코드는 실제로 커밋되는 코드와 다르게 동작한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 테스트에 `@Transactional`을 붙이면 어떤 함정이 생기는가?
- 실제 커밋을 검증해야 할 때는 어떻게 하는가?
- 트랜잭션 전파(Propagation)가 테스트에서 어떻게 예상과 다르게 동작하는가?

---

## 🔍 @Transactional 테스트의 편리함과 위험

`@Transactional`을 테스트에 붙이면 각 테스트가 트랜잭션 안에서 실행되고 종료 시 자동으로 롤백됩니다. 데이터 정리가 필요 없어 편리합니다.

```java
// 편리해 보이는 패턴
@SpringBootTest
@Transactional  // 각 테스트 후 자동 롤백
class OrderServiceTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_생성_후_조회() {
        orderService.place(aCommand().withUserId(1L).build());

        List<Order> orders = orderRepository.findByUserId(1L);
        assertThat(orders).hasSize(1);
    }
    // 테스트 종료 → 롤백 → DB 깨끗
}
```

하지만 이 패턴에는 여러 함정이 있습니다.

---

## 😱 함정 1: 트랜잭션 전파가 숨겨진다

```java
@Service
public class OrderService {

    @Transactional  // 새 트랜잭션 시작
    public Order place(PlaceOrderCommand command) {
        Order order = Order.from(command);
        orderRepository.save(order);
        eventPublisher.publish(new OrderPlaced(order));
        return order;
    }
}
```

```java
// 테스트 트랜잭션 안에서 orderService.place() 호출
@SpringBootTest
@Transactional
class OrderServiceTest {

    @Test
    void 주문_생성() {
        orderService.place(aCommand().build());
        // 실제로 어떤 트랜잭션이 동작하는가?
    }
}
```

**실제 동작:**

```
테스트 메서드 시작 → 트랜잭션 T1 시작

orderService.place() 호출
  → @Transactional(propagation = REQUIRED, 기본값)
  → 이미 T1이 있으므로 T1에 참여
  → place()의 @Transactional은 실제로 새 트랜잭션을 시작하지 않음

테스트 종료 → T1 롤백
```

**프로덕션에서의 실제 동작:**

```
HTTP 요청 시작 → 트랜잭션 없음

orderService.place() 호출
  → @Transactional이 새 트랜잭션 T1 시작
  → 작업 완료 → T1 커밋

HTTP 요청 종료
```

테스트에서 `@Transactional(propagation = REQUIRES_NEW)`인 메서드도 테스트 트랜잭션에 흡수됩니다. 실제 커밋/롤백 동작을 검증하지 못합니다.

---

## 😱 함정 2: LazyLoading이 테스트에서만 동작한다

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

```java
@SpringBootTest
@Transactional  // 테스트 전체가 하나의 트랜잭션
class OrderTest {

    @Test
    void 주문_아이템_조회() {
        Order order = orderRepository.save(anOrder().withItems(3).build());

        Order found = orderRepository.findById(order.id()).orElseThrow();
        assertThat(found.items()).hasSize(3); // ✅ 통과 — 같은 트랜잭션이라 Lazy 로딩 가능
    }
}
```

이 테스트는 통과하지만 실제 서비스에서는 실패합니다.

```java
// 프로덕션: orderService.getOrderItems()는 새 트랜잭션에서 실행
@Transactional(readOnly = true)
public List<OrderItem> getOrderItems(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    return order.items(); // LazyInitializationException!
    // 트랜잭션이 분리되어 있어 items가 초기화되지 않음
}
```

테스트의 `@Transactional`이 Lazy 로딩 예외를 감추고, 프로덕션에서 처음 발견합니다.

---

## 😱 함정 3: 이벤트와 부수효과가 검증되지 않는다

```java
@Service
public class OrderService {

    @Transactional
    public Order place(PlaceOrderCommand command) {
        Order order = orderRepository.save(Order.from(command));
        eventPublisher.publish(new OrderPlaced(order)); // 커밋 후 발행
        return order;
    }
}
```

```java
// @TransactionalEventListener는 트랜잭션 커밋 후에 동작
@TransactionalEventListener(phase = AFTER_COMMIT)
public void handleOrderPlaced(OrderPlaced event) {
    emailService.sendOrderConfirmation(event.order());
}
```

```java
@SpringBootTest
@Transactional  // 롤백되므로 AFTER_COMMIT 이벤트가 발생하지 않음
class OrderServiceTest {

    @Test
    void 주문_생성_시_이메일이_발송된다() {
        orderService.place(aCommand().build());

        verify(emailService).sendOrderConfirmation(any()); // ❌ 항상 실패
        // @TransactionalEventListener는 커밋 후에 동작
        // 테스트 트랜잭션은 롤백되므로 이벤트 리스너가 호출되지 않음
    }
}
```

---

## ✨ 실제 커밋을 검증하는 방법

### 방법 1: 테스트에서 @Transactional 제거

```java
@SpringBootTest
class OrderServiceTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();  // 직접 정리
    }

    @Test
    void 주문_생성_후_커밋_확인() {
        Order result = orderService.place(aCommand().withUserId(1L).build());

        // 실제 커밋된 데이터를 별도 조회
        Order found = orderRepository.findById(result.id()).orElseThrow();
        assertThat(found.status()).isEqualTo(PENDING);
        assertThat(found.userId()).isEqualTo(1L);
    }

    @Test
    void 이벤트_리스너가_호출된다() {
        orderService.place(aCommand().build());

        // 커밋이 실제로 발생했으므로 이벤트 리스너가 동작
        verify(emailService).sendOrderConfirmation(any());
    }
}
```

### 방법 2: @Transactional이 필요한 곳과 아닌 곳을 분리

```java
@SpringBootTest
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    // Repository 조회 테스트는 @Transactional 없어도 됨
    @Test
    void 상태별_조회() {
        // 저장은 @Transactional 없이도 됨 (Repository.save()가 트랜잭션 포함)
        orderRepository.save(anOrder().withStatus(PENDING).build());
        orderRepository.save(anOrder().withStatus(COMPLETED).build());

        // 조회
        List<Order> pending = orderRepository.findByStatus(PENDING);
        assertThat(pending).hasSize(1);
    }
}
```

### 방법 3: @Transactional을 테스트 메서드 레벨에서 선택적 적용

```java
@SpringBootTest
class OrderServiceTest {

    @Test
    // @Transactional 없음 — 커밋 검증
    void 주문_생성_커밋() {
        orderService.place(aCommand().build());
        assertThat(orderRepository.count()).isEqualTo(1);
    }

    @Test
    @Transactional  // 롤백 — 단순 동작 검증
    void 주문_객체_상태_확인() {
        Order order = orderService.place(aCommand().build());
        assertThat(order.status()).isEqualTo(PENDING);
    }
}
```

---

## 💻 실전 적용: 트랜잭션 전파 검증

```java
// 실제로 REQUIRES_NEW가 동작하는지 검증
@Service
public class AuditService {

    @Transactional(propagation = REQUIRES_NEW)
    public void log(String action, Long userId) {
        auditRepository.save(new AuditLog(action, userId));
        // 외부 트랜잭션이 롤백되어도 로그는 남아야 함
    }
}
```

```java
@SpringBootTest
class AuditServiceTest {

    @Autowired AuditService auditService;
    @Autowired AuditRepository auditRepository;

    @BeforeEach
    void setUp() {
        auditRepository.deleteAll();
    }

    @Test
    void 외부_트랜잭션_롤백_시_로그는_유지된다() {
        // @Transactional 없이 실행 — 실제 커밋/롤백 검증 가능
        try {
            // 외부 트랜잭션에서 예외 발생
            orderService.placeWithAudit(invalidCommand()); // 내부에서 auditService.log() 호출 후 실패
        } catch (Exception ignored) {}

        // REQUIRES_NEW로 별도 커밋된 로그가 남아있어야 함
        assertThat(auditRepository.count()).isEqualTo(1);
    }
}
```

---

## 🤔 트레이드오프

### "@Transactional을 제거하면 테스트 후 데이터 정리가 번거롭다"

맞습니다. `@BeforeEach`에서 `deleteAll()`을 호출하거나, Testcontainers의 `withReuse`와 조합해서 격리 전략을 별도로 관리해야 합니다. 번거롭지만 이것이 실제 동작을 검증한다는 의미입니다.

### "@DataJpaTest에서 @Transactional 롤백은 괜찮은가?"

Repository 레이어의 CRUD 동작 자체를 검증하는 경우엔 괜찮습니다. `save()` → `findById()` 흐름이 올바른지, JPA 쿼리가 올바른지 확인하는 용도라면 롤백이 문제되지 않습니다. 문제가 되는 것은 Service 레이어의 트랜잭션 경계를 `@Transactional` 롤백으로 검증하려 할 때입니다.

---

## 📌 핵심 정리

```
@Transactional 테스트의 함정:
  트랜잭션 전파 숨김 (REQUIRES_NEW도 흡수됨)
  LazyLoading 예외 감춤 (같은 트랜잭션이라 로딩됨)
  @TransactionalEventListener 미동작 (커밋 안 함)

실제 커밋 검증이 필요한 경우:
  @TransactionalEventListener 동작 검증
  REQUIRES_NEW 전파 검증
  LazyLoading 예외 검증
  실제 DB 제약조건 위반 검증

대안:
  테스트에서 @Transactional 제거 + @BeforeEach 정리
  @Commit으로 커밋 후 @AfterEach 정리
  테스트 메서드별 선택적 @Transactional

@DataJpaTest에서 @Transactional:
  Repository 동작 검증 → 롤백 OK
  Service 트랜잭션 경계 검증 → 롤백 사용 주의

판단 기준:
  "이 테스트가 커밋 후 동작에 의존하는가?"
  → YES: @Transactional 제거
  → NO: @Transactional 유지 가능
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트가 항상 통과하는데, 실제 서비스에서는 `LazyInitializationException`이 발생한다. 원인이 무엇이고, 테스트를 어떻게 수정해야 이 버그를 잡을 수 있는가?

```java
@SpringBootTest
@Transactional
class OrderServiceTest {

    @Test
    void 주문_아이템_조회() {
        Order order = orderService.place(aCommand().withItems(3).build());
        assertThat(order.items()).hasSize(3); // 항상 통과
    }
}

// 실제 코드
@Transactional(readOnly = true)
public List<OrderItem> getItems(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    return order.items(); // 실제로는 여기서 LazyInitializationException
}
```

**Q2.** `@TransactionalEventListener(phase = AFTER_COMMIT)`을 테스트하려면 어떻게 해야 하는가? `@Transactional`이 붙은 테스트에서 이 리스너가 호출되지 않는 이유를 설명하라.

**Q3.** 아래 시나리오에서 `@Transactional` 테스트가 적절한 경우와 그렇지 않은 경우를 구분하라.

```
① OrderRepository.findByStatus()가 올바른 결과를 반환하는가
② OrderService.place() 실행 후 이메일이 발송되는가
③ 동시 요청 시 재고가 음수가 되지 않는가
④ JPA Auditing(@CreatedDate)이 올바르게 동작하는가
⑤ REQUIRES_NEW로 분리된 AuditLog가 롤백 시에도 남는가
```

> 💡 **해설**

**Q1.**

원인: 테스트의 `@Transactional`이 `orderService.place()` 내부의 `@Transactional`을 하나의 트랜잭션으로 합쳐버린다. 테스트 전체가 하나의 트랜잭션이므로 `order.items()` 접근 시 트랜잭션이 열려 있어 Lazy 로딩이 성공한다.

실제 서비스에서 `getItems()` 호출 시에는 `orderService.place()`의 트랜잭션이 이미 종료된 상태이고, `getItems()`의 `@Transactional(readOnly = true)`가 새 트랜잭션을 시작해도 `place()`가 반환한 `order` 객체는 이 트랜잭션과 연결이 없어 Lazy 로딩에 실패한다.

수정:

```java
@SpringBootTest
// @Transactional 제거
class OrderServiceTest {

    @BeforeEach
    void setUp() { orderRepository.deleteAll(); }

    @Test
    void 주문_아이템_조회() {
        Order placed = orderService.place(aCommand().withItems(3).build());

        // 실제 getItems() 메서드를 통해 별도 트랜잭션으로 조회
        // 이렇게 해야 LazyInitializationException을 잡을 수 있음
        List<OrderItem> items = orderService.getItems(placed.id());
        assertThat(items).hasSize(3);
    }
}
```

**Q2.**

원인: `@TransactionalEventListener(phase = AFTER_COMMIT)`은 트랜잭션이 실제로 커밋된 후에 실행된다. 테스트에 `@Transactional`이 붙으면 테스트 종료 시 롤백이 발생하고, `AFTER_COMMIT` 이벤트는 발행되지 않는다.

테스트 방법:

```java
@SpringBootTest
// @Transactional 없음
class OrderEventTest {

    @Autowired OrderService orderService;
    @MockBean EmailService emailService;

    @BeforeEach
    void setUp() { orderRepository.deleteAll(); }

    @Test
    void 주문_생성_시_이메일_발송() {
        orderService.place(aCommand().build());
        // 실제 커밋이 발생했으므로 AFTER_COMMIT 리스너가 동작
        verify(emailService).sendOrderConfirmation(any());
    }
}
```

`EmailService`를 `@MockBean`으로 등록해서 실제 이메일 발송 없이 호출 여부만 검증한다.

**Q3.**

① `findByStatus()` 결과 → **@Transactional 적절.** Repository 쿼리 동작 검증은 롤백으로 충분하다. 실제 커밋이 필요하지 않다.

② `place()` 후 이메일 발송 → **@Transactional 부적절.** `@TransactionalEventListener(AFTER_COMMIT)`이 관여하므로 실제 커밋이 필요하다.

③ 동시 요청 재고 → **@Transactional 부적절.** 동시성 테스트는 여러 스레드/트랜잭션이 실제로 경합해야 한다. 하나의 테스트 트랜잭션으로는 동시성 시나리오를 재현할 수 없다.

④ `@CreatedDate` Auditing → **@Transactional 적절.** 저장 시 자동 채워지는지는 롤백 여부와 무관하다. `entityManager.flush()`로 실제 INSERT 후 확인.

⑤ `REQUIRES_NEW` AuditLog → **@Transactional 부적절.** 외부 트랜잭션 롤백 시 REQUIRES_NEW로 분리된 내부 트랜잭션이 살아남는지 검증하려면 실제 롤백/커밋이 발생해야 한다.

---

<div align="center">

**[⬅️ 이전: Spring Context Slicing](./04-spring-context-slicing.md)** | **[다음: Contract Testing ➡️](./06-contract-testing.md)**

</div>
