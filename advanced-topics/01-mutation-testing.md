# 01. Mutation Testing

> **커버리지 100%인데 버그를 잡지 못하는 테스트 — 코드를 실행했다는 증거와 코드를 검증했다는 증거는 다르다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 코드 커버리지가 테스트 품질을 보장하지 못하는 이유는 무엇인가?
- 뮤테이션 테스팅은 무엇을 측정하는가?
- PITest를 어떻게 설정하고, 결과를 어떻게 해석하는가?

---

## 🔍 커버리지의 거짓 안도감

```java
// 커버리지 100% 달성 — 하지만 이 테스트는 무엇을 검증하는가?
@Test
void 할인_계산() {
    DiscountService service = new DiscountService();
    service.calculate(vipUser, 20_000); // 반환값을 검증하지 않음
    // assertThat(...) 없음
}
```

이 테스트는 `calculate()` 메서드의 모든 라인을 실행합니다. 커버리지는 100%입니다. 하지만 반환값을 검증하지 않으므로 계산 결과가 틀려도 통과합니다.

```java
// 더 교묘한 경우
@Test
void 최소_금액_검증() {
    boolean result = orderValidator.isValidAmount(1_000);
    assertThat(result).isTrue();
}
// 1_000이 최솟값일 때 true여야 하는데,
// 코드가 "amount > 0"으로 잘못 구현되어도 통과
// "amount >= 1_000"이 올바른 구현이지만 이 테스트로는 구분 불가
```

---

## 🏛️ 뮤테이션 테스팅이란

뮤테이션 테스팅은 프로덕션 코드에 의도적으로 작은 오류(뮤턴트)를 주입하고, 테스트가 그 오류를 발견(킬)하는지 측정합니다.

```
뮤테이션 테스팅의 흐름:

원본 코드: if (amount >= 1_000)
뮤턴트 1:  if (amount > 1_000)    ← 경계값 조건 변경
뮤턴트 2:  if (amount <= 1_000)   ← 조건 반전
뮤턴트 3:  if (true)              ← 항상 참
뮤턴트 4:  return 0;              ← 반환값 변경

테스트 스위트 실행:
  뮤턴트 1 → 테스트 실패 → "Killed" ✅
  뮤턴트 2 → 테스트 통과 → "Survived" ❌ (테스트가 이 오류를 못 잡음)
```

```
뮤테이션 점수 (Mutation Score):
  = Killed / (Killed + Survived) × 100

100점: 모든 뮤턴트를 테스트가 잡아냄 → 높은 테스트 품질
낮은 점수: 테스트가 잡지 못한 오류가 많음 → 약한 단언
```

---

## 🏛️ PITest 설정

```kotlin
// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    targetClasses.set(listOf("com.example.order.*"))
    targetTests.set(listOf("com.example.order.*Test"))
    mutators.set(listOf("STRONGER"))  // 강력한 뮤테이터 세트
    outputFormats.set(listOf("HTML", "XML"))
    timestampedReports.set(false)
    threads.set(4)  // 병렬 실행
}
```

```bash
# 실행
./gradlew pitest

# 결과 위치
build/reports/pitest/index.html
```

---

## 😱 Survived 뮤턴트: 테스트가 약하다는 신호

### 예시: 경계값 테스트 누락

```java
// 프로덕션 코드
public boolean isEligibleForFreeShipping(int totalPrice) {
    return totalPrice >= 50_000;
}
```

```java
// ❌ 이 테스트는 경계값을 검증하지 않는다
@Test
void 무료_배송_자격_확인() {
    assertThat(service.isEligibleForFreeShipping(100_000)).isTrue();
    assertThat(service.isEligibleForFreeShipping(10_000)).isFalse();
}
```

PITest 뮤턴트:
- `>= 50_000` → `> 50_000` — **Survived** ❌ (100_000과 10_000은 구분하지만 정확한 경계를 잡지 못함)
- `>= 50_000` → `<= 50_000` — Killed ✅

```java
// ✅ 경계값 테스트 추가 → 뮤턴트 킬
@Test
void 무료_배송_경계값_검증() {
    assertThat(service.isEligibleForFreeShipping(49_999)).isFalse(); // 경계 직전
    assertThat(service.isEligibleForFreeShipping(50_000)).isTrue();  // 경계값
    assertThat(service.isEligibleForFreeShipping(50_001)).isTrue();  // 경계 직후
}
```

### 예시: 반환값 미검증

```java
// 프로덕션 코드
public int calculateDiscount(User user, int price) {
    if (user.grade() == Grade.VIP) {
        return price / 10;  // 10% 할인
    }
    return 0;
}
```

```java
// ❌ 반환값을 검증하지 않는다
@Test
void 할인_계산() {
    service.calculateDiscount(vipUser, 20_000);
    // assertThat 없음
}
```

PITest 뮤턴트:
- `return price / 10` → `return 0` — **Survived** ❌
- `return price / 10` → `return price` — **Survived** ❌

```java
// ✅ 반환값 검증 → 뮤턴트 킬
@Test
void VIP_10퍼센트_할인() {
    assertThat(service.calculateDiscount(vipUser, 20_000)).isEqualTo(2_000);
}

@Test
void 일반_회원_할인_없음() {
    assertThat(service.calculateDiscount(normalUser, 20_000)).isEqualTo(0);
}
```

---

## ✨ 뮤테이션 점수 해석과 목표 설정

```
일반적인 뮤테이션 점수 기준:

< 60%: 테스트가 코드를 제대로 검증하지 못함
       단언이 약하거나 경계값 테스트 누락

60~80%: 일반적인 수준
        중요한 비즈니스 로직에 취약점이 있을 수 있음

80~90%: 좋은 수준
        대부분의 비즈니스 로직이 충분히 검증됨

> 90%: 매우 높은 수준
       일부 뮤턴트는 동등 뮤턴트(equivalent mutant)일 가능성

100% 목표는 비현실적:
  동등 뮤턴트(Equivalent Mutant): 동작이 바뀌지 않는 코드 변경
  예: i++ → ++i in "for (int i = 0; i < n; i++)"
```

### 동등 뮤턴트 — 킬 불가능

```java
// 원본
for (int i = 0; i < list.size(); i++) { ... }

// 뮤턴트: i++ → ++i
// 동작이 완전히 동일 → 어떤 테스트로도 킬 불가능
// PITest에서 "NO_COVERAGE" 또는 통계에서 제외
```

---

## 💻 실전 적용: 어디에 집중할 것인가

```
뮤테이션 테스팅 적용 우선순위:

높음:
  핵심 비즈니스 로직 (할인 계산, 검증, 상태 전이)
  돈과 관련된 계산
  보안 관련 코드

중간:
  도메인 객체 메서드
  유틸리티 클래스

낮음 (제외해도 됨):
  DTO, 단순 getter/setter
  Spring 설정 클래스
  단순 위임(Delegation) 코드
```

```kotlin
// PITest 제외 설정
pitest {
    excludedClasses.set(listOf(
        "com.example.*.dto.*",
        "com.example.*.config.*",
        "com.example.*.*Config"
    ))
    excludedMethods.set(listOf("getId", "setId", "equals", "hashCode", "toString"))
}
```

---

## 🤔 트레이드오프

### "PITest 실행이 너무 느리다"

뮤테이션 테스팅은 각 뮤턴트마다 테스트 스위트를 실행하므로 본래 느립니다.

완화 방법:

① `threads` 설정으로 병렬 실행

② `targetClasses`로 핵심 비즈니스 로직 패키지만 대상으로 설정

③ CI에서 전체 빌드마다 실행하지 않고, PR 단위 또는 nightly 빌드로 분리

④ `--features +GIT_CI_LAST_COMMIT` 옵션으로 최근 변경된 클래스만 대상으로 실행

### "뮤테이션 점수 100%를 목표로 해야 하는가?"

아닙니다. 동등 뮤턴트 때문에 100%는 달성할 수 없는 경우가 많습니다. 또한 점수를 올리기 위해 단언만 추가하고 테스트 의도를 잃어버리는 것은 역효과입니다. 80~85% 목표를 설정하고, 중요한 비즈니스 로직의 Survived 뮤턴트를 우선 처리합니다.

---

## 📌 핵심 정리

```
커버리지 vs 뮤테이션 점수:
  커버리지: "이 코드가 실행됐는가?"
  뮤테이션 점수: "이 코드가 검증됐는가?"

PITest 설정:
  targetClasses: 핵심 비즈니스 로직 패키지
  mutators: STRONGER (기본보다 강력)
  excludedClasses: DTO, 설정, 단순 위임

Survived 뮤턴트의 의미:
  경계값 테스트 누락 → 경계값 테스트 추가
  반환값 미검증 → assertThat 추가
  조건 분기 미검증 → 음수/양수 케이스 추가

목표 점수:
  핵심 로직: 80~85%
  100% 목표 불필요 (동등 뮤턴트)

실행 전략:
  PR 단위 또는 Nightly 빌드
  최근 변경 파일만 대상
  병렬 실행 (threads 설정)
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 코드에 PITest를 실행하면 어떤 뮤턴트가 Survived할 것으로 예상되는가? 그 뮤턴트를 킬하기 위해 어떤 테스트를 추가해야 하는가?

```java
public class PointPolicy {
    public int calculate(int purchaseAmount) {
        if (purchaseAmount >= 100_000) return purchaseAmount / 50; // 2%
        if (purchaseAmount >= 50_000) return purchaseAmount / 100; // 1%
        return 0;
    }
}

// 현재 테스트
@Test
void 포인트_계산() {
    assertThat(policy.calculate(200_000)).isEqualTo(4_000);
    assertThat(policy.calculate(60_000)).isEqualTo(600);
    assertThat(policy.calculate(10_000)).isEqualTo(0);
}
```

**Q2.** 뮤테이션 점수가 50%인 팀이 있다. 이 점수를 단기간에 올리기 위해 "단언만 추가하는" 전략을 쓰면 어떤 문제가 생기는가? 올바른 접근 방법은 무엇인가?

**Q3.** PITest에서 아래 코드의 뮤턴트는 항상 Survived다. 왜 그런가? 이것은 테스트 문제인가, 코드 문제인가?

```java
public String formatOrderId(Long orderId) {
    return "ORD-" + orderId;
}

@Test
void 주문_ID_포맷() {
    assertThat(service.formatOrderId(123L)).startsWith("ORD-");
}
```

> 💡 **해설**

**Q1.**

예상 Survived 뮤턴트:

① `>= 100_000` → `> 100_000`: 경계값 100_000에서 어떤 포인트를 받는지 테스트가 없다. 현재 테스트에서 200_000만 사용하므로 이 뮤턴트를 구분하지 못한다.

② `>= 50_000` → `> 50_000`: 경계값 50_000에서의 동작이 테스트되지 않았다.

③ `return purchaseAmount / 50` → `return purchaseAmount / 100`: 두 조건의 반환 공식이 비슷해서 테스트가 이를 구분하지 못할 수 있다.

킬을 위해 추가해야 할 테스트:

```java
@Test
void 포인트_경계값_검증() {
    // 100_000 경계
    assertThat(policy.calculate(99_999)).isEqualTo(999);   // 1% = 999
    assertThat(policy.calculate(100_000)).isEqualTo(2_000); // 2% = 2_000
    assertThat(policy.calculate(100_001)).isEqualTo(2_000); // 2% = 2_000

    // 50_000 경계
    assertThat(policy.calculate(49_999)).isEqualTo(0);      // 0%
    assertThat(policy.calculate(50_000)).isEqualTo(500);    // 1% = 500
    assertThat(policy.calculate(50_001)).isEqualTo(500);    // 1% = 500
}
```

**Q2.**

단순히 단언만 추가하는 전략의 문제: 테스트가 검증하는 게 무엇인지 의도 없이 숫자만 채우게 된다. 예를 들어 뮤테이션 점수를 올리기 위해 `assertThat(obj).isNotNull()`만 추가하면 실제 의미 있는 검증은 없지만 점수는 오른다. 테스트 코드가 빠르게 늘어나지만 품질은 개선되지 않는다.

올바른 접근: Survived 뮤턴트를 보고 "이 오류가 실제 버그라면 어떤 상황에서 발생할까?"를 생각한다. 그 상황을 재현하는 테스트 시나리오를 먼저 설계하고, 시나리오에서 자연스럽게 단언이 나오도록 작성한다.

**Q3.**

이유: `startsWith("ORD-")`는 접두사만 검증한다. `"ORD-" + orderId`에서 `+ orderId` 부분을 제거하거나 다른 값으로 바꾸는 뮤턴트는 여전히 `startsWith("ORD-")`를 만족한다.

코드 문제가 아니라 테스트 문제다. `formatOrderId`의 목적은 정확한 형식(`ORD-123`)을 만드는 것이다.

수정:

```java
@Test
void 주문_ID_포맷() {
    assertThat(service.formatOrderId(123L)).isEqualTo("ORD-123");
}
```

`isEqualTo()`로 전체 값을 검증하면 뮤턴트가 즉시 Killed된다.

---

<div align="center">

**[다음: Property-Based Testing ➡️](./02-property-based-testing.md)**

</div>
