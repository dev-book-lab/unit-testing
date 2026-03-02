# 05. Fixtures & SetUp

> **@BeforeEach는 반복을 줄이는 도구가 아니다 — 모든 테스트의 공통 전제 조건을 설정하는 도구다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `@BeforeEach`에 넣어야 하는 것과 넣으면 안 되는 것은 무엇인가?
- 공유 픽스처가 만드는 "숨겨진 결합"이란 무엇인가?
- 픽스처가 복잡해질 때 어떤 전략으로 대응하는가?

---

## 🔍 문제 상황: 편리함이 만드는 결합

```java
class OrderServiceTest {

    private OrderService orderService;
    private User vipUser;
    private Cart cart;

    @BeforeEach
    void setUp() {
        InMemoryOrderRepository repo = new InMemoryOrderRepository();
        orderService = new OrderService(repo, new RateDiscountPolicy(10));
        vipUser = new User("alice", Grade.VIP);
        cart = new Cart(List.of(new Item("책", 20_000)));
    }

    @Test
    void VIP_할인_적용() {
        Order order = orderService.place(cart, vipUser);
        assertThat(order.totalPrice()).isEqualTo(18_000);
    }

    @Test
    void 일반_회원은_할인_없음() {
        User normalUser = new User("bob", Grade.NORMAL);  // setUp의 vipUser를 쓰지 않음
        Order order = orderService.place(cart, normalUser);
        assertThat(order.totalPrice()).isEqualTo(20_000);
    }

    @Test
    void 빈_장바구니는_주문_불가() {
        Cart emptyCart = new Cart(List.of());  // setUp의 cart를 쓰지 않음
        assertThatThrownBy(() -> orderService.place(emptyCart, vipUser))
            .isInstanceOf(EmptyCartException.class);
    }
}
```

`setUp()`에서 만들어진 `vipUser`와 `cart`는 모든 테스트에서 사용하지 않습니다. 일부 테스트는 자신의 값을 직접 만들면서 `setUp`의 값을 무시합니다. 이 상황에서 `setUp`을 읽는 사람은 "이 값들이 어떤 테스트에 쓰이는 걸까?"를 추적해야 합니다.

더 위험한 것은, `setUp`에서 만든 객체가 **변경 가능한(mutable) 상태**라면 한 테스트가 이 객체를 수정했을 때 다른 테스트가 오염됩니다.

---

## 🏛️ @BeforeEach의 올바른 범위

### 넣어야 하는 것: **모든** 테스트에서 공통으로 필요한 전제 조건

```java
@BeforeEach
void setUp() {
    // ✅ 모든 테스트에서 항상 필요한 것
    repository = new InMemoryOrderRepository();   // 상태 초기화
    orderService = new OrderService(repository, new RateDiscountPolicy(10));
    // 서비스 객체는 모든 테스트에서 씀 — 적절
}
```

### 넣으면 안 되는 것: 일부 테스트에서만 필요한 테스트 데이터

```java
@BeforeEach
void setUp() {
    // ❌ 일부 테스트에서만 쓰이는 테스트 데이터
    vipUser = new User("alice", Grade.VIP);  // VIP 테스트에서만 필요
    cart = new Cart(List.of(new Item("책", 20_000)));  // 일부 테스트에서만 필요
}
```

테스트 데이터는 **그것이 필요한 테스트 안에서** 직접 만들어야 합니다. 그래야 그 테스트만 읽었을 때 완전한 맥락이 보입니다.

---

## 😱 공유 픽스처가 만드는 숨겨진 결합

### 결합 1: 테스트가 다른 테스트의 실행 결과에 의존

```java
class UserServiceTest {

    private static InMemoryUserRepository sharedRepo
        = new InMemoryUserRepository();  // ← static! 모든 테스트 인스턴스가 공유

    @BeforeEach
    void setUp() {
        userService = new UserService(sharedRepo);
        // 주의: sharedRepo는 초기화하지 않음
    }

    @Test
    void 사용자_저장() {
        userService.register("alice@example.com");
        assertThat(sharedRepo.count()).isEqualTo(1);
    }

    @Test
    void 중복_이메일_예외() {
        // 이 테스트는 '사용자_저장'이 먼저 실행돼야 통과한다
        assertThatThrownBy(() -> userService.register("alice@example.com"))
            .isInstanceOf(DuplicateEmailException.class);
    }
}
```

실행 순서를 보장할 수 없으므로, `중복_이메일_예외`를 단독으로 실행하면 실패합니다.

### 결합 2: 가변 객체 공유

```java
@BeforeEach
void setUp() {
    cart = new Cart();      // 가변 객체!
    cart.addItem(book);
}

@Test
void 아이템_추가() {
    cart.addItem(pen);      // setUp의 cart에 추가
    assertThat(cart.items()).hasSize(2);
}

@Test
void 아이템_1개_주문() {
    // setUp의 cart가 이전 테스트에서 수정됐다면 items()는 2개
    assertThat(orderService.place(cart, user).items()).hasSize(1);  // 간헐적 실패!
}
```

---

## ✨ 올바른 픽스처 관리 전략

### 전략 1: 테스트 안에서 직접 만들기 + Test Data Builder

```java
class OrderServiceTest {

    // ✅ setUp은 서비스 객체 초기화만
    private InMemoryOrderRepository repository;
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        repository = new InMemoryOrderRepository();
        orderService = new OrderService(repository, new RateDiscountPolicy(10));
    }

    @Test
    void VIP_할인_적용() {
        // ✅ 이 테스트에 필요한 데이터는 여기서 직접 만든다
        User vipUser = aUser().withGrade(VIP).build();
        Cart cart = aCart().withItem(anItem().withPrice(20_000).build()).build();

        Order order = orderService.place(cart, vipUser);

        assertThat(order.totalPrice()).isEqualTo(18_000);
    }

    @Test
    void 빈_장바구니는_주문_불가() {
        // ✅ 이 테스트에만 필요한 데이터
        Cart emptyCart = aCart().empty().build();

        assertThatThrownBy(() -> orderService.place(emptyCart, aUser().build()))
            .isInstanceOf(EmptyCartException.class);
    }
}
```

### 전략 2: `@Nested`로 같은 컨텍스트끼리 묶기

같은 전제 조건을 공유하는 테스트들은 `@Nested` 클래스로 묶으면 `setUp` 범위가 명확해집니다.

```java
class OrderServiceTest {

    private OrderService orderService;
    private InMemoryOrderRepository repository;

    @BeforeEach
    void init() {
        repository = new InMemoryOrderRepository();
        orderService = new OrderService(repository, new RateDiscountPolicy(10));
    }

    @Nested
    @DisplayName("VIP 회원 주문")
    class VipOrderTests {

        private User vipUser;
        private Cart standardCart;

        @BeforeEach
        void setUpVipContext() {
            // 이 컨텍스트에서만 필요한 공통 픽스처
            vipUser = aUser().withGrade(VIP).build();
            standardCart = aCart().withItem(anItem().withPrice(20_000).build()).build();
        }

        @Test
        void 할인이_10퍼센트_적용된다() {
            Order order = orderService.place(standardCart, vipUser);
            assertThat(order.totalPrice()).isEqualTo(18_000);
        }

        @Test
        void 무료_배송이_적용된다() {
            Order order = orderService.place(standardCart, vipUser);
            assertThat(order.shippingFee()).isEqualTo(0);
        }
    }

    @Nested
    @DisplayName("일반 회원 주문")
    class NormalOrderTests {

        @Test
        void 할인이_없다() {
            User normalUser = aUser().withGrade(NORMAL).build();
            Cart cart = aCart().withItem(anItem().withPrice(20_000).build()).build();

            Order order = orderService.place(cart, normalUser);

            assertThat(order.totalPrice()).isEqualTo(20_000);
        }
    }
}
```

`VipOrderTests` 안의 `@BeforeEach`는 그 블록 안의 테스트에만 적용됩니다. 컨텍스트와 픽스처의 범위가 일치합니다.

---

## 💻 실전 적용: @AfterEach와 외부 자원 정리

단위 테스트에서는 드물지만, 통합 테스트에서 파일, 네트워크, DB를 쓴다면 정리가 필요합니다.

```java
@AfterEach
void tearDown() {
    // 파일 생성 테스트 후 정리
    Files.deleteIfExists(tempFile);
}

// 더 나은 방법: @TempDir로 JUnit5가 자동 관리
@Test
void 파일_생성_테스트(@TempDir Path tempDir) {
    Path file = tempDir.resolve("output.txt");
    fileService.write(file, "내용");
    assertThat(Files.readString(file)).isEqualTo("내용");
    // tempDir은 테스트 후 자동 삭제
}
```

---

## 🤔 트레이드오프

### "@BeforeEach에 아무것도 안 넣으면 코드 중복이 심해진다"

Test Data Builder로 Arrange 코드를 간결하게 만들면, 각 테스트 안에서 만들어도 중복 부담이 줄어듭니다. `aUser().withGrade(VIP).build()` 한 줄이라면 `@BeforeEach`에 올려서 숨길 이유가 없습니다.

중복이 진짜 문제가 될 때는 `@Nested`로 컨텍스트를 묶는 것이 `@BeforeEach` 남용보다 더 명확한 해결책입니다.

---

## 📌 핵심 정리

```
@BeforeEach 올바른 사용:
  ✅ 모든 테스트에서 공통으로 필요한 인프라 초기화
     (Repository, Service 객체 생성)
  ✅ 가변 상태의 초기화 (매 테스트마다 새로 생성)

  ❌ 일부 테스트에서만 필요한 테스트 데이터
  ❌ 테스트마다 다른 값이 필요한 필드

공유 픽스처의 위험:
  static 가변 객체 → 테스트 간 상태 오염
  일부에서만 쓰는 @BeforeEach → 숨겨진 컨텍스트

해결 전략:
  테스트 안에서 직접 만들기 + Test Data Builder
  @Nested로 같은 컨텍스트끼리 묶기
  @TempDir / @ExtendWith로 외부 자원 관리

판단 기준:
  "이 @BeforeEach 필드가 모든 테스트에서 쓰이는가?"
  → 아니라면 테스트 안으로 내려야 한다
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 공유 픽스처가 만드는 문제를 찾고, 어떻게 리팩터링하겠는가?

```java
class CartServiceTest {

    private Cart cart;
    private CartService cartService;

    @BeforeEach
    void setUp() {
        cart = new Cart();   // 가변 객체!
        cart.addItem(new Item("책", 20_000));
        cartService = new CartService(cart);
    }

    @Test
    void 아이템_추가() {
        cartService.addItem(new Item("펜", 3_000));
        assertThat(cart.items()).hasSize(2);
    }

    @Test
    void 아이템_1개_있는_장바구니_총액() {
        assertThat(cart.totalPrice()).isEqualTo(20_000);
    }
}
```

**Q2.** `@BeforeEach`, `@BeforeAll`, `@Nested` 각각의 픽스처는 어느 범위에서 공유되는가? 하나의 테스트 클래스에서 이 세 가지를 어떻게 조합해서 사용할지 설계하라.

**Q3.** 단위 테스트에서는 `@AfterEach`가 거의 필요 없다고 한다. 왜 그런가? `@AfterEach`가 반드시 필요한 경우는 어떤 경우인가?

> 💡 **해설**
>
> **Q1.** 문제: `cart`가 가변 객체이고 `setUp`에서 초기화되지만, `아이템_추가` 테스트가 `cart`를 수정하면 실행 순서에 따라 `아이템_1개_있는_장바구니_총액`이 실패한다(`hasSize(2)` 상태의 cart가 남음). 또한 `CartService`가 `cart`를 생성자에서 받는 것도 테스트 설계가 이상하다(서비스가 카트를 소유?). 리팩터링: `@BeforeEach`에서 `cartService = new CartService()`만 초기화, 각 테스트에서 `Cart cart = aCart().withItem(book).build()`를 직접 만든다.
>
> **Q2.** `@BeforeAll`: 테스트 클래스당 한 번 실행(모든 테스트 인스턴스 공유), static 메서드만 가능. 무거운 초기화(DB 연결, WireMock 서버 시작)에 사용. `@BeforeEach`: 각 테스트 메서드 실행 전 실행, 테스트마다 새 인스턴스. 서비스/Repository 초기화. `@Nested` 내부의 `@BeforeEach`: 해당 중첩 클래스의 테스트에만 적용. 조합 설계: `@BeforeAll`에서 Testcontainers/외부 서버 시작 → 외부 `@BeforeEach`에서 Repository/Service 생성 → `@Nested` 내부 `@BeforeEach`에서 컨텍스트별 테스트 데이터 설정.
>
> **Q3.** 단위 테스트에서 `@AfterEach`가 거의 필요 없는 이유: `@BeforeEach`에서 매 테스트마다 새 객체를 만들기 때문에(In-memory 컬렉션, Fake 객체), GC가 알아서 처리한다. 정리할 외부 자원이 없다. `@AfterEach`가 필요한 경우: 실제 파일을 생성하는 테스트(`@TempDir`으로 대체 가능), 실제 DB 연결(→ `@Transactional` 롤백으로 대체 가능), 네트워크 서버(WireMock) 정리. 즉, `@AfterEach`의 필요성 자체가 단위 테스트가 아닌 통합 테스트임을 나타내는 신호다.

---

<div align="center">

**[⬅️ 이전: Parameterized Tests](./04-parameterized-tests.md)** | **[다음: Meaningful Assertions ➡️](./06-meaningful-assertions.md)**

</div>
