# 05. Coverage Myths

> **커버리지는 테스트의 부재를 알려주지만, 테스트의 존재를 보장하지는 않는다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 커버리지 100%인데도 버그가 발생할 수 있는 이유는 무엇인가?
- 커버리지가 측정하는 것과 측정하지 못하는 것은 각각 무엇인가?
- 팀에서 커버리지 목표를 어떻게 설정해야 하는가?

---

## 🔍 문제 상황: 100% 커버리지, 그런데 버그가 있다

```java
public class DiscountCalculator {

    public int calculate(int price, String grade) {
        if (grade.equals("VIP")) {
            return (int) (price * 0.9);  // VIP: 10% 할인
        }
        return price;
    }
}
```

```java
// 커버리지 100% 달성하는 테스트
@Test
void VIP_회원_할인_계산() {
    DiscountCalculator calculator = new DiscountCalculator();
    int result = calculator.calculate(10_000, "VIP");
    assertThat(result).isGreaterThan(0);  // ← 단언이 너무 약하다
}

@Test
void 일반_회원_할인_없음() {
    DiscountCalculator calculator = new DiscountCalculator();
    int result = calculator.calculate(10_000, "NORMAL");
    assertThat(result).isNotNull();       // ← int는 null이 될 수 없다. 항상 통과
}
```

이 두 테스트는 모든 분기를 실행합니다. **커버리지는 100%입니다.**  
하지만 검증하는 것이 없습니다. `calculate(10_000, "VIP")`가 `1`을 반환해도, `-99`를 반환해도 테스트는 통과합니다.

또한 이 테스트는 다음 버그를 전혀 발견하지 못합니다.

```java
// grade가 null이면?
calculator.calculate(10_000, null);  // NullPointerException

// grade가 소문자면?
calculator.calculate(10_000, "vip");  // 할인 미적용, 무성한 실패
```

---

## 🏛️ 커버리지가 측정하는 것과 못 하는 것

### 커버리지가 측정하는 것

```
Line Coverage:   실행된 라인 / 전체 라인
Branch Coverage: 실행된 분기 / 전체 분기 (if-else 양쪽 포함)
Method Coverage: 호출된 메서드 / 전체 메서드
```

커버리지가 말하는 것: **"이 코드가 실행됐다"**

### 커버리지가 측정하지 못하는 것

**1. 단언의 품질**

코드가 실행됐다고 해서 결과가 올바른지는 알 수 없습니다. 위의 `isGreaterThan(0)` 예시처럼, 단언이 있어도 의미 없는 단언이면 커버리지와 관계없이 버그를 잡지 못합니다.

**2. 경계값**

```java
public boolean isAdult(int age) {
    return age >= 18;
}
```

```java
// Branch Coverage 100% 달성
@Test void 성인이면_true() { assertThat(isAdult(30)).isTrue(); }
@Test void 미성년자면_false() { assertThat(isAdult(10)).isFalse(); }
```

`isAdult(17)`과 `isAdult(18)`은 경계값입니다. 위 테스트는 `>= 18`이 `> 18`로 잘못 구현되어도 통과합니다. 커버리지는 경계값을 모릅니다.

**3. null과 예외 경로**

```java
public Order findOrder(Long id) {
    return repository.findById(id);  // null 반환 가능
}
```

```java
// findOrder() 호출 라인은 커버됨 — 하지만 null 케이스는?
@Test void 주문_조회() {
    Order order = service.findOrder(1L);
    assertThat(order.id()).isEqualTo(1L); // null이면 NPE, 테스트가 "실패"하지만 의도한 케이스가 아님
}
```

**4. 조합 케이스**

```java
public int calculate(int price, String grade, boolean hasCoupon) {
    // 2 × 2 = 4가지 조합
}
```

각 분기를 개별적으로 커버해도, 조합(`grade=VIP AND hasCoupon=true`)이 테스트되지 않을 수 있습니다.

---

## 😱 커버리지 목표를 숫자로 강제하면

```yaml
# 많은 팀이 이렇게 설정한다
jacoco:
  minimum:
    coverage: 80%
```

이 목표를 달성하기 위해 팀원이 무의식적으로 이런 테스트를 작성하기 시작합니다.

```java
// ❌ 커버리지를 채우기 위한 테스트 — 아무것도 검증하지 않는다
@Test
void getter_setter_테스트() {
    User user = new User();
    user.setName("alice");
    user.setEmail("alice@example.com");

    assertThat(user.getName()).isEqualTo("alice");     // getter 실행을 위한 단언
    assertThat(user.getEmail()).isEqualTo("alice@example.com");
}

@Test
void 생성자_테스트() {
    Order order = new Order(1L, List.of(), 0);  // 생성자 라인 커버
    assertThat(order).isNotNull();              // 항상 통과
}
```

이런 테스트는 커버리지는 높이지만 버그를 전혀 잡지 못합니다. 오히려 유지보수해야 할 코드만 늘어납니다.

---

## ✨ 커버리지를 올바르게 사용하는 방법

### 커버리지는 "부재의 지표"로 쓴다

커버리지가 낮은 부분을 찾아서 "여기에 테스트가 없다"는 것을 발견하는 용도입니다.

```bash
# IntelliJ에서 커버리지 리포트 생성
./gradlew test jacocoTestReport

# 리포트 확인 경로
open build/reports/jacoco/test/html/index.html
```

커버리지 리포트에서 빨간 줄을 보면 "이 코드가 테스트되지 않았다"는 신호입니다. 하지만 초록 줄이 "이 코드가 올바르게 검증됐다"는 의미는 아닙니다.

### 의미 있는 커버리지 목표 설정

```
❌ "전체 커버리지 80% 이상"
  → 게터/세터 테스트로 채울 수 있음

✅ "핵심 비즈니스 로직 패키지는 Branch Coverage 90% 이상"
  → 도메인 규칙, 결제 로직, 할인 정책에 집중

✅ "새로 추가되는 코드는 커버리지를 낮추지 않는다 (ratchet)"
  → 절대값이 아닌 방향성을 강제
```

### 커버리지 대신 이것을 확인하라

```java
// 커버리지 100%인 테스트
@Test void 할인_적용() {
    assertThat(calculator.calculate(10_000, "VIP")).isGreaterThan(0); // 약한 단언
}

// 커버리지는 90%지만 더 가치 있는 테스트
@Test void VIP_10퍼센트_할인_적용() {
    assertThat(calculator.calculate(10_000, "VIP")).isEqualTo(9_000); // 정확한 단언
}

@Test void 경계값_VIP_최소금액_1원에서도_할인_적용() {
    assertThat(calculator.calculate(1, "VIP")).isEqualTo(0); // 정수 절삭 검증
}

@Test void null_등급_전달_시_예외_발생() {
    assertThatThrownBy(() -> calculator.calculate(10_000, null))
        .isInstanceOf(InvalidGradeException.class);
}
```

---

## 💻 실전 적용: 커버리지가 낮은 부분 찾기

```java
// build.gradle
plugins {
    id 'jacoco'
}

jacocoTestReport {
    reports {
        html.required = true
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/dto/**',        // DTO getter/setter 제외
                '**/config/**',     // 설정 클래스 제외
                '**/*Application*'  // 메인 클래스 제외
            ])
        }))
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'PACKAGE'
            includes = ['com.example.domain.*']  // 핵심 비즈니스 로직만
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.90
            }
        }
    }
}
```

---

## 🤔 트레이드오프

### 커버리지 목표를 아예 없애야 하는가?

그렇지 않습니다. 커버리지 목표가 전혀 없으면 팀에서 테스트를 아예 안 작성하는 영역이 생깁니다. 커버리지 목표는 "이 정도는 최소한 실행해봐야 한다"는 하한선으로는 유효합니다.

다만 목표를 맹목적으로 쫓는 것이 문제입니다. "커버리지 숫자가 높다 = 코드베이스가 안전하다"는 착각을 경계해야 합니다.

### 실무 권장 기준

| 대상 | 커버리지 기준 |
|------|-------------|
| 도메인 로직, 비즈니스 규칙 | Branch Coverage 90%+ |
| Application Service | Line Coverage 80%+ |
| Controller, Repository | 통합 테스트로 보완 |
| DTO, 설정 클래스 | 측정 제외 |

---

## 📌 핵심 정리

```
커버리지가 말하는 것:
  "이 코드가 실행됐다"

커버리지가 말하지 못하는 것:
  "이 코드가 올바르게 동작했다"
  "경계값이 검증됐다"
  "null과 예외가 처리됐다"
  "조합 케이스가 맞다"

올바른 커버리지 사용:
  낮은 커버리지 → "여기에 테스트가 없다" (부재 탐지)
  높은 커버리지 → "실행은 됐다" (존재 보장 아님)

커버리지 목표 설정:
  절대값(80%) → 게터/세터 테스트를 양산할 위험
  핵심 패키지 + Branch Coverage → 더 의미 있음
  ratchet 방식 → 기존보다 낮아지지 않도록

진짜 중요한 것:
  단언의 품질 (assertThat(result).isEqualTo(9_000))
  경계값 테스트
  null과 예외 경로 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트의 커버리지와 실제 품질을 분석하라. 어떤 버그를 잡지 못하는가?

```java
public int divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("0으로 나눌 수 없음");
    return a / b;
}

@Test void divide_테스트() {
    assertThat(divide(10, 2)).isPositive();
    assertThatThrownBy(() -> divide(1, 0)).isInstanceOf(ArithmeticException.class);
}
```

**Q2.** 팀 리더가 "커버리지 70% 미만이면 PR 머지를 막겠다"는 규칙을 도입하려 한다. 이 규칙의 의도는 좋지만 어떤 부작용이 생길 수 있는가? 더 나은 대안은 무엇인가?

**Q3.** Mutation Testing은 커버리지의 어떤 한계를 극복하는가? 아래 코드에서 커버리지는 100%지만 Mutation Testing이 살아남는 뮤턴트를 찾는다면 어떤 케이스인가?

```java
public boolean isEligible(int age) {
    return age >= 18;  // 뮤턴트: age > 18
}

@Test void 성인() { assertThat(isEligible(20)).isTrue(); }
@Test void 미성년자() { assertThat(isEligible(15)).isFalse(); }
```

> 💡 **해설**
>
> **Q1.** 커버리지는 100%(두 분기 모두 커버). 잡지 못하는 버그: ① `divide(10, 2)`의 결과가 `5`인지 검증하지 않는다. `isPositive()`는 `divide(10, 2) = 4`여도 통과한다(정수 나눗셈 오차). ② 음수 나눗셈 케이스가 없다. `divide(-10, 2) = -5`인지, `divide(10, -2)`는 어떻게 처리하는지. ③ 정확한 단언 필요: `assertThat(divide(10, 2)).isEqualTo(5)`.
>
> **Q2.** 부작용: ① 게터/세터, DTO, 설정 클래스에 무의미한 테스트를 양산한다. ② 복잡한 비즈니스 로직보다 쉽게 커버할 수 있는 단순 코드의 테스트를 먼저 쓰는 왜곡된 인센티브가 생긴다. ③ 테스트 커버리지를 "숫자 게임"으로 인식하기 시작한다. 대안: DTO/설정 클래스를 커버리지에서 제외하고, 핵심 비즈니스 로직 패키지에만 Branch Coverage 기준을 적용한다. PR 리뷰에서 새로 추가된 비즈니스 로직에 테스트가 있는지를 리뷰어가 확인하는 문화가 숫자 규칙보다 효과적인 경우가 많다.
>
> **Q3.** Mutation Testing의 핵심: 코드를 의도적으로 변형(뮤턴트)하고 테스트가 이를 잡는지 확인한다. 위 예시에서 `age >= 18`을 `age > 18`로 변형한 뮤턴트는 `isEligible(20)`이나 `isEligible(15)` 테스트에서 모두 통과한다. `isEligible(18)`이 `true`여야 한다는 경계값 테스트가 없기 때문이다. 커버리지는 "분기를 실행했다"만 알고, "분기의 경계가 올바른가"는 모른다. Mutation Testing은 바로 이 경계 검증 여부를 찾는다.

---

<div align="center">

**[⬅️ 이전: Test Naming Conventions](./04-test-naming-conventions.md)** | **[다음: FIRST Principles ➡️](./06-first-principles.md)**

</div>
