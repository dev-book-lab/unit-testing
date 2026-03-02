# 06. Testing Legacy Code

> **레거시 코드란 테스트 없는 코드다 — 테스트 없이 변경하는 것은 안전망 없이 줄타기를 하는 것과 같다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 테스트 없는 레거시 코드에 테스트를 추가하는 순서는 무엇인가?
- Seam이란 무엇이고 어떻게 활용하는가?
- Characterization Test란 무엇이고 왜 필요한가?

---

## 🔍 레거시 코드의 딜레마

```
테스트 없이 리팩터링하면 위험하다
리팩터링 없이 테스트를 추가하기도 어렵다
→ 닭이 먼저냐 달걀이 먼저냐
```

Michael Feathers가 정의한 레거시 코드: "테스트 없는 코드(code without tests)". 연도가 오래됐거나 작성자가 퇴사했기 때문이 아니라, 테스트가 없어서 변경이 두려운 코드입니다.

해결 순서:

```
① 현재 동작을 Characterization Test로 고정한다
② 최소한의 리팩터링으로 Seam을 만든다
③ Seam을 통해 의존성을 교체하고 단위 테스트를 추가한다
④ 안전하게 리팩터링한다
```

---

## 🏛️ Characterization Test — 현재 동작을 고정

Characterization Test는 코드가 **무엇을 해야 하는지**가 아니라 **현재 실제로 무엇을 하는지**를 기록합니다. 변경 전 동작의 스냅샷입니다.

```java
// 레거시 코드
public class LegacyOrderProcessor {
    public String processOrder(int userId, int[] itemIds, String couponCode) {
        // 300줄의 복잡한 레거시 코드
        // 정확히 무엇을 하는지 파악하기 어렵다
        ...
    }
}
```

```java
// Characterization Test: 현재 동작을 그대로 기록
@Test
void characterization_일반_주문_처리() {
    LegacyOrderProcessor processor = new LegacyOrderProcessor();

    // 현재 실제로 어떤 값이 반환되는지 실행해서 확인
    String result = processor.processOrder(1, new int[]{10, 20}, null);

    // 지금 반환되는 값을 그대로 단언 — 올바른지 여부는 나중에 판단
    assertThat(result).isEqualTo("ORDER-1-2-ITEMS-NO-COUPON");
}

@Test
void characterization_쿠폰_적용_주문() {
    LegacyOrderProcessor processor = new LegacyOrderProcessor();
    String result = processor.processOrder(1, new int[]{10}, "SAVE10");

    assertThat(result).isEqualTo("ORDER-1-1-ITEM-COUPON-APPLIED");
}

@Test
void characterization_빈_아이템_처리() {
    LegacyOrderProcessor processor = new LegacyOrderProcessor();

    // 예외가 발생하는지, 어떤 예외인지 먼저 확인
    assertThatThrownBy(() -> processor.processOrder(1, new int[]{}, null))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("items");
}
```

이 테스트들은 지금 당장 "올바른" 동작을 검증하는 게 아닙니다. 변경 전 동작이 변경 후에도 동일하게 유지되는지 확인하는 안전망입니다.

---

## 🏛️ Seam — 최소 침습으로 의존성을 교체하는 지점

Seam은 코드를 크게 바꾸지 않고도 동작을 바꿀 수 있는 지점입니다. 테스트를 위해 의존성을 교체하는 진입점입니다.

### Seam 유형 1: 서브클래싱 (Object Seam)

```java
// 레거시 코드 — 직접 생성
public class LegacyOrderProcessor {
    public void process(Long orderId) {
        // 내부에서 직접 생성 → 교체 불가
        PaymentClient payment = new PaymentClient();
        payment.charge(orderId, amount);
    }
}
```

```java
// ✅ protected 메서드로 Seam 만들기 — 최소한의 변경
public class LegacyOrderProcessor {
    public void process(Long orderId) {
        PaymentClient payment = createPaymentClient(); // Seam
        payment.charge(orderId, amount);
    }

    // 서브클래싱으로 재정의 가능
    protected PaymentClient createPaymentClient() {
        return new PaymentClient();
    }
}

// 테스트용 서브클래스
class TestableOrderProcessor extends LegacyOrderProcessor {
    private final PaymentClient fakeClient;

    TestableOrderProcessor(PaymentClient fakeClient) {
        this.fakeClient = fakeClient;
    }

    @Override
    protected PaymentClient createPaymentClient() {
        return fakeClient; // 테스트용 교체
    }
}

// 테스트
@Test
void 결제_처리() {
    PaymentClient fakeClient = mock(PaymentClient.class);
    TestableOrderProcessor processor = new TestableOrderProcessor(fakeClient);

    processor.process(1L);

    verify(fakeClient).charge(eq(1L), anyInt());
}
```

### Seam 유형 2: 파라미터 추가 (Parameter Seam)

```java
// ❌ 현재: 내부에서 직접 시간 조회
public class PointExpiryChecker {
    public boolean isExpired(Point point) {
        return point.expiresAt().isBefore(LocalDateTime.now()); // 테스트 불가
    }
}

// ✅ 파라미터로 추출 — 시그니처 변경만으로 Seam 생성
public class PointExpiryChecker {
    public boolean isExpired(Point point) {
        return isExpired(point, LocalDateTime.now()); // 기존 API 유지
    }

    // 새 오버로드 — 테스트에서 사용
    public boolean isExpired(Point point, LocalDateTime now) {
        return point.expiresAt().isBefore(now);
    }
}

@Test
void 만료된_포인트_확인() {
    PointExpiryChecker checker = new PointExpiryChecker();
    Point expiredPoint = aPoint().withExpiresAt(
        LocalDateTime.of(2024, 1, 1, 0, 0)).build();

    // now를 직접 전달 → 시간 고정
    boolean expired = checker.isExpired(expiredPoint,
        LocalDateTime.of(2024, 6, 1, 0, 0));

    assertThat(expired).isTrue();
}
```

### Seam 유형 3: 인터페이스 추출 + 생성자 추가 (Interface Seam)

```java
// ❌ 현재: 구체 클래스에 직접 의존
public class ReportService {
    private final SlackNotifier notifier = new SlackNotifier("TOKEN");

    public void generateReport() {
        // ...
        notifier.send("보고서 생성 완료");
    }
}

// ✅ 인터페이스 추출 + 기존 생성자 유지
public interface Notifier {
    void send(String message);
}

public class SlackNotifier implements Notifier { ... }

public class ReportService {
    private final Notifier notifier;

    // 기존 코드가 깨지지 않도록 기본 생성자 유지
    public ReportService() {
        this.notifier = new SlackNotifier("TOKEN");
    }

    // 테스트를 위한 새 생성자 (점진적 개선)
    public ReportService(Notifier notifier) {
        this.notifier = notifier;
    }
}
```

---

## 💻 실전 적용: 레거시 코드 테스트 추가 순서

```
Step 1: 변경 전 Characterization Test 작성
  현재 동작을 실행하고 결과를 기록
  예외 케이스 포함
  "이게 올바른가?"는 나중에 판단

Step 2: Seam 식별 및 생성
  가장 적은 변경으로 의존성을 교체할 수 있는 지점 찾기
  서브클래싱 → 파라미터 추출 → 인터페이스 추출 순으로 시도

Step 3: 단위 테스트 추가
  Seam을 통해 의존성을 교체한 상태로 비즈니스 로직 검증
  Characterization Test는 유지

Step 4: 리팩터링
  단위 테스트가 동작을 보호하는 상태에서 구조 개선
  Characterization Test로 회귀 확인

Step 5: 레거시 테스트 정리 (선택)
  새 단위 테스트가 충분히 커버하면 Characterization Test 제거 또는 유지
```

---

## 🏛️ 레거시 코드에서 흔한 패턴과 해결

### God Object — 모든 것을 아는 클래스

```java
// ❌ 2000줄짜리 OrderManager — 모든 기능이 여기에
public class OrderManager {
    // 주문 생성, 취소, 결제, 배송, 환불, 이메일, SMS, 통계...
}

// ✅ 기능 단위로 점진적 추출
// Step 1: 가장 독립적인 부분부터 추출
public class OrderCancellationService {
    public void cancel(Long orderId) { ... }
}

// Step 2: OrderManager에서 위임
public class OrderManager {
    private final OrderCancellationService cancellationService;

    public void cancelOrder(Long orderId) {
        cancellationService.cancel(orderId); // 위임
    }
}
// 기존 인터페이스 유지하면서 내부를 점진적으로 분리
```

### 전역 상태 — static 변수

```java
// ❌
public class Config {
    public static DatabaseConfig DB_CONFIG = new DatabaseConfig();
    public static PaymentConfig PAYMENT_CONFIG = new PaymentConfig();
}

// ✅ 싱글톤 → 주입 가능한 설정으로
public class AppConfig {
    private final DatabaseConfig dbConfig;
    private final PaymentConfig paymentConfig;
    // 생성자 주입 → 테스트에서 교체 가능
}
```

---

## 🤔 트레이드오프

### "Characterization Test가 버그를 기록하는 것 아닌가?"

맞습니다. 현재 잘못된 동작도 Characterization Test로 고정됩니다. 그것이 목적입니다. 지금 당장은 변경하려는 부분에 집중하고, 버그가 확실한 케이스는 주석으로 표시해서 나중에 수정합니다.

```java
@Test
void characterization_엣지케이스() {
    // TODO: 이 동작은 버그로 추정됨 — 이슈 #789
    // 현재: 빈 배열에서 NullPointerException 발생
    // 기대: IllegalArgumentException이어야 함
    assertThatThrownBy(() -> processor.process(1, null, null))
        .isInstanceOf(NullPointerException.class); // 현재 동작 고정
}
```

### "너무 느리지 않은가? 그냥 처음부터 다시 짜는 게 낫지 않은가?"

"처음부터 다시 짜는 것(Big Rewrite)"은 레거시 소프트웨어에서 가장 위험한 접근입니다. 기존 코드에 암묵적으로 축적된 비즈니스 지식과 엣지케이스를 새 코드에서 재현하는 것은 매우 어렵습니다. 점진적 접근이 안전합니다.

---

## 📌 핵심 정리

```
레거시 코드 테스트 추가 순서:
  ① Characterization Test로 현재 동작 고정
  ② Seam 식별 → 최소 변경으로 의존성 교체 지점 생성
  ③ Seam을 통해 단위 테스트 추가
  ④ 단위 테스트 보호 아래 리팩터링

Characterization Test:
  "올바른" 동작이 아닌 현재 동작을 기록
  버그도 일단 고정 (주석으로 표시)
  변경 안전망 역할

Seam 유형:
  Object Seam: protected factory 메서드 → 서브클래싱
  Parameter Seam: now, config를 파라미터로 추출
  Interface Seam: 구체 클래스 → 인터페이스 추출 + 생성자 추가

점진적 접근:
  God Object → 기능별 클래스 추출 + 위임
  static 전역 상태 → 주입 가능한 설정으로
  Big Rewrite 피하기 → 암묵적 비즈니스 지식 소실 위험
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 레거시 코드에 Characterization Test를 작성하고, 어떤 Seam을 만들어서 단위 테스트를 추가할 수 있는지 설명하라.

```java
public class LegacyBillingService {
    public int calculateMonthlyBill(int userId) {
        Connection conn = DriverManager.getConnection(
            "jdbc:mysql://prod:3306/billing", "root", "password");
        ResultSet rs = conn.createStatement()
            .executeQuery("SELECT SUM(amount) FROM usage WHERE user_id = " + userId
                + " AND month = " + LocalDate.now().getMonthValue());
        int usage = rs.getInt(1);

        // 구간별 요금 계산
        if (usage <= 100) return usage * 10;
        if (usage <= 500) return 1_000 + (usage - 100) * 8;
        return 4_200 + (usage - 500) * 6;
    }
}
```

**Q2.** Characterization Test와 일반 단위 테스트는 어떤 점에서 다른가? 레거시 코드 작업을 마친 후 Characterization Test를 유지해야 하는가, 제거해야 하는가?

**Q3.** 아래 코드에서 서브클래싱 Seam과 파라미터 Seam을 각각 적용하는 방법을 보여라. 어느 쪽이 더 나은가?

```java
public class ScheduledReportSender {
    public void send() {
        if (LocalTime.now().getHour() != 9) {
            return; // 오전 9시에만 실행
        }
        Report report = generateReport();
        new EmailClient().send("team@company.com", report.toHtml());
    }
}
```

> 💡 **해설**

**Q1.**

Characterization Test:

```java
// 실제 DB 연결이 필요하므로 통합 테스트로 작성하거나
// 또는 가능한 케이스는 Seam 만든 후에 단위 테스트로 작성

// DB 없이 Characterization이 가능한 부분: 구간 계산 로직
// 먼저 계산 로직만 분리 가능성 파악
@Test
void characterization_100단위_이하_요금() {
    // 현재 코드를 실행해서 결과 기록
    // (DB 의존성 때문에 통합 환경에서 실행)
    assertThat(billing.calculateMonthlyBill(testUserId100)).isEqualTo(1_000);
}
```

Seam 만들기:

```java
// 1단계: 계산 로직을 protected 메서드로 추출 (Object Seam)
public class LegacyBillingService {
    public int calculateMonthlyBill(int userId) {
        int usage = fetchUsage(userId); // DB 조회 분리
        return calculateFee(usage);     // 계산 로직 분리
    }

    protected int fetchUsage(int userId) {
        // DB 조회 로직
    }

    protected int calculateFee(int usage) {
        if (usage <= 100) return usage * 10;
        if (usage <= 500) return 1_000 + (usage - 100) * 8;
        return 4_200 + (usage - 500) * 6;
    }
}

// 2단계: 단위 테스트
class TestableBillingService extends LegacyBillingService {
    private final int fakeUsage;
    TestableBillingService(int fakeUsage) { this.fakeUsage = fakeUsage; }

    @Override
    protected int fetchUsage(int userId) { return fakeUsage; }
}

@Test
void 100_이하_구간_요금() {
    assertThat(new TestableBillingService(100).calculateMonthlyBill(1)).isEqualTo(1_000);
}

@Test
void 100초과_500이하_구간_요금() {
    assertThat(new TestableBillingService(300).calculateMonthlyBill(1)).isEqualTo(2_600);
}

@Test
void 500초과_구간_요금() {
    assertThat(new TestableBillingService(600).calculateMonthlyBill(1)).isEqualTo(4_800);
}
```

**Q2.**

차이점:

Characterization Test는 "현재 동작"을 기록한다. 동작이 올바른지 여부를 묻지 않는다. 버그를 포함할 수 있다. 목적은 변경 안전망.

일반 단위 테스트는 "원하는 동작"을 명세한다. 요구사항에서 도출된다. 동작이 올바른지 검증한다.

유지 vs 제거: 새로 작성한 단위 테스트가 Characterization Test가 커버하는 모든 케이스를 포함한다면 제거해도 된다. 새 테스트로 대체되지 않은 케이스가 있다면 유지한다. 단, 버그를 기록한 Characterization Test는 버그 수정 후 올바른 동작을 검증하는 테스트로 교체한다.

**Q3.**

서브클래싱 Seam:

```java
public class ScheduledReportSender {
    public void send() {
        if (currentHour() != 9) return;
        Report report = generateReport();
        createEmailClient().send("team@company.com", report.toHtml());
    }

    protected int currentHour() {
        return LocalTime.now().getHour();
    }

    protected EmailClient createEmailClient() {
        return new EmailClient();
    }
}

class TestableReportSender extends ScheduledReportSender {
    private final int hour;
    private final EmailClient fakeClient;

    @Override protected int currentHour() { return hour; }
    @Override protected EmailClient createEmailClient() { return fakeClient; }
}
```

파라미터 Seam:

```java
public class ScheduledReportSender {
    public void send() {
        send(LocalTime.now(), new EmailClient());
    }

    void send(LocalTime now, EmailClient client) { // package-private
        if (now.getHour() != 9) return;
        Report report = generateReport();
        client.send("team@company.com", report.toHtml());
    }
}

@Test
void 오전9시에_발송() {
    EmailClient fakeClient = mock(EmailClient.class);
    new ScheduledReportSender()
        .send(LocalTime.of(9, 0), fakeClient);
    verify(fakeClient).send(any(), any());
}
```

어느 쪽이 더 나은가: 파라미터 Seam이 더 낫다. 서브클래싱은 테스트마다 서브클래스를 만들어야 하고 상속 계층이 늘어난다. 파라미터 Seam은 단순하고 이후에 생성자 주입으로 자연스럽게 발전시킬 수 있다. 최종적으로는 `Clock`과 `EmailClient`를 생성자로 주입받는 형태로 리팩터링하는 것이 목표다.

---

<div align="center">

**[⬅️ 이전: Test-Driven Design](./05-test-driven-design.md)**

</div>
