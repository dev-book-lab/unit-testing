# 03. Flickering Tests

> **간헐적으로 실패하는 테스트는 통과하는 테스트보다 나쁘다 — 팀이 테스트 결과를 무시하기 시작하면 테스트 스위트 전체의 신뢰가 무너진다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Flaky 테스트의 4가지 주요 원인은 무엇인가?
- 각 원인별 해결 전략은 무엇인가?
- Flaky 테스트를 발견했을 때 어떻게 대응해야 하는가?

---

## 🔍 Flaky 테스트가 위험한 이유

```
Flaky 테스트의 악순환:

테스트 간헐적 실패
  → "다시 실행하면 통과하겠지" → CI 재실행
  → 팀이 실패를 무시하는 습관 생성
  → 실제 버그로 인한 실패도 재실행으로 처리
  → 테스트 결과에 대한 신뢰 상실
  → 테스트 스위트가 있어도 배포가 두려운 상태
```

Flaky 테스트 하나가 팀 전체의 테스트 신뢰를 갉아먹습니다.

---

## 🏛️ 원인 1: 시간 의존성

```java
// ❌ 테스트 실행 시각에 따라 결과가 달라진다
@Test
void 오늘_생성된_주문_조회() {
    Order order = orderRepository.save(
        anOrder().withCreatedAt(LocalDate.now()).build()
    );

    List<Order> todayOrders = orderService.findTodayOrders();
    assertThat(todayOrders).contains(order);
}
// 자정 직전에 실행: 저장은 2024-06-01, 조회는 2024-06-02 → 실패
```

```java
// ❌ 만료 시간 계산이 실행 속도에 의존
@Test
void 30분_후_쿠폰_만료() {
    Coupon coupon = new Coupon(LocalDateTime.now().plusMinutes(30));

    // 테스트 실행에 31분이 걸리면?
    assertThat(coupon.isExpired()).isFalse();
}
```

**해결:** Clock 주입으로 현재 시간을 고정합니다.

```java
// ✅ Clock을 주입해서 시간 고정
@Test
void 오늘_생성된_주문_조회() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-06-01T10:00:00Z"), ZoneOffset.UTC);
    OrderService service = new OrderService(fixedClock, orderRepository);

    Order order = orderRepository.save(
        anOrder().withCreatedAt(LocalDate.now(fixedClock)).build()
    );

    List<Order> todayOrders = service.findTodayOrders();
    assertThat(todayOrders).contains(order);
}

// ✅ 30분 후 만료 — 기준 시간을 고정
@Test
void 30분_후_쿠폰_만료() {
    Instant base = Instant.parse("2024-06-01T10:00:00Z");
    Clock baseClock = Clock.fixed(base, ZoneOffset.UTC);

    Coupon coupon = new Coupon(LocalDateTime.now(baseClock).plusMinutes(30));

    // 29분 후 시점으로 고정
    Clock laterClock = Clock.fixed(base.plusSeconds(29 * 60), ZoneOffset.UTC);
    assertThat(coupon.isExpired(laterClock)).isFalse();

    // 31분 후 시점으로 고정
    Clock expiredClock = Clock.fixed(base.plusSeconds(31 * 60), ZoneOffset.UTC);
    assertThat(coupon.isExpired(expiredClock)).isTrue();
}
```

---

## 🏛️ 원인 2: 테스트 간 공유 상태

```java
// ❌ static 변수 공유 — 테스트 실행 순서에 의존
class OrderServiceTest {

    static InMemoryOrderRepository sharedRepo = new InMemoryOrderRepository();
    static OrderService service = new OrderService(sharedRepo);

    @Test
    void 주문_저장() {
        service.place(aCommand().build());
        assertThat(sharedRepo.count()).isEqualTo(1); // 다른 테스트가 먼저 실행되면 실패
    }

    @Test
    void 빈_저장소_확인() {
        assertThat(sharedRepo.count()).isEqualTo(0); // 다른 테스트가 먼저 실행되면 실패
    }
}
```

```java
// ❌ Spring 컨텍스트에서 Bean이 상태를 가짐
@SpringBootTest
class UserServiceTest {

    @Autowired UserService userService; // 상태를 가진 싱글톤 빈

    @Test
    void 이메일_업데이트() {
        userService.updateEmail(1L, "new@example.com");
        assertThat(userService.findById(1L).email()).isEqualTo("new@example.com");
    }

    @Test
    void 기본_이메일_확인() {
        // 이전 테스트가 먼저 실행되면 이미 email이 변경되어 있음
        assertThat(userService.findById(1L).email()).isEqualTo("original@example.com");
    }
}
```

**해결:** `@BeforeEach`에서 초기화하거나, 테스트마다 독립적인 객체를 생성합니다.

```java
// ✅ 테스트마다 독립적인 객체 생성
class OrderServiceTest {

    private InMemoryOrderRepository repo;
    private OrderService service;

    @BeforeEach
    void setUp() {
        repo = new InMemoryOrderRepository(); // 매 테스트마다 새 인스턴스
        service = new OrderService(repo);
    }

    @Test
    void 주문_저장() {
        service.place(aCommand().build());
        assertThat(repo.count()).isEqualTo(1); // 항상 0에서 시작
    }

    @Test
    void 빈_저장소_확인() {
        assertThat(repo.count()).isEqualTo(0); // 항상 0에서 시작
    }
}
```

```java
// ✅ DB를 쓰는 통합 테스트: @BeforeEach에서 정리
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired UserService userService;
    @Autowired UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll(); // 각 테스트 전 초기화
        userRepository.save(defaultUser()); // 기준 데이터 준비
    }

    @Test
    void 이메일_업데이트() {
        userService.updateEmail(1L, "new@example.com");
        assertThat(userService.findById(1L).email()).isEqualTo("new@example.com");
    }
}
```

---

## 🏛️ 원인 3: 비동기 타이밍

```java
// ❌ Thread.sleep으로 비동기 대기 — 이전 문서의 주제
@Test
void 이벤트_처리_완료() throws InterruptedException {
    eventPublisher.publish(new OrderPlaced(orderId));
    Thread.sleep(1000); // CI 서버 부하에 따라 간헐적 실패
    assertThat(eventLog.count()).isEqualTo(1);
}
```

**해결:** Awaitility 또는 동기 설계 (02 문서 참고)

```java
// ✅
await()
    .atMost(10, SECONDS)
    .untilAsserted(() ->
        assertThat(eventLog.count()).isEqualTo(1)
    );
```

---

## 🏛️ 원인 4: 외부 시스템 의존성

```java
// ❌ 실제 외부 API 호출 — 네트워크 상태에 따라 실패
@Test
void 환율_조회() {
    ExchangeRate rate = exchangeClient.getRate("USD", "KRW"); // 실제 HTTP 호출
    assertThat(rate.value()).isGreaterThan(0);
}

// ❌ 실제 SMTP 서버에 이메일 전송 — 서버 상태에 따라 실패
@Test
void 이메일_발송() {
    emailService.send("user@example.com", "테스트 메일");
    // 검증 불가능한 외부 부수효과
}
```

**해결:** WireMock으로 외부 API를 Mock HTTP 서버로 대체합니다.

```kotlin
// build.gradle.kts
testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock")
testImplementation("com.github.tomakehurst:wiremock-jre8:2.35.0")
```

```java
// ✅ WireMock으로 외부 API 대체
@SpringBootTest
@AutoConfigureWireMock(port = 0) // 랜덤 포트
class ExchangeRateClientTest {

    @Autowired ExchangeRateClient client;

    @Test
    void 환율_조회_성공() {
        // WireMock 스텁 설정
        stubFor(get(urlEqualTo("/rates/USD/KRW"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""{"currency": "USD", "rate": 1320.5}""")));

        ExchangeRate rate = client.getRate("USD", "KRW");

        assertThat(rate.value()).isEqualTo(1320.5);
    }

    @Test
    void 외부_API_장애_시_기본값_반환() {
        stubFor(get(urlEqualTo("/rates/USD/KRW"))
            .willReturn(aResponse().withStatus(503))); // 서비스 불가

        ExchangeRate rate = client.getRate("USD", "KRW");

        assertThat(rate).isEqualTo(ExchangeRate.FALLBACK); // 기본값
    }
}
```

---

## 💻 실전 적용: Flaky 테스트 발견 및 대응

### 발견 방법

```
① CI 히스토리에서 "Re-run" 버튼이 자주 눌린 테스트 추적
② 테스트 결과 리포트에서 "flaky" 태그 또는 재실행 기록 확인
③ 로컬에서 같은 테스트를 여러 번 실행해서 간헐성 확인

mvn test -Dsurefire.rerunFailingTestsCount=3
// 실패한 테스트를 자동으로 최대 3번 재실행
```

### 발견했을 때 대응 전략

```
즉각 격리:
  @Disabled("Flaky: 원인 추적 중 — 이슈 #123")
  원인을 찾을 때까지 CI에서 제외
  → 나머지 테스트의 신뢰 유지

원인 분류:
  시간 의존 → Clock 주입
  공유 상태 → @BeforeEach 초기화 또는 독립 객체 생성
  비동기    → Awaitility
  외부 시스템 → WireMock 또는 Testcontainers

수정 후:
  @Disabled 제거
  로컬에서 10회 이상 연속 실행해서 안정성 확인
```

### 테스트 순서 독립성 확인

```java
// JUnit 5: 무작위 실행 순서로 순서 의존성 발견
@TestMethodOrder(MethodOrderer.Random.class)
class OrderServiceTest { ... }
```

---

## 🤔 트레이드오프

### "Flaky 테스트를 @Disabled로 격리하면 버그를 놓치지 않는가?"

격리된 테스트는 버그를 잡지 못합니다. 하지만 Flaky 테스트가 CI에 있으면 팀이 모든 실패를 무시하게 됩니다. 더 큰 위험입니다. `@Disabled`로 격리 후 이슈를 남기고, 최우선으로 수정합니다. 일시적인 격리가 영구화되지 않도록 이슈 트래킹이 중요합니다.

### "외부 API 테스트는 WireMock이 아닌 실제 테스트 환경을 써야 하지 않는가?"

외부 API의 계약을 검증하는 통합 테스트는 실제 테스트 환경을 써야 합니다. 하지만 비즈니스 로직이 올바른 요청을 만드는지, 응답을 올바르게 처리하는지 검증하는 것은 WireMock으로 충분합니다. 두 계층을 분리합니다.

---

## 📌 핵심 정리

```
Flaky 테스트의 4가지 원인:

① 시간 의존성
  LocalDate.now(), System.currentTimeMillis()
  해결: Clock 주입 → 시간 고정

② 테스트 간 공유 상태
  static 필드, Spring 싱글톤 빈
  해결: @BeforeEach 초기화, 독립 객체 생성

③ 비동기 타이밍
  Thread.sleep()으로 대기
  해결: Awaitility, 동기 설계

④ 외부 시스템 의존성
  실제 API, SMTP, 외부 DB
  해결: WireMock, Testcontainers

발견 및 대응:
  즉각 @Disabled + 이슈 생성
  원인 분류 후 해당 전략 적용
  수정 후 10회 이상 연속 실행 확인

예방:
  @TestMethodOrder(MethodOrderer.Random.class)
  각 테스트는 독립적으로 실행 가능해야 함
  공유 상태 최소화
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트가 월요일에만 실패한다. 원인이 무엇이고 어떻게 수정하는가?

```java
@Test
void 주말_배송_불가_주문_검증() {
    Order order = anOrder()
        .withDeliveryDate(LocalDate.now().plusDays(2))
        .build();

    // 금요일 실행: 2일 후 = 일요일 → 검증 실패 예상
    // 월요일 실행: 2일 후 = 수요일 → 검증 통과 → 테스트 실패
    assertThatThrownBy(() -> orderValidator.validate(order))
        .isInstanceOf(WeekendDeliveryException.class);
}
```

**Q2.** 아래 테스트가 단독 실행 시 통과하지만, 전체 테스트 스위트에서 실행하면 간헐적으로 실패한다. 원인과 해결 방법을 설명하라.

```java
@SpringBootTest
class InventoryTest {

    @Autowired InventoryRepository inventoryRepository;

    @Test
    void 재고_차감() {
        inventoryRepository.save(Inventory.of("PRODUCT-1", 10));
        inventoryService.deduct("PRODUCT-1", 3);
        assertThat(inventoryRepository.findByProductId("PRODUCT-1").quantity()).isEqualTo(7);
    }

    @Test
    void 재고_0_이하_차감_불가() {
        // PRODUCT-1의 재고가 다른 테스트에 의해 변경되어 있을 수 있음
        assertThatThrownBy(() -> inventoryService.deduct("PRODUCT-1", 100))
            .isInstanceOf(OutOfStockException.class);
    }
}
```

**Q3.** WireMock을 사용할 때 외부 API가 실제로 변경됐을 경우 WireMock 스텁이 구버전 응답을 반환하게 된다. 이 문제를 어떻게 감지하는가?

> 💡 **해설**

**Q1.**

원인: `LocalDate.now().plusDays(2)`가 테스트 실행 날짜에 따라 달라진다. 금요일에 실행하면 배송일이 일요일(주말)이 되어 예외가 발생하고 테스트가 통과한다. 월요일에 실행하면 배송일이 수요일(평일)이 되어 예외가 발생하지 않아 테스트가 실패한다.

수정:

```java
@Test
void 주말_배송_불가_주문_검증() {
    // 특정 날짜(금요일)로 고정 — 2일 후가 일요일임이 보장
    Clock fridayClock = Clock.fixed(
        Instant.parse("2024-06-07T10:00:00Z"), // 2024-06-07은 금요일
        ZoneOffset.UTC
    );
    OrderValidator validator = new OrderValidator(fridayClock);

    Order order = anOrder()
        .withDeliveryDate(LocalDate.now(fridayClock).plusDays(2)) // 일요일
        .build();

    assertThatThrownBy(() -> validator.validate(order))
        .isInstanceOf(WeekendDeliveryException.class);
}
```

**Q2.**

원인: 두 테스트가 동일한 `PRODUCT-1` 재고 데이터를 공유한다. `재고_차감` 테스트가 먼저 실행되면 `PRODUCT-1`의 재고가 7로 변경된다. `재고_0_이하_차감_불가` 테스트에서 100을 차감하려 하면 7 < 100이므로 예외가 발생해서 통과한다. 하지만 반대 순서로 실행되거나, `PRODUCT-1`이 아예 없으면 다른 결과가 나온다.

수정:

```java
@SpringBootTest
class InventoryTest {

    @Autowired InventoryRepository inventoryRepository;

    @BeforeEach
    void setUp() {
        inventoryRepository.deleteAll();
    }

    @Test
    void 재고_차감() {
        inventoryRepository.save(Inventory.of("PRODUCT-A", 10));
        inventoryService.deduct("PRODUCT-A", 3);
        assertThat(inventoryRepository.findByProductId("PRODUCT-A").quantity()).isEqualTo(7);
    }

    @Test
    void 재고_0_이하_차감_불가() {
        inventoryRepository.save(Inventory.of("PRODUCT-B", 5)); // 독립적인 데이터
        assertThatThrownBy(() -> inventoryService.deduct("PRODUCT-B", 100))
            .isInstanceOf(OutOfStockException.class);
    }
}
```

① `@BeforeEach`에서 `deleteAll()`로 공유 상태 초기화. ② 각 테스트가 자신만의 고유한 제품 ID를 사용해서 서로 영향을 주지 않는다.

**Q3.**

WireMock 스텁이 구버전 응답을 반환하는 문제는 Contract Testing으로 감지한다. (06 문서 주제)

실용적인 대안들:

① **Provider 계약 검증 테스트**: 실제 외부 API(또는 테스트 환경)를 대상으로 WireMock 스텁과 동일한 요청/응답이 여전히 유효한지 주기적으로 검증하는 별도 테스트를 작성한다.

② **API 스냅샷 테스트**: 외부 API의 실제 응답을 스냅샷으로 저장하고, 응답이 변경되면 스냅샷 테스트가 실패하도록 한다.

③ **Pact**: Consumer(WireMock 스텁을 쓰는 쪽)가 계약을 정의하고, Provider가 계약을 검증하도록 Pact를 도입한다. Provider API가 변경되면 Pact 검증에서 실패해 알 수 있다.

근본적으로 WireMock은 "이 API가 이렇게 동작한다고 가정"하는 도구다. 그 가정이 현실과 다를 때를 감지하려면 가정을 검증하는 별도 메커니즘이 필요하다.

---

<div align="center">

**[⬅️ 이전: Sleepy Tests](./02-sleepy-tests.md)** | **[다음: Overspecified Tests ➡️](./04-overspecified-tests.md)**

</div>
