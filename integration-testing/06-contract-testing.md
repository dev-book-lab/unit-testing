# 06. Contract Testing

> **서비스 A의 테스트가 통과하고 서비스 B의 테스트도 통과했는데 함께 통신하면 실패한다 — 계약이 맞지 않기 때문이다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Contract Testing이 해결하는 문제는 무엇인가?
- Consumer-Driven Contracts란 무엇이고 왜 이 방향인가?
- Spring Cloud Contract와 Pact는 어떻게 다른가?

---

## 🔍 MSA에서 생기는 계약 문제

```
주문 서비스 (Consumer)         재고 서비스 (Provider)
     │                               │
     │  GET /inventory/{productId}   │
     │ ──────────────────────────▶   │
     │                               │
     │  { "available": true,         │
     │    "quantity": 10 }           │
     │ ◀──────────────────────────   │
```

재고 서비스 개발자가 응답 형식을 변경합니다.

```json
// 변경 전
{ "available": true, "quantity": 10 }

// 변경 후 (필드명 변경)
{ "isAvailable": true, "stock": 10 }
```

재고 서비스의 모든 단위 테스트가 통과합니다. 주문 서비스의 모든 단위 테스트도 통과합니다. 두 서비스를 함께 배포했을 때 주문 서비스가 `available` 필드를 읽지 못해 장애가 발생합니다.

---

## 🏛️ Contract Testing이란

**계약(Contract):** Consumer(소비자)가 Provider(제공자)에게 기대하는 요청/응답의 명세

**Consumer-Driven Contracts:** Consumer가 계약을 정의하고, Provider는 그 계약을 만족하는지 검증한다.

```
흐름:
  ① Consumer: "나는 이런 요청을 보내고 이런 응답을 기대한다" → 계약 파일 생성
  ② 계약 파일을 Pact Broker 또는 저장소에 공유
  ③ Provider: 계약 파일을 읽어서 "이 계약을 내가 만족하는가?" → 검증
  ④ Provider 변경 시 계약 검증 실패 → 배포 전 발견
```

이것은 E2E 테스트처럼 두 서비스를 실제로 띄울 필요 없이, 각자의 테스트 스위트에서 계약을 검증합니다.

---

## 🏛️ Spring Cloud Contract — Provider 주도 방식

Spring Cloud Contract는 Provider가 계약을 정의하고, Consumer는 그 계약으로부터 생성된 Stub을 사용합니다.

### Provider 쪽 — 계약 정의

```groovy
// src/test/resources/contracts/shouldReturnInventory.groovy
Contract.make {
    description "재고 조회 성공"

    request {
        method GET()
        url "/inventory/100"
    }

    response {
        status OK()
        headers { contentType(applicationJson()) }
        body([
            available: true,
            quantity: 10
        ])
    }
}
```

### Provider 쪽 — 계약 검증 테스트 자동 생성

```java
// Spring Cloud Contract가 계약 파일로부터 테스트를 자동 생성
@SpringBootTest(webEnvironment = RANDOM_PORT)
public class InventoryContractVerificationTest extends ContractVerifierBase {
    // 계약에 정의된 요청을 Provider에 보내서 응답이 계약과 일치하는지 검증
    // 자동 생성된 테스트 — 직접 작성하지 않음
}
```

### Consumer 쪽 — 생성된 Stub 사용

```java
// Provider가 배포한 Stub을 사용해서 Consumer 테스트
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:inventory-service:+:stubs:8080",
    stubsMode = CLASSPATH
)
class OrderServiceContractTest {

    @Autowired OrderService orderService;

    @Test
    void 재고_조회_후_주문_생성() {
        // Stub 서버가 계약에 정의된 응답을 반환
        Order order = orderService.place(aCommand().withProductId(100L).build());
        assertThat(order).isNotNull();
    }
}
```

---

## 🏛️ Pact — Consumer 주도 방식

Pact는 Consumer가 계약을 먼저 정의하고 Provider가 이를 검증하는 방식으로, 진정한 Consumer-Driven Contracts를 구현합니다.

### Consumer 쪽 — 계약 정의 + Stub 생성

```java
// build.gradle.kts
testImplementation("au.com.dius.pact.consumer:junit5:4.6.0")

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "inventory-service")
class OrderServicePactTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact inventoryAvailable(PactDslWithProvider builder) {
        return builder
            .given("productId=100 재고 있음")  // Provider State
            .uponReceiving("productId=100 재고 조회")
                .method("GET")
                .path("/inventory/100")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .booleanValue("available", true)
                    .integerType("quantity", 10))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "inventoryAvailable")
    void 재고_있을_때_주문_성공(MockServer mockServer) {
        // Pact가 계약 기반 Mock 서버를 기동
        InventoryClient client = new InventoryClient(mockServer.getUrl());

        InventoryResponse response = client.check(100L);

        assertThat(response.available()).isTrue();
        assertThat(response.quantity()).isEqualTo(10);
    }
    // 테스트 종료 시 계약 파일(pact) 생성 → target/pacts/
}
```

### Consumer 쪽 — 재고 없는 경우 계약 추가

```java
@Pact(consumer = "order-service")
public RequestResponsePact inventoryUnavailable(PactDslWithProvider builder) {
    return builder
        .given("productId=200 재고 없음")
        .uponReceiving("productId=200 재고 조회")
            .method("GET")
            .path("/inventory/200")
        .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody()
                .booleanValue("available", false)
                .integerType("quantity", 0))
        .toPact();
}

@Test
@PactTestFor(pactMethod = "inventoryUnavailable")
void 재고_없을_때_주문_실패(MockServer mockServer) {
    InventoryClient client = new InventoryClient(mockServer.getUrl());

    assertThatThrownBy(() -> orderService.placeWithInventoryCheck(200L))
        .isInstanceOf(OutOfStockException.class);
}
```

### Provider 쪽 — 계약 검증

```java
// build.gradle.kts
testImplementation("au.com.dius.pact.provider:junit5spring:4.6.0")

@SpringBootTest(webEnvironment = RANDOM_PORT)
@Provider("inventory-service")
@PactFolder("src/test/resources/pacts")  // Consumer가 생성한 계약 파일
class InventoryServicePactVerificationTest {

    @LocalServerPort int port;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Provider State 설정 — Consumer가 정의한 전제 조건 구성
    @State("productId=100 재고 있음")
    void productAvailable() {
        inventoryRepository.save(
            Inventory.of(100L, 10, true)
        );
    }

    @State("productId=200 재고 없음")
    void productUnavailable() {
        inventoryRepository.save(
            Inventory.of(200L, 0, false)
        );
    }
}
```

---

## ✨ Pact Broker — 계약 파일 공유

팀 간 계약 파일을 공유하고 버전을 관리하는 중앙 저장소입니다.

```yaml
# docker-compose.yml
services:
  pact-broker:
    image: pactfoundation/pact-broker
    environment:
      PACT_BROKER_DATABASE_URL: postgres://user:pass@db/pact
    ports:
      - "9292:9292"
```

```groovy
// build.gradle — Consumer: 계약 파일 Broker에 업로드
pact {
    publish {
        pactBrokerUrl = "http://localhost:9292"
        version = project.version
    }
}

// build.gradle — Provider: Broker에서 계약 파일 읽어 검증
pact {
    serviceProviders {
        "inventory-service" {
            hasPactsFromPactBroker("http://localhost:9292")
        }
    }
}
```

```
CI 파이프라인에서의 흐름:

Consumer PR 머지
  → Consumer 테스트 실행 → 계약 파일 생성
  → Pact Broker에 계약 업로드

Provider PR 배포 전
  → Pact Broker에서 계약 파일 다운로드
  → Provider 계약 검증 실행
  → 검증 실패 → 배포 차단
```

---

## 💻 Spring Cloud Contract vs Pact 비교

```
Spring Cloud Contract:
  Provider가 계약 정의 주도
  Spring 생태계에 밀착
  JVM 기반 프로젝트에 적합
  WireMock 기반 Stub 자동 생성

Pact:
  Consumer가 계약 정의 주도
  다양한 언어 지원 (Java, JS, Python, Go...)
  Pact Broker로 계약 파일 중앙 관리
  MSA에서 서비스가 다른 언어를 쓸 때 강점

선택 기준:
  Spring Boot + 같은 팀 관리 → Spring Cloud Contract
  다른 언어 서비스 간 계약 → Pact
  Consumer가 계약 주도권을 가져야 함 → Pact
  Provider가 여러 Consumer를 갖고 있음 → Pact Broker
```

---

## 🤔 트레이드오프

### "E2E 테스트가 있으면 Contract Testing이 필요한가?"

E2E 테스트는 특정 시나리오를 검증하지만, API 형식 변경이라는 세부 계약을 체계적으로 추적하지 못합니다. Contract Testing은 각 서비스가 독립적으로 배포될 때 계약을 사전에 검증합니다. 두 방식은 보완적입니다.

### "계약 파일 관리가 번거롭지 않은가?"

Pact Broker가 버전 관리와 공유를 담당합니다. CI 파이프라인에 통합하면 배포 전 자동으로 검증됩니다. 초기 설정 비용은 있지만, 서비스 간 계약 위반으로 생기는 장애 비용이 훨씬 큽니다.

---

## 📌 핵심 정리

```
Contract Testing이 해결하는 문제:
  각 서비스의 단위 테스트는 통과하는데
  API 형식 불일치로 통합 시 실패하는 문제

Consumer-Driven Contracts:
  Consumer가 기대하는 계약을 정의
  Provider가 계약을 만족하는지 검증
  배포 전에 호환성 보장

Spring Cloud Contract:
  Provider 주도, Spring 생태계
  계약 → Provider 검증 테스트 자동 생성
  계약 → Consumer용 Stub 자동 생성

Pact:
  Consumer 주도, 다언어 지원
  Consumer: 계약 파일(.json) 생성
  Provider: 계약 파일 읽어서 @State 설정 후 검증
  Pact Broker: 중앙 계약 파일 저장소

CI 통합:
  Consumer 변경 → 계약 업데이트 → Broker 업로드
  Provider 배포 전 → 계약 검증 → 실패 시 배포 차단
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 서비스(Consumer)가 회원 서비스(Provider)의 `/users/{id}` API를 사용한다. Provider가 응답에서 `email` 필드를 `emailAddress`로 이름을 바꿨다. Contract Testing이 있었다면 어느 시점에 이 문제를 발견할 수 있는가? Contract Testing이 없었다면 언제 발견하는가?

**Q2.** Pact에서 `@State("productId=100 재고 있음")`의 역할은 무엇인가? Provider State가 없으면 어떤 문제가 생기는가?

**Q3.** 같은 Provider에 Consumer가 3개(`order-service`, `cart-service`, `report-service`) 있다. 각 Consumer가 서로 다른 필드를 요구한다. Pact Broker에서 이 상황을 어떻게 관리하는가? Provider는 3개의 계약을 모두 만족해야 하는가?

> 💡 **해설**

**Q1.**

Contract Testing이 있는 경우:

Provider(회원 서비스) 개발자가 `email` → `emailAddress`로 변경하고 배포 파이프라인을 실행한다. Pact Broker에서 Consumer(주문 서비스)가 정의한 계약 파일을 읽는다. 계약에는 `email` 필드가 있고, Provider의 실제 응답에는 `emailAddress`가 있다. **Provider 계약 검증 테스트 실패 → 배포 파이프라인 차단.** 배포 전에 발견한다.

Contract Testing이 없는 경우:

회원 서비스가 변경 후 배포 → 주문 서비스가 `email` 필드를 읽지 못해 NullPointerException → **프로덕션 장애로 발견.** 최악의 경우 야간 배포 후 새벽에 발견한다.

**Q2.**

`@State`는 Provider가 특정 계약 시나리오를 검증하기 전에 필요한 데이터/상태를 구성하는 설정이다. Consumer는 "productId=100 재고 있음"이라는 전제 조건을 가정하고 계약을 정의했다. Provider는 이 상태를 실제로 구성해야 계약 검증이 의미 있다.

Provider State가 없으면: `GET /inventory/100` 요청 시 DB에 데이터가 없어서 404를 반환하거나 다른 응답이 나온다. 계약에서는 `available: true`를 기대하는데 실제 응답과 다르다. 계약 검증이 **의미 없이 실패**하거나, 우연히 데이터가 있으면 **의미 없이 통과**한다.

**Q3.**

Pact Broker는 Consumer별로 계약 파일을 분리 관리한다. `order-service`, `cart-service`, `report-service`가 각각 자신의 계약 파일을 업로드한다.

Provider는 Broker에서 자신(inventory-service)을 Consumer로 하는 **모든 계약 파일을 다운로드**해서 검증한다. 3개 Consumer의 계약을 모두 만족해야 배포 가능하다.

```java
@PactBroker(host = "localhost", port = "9292")
class InventoryServicePactVerificationTest {
    // Broker에 등록된 모든 Consumer(order, cart, report)의
    // 계약을 자동으로 읽어서 각각 검증
}
```

이것이 Consumer-Driven Contracts의 핵심이다. Provider 혼자 계약을 정의하면 실제 Consumer의 요구사항이 반영되지 않을 수 있지만, 각 Consumer가 직접 계약을 정의하면 Provider는 모든 Consumer를 만족시켜야 한다.

---

<div align="center">

**[⬅️ 이전: Test Transaction Management](./05-test-transaction-management.md)** | **[홈으로 🏠](../README.md)**

</div>
