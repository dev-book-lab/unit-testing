# 03. Interface Segregation

> **인터페이스가 클수록 Stub이 복잡해진다 — 테스트하기 어려운 인터페이스는 너무 많은 것을 알고 있다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 인터페이스가 크면 테스트가 왜 복잡해지는가?
- SOLID의 ISP(Interface Segregation Principle)와 테스트 용이성은 어떻게 연결되는가?
- 역할 인터페이스(Role Interface)가 Stub을 어떻게 단순화하는가?

---

## 🔍 문제: 거대한 인터페이스의 테스트 비용

```java
// 하나의 인터페이스가 너무 많은 책임을 가진다
public interface UserRepository {
    User save(User user);
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
    List<User> findAll();
    List<User> findByGrade(Grade grade);
    List<User> findByCreatedAtBetween(LocalDate from, LocalDate to);
    void delete(Long id);
    void deleteAll();
    long count();
    boolean existsByEmail(String email);
    List<User> findInactiveUsers(Duration threshold);
    void updateGrade(Long id, Grade grade);
}
```

이 인터페이스를 Stub으로 만들어야 하는 테스트를 생각해보겠습니다.

```java
// OrderService는 findById() 하나만 필요한데...
public class OrderService {
    private final UserRepository userRepository;

    public Order place(Long userId, Cart cart) {
        User user = userRepository.findById(userId).orElseThrow();
        // user 정보로 할인 계산
        ...
    }
}
```

```java
// ❌ 하나만 필요한데 12개 메서드를 가진 인터페이스를 Stub해야 한다
UserRepository stubRepo = mock(UserRepository.class);
when(stubRepo.findById(1L)).thenReturn(Optional.of(vipUser));
// 나머지 11개는 쓰이지 않지만 인터페이스에 존재
// → Mock 객체에 설정하지 않으면 null/empty 반환 — 의도 불명확
```

Mock이나 Fake를 구현할 때 실제로 쓰이지 않는 메서드를 처리해야 합니다. Fake라면:

```java
// ❌ Fake도 12개 메서드를 모두 구현해야 한다
class InMemoryUserRepository implements UserRepository {
    @Override public User save(User user) { ... }
    @Override public Optional<User> findById(Long id) { ... }
    @Override public Optional<User> findByEmail(String email) { ... }
    @Override public List<User> findAll() { throw new UnsupportedOperationException(); }
    @Override public List<User> findByGrade(Grade grade) { throw new UnsupportedOperationException(); }
    // ... 나머지 7개도 처리
}
```

테스트에서 `findById()` 하나만 필요한데 12개 메서드를 신경 써야 합니다.

---

## 🏛️ ISP: 인터페이스를 역할 단위로 쪼갠다

ISP는 "클라이언트가 자신이 사용하지 않는 메서드에 의존하도록 강제하지 마라"는 원칙입니다. 테스트 관점에서는 "Stub이 알아야 할 것을 최소화하라"는 의미가 됩니다.

```java
// 역할별로 인터페이스를 분리
public interface UserSaver {
    User save(User user);
}

public interface UserFinder {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
}

public interface UserReader {
    List<User> findAll();
    List<User> findByGrade(Grade grade);
    long count();
}

public interface UserDeleter {
    void delete(Long id);
    void deleteAll();
}

// 실제 구현체는 필요한 인터페이스를 모두 구현
@Repository
public class JpaUserRepository implements UserSaver, UserFinder, UserReader, UserDeleter {
    // 실제 JPA 구현
}
```

이제 각 서비스는 필요한 역할만 선언합니다:

```java
// OrderService는 UserFinder만 필요하다
public class OrderService {
    private final UserFinder userFinder; // UserRepository 전체가 아닌 역할만

    public Order place(Long userId, Cart cart) {
        User user = userFinder.findById(userId).orElseThrow();
        ...
    }
}

// UserCleanupService는 UserReader와 UserDeleter만 필요하다
public class UserCleanupService {
    private final UserReader userReader;
    private final UserDeleter userDeleter;
    ...
}
```

---

## 😱 Before vs After: Stub 복잡도 비교

### Before: 큰 인터페이스

```java
// OrderServiceTest — UserRepository 전체를 Stub
@Test
void 주문_생성_시_VIP_할인이_적용된다() {
    UserRepository stubRepo = mock(UserRepository.class);
    when(stubRepo.findById(1L)).thenReturn(Optional.of(vipUser));
    // 나머지 11개 메서드는 설정 안 함 — 테스트가 의도하지 않게 호출되면 null

    OrderService service = new OrderService(stubRepo, ...);
    ...
}
```

`UserRepository`를 사용하는 다른 코드가 내부에서 `count()`나 `findAll()`을 호출해도 Mock이 `null`이나 `0`을 반환해서 예외 없이 넘어갈 수 있습니다. 의도하지 않은 호출이 감지되지 않습니다.

### After: 역할 인터페이스

```java
// OrderServiceTest — UserFinder만 Stub
@Test
void 주문_생성_시_VIP_할인이_적용된다() {
    UserFinder stubFinder = id -> Optional.of(vipUser); // 람다 Stub!

    OrderService service = new OrderService(stubFinder, ...);

    Order order = service.place(1L, cart);
    assertThat(order.discount()).isEqualTo(10);
}
```

`UserFinder`는 `findById()` 하나(또는 두 개)만 있습니다. 람다 하나로 Stub이 완성됩니다. `UserRepository`의 다른 12개 메서드는 이 테스트와 무관합니다.

---

## ✨ 역할 인터페이스 설계 패턴

### 패턴 1: 단일 메서드 인터페이스 (SAM Interface)

```java
// 람다로 구현 가능한 단일 메서드 인터페이스
@FunctionalInterface
public interface DiscountPolicy {
    int calculate(User user);
}

// 테스트에서 람다 Stub
DiscountPolicy vipPolicy = user -> 10;
DiscountPolicy noDiscount = user -> 0;
DiscountPolicy conditionalPolicy = user -> user.grade() == VIP ? 10 : 0;
```

SAM 인터페이스는 테스트에서 람다로 즉시 Stub이 됩니다. Mockito 없이도 완전한 Stub이 가능합니다.

### 패턴 2: 용도별 인터페이스 그룹

```java
// 읽기와 쓰기를 분리 — CQRS 패턴과도 자연스럽게 연결
public interface OrderCommandRepository {
    Order save(Order order);
    void delete(Long id);
}

public interface OrderQueryRepository {
    Optional<Order> findById(Long id);
    List<Order> findByUserId(Long userId);
    List<Order> findByStatus(OrderStatus status);
}

// 명령 서비스: 쓰기만
public class OrderCommandService {
    private final OrderCommandRepository commandRepo;
    private final OrderQueryRepository queryRepo; // 조회도 필요하면 추가
}

// 조회 서비스: 읽기만
public class OrderQueryService {
    private final OrderQueryRepository queryRepo;
}
```

```java
// OrderCommandServiceTest: 쓰기만 Stub
@Test
void 주문이_저장된다() {
    InMemoryOrderCommandRepository fakeRepo = new InMemoryOrderCommandRepository();
    OrderCommandService service = new OrderCommandService(fakeRepo);
    ...
}
```

`InMemoryOrderCommandRepository`는 `save()`와 `delete()` 두 메서드만 구현하면 됩니다.

---

## 💻 실전 적용: 기존 큰 인터페이스를 점진적으로 분리

기존 코드에서 당장 인터페이스를 쪼개기 어렵다면, 필요한 역할만 추출하는 방식으로 시작합니다.

```java
// 기존: JpaUserRepository implements UserRepository (12개 메서드)

// Step 1: 테스트에서 필요한 역할만 담은 인터페이스 추출
public interface UserFinder {
    Optional<User> findById(Long id);
}

// Step 2: 기존 구현체가 새 인터페이스를 구현하도록 추가
public class JpaUserRepository implements UserRepository, UserFinder {
    // 기존 코드 변경 없음 — findById()가 이미 있음
}

// Step 3: OrderService의 타입을 UserRepository에서 UserFinder로 변경
public class OrderService {
    private final UserFinder userFinder; // 범위가 좁아짐

    public OrderService(UserFinder userFinder, ...) { ... }
}

// Step 4: 테스트에서 람다 Stub 사용
OrderService service = new OrderService(id -> Optional.of(vipUser), ...);
```

기존 프로덕션 코드(`JpaUserRepository`)는 변경 없이, 테스트는 훨씬 단순해집니다.

---

## 🤔 트레이드오프

### "인터페이스가 너무 많아지면 관리하기 어렵지 않은가?"

맞습니다. 모든 메서드를 별도 인터페이스로 만드는 것은 과도합니다. 기준은 "이 메서드들이 항상 함께 쓰이는가?"입니다. 항상 함께 쓰이면 같은 인터페이스, 독립적으로 쓰이면 분리합니다.

실제로 4~5개의 역할 인터페이스가 있어도, 각 서비스가 자신이 필요한 1~2개만 선언하면 의존성이 명확합니다.

### "기존 레거시 코드에 인터페이스가 없다면?"

인터페이스를 먼저 추출하고(Extract Interface 리팩터링), 기존 구현체가 그 인터페이스를 구현하도록 합니다. 이것은 기존 동작에 영향을 주지 않고 테스트 가능성을 높이는 안전한 리팩터링입니다.

---

## 📌 핵심 정리

```
인터페이스가 크면:
  Stub/Fake가 모든 메서드를 처리해야 함
  테스트가 관련 없는 메서드를 알아야 함
  Mock 설정 코드가 길어짐

ISP와 테스트 용이성:
  역할 인터페이스 → 필요한 것만 Stub
  단일 메서드 인터페이스 → 람다로 즉시 Stub
  용도별 분리 → Fake 구현이 단순

분리 기준:
  "이 메서드들이 항상 함께 쓰이는가?"
  → YES: 같은 인터페이스
  → NO: 분리 고려

점진적 도입:
  기존 코드 변경 없이 역할 인터페이스 추가
  구현체가 새 인터페이스도 implements
  클라이언트 타입을 좁은 인터페이스로 교체

람다 Stub의 조건:
  @FunctionalInterface (메서드 1개)
  → mock(Interface.class) 없이 테스트 가능
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 `NotificationService`에서 `EmailSender`와 `SmsSender`를 하나의 인터페이스(`MessageSender`)로 합치는 것과 분리하는 것의 테스트 비용 차이를 비교하라.

```java
// 합쳐진 경우
public interface MessageSender {
    void sendEmail(String to, String subject, String body);
    void sendSms(String phone, String message);
    void sendPush(String deviceId, String message);
}

// 분리된 경우
public interface EmailSender {
    void send(String to, String subject, String body);
}
public interface SmsSender {
    void send(String phone, String message);
}
```

**Q2.** 아래 인터페이스를 역할별로 분리하고, 분리 후 각각을 람다로 Stub 가능한지 확인하라.

```java
public interface InventoryService {
    boolean isAvailable(String itemId, int quantity);
    void reserve(String itemId, int quantity);
    void release(String itemId, int quantity);
    int currentStock(String itemId);
}
```

**Q3.** ISP를 너무 극단적으로 적용해서 인터페이스마다 메서드가 1개씩인 코드가 됐다. 이것의 문제는 무엇인가? 적절한 균형을 어떻게 찾는가?

> 💡 **해설**
>
> **Q1.** 합쳐진 `MessageSender`: 이메일 발송만 필요한 `WelcomeService`를 테스트하려면 `sendSms()`와 `sendPush()`도 처리하는 Mock이나 Fake를 만들어야 한다. `mock(MessageSender.class)`에서 설정하지 않은 `sendSms()`가 내부적으로 호출돼도 감지되지 않는다. 분리된 경우: `WelcomeService(EmailSender emailSender)`는 `emailSender -> {}` 람다 하나로 Stub이 완성된다. `SmsSender`는 이 테스트와 완전히 무관하다. 테스트 코드가 훨씬 짧아지고 의도가 명확해진다.
>
> **Q2.** 분리 방안: `InventoryChecker { boolean isAvailable(String itemId, int quantity); }` → 람다 Stub 가능: `(id, qty) -> true`. `InventoryReserver { void reserve(String itemId, int quantity); }` → 람다 Stub: `(id, qty) -> {}`. `InventoryReleaser { void release(String itemId, int quantity); }` → 람다 Stub: `(id, qty) -> {}`. `StockReader { int currentStock(String itemId); }` → 람다 Stub: `id -> 100`. OrderService가 예약만 필요하다면 `InventoryReserver`만 주입받으면 된다. 실제 `JpaInventoryService`는 4개 인터페이스를 모두 구현한다.
>
> **Q3.** 극단적 ISP의 문제: ① 인터페이스 수가 폭발적으로 늘어나 패키지 구조가 복잡해진다. ② 한 서비스가 10개의 단일 메서드 인터페이스를 주입받으면 생성자 파라미터가 10개가 된다 — 이것도 설계 문제다. ③ 함께 변경되는 메서드들이 분리되면 인터페이스 변경 시 여러 곳을 수정해야 한다. 균형 찾기 기준: "이 메서드들을 함께 쓰는 클라이언트가 몇 개인가?" — 항상 같이 쓰이면 하나의 인터페이스. "이 메서드들이 독립적으로 테스트되는가?" — 독립적이면 분리. 실무에서는 읽기/쓰기 분리, 명령/조회 분리 정도가 자연스러운 균형점이 된다.

---

<div align="center">

**[⬅️ 이전: Avoiding Static Methods](./02-avoiding-static-methods.md)** | **[다음: Humble Object Pattern ➡️](./04-humble-object-pattern.md)**

</div>
