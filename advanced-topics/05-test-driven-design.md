# 05. Test-Driven Design

> **테스트가 통과하는 것이 목적이 아니라 — 테스트를 먼저 쓰면서 좋은 설계가 드러나는 것이 목적이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- "테스트하기 어렵다"는 것이 설계 문제의 신호인 이유는 무엇인가?
- 테스트를 먼저 쓰면 설계가 어떻게 달라지는가?
- 테스트 가능성과 설계 품질은 어떻게 연결되는가?

---

## 🔍 테스트하기 어렵다 = 설계가 어렵다

테스트를 먼저 쓰다가 막히는 순간을 주목해야 합니다. 막히는 이유는 항상 설계 문제에서 비롯됩니다.

```
테스트 작성이 어려운 상황 → 설계 문제의 신호

"이 클래스를 테스트하려면 DB가 필요하다"
  → 비즈니스 로직이 인프라와 분리되지 않음

"이 메서드를 테스트하려면 외부 API를 실제로 호출해야 한다"
  → 의존성이 인터페이스로 추상화되지 않음

"이 클래스를 생성하려면 10개의 의존성이 필요하다"
  → 단일 책임 원칙 위반, 클래스가 너무 많은 일을 함

"private 메서드를 테스트하고 싶다"
  → 클래스 책임이 잘못 분배됨, 별도 클래스로 추출 신호

"테스트 순서가 중요하다"
  → 공유 상태 또는 숨겨진 의존성
```

테스트를 먼저 쓰면 이 신호들이 구현 전에 드러납니다. 사후 테스트는 이미 완성된 코드를 검증하므로 설계 피드백이 늦습니다.

---

## 😱 테스트 없이 설계 → 테스트하기 어려운 코드

```java
// 테스트 없이 구현 — 동작은 하지만 테스트하기 어렵다
public class OrderProcessor {

    public void process(Long orderId) {
        // 직접 DB 조회
        Order order = new JdbcTemplate(dataSource)
            .queryForObject("SELECT * FROM orders WHERE id = ?",
                this::mapRow, orderId);

        // 비즈니스 로직
        if (order.totalPrice() >= 100_000) {
            order.applyVipDiscount();
        }

        // 외부 API 직접 호출
        new PaymentApiClient().charge(order.userId(), order.totalPrice());

        // 직접 저장
        new JdbcTemplate(dataSource)
            .update("UPDATE orders SET status = 'PROCESSED' WHERE id = ?", orderId);

        // 이메일 직접 발송
        new SmtpEmailSender().send(order.userEmail(), "처리 완료");
    }
}
```

이 클래스를 테스트하려면:
- 실제 DataSource가 필요 → DB 없이 테스트 불가
- 실제 결제 API 호출 → 테스트마다 실제 결제 발생
- SMTP 서버 필요 → 이메일 없이 테스트 불가

---

## ✨ 테스트 먼저 → 설계가 이끌려 나온다

테스트를 먼저 쓰면 구현하기 전에 "어떻게 테스트할 것인가"를 먼저 생각합니다. 이 과정에서 의존성 분리가 자연스럽게 나타납니다.

```java
// 1단계: 테스트 먼저 작성
// "주문 금액이 10만원 이상이면 VIP 할인이 적용된다"
@Test
void 고액_주문에_VIP_할인_적용() {
    // 테스트를 작성하다가 막힌다:
    // "OrderProcessor를 어떻게 만들지?"
    // "Order 객체를 어떻게 만들지?"
    // "결제는? DB는?"

    // 이 고민이 설계를 이끈다
    Order order = anOrder().withTotalPrice(100_000).build();

    // 처리하려면 필요한 것들을 파라미터로 받아야 한다는 것을 깨닫는다
    OrderProcessor processor = new OrderProcessor(
        fakeOrderRepository,
        mockPaymentGateway,
        mockEmailSender
    );

    processor.process(order.id());

    assertThat(order.discountApplied()).isTrue();
}
```

```java
// 2단계: 테스트가 설계를 이끌어 낸다
public class OrderProcessor {
    private final OrderRepository orderRepository; // 인터페이스로 분리
    private final PaymentGateway paymentGateway;   // 인터페이스로 분리
    private final EmailSender emailSender;          // 인터페이스로 분리

    public OrderProcessor(OrderRepository orderRepository,
                          PaymentGateway paymentGateway,
                          EmailSender emailSender) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.emailSender = emailSender;
    }

    public void process(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        if (order.totalPrice() >= 100_000) {
            order.applyVipDiscount(); // 비즈니스 로직 분리
        }

        paymentGateway.charge(order.userId(), order.finalPrice());
        orderRepository.save(order);
        emailSender.send(order.userEmail(), "처리 완료");
    }
}
```

테스트를 먼저 쓰는 과정에서:
- 의존성이 인터페이스로 추상화됨 (생성자 주입)
- 비즈니스 로직이 인프라에서 분리됨
- 클래스 책임이 명확해짐

---

## 🏛️ 테스트 가능성이 드러내는 설계 원칙

### 신호 1: 생성자가 복잡하다 → SRP 위반

```java
// ❌ 테스트를 쓰다가 생성자가 너무 복잡함을 발견
OrderService service = new OrderService(
    orderRepository,
    userRepository,
    couponRepository,
    inventoryClient,
    paymentGateway,
    emailSender,
    smsService,
    pushNotificationService,
    auditLogger,
    metricsCollector
);
// 10개의 의존성 → OrderService가 너무 많은 것을 한다
```

설계 개선:

```java
// NotificationService로 묶기
NotificationService notification = new NotificationService(
    emailSender, smsService, pushNotificationService);

// OrderService 책임 분리
OrderService service = new OrderService(
    orderRepository,
    inventoryClient,
    paymentGateway,
    notification  // 3개를 하나로
);
```

### 신호 2: 테스트에서 private 메서드를 직접 호출하고 싶다 → 책임 분리 신호

```java
// ❌ 이런 충동이 생긴다면
@Test
void 할인율_계산_로직_단독_검증() {
    // PriceCalculator의 private applyVipDiscount()를 직접 테스트하고 싶다
    // 하지만 private이라 접근 불가...
    // @VisibleForTesting으로 노출하면 안 되는데
}
```

```java
// ✅ private 메서드가 독립적으로 테스트할 가치가 있다면
// 그것은 별도 클래스로 추출하라는 신호
public class DiscountCalculator {
    public int calculate(User user, int price) {
        // 이제 독립적으로 테스트 가능
    }
}

@Test
void VIP_할인_계산() {
    DiscountCalculator calculator = new DiscountCalculator();
    assertThat(calculator.calculate(vipUser, 20_000)).isEqualTo(2_000);
}
```

### 신호 3: 테스트가 특정 실행 순서에 의존한다 → 숨겨진 상태

```java
// ❌ 이런 테스트가 필요하다면
@Test
@Order(1)
void 먼저_초기화() { service.init(); }

@Test
@Order(2)
void 그다음_처리() { service.process(); }
// → service에 공유 상태가 있다는 신호
// → 각 메서드가 독립적으로 동작하도록 설계 개선 필요
```

---

## 💻 실전 적용: "테스트하기 어렵다"를 설계 개선 기회로

```
테스트 작성 중 막히면 질문한다:

"DB/외부 API 없이 이 로직을 검증할 수 없는가?"
  → YES: 비즈니스 로직을 인프라에서 분리한다

"이 클래스를 생성하는 데 5개 이상의 의존성이 필요한가?"
  → YES: 관련 의존성을 묶어 책임을 분리한다

"이 private 메서드를 직접 테스트하고 싶은가?"
  → YES: 별도 클래스로 추출한다

"이 테스트가 다른 테스트 실행 후에만 통과하는가?"
  → YES: 공유 상태를 제거한다
```

---

## 🏛️ 테스트 가능성 = 느슨한 결합

테스트 가능성과 좋은 설계는 같은 것의 다른 이름입니다.

```
테스트하기 쉬운 코드의 특징:
  의존성이 인터페이스로 추상화됨 → 다형성
  의존성이 생성자로 주입됨 → DIP
  하나의 클래스가 하나의 책임 → SRP
  외부 상태에 의존하지 않음 → 순수 함수
  실행 순서에 의존하지 않음 → 독립성

이것은 모두 좋은 OOP 설계의 원칙과 일치한다
테스트 가능성은 설계 품질의 프록시(proxy)다
```

---

## 🤔 트레이드오프

### "테스트를 위해 인터페이스를 과도하게 만드는 것 아닌가?"

인터페이스가 필요한 이유가 오직 테스트 때문이라면, 그것은 Fake/Stub을 쓰기 위한 적절한 추상화입니다. 하지만 나중에 다른 구현이 생길 가능성이 전혀 없다면 인터페이스 없이 `@MockBean`으로 처리하는 것도 현실적인 선택입니다. 중요한 것은 의존성이 외부에서 주입되는 것 자체입니다.

### "TDD로 짜면 설계가 항상 좋아지는가?"

TDD는 설계를 보장하지 않습니다. "테스트하기 어렵다"는 신호에 반응하지 않고, 테스트를 억지로 통과시키는 방향으로만 가면 설계는 나아지지 않습니다. TDD는 설계 피드백 루프를 제공하는 도구입니다. 그 신호를 어떻게 해석하고 반응하는지는 개발자의 역량입니다.

---

## 📌 핵심 정리

```
테스트하기 어렵다 = 설계 문제의 신호:
  DB 없이 테스트 불가 → 인프라 분리 필요
  생성자에 의존성 10개 → SRP 위반
  private 테스트 충동 → 클래스 추출 신호
  테스트 순서 의존 → 공유 상태 존재

테스트 먼저 쓰면 달라지는 것:
  의존성이 자연스럽게 인터페이스로 추상화됨
  생성자 주입이 강제됨
  클래스 책임이 명확해짐
  비즈니스 로직이 인프라에서 분리됨

테스트 가능성과 설계 품질의 관계:
  테스트하기 쉬움 = 느슨한 결합
  느슨한 결합 = 좋은 OOP 설계
  테스트 가능성은 설계 품질의 프록시

TDD가 설계를 보장하지 않는 이유:
  신호를 인식하고 반응해야 함
  테스트를 억지로 통과시키면 설계는 나아지지 않음
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드를 단위 테스트로 작성하려 할 때 어떤 어려움이 생기는가? 그 어려움이 어떤 설계 문제를 드러내는지 분석하라.

```java
public class ReportGenerator {
    public void generate(Long reportId) {
        String data = new JdbcTemplate(DriverManager.getConnection(
            "jdbc:mysql://prod-db:3306/db", "user", "pass"))
            .queryForObject("SELECT * FROM reports WHERE id = ?",
                this::mapRow, reportId);

        String processed = data.toUpperCase(); // 비즈니스 로직

        Files.writeString(Path.of("/reports/" + reportId + ".txt"), processed);

        new SlackClient("SLACK_TOKEN").send("#reports",
            "Report " + reportId + " generated");
    }
}
```

**Q2.** "private 메서드를 테스트하고 싶다"는 충동이 생겼을 때 세 가지 선택지가 있다. 각각의 적절한 상황을 설명하라.

```
선택지 A: @VisibleForTesting으로 가시성을 높인다
선택지 B: 공개 메서드를 통해 간접 검증한다
선택지 C: 별도 클래스로 추출하고 해당 클래스를 테스트한다
```

**Q3.** TDD로 작성된 코드와 테스트 없이 작성된 코드를 비교할 때, 리팩터링 비용에서 어떤 차이가 나타나는가? 구체적인 시나리오로 설명하라.

> 💡 **해설**

**Q1.**

발생하는 어려움:

① DB 연결 (`DriverManager.getConnection`)이 하드코딩 → 테스트에서 프로덕션 DB에 연결하거나, DB 없이 테스트할 방법이 없음

② 파일 시스템 (`Files.writeString`)에 직접 쓰기 → 테스트마다 실제 파일이 생성되고, 경로가 하드코딩되어 있어 CI 환경에서 실패할 수 있음

③ 외부 API (`SlackClient`) 직접 호출 → 테스트마다 Slack 메시지 발송, API 토큰이 환경 변수가 아닌 하드코딩

드러나는 설계 문제:

`ReportGenerator`가 세 가지 인프라 관심사(DB, 파일 시스템, Slack)와 비즈니스 로직(`toUpperCase`)을 동시에 담당한다. SRP 위반. 의존성이 내부에서 직접 생성되어 교체 불가.

개선 방향:

```java
public class ReportGenerator {
    private final ReportRepository repository;
    private final ReportWriter writer;
    private final NotificationSender notifier;

    public void generate(Long reportId) {
        String data = repository.find(reportId);
        String processed = data.toUpperCase(); // 비즈니스 로직만 남음
        writer.write(reportId, processed);
        notifier.send("Report " + reportId + " generated");
    }
}
```

**Q2.**

선택지 A (`@VisibleForTesting`): 원칙적으로 피해야 하지만, private 메서드가 복잡하고 독립적으로 검증할 필요가 있는데 클래스 추출 비용이 크거나 시간이 없는 경우의 임시방편. 기술 부채로 인식하고 추후 리팩터링 계획을 세운다.

선택지 B (공개 메서드 간접 검증): private 메서드가 공개 메서드의 일부로만 의미가 있을 때. 예를 들어 `validate()`가 내부적으로 `checkAmount()`, `checkItems()`를 호출한다면, `validate()`의 결과로 두 메서드의 동작을 충분히 검증할 수 있다. 가장 일반적인 케이스.

선택지 C (별도 클래스 추출): private 메서드가 독립적인 책임을 가질 때. 예를 들어 `OrderService` 안의 `calculateDiscountedPrice()`가 복잡한 계산 로직을 담고 있다면 `DiscountCalculator`로 추출해서 독립 테스트한다. 추출 후 `OrderService`에서 `DiscountCalculator`를 주입받으면 두 클래스 모두 테스트하기 쉬워진다.

**Q3.**

시나리오: 주문 할인 계산 로직을 퍼센트 기반에서 금액 기반으로 변경한다.

TDD로 작성된 코드: `DiscountCalculator` 단위 테스트가 있다. 리팩터링 범위를 `DiscountCalculator`로 좁힐 수 있다. 변경 후 테스트를 실행하면 변경이 의도한 대로 됐는지 즉시 확인된다. 다른 클래스로의 파급 효과는 상위 통합 테스트가 잡아준다.

테스트 없이 작성된 코드: 할인 로직이 `OrderService`, `CartService`, `PromotionService` 여러 곳에 퍼져 있을 가능성이 높다. 변경 범위를 파악하기 어렵고, 변경 후 올바르게 동작하는지 직접 실행해서 확인해야 한다. "혹시 다른 곳에서 쓰이는 건 아닐까?"라는 불안감이 리팩터링을 막는다.

결론: TDD로 작성된 코드는 테스트가 변경의 안전망이 되어 리팩터링 비용이 낮다. 테스트 없이 작성된 코드는 변경이 두렵고, 변경 범위가 불명확해서 리팩터링을 미루게 된다.

---

<div align="center">

**[⬅️ 이전: TDD Workflow](./04-tdd-workflow.md)** | **[다음: Testing Legacy Code ➡️](./06-testing-legacy-code.md)**

</div>
