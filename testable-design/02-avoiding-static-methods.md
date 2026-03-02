# 02. Avoiding Static Methods

> **static은 전역 상태다 — 교체할 수 없고, 격리할 수 없고, 그래서 테스트할 수 없다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- static 메서드가 테스트를 방해하는 구체적인 이유는 무엇인가?
- PowerMock 없이 static 의존성을 해결하는 방법은 무엇인가?
- static이 적합한 경우와 그렇지 않은 경우는 어떻게 구분하는가?

---

## 🔍 static 메서드의 테스트 불가 이유

```java
public class OrderService {

    public Order place(Cart cart, User user) {
        // 세 가지 static 의존성
        String traceId = TraceContext.currentTraceId();          // 추적 ID 생성
        int discount = DiscountCalculator.calculate(user);       // 할인 계산
        LocalDate today = DateUtils.today();                     // 현재 날짜

        Order order = Order.from(cart, user, discount, traceId, today);
        return orderRepository.save(order);
    }
}
```

이 코드의 테스트를 작성해보겠습니다.

```java
@Test
void VIP_할인_테스트() {
    Order order = orderService.place(cart, vipUser);
    assertThat(order.discount()).isEqualTo(10);
}
```

문제가 하나씩 드러납니다.

**`TraceContext.currentTraceId()`:** 이 메서드가 내부에서 ThreadLocal이나 외부 트레이싱 시스템에 접근하면, 테스트 환경에서 null이나 예외가 발생할 수 있습니다. 교체할 방법이 없습니다.

**`DiscountCalculator.calculate(user)`:** VIP 할인율을 10%에서 15%로 바꾸는 테스트를 하려면? static이라 다른 구현으로 교체할 수 없습니다. 실제 `DiscountCalculator`의 로직이 항상 실행됩니다.

**`DateUtils.today()`:** "오늘" 기준의 유효성 검증이 있다면 테스트 실행 날짜에 따라 결과가 달라집니다. FIRST의 Repeatable 원칙 위반입니다.

---

## 😱 PowerMock: 증상이 아닌 원인을 치료하지 않는다

static 메서드를 Mock하기 위해 PowerMock 또는 Mockito의 `MockedStatic`을 쓰는 방법이 있습니다.

```java
// Mockito 3.4+의 MockedStatic
@Test
void static_메서드_mock_예제() {
    try (MockedStatic<DiscountCalculator> mocked =
             mockStatic(DiscountCalculator.class)) {

        mocked.when(() -> DiscountCalculator.calculate(vipUser))
              .thenReturn(15);

        Order order = orderService.place(cart, vipUser);

        assertThat(order.discount()).isEqualTo(15);
    }
}
```

이것이 동작하지만, 여러 문제가 있습니다.

**1. try-with-resources가 필수다.** `MockedStatic`은 스레드 로컬에 Mock을 등록하므로 반드시 닫아야 합니다. 닫지 않으면 다른 테스트가 오염됩니다.

**2. 테스트가 구현 세부사항에 묶인다.** `DiscountCalculator`라는 구체 클래스 이름이 테스트에 하드코딩됩니다. 내부 구현을 바꾸면 테스트도 바꿔야 합니다.

**3. static 사용 자체가 설계 문제의 신호다.** MockedStatic은 설계 문제를 해결하지 않고 감춥니다.

---

## ✨ static 의존성을 제거하는 방법

### 방법 1: 인스턴스 메서드로 전환 + 생성자 주입

```java
// ❌ Before: static
public class DiscountCalculator {
    public static int calculate(User user) {
        if (user.grade() == Grade.VIP) return 10;
        return 0;
    }
}

// ✅ After: 인터페이스 + 인스턴스
public interface DiscountPolicy {
    int calculate(User user);
}

public class RateDiscountPolicy implements DiscountPolicy {
    private final int rate;

    public RateDiscountPolicy(int rate) {
        this.rate = rate;
    }

    @Override
    public int calculate(User user) {
        if (user.grade() == Grade.VIP) return rate;
        return 0;
    }
}
```

```java
// OrderService에서 주입받아 사용
public class OrderService {
    private final DiscountPolicy discountPolicy;

    public OrderService(DiscountPolicy discountPolicy, ...) {
        this.discountPolicy = discountPolicy;
    }

    public Order place(Cart cart, User user) {
        int discount = discountPolicy.calculate(user); // 교체 가능
        ...
    }
}

// 테스트: 15% 할인 정책으로 교체
OrderService service = new OrderService(user -> 15, ...);
```

### 방법 2: Clock 패턴 — 시간 의존성

```java
// ❌ Before
public class CouponService {
    public boolean isValid(Coupon coupon) {
        return !coupon.expiresAt().isBefore(LocalDate.now()); // static
    }
}

// ✅ After: Clock 주입
public class CouponService {
    private final Clock clock;

    public CouponService(Clock clock) {
        this.clock = clock;
    }

    public boolean isValid(Coupon coupon) {
        return !coupon.expiresAt().isBefore(LocalDate.now(clock));
    }
}

// 프로덕션: Clock.systemDefaultZone()
// 테스트:   Clock.fixed(특정시각, ZoneOffset.UTC)
```

### 방법 3: 래퍼 객체(Wrapper) — 외부 라이브러리 static

외부 라이브러리의 static 메서드를 직접 제어할 수 없을 때, 래퍼 인터페이스를 만듭니다.

```java
// 외부 라이브러리: UUID.randomUUID() 교체 불가
public class Order {
    public static Order create(Cart cart) {
        String id = UUID.randomUUID().toString(); // static
        return new Order(id, cart);
    }
}

// ✅ After: IdGenerator 인터페이스로 추상화
public interface IdGenerator {
    String generate();
}

public class UuidGenerator implements IdGenerator {
    @Override
    public String generate() {
        return UUID.randomUUID().toString();
    }
}

// 테스트용 고정 ID 생성기
IdGenerator fixedGenerator = () -> "test-order-id-001";

// 테스트에서 예측 가능한 ID를 검증할 수 있음
Order order = orderFactory.create(cart, fixedGenerator);
assertThat(order.id()).isEqualTo("test-order-id-001");
```

---

## 🏛️ static이 적합한 경우

모든 static이 나쁜 것은 아닙니다. **순수 함수** 형태의 static은 테스트 문제를 만들지 않습니다.

```java
// ✅ 외부 상태 없음, 입력만으로 출력이 결정됨 — 순수 함수
public class PriceCalculator {
    public static int applyVat(int price) {
        return (int) (price * 1.1);
    }

    public static int roundDown(int price, int unit) {
        return (price / unit) * unit;
    }
}
```

```java
// ✅ 테스트하기 쉬움 — static이지만 순수함수
assertThat(PriceCalculator.applyVat(10_000)).isEqualTo(11_000);
assertThat(PriceCalculator.roundDown(9_870, 100)).isEqualTo(9_800);
```

순수 함수인 static은 교체할 필요가 없습니다. 언제나 같은 입력에 같은 출력을 냅니다.

### static을 피해야 하는 경우

```
외부 상태를 읽는다 → LocalDate.now(), System.currentTimeMillis()
외부 시스템에 접근한다 → DB, API, 파일 시스템
전역 상태를 변경한다 → static 변수에 쓰기
다형성이 필요하다 → 구현을 교체해야 하는 경우
```

---

## 💻 실전 적용: 레거시 static을 점진적으로 제거

한 번에 모든 static을 제거하기 어렵다면, 래퍼 패턴으로 점진적으로 전환합니다.

```java
// 1단계: 래퍼 인터페이스 도입
public interface TraceIdProvider {
    String currentTraceId();
}

// 2단계: 기존 static을 래핑하는 기본 구현
public class DefaultTraceIdProvider implements TraceIdProvider {
    @Override
    public String currentTraceId() {
        return TraceContext.currentTraceId(); // 기존 static 유지
    }
}

// 3단계: 테스트용 구현
public class FixedTraceIdProvider implements TraceIdProvider {
    private final String fixed;

    public FixedTraceIdProvider(String fixed) { this.fixed = fixed; }

    @Override
    public String currentTraceId() { return fixed; }
}

// 4단계: OrderService가 인터페이스를 주입받음
public class OrderService {
    private final TraceIdProvider traceIdProvider;
    ...
}

// 나중에: TraceContext 내부를 개선하더라도 OrderService는 영향 없음
```

---

## 🤔 트레이드오프

### "유틸리티 클래스는 static이 자연스럽지 않는가?"

유틸리티 메서드가 순수 함수라면 static이 적합합니다. 문제는 "유틸리티"라는 이름으로 외부 상태에 접근하는 static을 정당화할 때입니다. `DateUtils.today()`나 `SecurityUtils.currentUser()` 같은 것들이 대표적인 위험입니다.

### "MockedStatic이 있는데 굳이 설계를 바꿔야 하는가?"

MockedStatic은 이미 작성된 레거시 코드에서 단기적으로 유용합니다. 하지만 새로 작성하는 코드에 static 의존성을 두고 MockedStatic으로 해결하는 것은 기술 부채를 쌓는 것입니다. 설계를 고치면 MockedStatic 자체가 필요 없어집니다.

---

## 📌 핵심 정리

```
static 메서드가 테스트를 방해하는 이유:
  교체(Override)할 수 없음
  다형성 불가 — 인터페이스로 추상화 불가
  외부 상태에 접근하면 Repeatable 위반

PowerMock/MockedStatic의 문제:
  설계 문제를 해결하지 않고 감춤
  try-with-resources 필수 — 닫지 않으면 오염
  구체 클래스 이름이 테스트에 하드코딩

해결 방법:
  계산 로직 → 인터페이스 + 생성자 주입
  현재 시간 → Clock 주입
  외부 라이브러리 → 래퍼 인터페이스

static이 괜찮은 경우:
  순수 함수 (외부 상태 없음, 입력 → 출력만)
  교체 필요 없는 유틸리티 계산

판단 기준:
  "이 static을 다른 구현으로 교체해야 할 상황이 있는가?"
  → YES: 인스턴스 메서드 + 인터페이스
  → NO: static 유지 가능
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 static 의존성을 찾고, 각각을 어떻게 교체하겠는가?

```java
public class AuditService {

    public void recordLogin(String email) {
        String ip = RequestContext.getClientIp();       // static
        LocalDateTime time = LocalDateTime.now();       // static
        String hash = HashUtils.sha256(email + ip);    // static
        auditRepository.save(new AuditLog(email, ip, time, hash));
    }
}
```

**Q2.** `HashUtils.sha256()`와 `RequestContext.getClientIp()`를 static으로 두는 것이 문제인지 각각 판단하라. 순수 함수인가 아닌가?

**Q3.** MockedStatic을 사용하는 기존 레거시 테스트가 100개 있다. 이것을 한 번에 제거하지 않고 점진적으로 개선하는 전략을 설명하라.

> 💡 **해설**
>
> **Q1.** `RequestContext.getClientIp()`: 외부 상태(HTTP 요청 컨텍스트)에 접근하는 static. `ClientIpProvider` 인터페이스를 만들고 주입받는다. 테스트: `ClientIpProvider stubIp = () -> "127.0.0.1"`. `LocalDateTime.now()`: 현재 시간에 접근하는 static. `Clock` 주입으로 해결. `HashUtils.sha256(email + ip)`: 순수 함수일 가능성이 높다. 입력이 같으면 항상 같은 해시를 반환한다면 static을 유지해도 된다. 단, 테스트에서 "어떤 해시가 생성됐는지" 예측하기 어렵다면 `Hasher` 인터페이스로 추상화하고 테스트에서는 `input -> "hashed-" + input` 같은 Stub을 쓴다.
>
> **Q2.** `HashUtils.sha256(input)`: 외부 상태 없음, 항상 같은 입력에 같은 출력 → 순수 함수. static 유지 가능. 단, 테스트에서 "생성된 해시를 예측해야 한다면" 래퍼 인터페이스가 유용할 수 있다. `RequestContext.getClientIp()`: HTTP 요청 스레드 로컬 또는 서블릿 컨텍스트에서 값을 읽는다 → 외부 상태 접근. 순수 함수가 아님. static으로 두면 테스트 환경에서 null이나 예외가 발생한다.
>
> **Q3.** 단계적 전략: ① 새로 작성하는 코드에는 static 의존성을 두지 않는다 (확산 방지). ② MockedStatic을 쓰는 테스트 중 자주 실패하거나(Flaky) 변경 빈도가 높은 것부터 우선 리팩터링한다. ③ 리팩터링 순서: static 메서드를 쓰는 클래스에 인터페이스 도입 → 래퍼 구현 추가 → 주입받도록 변경 → MockedStatic 제거. ④ 한 클래스씩 처리하고, 처리 전후를 PR 단위로 분리한다. 한 번에 하지 않는 것이 핵심이다.

---

<div align="center">

**[⬅️ 이전: DI for Testability](./01-dependency-injection-for-testability.md)** | **[다음: Interface Segregation ➡️](./03-interface-segregation.md)**

</div>
