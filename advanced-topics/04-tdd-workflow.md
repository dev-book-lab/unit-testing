# 04. TDD Workflow

> **Red → Green → Refactor는 세 단계가 아니라 하나의 리듬이다 — 각 단계의 목적을 이해하면 사이클이 10분에서 2분으로 줄어든다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Red → Green → Refactor 각 단계에서 무엇을 해야 하고 무엇을 하면 안 되는가?
- TDD 사이클을 짧게 유지하는 방법은 무엇인가?
- 테스트를 먼저 쓰기 어려운 상황에서 어떻게 접근하는가?

---

## 🏛️ Red → Green → Refactor

```
Red:    실패하는 테스트를 작성한다
        목적: 다음 구현해야 할 동작을 하나 선택하고 명세로 만든다

Green:  테스트를 통과시킨다
        목적: 오직 테스트를 통과시키는 것. 설계나 중복은 신경 쓰지 않는다

Refactor: 코드를 개선한다
        목적: 동작을 바꾸지 않고 구조를 개선한다. 테스트가 보호망이 된다
```

이 세 단계는 각자 다른 목적을 가집니다. 단계를 섞으면 TDD가 어려워집니다.

---

## 🔍 각 단계에서 흔히 저지르는 실수

### Red 단계 실수: 너무 많은 것을 테스트하려 한다

```java
// ❌ Red 단계에서 너무 큰 테스트
@Test
void 주문_생성_전체_플로우() {
    // 재고 확인, 가격 계산, 저장, 이메일 발송까지 한 번에
    Order order = orderService.place(
        aCommand().withUser(vipUser).withItems(3).withCoupon("SAVE10").build()
    );
    assertThat(order.id()).isNotNull();
    assertThat(order.totalPrice()).isEqualTo(16_200); // 복잡한 계산
    assertThat(order.status()).isEqualTo(PENDING);
    verify(emailSender).send(any(), any());
}
// 이 테스트를 통과시키려면 구현해야 할 것이 너무 많다
// Red에서 Green으로 가는 데 1시간이 걸릴 수 있다
```

```java
// ✅ Red 단계: 가장 작은 의미 있는 동작 하나
@Test
void 주문을_생성하면_PENDING_상태다() {
    Order order = orderService.place(aCommand().build());
    assertThat(order.status()).isEqualTo(PENDING);
}
// 이것을 통과시키는 데 5분이면 충분하다
```

### Green 단계 실수: 설계를 개선하려 한다

```java
// ❌ Green 단계에서 설계까지 고치려 함
// → 테스트도 통과시키면서 리팩터링도 하려다 둘 다 실패
public Order place(PlaceOrderCommand command) {
    // 테스트를 통과시키면서
    Order order = new Order(command.userId(), PENDING);
    // 동시에 설계도 개선하려고 DiscountPolicy, InventoryChecker 도입
    // → 컴파일 오류, 의존성 문제 발생
    // → 30분째 Green을 못 만듦
}
```

```java
// ✅ Green 단계: 일단 통과만. 가장 단순한 구현
public Order place(PlaceOrderCommand command) {
    return new Order(command.userId(), PENDING); // 충분히 단순하다
}
// 테스트 통과 → 즉시 Refactor 단계로 이동
```

### Refactor 단계 실수: 새 기능을 추가한다

```java
// ❌ Refactor 단계에서 새 기능 추가
// → 테스트가 보호하지 않는 영역을 건드림
public Order place(PlaceOrderCommand command) {
    // 리팩터링 중에
    Order order = new Order(command.userId(), PENDING);
    // 새 기능: 할인 적용 (테스트 없음)
    int discount = discountPolicy.calculate(command.user()); // ❌
    order.applyDiscount(discount);
    return order;
}
// Refactor는 동작을 바꾸지 않는다
// 새 기능은 다음 Red 단계에서 테스트와 함께 추가한다
```

---

## ✨ 실전 TDD 사이클: 포인트 적립 서비스

### 사이클 1: 가장 기본적인 동작

```java
// Red: 가장 단순한 케이스
@Test
void 구매금액에_따라_포인트를_적립한다() {
    PointService service = new PointService();
    int points = service.calculate(10_000);
    assertThat(points).isEqualTo(100); // 1% 적립
}
```

```java
// Green: 가장 단순한 구현
public class PointService {
    public int calculate(int amount) {
        return amount / 100;
    }
}
```

Refactor: 단순해서 개선할 게 없다. 다음 사이클로.

---

### 사이클 2: 등급별 차등 적립

```java
// Red: 새 동작 추가
@Test
void VIP_회원은_2퍼센트_적립한다() {
    PointService service = new PointService();
    int points = service.calculate(10_000, Grade.VIP);
    assertThat(points).isEqualTo(200);
}
// 컴파일 오류 — calculate(int, Grade) 메서드 없음 → 정상적인 Red
```

```java
// Green: 통과만 목표
public int calculate(int amount, Grade grade) {
    if (grade == Grade.VIP) return amount / 50;
    return amount / 100;
}
```

```java
// Refactor: 기존 메서드와 통합
public int calculate(int amount, Grade grade) {
    int rate = (grade == Grade.VIP) ? 50 : 100;
    return amount / rate;
}

// 기존 calculate(int)도 동작해야 하므로
public int calculate(int amount) {
    return calculate(amount, Grade.NORMAL);
}
```

테스트 재실행 → 모두 통과 확인.

---

### 사이클 3: 최소 적립 금액 조건

```java
// Red
@Test
void 최소_금액_미만은_포인트_없음() {
    int points = service.calculate(999, Grade.NORMAL);
    assertThat(points).isEqualTo(0);
}
```

```java
// Green
public int calculate(int amount, Grade grade) {
    if (amount < 1_000) return 0; // 추가
    int rate = (grade == Grade.VIP) ? 50 : 100;
    return amount / rate;
}
```

Refactor: 상수로 추출.

```java
private static final int MINIMUM_AMOUNT = 1_000;

public int calculate(int amount, Grade grade) {
    if (amount < MINIMUM_AMOUNT) return 0;
    int rate = (grade == Grade.VIP) ? 50 : 100;
    return amount / rate;
}
```

---

## 💻 사이클을 짧게 유지하는 기준

```
좋은 Red 테스트의 기준:
  이 테스트 하나를 통과시키는 데 10분 이내
  구현해야 할 새 개념이 1개 이하
  테스트 이름이 동작을 명확히 설명함

사이클이 길어지는 신호:
  Green을 만들기 위해 여러 클래스를 동시에 수정
  테스트를 작성하는 데 30분 이상 소요
  Refactor 후 어떤 테스트가 보호하는지 불명확

짧게 유지하는 방법:
  요구사항을 가능한 작은 동작으로 쪼갠다
  "이것보다 더 작은 첫 단계가 있는가?" 자문
  Fake/Stub으로 먼저 인터페이스를 만들고 구현은 나중에
```

---

## 🏛️ 테스트 목록 작성 — TDD 시작 전 준비

TDD를 시작하기 전에 구현해야 할 동작을 목록으로 만듭니다. 모든 것을 한 번에 구현하지 않고, 한 번에 하나씩 선택합니다.

```
포인트 서비스 구현 테스트 목록:
  [ ] 구매금액에 따라 1% 포인트 적립
  [ ] VIP 회원은 2% 적립
  [ ] 최소 금액(1,000원) 미만은 포인트 없음
  [ ] 적립 포인트는 정수 (소수점 내림)
  [ ] 포인트 유효기간은 1년
  [ ] 만료된 포인트는 차감 불가
```

목록을 모두 작성하고 나서 가장 단순한 것부터 시작합니다. 구현 중 새로운 케이스가 떠오르면 목록에 추가하고, 지금 사이클이 끝난 후에 처리합니다.

---

## 🏛️ 언제 테스트를 먼저 쓰고 언제 나중에 쓰는가

```
테스트를 먼저 쓰기 좋은 상황:
  비즈니스 로직이 명확한 경우 (계산, 검증, 상태 전이)
  인터페이스를 설계하면서 사용성을 검토하고 싶을 때
  버그를 재현하고 수정할 때 (버그 → Red 테스트 → Fix → Green)

테스트를 나중에 쓰기 좋은 상황:
  탐색적 코딩 (API가 어떻게 동작하는지 실험)
  UI, 프레임워크 설정, 외부 라이브러리 연동
  "어떻게 구현할지" 아직 불명확한 경우
  → 먼저 동작하는 코드를 만들고 테스트로 고정
```

---

## 🤔 트레이드오프

### "TDD로 짜면 처음엔 느린데 결국 빨라지는가?"

초기에는 테스트를 먼저 작성하는 습관이 없으면 느립니다. 익숙해지면 두 가지 이유로 빨라집니다.

① 디버깅 시간이 줄어듭니다. 각 사이클이 짧으므로 실패하면 방금 추가한 코드가 원인임을 바로 알 수 있습니다.

② 리팩터링이 두렵지 않습니다. 테스트가 동작을 보호하므로 구조를 자유롭게 개선할 수 있습니다.

### "테스트를 먼저 쓰면 좋은 설계가 자동으로 나오는가?"

TDD는 좋은 설계를 보장하지 않습니다. TDD는 설계를 강제하는 피드백 루프를 만듭니다. "테스트하기 어렵다"는 신호를 설계 문제로 인식하고 개선하면 설계가 나아집니다. 05 문서에서 자세히 다룹니다.

---

## 📌 핵심 정리

```
각 단계의 목적:
  Red:      다음 동작을 명세로 만들기 (실패 확인 필수)
  Green:    오직 테스트 통과 (가장 단순한 구현)
  Refactor: 동작 유지, 구조 개선 (새 기능 추가 금지)

단계를 섞으면 안 되는 이유:
  Green 중 설계 개선 → 둘 다 실패
  Refactor 중 기능 추가 → 테스트 보호 없이 변경

사이클을 짧게 유지하는 방법:
  동작을 가능한 작게 쪼갠다
  테스트 목록으로 전체 그림 유지
  한 번에 하나씩

테스트 먼저 쓰기 좋은 경우:
  비즈니스 로직, 버그 재현/수정
나중에 써도 되는 경우:
  탐색적 코딩, 프레임워크 설정
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 TDD 사이클에서 잘못된 단계를 찾고, 올바른 순서로 수정하라.

```
① 주문 서비스 전체 구현 (OrderService, DiscountPolicy, InventoryChecker, EmailSender)
② 테스트 작성: 주문_생성_전체_플로우 검증
③ 테스트 통과 확인
④ 리팩터링: 중복 제거, 메서드 추출
```

**Q2.** Green 단계에서 "가장 단순한 구현"의 한계는 무엇인가? 아래 케이스에서 어디까지가 "충분히 단순"인가?

```java
@Test
void 첫번째_테스트() {
    assertThat(service.calculate(10_000)).isEqualTo(100);
}

// 선택지 A
public int calculate(int amount) {
    return 100; // 하드코딩
}

// 선택지 B
public int calculate(int amount) {
    return amount / 100;
}
```

**Q3.** 버그 수정 시 TDD를 적용하는 방법을 설명하라. "주문 금액이 0원일 때 포인트가 -1로 계산된다"는 버그가 보고됐을 때 순서를 설명하라.

> 💡 **해설**

**Q1.**

잘못된 부분: 구현(①)이 테스트(②)보다 먼저 작성됐다. TDD는 테스트가 먼저다.

더 근본적인 문제: 한 번에 너무 많이 구현했다. "전체 플로우"를 한 번에 구현하고 테스트하면 TDD의 짧은 사이클이 깨진다.

올바른 순서:

```
사이클 1:
  ① Red: 주문을_생성하면_PENDING_상태다 테스트 작성
  ② Green: Order 객체에 status = PENDING 구현
  ③ Refactor: 개선 없음

사이클 2:
  ① Red: VIP_회원은_10퍼센트_할인 테스트 작성
  ② Green: DiscountPolicy 도입
  ③ Refactor: 할인 계산 추출

사이클 3:
  ① Red: 주문_생성_시_이메일_발송 테스트 작성
  ② Green: EmailSender 주입 및 호출
  ③ Refactor: ...
```

각 사이클이 10분 이내에 완료되어야 한다.

**Q2.**

두 선택지 모두 테스트를 통과시키지만 의미가 다르다.

선택지 A(`return 100`)는 TDD의 "Fake It" 전략으로 일시적으로 허용된다. 단, 두 번째 테스트가 추가되면 즉시 통하지 않는다.

```java
@Test
void 두번째_테스트() {
    assertThat(service.calculate(20_000)).isEqualTo(200);
}
// A: return 100 → 실패 → 이제 일반화 필요
// B: return amount / 100 → 통과
```

"충분히 단순"의 기준: 다음 테스트를 추가했을 때 Green이 깨지는 구현이라면 Refactor에서 일반화한다. 선택지 A는 두 번째 테스트에서 강제로 일반화하게 된다. 이것도 유효한 TDD 전략이다. 선택지 B는 이미 일반화된 구현이므로 더 직접적이다. 실무에서는 B처럼 바로 일반화하는 경우가 많다.

**Q3.**

버그 수정에 TDD 적용:

```
① Red: 버그를 재현하는 테스트 작성

@Test
void 금액이_0원이면_포인트는_0() {
    assertThat(service.calculate(0)).isEqualTo(0);
}
// 실행 → -1 반환 → 테스트 실패 확인 (버그 재현 성공)

② Green: 버그 수정

public int calculate(int amount) {
    if (amount <= 0) return 0; // 버그 수정
    return amount / 100;
}
// 테스트 통과 확인

③ Refactor: 코드 정리 (필요한 경우)

④ 회귀 보장:
  이 테스트가 코드베이스에 영구적으로 남아
  같은 버그가 재발하면 즉시 감지한다
```

버그 수정에 TDD를 적용하면 두 가지 이점이 있다. 버그가 실제로 존재함을 테스트로 증명하고, 수정 후 같은 버그가 재발하지 않도록 회귀 테스트가 자동으로 만들어진다.

---

<div align="center">

**[⬅️ 이전: Architecture Testing](./03-architecture-testing.md)** | **[다음: Test-Driven Design ➡️](./05-test-driven-design.md)**

</div>
