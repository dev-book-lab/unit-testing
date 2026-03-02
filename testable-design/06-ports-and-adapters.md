# 06. Ports and Adapters

> **테스트 경계는 아키텍처 경계와 일치해야 한다 — Hexagonal Architecture는 테스트 가능한 설계의 자연스러운 귀결이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Ports and Adapters(Hexagonal Architecture)가 테스트 경계를 어떻게 명확하게 만드는가?
- Port와 Adapter는 각각 무엇이고, 어떤 것을 테스트해야 하는가?
- 이 아키텍처에서 각 레이어별 테스트 전략은 어떻게 되는가?

---

## 🔍 기존 레이어드 아키텍처의 테스트 경계 문제

전통적인 레이어드 아키텍처에서 테스트 경계는 모호합니다.

```
Controller → Service → Repository → DB
```

```java
// Service를 테스트하려면:
// - Repository를 Mock해야 한다 (어느 메서드를? 어디까지?)
// - Controller에서 넘어오는 DTO 변환은 어디서 테스트하는가?
// - DB 연결은 Repository 테스트에서? 통합 테스트에서?
```

각 레이어가 어디서 끝나고 어디서 시작하는지, 어떤 것을 단위 테스트하고 통합 테스트하는지 명확한 기준이 없습니다.

---

## 🏛️ Ports and Adapters의 구조

Alistair Cockburn이 정의한 이 패턴은 애플리케이션을 세 영역으로 나눕니다.

```
  ┌─────────────────────────────────────────────────────────┐
  │  Driving Adapters (Inbound)                             │
  │  HTTP Controller, Kafka Consumer, CLI                   │
  └────────────────────────┬────────────────────────────────┘
                           │ uses
  ┌────────────────────────▼────────────────────────────────┐
  │  Inbound Ports (Driving)                                │
  │  OrderUseCase, UserUseCase (인터페이스)                  │
  └────────────────────────┬────────────────────────────────┘
                           │ implements
  ┌────────────────────────▼────────────────────────────────┐
  │  Application Core                                       │
  │  OrderService, UserService (비즈니스 로직)               │
  └────────────────────────┬────────────────────────────────┘
                           │ uses
  ┌────────────────────────▼────────────────────────────────┐
  │  Outbound Ports (Driven)                                │
  │  OrderRepository, EventPublisher (인터페이스)            │
  └────────────────────────┬────────────────────────────────┘
                           │ implements
  ┌────────────────────────▼────────────────────────────────┐
  │  Driven Adapters (Outbound)                             │
  │  JpaOrderRepository, KafkaEventPublisher                │
  └─────────────────────────────────────────────────────────┘
```

핵심은 **Application Core가 어느 방향에서도 인터페이스(Port)에만 의존**한다는 것입니다. 구체적인 기술(JPA, Kafka, HTTP)은 Adapter에만 있습니다.

---

## 🏛️ Port와 Adapter를 코드로 표현하기

### Inbound Port — 애플리케이션이 외부에게 제공하는 기능

```java
// 인바운드 포트: "이 애플리케이션이 무엇을 할 수 있는가"
public interface PlaceOrderUseCase {
    Order place(PlaceOrderCommand command);
}

public interface CancelOrderUseCase {
    void cancel(Long orderId, String reason);
}
```

### Application Core — 비즈니스 로직

```java
// 인바운드 포트를 구현, 아웃바운드 포트에 의존
@Service
public class OrderService implements PlaceOrderUseCase, CancelOrderUseCase {

    // 아웃바운드 포트 (인터페이스)에만 의존 — JPA, Kafka를 모름
    private final OrderRepository orderRepository;     // Outbound Port
    private final InventoryPort inventoryPort;         // Outbound Port
    private final OrderEventPublisher eventPublisher;  // Outbound Port

    @Override
    public Order place(PlaceOrderCommand command) {
        inventoryPort.reserve(command.items());
        Order order = Order.from(command);
        Order saved = orderRepository.save(order);
        eventPublisher.orderPlaced(saved);
        return saved;
    }

    @Override
    public void cancel(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        Order cancelled = order.cancel(reason);
        orderRepository.save(cancelled);
        inventoryPort.release(cancelled.items());
        eventPublisher.orderCancelled(cancelled);
    }
}
```

### Outbound Port — 애플리케이션이 외부에게 요청하는 것

```java
// 아웃바운드 포트: "애플리케이션이 무엇을 필요로 하는가"
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}

public interface InventoryPort {
    void reserve(List<OrderItem> items);
    void release(List<OrderItem> items);
}

public interface OrderEventPublisher {
    void orderPlaced(Order order);
    void orderCancelled(Order order);
}
```

### Driving Adapter — 인바운드 포트를 호출하는 외부

```java
// HTTP Adapter: Controller
@RestController
public class OrderController {
    private final PlaceOrderUseCase placeOrderUseCase; // 인바운드 포트

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> place(@RequestBody PlaceOrderRequest req) {
        PlaceOrderCommand command = req.toCommand();  // HTTP → 도메인 변환
        Order order = placeOrderUseCase.place(command);
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}

// Kafka Consumer Adapter
@KafkaListener(topics = "payment-completed")
public class PaymentCompletedAdapter {
    private final PlaceOrderUseCase placeOrderUseCase;

    public void consume(PaymentCompletedEvent event) {
        PlaceOrderCommand command = event.toCommand(); // 이벤트 → 도메인 변환
        placeOrderUseCase.place(command);
    }
}
```

### Driven Adapter — 아웃바운드 포트를 구현하는 외부

```java
// JPA Adapter
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final JpaOrderJpaRepository jpa;

    @Override
    public Order save(Order order) {
        return jpa.save(OrderEntity.from(order)).toDomain();
    }

    @Override
    public Optional<Order> findById(Long id) {
        return jpa.findById(id).map(OrderEntity::toDomain);
    }
}

// Kafka Adapter
@Component
public class KafkaOrderEventPublisher implements OrderEventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Override
    public void orderPlaced(Order order) {
        kafkaTemplate.send("order-placed", OrderPlacedEvent.from(order));
    }
}
```

---

## ✨ 레이어별 테스트 전략

### 1. Application Core 단위 테스트 — 가장 빠르고 많음

```java
class OrderServiceTest {

    // Fake: 상태 연결이 필요
    private final InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();

    // Stub: 재고 확인 결과 제어
    private final InventoryPort stubInventory = items -> {}; // 항상 성공

    // Mock: 이벤트 발행 행동 검증
    private final OrderEventPublisher mockPublisher = mock(OrderEventPublisher.class);

    private final OrderService service =
        new OrderService(fakeRepo, stubInventory, mockPublisher);

    @Test
    void 주문_생성_시_저장되고_이벤트가_발행된다() {
        PlaceOrderCommand command = aCommand().withUserId(1L).withPrice(20_000).build();

        Order result = service.place(command);

        assertThat(fakeRepo.findById(result.id())).isPresent();
        verify(mockPublisher).orderPlaced(argThat(o -> o.id().equals(result.id())));
    }

    @Test
    void 재고_부족_시_주문이_실패한다() {
        InventoryPort outOfStock = items -> { throw new OutOfStockException(); };
        OrderService failingService = new OrderService(fakeRepo, outOfStock, mockPublisher);

        assertThatThrownBy(() -> failingService.place(aCommand().build()))
            .isInstanceOf(OutOfStockException.class);
    }
}
```

### 2. Driving Adapter 테스트 — HTTP 변환 검증

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean PlaceOrderUseCase placeOrderUseCase; // 인바운드 포트 Mock

    @Test
    void 주문_성공_시_201_Created를_반환한다() throws Exception {
        when(placeOrderUseCase.place(any())).thenReturn(sampleOrder);

        mockMvc.perform(post("/orders")
                .contentType(APPLICATION_JSON)
                .content("""
                    {"userId": 1, "items": [{"productId": 10, "quantity": 1}]}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").isNotEmpty());
    }

    @Test
    void 재고_부족_시_409_Conflict를_반환한다() throws Exception {
        when(placeOrderUseCase.place(any()))
            .thenThrow(new OutOfStockException());

        mockMvc.perform(post("/orders").contentType(APPLICATION_JSON).content("{}"))
            .andExpect(status().isConflict());
    }
}
```

### 3. Driven Adapter 테스트 — 외부 기술 연동 검증

```java
// JPA Adapter: 실제 DB로 매핑 검증
@DataJpaTest
class JpaOrderRepositoryTest {

    @Autowired JpaOrderRepository repository;

    @Test
    void 주문_저장_후_ID로_조회할_수_있다() {
        Order order = anOrder().build();
        Order saved = repository.save(order);

        assertThat(repository.findById(saved.id()))
            .isPresent()
            .hasValueSatisfying(o -> assertThat(o.totalPrice()).isEqualTo(order.totalPrice()));
    }
}

// Kafka Adapter: 실제 메시지 발행 검증 (@EmbeddedKafka)
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = "order-placed")
class KafkaOrderEventPublisherTest {
    // 메시지가 올바른 형식으로 발행되는지 검증
}
```

---

## 💻 실전 적용: 패키지 구조

```
com.example.order/
  ├── application/
  │   ├── port/
  │   │   ├── in/                      ← Inbound Ports
  │   │   │   ├── PlaceOrderUseCase.java
  │   │   │   └── CancelOrderUseCase.java
  │   │   └── out/                     ← Outbound Ports
  │   │       ├── OrderRepository.java
  │   │       ├── InventoryPort.java
  │   │       └── OrderEventPublisher.java
  │   └── service/                     ← Application Core
  │       └── OrderService.java
  ├── adapter/
  │   ├── in/                          ← Driving Adapters
  │   │   ├── web/
  │   │   │   └── OrderController.java
  │   │   └── kafka/
  │   │       └── PaymentCompletedAdapter.java
  │   └── out/                         ← Driven Adapters
  │       ├── persistence/
  │       │   ├── JpaOrderRepository.java
  │       │   └── OrderEntity.java
  │       └── messaging/
  │           └── KafkaOrderEventPublisher.java
  └── domain/
      └── Order.java                   ← 순수 도메인 객체
```

---

## 🤔 트레이드오프

### "Hexagonal Architecture는 클래스가 너무 많아지지 않는가?"

맞습니다. 각 기능마다 UseCase 인터페이스, Service 구현, 여러 Adapter, Entity 변환 객체가 생깁니다. 작은 프로젝트에는 과도할 수 있습니다. 하지만 팀이 커지고 서비스가 복잡해질수록 이 명확한 경계가 큰 가치를 발휘합니다.

단순한 CRUD 위주의 서비스라면 레이어드 아키텍처가 더 적합할 수 있습니다.

### "UseCase 인터페이스가 꼭 필요한가? Service 클래스만으로 충분하지 않은가?"

Controller가 Service 구체 클래스에 직접 의존하면, Controller 단위 테스트에서 Service를 Mock하기 위해 `@MockBean`을 쓰게 됩니다. UseCase 인터페이스가 있으면 `@MockBean` 없이도 인터페이스를 구현한 Stub으로 Controller를 테스트할 수 있고, Service와 Controller의 변경이 서로에게 영향을 덜 미칩니다.

---

## 📌 핵심 정리

```
Ports and Adapters의 세 영역:
  Application Core: 비즈니스 로직 (인터페이스만 앎)
  Inbound Port: Core가 외부에 제공하는 기능
  Outbound Port: Core가 외부에 요청하는 것
  Adapter: 구체 기술 구현 (HTTP, JPA, Kafka)

Core의 특성:
  어떤 프레임워크도 모름
  Spring, JPA, Kafka 의존 없음
  순수 Java 단위 테스트 가능

레이어별 테스트:
  Core → Mock/Fake로 빠른 단위 테스트 (많음)
  Driving Adapter → @WebMvcTest, HTTP 변환 검증 (소수)
  Driven Adapter → @DataJpaTest, 기술 연동 검증 (소수)

테스트 경계 = 아키텍처 경계:
  Port가 Mock/Stub의 경계가 됨
  Adapter는 통합 테스트 대상
  Core는 단위 테스트 대상
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 레이어드 아키텍처 코드를 Ports and Adapters 구조로 리팩터링하라. 각 클래스가 어느 영역(Core, Port, Adapter)에 속하는지 명시하라.

```java
@Service
public class NotificationService {
    @Autowired UserRepository userRepository;       // JPA Repository
    @Autowired JavaMailSender mailSender;           // Spring Mail
    @Autowired SmsApiClient smsApiClient;           // 외부 SMS API

    public void notifyOrderPlaced(Long orderId, Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        String emailBody = "주문 " + orderId + "이 접수됐습니다.";
        mailSender.send(createMessage(user.email(), emailBody));
        smsApiClient.send(user.phone(), "주문이 접수됐습니다.");
    }
}
```

**Q2.** Ports and Adapters에서 Application Core가 Spring의 `@Transactional`을 사용해야 할 경우, 이것이 아키텍처 원칙(Core는 프레임워크를 모름)과 충돌하는가? 어떻게 해결하는가?

**Q3.** 같은 비즈니스 로직을 HTTP API와 Kafka 이벤트 두 가지 인바운드로 처리해야 한다. Ports and Adapters에서 코드 중복 없이 어떻게 구현하는가?

> 💡 **해설**

**Q1.**

분리 결과:

**Inbound Port**
```java
interface NotifyOrderPlacedUseCase {
    void notify(Long orderId, Long userId);
}
```

**Outbound Ports**
```java
interface UserFinder       { Optional<User> findById(Long id); }
interface EmailNotifier    { void sendEmail(String to, String body); }
interface SmsNotifier      { void sendSms(String phone, String message); }
```

**Application Core**
```java
class NotificationService implements NotifyOrderPlacedUseCase {
    private final UserFinder userFinder;
    private final EmailNotifier emailNotifier;
    private final SmsNotifier smsNotifier;

    public void notify(Long orderId, Long userId) {
        User user = userFinder.findById(userId).orElseThrow();
        emailNotifier.sendEmail(user.email(), "주문 " + orderId + "이 접수됐습니다.");
        smsNotifier.sendSms(user.phone(), "주문이 접수됐습니다.");
    }
}
```

**Driven Adapters**
```
JpaUserRepository       implements UserFinder
JavaMailEmailNotifier   implements EmailNotifier
SmsApiClientAdapter     implements SmsNotifier
```

Core는 Spring, `JavaMailSender`, `SmsApiClient`를 모른다. 단위 테스트에서 `UserFinder`, `EmailNotifier`, `SmsNotifier`를 람다 Stub으로 즉시 교체 가능.

**Q2.**

충돌이 발생한다. `@Transactional`은 Spring AOP 의존이고, Core가 Spring을 알게 된다.

해결 방법:

① **Facade 패턴** — `@Transactional`을 가진 별도 Application Service가 Core UseCase를 감싼다.

```java
@Service
@Transactional
class TransactionalOrderFacade {
    private final PlaceOrderUseCase useCase;

    public Order place(PlaceOrderCommand command) {
        return useCase.place(command);
    }
}
```

Core는 `@Transactional` 없이 순수하게 유지.

② **현실적 타협** — Core에 `@Transactional`을 허용하되, 이것이 아키텍처 원칙의 예외임을 팀에서 인식한다. 많은 프로젝트에서 ②를 선택하는데, 완벽한 순수성보다 실용성이 중요할 때 합리적인 선택이다.

**Q3.**

Inbound Port 하나, Adapter 두 개로 구현한다.

```java
interface PlaceOrderUseCase {
    Order place(PlaceOrderCommand command);
}
```

```java
// HTTP Adapter
@PostMapping("/orders")
public ResponseEntity<?> placeViaHttp(@RequestBody PlaceOrderRequest req) {
    Order order = placeOrderUseCase.place(req.toCommand());
    return ResponseEntity.ok(OrderResponse.from(order));
}

// Kafka Adapter
@KafkaListener(topics = "payment-completed")
public void placeViaKafka(PaymentCompletedEvent event) {
    placeOrderUseCase.place(event.toCommand());
}
```

비즈니스 로직(`OrderService.place()`)은 한 곳에만 있다. HTTP와 Kafka는 각자의 변환(`toCommand()`)만 담당한다. 인바운드 수단이 추가돼도 Core 코드는 변경 없다.

---

<div align="center">

**[⬅️ 이전: Pure Functions First](./05-pure-functions-first.md)** | **[홈으로 🏠](../README.md)**

</div>
