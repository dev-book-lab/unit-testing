# 01. Dependency Injection for Testability

> **테스트하기 어렵다는 느낌은 설계 냄새다 — `new`로 의존성을 직접 생성하는 순간 테스트 가능성이 사라진다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 생성자 주입과 필드 주입의 테스트 비용은 왜 다른가?
- `new`로 의존성을 직접 생성하는 코드를 어떻게 테스트 가능하게 바꾸는가?
- DI가 없는 코드를 테스트하기 위해 PowerMock 같은 도구를 쓰는 것은 왜 나쁜 선택인가?

---

## 🔍 의존성 주입이 없으면 무슨 일이 생기는가

```java
public class OrderService {

    public Order place(Cart cart, User user) {
        // 의존성을 내부에서 직접 생성
        InventoryClient inventoryClient = new HttpInventoryClient();        // 외부 API
        OrderRepository repository = new JpaOrderRepository();              // DB
        EventPublisher eventPublisher = new KafkaEventPublisher();          // Kafka
        DiscountPolicy discountPolicy = new RateDiscountPolicy(10);

        inventoryClient.reserve(cart);
        Order order = Order.from(cart, user, discountPolicy.calculate(user));
        repository.save(order);
        eventPublisher.publish(new OrderPlaced(order));
        return order;
    }
}
```

이 코드를 단위 테스트하려면 어떻게 해야 할까요?

```java
@Test
void VIP_회원_주문_시_할인이_적용된다() {
    OrderService service = new OrderService();
    service.place(cart, vipUser); // ← 실제 HTTP 호출, 실제 DB, 실제 Kafka가 발생한다
}
```

`HttpInventoryClient`가 실제 재고 서버에 연결을 시도합니다. `JpaOrderRepository`가 실제 DB를 찾습니다. 테스트 환경에 이것들이 없으면 예외가 발생합니다. 있다면 테스트가 외부 시스템 상태에 의존하게 됩니다.

**`new`로 의존성을 생성하는 순간, 그 의존성을 교체할 방법이 없어집니다.**

---

## 🏛️ 생성자 주입 vs 필드 주입의 테스트 비용

### 필드 주입 (Field Injection)

```java
@Service
public class OrderService {

    @Autowired
    private InventoryClient inventoryClient;   // Spring이 주입

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private EventPublisher eventPublisher;

    public Order place(Cart cart, User user) { ... }
}
```

Spring 컨텍스트 없이 테스트하려면:

```java
@Test
void 할인_적용_테스트() {
    OrderService service = new OrderService();
    // inventoryClient, orderRepository, eventPublisher는 모두 null
    // place()를 호출하면 NullPointerException
}
```

필드 주입은 Java 기본 문법으로 의존성을 설정할 방법이 없습니다. `@SpringBootTest`를 써서 전체 컨텍스트를 올리거나, 리플렉션으로 강제 주입하거나(`ReflectionTestUtils.setField()`), `@ExtendWith(MockitoExtension.class)` + `@InjectMocks`에 의존해야 합니다.

모두 비용이 있습니다. 컨텍스트 로딩은 느리고, 리플렉션은 컴파일 타임 안전성이 없고, `@InjectMocks`는 앞서 본 함정이 있습니다.

### 생성자 주입 (Constructor Injection)

```java
@Service
public class OrderService {

    private final InventoryClient inventoryClient;
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;
    private final DiscountPolicy discountPolicy;

    // Spring이 생성자를 호출해서 주입
    public OrderService(InventoryClient inventoryClient,
                        OrderRepository orderRepository,
                        EventPublisher eventPublisher,
                        DiscountPolicy discountPolicy) {
        this.inventoryClient = inventoryClient;
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
        this.discountPolicy = discountPolicy;
    }

    public Order place(Cart cart, User user) { ... }
}
```

테스트에서는 Spring 없이 직접 생성합니다:

```java
class OrderServiceTest {

    private final InventoryClient inventoryStub = cart -> {}; // 재고 항상 성공
    private final InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
    private final EventPublisher mockPublisher = mock(EventPublisher.class);
    private final DiscountPolicy vipPolicy = user -> 10;

    private final OrderService service =
        new OrderService(inventoryStub, fakeRepo, mockPublisher, vipPolicy);

    @Test
    void VIP_회원_주문_시_10퍼센트_할인이_적용된다() {
        Order order = service.place(cart, vipUser);
        assertThat(order.totalPrice()).isEqualTo(18_000);
    }
}
```

Spring, 리플렉션, `@InjectMocks` 없이 일반 Java 코드로 테스트가 완성됩니다.

---

## 😱 내부에서 new를 쓰는 패턴과 해결

### 패턴 1: 메서드 내부에서 직접 생성

```java
// ❌
public PaymentResult pay(Order order) {
    PaymentGateway gateway = new TossPaymentGateway(API_KEY); // 교체 불가
    return gateway.charge(order.totalPrice());
}

// ✅ 생성자로 주입받는다
public class PaymentService {
    private final PaymentGateway gateway;

    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public PaymentResult pay(Order order) {
        return gateway.charge(order.totalPrice());
    }
}
```

### 패턴 2: `LocalDate.now()` 같은 시간 의존성

```java
// ❌ 현재 시간을 직접 사용 — Repeatable 원칙 위반
public boolean isCouponExpired(Coupon coupon) {
    return coupon.expiresAt().isBefore(LocalDate.now());
}

// ✅ Clock을 주입받는다
public class CouponValidator {
    private final Clock clock;

    public CouponValidator(Clock clock) {
        this.clock = clock;
    }

    public boolean isExpired(Coupon coupon) {
        return coupon.expiresAt().isBefore(LocalDate.now(clock));
    }
}

// 테스트
@Test
void 만료일이_지난_쿠폰은_유효하지_않다() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-06-01T00:00:00Z"), ZoneOffset.UTC);
    CouponValidator validator = new CouponValidator(fixedClock);

    Coupon expiredCoupon = aCoupon()
        .withExpiresAt(LocalDate.of(2024, 5, 31))
        .build();

    assertThat(validator.isExpired(expiredCoupon)).isTrue();
}
```

### 패턴 3: static 팩토리 메서드 호출

```java
// ❌ 정적 메서드 — 교체 불가 (다음 문서에서 자세히)
public class OrderService {
    public Order place(Cart cart, User user) {
        String traceId = TraceContext.currentTraceId(); // static 호출
        ...
    }
}
```

---

## ✨ 생성자 주입이 설계를 개선하는 방식

생성자 주입은 단순히 테스트를 위한 기법이 아닙니다. 의존성이 명시적으로 드러나기 때문에 설계 문제를 빨리 발견할 수 있습니다.

```java
// 생성자 파라미터가 7개라면?
public OrderService(
    InventoryClient inventoryClient,
    OrderRepository orderRepository,
    EventPublisher eventPublisher,
    DiscountPolicy discountPolicy,
    ShippingCalculator shippingCalculator,
    CouponValidator couponValidator,
    AuditLogger auditLogger   // 7번째
) { ... }
```

생성자 파라미터가 많아지면 불편합니다. 이 불편함이 "이 클래스가 너무 많은 책임을 갖고 있다"는 신호입니다. 필드 주입이라면 `@Autowired`만 추가하면 되어 이 신호를 놓치게 됩니다.

**테스트하기 불편하다는 느낌이 설계를 개선하게 만드는 것입니다.**

---

## 💻 실전 적용: Spring과 생성자 주입

```java
// @RequiredArgsConstructor로 보일러플레이트 제거
@Service
@RequiredArgsConstructor  // final 필드의 생성자를 Lombok이 생성
public class OrderService {

    private final InventoryClient inventoryClient;
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;
    private final DiscountPolicy discountPolicy;

    public Order place(Cart cart, User user) { ... }
}
```

Spring 4.3+부터는 생성자가 하나면 `@Autowired` 없이도 자동으로 생성자 주입이 됩니다. `@RequiredArgsConstructor`로 생성자 코드를 제거하고, 테스트에서는 생성자를 직접 호출합니다.

---

## 🤔 트레이드오프

### "생성자 주입으로 하면 테스트 코드에서 의존성을 너무 많이 설정해야 한다"

맞습니다. 하지만 이것은 의존성 수가 많다는 신호이지, 생성자 주입의 문제가 아닙니다. Test Data Builder와 `@BeforeEach`를 활용해서 반복을 줄이고, 클래스 분리를 고려합니다.

### "기존 코드가 모두 필드 주입이다. 다 바꿔야 하는가?"

한 번에 바꿀 필요는 없습니다. 새로 작성하는 코드부터 생성자 주입을 적용하고, 테스트를 추가하면서 점진적으로 리팩터링하는 것이 현실적입니다.

---

## 📌 핵심 정리

```
new로 의존성을 생성하면:
  교체 불가 → 테스트 불가
  외부 시스템에 묶임 → Fast, Repeatable 위반

필드 주입의 테스트 비용:
  Spring 컨텍스트 없이 null
  리플렉션 또는 @InjectMocks에 의존
  함정 있음, 느림

생성자 주입의 테스트 이점:
  Spring 없이 일반 Java로 객체 생성
  의존성이 명시적 — 설계 문제를 드러냄
  컴파일 타임 안전성

주입해야 하는 것들:
  DB/API 등 외부 시스템 (Repository, Client)
  현재 시간 (Clock)
  정책/전략 (DiscountPolicy, ShippingCalculator)

생성자 파라미터 > 5개:
  클래스 분리 신호
  필드 주입이었다면 보이지 않았을 신호
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드를 테스트하려면 무엇이 문제인가? 생성자 주입으로 리팩터링하고, 리팩터링 후 테스트 코드를 작성하라.

```java
public class ReportService {

    public String generateMonthlyReport(int year, int month) {
        ReportRepository repo = new JdbcReportRepository(); // 직접 생성
        List<Sale> sales = repo.findByYearMonth(year, month);
        return ReportFormatter.format(sales); // static 메서드
    }
}
```

**Q2.** `@SpringBootTest`를 이용한 통합 테스트와 생성자 주입 기반의 단위 테스트 중 어느 것이 더 좋은가? 두 접근의 차이와 어떤 상황에서 각각을 선택하는지 설명하라.

**Q3.** 생성자가 7개의 파라미터를 받는 `UserService`가 있다. 이 신호를 어떻게 해석하고, 어떻게 리팩터링하겠는가?

> 💡 **해설**
>
> **Q1.** 문제: ① `new JdbcReportRepository()` — 실제 DB 연결을 시도한다. 테스트에서 교체할 방법이 없다. ② `ReportFormatter.format(sales)` — static 메서드라 교체 불가 (다음 문서 주제). 리팩터링: `public class ReportService { private final ReportRepository repository; private final ReportFormatter formatter; public ReportService(ReportRepository repository, ReportFormatter formatter) { ... } public String generateMonthlyReport(int year, int month) { return formatter.format(repository.findByYearMonth(year, month)); } }`. 테스트: `ReportRepository fakeRepo = (y, m) -> List.of(new Sale(...)); ReportFormatter formatter = new ReportFormatter(); ReportService service = new ReportService(fakeRepo, formatter); assertThat(service.generateMonthlyReport(2024, 6)).contains("2024년 6월");`
>
> **Q2.** `@SpringBootTest`: Spring 전체 컨텍스트를 로딩하므로 수십 초 걸린다. 실제 빈 연결, 설정, DB 연결까지 검증된다. 통합 오류를 발견하기 좋지만 빠른 피드백이 필요한 비즈니스 로직 테스트에는 과도하다. 생성자 주입 단위 테스트: ms 단위, Spring 없음, 비즈니스 로직에 집중. 선택 기준: 비즈니스 규칙 검증 → 단위 테스트. Spring 빈 연결, 설정, DB 연결 검증 → `@SpringBootTest` 또는 슬라이스 테스트(`@WebMvcTest`, `@DataJpaTest`). 대부분의 테스트는 단위 테스트이고, 통합 테스트는 소수.
>
> **Q3.** 해석: 7개 파라미터는 `UserService`가 7가지 책임을 가진다는 신호다. 단일 책임 원칙(SRP) 위반 가능성이 높다. 리팩터링 방향: 책임을 묶어서 별도 서비스로 분리한다. 예: `UserRegistrationService(UserRepository, PasswordEncoder, EmailSender)` + `UserAuthService(UserRepository, TokenGenerator, SessionStore)` + `UserProfileService(UserRepository, ProfileRepository, ImageStorage)`. 분리 기준: 함께 변경되는 것끼리 묶는다. 7개 의존성 중 어떤 것들이 동시에 쓰이는지 확인하면 그룹이 보인다.

---

<div align="center">

**[다음: Avoiding Static Methods ➡️](./02-avoiding-static-methods.md)**

</div>
