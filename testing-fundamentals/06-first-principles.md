# 06. FIRST Principles

> **좋은 단위 테스트의 5가지 특성 — 하나라도 깨지면 테스트가 팀의 짐이 된다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- FIRST의 각 원칙이 깨졌을 때 어떤 증상이 나타나는가?
- 가장 자주 깨지는 원칙은 무엇이고, 어떻게 고치는가?
- 테스트가 "있지만 아무도 안 믿는" 상태가 되는 원인은 FIRST 중 어디에 있는가?

---

## 🔍 왜 FIRST가 필요한가

단위 테스트가 있는데 아무도 신뢰하지 않는 상황이 있습니다.

```
"CI가 빨간데 그냥 머지해도 돼. 원래 가끔 깨져."
"이 테스트 수정하다가 2시간 날렸어. 건드리지 마."
"테스트 통과했는데 배포하면 왜 오류나는 거지?"
```

이 말들은 각각 FIRST 원칙 중 하나가 깨졌을 때 나오는 신호입니다. FIRST는 `Tim Ottinger`가 *Clean Code*에서 제시한 좋은 단위 테스트의 5가지 속성입니다.

---

## 🏛️ FIRST 원칙 전체 구조

```
F — Fast        빠르게 실행된다
I — Isolated    다른 테스트에 의존하지 않는다
R — Repeatable  어디서 실행해도 같은 결과가 나온다
S — Self-validating  사람의 판단 없이 Pass/Fail을 판단한다
T — Timely      테스트 작성 시점이 적절하다
```

---

## F — Fast

### 원칙이 깨지는 이유

```java
// ❌ 느린 테스트의 3가지 원인

// 1. 실제 DB 연결
@Test void 사용자_저장() {
    userRepository.save(new User("alice"));  // DB I/O: 200ms
    assertThat(userRepository.count()).isEqualTo(1);
}

// 2. Thread.sleep()
@Test void 비동기_이벤트_처리() {
    eventBus.publish(new OrderCreated(orderId));
    Thread.sleep(3_000);  // 3초 대기
    assertThat(orderStatusRepository.find(orderId)).isEqualTo(PROCESSING);
}

// 3. 외부 API 호출
@Test void 환율_변환() {
    double rate = exchangeRateApi.getRate("USD", "KRW"); // 네트워크: 500ms
    assertThat(converter.convert(100, rate)).isGreaterThan(0);
}
```

1000개의 테스트가 각각 100ms씩 걸린다면 전체 실행에 100초가 걸립니다. 개발자는 자연스럽게 "테스트 건너뛰기"를 시작합니다.

### 해결 방법

```java
// ✅ Fast하게 만드는 방법

// 1. DB → In-memory Fake
@Test void 사용자_저장() {
    InMemoryUserRepository repo = new InMemoryUserRepository();
    repo.save(new User("alice"));
    assertThat(repo.count()).isEqualTo(1);  // 메모리 접근: < 1ms
}

// 2. Thread.sleep → Awaitility
@Test void 비동기_이벤트_처리() {
    eventBus.publish(new OrderCreated(orderId));
    await().atMost(5, SECONDS)
           .until(() -> orderStatusRepository.find(orderId) == PROCESSING);
    // Awaitility는 조건이 만족될 때까지 폴링 — Thread.sleep보다 빠르고 안전
}

// 3. 외부 API → Stub
@Test void 환율_변환() {
    ExchangeRateApi stubApi = (from, to) -> 1_300.0;  // 람다로 Stub
    double result = new CurrencyConverter(stubApi).convert(100, "USD", "KRW");
    assertThat(result).isEqualTo(130_000);
}
```

**목표:** 단위 테스트 전체가 10초 이내에 완료되어야 합니다. 개발자가 코드 변경 후 즉시 실행할 수 있어야 합니다.

---

## I — Isolated

### 원칙이 깨지는 이유

```java
// ❌ 테스트 A가 테스트 B의 결과에 의존한다

class UserServiceTest {

    static InMemoryUserRepository sharedRepo = new InMemoryUserRepository(); // 공유 상태!

    @Test
    void 사용자_저장() {
        sharedRepo.save(new User("alice"));
        assertThat(sharedRepo.count()).isEqualTo(1);
    }

    @Test
    void 사용자_조회() {
        // 이 테스트는 '사용자_저장'이 먼저 실행됐다고 가정한다
        assertThat(sharedRepo.findByName("alice")).isPresent();
    }
}
```

`사용자_조회`를 단독으로 실행하면 실패합니다. `사용자_저장` 다음에 실행하면 통과합니다. 실행 순서에 의존하는 테스트는 언제든지 터질 수 있는 시한폭탄입니다.

### 해결 방법

```java
// ✅ 각 테스트가 독립적인 상태를 만든다

class UserServiceTest {

    private InMemoryUserRepository repo;  // 인스턴스 필드

    @BeforeEach
    void setUp() {
        repo = new InMemoryUserRepository();  // 매 테스트마다 새로 생성
    }

    @Test
    void 사용자_저장() {
        repo.save(new User("alice"));
        assertThat(repo.count()).isEqualTo(1);
    }

    @Test
    void 존재하는_사용자_조회() {
        repo.save(new User("alice"));  // 자신의 상태를 직접 준비
        assertThat(repo.findByName("alice")).isPresent();
    }
}
```

**기준:** 어떤 테스트든 단독으로 실행해도, 순서를 바꿔도 동일한 결과가 나와야 합니다.

---

## R — Repeatable

### 원칙이 깨지는 이유

```java
// ❌ 실행 환경이나 시간에 따라 결과가 달라진다

// 1. 현재 시간에 의존
@Test void 오늘_만료된_쿠폰_검증() {
    Coupon coupon = new Coupon(LocalDate.now()); // 테스트 실행 날짜가 만료일
    assertThat(coupon.isExpired()).isTrue();     // 오늘만 통과, 내일은 실패
}

// 2. 랜덤 값에 의존
@Test void 추첨_당첨_테스트() {
    LotteryResult result = lottery.draw();      // 50% 확률
    assertThat(result.isWinner()).isTrue();     // 간헐적으로 실패
}

// 3. 실행 환경에 의존
@Test void 파일_경로_테스트() {
    File file = new File("/Users/alice/project/data.csv"); // Mac에서만 존재
    assertThat(file.exists()).isTrue();                    // CI(Linux)에서 실패
}
```

이런 테스트들은 "가끔 실패하는 테스트"(Flaky Test)가 됩니다. 팀이 빨간 CI를 무시하기 시작하면 진짜 버그도 놓치게 됩니다.

### 해결 방법

```java
// ✅ 외부 상태를 테스트 내부로 가져온다

// 1. 시간 → Clock 주입
@Test void 만료된_쿠폰_검증() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-01-15T00:00:00Z"), ZoneOffset.UTC);
    Coupon coupon = new Coupon(LocalDate.of(2024, 1, 10), fixedClock);
    assertThat(coupon.isExpired()).isTrue();  // 고정 시간 기준 — 항상 통과
}

// 2. 랜덤 → 시드 고정 또는 동작 자체 검증
@Test void 추첨_결과는_참여자_중에서_나온다() {
    List<String> participants = List.of("alice", "bob", "charlie");
    LotteryResult result = lottery.draw(participants, new Random(42)); // 시드 고정
    assertThat(participants).contains(result.winner());  // 당첨 여부 아닌 범위 검증
}

// 3. 경로 → 상대 경로 또는 리소스 폴더
@Test void CSV_파일_파싱() {
    // src/test/resources에 test-data.csv 배치
    URL resource = getClass().getClassLoader().getResource("test-data.csv");
    List<Row> rows = csvParser.parse(Path.of(resource.toURI()));
    assertThat(rows).hasSize(3);
}
```

---

## S — Self-validating

### 원칙이 깨지는 이유

```java
// ❌ 사람이 결과를 직접 확인해야 한다

@Test void 주문_저장() {
    Order order = orderService.place(cart, user);
    System.out.println("저장된 주문: " + order);  // 눈으로 확인해야 함
    System.out.println("총 금액: " + order.totalPrice());
}

// ❌ 항상 통과하는 단언
@Test void 할인_적용() {
    int result = calculator.calculate(10_000, "VIP");
    assertThat(result).isInstanceOf(Integer.class);  // int는 항상 Integer
}
```

첫 번째는 테스트가 아니라 디버깅 코드입니다. 두 번째는 항상 통과하므로 아무것도 검증하지 않습니다.

### 해결 방법

```java
// ✅ 명확하고 구체적인 단언

@Test void 주문_저장() {
    Order order = orderService.place(cart, user);
    assertThat(order.id()).isNotNull();
    assertThat(order.totalPrice()).isEqualTo(18_000);
    assertThat(order.status()).isEqualTo(OrderStatus.PENDING);
}

// ✅ 숫자 하나로 결과를 못 박는다
@Test void VIP_할인_적용() {
    assertThat(calculator.calculate(10_000, "VIP")).isEqualTo(9_000);
}
```

**기준:** 테스트 결과는 Green/Red 두 가지 뿐입니다. 로그를 보거나 사람이 판단해야 하는 것은 테스트가 아닙니다.

---

## T — Timely

### 원칙이 깨지는 이유

```
흔한 시나리오:
  1. 기능 구현 완료 (2주)
  2. "이제 테스트 작성하자" → 코드가 이미 복잡해서 테스트하기 어렵다
  3. 테스트를 위해 코드를 다시 리팩터링 (1주 추가)

  또는:

  1. 기능 구현 완료
  2. "다음에 테스트 작성하겠습니다" → 영원히 안 씀
```

구현이 끝난 후 테스트를 추가하면 몇 가지 문제가 생깁니다. 코드가 이미 테스트하기 어렵게 작성되어 있을 수 있고(`static` 메서드, `new`로 직접 생성된 의존성), 테스트가 설계에 영향을 미칠 기회를 이미 놓쳤습니다.

### 해결 방법

```java
// ✅ TDD: 테스트 먼저 작성 (Red → Green → Refactor)

// Step 1: 실패하는 테스트 먼저
@Test void VIP_회원_10퍼센트_할인() {
    // 아직 DiscountCalculator가 없어도 이 테스트를 먼저 작성
    DiscountCalculator calculator = new DiscountCalculator();
    assertThat(calculator.calculate(10_000, "VIP")).isEqualTo(9_000);
}
// → 컴파일 오류: DiscountCalculator 없음 (Red)

// Step 2: 컴파일 통과하는 최소 구현
public class DiscountCalculator {
    public int calculate(int price, String grade) {
        return 9_000; // 일단 하드코딩 (Green)
    }
}

// Step 3: 두 번째 테스트로 하드코딩 제거 강제
@Test void 일반_회원_할인_없음() {
    assertThat(calculator.calculate(10_000, "NORMAL")).isEqualTo(10_000);
}
// → 이제 하드코딩으로는 두 테스트를 통과할 수 없으므로 실제 구현 필요 (Refactor)
```

**TDD가 강제하는 것:** 코드를 작성하기 전에 "이 코드를 어떻게 사용할 것인가"를 먼저 생각하게 됩니다. 자연스럽게 테스트하기 쉬운 설계가 나옵니다.

---

## 💻 실전 적용: 내 테스트의 FIRST 체크리스트

```
[ ] Fast:
    단위 테스트 전체 실행 시간이 10초 이내인가?
    Thread.sleep()이 있는가?
    실제 DB/네트워크 연결이 있는가?

[ ] Isolated:
    static 공유 상태가 있는가?
    테스트를 단독으로 실행해도 통과하는가?
    테스트 순서를 바꿔도 결과가 같은가?

[ ] Repeatable:
    LocalDate.now() / new Date() / Math.random()이 있는가?
    특정 OS 경로나 환경 변수에 의존하는가?

[ ] Self-validating:
    System.out.println()이 테스트 안에 있는가?
    assertThat(result).isNotNull() 외에 아무것도 없는가?

[ ] Timely:
    새 기능에 테스트가 함께 들어오는가?
    테스트 없이 구현만 머지되는 PR이 있는가?
```

---

## 🤔 트레이드오프

### T(Timely)는 항상 TDD를 의미하는가?

아닙니다. Timely의 핵심은 "구현과 테스트가 너무 멀어지지 않아야 한다"는 것입니다. 순수한 TDD가 어려운 상황(탐색적 코딩, 프로토타이핑)에서는 구현 직후 테스트를 추가하는 것도 허용됩니다. 단, 배포 전에는 반드시 테스트가 있어야 합니다.

### I(Isolated)와 설정 코드의 중복

`@BeforeEach`에서 같은 초기화 코드를 반복하는 것이 불편할 수 있습니다. Test Data Builder 패턴이나 팩토리 메서드로 Arrange 중복을 줄이면서 Isolated를 유지할 수 있습니다. 이 내용은 `anatomy-of-good-tests/02-test-data-builders.md`에서 다룹니다.

---

## 📌 핵심 정리

```
FIRST 원칙

F — Fast
  단위 테스트 전체: 10초 이내
  느리게 만드는 것: DB, 네트워크, Thread.sleep

I — Isolated
  각 테스트는 다른 테스트에 의존하지 않음
  공유 상태 → @BeforeEach에서 초기화

R — Repeatable
  시간, 랜덤, 환경에 의존하지 않음
  현재 시간 → Clock 주입, 랜덤 → 시드 고정

S — Self-validating
  실행 결과는 Green/Red만 존재
  System.out.println → assertThat으로 교체

T — Timely
  구현 직전 또는 직후에 테스트 작성
  "나중에 테스트 추가" → 대부분 영원히 안 씀

어떤 원칙이 깨졌는지 신호:
  "CI가 가끔 빨개" → R(Repeatable) 위반
  "테스트 순서 바꾸면 실패해" → I(Isolated) 위반
  "테스트 실행하는 데 5분 걸려" → F(Fast) 위반
  "아무도 안 믿는 초록불" → S(Self-validating) 위반
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트에서 FIRST의 어떤 원칙이 위반됐는가? 각 위반을 수정하라.

```java
static int callCount = 0; // 클래스 레벨 공유 상태

@Test
void 할인_계산_호출_횟수() {
    callCount++;
    int result = calculator.calculate(10_000, "VIP");
    System.out.println("호출 횟수: " + callCount);
    assertThat(result).isInstanceOf(Integer.class);
}

@Test
void 할인_계산_두번째() {
    assertThat(callCount).isEqualTo(1); // 이전 테스트가 먼저 실행됐다고 가정
}
```

**Q2.** 다음 중 F(Fast) 원칙을 지키면서 "실제 DB가 없으면 통합 테스트가 의미 없다"는 주장을 어떻게 조화시키겠는가?

**Q3.** T(Timely) 원칙과 관련해서, TDD를 처음 도입하는 팀에서 가장 흔하게 부딪히는 장벽은 무엇이고, 그 장벽을 낮추는 실용적 방법은 무엇인가?

> 💡 **해설**
>
> **Q1.** 위반 원칙: ① I(Isolated) — `callCount`가 static 공유 상태이고, `두번째` 테스트가 `첫번째` 테스트의 실행 결과에 의존한다. ② S(Self-validating) — `System.out.println`으로 사람이 로그를 확인해야 한다. ③ S 위반 — `isInstanceOf(Integer.class)`는 항상 통과하는 단언이다. 수정: `callCount`를 `@BeforeEach`에서 0으로 초기화, `System.out.println` 제거, `assertThat(result).isEqualTo(9_000)`으로 구체적 단언 작성, `두번째` 테스트는 자체적으로 상태를 준비해야 함.
>
> **Q2.** 테스트를 목적에 따라 분리한다. 비즈니스 로직 검증(단위 테스트) — Fast, In-memory Fake 사용. DB 연결 검증(통합 테스트) — `@DataJpaTest` + Testcontainers, CI에서만 실행, 로컬에서는 선택적 실행. 이 두 레이어는 검증 목적이 다르다. "실제 DB가 있어야 의미 있는 테스트"는 통합 테스트 레이어의 것이고, 단위 테스트가 느려질 이유가 없다.
>
> **Q3.** 가장 흔한 장벽: "아직 설계가 안 됐는데 테스트를 어떻게 먼저 쓰나?" 장벽을 낮추는 방법: 완벽한 설계를 요구하지 않는다. TDD에서 테스트는 설계를 탐색하는 도구이다. 테스트를 먼저 쓸 때 "이 기능을 사용하는 쪽에서 어떤 인터페이스가 편한가?"를 고민하는 것이 테스트의 목적이다. 처음에는 새 기능 하나에만 TDD를 적용하는 것부터 시작하고, 기존 코드는 테스트 후작성으로 안전망을 만든 뒤 점진적으로 TDD 비중을 늘리는 것이 현실적이다.

---

<div align="center">

**[⬅️ 이전: Coverage Myths](./05-coverage-myths.md)**

</div>
