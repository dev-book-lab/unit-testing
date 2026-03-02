# 02. Stub vs Mock

> **상태 검증 vs 행동 검증 — 무엇을 검증하느냐가 테스트의 리팩터링 내성을 결정한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 상태 검증과 행동 검증은 무엇이 다른가?
- Mock으로 행동을 검증하는 테스트가 왜 리팩터링에 취약한가?
- "Mock을 쓰면 안 된다"는 말이 틀린 이유는 무엇인가?

---

## 🔍 같은 기능, 두 가지 검증 방식

```java
public class NotificationService {
    private final UserRepository userRepository;
    private final EmailSender emailSender;

    public void sendWelcome(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        emailSender.send(user.email(), "가입을 환영합니다!");
    }
}
```

이 메서드를 검증하는 두 가지 접근이 있습니다.

---

## 🏛️ 상태 검증 (State Verification)

```java
@Test
void 가입_환영_이메일이_발송된다_상태검증() {
    // Arrange
    User user = new User(1L, "alice@example.com");
    userRepository.save(user);

    FakeEmailSender fakeEmailSender = new FakeEmailSender(); // 발송 내역을 기억하는 Fake

    NotificationService service =
        new NotificationService(userRepository, fakeEmailSender);

    // Act
    service.sendWelcome(1L);

    // Assert — 결과 상태를 확인
    assertThat(fakeEmailSender.lastSentTo()).isEqualTo("alice@example.com");
    assertThat(fakeEmailSender.lastBody()).contains("환영");
}

// FakeEmailSender: 실제로 발송하지 않고 내역만 기억
class FakeEmailSender implements EmailSender {
    private String lastTo;
    private String lastBody;

    @Override
    public void send(String to, String body) {
        this.lastTo = to;
        this.lastBody = body;
    }

    public String lastSentTo() { return lastTo; }
    public String lastBody() { return lastBody; }
}
```

**관심사:** "이메일이 올바른 주소와 내용으로 발송됐는가"를 **결과 상태**로 확인합니다.

---

## 🏛️ 행동 검증 (Behavior Verification)

```java
@Test
void 가입_환영_이메일이_발송된다_행동검증() {
    // Arrange
    User user = new User(1L, "alice@example.com");
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));
    EmailSender mockEmailSender = mock(EmailSender.class);

    NotificationService service =
        new NotificationService(userRepository, mockEmailSender);

    // Act
    service.sendWelcome(1L);

    // Assert — 협력 객체가 어떻게 호출됐는가를 확인
    verify(mockEmailSender).send("alice@example.com", "가입을 환영합니다!");
}
```

**관심사:** "`emailSender.send()`가 올바른 인자로 호출됐는가"를 **호출 행동**으로 확인합니다.

---

## 😱 행동 검증이 만드는 리팩터링 취약성

두 테스트는 현재 동일한 것을 검증하는 것처럼 보입니다. 하지만 내부 구현이 바뀌면 다르게 반응합니다.

```java
// 리팩터링: 이메일 발송을 EmailComposer를 통해 간접 호출하도록 변경
public class NotificationService {
    private final UserRepository userRepository;
    private final EmailComposer emailComposer;  // 새로운 협력자
    private final EmailSender emailSender;

    public void sendWelcome(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();
        Email email = emailComposer.composeWelcome(user);  // 메일 조합을 위임
        emailSender.send(email);
    }
}
```

```
상태 검증 테스트 (Fake 사용):
  → 여전히 "alice@example.com"으로 환영 메일이 발송되면 통과
  → 리팩터링 후에도 통과 ✅

행동 검증 테스트 (Mock 사용):
  → verify(mockEmailSender).send("alice@example.com", "가입을 환영합니다!")
  → EmailSender의 send() 시그니처가 바뀌거나 (send(Email) vs send(String, String))
     EmailComposer를 거치는 흐름이 달라지면 실패 ❌
  → 기능은 동일한데 테스트가 깨진다
```

이것이 행동 검증이 만드는 **리팩터링 내성 문제**입니다. 내부 구조를 바꾸면 테스트가 깨지지만, 실제 동작은 올바릅니다.

---

## ✨ 언제 무엇을 선택하는가

### 상태 검증을 선택하는 경우

```java
// 1. 협력 객체가 상태를 남기는 경우 (Repository, Cache, Queue)
@Test
void 주문이_저장된다() {
    Order order = orderService.place(cart, user);
    // Fake Repository의 상태 확인
    assertThat(repository.findById(order.id())).isPresent();
}

// 2. 반환값이 있어서 결과를 직접 검증할 수 있는 경우
@Test
void 할인_금액이_올바르게_계산된다() {
    int result = discountService.calculate(10_000, "VIP");
    assertThat(result).isEqualTo(9_000);
    // emailSender.send()를 verify할 이유가 없음
}
```

### 행동 검증(Mock + verify)을 선택하는 경우

```java
// 1. 부수효과만 있고 반환값이 없는 경우 (알림, 로그, 이벤트)
@Test
void 주문_생성_시_OrderPlaced_이벤트가_발행된다() {
    orderService.place(cart, user);
    // 이벤트 발행은 반환값이 없고 상태를 남기지 않음 → verify
    verify(eventPublisher).publish(argThat(e -> e instanceof OrderPlaced));
}

// 2. 외부 시스템에 정확한 메시지를 보내야 하는 경우
@Test
void 결제_요청이_올바른_금액으로_전송된다() {
    paymentService.pay(order);
    verify(paymentGateway).charge(argThat(req ->
        req.amount() == 18_000 && req.currency().equals("KRW")));
}
```

**핵심 기준:**

```
반환값이 있거나 상태가 남는가?
  → YES: 상태 검증 (Stub/Fake + assertThat)

부수효과만 있고 흔적이 없는가?
  → YES: 행동 검증 (Mock + verify)
```

---

## 💻 실전 적용: 두 방식을 함께 쓰는 경우

하나의 메서드가 상태를 바꾸고 부수효과도 발생시킨다면 두 방식을 조합합니다.

```java
@Test
void 주문_성공_시_저장되고_이벤트가_발행된다() {
    // Fake: 저장 상태 검증
    InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
    // Mock: 이벤트 발행 행동 검증
    EventPublisher mockPublisher = mock(EventPublisher.class);

    OrderService service = new OrderService(fakeRepo, mockPublisher, ...);

    Order result = service.place(cart, user);

    // 상태 검증
    assertThat(fakeRepo.findById(result.id())).isPresent();

    // 행동 검증
    verify(mockPublisher).publish(argThat(e -> e instanceof OrderPlaced));
}
```

---

## 🤔 트레이드오프

### "행동 검증이 무조건 나쁜 것인가?"

아닙니다. **부수효과가 있는 외부 호출**은 행동 검증이 유일한 선택입니다. 이메일을 실제 발송하거나, Slack에 실제 메시지를 보내지 않으면서 "발송됐는가"를 검증하려면 `verify()`가 필요합니다.

문제는 상태로 검증할 수 있는데 행동으로 검증할 때입니다.

### "Stub + Fake 조합이 항상 가능한가?"

Fake를 만들 수 없는 경우가 있습니다. 인터페이스가 없거나 final class인 경우, 또는 협력 객체가 너무 복잡해서 Fake 구현 비용이 과도할 때입니다. 이럴 때는 Mock + verify로 현실적인 타협을 합니다.

---

## 📌 핵심 정리

```
상태 검증 (State Verification):
  Stub 또는 Fake 사용
  결과가 올바른지 assertThat으로 확인
  리팩터링 내성 높음 — 내부 구현이 바뀌어도 결과가 같으면 통과

행동 검증 (Behavior Verification):
  Mock 사용
  협력 객체가 어떻게 호출됐는지 verify로 확인
  리팩터링 내성 낮음 — 내부 구현이 바뀌면 깨질 수 있음

선택 기준:
  반환값이 있거나 상태가 남는다 → 상태 검증
  부수효과만 있고 흔적이 없다  → 행동 검증

Mock의 올바른 사용처:
  이메일/알림/이벤트 발행 검증
  외부 API에 전달되는 정확한 메시지 검증

Mock의 잘못된 사용처:
  Repository 저장 여부 (→ Fake 사용)
  반환값이 있는 계산 결과 (→ assertThat 사용)
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 두 테스트 중 어느 것이 리팩터링에 더 강한가? `DiscountService`가 내부적으로 `RuleEngine`을 통해 할인율을 계산하도록 변경됐을 때 각 테스트가 어떻게 반응하는지 분석하라.

```java
// 테스트 A
@Test
void 할인_계산_A() {
    when(discountPolicy.calculate(vipUser)).thenReturn(10);
    int result = discountService.apply(20_000, vipUser);
    verify(discountPolicy).calculate(vipUser);
    assertThat(result).isEqualTo(18_000);
}

// 테스트 B
@Test
void 할인_계산_B() {
    int result = discountService.apply(20_000, vipUser);
    assertThat(result).isEqualTo(18_000);
}
```

**Q2.** 다음 기능을 테스트할 때 상태 검증과 행동 검증 중 무엇을 선택하겠는가? 이유를 설명하라.

```java
// 기능: 사용자가 비밀번호를 틀리면 보안 로그를 남긴다
public void login(String email, String password) {
    User user = userRepository.findByEmail(email);
    if (!passwordEncoder.matches(password, user.passwordHash())) {
        securityLogger.warn("FAILED_LOGIN", email);  // 반환값 없음
        throw new AuthenticationException();
    }
}
```

**Q3.** 팀에서 "우리 코드베이스는 Mock을 쓰지 않는다"는 규칙을 만들었다. 이 규칙이 모든 상황에서 올바른가? 규칙이 무너져야 하는 예외 상황을 설명하라.

> 💡 **해설**
>
> **Q1.** 테스트 B가 리팩터링에 더 강하다. 테스트 A는 두 가지 이유로 취약하다: ① `when(discountPolicy.calculate(vipUser)).thenReturn(10)` — `RuleEngine`으로 교체 후 `discountPolicy`가 더 이상 직접 호출되지 않으면 Mock 설정이 의미 없어지고 `when()`이 반환하는 값도 쓰이지 않는다. ② `verify(discountPolicy).calculate(vipUser)` — `discountPolicy`를 직접 호출하지 않으므로 이 verify가 실패한다. 기능(18,000원 할인 적용)은 동일한데 테스트가 깨진다. 테스트 B는 "결과가 18,000원인가"만 검증하므로 내부 구현이 바뀌어도 결과가 같으면 통과한다.
>
> **Q2.** `securityLogger.warn()`은 반환값이 없고 외부 시스템(로그 저장소)에 대한 부수효과다. 상태 검증으로는 "로그가 실제로 저장됐는가"를 확인하려면 실제 로그 저장소가 필요하다. 행동 검증(Mock + verify)이 적합하다: `verify(securityLogger).warn("FAILED_LOGIN", "alice@example.com")`. 단, `AuthenticationException`이 발생하는지는 `assertThatThrownBy`로 상태 검증이 가능하다. 즉 이 테스트는 두 가지를 함께 검증한다.
>
> **Q3.** 규칙이 무너져야 하는 예외 상황: ① 이메일/SMS/Slack 같은 외부 발송 — Fake를 만들면 "발송됐는가"를 확인할 수 있지만, "발송 서버에 올바른 페이로드가 전달됐는가"는 Mock + verify가 더 직접적이다. ② 외부 결제 API에 정확한 금액/통화가 전달됐는지 — Fake가 검증하기 어려운 페이로드 구조. ③ 아직 구현되지 않은 협력자 — TDD에서 인터페이스만 있고 구현이 없는 경우. 규칙의 올바른 표현: "Mock을 기본으로 쓰지 않는다. 부수효과 검증이 필요하거나 Fake 구현 비용이 과도할 때 Mock을 쓴다."

---

<div align="center">

**[⬅️ 이전: Test Doubles Taxonomy](./01-test-doubles-taxonomy.md)** | **[다음: Mockito Best Practices ➡️](./03-mockito-best-practices.md)**

</div>
