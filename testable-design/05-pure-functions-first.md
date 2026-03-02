# 05. Pure Functions First

> **부수효과 없는 함수는 테스트하기 위해 아무것도 설정할 필요가 없다 — 입력을 넣고 출력을 확인하면 끝이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 순수 함수(Pure Function)란 무엇이고 왜 테스트하기 가장 쉬운가?
- 도메인 모델에서 순수한 로직과 부수효과를 어떻게 분리하는가?
- 순수하지 않은 코드를 순수하게 만드는 리팩터링 패턴은 무엇인가?

---

## 🔍 순수 함수란 무엇인가

순수 함수는 두 조건을 만족합니다.

1. **같은 입력에 항상 같은 출력** — 외부 상태에 의존하지 않음
2. **부수효과 없음** — 외부 상태를 변경하지 않음

```java
// ✅ 순수 함수: 입력만으로 출력이 결정됨
public static int applyDiscount(int price, int discountRate) {
    return price - (price * discountRate / 100);
}

// ✅ 순수 함수: 외부 상태 없음, 새 객체 반환
public Order withDiscount(int discountRate) {
    int discountedPrice = applyDiscount(this.totalPrice, discountRate);
    return new Order(this.id, this.userId, discountedPrice, this.status);
}

// ❌ 순수하지 않음: 외부 상태(DB) 조회
public int getDiscountRate(User user) {
    Grade grade = gradeRepository.findByUserId(user.id()); // 부수효과
    return grade == Grade.VIP ? 10 : 0;
}

// ❌ 순수하지 않음: 외부 상태 변경
public void applyAndSave(Order order, int rate) {
    order.setDiscountedPrice(order.totalPrice() * (1 - rate / 100.0));
    orderRepository.save(order); // 부수효과
}
```

---

## 🏛️ 순수 함수가 테스트하기 쉬운 이유

```java
// 순수 함수 테스트 — 준비물이 없다
@Test
void 10퍼센트_할인_적용() {
    int result = applyDiscount(20_000, 10);
    assertThat(result).isEqualTo(18_000);
}

// 부수효과가 있는 함수 테스트 — 준비물이 많다
@Test
void 할인_적용_후_저장() {
    // Arrange
    OrderRepository mockRepo = mock(OrderRepository.class);
    GradeRepository mockGradeRepo = mock(GradeRepository.class);
    when(mockGradeRepo.findByUserId(1L)).thenReturn(Grade.VIP);

    OrderService service = new OrderService(mockRepo, mockGradeRepo);
    Order order = new Order(1L, 1L, 20_000, PENDING);

    // Act
    service.applyAndSave(order, 1L);

    // Assert
    verify(mockRepo).save(argThat(o -> o.discountedPrice() == 18_000));
}
```

순수 함수 테스트는 Mock이 없습니다. `@BeforeEach`가 없습니다. 테스트 실행 환경에 아무 의존도 없습니다. 경계값 테스트를 3줄에 추가할 수 있습니다.

---

## 😱 도메인 로직이 부수효과와 섞이는 흔한 패턴

### 패턴 1: 도메인 객체가 Repository를 알고 있다

```java
// ❌ Order가 Repository에 의존 — 순수하지 않음
public class Order {
    @Autowired
    private OrderRepository orderRepository;  // 도메인 객체에 주입

    public void complete() {
        this.status = COMPLETED;
        this.completedAt = LocalDateTime.now();  // 현재 시간 의존
        orderRepository.save(this);              // 부수효과
    }
}

// 테스트: complete()를 호출하면 실제 DB 또는 Mock이 필요
```

```java
// ✅ 도메인 객체는 순수하게, 부수효과는 서비스가 담당
public class Order {
    public Order complete(Clock clock) {
        // 새 객체 반환 — 기존 객체 불변
        return new Order(this.id, this.userId, this.totalPrice,
                COMPLETED, LocalDateTime.now(clock));
    }
}

// 서비스: 도메인 로직 호출 + 부수효과 처리
public class OrderService {
    public void complete(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        Order completed = order.complete(clock);  // 순수 함수 호출
        orderRepository.save(completed);          // 부수효과
    }
}

// 도메인 로직 단위 테스트 — Mock 없음
@Test
void 주문_완료_시_상태가_COMPLETED로_변경된다() {
    Clock fixedClock = Clock.fixed(Instant.parse("2024-06-01T12:00:00Z"), ZoneOffset.UTC);
    Order order = anOrder().withStatus(PENDING).build();

    Order completed = order.complete(fixedClock);

    assertThat(completed.status()).isEqualTo(COMPLETED);
    assertThat(completed.completedAt()).isEqualTo(LocalDateTime.of(2024, 6, 1, 12, 0));
}
```

### 패턴 2: 계산 로직과 저장 로직이 한 메서드에 있다

```java
// ❌ 계산 + 저장이 섞임
public void processMonthlyPoints(Long userId) {
    List<Order> orders = orderRepository.findByUserId(userId);  // I/O
    int totalPoints = orders.stream()
            .mapToInt(o -> o.totalPrice() / 100)                   // 계산
            .sum();
    if (totalPoints > 1000) totalPoints += 200;                 // 보너스 계산

    Point point = new Point(userId, totalPoints);
    pointRepository.save(point);                                // I/O
    eventPublisher.publish(new PointsEarned(userId, totalPoints)); // 부수효과
}
```

```java
// ✅ 계산 로직 추출 — 순수 함수
public static int calculateMonthlyPoints(List<Order> orders) {
    int base = orders.stream()
            .mapToInt(o -> o.totalPrice() / 100)
            .sum();
    return base > 1000 ? base + 200 : base; // 보너스 포함
}

// 서비스: I/O + 순수 함수 호출
public void processMonthlyPoints(Long userId) {
    List<Order> orders = orderRepository.findByUserId(userId);
    int totalPoints = calculateMonthlyPoints(orders);  // 순수 함수
    pointRepository.save(new Point(userId, totalPoints));
    eventPublisher.publish(new PointsEarned(userId, totalPoints));
}

// ✅ 계산 로직만 단위 테스트 — 경계값 포함
@Test
void 포인트_합계가_1000_이하면_보너스_없음() {
    List<Order> orders = List.of(anOrder().withPrice(50_000).build()); // 500포인트
    assertThat(calculateMonthlyPoints(orders)).isEqualTo(500);
}

@Test
void 포인트_합계가_1001_이상이면_200포인트_보너스() {
    List<Order> orders = List.of(anOrder().withPrice(110_000).build()); // 1100포인트
    assertThat(calculateMonthlyPoints(orders)).isEqualTo(1_300);
}
```

---

## ✨ 도메인 모델을 순수하게 유지하는 전략

### 전략 1: 불변 도메인 객체 (Immutable Domain Object)

```java
// 상태 변경 메서드가 새 객체를 반환한다
public final class Money {
    private final int amount;
    private final String currency;

    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new CurrencyMismatchException();
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money discountBy(int rate) {
        return new Money((int)(this.amount * (1 - rate / 100.0)), this.currency);
    }
}

// 테스트: 입력 → 출력만
@Test
void 10퍼센트_할인_적용() {
    Money price = new Money(20_000, "KRW");
    assertThat(price.discountBy(10)).isEqualTo(new Money(18_000, "KRW"));
}
```

### 전략 2: 함수형 코어, 명령형 쉘 (Functional Core, Imperative Shell)

전체 아키텍처를 두 레이어로 나눕니다.

```
  ┌─────────────────────────────────────────┐
  │   Imperative Shell (명령형 쉘)           │
  │   Service, Repository, Controller       │
  │   I/O, 부수효과, 상태 변경               │
  └────────────────┬────────────────────────┘
                   │ 순수 함수 호출
  ┌────────────────▼────────────────────────┐
  │   Functional Core (함수형 코어)          │
  │   Domain Object, Policy, Calculator     │
  │   순수 함수, 불변, 부수효과 없음          │
  └─────────────────────────────────────────┘
```

```java
// Functional Core: 모든 비즈니스 결정
public class OrderDecider {
    public static OrderDecision decide(
            Cart cart, User user, DiscountRule rule, Stock stock) {

        if (!stock.isAvailable(cart)) {
            return OrderDecision.reject("재고 부족");
        }
        int discount = rule.calculate(user);
        int finalPrice = cart.totalPrice() - discount;

        return OrderDecision.approve(finalPrice, discount);
    }
}

// Imperative Shell: 결정을 실행
public class OrderService {
    public Order place(Long userId, Long cartId) {
        // I/O
        User user = userFinder.findById(userId).orElseThrow();
        Cart cart = cartFinder.findById(cartId).orElseThrow();
        Stock stock = stockChecker.check(cart);

        // 순수 함수 호출 — 모든 비즈니스 결정
        OrderDecision decision = OrderDecider.decide(cart, user, discountRule, stock);

        if (decision.isRejected()) throw new OrderRejectedException(decision.reason());

        // I/O
        Order order = Order.from(cart, user, decision);
        return orderRepository.save(order);
    }
}
```

`OrderDecider.decide()`는 완전한 순수 함수입니다. Mock 없이 모든 비즈니스 결정을 테스트할 수 있습니다.

---

## 💻 실전 적용: 순수성 체크리스트

```java
// 이 메서드가 순수한지 확인하는 질문들
// 1. 현재 시간을 쓰는가? → LocalDate.now(), System.currentTimeMillis()
// 2. 난수를 쓰는가? → Math.random(), UUID.randomUUID()
// 3. 외부 저장소에 접근하는가? → Repository, Cache, File
// 4. 네트워크를 쓰는가? → HttpClient, RestTemplate
// 5. 인자 외의 필드를 읽는가? → this.someState
// 6. 인자나 필드를 변경하는가? → list.add(), this.status = ...

// 하나라도 YES면 순수하지 않음
// → 분리 또는 의존성 주입 고려
```

---

## 🤔 트레이드오프

### "불변 객체를 쓰면 성능 문제가 있지 않은가?"

새 객체 생성 비용이 있지만, 대부분의 비즈니스 애플리케이션에서 이 비용은 DB I/O나 네트워크 비용에 비해 무시할 수준입니다. 불변성이 주는 테스트 용이성과 스레드 안전성의 가치가 더 큽니다.

### "도메인 로직을 static 메서드로 만드는 것이 항상 좋은가?"

순수 계산 로직은 static이 자연스럽습니다. 그러나 나중에 정책이 교체될 가능성이 있다면 인터페이스로 추상화하는 것이 낫습니다. "이 계산 방식이 변할 수 있는가?"를 기준으로 선택합니다.

---

## 📌 핵심 정리

```
순수 함수의 두 조건:
  같은 입력 → 항상 같은 출력
  부수효과 없음 (외부 상태 변경 없음)

순수 함수 테스트의 특징:
  Mock 없음
  @BeforeEach 없음
  경계값 추가가 한 줄

부수효과를 분리하는 방법:
  도메인 객체에서 Repository 제거
  계산 로직을 별도 정적 메서드로 추출
  불변 도메인 객체 (상태 변경 → 새 객체 반환)

Functional Core / Imperative Shell:
  Core: 순수 함수, 모든 비즈니스 결정
  Shell: I/O, 부수효과, Core 호출 및 결과 실행

순수성 점검:
  LocalDate.now() → Clock 주입
  UUID.randomUUID() → IdGenerator 주입
  repository.find() → 파라미터로 받기
  this.state 변경 → 새 객체 반환
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 메서드에서 순수한 부분과 순수하지 않은 부분을 구분하고, 순수한 계산 로직을 분리하라.

```java
public class ShippingService {
    public ShippingFee calculate(Order order, User user) {
        int baseWeight = productRepository.findWeightByIds(order.itemIds()); // I/O
        boolean isFreeShipping = user.grade() == Grade.VIP
            || order.totalPrice() >= 50_000;

        if (isFreeShipping) return ShippingFee.FREE;

        int fee = baseWeight <= 5_000 ? 3_000 : 5_000;
        if (LocalDate.now().getDayOfWeek() == DayOfWeek.SATURDAY) fee += 1_000;

        return new ShippingFee(fee);
    }
}
```

**Q2.** 불변 도메인 객체의 상태 변경 메서드가 새 객체를 반환하면, JPA Entity와 어떻게 함께 사용할 수 있는가? 불변 Entity는 현실적인가?

**Q3.** `OrderDecider.decide()` 같은 순수 함수를 static으로 만드는 것과 인스턴스 메서드로 만드는 것의 차이는 무엇인가? 언제 인스턴스 메서드가 필요한가?

> 💡 **해설**

**Q1.**

순수하지 않은 부분: `productRepository.findWeightByIds()` (I/O), `LocalDate.now()` (현재 시간).

순수한 부분: `isFreeShipping` 계산, `fee` 계산.

분리 — 모든 입력을 파라미터로 받는 순수 함수:

```java
public static ShippingFee calculate(
        int baseWeight, Grade userGrade, int totalPrice, DayOfWeek dayOfWeek) {

    boolean isFreeShipping = userGrade == Grade.VIP || totalPrice >= 50_000;
    if (isFreeShipping) return ShippingFee.FREE;

    int fee = baseWeight <= 5_000 ? 3_000 : 5_000;
    if (dayOfWeek == DayOfWeek.SATURDAY) fee += 1_000;
    return new ShippingFee(fee);
}
```

서비스에서 I/O 후 순수 함수 호출:

```java
int weight = productRepository.findWeightByIds(order.itemIds());
DayOfWeek day = LocalDate.now(clock).getDayOfWeek();
return ShippingFee.calculate(weight, user.grade(), order.totalPrice(), day);
```

순수 함수는 `(5_000, NORMAL, 30_000, SATURDAY)` 같은 인자로 바로 테스트 가능.

**Q2.**

JPA Entity가 완전한 불변이기는 어렵다. JPA는 기본적으로 가변 객체를 기대하고, 변경 감지(Dirty Checking)는 객체의 상태가 변경됨을 전제로 한다.

현실적인 접근:

① **Rich Domain Model** — Entity 내부에 비즈니스 메서드를 두되, 그 메서드가 순수 계산 로직을 별도 메서드로 위임한다. `order.completeWith(clock)` 내부에서 계산은 순수 함수가 하고, Entity의 상태 변경은 메서드 안에서 한다.

② **도메인 객체와 JPA Entity 분리** — 비즈니스 로직을 가진 순수 도메인 객체와 DB 매핑을 위한 JPA Entity를 별도로 둔다(복잡하지만 순수성 최대화).

실무에서는 ①이 더 현실적이며, 계산 로직만이라도 순수 함수로 추출하는 것으로 충분한 이점을 얻을 수 있다.

**Q3.**

**static 순수 함수**: 정책이 고정되어 있을 때 적합. `applyDiscount(price, rate)`처럼 알고리즘 자체가 변하지 않는다. 테스트에서 클래스 이름으로 직접 호출.

**인스턴스 메서드(인터페이스)**: 정책이 교체될 수 있을 때 필요. `DiscountPolicy.calculate(user)` — VIP 10%, GOLD 5%, 시즌별 다른 정책 등 구현이 바뀔 수 있다. 인스턴스로 주입받아서 테스트에서 다른 구현으로 교체 가능.

기준: "이 계산 방식이 변할 수 있는가?" → 변한다면 인스턴스 + 인터페이스, 변하지 않는다면 static.

---

<div align="center">

**[⬅️ 이전: Humble Object Pattern](./04-humble-object-pattern.md)** | **[다음: Ports and Adapters ➡️](./06-ports-and-adapters.md)**

</div>
