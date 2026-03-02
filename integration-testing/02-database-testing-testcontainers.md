# 02. Database Testing with Testcontainers

> **H2는 MySQL이 아니다 — 프로덕션 DB로 테스트하지 않으면 프로덕션에서 처음 문제를 발견한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- H2 인메모리 DB로 테스트하는 것의 한계는 무엇인가?
- Testcontainers는 어떻게 동작하고 어떻게 설정하는가?
- 컨테이너 기동 비용을 어떻게 줄이는가?

---

## 🔍 H2의 한계 — "테스트는 통과하는데 배포하면 터진다"

H2는 빠르고 설정이 간단합니다. 하지만 프로덕션에서 MySQL이나 PostgreSQL을 사용한다면, H2와의 차이가 런타임 오류로 나타납니다.

```sql
-- MySQL에서 동작하지만 H2에서 다르게 처리되는 예시들

-- 1. JSON 타입
ALTER TABLE orders ADD COLUMN metadata JSON;
-- H2: JSON 타입 미지원 → 테스트 실패 또는 VARCHAR로 처리

-- 2. MySQL 전용 함수
SELECT * FROM orders WHERE DATE_FORMAT(created_at, '%Y-%m') = '2024-06';
-- H2: DATE_FORMAT 함수 없음 → 테스트 실패

-- 3. Full-text search
SELECT * FROM products WHERE MATCH(name) AGAINST ('커피');
-- H2: Full-text search 미지원

-- 4. 격리 수준 차이
-- MySQL의 REPEATABLE READ와 H2의 기본 격리 수준 다름
-- 동시성 버그가 H2에서 재현 안 됨
```

```java
// JPA Named Query 오류도 H2에서는 안 잡힌다
@Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.status = :status")
List<Order> findByUserIdAndStatus(Long userId, String status);
// H2에서 통과했지만 MySQL에서 컬럼명 대소문자 차이로 실패
```

---

## 🏛️ Testcontainers — 실제 DB를 테스트에서 사용

Testcontainers는 Docker 컨테이너를 JUnit 테스트 생명주기와 연동해서 실행합니다. 테스트가 시작될 때 컨테이너를 기동하고, 테스트가 끝나면 자동으로 제거합니다.

### 의존성 설정

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.testcontainers:junit-jupiter:1.19.0")
    testImplementation("org.testcontainers:mysql:1.19.0")
    // 또는 PostgreSQL
    testImplementation("org.testcontainers:postgresql:1.19.0")
}
```

### 기본 설정 — @Testcontainers + @Container

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)  // H2 자동 대체 비활성화
@Testcontainers
class OrderRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired OrderRepository orderRepository;

    @Test
    void 상태별_주문_조회() {
        orderRepository.save(anOrder().withStatus(PENDING).build());
        orderRepository.save(anOrder().withStatus(COMPLETED).build());

        List<Order> pending = orderRepository.findByStatus(PENDING);

        assertThat(pending).hasSize(1);
        assertThat(pending.get(0).status()).isEqualTo(PENDING);
    }

    @Test
    void JSON_컬럼이_올바르게_저장된다() {
        Map<String, String> metadata = Map.of("source", "mobile", "version", "2.0");
        Order order = anOrder().withMetadata(metadata).build();

        Order saved = orderRepository.save(order);

        assertThat(saved.metadata()).containsEntry("source", "mobile");
    }
}
```

---

## 😱 테스트마다 컨테이너를 새로 기동하면 너무 느리다

```
@Container 위치에 따라 생명주기가 달라진다:

인스턴스 필드 (@Container):
  → 테스트 메서드마다 새 컨테이너 기동/종료
  → 완전한 격리 but 매우 느림

static 필드 (@Container static):
  → 테스트 클래스 전체에서 컨테이너 공유
  → 클래스 내 테스트들은 같은 컨테이너 사용
  → 적절한 속도와 격리의 균형
```

### 더 나아가: 테스트 스위트 전체에서 컨테이너 재사용

Spring Boot의 `@TestConfiguration`과 `static` 컨테이너를 조합하면 JVM 내 모든 테스트가 하나의 컨테이너를 공유합니다.

```java
// 공통 설정 클래스 — 모든 DB 통합 테스트가 상속
@TestConfiguration(proxyBeanMethods = false)
public class TestDatabaseConfig {

    @Bean
    @ServiceConnection  // Spring Boot 3.1+ — @DynamicPropertySource 불필요
    MySQLContainer<?> mysqlContainer() {
        return new MySQLContainer<>("mysql:8.0")
            .withReuse(true);  // 컨테이너 재사용 활성화
    }
}
```

```java
// Spring Boot 3.1+ — @ServiceConnection으로 자동 연결
@DataJpaTest
@Import(TestDatabaseConfig.class)
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_저장_후_조회() {
        Order saved = orderRepository.save(anOrder().build());
        assertThat(orderRepository.findById(saved.id())).isPresent();
    }
}
```

### withReuse(true) — JVM 재시작 없이 컨테이너 유지

```
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
```

`withReuse(true)` + 설정 파일이 있으면 같은 Docker 이미지/설정의 컨테이너를 JVM 재시작 후에도 재사용합니다. 개발 중 반복 테스트 실행 시 컨테이너 기동 비용이 거의 없어집니다.

---

## ✨ 데이터 격리 전략

여러 테스트가 같은 DB를 공유할 때 테스트 간 데이터 오염이 문제가 됩니다.

### 전략 1: @Transactional 롤백 — 가장 간단

```java
@DataJpaTest  // 기본적으로 @Transactional이 붙음
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void 주문_저장() {
        orderRepository.save(anOrder().build());
        assertThat(orderRepository.count()).isEqualTo(1);
    }
    // 테스트 메서드 종료 시 자동 롤백
    // 다음 테스트는 깨끗한 상태에서 시작

    @Test
    void 빈_저장소에서_조회() {
        assertThat(orderRepository.count()).isEqualTo(0); // 오염 없음
    }
}
```

**주의:** `@Transactional` 롤백은 실제 커밋을 테스트하지 못합니다. (→ 05 문서)

### 전략 2: @BeforeEach 에서 테이블 초기화

```java
@SpringBootTest
class OrderServiceIntegrationTest {

    @Autowired OrderRepository orderRepository;

    @BeforeEach
    void cleanUp() {
        orderRepository.deleteAll();  // 각 테스트 전 초기화
    }

    @Test
    void 주문_생성_후_이벤트_발행() {
        // 이 테스트는 실제 커밋이 발생
        // @Transactional 롤백 불가 (Kafka 이벤트 발행 등)
    }
}
```

### 전략 3: 고유한 테스트 데이터 식별자 사용

```java
@Test
void 특정_사용자_주문만_조회된다() {
    // 각 테스트마다 고유한 userId 사용 → 서로 오염 없음
    Long uniqueUserId = System.currentTimeMillis();
    orderRepository.save(anOrder().withUserId(uniqueUserId).build());

    List<Order> found = orderRepository.findByUserId(uniqueUserId);
    assertThat(found).hasSize(1);
}
```

---

## 💻 실전 적용: Flyway/Liquibase와 함께

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Testcontainers
class OrderSchemaTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb");

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");  // 마이그레이션 실행
    }

    @Autowired OrderRepository orderRepository;

    @Test
    void 마이그레이션_후_스키마가_올바르다() {
        // Flyway가 마이그레이션을 실행한 후
        // 실제 스키마로 JPA 동작 검증
        Order order = anOrder()
            .withMetadata(Map.of("channel", "web"))  // JSON 컬럼
            .build();

        Order saved = orderRepository.save(order);
        assertThat(saved.metadata()).containsKey("channel");
    }
}
```

---

## 🤔 트레이드오프

### "Testcontainers는 Docker가 필요한데 CI에서 설정이 복잡하지 않은가?"

대부분의 CI 환경(GitHub Actions, GitLab CI)은 Docker-in-Docker를 지원합니다. `testcontainers-cloud`를 사용하면 CI에서 컨테이너 기동 비용을 더 줄일 수 있습니다.

### "H2를 MySQL 모드로 실행하면 되지 않는가?"

```java
spring.datasource.url=jdbc:h2:mem:testdb;MODE=MySQL
```

MySQL 모드가 일부 호환성을 제공하지만, JSON 타입, 특정 함수, 인덱스 힌트, 격리 수준 등 완전한 호환이 아닙니다. "테스트는 통과하는데 prod에서 터진다"는 패턴이 여전히 남습니다.

---

## 📌 핵심 정리

```
H2의 한계:
  JSON 타입, DB 전용 함수 미지원
  격리 수준 차이 → 동시성 버그 재현 불가
  MySQL MODE도 완전한 호환 아님

Testcontainers의 동작:
  JUnit 생명주기와 연동
  실제 Docker 컨테이너로 DB 실행
  @Container static → 클래스 내 공유
  withReuse → JVM 재시작 후에도 재사용

Spring Boot 3.1+ 간편 설정:
  @ServiceConnection → @DynamicPropertySource 불필요

데이터 격리 전략:
  @Transactional 롤백 → 빠르고 간단 (커밋 검증 불가)
  @BeforeEach 초기화 → 실제 커밋 검증 가능
  고유 ID 사용 → 병렬 실행 가능

언제 Testcontainers:
  프로덕션 DB 전용 기능 사용 시
  마이그레이션 스크립트 검증
  동시성/락 동작 검증
  인덱스, 실행 계획 검증
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 Repository 메서드를 H2와 Testcontainers(MySQL)로 각각 테스트했을 때, 어느 쪽에서 문제가 발견될 가능성이 높은가? 이유를 설명하라.

```java
// 최근 30일 내 주문 + 특정 상태 + JSON 메타데이터 조건
@Query(value = """
    SELECT * FROM orders
    WHERE user_id = :userId
      AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
      AND status IN :statuses
      AND JSON_EXTRACT(metadata, '$.channel') = :channel
    """, nativeQuery = true)
List<Order> findRecentOrders(Long userId, List<String> statuses, String channel);
```

**Q2.** `withReuse(true)`를 설정하면 테스트 간 데이터 오염이 발생할 수 있다. 이를 방지하기 위한 전략을 두 가지 이상 설명하라.

**Q3.** `@DataJpaTest`는 기본적으로 H2 인메모리 DB를 사용한다. Testcontainers로 교체할 때 `@AutoConfigureTestDatabase(replace = NONE)`을 반드시 추가해야 하는 이유는 무엇인가? 없으면 어떻게 되는가?

> 💡 **해설**

**Q1.**

`DATE_SUB(NOW(), INTERVAL 30 DAY)` — MySQL 전용 함수. H2에서는 `DATEADD('DAY', -30, NOW())`를 써야 한다. nativeQuery라서 H2에서 실행하면 함수를 찾지 못해 테스트가 실패하거나, H2 MySQL 모드에서도 완벽히 지원되지 않을 수 있다.

`JSON_EXTRACT(metadata, '$.channel')` — MySQL의 JSON 함수. H2는 JSON 타입과 JSON 함수를 지원하지 않는다.

결론: 이 쿼리는 H2에서 아예 실행이 안 된다. Testcontainers(MySQL)에서만 올바르게 검증할 수 있다. 이것이 nativeQuery + MySQL 전용 기능을 쓸 때 Testcontainers가 필수인 이유다.

**Q2.**

① `@BeforeEach` 또는 `@AfterEach`에서 테이블 초기화 (`deleteAll()` 또는 `TRUNCATE`). 가장 확실하지만 테스트 순서에 의존.

② `@Transactional` 롤백 — `@DataJpaTest`에 기본 포함. 각 테스트가 트랜잭션 안에서 실행되고 종료 시 자동 롤백. `withReuse`와 조합해도 안전.

③ 고유 식별자 사용 — 각 테스트에서 `UUID.randomUUID()`나 `System.currentTimeMillis()`를 userId 등으로 사용. 테스트 간 데이터가 겹치지 않음. 병렬 실행에도 안전.

④ `@Sql(scripts = "classpath:cleanup.sql", executionPhase = AFTER_TEST_METHOD)` — 테스트 후 정리 SQL 실행.

**Q3.**

`@DataJpaTest`는 기본적으로 `@AutoConfigureTestDatabase`가 적용되어 있고, `replace = ANY`가 기본값이다. 이 설정은 `application.properties`의 datasource 설정을 무시하고 자동으로 내장 DB(H2)로 교체한다.

`replace = NONE`을 추가하지 않으면:

- `@DynamicPropertySource`로 Testcontainers URL을 설정해도 무시됨
- Spring이 H2를 자동으로 사용함
- Testcontainers 컨테이너는 기동되지만 실제로 연결되지 않음
- 테스트는 H2로 실행되어 Testcontainers를 쓰는 의미가 없음

따라서 Testcontainers(또는 외부 DB)를 쓸 때는 반드시 `replace = NONE`으로 자동 교체를 비활성화해야 한다.

---

<div align="center">

**[⬅️ 이전: Integration Test Scope](./01-integration-test-scope.md)** | **[다음: REST API Testing ➡️](./03-rest-api-testing.md)**

</div>
