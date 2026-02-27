# 02. The Test Pyramid

> **피라미드의 비율은 "빠름 vs 정확함"의 경제학이다 — 이유 없이 E2E를 쌓으면 팀이 느려진다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 피라미드의 3층은 각각 무엇을 검증하고, 무엇을 검증하지 않는가?
- 피라미드가 역삼각형이 되면 팀에 어떤 일이 생기는가?
- 단위 테스트만 있을 때 잡지 못하는 버그는 어떤 종류인가?

---

## 🔍 왜 이 구조가 필요한가

```java
// 이 코드에는 3가지 종류의 잠재적 버그가 있다

public class PaymentService {
    private final PaymentGateway gateway;        // 외부 결제 API
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;

    public PaymentResult pay(Order order, Card card) {
        validate(order);                          // ← 버그 A: 비즈니스 로직 오류
        PaymentResult result = gateway.charge(card, order.amount()); // ← 버그 B: API 연동 오류
        orderRepository.updateStatus(order.id(), PAID);
        eventPublisher.publish(new PaymentCompleted(order)); // ← 버그 C: 이벤트 누락
        return result;
    }
}
```

**버그 A** (비즈니스 로직): 단위 테스트가 잡는다  
**버그 B** (외부 연동): 실제 게이트웨이 또는 WireMock 연동 테스트가 잡는다  
**버그 C** (전체 흐름): E2E 테스트가 잡는다

세 종류의 버그를 하나의 테스트 레이어로 잡으려 하면, 느리고 불안정하고 유지보수하기 어려운 테스트가 됩니다. 테스트 피라미드는 **버그의 종류에 따라 적합한 테스트 레이어를 분리**하는 전략입니다.

---

## 🏛️ 피라미드의 3층

```
          ▲
        /E2E\          소수 (5~10%)
       /─────\         느림, 비용 높음, 전체 흐름 검증
      /  INT  \        중간 (20~30%)
     /─────────\       DB·API·Spring 컨텍스트 경계 검증
    /   UNIT    \      다수 (60~70%)
   /─────────────\     빠름, 비용 낮음, 비즈니스 로직 검증
```

### Unit (단위 테스트)

- **무엇을 검증하는가:** 하나의 동작(behavior), 비즈니스 규칙, 도메인 로직
- **무엇을 검증하지 않는가:** DB 저장 여부, 외부 API 응답, 스프링 빈 연결
- **속도:** 수천 개가 수초 내 완료
- **실패 의미:** "이 로직이 틀렸다"

### Integration (통합 테스트)

- **무엇을 검증하는가:** 두 컴포넌트가 연결되어 올바르게 동작하는가 (Repository + DB, Controller + Spring)
- **무엇을 검증하지 않는가:** 전체 사용자 시나리오, 비즈니스 로직 세부사항
- **속도:** 수십~수백 개, 수십 초~수 분
- **실패 의미:** "이 연결이 잘못됐다"

### E2E (End-to-End)

- **무엇을 검증하는가:** 사용자 관점의 전체 시나리오, UI부터 DB까지
- **무엇을 검증하지 않는가:** 경계값, 예외 케이스 세부사항
- **속도:** 수 개~수십 개, 수 분~수십 분
- **실패 의미:** "이 사용자 흐름이 깨졌다"

---

## 😱 역삼각형 피라미드: 팀에 어떤 일이 생기는가

많은 팀이 "실제와 가장 비슷한 테스트가 좋은 테스트"라고 생각해서 E2E를 쌓기 시작합니다.

```
         ▲
        /E2E\          다수 (60%)  ← 잘못된 구조
       /─────\
      /  INT  \        중간 (30%)
     /─────────\
    /   UNIT    \      소수 (10%)
   /─────────────\
```

```java
// E2E 테스트 한 줄의 비용

@Test
void 주문부터_결제까지_전체_플로우() {
    // 1. 브라우저 띄우기 (3초)
    // 2. 로그인 (2초)
    // 3. 상품 검색 (1초)
    // 4. 장바구니 담기 (1초)
    // 5. 결제 정보 입력 (2초)
    // 6. 결제 완료 확인 (2초)
    // 합계: 11초 × 100개 = 18분
}
```

### 발생하는 문제들

**CI가 30분이 넘어가기 시작한다**  
PR마다 30분씩 기다리면 개발자들은 피드백 루프가 너무 느리다고 느끼기 시작합니다. 결국 테스트를 건너뛰거나 PR을 크게 만들게 됩니다.

**Flaky Test가 팀을 괴롭힌다**  
E2E는 네트워크, 타이밍, 브라우저 렌더링, 외부 서비스 상태에 의존합니다. 이유 없이 실패하는 테스트가 쌓이면 팀은 빨간 CI를 무시하기 시작합니다. "빨간 건 원래 빨개"라는 문화가 생깁니다.

**실패 원인을 찾는 데 시간이 걸린다**  
E2E가 실패했을 때 원인이 프론트엔드인지, API인지, DB인지, 네트워크인지 찾는 데 단위 테스트 실패보다 훨씬 오래 걸립니다.

---

## ✨ 올바른 피라미드: 레이어별 책임 분리

```java
// 같은 기능을 세 레이어로 나눠서 검증하는 방법

// ─────────────────────────────────────────
// UNIT: 비즈니스 로직만 격리해서 검증
// ─────────────────────────────────────────
class PaymentServiceTest {
    private final PaymentGateway gateway = new FakePaymentGateway();
    private final OrderRepository repo = new InMemoryOrderRepository();
    private final PaymentService service = new PaymentService(gateway, repo, ...);

    @Test
    void 만원_미만_주문은_결제할_수_없다() {
        Order tooSmall = Order.of(amount(500));
        assertThatThrownBy(() -> service.pay(tooSmall, anyCard()))
            .isInstanceOf(MinimumAmountException.class);
    }

    @Test
    void 결제_성공_시_주문_상태가_PAID로_변경된다() {
        Order order = Order.of(amount(10_000));
        service.pay(order, validCard());
        assertThat(repo.findById(order.id()).status()).isEqualTo(PAID);
    }
}

// ─────────────────────────────────────────
// INTEGRATION: DB 연결만 검증
// ─────────────────────────────────────────
@DataJpaTest
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_저장_후_상태_변경이_반영된다() {
        Order order = orderRepository.save(Order.of(amount(10_000)));
        orderRepository.updateStatus(order.id(), PAID);
        assertThat(orderRepository.findById(order.id()).status()).isEqualTo(PAID);
    }
}

// ─────────────────────────────────────────
// E2E: 사용자 흐름 전체만 검증 (소수)
// ─────────────────────────────────────────
class PaymentE2ETest {
    @Test
    void 상품을_선택하고_결제하면_주문완료_페이지로_이동한다() {
        // 핵심 happy path만, 경계값은 여기서 검증하지 않음
    }
}
```

### 레이어별 테스트 개수 가이드

| 레이어 | 비율 | 검증 포인트 |
|--------|------|------------|
| Unit | 60~70% | 모든 비즈니스 규칙, 경계값, 예외 경로 |
| Integration | 20~30% | 각 외부 연결 1~2개씩 (Repository, Controller 슬라이스) |
| E2E | 5~10% | 핵심 사용자 시나리오 happy path 위주 |

---

## 💻 실전 적용: 내 프로젝트 피라미드 점검하기

```bash
# 테스트 개수 현황 빠르게 파악
grep -r "@Test" src/test/java --include="*.java" | wc -l          # 전체
grep -r "@SpringBootTest" src/test/java --include="*.java" | wc -l # E2E/통합
grep -r "@DataJpaTest\|@WebMvcTest" src/test/java --include="*.java" | wc -l # 슬라이스
```

```java
// 현재 E2E 테스트에서 이런 케이스가 있다면 Unit으로 내려야 한다
@SpringBootTest
class PaymentE2ETest {

    // ❌ 경계값 테스트 → Unit으로 내려야 함
    @Test void 금액이_0원이면_실패한다() { ... }
    @Test void 금액이_음수면_실패한다() { ... }
    @Test void 카드번호가_16자리가_아니면_실패한다() { ... }

    // ✅ 이것만 E2E에 남겨야 함
    @Test void 정상_결제_플로우() { ... }
}
```

---

## 🤔 트레이드오프

### 피라미드가 항상 정답은 아닌 경우

**UI가 핵심인 제품 (프론트엔드 중심)**  
비즈니스 로직보다 화면 렌더링과 사용자 인터랙션이 핵심이라면 E2E의 비중이 자연스럽게 높아집니다. *Testing Trophy* (Kent C. Dodds) 모델은 Integration 비중을 높이는 방향을 제안합니다.

**마이크로서비스 경계가 복잡한 경우**  
서비스 간 계약이 자주 바뀐다면 Contract Testing (Pact)이 E2E보다 효과적입니다. E2E 없이 Contract Test + Unit Test 조합으로 충분할 수 있습니다.

**레거시 코드베이스에 처음 테스트를 추가할 때**  
단위 테스트를 넣기 어려운 코드라면 E2E로 먼저 안전망을 만들고, 리팩터링하면서 단위 테스트로 옮기는 전략이 현실적입니다.

---

## 📌 핵심 정리

```
테스트 피라미드

Unit (60~70%):  비즈니스 로직, 도메인 규칙, 경계값
Integration (20~30%): DB · API · Spring 연결 검증
E2E (5~10%): 핵심 사용자 시나리오 happy path

역삼각형이 만드는 문제:
  CI 속도 저하 → PR 주기 길어짐
  Flaky Test → 신뢰 하락
  실패 원인 파악 어려움

레이어 분리 기준:
  "이 테스트가 느려도 되는 이유가 있는가?"
  → 없다면 Unit으로 내린다

  "외부 시스템 없이도 이 로직을 검증할 수 있는가?"
  → 있다면 단위 테스트가 맞다
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트들을 Unit / Integration / E2E 중 어느 레이어로 분류하겠는가? 그리고 잘못된 레이어에 있는 것이 있다면 어떻게 옮기겠는가?

```java
// 테스트 A
@SpringBootTest
void 쿠폰_코드가_만료됐으면_할인이_적용되지_않는다() { ... }

// 테스트 B
@DataJpaTest
void 동일한_이메일로_두_번_가입하면_예외가_발생한다() { ... }

// 테스트 C — Fake 사용
void 재고가_0이면_주문_생성이_실패한다() { ... }
```

**Q2.** 팀원이 "단위 테스트만 있으면 통합 테스트는 필요 없다. 어차피 Mock으로 다 검증되지 않느냐"고 주장한다. 어떻게 반론하겠는가? 단위 테스트가 절대 잡을 수 없는 버그 유형을 예시 코드와 함께 설명하라.

**Q3.** `@SpringBootTest`로 작성된 테스트 100개짜리 레거시 프로젝트를 맡았다. CI가 20분이 걸린다. 지금 당장 할 수 있는 가장 효과적인 첫 번째 조치는 무엇인가?

> 💡 **해설**
>
> **Q1.** 테스트 A: 쿠폰 만료 검증은 비즈니스 로직이다. `@SpringBootTest`는 과도하다 — Unit으로 내려야 한다. `CouponPolicy` 단위로 격리해서 빠르게 검증 가능하다. 테스트 B: `@DataJpaTest`는 적절하다. 이메일 유니크 제약은 DB 레벨 제약이라 실제 DB 연결이 필요하다. 단, 비즈니스 로직 검증(`UserService.register()`에서 중복 체크)은 별도로 Unit에서도 해야 한다. 테스트 C: Unit이 맞다. Fake를 사용하고 외부 의존 없이 비즈니스 규칙만 검증하고 있다.
>
> **Q2.** 단위 테스트가 잡을 수 없는 대표적 버그: ① DB 스키마 불일치 — `@Column(name = "user_email")`인데 실제 DB 컬럼이 `email`이면 단위 테스트(Mock Repository)는 통과, 실제 실행에서 `SqlException`이 발생한다. ② JPA N+1 — Mock으로 `findAll()`을 테스트하면 쿼리 수를 검증할 수 없다. 실제 DB에서 1000개 조회 시 1001번 쿼리가 발생하는 걸 통합 테스트만 잡는다. ③ 트랜잭션 경계 — `@Transactional` 설정이 잘못되면 도메인 로직은 맞아도 커밋이 안 되는 버그가 발생한다.
>
> **Q3.** `@SpringBootTest` 100개를 `@WebMvcTest` / `@DataJpaTest`로 분류해서 슬라이싱하는 것이 가장 효과적이다. 전체 컨텍스트 로딩 대신 필요한 빈만 로딩하면 테스트 하나당 시간이 크게 줄어든다. 그 다음은 비즈니스 로직 검증 목적의 `@SpringBootTest`를 Unit으로 내리는 것이다.

---

<div align="center">

**[⬅️ 이전: What Is a Unit](./01-what-is-a-unit.md)** | **[다음: AAA Pattern ➡️](./03-aaa-pattern.md)**

</div>
