# 04. Partial Mocking with Spy

> **Spy가 필요하다는 느낌이 들 때, 먼저 설계를 의심하라**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Spy는 Mock과 무엇이 다른가?
- Spy가 실제로 유용한 경우와 설계 문제를 숨기는 경우를 어떻게 구분하는가?
- Spy 없이 같은 목적을 달성할 수 있는 설계 변경은 무엇인가?

---

## 🔍 Spy란 무엇인가

Mock은 실제 구현을 가지지 않는 빈 껍데기입니다. 설정하지 않은 메서드는 기본값(`null`, `0`, `false`)을 반환합니다.

Spy는 **실제 객체를 감싸는** 래퍼입니다. 설정하지 않은 메서드는 실제 구현이 실행됩니다.

```java
DiscountPolicy realPolicy = new RateDiscountPolicy(10);

// Mock: 모든 메서드가 기본값 반환
DiscountPolicy mock = mock(DiscountPolicy.class);
mock.calculate(vipUser);  // → 0 (설정 없음)

// Spy: 설정하지 않은 메서드는 실제 구현 실행
DiscountPolicy spy = spy(realPolicy);
spy.calculate(vipUser);   // → 실제 RateDiscountPolicy.calculate() 실행
```

---

## 🏛️ Spy가 유용한 경우

### 1. 레거시 코드에서 일부 메서드만 교체할 때

```java
// 레거시 코드: 생성자에서 외부 API를 직접 호출 (테스트하기 어려운 설계)
public class LegacyOrderService {
    private final ExchangeRateApi api;

    public LegacyOrderService() {
        this.api = new HttpExchangeRateApi(); // 직접 생성 — 교체 불가
    }

    public int convertToKrw(int usd) {
        double rate = api.getUsdKrwRate(); // 실제 API 호출
        return (int) (usd * rate);
    }

    public Order place(Cart cart, User user) {
        // convertToKrw를 내부적으로 사용
    }
}
```

```java
// Spy로 convertToKrw만 교체
LegacyOrderService spyService = spy(new LegacyOrderService());
doReturn(1_300).when(spyService).convertToKrw(anyInt()); // 환율 고정

// place()는 실제 구현 실행, convertToKrw만 교체됨
Order order = spyService.place(cart, user);
```

레거시 코드를 당장 리팩터링하기 어려울 때 Spy가 임시방편으로 유용합니다. 하지만 이것은 리팩터링의 이정표가 되어야 합니다.

### 2. 추상 클래스 테스트

```java
// 템플릿 메서드 패턴에서 추상 클래스를 직접 테스트할 때
abstract class ReportGenerator {
    public final String generate() {
        String data = fetchData();      // 추상 메서드 — 서브클래스에서 구현
        return format(data);            // 공통 로직
    }

    protected abstract String fetchData();

    private String format(String data) { ... }
}

// Spy로 추상 메서드만 Stub
ReportGenerator spy = spy(ReportGenerator.class); // 추상 클래스 인스턴스화
doReturn("raw data").when(spy).fetchData();

String result = spy.generate();
assertThat(result).contains("formatted: raw data");
```

---

## 😱 Spy가 설계 문제를 숨기는 경우

### 패턴 1: 자기 자신의 메서드를 Mock하는 Spy

```java
public class OrderService {
    public Order place(Cart cart, User user) {
        int discount = calculateDiscount(user);  // 내부 메서드 호출
        Order order = Order.from(cart, discount);
        repository.save(order);
        return order;
    }

    private int calculateDiscount(User user) {
        // 복잡한 할인 로직
    }
}
```

```java
// ❌ 자기 자신의 calculateDiscount를 Stub
OrderService spy = spy(new OrderService(repository));
doReturn(10).when(spy).calculateDiscount(any()); // private 메서드는 못 함

// → 이게 안 되니까 package-private으로 바꾸는 시도 → 설계 훼손
```

`calculateDiscount()`가 테스트하기 어렵다면 그것을 **별도 클래스로 분리**하는 것이 올바른 해결입니다.

```java
// ✅ 올바른 해결: DiscountCalculator 분리
public class OrderService {
    private final DiscountCalculator discountCalculator; // 주입받음

    public Order place(Cart cart, User user) {
        int discount = discountCalculator.calculate(user);
        // ...
    }
}

// 테스트에서 DiscountCalculator를 Stub
DiscountCalculator stubCalculator = user -> 10;
OrderService service = new OrderService(stubCalculator, repository);
```

### 패턴 2: Spy가 지속적으로 필요한 클래스

```java
// 매 테스트마다 spy가 필요하다면
@BeforeEach
void setUp() {
    emailService = spy(new EmailService());
    doReturn(true).when(emailService).isServerReachable(); // 외부 상태 차단
}
```

`isServerReachable()`이 프로덕션 코드에서 직접 호출된다는 것은 `EmailService`가 외부 의존성을 내부에서 직접 생성하고 있다는 신호입니다.

```java
// ✅ 올바른 해결: 의존성을 주입받도록 변경
public class EmailService {
    private final SmtpConnector smtpConnector; // 주입

    public void send(String to, String body) {
        if (!smtpConnector.isReachable()) throw new MailServerException();
        // ...
    }
}

// 테스트에서 SmtpConnector를 Stub
SmtpConnector stubConnector = () -> true;
EmailService service = new EmailService(stubConnector);
```

---

## ✨ Spy 대신 설계를 바꾸는 체크리스트

```
Spy가 필요하다고 느낄 때 먼저 확인한다

"자기 자신의 메서드를 Spy하려는가?"
    → 그 메서드를 별도 클래스로 분리하라

"외부 상태에 직접 접근하는 메서드를 Spy하려는가?"
    → 그 의존성을 생성자 주입으로 바꾸어라

"레거시 코드여서 어쩔 수 없는가?"
    → Spy를 임시로 쓰되, TODO로 리팩터링 계획을 남겨라

"추상 클래스의 공통 로직을 테스트하려는가?"
    → 이 경우 Spy가 적합하다
```

---

## 💻 실전 적용: doReturn vs when의 Spy에서의 중요성

```java
RateDiscountPolicy realPolicy = new RateDiscountPolicy(10);
RateDiscountPolicy spy = spy(realPolicy);

// ❌ when().thenReturn() — 실제 메서드가 먼저 호출된다
when(spy.calculate(vipUser)).thenReturn(20);
// 내부적으로 spy.calculate(vipUser)를 실제 호출 → 부수효과 발생 가능

// ✅ doReturn().when() — 실제 메서드 호출 없이 덮어쓴다
doReturn(20).when(spy).calculate(vipUser);
// 실제 calculate() 실행 없이 20 반환으로 교체
```

Spy에서 메서드를 덮어쓸 때는 **반드시 `doReturn().when()` 형식**을 사용합니다.

---

## 🤔 트레이드오프

### "Spy가 나쁜 것이라면 왜 Mockito에 있는가?"

Spy는 **레거시 코드 테스트**와 **추상 클래스 테스트**라는 합당한 사용처가 있습니다. Mockito가 Spy를 제공하는 것은 맞지만, 새로 작성하는 코드에서 Spy가 반복적으로 필요하다면 설계 개선의 신호로 봐야 합니다.

### "private 메서드를 테스트하고 싶을 때 Spy를 써야 하지 않는가?"

private 메서드는 직접 테스트할 필요가 없습니다. 그 메서드를 호출하는 public 메서드를 통해 간접적으로 검증됩니다. private 메서드를 직접 테스트하고 싶다면 그것이 별도 클래스로 분리되어야 할 로직임을 의미합니다.

---

## 📌 핵심 정리

```
Spy = 실제 객체 래퍼
  설정하지 않은 메서드 → 실제 구현 실행
  설정한 메서드 → Stub처럼 동작

Spy의 올바른 사용:
  레거시 코드에서 일부 메서드만 임시 교체
  추상 클래스의 공통 로직 테스트

Spy가 설계 문제를 숨기는 경우:
  자기 메서드를 Spy → 별도 클래스로 분리
  외부 상태 직접 접근 → 의존성 주입으로

Spy에서의 Stub 설정:
  반드시 doReturn().when() 사용
  when().thenReturn()은 실제 메서드를 먼저 호출

판단 기준:
  "이 Spy가 임시방편인가, 설계 문제의 징후인가?"
  → 임시방편이라면 TODO와 함께 사용
  → 징후라면 설계를 먼저 고쳐라
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에서 Spy가 필요한 이유를 분석하고, Spy 없이 테스트할 수 있는 설계 변경을 제안하라.

```java
public class ReportService {
    public String generateSalesReport(LocalDate date) {
        List<Order> orders = fetchOrders(date);   // DB 조회
        return buildReport(orders);               // 포맷팅
    }

    private List<Order> fetchOrders(LocalDate date) {
        // JdbcTemplate으로 직접 조회
    }

    private String buildReport(List<Order> orders) {
        // 통계 계산 및 포맷팅
    }
}

// 현재 테스트 방식
ReportService spy = spy(reportService);
doReturn(mockOrders).when(spy).fetchOrders(any()); // private이라 안 됨
```

**Q2.** 다음 두 접근 중 어느 것이 더 나은가? 이유를 설명하라.

```java
// 접근 A: Spy로 내부 메서드 교체
NotificationService spy = spy(new NotificationService());
doReturn("template content").when(spy).loadTemplate(anyString());

// 접근 B: 의존성 주입으로 교체
TemplateLoader stubLoader = name -> "template content";
NotificationService service = new NotificationService(stubLoader);
```

**Q3.** 레거시 코드에 Spy를 임시로 적용했다. 이후 리팩터링 방향을 어떻게 계획하겠는가? Spy를 제거하기 위한 단계를 설명하라.

> 💡 **해설**
>
> **Q1.** Spy가 필요한 이유: `fetchOrders()`가 private이고 DB를 직접 호출하기 때문에 테스트에서 교체할 방법이 없다. private이라 Spy로도 교체가 안 된다. 설계 변경: `fetchOrders()`를 `OrderRepository` 인터페이스로 추출하고 주입받는다. `public class ReportService { private final OrderRepository orderRepository; public String generateSalesReport(LocalDate date) { List<Order> orders = orderRepository.findByDate(date); return buildReport(orders); } }`. 테스트: `OrderRepository fakeRepo = date -> mockOrders; ReportService service = new ReportService(fakeRepo);`. 핵심: DB 접근을 인터페이스 뒤로 숨기면 Spy가 필요 없어진다.
>
> **Q2.** 접근 B가 낫다. 이유: ① B는 `TemplateLoader`라는 명시적 의존성을 만들었다. 어떤 의존성이 있는지 생성자 시그니처만 봐도 알 수 있다. ② A는 `NotificationService` 내부에 `loadTemplate()`이라는 메서드가 있다는 구현 세부사항을 테스트가 알고 있다. 내부 구현을 변경하면 테스트가 깨진다. ③ B는 `TemplateLoader` 인터페이스를 다양한 구현으로 교체할 수 있어 유연하다. ④ A의 Spy 사용은 `NotificationService`의 설계 문제(의존성을 직접 생성)를 숨기는 것이다.
>
> **Q3.** 단계: ① 현재 Spy가 교체하는 메서드가 무엇인지 파악 (예: `isServerReachable()`). ② 그 메서드가 사용하는 외부 자원 식별 (예: SMTP 서버 연결). ③ 인터페이스 추출: `SmtpConnector { boolean isReachable(); }`. ④ 실제 구현 클래스 생성: `HttpSmtpConnector implements SmtpConnector`. ⑤ `EmailService` 생성자에 `SmtpConnector`를 주입받도록 변경. ⑥ Spy 제거, Stub으로 대체: `SmtpConnector alwaysReachable = () -> true;`. ⑦ 프로덕션에서는 `HttpSmtpConnector`를 DI 컨테이너로 주입.

---

<div align="center">

**[⬅️ 이전: Mockito Best Practices](./03-mockito-best-practices.md)** | **[다음: Argument Matchers ➡️](./05-argument-matchers.md)**

</div>
