# 03. REST API Testing

> **MockMvc, WebTestClient, RestAssured — 무엇을 골라도 동작하지만 목적이 다르다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- MockMvc, WebTestClient, RestAssured의 차이는 무엇인가?
- 어떤 상황에서 어떤 도구를 선택하는가?
- REST API 테스트에서 무엇을 검증해야 하는가?

---

## 🔍 세 도구의 포지션

```
MockMvc
  Spring MVC 테스트 전용
  실제 HTTP 서버 없이 DispatcherServlet 직접 호출
  @WebMvcTest, @SpringBootTest 모두 사용 가능
  Spring 생태계에 가장 밀착

WebTestClient
  Spring WebFlux 공식 테스트 클라이언트
  Reactive 스택 전용 (WebFlux)
  블로킹/논블로킹 모두 지원 (Spring Boot 2.7+)
  실제 서버 또는 Mock 서버 모두 가능

RestAssured
  HTTP 클라이언트 독립적 테스트 라이브러리
  실제 HTTP 서버가 필요 (랜덤 포트)
  BDD 스타일 (given/when/then)
  Spring 외 다른 프레임워크에서도 사용 가능
```

---

## 🏛️ MockMvc — Spring MVC의 표준

### 기본 설정

```java
// @WebMvcTest: Service를 Mock으로 교체, Web 레이어만 로딩
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;

    @Test
    void 주문_생성_성공() throws Exception {
        Order mockOrder = anOrder().withId(1L).withPrice(18_000).build();
        when(orderService.place(any())).thenReturn(mockOrder);

        mockMvc.perform(
                post("/orders")
                    .contentType(APPLICATION_JSON)
                    .content("""
                        {
                            "userId": 1,
                            "totalPrice": 20000
                        }
                        """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.totalPrice").value(18_000))
            .andDo(print());  // 요청/응답 콘솔 출력
    }

    @Test
    void 필수_필드_누락_시_400_반환() throws Exception {
        mockMvc.perform(
                post("/orders")
                    .contentType(APPLICATION_JSON)
                    .content("{}"))  // userId, totalPrice 없음
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
}
```

### @SpringBootTest와 함께 — 전체 컨텍스트에서 MockMvc

```java
// @SpringBootTest + MockMvc: 전체 컨텍스트 + HTTP 없이 테스트
@SpringBootTest
@AutoConfigureMockMvc
class OrderApiIntegrationTest {

    @Autowired MockMvc mockMvc;
    @Autowired OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    void 주문_생성_후_조회() throws Exception {
        // 생성
        MvcResult createResult = mockMvc.perform(
                post("/orders")
                    .contentType(APPLICATION_JSON)
                    .content("""{"userId": 1, "totalPrice": 20000}"""))
            .andExpect(status().isCreated())
            .andReturn();

        // 응답에서 ID 추출
        String responseBody = createResult.getResponse().getContentAsString();
        Long orderId = JsonPath.read(responseBody, "$.id");

        // 조회
        mockMvc.perform(get("/orders/{id}", orderId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.totalPrice").value(20_000));
    }
}
```

---

## 🏛️ WebTestClient — Reactive 스택

```java
// WebFlux 애플리케이션 또는 Spring Boot 2.7+ MVC에서도 사용 가능
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderWebClientTest {

    @Autowired WebTestClient webTestClient;

    @Test
    void 주문_생성_성공() {
        webTestClient
            .post().uri("/orders")
            .contentType(APPLICATION_JSON)
            .bodyValue(Map.of("userId", 1, "totalPrice", 20_000))
            .exchange()
            .expectStatus().isCreated()
            .expectBody()
                .jsonPath("$.id").isNotEmpty()
                .jsonPath("$.totalPrice").isEqualTo(18_000);
    }

    @Test
    void 스트리밍_응답_검증() {
        // WebFlux의 Server-Sent Events 테스트
        webTestClient
            .get().uri("/orders/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(OrderEvent.class)
            .hasSize(3);
    }
}
```

---

## 🏛️ RestAssured — BDD 스타일, 실제 HTTP 서버

```java
// RestAssured는 실제 HTTP 서버가 필요 → RANDOM_PORT
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderRestAssuredTest {

    @LocalServerPort int port;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }

    @Test
    void 주문_생성_성공() {
        given()
            .contentType(ContentType.JSON)
            .body(Map.of("userId", 1, "totalPrice", 20_000))
        .when()
            .post("/orders")
        .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("totalPrice", equalTo(18_000));
    }

    @Test
    void 인증_없이_접근_시_401() {
        given()
            .contentType(ContentType.JSON)
        .when()
            .get("/orders/protected")
        .then()
            .statusCode(401);
    }

    @Test
    void 복잡한_응답_검증() {
        given()
            .param("status", "PENDING")
            .param("page", 0)
            .param("size", 10)
        .when()
            .get("/orders")
        .then()
            .statusCode(200)
            .body("content.size()", equalTo(10))
            .body("content[0].status", equalTo("PENDING"))
            .body("totalElements", greaterThan(0));
    }
}
```

---

## ✨ 선택 기준

```
MockMvc 선택:
  Spring MVC 애플리케이션
  @WebMvcTest에서 Controller 단독 검증
  실제 HTTP 서버 기동 없이 빠르게 실행
  Spring 생태계 도구와 자연스러운 통합

WebTestClient 선택:
  Spring WebFlux 애플리케이션
  Reactive 타입(Mono, Flux) 응답 검증
  SSE(Server-Sent Events) 테스트

RestAssured 선택:
  BDD(given/when/then) 스타일을 팀이 선호
  Spring 외 프레임워크와 공통 테스트 스타일
  복잡한 JSON 응답 구조 검증 (Hamcrest 매처)
  실제 HTTP 레이어 전체를 검증해야 할 때
```

---

## 💻 실전 적용: 무엇을 검증해야 하는가

### HTTP 상태 코드

```java
// 상황별 올바른 상태 코드
.andExpect(status().isOk())          // 200: 조회 성공
.andExpect(status().isCreated())     // 201: 생성 성공
.andExpect(status().isNoContent())   // 204: 삭제 성공
.andExpect(status().isBadRequest())  // 400: 요청 오류 (유효성 검증 실패)
.andExpect(status().isUnauthorized()) // 401: 인증 없음
.andExpect(status().isForbidden())   // 403: 권한 없음
.andExpect(status().isNotFound())    // 404: 리소스 없음
.andExpect(status().isConflict())    // 409: 충돌 (중복)
```

### JSON 응답 검증

```java
// jsonPath로 응답 구조 검증
.andExpect(jsonPath("$.id").isNotEmpty())
.andExpect(jsonPath("$.status").value("PENDING"))
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.items.length()").value(2))
.andExpect(jsonPath("$.items[0].productId").value(100))
.andExpect(jsonPath("$.createdAt").exists())

// 페이징 응답
.andExpect(jsonPath("$.content").isArray())
.andExpect(jsonPath("$.totalElements").value(25))
.andExpect(jsonPath("$.totalPages").value(3))
.andExpect(jsonPath("$.last").value(false))
```

### Content-Type 헤더

```java
.andExpect(content().contentType(APPLICATION_JSON))
.andExpect(header().string("Location", containsString("/orders/")))
```

### 에러 응답 형식

```java
// 유효성 검증 실패 시 에러 형식 검증
mockMvc.perform(post("/orders")
        .contentType(APPLICATION_JSON)
        .content("""{"totalPrice": -1}"""))
    .andExpect(status().isBadRequest())
    .andExpect(jsonPath("$.errors[*].field",
        hasItems("userId", "totalPrice")))
    .andExpect(jsonPath("$.errors[*].message").isNotEmpty());
```

---

## 🤔 트레이드오프

### "MockMvc와 RestAssured 중 무엇이 더 나은가?"

둘 중 하나가 절대적으로 나은 것은 없습니다.

MockMvc는 Spring 내부를 직접 호출하므로 실제 HTTP 서버 기동 없이 빠릅니다. `@WebMvcTest`와 자연스럽게 조합됩니다. 하지만 HTTP 레이어 전체(서블릿 컨테이너, 필터 체인)를 완전히 거치지 않습니다.

RestAssured는 실제 HTTP 요청을 보내므로 서블릿 컨테이너, 필터, 인터셉터 전체를 거칩니다. 하지만 서버 기동이 필요하고 포트 설정이 필요합니다.

팀에서 한 가지를 선택해서 일관되게 쓰는 것이 중요합니다.

### "jsonPath 대신 응답을 DTO로 역직렬화해서 검증하면 안 되는가?"

가능합니다. 응답을 DTO로 역직렬화하면 타입 안전성이 높아지고 필드 이름 오타가 컴파일 타임에 잡힙니다.

```java
String body = result.getResponse().getContentAsString();
OrderResponse response = objectMapper.readValue(body, OrderResponse.class);
assertThat(response.totalPrice()).isEqualTo(18_000);
```

다만 `jsonPath`는 응답 구조가 DTO 정의와 다를 때 더 유연합니다.

---

## 📌 핵심 정리

```
MockMvc:
  Spring MVC 전용
  실제 서버 없이 빠름
  @WebMvcTest와 조합
  Controller 단독 or 전체 컨텍스트 모두 가능

WebTestClient:
  WebFlux 전용 (or Spring Boot 2.7+ MVC)
  Reactive 타입 검증
  SSE, Streaming 테스트

RestAssured:
  BDD 스타일 (given/when/then)
  실제 HTTP 서버 필요 (RANDOM_PORT)
  프레임워크 독립적
  복잡한 JSON 검증에 강함

검증 대상:
  HTTP 상태 코드
  응답 JSON 구조와 값
  Content-Type 헤더
  에러 응답 형식
  페이징 메타데이터

선택 기준:
  Spring MVC + 빠른 피드백 → MockMvc
  WebFlux → WebTestClient
  BDD 스타일 + 실제 HTTP → RestAssured
```

---

## 🤔 생각해볼 문제

**Q1.** `@WebMvcTest`와 `@SpringBootTest(webEnvironment = RANDOM_PORT)`의 차이는 무엇인가? 각각 어떤 상황에서 선택하는가? MockMvc는 두 경우 모두 사용할 수 있는가?

**Q2.** 아래 테스트에서 검증하지 않은 것들을 찾고, 추가해야 하는 이유를 설명하라.

```java
@Test
void 주문_생성() throws Exception {
    mockMvc.perform(post("/orders")
            .contentType(APPLICATION_JSON)
            .content("""{"userId": 1, "totalPrice": 20000}"""))
        .andExpect(status().isOk());
        // 이것만 검증
}
```

**Q3.** Spring Security가 적용된 API를 `@WebMvcTest`로 테스트하면 인증 설정이 개입한다. 이를 처리하는 방법 두 가지를 설명하라.

> 💡 **해설**

**Q1.**

`@WebMvcTest`: Spring MVC 레이어만 로딩 (Controller, Filter, Interceptor, Security). Service, Repository는 `@MockBean`으로 교체. 실제 HTTP 서버 없이 `DispatcherServlet`을 직접 호출. 빠름. 선택 기준: Controller의 HTTP 처리 로직만 검증할 때 (요청 파싱, 응답 직렬화, Security 설정).

`@SpringBootTest(webEnvironment = RANDOM_PORT)`: 전체 Spring 컨텍스트 + 실제 HTTP 서버 기동. 실제 포트로 HTTP 요청. MockMvc도 사용할 수 있지만 RestAssured나 WebTestClient가 더 자연스럽다. 선택 기준: Controller → Service → Repository 전체 흐름을 검증하거나, 실제 HTTP 레이어 전체가 관여하는 테스트.

MockMvc는 `@WebMvcTest`에서는 자동으로, `@SpringBootTest`에서는 `@AutoConfigureMockMvc`를 추가하면 사용할 수 있다.

**Q2.**

누락된 검증들:

① **상태 코드**: `isOk()` (200)인데 생성은 `isCreated()` (201)이 올바르다. 시멘틱이 틀림.

② **응답 본문**: 생성된 주문의 `id`, `status`, `totalPrice` 등을 검증하지 않는다. 응답이 올바른 데이터를 포함하는지 알 수 없다.

③ **Location 헤더**: REST 관례상 `POST` 생성 성공 시 `Location: /orders/{id}` 헤더가 있어야 한다.

④ **Content-Type**: 응답이 `application/json`인지 검증 없음.

추가 검증:

```java
.andExpect(status().isCreated())
.andExpect(header().string("Location", matchesPattern("/orders/\\d+")))
.andExpect(content().contentType(APPLICATION_JSON))
.andExpect(jsonPath("$.id").isNotEmpty())
.andExpect(jsonPath("$.status").value("PENDING"))
```

**Q3.**

① **`@WithMockUser`** — Spring Security Test 지원. 특정 권한을 가진 가상 사용자로 요청.

```java
@Test
@WithMockUser(roles = "USER")
void 인증된_사용자는_주문_목록_조회_가능() throws Exception {
    mockMvc.perform(get("/orders"))
        .andExpect(status().isOk());
}

@Test
void 인증_없이_접근_시_401() throws Exception {
    mockMvc.perform(get("/orders"))
        .andExpect(status().isUnauthorized());
}
```

② **Security 설정 비활성화** — Controller 로직에 집중하고 Security는 별도 테스트할 때.

```java
@WebMvcTest(OrderController.class)
@Import(SecurityConfig.class)  // 실제 Security 설정 사용
// 또는
@WebMvcTest(excludeAutoConfiguration = SecurityAutoConfiguration.class)
// Security 자체를 제외
```

두 방식의 차이: `@WithMockUser`는 Security 설정을 유지하면서 인증만 우회. Security 비활성화는 Security 설정 자체를 테스트에서 제외. 보안 인가 로직을 검증하려면 `@WithMockUser`가 적합.

---

<div align="center">

**[⬅️ 이전: Database Testing with Testcontainers](./02-database-testing-testcontainers.md)** | **[다음: Spring Context Slicing ➡️](./04-spring-context-slicing.md)**

</div>
