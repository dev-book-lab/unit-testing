# 04. Spring Context Slicing

> **전체 컨텍스트를 올리는 것은 무거운 짐을 지는 것과 같다 — 필요한 레이어만 로딩하면 테스트가 10배 빨라진다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`는 각각 무엇을 로딩하는가?
- 어떤 상황에서 어떤 어노테이션을 선택해야 하는가?
- 슬라이스 테스트에서 부족한 빈을 어떻게 추가하는가?

---

## 🔍 왜 컨텍스트 로딩 범위가 중요한가

```
@SpringBootTest 기동 시간 측정 예시:
  애플리케이션 빈 수: 200개
  컨텍스트 로딩: 8초
  테스트 메서드 실행: 0.1초

  전체 테스트 50개 × 8초 = 400초 (컨텍스트 캐싱 없을 때)
  컨텍스트 캐싱 적용 후: 8초 (최초) + 0.1초 × 50 = 13초

@WebMvcTest 기동 시간:
  로딩 빈 수: 20개
  컨텍스트 로딩: 0.5초
  전체 테스트 50개: 0.5초 + 0.1초 × 50 = 5.5초
```

컨텍스트를 좁게 가져갈수록 테스트가 빨라집니다. 그리고 무엇을 테스트하는지가 더 명확해집니다.

---

## 🏛️ 각 슬라이스 어노테이션의 범위

### @SpringBootTest — 전체 컨텍스트

```java
@SpringBootTest
class FullContextTest {
    // 모든 @Component, @Service, @Repository, @Controller 로딩
    // 실제 DB 연결, Kafka 연결 등 모든 인프라 설정
    // 가장 현실에 가깝지만 가장 느림
}
```

```
@SpringBootTest가 로딩하는 것:
  모든 Spring 빈
  DataSource, JPA, Kafka, Redis 등 인프라
  Security 설정
  AOP, 트랜잭션 관리
  외부 설정 (application.yml)
```

### @WebMvcTest — Web 레이어만

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    // Controller, ControllerAdvice, Filter, WebMvcConfigurer 로딩
    // Service, Repository는 로딩 안 됨 → @MockBean 필요
    // Security 설정 자동 적용
    // 실제 DB 없음
}
```

```
@WebMvcTest가 로딩하는 것:
  @Controller, @RestController
  @ControllerAdvice (예외 처리)
  @JsonComponent (직렬화 커스터마이징)
  Filter, WebSecurityConfigurer
  MockMvc 자동 설정

@WebMvcTest가 로딩하지 않는 것:
  @Service, @Component
  @Repository
  DataSource, JPA
```

### @DataJpaTest — JPA 레이어만

```java
@DataJpaTest
class OrderRepositoryTest {
    // @Entity, @Repository 로딩
    // JPA 설정, 트랜잭션 관리 로딩
    // H2 인메모리 DB 자동 구성 (기본값)
    // Service, Controller는 로딩 안 됨
    // 각 테스트는 기본적으로 @Transactional (롤백)
}
```

```
@DataJpaTest가 로딩하는 것:
  @Entity 클래스
  @Repository
  JPA 관련 설정
  TestEntityManager

@DataJpaTest가 로딩하지 않는 것:
  @Service, @Component
  @Controller
  Security 설정
```

### 그 외 슬라이스

```java
@JsonTest              // ObjectMapper, 직렬화/역직렬화만
@RestClientTest        // RestTemplate, WebClient 클라이언트만
@DataMongoTest         // MongoDB 관련만
@DataRedisTest         // Redis 관련만
@WebFluxTest           // WebFlux 레이어만
```

---

## 😱 흔한 실수: 슬라이스 테스트에서 빈을 못 찾는 문제

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    // ❌ OrderService는 @WebMvcTest 범위에 없음
    // → OrderService 빈을 찾지 못해 컨텍스트 로딩 실패
    @Autowired OrderService orderService; // NoSuchBeanDefinitionException
}
```

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean OrderService orderService; // ✅ Mock으로 대체

    @Autowired MockMvc mockMvc;

    @Test
    void 주문_생성() throws Exception {
        when(orderService.place(any())).thenReturn(anOrder().build());

        mockMvc.perform(post("/orders")
                .contentType(APPLICATION_JSON)
                .content("""{"userId": 1, "totalPrice": 20000}"""))
            .andExpect(status().isCreated());
    }
}
```

---

## ✨ 슬라이스 테스트에 추가 빈이 필요할 때

### @Import — 특정 빈 추가

```java
// 커스텀 Validator나 Converter가 필요한 경우
@WebMvcTest(OrderController.class)
@Import({OrderValidator.class, PriceConverter.class})
class OrderControllerTest {

    @MockBean OrderService orderService;
    @Autowired MockMvc mockMvc;

    @Test
    void 커스텀_검증이_동작한다() throws Exception {
        // OrderValidator가 실제로 동작
        mockMvc.perform(post("/orders")
                .contentType(APPLICATION_JSON)
                .content("""{"userId": 1, "totalPrice": -1}"""))
            .andExpect(status().isBadRequest());
    }
}
```

### @TestConfiguration — 테스트 전용 빈 설정

```java
@DataJpaTest
@Import(TestDatabaseConfig.class) // Testcontainers 설정
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void 상태별_조회() {
        // 실제 MySQL로 테스트
    }
}

@TestConfiguration(proxyBeanMethods = false)
class TestDatabaseConfig {
    @Bean
    @ServiceConnection
    MySQLContainer<?> mysqlContainer() {
        return new MySQLContainer<>("mysql:8.0");
    }
}
```

### @MockBean vs @SpyBean

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    // @MockBean: 완전한 Mock — 모든 메서드가 기본값 반환
    @MockBean OrderService orderService;

    // @SpyBean: 실제 빈 + 일부 메서드만 Stub
    // @SpyBean OrderService orderService; // Service가 컨텍스트에 있어야 함
}
```

---

## 💻 실전 적용: 레이어별 테스트 전략 정리

```java
// Controller 레이어 → @WebMvcTest
// 검증 대상: 요청 파싱, 응답 직렬화, 유효성 검증, 예외 처리, Security

@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockBean OrderService orderService;
    @Autowired MockMvc mockMvc;

    @Test
    @WithMockUser(roles = "USER")
    void VIP_아닌_사용자는_특가_주문_불가() throws Exception {
        mockMvc.perform(post("/orders/flash-sale")
                .contentType(APPLICATION_JSON)
                .content("""{"userId": 1}"""))
            .andExpect(status().isForbidden());
    }
}
```

```java
// Repository 레이어 → @DataJpaTest
// 검증 대상: JPA 쿼리, DB 제약조건, Auditing, 페이징

@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Import(TestDatabaseConfig.class)
class OrderRepositoryTest {
    @Autowired OrderRepository orderRepository;

    @Test
    void 사용자별_주문_수_집계() {
        orderRepository.saveAll(List.of(
            anOrder().withUserId(1L).build(),
            anOrder().withUserId(1L).build(),
            anOrder().withUserId(2L).build()
        ));

        Map<Long, Long> counts = orderRepository.countByUserId();
        assertThat(counts.get(1L)).isEqualTo(2);
        assertThat(counts.get(2L)).isEqualTo(1);
    }
}
```

```java
// 전체 흐름 → @SpringBootTest
// 검증 대상: 레이어 간 연동, 트랜잭션 전파, 이벤트 리스너, 실제 커밋

@SpringBootTest
@Testcontainers
class OrderFlowIntegrationTest {
    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_생성_후_재고_차감_확인() {
        orderService.place(aCommand().build());

        // 실제 커밋이 발생했는지 별도 트랜잭션으로 확인
        assertThat(orderRepository.count()).isEqualTo(1);
    }
}
```

---

## 🏛️ 컨텍스트 캐싱 — 같은 설정이면 재사용

Spring은 같은 컨텍스트 설정을 가진 테스트끼리 컨텍스트를 재사용합니다. 컨텍스트 캐싱을 최대화하면 전체 테스트 실행 시간이 크게 줄어듭니다.

```
컨텍스트 캐싱을 깨는 것들:
  @MockBean / @SpyBean 추가 (빈 조합이 달라짐)
  @ActiveProfiles 변경
  @TestPropertySource로 다른 설정 주입
  @DirtiesContext 사용

최적화 방법:
  @MockBean 설정을 테스트 클래스마다 다르게 하지 않기
  공통 @MockBean 설정을 부모 클래스로 추출
  @DirtiesContext 사용 최소화
```

```java
// 공통 부모 클래스로 @MockBean 일원화 → 컨텍스트 재사용
@WebMvcTest({OrderController.class, UserController.class})
abstract class ControllerTestBase {
    @MockBean OrderService orderService;
    @MockBean UserService userService;
    @Autowired MockMvc mockMvc;
}

class OrderControllerTest extends ControllerTestBase {
    @Test
    void 주문_생성() { ... }
}

class UserControllerTest extends ControllerTestBase {
    @Test
    void 사용자_조회() { ... }
}
```

---

## 🤔 트레이드오프

### "`@SpringBootTest`만 써도 되지 않는가? 어차피 실제 환경에 가까우니까"

빠른 피드백 사이클이 깨집니다. `@SpringBootTest`가 수 초 걸리면 TDD가 어렵습니다. 또한 전체 컨텍스트에서는 어떤 레이어가 문제인지 격리하기 어렵습니다. 슬라이스 테스트는 "이 문제는 Controller 레이어에 있다"를 명확히 해줍니다.

### "@WebMvcTest에서 @MockBean이 너무 많아진다"

Controller가 너무 많은 Service에 의존한다는 신호일 수 있습니다. 또는 공통 부모 클래스로 `@MockBean` 설정을 추출해서 컨텍스트 재사용과 설정 중복을 동시에 해결합니다.

---

## 📌 핵심 정리

```
@SpringBootTest:
  모든 빈 로딩, 느림
  레이어 간 흐름, 트랜잭션 전파, 이벤트 검증

@WebMvcTest:
  Web 레이어만 (Controller, Security, Filter)
  Service는 @MockBean으로 대체
  요청 파싱, 응답 직렬화, 인가 검증

@DataJpaTest:
  JPA 레이어만 (Entity, Repository)
  기본값: H2 인메모리 + @Transactional 롤백
  Testcontainers와 조합: @AutoConfigureTestDatabase(replace = NONE)

슬라이스에 빈 추가:
  @Import → 특정 빈 추가
  @MockBean → 의존 빈을 Mock으로 대체
  @TestConfiguration → 테스트 전용 빈 설정

컨텍스트 캐싱:
  같은 @MockBean 조합 = 컨텍스트 재사용
  부모 클래스 추출 → 재사용 극대화
  @DirtiesContext 최소화
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트에서 컨텍스트 로딩이 실패하는 이유는 무엇이고, 어떻게 수정하는가?

```java
@DataJpaTest
class OrderServiceTest {

    @Autowired OrderService orderService; // 실패

    @Test
    void 주문_생성() {
        orderService.place(aCommand().build());
    }
}
```

**Q2.** 같은 테스트 스위트에 아래 두 클래스가 있다. Spring이 컨텍스트를 몇 개 생성하는가? 어떻게 하면 하나로 줄일 수 있는가?

```java
// 클래스 A
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockBean OrderService orderService;
    @MockBean UserService userService;
}

// 클래스 B
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean OrderService orderService;
    @MockBean UserService userService;
}
```

**Q3.** `@DataJpaTest`는 기본적으로 각 테스트에 `@Transactional`이 적용되어 롤백된다. 하지만 특정 테스트에서 실제 커밋이 필요하다면 어떻게 하는가?

> 💡 **해설**

**Q1.**

`@DataJpaTest`는 JPA 레이어만 로딩한다. `@Service` 어노테이션이 붙은 `OrderService`는 로딩 대상이 아니므로 `NoSuchBeanDefinitionException`이 발생한다.

수정 방법:

```java
// ① 올바른 슬라이스로 교체
@SpringBootTest  // 전체 컨텍스트에서 Service 테스트
class OrderServiceTest { ... }

// ② 또는 Service를 단위 테스트로 전환 (더 권장)
class OrderServiceTest {
    // Spring 없이 직접 생성자 주입
    OrderRepository fakeRepo = new InMemoryOrderRepository();
    OrderService service = new OrderService(fakeRepo, ...);
}
```

`@DataJpaTest`는 Repository 테스트에, Service 통합 테스트는 `@SpringBootTest`에 할당하는 것이 올바른 역할 분리다.

**Q2.**

두 개의 컨텍스트가 생성된다. `@WebMvcTest`는 대상 Controller 클래스를 컨텍스트 캐시 키의 일부로 사용하기 때문에, `OrderController.class`와 `UserController.class`가 다르면 다른 컨텍스트다. `@MockBean` 조합이 같아도 무관하다.

하나로 줄이는 방법 — Controller 두 개를 같은 테스트에서 로딩:

```java
@WebMvcTest({OrderController.class, UserController.class})
abstract class ControllerTestBase {
    @MockBean OrderService orderService;
    @MockBean UserService userService;
    @Autowired MockMvc mockMvc;
}

class OrderControllerTest extends ControllerTestBase { ... }
class UserControllerTest extends ControllerTestBase { ... }
```

부모 클래스가 같은 `@WebMvcTest` 설정과 `@MockBean` 조합을 가지므로 컨텍스트가 하나로 공유된다.

**Q3.**

`@Transactional` 롤백을 비활성화하는 방법:

```java
@DataJpaTest
class OrderRepositoryCommitTest {

    @Autowired OrderRepository orderRepository;
    @Autowired TestEntityManager entityManager;

    @Test
    @Commit  // 롤백 대신 커밋
    void 저장_후_실제_커밋_확인() {
        Order saved = orderRepository.save(anOrder().build());
        entityManager.flush();
        entityManager.clear();

        // 실제 커밋된 데이터를 새 영속성 컨텍스트로 조회
        Order found = orderRepository.findById(saved.id()).orElseThrow();
        assertThat(found.status()).isEqualTo(PENDING);
    }

    @AfterEach
    void cleanUp() {
        orderRepository.deleteAll(); // @Commit 후 직접 정리
    }
}
```

또는 `@Transactional`을 아예 제거하고 `@BeforeEach`/`@AfterEach`로 수동 관리한다. `@Commit`을 쓸 경우 다음 테스트를 위해 `@AfterEach`에서 직접 정리해야 한다.

---

<div align="center">

**[⬅️ 이전: REST API Testing](./03-rest-api-testing.md)** | **[다음: Test Transaction Management ➡️](./05-test-transaction-management.md)**

</div>
