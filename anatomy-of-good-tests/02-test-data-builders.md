# 02. Test Data Builders

> **픽스처가 길어질수록 테스트의 의도가 묻힌다 — Builder는 "관련 없는 세부사항을 숨기고 중요한 것만 드러내는" 도구다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Arrange 코드가 길어지면 테스트의 어떤 가치가 훼손되는가?
- Object Mother와 Test Data Builder는 어떻게 다른가?
- 프로덕션 코드의 Builder와 테스트 전용 Builder는 어떻게 분리해야 하는가?

---

## 🔍 왜 픽스처가 문제가 되는가

```java
@Test
void 쿠폰_적용_시_할인이_추가된다() {

    // Arrange — 26줄, 실제로 이 테스트와 관련 있는 것은 몇 줄인가?
    Address address = new Address("서울시", "강남구", "테헤란로", "12345");
    UserProfile profile = new UserProfile("alice", "010-1234-5678", address);
    User user = new User(1L, "alice@example.com", profile, Grade.VIP, LocalDate.of(1990, 1, 1));

    Item item1 = new Item(1L, "자바 책", 20_000, Category.BOOK, true);
    Cart cart = new Cart(user);
    cart.addItem(item1, 1);

    Coupon coupon = new Coupon("SUMMER2024", CouponType.RATE, 5, LocalDate.of(2024, 12, 31));

    // Act — 정작 이 한 줄이 핵심
    Order order = orderService.placeWithCoupon(cart, user, coupon);

    // Assert
    assertThat(order.totalPrice()).isEqualTo(19_000);
}
```

이 테스트에서 **쿠폰 할인 검증**에 영향을 미치는 것은:
- `Item`의 가격: `20_000`
- `Coupon`의 할인율: `5%`

나머지 — 주소, 전화번호, 생년월일, 카테고리, 배송 여부 — 는 이 테스트와 무관합니다. 하지만 코드를 읽는 사람은 "이 정보가 혹시 테스트 결과에 영향을 미치는 건 아닐까?"라고 의심하며 읽게 됩니다.

---

## 🏛️ 두 가지 해결 접근

### Object Mother: 미리 만들어진 픽스처 객체

```java
// 공통 팩토리 클래스
public class UserFixture {
    public static User aVipUser() {
        return new User(1L, "alice@example.com",
            new UserProfile("alice", "010-0000-0000",
                new Address("서울시", "강남구", "테헤란로", "12345")),
            Grade.VIP, LocalDate.of(1990, 1, 1));
    }

    public static User aNormalUser() { ... }
}
```

```java
@Test
void 쿠폰_적용_시_할인이_추가된다() {
    User user = UserFixture.aVipUser();  // 짧아졌다
    ...
}
```

편리하지만, **변형이 어렵습니다.** "VIP인데 생년월일이 미성년자인 사용자"가 필요하면 새 메서드를 추가해야 합니다. Object Mother는 시간이 지날수록 메서드가 폭발적으로 늘어납니다.

### Test Data Builder: 유연한 변형이 가능한 빌더

```java
// 테스트 전용 Builder — 프로덕션 코드와 분리
public class UserBuilder {
    private Long id = 1L;
    private String email = "alice@example.com";
    private Grade grade = Grade.VIP;
    private LocalDate birthDate = LocalDate.of(1990, 1, 1);
    // 나머지 필드는 테스트에서 거의 안 쓰이는 기본값으로

    public static UserBuilder aUser() { return new UserBuilder(); }

    public UserBuilder withGrade(Grade grade) {
        this.grade = grade;
        return this;
    }

    public UserBuilder withEmail(String email) {
        this.email = email;
        return this;
    }

    public UserBuilder withBirthDate(LocalDate birthDate) {
        this.birthDate = birthDate;
        return this;
    }

    public User build() {
        return new User(id, email,
            new UserProfile("alice", "010-0000-0000",  // 테스트에 무관한 값은 기본값
                new Address("서울시", "강남구", "테헤란로", "12345")),
            grade, birthDate);
    }
}
```

---

## 😱 Builder 없는 테스트 vs Builder 있는 테스트

```java
// ❌ Before: 중요한 것과 중요하지 않은 것이 섞여있다

@Test
void 미성년자는_성인_상품을_주문할_수_없다() {
    User minor = new User(2L, "bob@example.com",
        new UserProfile("bob", "010-9999-8888",
            new Address("서울시", "마포구", "홍대로", "54321")),  // 무관
        Grade.NORMAL,
        LocalDate.of(2010, 5, 15));  // ← 이게 핵심인데 26번째 줄에 묻혀있다

    Item adultItem = new Item(10L, "성인용품", 30_000, Category.ADULT, true);
    Cart cart = new Cart(minor);
    cart.addItem(adultItem, 1);

    assertThatThrownBy(() -> orderService.place(cart, minor))
        .isInstanceOf(AgeRestrictionException.class);
}
```

```java
// ✅ After: 테스트와 관련 있는 것만 드러난다

@Test
void 미성년자는_성인_상품을_주문할_수_없다() {
    User minor = aUser()
        .withBirthDate(LocalDate.of(2010, 5, 15))  // ← 핵심이 첫 줄에 명확히
        .build();

    Item adultItem = anItem()
        .withCategory(Category.ADULT)              // ← 이것도 핵심
        .build();

    Cart cart = aCart().withUser(minor).withItem(adultItem).build();

    assertThatThrownBy(() -> orderService.place(cart, minor))
        .isInstanceOf(AgeRestrictionException.class);
}
```

테스트를 읽는 사람이 3초 안에 "미성년자 + 성인 상품 = 예외"를 파악할 수 있습니다.

---

## ✨ 실전 Test Data Builder 설계

### 원칙 1: 기본값은 "유효하지만 관심 없는 값"으로

```java
public class OrderBuilder {
    // 모든 기본값은 유효한 상태여야 한다 (테스트가 기본값 때문에 실패하면 안 됨)
    private User user = aUser().build();           // 유효한 기본 사용자
    private List<CartItem> items = List.of(
        anItem().withPrice(10_000).build()          // 유효한 기본 아이템
    );
    private Coupon coupon = null;                   // 쿠폰 없음이 기본

    public static OrderBuilder anOrder() { return new OrderBuilder(); }

    public OrderBuilder withUser(User user) { this.user = user; return this; }
    public OrderBuilder withItems(List<CartItem> items) { this.items = items; return this; }
    public OrderBuilder withCoupon(Coupon coupon) { this.coupon = coupon; return this; }

    public Order build() {
        Cart cart = new Cart(user, items);
        return coupon != null
            ? orderService.placeWithCoupon(cart, user, coupon)
            : orderService.place(cart, user);
    }
}
```

### 원칙 2: 빌더를 중첩해서 조합한다

```java
@Test
void VIP_회원이_쿠폰을_사용하면_중복_할인이_적용된다() {
    User vipUser = aUser().withGrade(Grade.VIP).build();
    Coupon rateCoupon = aCoupon().withDiscountRate(5).build();

    Order order = anOrder()
        .withUser(vipUser)
        .withCoupon(rateCoupon)
        .withItems(List.of(anItem().withPrice(20_000).build()))
        .build();

    // VIP 10% + 쿠폰 5% = 총 14.5% 할인
    assertThat(order.totalPrice()).isEqualTo(17_100);
}
```

### 원칙 3: 테스트 패키지에 배치, 프로덕션 패키지에 노출하지 않는다

```
src/
  main/java/com/example/
    order/Order.java
    user/User.java
  test/java/com/example/
    support/                        ← 테스트 지원 패키지
      builder/
        UserBuilder.java
        OrderBuilder.java
        CouponBuilder.java
      fixture/
        UserFixture.java            ← 자주 쓰는 조합은 Object Mother로도 제공
```

---

## 💻 실전 적용: Lombok과 함께 간결하게

```java
// Lombok @Builder + 테스트 전용 기본값 패턴
@Builder
public class User {
    private Long id;
    private String email;
    private Grade grade;
    // ...
}

// 테스트 지원 클래스
public class TestUserFactory {

    public static User.UserBuilder aValidUser() {
        return User.builder()
            .id(1L)
            .email("test@example.com")
            .grade(Grade.NORMAL)
            .birthDate(LocalDate.of(1990, 1, 1));
        // 기본값을 채워둔 builder를 반환 → 원하는 필드만 오버라이드 가능
    }
}

// 사용
@Test
void 테스트() {
    User vip = TestUserFactory.aValidUser()
        .grade(Grade.VIP)   // VIP만 오버라이드
        .build();
}
```

---

## 🤔 트레이드오프

### "Builder 코드 자체를 유지보수하는 비용이 있다"

맞습니다. 프로덕션 `User` 클래스에 필드가 추가될 때마다 `UserBuilder`도 업데이트해야 합니다. 이 비용은 클래스가 크고 복잡할수록 높아집니다.

단순한 값 객체(`Money`, `Address`)는 빌더 없이 생성자로 직접 만들어도 충분합니다. Builder가 필요한 시점은 Arrange 코드가 테스트 의도를 묻기 시작할 때입니다.

### "Object Mother를 쓰면 안 되는가?"

Object Mother와 Builder는 함께 쓸 수 있습니다. `UserFixture.aVipUser()`처럼 자주 쓰는 조합은 Object Mother로 제공하고, 변형이 필요한 경우는 `aUser().withGrade(VIP).build()`로 Builder를 씁니다.

---

## 📌 핵심 정리

```
Test Data Builder의 목적:
  관련 없는 세부사항을 숨기고
  테스트와 관련 있는 값만 드러낸다

Object Mother vs Builder:
  Object Mother: 미리 만들어진 고정 픽스처, 변형 어려움
  Builder: 기본값 + 원하는 필드만 오버라이드, 유연함

좋은 Builder의 기본값:
  유효한 상태여야 함 (기본값으로 테스트가 실패하면 안 됨)
  테스트 관심사와 무관한 임의의 값

빌더가 필요한 시점:
  Arrange 코드가 테스트 의도를 묻기 시작할 때
  같은 객체를 여러 테스트에서 조금씩 다르게 생성할 때

배치:
  src/test/java/support/builder/
  프로덕션 코드와 절대 섞지 않는다
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트의 Arrange 코드에서 테스트 의도와 직접 관련 있는 값과 없는 값을 구분하고, Test Data Builder를 사용해 리팩터링하라.

```java
@Test
void 만료된_쿠폰으로_주문하면_예외가_발생한다() {
    User user = new User(1L, "alice@example.com",
        new UserProfile("alice", "010-1234-5678", Grade.VIP),
        LocalDate.of(1990, 3, 15));

    Coupon expiredCoupon = new Coupon("EXPIRED01", CouponType.RATE, 10,
        LocalDate.of(2020, 12, 31));  // 이미 만료

    Item item = new Item(1L, "책", 20_000, Category.BOOK, true);
    Cart cart = new Cart(List.of(new CartItem(item, 1)));

    assertThatThrownBy(() -> orderService.place(cart, user, expiredCoupon))
        .isInstanceOf(ExpiredCouponException.class);
}
```

**Q2.** 같은 `User` 빌더를 10개 테스트에서 쓰다가, 프로덕션 `User` 클래스에 필수 필드 `phoneVerified: boolean`이 추가됐다. 이 변경을 어떻게 처리하겠는가? Builder 패턴이 이 상황에서 어떤 장점을 제공하는가?

**Q3.** 아래 두 접근 중 어느 것이 더 나은가? 이유를 설명하라.

```java
// 접근 A: Object Mother
User vipUser = UserFixture.aVipUser();

// 접근 B: Builder
User vipUser = aUser().withGrade(Grade.VIP).build();
```

> 💡 **해설**
>
> **Q1.** 테스트 의도와 관련 있는 값: `Coupon`의 만료일(`2020-12-31`)만이 핵심이다. 나머지 — 사용자 이름, 전화번호, 생년월일, 아이템 카테고리 등 — 는 이 테스트의 결과에 영향을 미치지 않는다. 리팩터링: `User user = aUser().build(); Coupon expired = aCoupon().withExpiredDate(LocalDate.of(2020, 12, 31)).build(); Cart cart = aCart().build();` — 핵심인 만료일만 명시적으로 드러난다.
>
> **Q2.** `UserBuilder.build()` 내부에서 `phoneVerified`에 기본값(`false` 또는 `true`)을 추가하면 된다. 10개 테스트 코드는 수정할 필요가 없다. Builder 패턴의 장점: 변경이 Builder 한 곳에 집중된다. Object Mother였다면 `UserFixture`의 모든 메서드를 수정해야 하고, 직접 생성자를 쓰는 방식이었다면 10개 테스트 전부를 수정해야 한다. Builder는 유지보수 변경점을 한 곳으로 모은다.
>
> **Q3.** 맥락에 따라 다르다. "VIP 사용자"가 테스트의 핵심 조건일 때는 Builder B가 낫다 — "Grade.VIP"가 명시적으로 드러나서 테스트 의도가 분명하다. 반면 사용자가 테스트의 핵심이 아니라 단순한 전제 조건일 때(예: 특정 사용자가 어떤 동작을 한다는 컨텍스트)는 Object Mother A가 더 간결하다. 두 접근을 혼용하되, "이 필드가 이 테스트의 핵심인가?"를 기준으로 선택하면 된다.

---

<div align="center">

**[⬅️ 이전: Single Assert Principle](./01-single-assert-principle.md)** | **[다음: Boundary Testing ➡️](./03-boundary-testing.md)**

</div>
