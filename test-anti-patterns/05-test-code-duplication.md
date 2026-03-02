# 05. Test Code Duplication

> **DRY는 테스트에서도 적용되지만 — 추상화가 과도하면 테스트가 무엇을 검증하는지 알기 어려워진다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 테스트 코드에서 반복이 문제가 되는 경우는 언제인가?
- Test Data Builder(Builder 패턴)를 어떻게 활용하는가?
- 추상화가 과도한 테스트는 어떤 문제를 만드는가?

---

## 🔍 테스트 중복의 두 가지 얼굴

테스트의 중복은 프로덕션 코드의 중복과 다릅니다. 일부 반복은 테스트의 **가독성과 독립성을 위해 의도적으로 허용**해야 할 수 있습니다.

```
제거해야 하는 중복:
  동일한 객체 생성 코드가 50개 테스트에 복사됨
  같은 검증 로직이 반복됨
  @BeforeEach가 모든 테스트와 무관한 설정을 포함

허용하거나 신중해야 하는 "중복":
  각 테스트가 의도를 명확히 보여주기 위해
  비슷해 보이지만 다른 시나리오를 검증하는 코드
```

---

## 😱 문제 1: 객체 생성 코드 반복

```java
// ❌ 50개 테스트에 동일한 생성 코드 복사
@Test
void VIP_할인_테스트() {
    User user = new User(1L, "홍길동", "hong@example.com",
        Grade.VIP, LocalDate.of(1990, 1, 1), "010-1234-5678",
        List.of(new Address("서울", "강남구")));
    // ... 실제 테스트 3줄
}

@Test
void 이메일_발송_테스트() {
    User user = new User(1L, "홍길동", "hong@example.com",
        Grade.VIP, LocalDate.of(1990, 1, 1), "010-1234-5678",
        List.of(new Address("서울", "강남구"))); // 동일한 복사
    // ... 실제 테스트 3줄
}
```

`User` 생성자에 파라미터 하나가 추가되면 50개를 전부 수정해야 합니다.

---

## ✨ 해결 1: Test Data Builder

Test Data Builder는 기본값을 가진 빌더로, 테스트에서 중요한 필드만 오버라이드합니다.

```java
// ✅ Test Data Builder
public class UserBuilder {
    private Long id = 1L;
    private String name = "홍길동";
    private String email = "hong@example.com";
    private Grade grade = Grade.NORMAL;
    private LocalDate birthDate = LocalDate.of(1990, 1, 1);
    private String phone = "010-1234-5678";
    private List<Address> addresses = List.of(new Address("서울", "강남구"));

    public static UserBuilder aUser() {
        return new UserBuilder();
    }

    public UserBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public UserBuilder withGrade(Grade grade) {
        this.grade = grade;
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public User build() {
        return new User(id, name, email, grade, birthDate, phone, addresses);
    }
}
```

```java
// ✅ 테스트에서는 관심 있는 필드만 지정
@Test
void VIP_할인_테스트() {
    User vipUser = aUser().withGrade(VIP).build();
    // 이 테스트는 grade만 중요하다는 것이 명확
}

@Test
void 이메일_발송_테스트() {
    User user = aUser().withEmail("target@example.com").build();
    // 이 테스트는 email만 중요하다는 것이 명확
}
```

`User` 생성자가 바뀌면 `UserBuilder.build()` 하나만 수정합니다.

### Lombok + 빌더 패턴 조합

```java
// Lombok으로 보일러플레이트 제거
@Builder
@AllArgsConstructor
public class Order {
    private Long id;
    private Long userId;
    private int totalPrice;
    private OrderStatus status;
    private LocalDateTime createdAt;
}

// Test용 기본값 제공
public class OrderBuilder {
    private static final AtomicLong ID_SEQ = new AtomicLong(1);

    public static Order.OrderBuilder anOrder() {
        return Order.builder()
            .id(ID_SEQ.getAndIncrement())  // 자동 증가 ID
            .userId(1L)
            .totalPrice(20_000)
            .status(OrderStatus.PENDING)
            .createdAt(LocalDateTime.now());
    }
}

// 사용
Order order = anOrder().withUserId(2L).withTotalPrice(50_000).build();
```

---

## 😱 문제 2: @BeforeEach 과잉 — 모든 테스트에 불필요한 설정

```java
// ❌ @BeforeEach가 특정 테스트에만 필요한 것을 모두 설정
class OrderServiceTest {

    @BeforeEach
    void setUp() {
        // 할인 테스트에는 필요 없음
        when(emailSender.send(any(), any())).thenReturn(true);
        // 이메일 테스트에는 필요 없음
        when(discountPolicy.calculate(any())).thenReturn(1_000);
        // 일부 테스트에만 필요
        orderRepository.save(defaultOrder());
        // 모든 테스트에 필요 — 이것만 @BeforeEach에 있어야 함
        orderRepository.deleteAll();
    }
}
```

`setUp()`이 커질수록 각 테스트가 어떤 환경에서 실행되는지 파악하기 위해 스크롤을 오가야 합니다.

```java
// ✅ @BeforeEach는 공통 준비만 — 나머지는 테스트 내부에서
class OrderServiceTest {

    private InMemoryOrderRepository orderRepository;
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderRepository = new InMemoryOrderRepository(); // 공통: 초기화
        // discountPolicy, emailSender는 각 테스트에서 필요에 따라 설정
    }

    @Test
    void VIP_할인_테스트() {
        // 이 테스트에 필요한 설정을 바로 아래에서 볼 수 있음
        DiscountPolicy vipPolicy = user -> 2_000;
        orderService = new OrderService(orderRepository, vipPolicy, noOpEmailSender());

        Order order = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());
        assertThat(order.totalPrice()).isEqualTo(18_000);
    }

    @Test
    void 이메일_발송_테스트() {
        // 이 테스트에 필요한 설정
        EmailSender spyEmailSender = mock(EmailSender.class);
        orderService = new OrderService(orderRepository, noDiscountPolicy(), spyEmailSender);

        orderService.place(aCommand().withUser(user).build());
        verify(spyEmailSender).send(eq(user.email()), any());
    }
}
```

---

## 😱 문제 3: 검증 로직 반복

```java
// ❌ 동일한 검증 로직이 여러 테스트에 반복
@Test
void 생성된_주문_검증_A() {
    Order order = orderService.place(commandA);
    assertThat(order.id()).isNotNull();
    assertThat(order.status()).isEqualTo(PENDING);
    assertThat(order.createdAt()).isNotNull();
    assertThat(order.createdAt()).isBefore(LocalDateTime.now());
}

@Test
void 생성된_주문_검증_B() {
    Order order = orderService.place(commandB);
    assertThat(order.id()).isNotNull();                         // 동일한 검증
    assertThat(order.status()).isEqualTo(PENDING);              // 동일한 검증
    assertThat(order.createdAt()).isNotNull();                  // 동일한 검증
    assertThat(order.createdAt()).isBefore(LocalDateTime.now()); // 동일한 검증
    // 추가 검증
    assertThat(order.totalPrice()).isEqualTo(20_000);
}
```

### 커스텀 AssertJ Condition 또는 AbstractAssert

```java
// ✅ 커스텀 단언으로 공통 검증 추출
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {

    public OrderAssert(Order order) {
        super(order, OrderAssert.class);
    }

    public static OrderAssert assertThatOrder(Order order) {
        return new OrderAssert(order);
    }

    public OrderAssert isValidNewOrder() {
        isNotNull();
        assertThat(actual.id()).as("주문 ID").isNotNull();
        assertThat(actual.status()).as("주문 상태").isEqualTo(PENDING);
        assertThat(actual.createdAt()).as("생성 시각").isNotNull()
            .isBefore(LocalDateTime.now());
        return this;
    }

    public OrderAssert hasTotalPrice(int expected) {
        assertThat(actual.totalPrice())
            .as("주문 금액")
            .isEqualTo(expected);
        return this;
    }
}
```

```java
// ✅ 사용 — 중복 제거 + 가독성
@Test
void 생성된_주문_검증_A() {
    Order order = orderService.place(commandA);
    assertThatOrder(order).isValidNewOrder();
}

@Test
void 생성된_주문_검증_B() {
    Order order = orderService.place(commandB);
    assertThatOrder(order)
        .isValidNewOrder()
        .hasTotalPrice(20_000);
}
```

---

## 🏛️ 적절한 중복 — 추상화보다 명확성

```java
// 이 두 테스트는 비슷해 보이지만 다른 시나리오
@Test
void 일반_회원_할인_없음() {
    User normalUser = aUser().withGrade(NORMAL).build();
    Order order = orderService.place(aCommand().withUser(normalUser).withPrice(20_000).build());
    assertThat(order.totalPrice()).isEqualTo(20_000);
}

@Test
void VIP_회원_10퍼센트_할인() {
    User vipUser = aUser().withGrade(VIP).build();
    Order order = orderService.place(aCommand().withUser(vipUser).withPrice(20_000).build());
    assertThat(order.totalPrice()).isEqualTo(18_000);
}
```

이 두 테스트를 하나의 파라미터화된 테스트나 공통 메서드로 합치면 각 시나리오의 의도가 흐려집니다. 코드가 조금 반복되더라도 각각의 테스트가 의도를 명확히 드러내는 것이 낫습니다.

**기준:** 중복 제거가 가독성을 높이면 제거하고, 낮추면 중복을 허용합니다.

---

## 🤔 트레이드오프

### "@ParameterizedTest로 비슷한 테스트를 하나로 합치면 안 되는가?"

비슷한 입력에 비슷한 동작을 경계값별로 검증할 때는 `@ParameterizedTest`가 효과적입니다.

```java
// ✅ 경계값 검증 — @ParameterizedTest 적합
@ParameterizedTest
@CsvSource({"999, false", "1000, true", "50000, true"})
void 최소_주문_금액_검증(int price, boolean isValid) {
    assertThat(orderValidator.isValidAmount(price)).isEqualTo(isValid);
}
```

그러나 서로 다른 시나리오를 하나로 합치면 실패 시 어떤 케이스가 실패했는지 파악하기 어려워집니다. 각 케이스가 명확한 이름을 갖는 독립 테스트가 나을 때도 있습니다.

---

## 📌 핵심 정리

```
제거해야 하는 중복:
  동일한 객체 생성 코드 반복
  → Test Data Builder로 기본값 제공
  동일한 검증 로직 반복
  → 커스텀 AssertJ AbstractAssert
  @BeforeEach에 모든 설정 몰아넣기
  → 테스트별 필요한 설정만

Test Data Builder 원칙:
  기본값으로 유효한 객체 생성
  with() 메서드로 관심 있는 필드만 오버라이드
  빌더가 변경되면 한 곳만 수정

@BeforeEach 원칙:
  모든 테스트에 필요한 것만 (DB 초기화 등)
  테스트별 설정은 해당 테스트 안에서

허용하는 중복:
  각 시나리오의 의도를 명확히 드러내는 코드
  추상화로 인해 오히려 이해하기 어려워지는 경우

판단 기준:
  "이 추상화가 테스트를 더 읽기 쉽게 만드는가?"
  → YES: 추상화 도입
  → NO: 중복 허용
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트 클래스에서 중복을 찾고, Test Data Builder와 커스텀 단언을 사용해서 리팩터링하라.

```java
class CouponServiceTest {

    @Test
    void 쿠폰_적용_성공() {
        User user = new User(1L, "홍길동", "hong@example.com", Grade.VIP,
            LocalDate.of(1990, 1, 1));
        Coupon coupon = new Coupon("SAVE10", 10, LocalDate.now().plusDays(7),
            CouponType.RATE, true);
        Order order = new Order(null, 1L, 20_000, null, null, null);

        Order result = couponService.apply(order, coupon, user);

        assertThat(result.id()).isNull();
        assertThat(result.totalPrice()).isEqualTo(18_000);
        assertThat(result.appliedCoupon()).isEqualTo("SAVE10");
    }

    @Test
    void 만료된_쿠폰_적용_실패() {
        User user = new User(1L, "홍길동", "hong@example.com", Grade.VIP,
            LocalDate.of(1990, 1, 1));
        Coupon expiredCoupon = new Coupon("SAVE10", 10, LocalDate.now().minusDays(1),
            CouponType.RATE, true);
        Order order = new Order(null, 1L, 20_000, null, null, null);

        assertThatThrownBy(() -> couponService.apply(order, expiredCoupon, user))
            .isInstanceOf(ExpiredCouponException.class);
    }
}
```

**Q2.** `@BeforeEach`가 100줄이 넘는 레거시 테스트 클래스가 있다. 이것을 어떻게 점진적으로 개선하는가?

**Q3.** Test Data Builder에서 ID 필드를 어떻게 처리하는 것이 좋은가? 아래 두 방법의 차이와 장단점을 설명하라.

```java
// 방법 A: 고정 ID
public static UserBuilder aUser() {
    return new UserBuilder().withId(1L);
}

// 방법 B: 자동 증가 ID
private static final AtomicLong ID_SEQ = new AtomicLong(1);

public static UserBuilder aUser() {
    return new UserBuilder().withId(ID_SEQ.getAndIncrement());
}
```

> 💡 **해설**

**Q1.**

중복: `User`, `Coupon`, `Order` 생성 코드가 두 테스트에 반복. `Order` 검증 로직이 분산.

리팩터링:

```java
// Test Data Builder
public class CouponBuilder {
    private String code = "SAVE10";
    private int discountRate = 10;
    private LocalDate expiresAt = LocalDate.now().plusDays(7);
    private CouponType type = CouponType.RATE;
    private boolean active = true;

    public static CouponBuilder aCoupon() { return new CouponBuilder(); }
    public CouponBuilder expired() { this.expiresAt = LocalDate.now().minusDays(1); return this; }
    public Coupon build() { return new Coupon(code, discountRate, expiresAt, type, active); }
}

// 커스텀 단언
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {
    public static OrderAssert assertThatOrder(Order order) { return new OrderAssert(order); }
    public OrderAssert hasCouponApplied(String code) {
        assertThat(actual.appliedCoupon()).isEqualTo(code);
        return this;
    }
    public OrderAssert hasTotalPrice(int price) {
        assertThat(actual.totalPrice()).isEqualTo(price);
        return this;
    }
}

// 리팩터링된 테스트
class CouponServiceTest {

    User user = aUser().withGrade(VIP).build();
    Order order = anOrder().withUserId(1L).withTotalPrice(20_000).build();

    @Test
    void 쿠폰_적용_성공() {
        Coupon coupon = aCoupon().build(); // 유효한 쿠폰 (기본값)
        Order result = couponService.apply(order, coupon, user);
        assertThatOrder(result)
            .hasTotalPrice(18_000)
            .hasCouponApplied("SAVE10");
    }

    @Test
    void 만료된_쿠폰_적용_실패() {
        Coupon expiredCoupon = aCoupon().expired().build(); // 만료 쿠폰만 변경
        assertThatThrownBy(() -> couponService.apply(order, expiredCoupon, user))
            .isInstanceOf(ExpiredCouponException.class);
    }
}
```

**Q2.**

점진적 개선 전략:

① 먼저 현재 `@BeforeEach`의 내용을 분류한다. "모든 테스트에 필요한 것"과 "일부 테스트에만 필요한 것"으로 나눈다.

② "일부 테스트에만 필요한 것"을 해당 테스트 메서드 안으로 이동한다. 테스트를 하나씩 수정하고 실행해서 안전하게 진행한다.

③ 반복되는 객체 생성을 Test Data Builder로 추출한다.

④ 최종적으로 `@BeforeEach`에는 공통 초기화(DB 정리, 공통 Fake 생성)만 남긴다.

한 번에 전체를 바꾸지 말고, PR 단위로 작은 변경을 반복한다.

**Q3.**

**방법 A (고정 ID)**: 테스트가 항상 같은 ID를 사용하므로 예측 가능하다. `findById(1L)`처럼 ID를 명시적으로 쓰는 테스트에 적합하다. 단, 같은 ID를 가진 여러 객체를 저장할 경우 충돌이 발생할 수 있다.

**방법 B (자동 증가 ID)**: 각 테스트에서 생성하는 객체가 서로 다른 ID를 가지므로 충돌이 없다. `InMemoryRepository`에 여러 객체를 저장하는 테스트에 적합하다. 단, ID 값이 예측 불가능해서 `findById(specificId)` 형태의 테스트 작성이 어렵다.

실무 조합: 특정 ID가 중요한 테스트는 `withId()`로 명시적 지정, 나머지는 자동 증가를 기본값으로 사용.

---

<div align="center">

**[⬅️ 이전: Overspecified Tests](./04-overspecified-tests.md)** | **[다음: Hidden Test Dependencies ➡️](./06-hidden-test-dependencies.md)**

</div>
