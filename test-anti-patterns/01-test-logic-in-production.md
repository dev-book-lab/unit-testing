# 01. Test Logic in Production

> **`if (isTest)`가 프로덕션 코드에 있다면 — 테스트가 코드를 검증하는 게 아니라 코드가 테스트를 속이고 있는 것이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 프로덕션 코드에 테스트 전용 분기가 생기는 근본 원인은 무엇인가?
- 테스트 전용 생성자나 `setXxx()` 메서드가 위험한 이유는 무엇인가?
- 이 안티패턴을 올바른 설계로 어떻게 제거하는가?

---

## 🔍 테스트 로직이 프로덕션을 오염시키는 패턴들

### 패턴 1: 환경 분기 (`isTest`)

```java
public class PaymentService {

    public PaymentResult pay(Order order, Card card) {
        // ❌ 프로덕션 코드에 테스트 분기
        if (isTestEnvironment()) {
            return PaymentResult.success("test-transaction-id");
        }

        return paymentGateway.charge(order.totalPrice(), card);
    }

    private boolean isTestEnvironment() {
        return System.getProperty("env") != null
            && System.getProperty("env").equals("test");
    }
}
```

이 코드의 테스트는 실제 `paymentGateway.charge()`를 한 번도 실행하지 않습니다. 게이트웨이 연동 버그가 있어도 테스트는 통과합니다. 프로덕션에서 처음 발견합니다.

### 패턴 2: 테스트 전용 생성자

```java
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
    private final EmailSender emailSender;

    // 프로덕션 생성자
    public OrderService() {
        this.repository = new JpaOrderRepository();
        this.paymentGateway = new TossPaymentGateway();
        this.emailSender = new SmtpEmailSender();
    }

    // ❌ 테스트를 위해 추가한 생성자
    // 프로덕션에서는 절대 쓰이지 않지만 코드베이스에 남아있음
    OrderService(OrderRepository repository,
                 PaymentGateway paymentGateway,
                 EmailSender emailSender) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
        this.emailSender = emailSender;
    }
}
```

이 패턴이 나타나는 이유는 프로덕션 생성자가 의존성을 내부에서 직접 생성하기 때문입니다. 테스트를 위해 "뒷문"을 만든 것입니다.

### 패턴 3: 테스트 전용 setter / 가시성 변경

```java
public class DiscountCalculator {

    private int baseRate = 10;

    // ❌ 테스트에서 baseRate를 바꾸기 위해 추가한 메서드
    // 프로덕션 코드가 외부 변경에 노출됨
    void setBaseRate(int rate) {  // package-private으로 타협
        this.baseRate = rate;
    }

    public int calculate(User user) {
        return user.grade() == Grade.VIP ? baseRate : 0;
    }
}
```

```java
// 테스트에서 사용
@Test
void 커스텀_할인율_적용() {
    DiscountCalculator calculator = new DiscountCalculator();
    calculator.setBaseRate(20);  // 테스트를 위해 내부 상태를 직접 변경
    assertThat(calculator.calculate(vipUser)).isEqualTo(20);
}
```

테스트가 내부 구현 세부사항(`baseRate` 필드)을 알고 직접 조작합니다. `baseRate`의 의미가 바뀌면 이 setter도, 이 테스트도 함께 무너집니다.

### 패턴 4: `@VisibleForTesting` 남용

```java
public class OrderValidator {

    public void validate(Order order) {
        validateAmount(order);
        validateItems(order);
    }

    // ❌ private이어야 할 메서드를 테스트를 위해 package-private으로 노출
    @VisibleForTesting
    void validateAmount(Order order) {
        if (order.totalPrice() < 1_000) {
            throw new InvalidAmountException("최소 주문 금액은 1,000원입니다");
        }
    }
}
```

---

## 🏛️ 근본 원인: 의존성 주입이 없다

패턴 1, 2, 3의 공통 원인은 **프로덕션 코드가 의존성을 스스로 생성**하기 때문입니다. 테스트에서 의존성을 교체할 방법이 없으니 뒷문을 만들게 됩니다.

```
테스트 전용 코드 발생의 연쇄:
  의존성을 내부에서 생성 (new)
  → 테스트에서 교체 불가
  → 테스트를 위한 뒷문 추가 (if isTest / 테스트 생성자)
  → 프로덕션 코드가 테스트 관심사에 오염
```

---

## ✨ 올바른 설계로 제거하기

### 해결책 1: 생성자 주입으로 뒷문 제거

```java
// ✅ 생성자 주입 — 테스트 전용 생성자 불필요
public class OrderService {
    private final OrderRepository repository;
    private final PaymentGateway paymentGateway;
    private final EmailSender emailSender;

    // 하나의 생성자로 통일 — 프로덕션과 테스트 모두 사용
    public OrderService(OrderRepository repository,
                        PaymentGateway paymentGateway,
                        EmailSender emailSender) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
        this.emailSender = emailSender;
    }
}
```

```java
// 프로덕션: Spring이 실제 구현 주입
@Bean
OrderService orderService(JpaOrderRepository repo,
                           TossPaymentGateway gateway,
                           SmtpEmailSender sender) {
    return new OrderService(repo, gateway, sender);
}

// 테스트: 원하는 구현 주입
OrderService service = new OrderService(
    new InMemoryOrderRepository(),
    mock(PaymentGateway.class),
    email -> {}
);
```

### 해결책 2: 정책을 파라미터로 주입 — setter 제거

```java
// ✅ baseRate를 외부에서 주입
public class DiscountCalculator {
    private final int baseRate;

    public DiscountCalculator(int baseRate) {
        this.baseRate = baseRate;
    }

    public int calculate(User user) {
        return user.grade() == Grade.VIP ? baseRate : 0;
    }
}

// 테스트
@Test
void 커스텀_할인율_적용() {
    DiscountCalculator calculator = new DiscountCalculator(20);
    assertThat(calculator.calculate(vipUser)).isEqualTo(20);
}

@Test
void 기본_할인율_적용() {
    DiscountCalculator calculator = new DiscountCalculator(10);
    assertThat(calculator.calculate(vipUser)).isEqualTo(10);
}
```

### 해결책 3: 환경 분기 대신 전략 패턴

```java
// ✅ 테스트 환경 분기 없이 교체 가능한 구현
public interface PaymentGateway {
    PaymentResult charge(int amount, Card card);
}

// 프로덕션 구현
public class TossPaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(int amount, Card card) {
        return tossApi.charge(amount, card.number());
    }
}

// 테스트용 Stub — 프로덕션 코드에 없음
class StubPaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(int amount, Card card) {
        return PaymentResult.success("stub-transaction-id");
    }
}

// PaymentService — if (isTest) 없음
public class PaymentService {
    private final PaymentGateway paymentGateway;

    public PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public PaymentResult pay(Order order, Card card) {
        return paymentGateway.charge(order.totalPrice(), card);  // 항상 같은 코드
    }
}
```

### 해결책 4: private 메서드는 공개 메서드로 간접 검증

```java
// ✅ validateAmount()를 직접 호출하지 않고
// validate()를 통해 간접 검증
@Test
void 최소_금액_미만_주문은_예외가_발생한다() {
    Order order = anOrder().withTotalPrice(999).build();

    assertThatThrownBy(() -> validator.validate(order))
        .isInstanceOf(InvalidAmountException.class)
        .hasMessageContaining("1,000원");
}
```

`validateAmount()`가 `private`이면 외부에서 직접 호출할 수 없습니다. 그 결과가 `validate()`의 동작으로 드러나므로, 공개 인터페이스를 통해 검증합니다.

---

## 💻 실전 적용: 발견 방법

```
프로덕션 코드에서 아래 징후를 찾는다:

코드에서:
  "test", "isTest", "TestMode" 가 포함된 조건문
  @VisibleForTesting 어노테이션
  package-private 메서드나 필드 (이유 없이)
  테스트 클래스 이름을 import하는 프로덕션 코드

설계에서:
  의존성을 내부에서 new로 직접 생성
  두 번째 생성자가 외부에서 주입 받는 형태
  setter가 있지만 프로덕션에서 한 번도 호출되지 않음
```

---

## 🤔 트레이드오프

### "`@VisibleForTesting`은 문서화 목적으로 괜찮지 않은가?"

의도를 문서화하는 것은 좋지만, 의도를 표현하기 위해 실제 가시성을 변경하는 것은 문제입니다. `private`이어야 할 메서드가 `package-private`으로 노출되면 같은 패키지의 다른 코드가 이 메서드에 의존하게 될 수 있습니다. 더 나은 방법은 공개 인터페이스를 통해 동작을 검증하는 것입니다.

### "테스트 프로파일(`@Profile("test")`)로 다른 Bean을 주입하면 되지 않는가?"

가능합니다. 하지만 이 방식은 프로덕션 설정과 테스트 설정이 암묵적으로 분리되어 있어 추적이 어렵습니다. 생성자 주입 + 테스트에서 명시적으로 구현을 전달하는 방식이 더 명확합니다.

---

## 📌 핵심 정리

```
테스트가 프로덕션을 오염시키는 패턴:
  if (isTest) 분기
  테스트 전용 생성자 (두 번째 생성자)
  테스트를 위한 setter / 가시성 변경
  @VisibleForTesting 남용

근본 원인:
  의존성을 내부에서 직접 생성 (new)
  → 교체 수단이 없어 뒷문 생성

해결 방법:
  생성자 주입 → 테스트 전용 생성자 불필요
  정책/설정을 파라미터로 → setter 불필요
  인터페이스 추상화 → isTest 분기 불필요
  공개 인터페이스로 간접 검증 → @VisibleForTesting 불필요

발견 신호:
  프로덕션 코드에 "test" 문자열
  사용되지 않는 setter
  두 번째 생성자가 외부 주입 형태
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 안티패턴을 찾고, 올바른 설계로 리팩터링하라.

```java
public class NotificationService {
    private boolean testMode = false;

    public void setTestMode(boolean testMode) {
        this.testMode = testMode;
    }

    public void send(User user, String message) {
        if (testMode) {
            System.out.println("[TEST] 알림 전송: " + message);
            return;
        }
        smsClient.send(user.phone(), message);
        emailClient.send(user.email(), message);
    }
}
```

**Q2.** `@VisibleForTesting`으로 노출된 private 메서드를 테스트 전용 setter 없이 검증하는 방법을 설명하라. 다음 메서드를 예시로 사용하라.

```java
public class PriceCalculator {

    @VisibleForTesting
    int applyVat(int price) {
        return (int)(price * 1.1);
    }

    public int finalPrice(int price) {
        return applyVat(price);
    }
}
```

**Q3.** 아래 코드는 프로덕션 생성자와 테스트 생성자가 공존한다. 이 코드가 점차 유지보수하기 어려워지는 과정을 설명하고, 어떻게 해결하는가?

```java
public class ReportService {
    private final ReportRepository reportRepository;
    private final PdfGenerator pdfGenerator;

    // 프로덕션
    public ReportService() {
        this.reportRepository = new JdbcReportRepository();
        this.pdfGenerator = new ITextPdfGenerator();
    }

    // 테스트
    ReportService(ReportRepository reportRepository, PdfGenerator pdfGenerator) {
        this.reportRepository = reportRepository;
        this.pdfGenerator = pdfGenerator;
    }
}
```

> 💡 **해설**

**Q1.**

안티패턴: `testMode` 플래그와 setter. `smsClient`, `emailClient`가 내부에서 직접 생성된다고 가정하면 이것도 문제다.

리팩터링:

```java
// ✅ NotificationSender 인터페이스로 추상화
public interface NotificationSender {
    void send(String phone, String email, String message);
}

// 프로덕션 구현
public class RealNotificationSender implements NotificationSender {
    private final SmsClient smsClient;
    private final EmailClient emailClient;

    @Override
    public void send(String phone, String email, String message) {
        smsClient.send(phone, message);
        emailClient.send(email, message);
    }
}

// 테스트 Stub — 프로덕션 코드에 없음
class RecordingNotificationSender implements NotificationSender {
    final List<String> sent = new ArrayList<>();

    @Override
    public void send(String phone, String email, String message) {
        sent.add(message);
    }
}

// NotificationService — testMode 없음
public class NotificationService {
    private final NotificationSender sender;

    public NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    public void send(User user, String message) {
        sender.send(user.phone(), user.email(), message);
    }
}
```

**Q2.**

`applyVat()`를 직접 테스트할 필요가 없다. `finalPrice()`를 통해 간접 검증한다.

```java
@Test
void 부가세가_포함된_최종_가격() {
    PriceCalculator calculator = new PriceCalculator();
    assertThat(calculator.finalPrice(10_000)).isEqualTo(11_000);
}

@Test
void 소수점_내림_처리() {
    PriceCalculator calculator = new PriceCalculator();
    assertThat(calculator.finalPrice(9_999)).isEqualTo(10_998); // 9999 * 1.1 = 10998.9 → 10998
}
```

`applyVat()`가 `private`이어야 한다면 가시성을 변경할 이유가 없다. `@VisibleForTesting`과 `package-private`을 제거하고, `finalPrice()`를 통한 경계값 테스트로 동작을 완전히 검증할 수 있다.

만약 `applyVat()`가 독립적으로 테스트할 만큼 복잡하다면, 별도 클래스로 추출하는 신호다.

**Q3.**

유지보수가 어려워지는 과정:

① `ReportRepository`에 메서드가 추가되면 `JdbcReportRepository`와 테스트용 Fake 모두 업데이트해야 한다. 프로덕션 생성자와 테스트 생성자가 분리되어 있어 변경 범위가 불명확하다.

② 새로운 의존성(예: `EmailSender`)을 추가하면 프로덕션 생성자 + 테스트 생성자 두 곳 모두 수정해야 한다. 테스트 생성자를 수정하지 않으면 컴파일 오류.

③ 시간이 지나면 "왜 생성자가 두 개인가?"를 파악하기 어려워진다. 새 팀원이 프로덕션 코드에서 테스트 생성자를 우연히 사용할 수도 있다.

해결:

```java
// ✅ 생성자 하나로 통일
public class ReportService {
    private final ReportRepository reportRepository;
    private final PdfGenerator pdfGenerator;

    public ReportService(ReportRepository reportRepository,
                         PdfGenerator pdfGenerator) {
        this.reportRepository = reportRepository;
        this.pdfGenerator = pdfGenerator;
    }
}

// Spring 설정에서 실제 구현 조립
@Bean
ReportService reportService(JdbcReportRepository repo,
                             ITextPdfGenerator generator) {
    return new ReportService(repo, generator);
}

// 테스트에서 원하는 구현 주입
ReportService service = new ReportService(
    new InMemoryReportRepository(),
    new NoOpPdfGenerator()
);
```

생성자가 하나이므로 의존성 추가 시 Spring 설정과 테스트 설정 모두 컴파일 타임에 강제된다.

---

<div align="center">

**[다음: Sleepy Tests ➡️](./02-sleepy-tests.md)**

</div>
