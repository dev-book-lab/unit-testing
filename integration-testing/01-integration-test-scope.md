# 01. Integration Test Scope

> **단위 테스트가 모두 통과했는데 시스템이 망가지는 이유 — 부품이 각각 작동해도 조립이 잘못되면 전체는 실패한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 단위 테스트와 통합 테스트는 각각 무엇을 검증하는가?
- 통합 테스트가 필요한 경계는 어디인가?
- 통합 테스트를 너무 많이 쓰면 어떤 문제가 생기는가?

---

## 🔍 단위 테스트가 잡지 못하는 것

```java
// 단위 테스트: OrderService.place()가 할인을 올바르게 계산한다 ✅
// 단위 테스트: JpaOrderRepository.save()가 Entity를 올바르게 매핑한다 ✅
// 단위 테스트: OrderController가 요청을 올바르게 파싱한다 ✅

// 하지만 실제 시스템에서
POST /orders → 500 Internal Server Error
```

각 부품은 단독으로 잘 작동했지만, 연결되는 순간 문제가 드러납니다.

```java
// 실제 원인 예시들
// 1. JPA Entity에 @Column(nullable = false)가 있는데
//    Service가 null을 저장하려 한다 → DB 제약조건 위반
//
// 2. Controller의 요청 DTO 필드명이 "userId"인데
//    실제 JSON 키는 "user_id"로 클라이언트가 보낸다 → 역직렬화 실패
//
// 3. Repository 메서드 이름이 findByUserIdAndStatus인데
//    JPA가 쿼리를 잘못 생성한다 → 잘못된 데이터 반환
//
// 4. @Transactional 경계가 잘못 설정되어
//    일부 저장만 커밋되고 나머지는 롤백된다
```

이것들은 모두 **경계(boundary)**에서 발생하는 문제입니다. 단위 테스트는 경계 안쪽만 검증하므로 경계 자체의 문제를 잡지 못합니다.

---

## 🏛️ 통합 테스트가 검증해야 하는 경계

```
검증해야 하는 경계:
  ① 애플리케이션 ↔ 데이터베이스
      JPA 쿼리, 제약조건, 트랜잭션, 마이그레이션

  ② 애플리케이션 ↔ 외부 HTTP API
      요청/응답 형식, 인증, 에러 처리

  ③ 레이어 간 (Controller ↔ Service ↔ Repository)
      Spring 빈 연결, 요청 역직렬화, 응답 직렬화

  ④ 서비스 간 (MSA)
      계약(Contract), 이벤트 형식, API 버전
```

---

## 😱 통합 테스트를 잘못 쓰는 두 가지 패턴

### 패턴 1: 모든 것을 통합 테스트로 — 느리고 유지보수 비용이 높다

```java
// ❌ 비즈니스 로직을 통합 테스트로 검증
@SpringBootTest
class OrderServiceIntegrationTest {

    @Autowired OrderService orderService;

    @Test
    void VIP_회원_10퍼센트_할인() {
        // 실제 DB, 실제 Kafka, 전체 Spring 컨텍스트 실행
        // 수 초~수십 초 소요
        // 할인 로직 하나를 위해 너무 많은 비용
        Order order = orderService.place(cart, vipUser);
        assertThat(order.discountRate()).isEqualTo(10);
    }
}
```

이 테스트는 단위 테스트로 ms 만에 검증할 수 있는 것을 수 초에 걸쳐 검증합니다.

### 패턴 2: 통합 테스트가 전혀 없다 — 연결 문제를 prod에서 발견

```java
// ❌ 단위 테스트만 있고 통합 테스트가 없다
// findByUserIdAndStatus 쿼리가 올바르게 생성되는지 검증 없음
// @Transactional 경계가 올바른지 검증 없음
// DB 마이그레이션 스크립트가 올바른지 검증 없음
// → 배포 후 prod에서 발견
```

---

## ✨ 역할 분리: 무엇을 어디서 검증하는가

```
단위 테스트가 담당하는 것:
  비즈니스 규칙 (할인 계산, 검증 로직, 상태 전이)
  도메인 객체의 동작
  복잡한 조건 분기
  경계값

통합 테스트가 담당하는 것:
  JPA 쿼리가 올바른 SQL을 생성하는가
  DB 제약조건이 의도대로 동작하는가
  트랜잭션 경계가 올바른가
  Spring 빈이 올바르게 연결됐는가
  HTTP 요청/응답 직렬화가 올바른가
  외부 API 연동이 올바른가
```

### 겹치는 영역 — 무엇을 기준으로 선택하는가

```
비즈니스 로직 검증 → 단위 테스트
  이유: 빠름, 격리됨, 피드백이 즉각적

기술적 연결 검증 → 통합 테스트
  이유: Mock으로는 실제 DB/프레임워크 동작을 재현할 수 없음
```

---

## 🏛️ 통합 테스트의 종류와 비용

```
빠름 ←──────────────────────────────────→ 느림

단위    슬라이스 테스트         @SpringBootTest    E2E
테스트  @DataJpaTest            with real DB       테스트
ms      @WebMvcTest             수 초~수십 초      수십 초~분
        수백 ms~수 초
```

### 슬라이스 테스트 — 필요한 레이어만 로딩

```java
// @DataJpaTest: JPA 관련 빈만 로딩
// Repository, Entity, JPA 설정만 — Service, Controller 없음
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repository;

    @Test
    void 상태별_주문_조회() {
        repository.save(anOrder().withStatus(PENDING).build());
        repository.save(anOrder().withStatus(COMPLETED).build());

        List<Order> pending = repository.findByStatus(PENDING);
        assertThat(pending).hasSize(1);
    }
}
```

```java
// @WebMvcTest: Web 레이어만 로딩
// Controller, Filter, Interceptor — Service, Repository 없음
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockBean OrderService orderService;

    @Test
    void 주문_생성_요청_역직렬화() throws Exception {
        mockMvc.perform(post("/orders")
                .contentType(APPLICATION_JSON)
                .content("""{"userId": 1, "totalPrice": 20000}"""))
            .andExpect(status().isOk());
    }
}
```

---

## 💻 실전 적용: 통합 테스트 작성 기준

```
이 테스트를 통합 테스트로 작성해야 하는가?

"단위 테스트(Mock/Fake)로 검증할 수 없는가?"
  → NO: 단위 테스트로 작성

"실제 DB/프레임워크 동작이 관여하는가?"
  → YES: 통합 테스트로 작성

구체적인 YES 케이스:
  JPA Named Query / JPQL이 올바른지
  DB Unique 제약조건이 동작하는지
  @Transactional 롤백/커밋이 올바른지
  Spring Security 설정이 올바른지
  JSON 직렬화 설정(@JsonProperty 등)이 올바른지
  외부 API 클라이언트가 올바른 요청을 만드는지
```

---

## 🤔 트레이드오프

### "통합 테스트가 느리면 CI가 느려지지 않는가?"

맞습니다. 통합 테스트는 단위 테스트보다 느립니다. 이를 완화하는 방법:

① 슬라이스 테스트(`@DataJpaTest`, `@WebMvcTest`)를 `@SpringBootTest` 대신 사용 — 로딩 범위 최소화

② 통합 테스트와 단위 테스트를 별도 Maven/Gradle 태스크로 분리 — 단위 테스트는 매 커밋, 통합 테스트는 PR/배포 시

③ Testcontainers의 컨테이너를 테스트 스위트 전체에서 재사용 — 컨테이너 기동 비용 분산

### "E2E 테스트로 전부 검증하면 안 되는가?"

E2E 테스트는 가장 현실적이지만 가장 느리고 불안정합니다. 네트워크, 외부 서비스, 데이터 상태에 따라 결과가 달라집니다. 통합 테스트는 E2E보다 빠르고 안정적으로 경계를 검증합니다.

---

## 📌 핵심 정리

```
단위 테스트가 잡지 못하는 것:
  DB 제약조건, JPA 쿼리, 트랜잭션 경계
  Spring 빈 연결, 직렬화/역직렬화
  외부 시스템 연동

통합 테스트의 역할:
  경계(boundary)에서 발생하는 문제 검증
  기술적 연결의 정확성 검증

통합 테스트의 종류 (빠름 → 느림):
  슬라이스: @DataJpaTest, @WebMvcTest
  전체: @SpringBootTest
  완전: E2E (Testcontainers + 실제 인프라)

비율 기준 (테스트 피라미드):
  단위 테스트: 70% — 빠르고 많이
  통합 테스트: 20% — 경계 중심으로
  E2E 테스트:  10% — 핵심 시나리오만

판단 기준:
  "Mock으로 재현할 수 없는 동작인가?"
  → YES: 통합 테스트
  → NO: 단위 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 항목들 각각이 단위 테스트와 통합 테스트 중 무엇으로 검증해야 하는지 판단하고, 이유를 설명하라.

```
① VIP 회원에게 10% 할인이 적용되는가
② findByUserIdAndStatus() 쿼리가 올바른 SQL을 생성하는가
③ 주문 생성 시 재고가 충분하지 않으면 예외가 발생하는가
④ POST /orders 요청의 totalPrice 필드가 음수면 400을 반환하는가
⑤ 주문 저장 시 created_at이 자동으로 채워지는가
⑥ 동시에 같은 상품을 주문할 때 재고가 음수가 되지 않는가
```

**Q2.** 팀에서 "통합 테스트가 너무 느리니까 @SpringBootTest를 최소화하자"는 방향을 정했다. 어떤 기준으로 @SpringBootTest와 슬라이스 테스트를 나누는가?

**Q3.** 단위 테스트 커버리지는 90%인데, 배포 후 매번 장애가 발생한다. 어떤 종류의 통합 테스트를 추가해야 하는가? 장애의 원인이 어느 경계에 있을 가능성이 높은지 분석하라.

> 💡 **해설**

**Q1.**

① VIP 할인 10% → **단위 테스트.** 비즈니스 규칙이다. Mock/Stub으로 빠르게 검증 가능.

② JPA 쿼리 → **통합 테스트** (`@DataJpaTest`). JPA 쿼리 메서드 이름이 올바른 SQL을 생성하는지는 실제 JPA 컨텍스트가 필요하다.

③ 재고 부족 예외 → **단위 테스트.** 비즈니스 규칙이다. `InventoryPort`를 Stub으로 "재고 없음" 시나리오를 만들 수 있다.

④ 400 응답 검증 → **슬라이스 테스트** (`@WebMvcTest`). HTTP 레이어의 유효성 검증(`@Valid`, `@NotNull`)이 올바르게 동작하는지는 Spring MVC 컨텍스트가 필요하다.

⑤ `created_at` 자동 채워짐 → **통합 테스트** (`@DataJpaTest`). `@CreatedDate`, `@EnableJpaAuditing` 설정이 실제로 동작하는지는 DB와 JPA가 필요하다.

⑥ 동시 재고 → **통합 테스트** (실제 DB + 트랜잭션). DB 수준의 락(`SELECT FOR UPDATE`, `낙관적 락`)이 실제로 동작하는지는 실제 DB가 필요하다. H2보다 실제 DB(Testcontainers)가 더 신뢰도 높다.

**Q2.**

`@WebMvcTest` 선택: Controller 레이어만 검증할 때. 요청 역직렬화, 응답 직렬화, `@Valid` 검증, Security 인가 규칙.

`@DataJpaTest` 선택: Repository 레이어만 검증할 때. JPA 쿼리, DB 제약조건, Auditing.

`@SpringBootTest` 선택: ① 여러 레이어를 가로지르는 흐름 검증(Controller → Service → Repository 전체). ② Spring의 특정 기능이 실제 환경에서 동작하는지 (AOP, 트랜잭션 전파, 이벤트 리스너). ③ 외부 시스템 연동 (Kafka Consumer, Scheduler). 규칙: 한 레이어만 검증하면 슬라이스, 레이어를 가로질러야 하면 `@SpringBootTest`.

**Q3.**

커버리지 90%에도 장애가 반복된다면 경계 문제일 가능성이 높다. 점검 순서:

1. **DB 경계** — JPA 쿼리가 실제로 올바른 결과를 반환하는가? (`@DataJpaTest` + Testcontainers)
2. **트랜잭션 경계** — `@Transactional` 롤백/커밋이 의도대로인가? 실제 커밋 후 데이터를 확인하는 테스트
3. **직렬화 경계** — API 요청/응답 JSON 형식이 클라이언트와 일치하는가? (`@WebMvcTest` + 실제 JSON 검증)
4. **외부 API 경계** — 외부 시스템이 바뀌었을 때 감지하는 Contract 테스트가 있는가?

단위 테스트 커버리지는 높지만 이런 경계 테스트가 없으면 각 부품은 작동하는데 연결이 실패하는 패턴이 반복된다.

---

<div align="center">

**[다음: Database Testing with Testcontainers ➡️](./02-database-testing-testcontainers.md)**

</div>
