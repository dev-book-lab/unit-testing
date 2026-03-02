# 06. Hidden Test Dependencies

> **테스트 A를 먼저 실행해야 테스트 B가 통과한다 — 의존 관계가 숨겨진 순간 테스트 스위트는 시한폭탄이 된다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- 숨겨진 테스트 의존성의 세 가지 패턴은 무엇인가?
- 테스트 실행 순서 의존성은 어떻게 발생하고 어떻게 제거하는가?
- 공유 상태가 테스트에 어떤 식으로 개입하는가?

---

## 🔍 숨겨진 의존성이란

숨겨진 테스트 의존성은 테스트 코드에 명시되지 않은 전제 조건이 있을 때 발생합니다. 테스트 자체만 보면 통과해야 할 것 같은데, 실행 환경이나 다른 테스트의 상태에 따라 결과가 달라집니다.

```
숨겨진 의존성의 세 가지 유형:

① 실행 순서 의존: 특정 테스트가 먼저 실행되어야 통과
② 공유 상태 의존: 다른 테스트가 만들어둔 데이터에 의존
③ 환경 의존: 특정 시스템 상태(파일, 환경변수)에 의존
```

---

## 😱 유형 1: 실행 순서 의존

```java
// ❌ 테스트 A가 만든 User를 테스트 B가 사용
@SpringBootTest
class UserServiceTest {

    @Autowired UserService userService;
    @Autowired UserRepository userRepository;

    @Test
    void A_사용자_생성() {
        userService.create(new CreateUserCommand("hong@example.com", "홍길동"));
        assertThat(userRepository.count()).isEqualTo(1);
    }

    @Test
    void B_이메일로_사용자_조회() {
        // 테스트 A가 먼저 실행됐다고 암묵적으로 가정
        User found = userService.findByEmail("hong@example.com");
        assertThat(found).isNotNull(); // A가 실행되지 않았으면 실패
    }
}
```

JUnit 5는 실행 순서를 보장하지 않습니다. 테스트 클래스나 JVM 버전에 따라 순서가 달라질 수 있습니다.

```java
// ✅ 각 테스트가 자신의 전제 조건을 스스로 만든다
@SpringBootTest
class UserServiceTest {

    @Autowired UserService userService;
    @Autowired UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll(); // 깨끗한 상태에서 시작
    }

    @Test
    void 사용자_생성() {
        userService.create(new CreateUserCommand("hong@example.com", "홍길동"));
        assertThat(userRepository.count()).isEqualTo(1);
    }

    @Test
    void 이메일로_사용자_조회() {
        // 이 테스트가 필요한 데이터를 직접 만든다
        userRepository.save(new User("hong@example.com", "홍길동"));

        User found = userService.findByEmail("hong@example.com");
        assertThat(found).isNotNull();
    }
}
```

---

## 😱 유형 2: static 공유 상태

```java
// ❌ static 카운터가 테스트 간 공유됨
public class OrderIdGenerator {
    private static int counter = 0;

    public static Long nextId() {
        return (long) ++counter;
    }
}

class OrderTest {

    @Test
    void 첫번째_주문_ID는_1() {
        Long id = OrderIdGenerator.nextId();
        assertThat(id).isEqualTo(1L); // 다른 테스트가 먼저 호출했으면 실패
    }

    @Test
    void 두번째_주문_ID는_2() {
        OrderIdGenerator.nextId(); // 첫 번째 호출
        Long id = OrderIdGenerator.nextId();
        assertThat(id).isEqualTo(2L); // 다른 테스트가 먼저 호출했으면 실패
    }
}
```

```java
// ✅ 각 테스트마다 독립적인 Generator 인스턴스 사용
public class OrderIdGenerator {
    private int counter = 0; // static 제거

    public Long nextId() {
        return (long) ++counter;
    }
}

class OrderTest {

    @Test
    void 첫번째_주문_ID는_1() {
        OrderIdGenerator generator = new OrderIdGenerator(); // 독립 인스턴스
        assertThat(generator.nextId()).isEqualTo(1L);
    }

    @Test
    void 두번째_주문_ID는_2() {
        OrderIdGenerator generator = new OrderIdGenerator();
        generator.nextId();
        assertThat(generator.nextId()).isEqualTo(2L);
    }
}
```

---

## 😱 유형 3: Spring Bean 상태 오염

```java
// ❌ Spring Singleton Bean이 상태를 가짐
@Component
public class NotificationQueue {
    private final List<Notification> queue = new ArrayList<>(); // 인스턴스 상태

    public void enqueue(Notification notification) {
        queue.add(notification);
    }

    public List<Notification> drain() {
        List<Notification> snapshot = List.copyOf(queue);
        queue.clear();
        return snapshot;
    }
}

@SpringBootTest
class NotificationTest {

    @Autowired NotificationQueue queue;
    @Autowired OrderService orderService;

    @Test
    void 주문_시_알림_1개_생성() {
        orderService.place(aCommand().build());
        assertThat(queue.drain()).hasSize(1); // 다른 테스트에서 enqueue했으면 실패
    }
}
```

```java
// ✅ @BeforeEach에서 Bean 상태 초기화
@SpringBootTest
class NotificationTest {

    @Autowired NotificationQueue queue;
    @Autowired OrderService orderService;

    @BeforeEach
    void setUp() {
        queue.drain(); // Bean 상태 초기화 — 이전 테스트의 알림 제거
    }

    @Test
    void 주문_시_알림_1개_생성() {
        orderService.place(aCommand().build());
        assertThat(queue.drain()).hasSize(1);
    }
}
```

---

## 😱 유형 4: 파일 시스템 / 환경변수 의존

```java
// ❌ 파일이 미리 존재해야 통과
@Test
void 설정_파일_파싱() {
    ConfigParser parser = new ConfigParser();
    Config config = parser.parse("/etc/app/config.properties"); // 파일이 있어야 함
    assertThat(config.timeout()).isEqualTo(30);
}

// ❌ 환경변수가 설정되어 있어야 통과
@Test
void 외부_API_키_설정() {
    String apiKey = System.getenv("EXTERNAL_API_KEY"); // CI에서 설정 안 되면 null
    assertThat(apiKey).isNotNull();
}
```

```java
// ✅ 테스트에서 필요한 파일을 직접 만들고, 정리한다
@Test
void 설정_파일_파싱() throws IOException {
    Path tempConfig = Files.createTempFile("config", ".properties");
    Files.writeString(tempConfig, "timeout=30");

    try {
        ConfigParser parser = new ConfigParser();
        Config config = parser.parse(tempConfig.toString());
        assertThat(config.timeout()).isEqualTo(30);
    } finally {
        Files.deleteIfExists(tempConfig); // 반드시 정리
    }
}

// ✅ JUnit 5 @TempDir
@Test
void 설정_파일_파싱(@TempDir Path tempDir) throws IOException {
    Path configFile = tempDir.resolve("config.properties");
    Files.writeString(configFile, "timeout=30");

    Config config = new ConfigParser().parse(configFile.toString());
    assertThat(config.timeout()).isEqualTo(30);
    // @TempDir: 테스트 종료 시 JUnit이 자동 삭제
}

// ✅ 환경변수 의존 제거 — 주입으로 교체
@Test
void 외부_API_키_설정() {
    ApiClient client = new ApiClient("test-api-key"); // 직접 주입
    assertThat(client.isConfigured()).isTrue();
}
```

---

## 💻 실전 적용: 의존성 발견 방법

### 무작위 순서 실행

```java
// ✅ 무작위 순서로 실행해서 순서 의존성 발견
@TestMethodOrder(MethodOrderer.Random.class)
class OrderServiceTest { ... }
```

```
// Maven Surefire: 무작위 순서
<configuration>
    <runOrder>random</runOrder>
</configuration>
```

### 테스트 단독 실행

```java
// ✅ 테스트를 단독으로 실행해서 숨겨진 의존성 확인
// IDE에서 특정 테스트만 선택해서 실행
// → 전체 스위트에서는 통과하는데 단독으로는 실패 → 의존성 존재
```

### @DirtiesContext — 최후의 수단

```java
// 꼭 필요한 경우에만 사용 — 컨텍스트를 재생성하므로 매우 느림
@SpringBootTest
@DirtiesContext(classMode = AFTER_EACH_TEST_METHOD)
class StatefulBeanTest {
    // Bean 상태를 리셋할 다른 방법이 없을 때
    // 대부분의 경우 @BeforeEach 정리나 설계 변경으로 해결 가능
}
```

---

## 🤔 트레이드오프

### "@BeforeEach에서 deleteAll()을 호출하면 테스트가 느려지지 않는가?"

Testcontainers나 `@DataJpaTest`에서 매 테스트마다 `deleteAll()`을 호출하면 약간의 오버헤드가 있습니다. 하지만 테스트 격리를 포기하고 얻는 속도 향상보다, 간헐적 실패와 디버깅에 드는 비용이 훨씬 큽니다. `@Transactional` 롤백을 쓸 수 있는 경우라면 더 빠릅니다.

### "테스트 실행 순서를 고정(@TestMethodOrder)하면 안 되는가?"

순서를 고정하는 것은 숨겨진 의존성을 드러내지 않고 회피하는 것입니다. 나중에 테스트를 추가하거나 순서가 바뀌면 문제가 재발합니다. 근본 원인(공유 상태)을 제거하는 것이 올바른 해결책입니다.

---

## 📌 핵심 정리

```
숨겨진 의존성의 세 유형:
  실행 순서: 특정 테스트가 만든 데이터에 의존
  공유 상태: static 변수, Spring Singleton Bean 상태
  환경 의존: 외부 파일, 환경변수, 네트워크 상태

원칙: 각 테스트는 독립적으로 실행 가능해야 한다
  → 전제 조건을 스스로 만든다
  → 자신이 만든 상태를 스스로 정리한다

해결 방법:
  @BeforeEach에서 공유 상태 초기화
  static 필드 → 인스턴스 필드로 교체
  @TempDir으로 파일 시스템 의존성 격리
  환경변수 → 생성자 주입으로 교체

발견 방법:
  @TestMethodOrder(Random): 순서 의존성
  테스트 단독 실행: 환경/상태 의존성
  CI 간헐적 실패: 공유 상태

최후 수단:
  @DirtiesContext — 느리므로 최소화
  대부분 @BeforeEach 초기화로 대체 가능
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트들이 전체 스위트에서 실행할 때는 통과하는데 개별 실행 시 실패한다. 원인을 찾고 수정하라.

```java
@SpringBootTest
class ProductTest {

    @Autowired ProductRepository productRepository;
    @Autowired CategoryRepository categoryRepository;

    @Test
    void 카테고리_생성() {
        categoryRepository.save(new Category(1L, "전자기기"));
    }

    @Test
    void 카테고리에_속한_상품_조회() {
        // 카테고리가 없으면 빈 결과 반환 → 단독 실행 시 실패
        List<Product> products = productRepository.findByCategoryId(1L);
        assertThat(products).isNotEmpty();
    }
}
```

**Q2.** `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)`로 테스트 순서를 고정하는 것은 언제 적절하고 언제 안티패턴인가?

**Q3.** 병렬 테스트 실행 환경에서 숨겨진 의존성이 더 자주 드러나는 이유는 무엇인가? 병렬 실행을 안전하게 하기 위한 조건 두 가지를 설명하라.

> 💡 **해설**

**Q1.**

원인: `카테고리에_속한_상품_조회` 테스트가 `카테고리_생성` 테스트에서 만든 데이터에 암묵적으로 의존한다. 전체 스위트에서는 `카테고리_생성`이 먼저 실행되어 데이터가 있지만, 단독 실행 시에는 데이터가 없다.

수정:

```java
@SpringBootTest
class ProductTest {

    @Autowired ProductRepository productRepository;
    @Autowired CategoryRepository categoryRepository;

    @BeforeEach
    void setUp() {
        productRepository.deleteAll();
        categoryRepository.deleteAll();
    }

    @Test
    void 카테고리_생성() {
        categoryRepository.save(new Category(1L, "전자기기"));
        assertThat(categoryRepository.count()).isEqualTo(1);
    }

    @Test
    void 카테고리에_속한_상품_조회() {
        // 이 테스트가 필요한 데이터를 직접 만든다
        categoryRepository.save(new Category(1L, "전자기기"));
        productRepository.save(new Product("노트북", 1L));

        List<Product> products = productRepository.findByCategoryId(1L);
        assertThat(products).isNotEmpty();
    }
}
```

**Q2.**

`@TestMethodOrder(OrderAnnotation.class)` 적절한 경우: 통합 시나리오 테스트처럼 "생성 → 수정 → 삭제" 순서가 비즈니스 흐름 자체를 나타낼 때. 단, 이런 경우도 각 단계를 독립 테스트 대신 하나의 테스트 메서드로 작성하는 것을 먼저 고려해야 한다.

안티패턴인 경우: 공유 데이터 의존성을 숨기기 위해 순서를 강제하는 것. 순서를 고정해도 새 테스트를 추가하거나 실행 환경이 바뀌면 문제가 재발한다. 근본 원인(공유 상태)을 해결하지 않는 미봉책이다.

**Q3.**

병렬 실행에서 의존성이 더 잘 드러나는 이유: 순차 실행에서는 우연히 올바른 순서로 실행되어 통과하는 테스트가, 병렬 실행에서는 임의 순서와 타이밍으로 실행되어 공유 상태 충돌이 발생한다. 숨겨진 의존성이 강제로 노출된다.

병렬 실행을 안전하게 하기 위한 조건:

① **테스트 간 공유 상태 없음**: static 변수, Spring Singleton Bean의 가변 상태를 사용하지 않는다. 각 테스트가 독립적인 Fake/In-Memory 객체를 사용한다.

② **DB 격리**: 동시 실행되는 테스트들이 서로 다른 데이터를 사용한다. 고유 ID 전략 사용 (`UUID` 또는 테스트별 고유 접두사), 또는 Testcontainers를 사용해 테스트마다 별도 컨테이너 할당. `@Transactional` 롤백은 병렬에서도 안전하다.

---

<div align="center">

**[⬅️ 이전: Test Code Duplication](./05-test-code-duplication.md)** | **[다음: Assertion Roulette ➡️](./07-assertion-roulette.md)**

</div>
