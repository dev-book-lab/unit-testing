# 07. Fakes over Mocks

> **Fake는 테스트 코드가 아니라 프로덕션 코드와 같은 수준으로 다뤄야 한다 — 그만큼 가치가 있다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- In-memory Repository가 Mockito Repository보다 나은 구체적인 이유는 무엇인가?
- Fake를 만드는 비용을 언제 감수해야 하는가?
- Fake 자체가 잘못 구현됐을 때 어떻게 검증하는가?

---

## 🔍 같은 테스트, 두 가지 방식

```java
public class OrderService {
    private final OrderRepository orderRepository;

    public Order place(Cart cart, User user) {
        Order order = Order.from(cart, user);
        return orderRepository.save(order);
    }

    public List<Order> findByUser(Long userId) {
        return orderRepository.findByUserId(userId);
    }
}
```

이 두 메서드를 Mock과 Fake로 각각 테스트합니다.

---

## 😱 Mock Repository의 한계

```java
OrderRepository mockRepo = mock(OrderRepository.class);
OrderService service = new OrderService(mockRepo);

@Test
void 주문_생성_후_조회() {
    // Stub 설정 — save가 반환할 값을 미리 정해야 한다
    Order savedOrder = new Order(1L, 18_000, PENDING);
    when(mockRepo.save(any(Order.class))).thenReturn(savedOrder);

    // findByUserId도 Stub 설정 — save와 연결이 없다
    when(mockRepo.findByUserId(vipUser.id()))
        .thenReturn(List.of(savedOrder));

    Order result = service.place(cart, vipUser);
    List<Order> orders = service.findByUser(vipUser.id());

    assertThat(result.id()).isEqualTo(1L);
    assertThat(orders).hasSize(1);
}
```

이 테스트의 문제:

**1. save()와 findByUserId()가 연결되지 않는다.**  
실제 시스템에서는 DB가 연결을 보장하지만, Mock에서는 테스트 작성자가 두 Stub을 손으로 맞춰야 합니다. `findByUserId` Stub을 빠뜨리면 `null`이 반환됩니다.

**2. 테스트가 구현 세부사항에 묶인다.**  
`save()`가 어떤 인자로 호출되는지, 무엇을 반환하는지를 Stub에 명시해야 합니다. 내부 구현이 바뀌면 Stub도 바꿔야 합니다.

**3. 비현실적인 데이터 흐름.**  
"저장 후 조회"라는 도메인 흐름이 테스트에서 실제로 동작하지 않습니다.

---

## ✨ In-memory Fake Repository

```java
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    private long sequence = 1L;

    @Override
    public Order save(Order order) {
        Order saved = order.withId(sequence++);
        store.put(saved.id(), saved);
        return saved;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Order> findByUserId(Long userId) {
        return store.values().stream()
            .filter(o -> o.userId().equals(userId))
            .collect(Collectors.toList());
    }

    // 테스트 지원 메서드
    public int count() { return store.size(); }
    public void clear() { store.clear(); sequence = 1L; }
}
```

```java
InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
OrderService service = new OrderService(fakeRepo);

@Test
void 주문_생성_후_조회() {
    // Stub 설정 없음 — Fake가 실제처럼 동작
    Order result = service.place(cart, vipUser);
    List<Order> orders = service.findByUser(vipUser.id());

    // save()와 findByUserId()가 실제로 연결됨
    assertThat(orders).hasSize(1);
    assertThat(orders.get(0).id()).isEqualTo(result.id());
}

@Test
void 여러_주문_중_본인_주문만_조회된다() {
    service.place(cartOf(vipUser), vipUser);
    service.place(cartOf(vipUser), vipUser);
    service.place(cartOf(normalUser), normalUser); // 다른 사용자

    List<Order> vipOrders = service.findByUser(vipUser.id());

    assertThat(vipOrders).hasSize(2); // vipUser 주문만 2개
}
```

`when().thenReturn()` 없이 자연스럽게 흐릅니다.

---

## 🏛️ Fake가 Mock보다 나은 경우 vs 아닌 경우

### Fake가 나은 경우

```
"저장 후 조회"처럼 상태 연결이 필요한 흐름
여러 테스트에서 같은 협력자를 재사용할 때
Stub 설정 코드가 테스트 의도를 가릴 때
도메인 흐름을 현실적으로 검증하고 싶을 때
```

### Mock이 나은 경우

```
Fake 구현 비용이 과도할 때 (메서드 20개짜리 인터페이스)
한 번만 쓰이는 단순 협력자
부수효과 검증이 목적일 때 (이메일, 이벤트)
아직 구현되지 않은 협력자 (TDD 탐색 단계)
```

---

## 💻 실전 적용: Fake를 안전하게 만드는 Contract Test

Fake가 잘못 구현됐다면 모든 테스트가 거짓을 검증하게 됩니다. Contract Test는 Fake와 실제 구현이 동일한 계약을 지키는지 보장합니다.

```java
// 추상 계약 테스트 — 같은 테스트를 두 구현에 모두 돌린다
abstract class OrderRepositoryContractTest {

    abstract OrderRepository createRepository();

    private OrderRepository repository;

    @BeforeEach
    void setUp() { repository = createRepository(); }

    @Test
    void 저장_후_ID로_조회할_수_있다() {
        Order saved = repository.save(Order.of(18_000));
        assertThat(repository.findById(saved.id())).isPresent();
    }

    @Test
    void 저장하지_않은_ID는_비어있다() {
        assertThat(repository.findById(999L)).isEmpty();
    }

    @Test
    void 다른_사용자_주문은_조회되지_않는다() {
        repository.save(orderOf(userId(1L)));
        repository.save(orderOf(userId(2L)));

        assertThat(repository.findByUserId(1L)).hasSize(1);
    }

    @Test
    void 여러_주문을_저장하면_각각_다른_ID를_가진다() {
        Order first = repository.save(Order.of(10_000));
        Order second = repository.save(Order.of(20_000));
        assertThat(first.id()).isNotEqualTo(second.id());
    }
}

// Fake에 계약 테스트 적용
class InMemoryOrderRepositoryTest extends OrderRepositoryContractTest {
    @Override
    OrderRepository createRepository() {
        return new InMemoryOrderRepository();
    }
}

// 실제 JPA 구현에도 같은 계약 테스트 적용
@DataJpaTest
class JpaOrderRepositoryTest extends OrderRepositoryContractTest {
    @Autowired JpaOrderRepository jpaOrderRepository;

    @Override
    OrderRepository createRepository() { return jpaOrderRepository; }
}
```

`InMemoryOrderRepository`가 `findByUserId()`에서 필터링을 빠뜨렸다면, `다른_사용자_주문은_조회되지_않는다` 테스트가 `InMemoryOrderRepositoryTest`에서 실패합니다.

---

## 🤔 트레이드오프

### "Fake 코드도 유지보수해야 하지 않는가?"

맞습니다. `OrderRepository` 인터페이스에 메서드가 추가되면 Fake도 업데이트해야 합니다. 단, 이 비용은 여러 테스트에 걸쳐 분산됩니다. 10개 테스트에서 각각 복잡한 Stub을 관리하는 것과, Fake 하나를 유지하는 것의 총 비용을 비교하면 Fake가 유리한 경우가 많습니다.

### "Fake가 프로덕션 코드와 동일하게 동작한다는 보장이 없지 않은가?"

Contract Test가 이 문제를 완화합니다. 완전한 보장은 아니지만, 핵심 동작은 양쪽 모두 검증됩니다. DB 제약조건이나 트랜잭션 동작까지 검증이 필요하다면 Testcontainers로 실제 DB를 쓰는 통합 테스트를 병행합니다.

---

## 📌 핵심 정리

```
Mock Repository의 문제:
  save()와 findById()가 연결되지 않음
  Stub 설정이 구현에 묶임
  "저장 후 조회" 흐름을 검증 못함

Fake Repository의 장점:
  실제처럼 동작 — 상태 연결 자동
  Stub 설정 코드가 없어 테스트가 간결
  도메인 흐름을 현실적으로 검증

Fake를 안전하게 쓰는 방법:
  Contract Test: Fake와 실제 구현이 같은 계약을 지키는지 검증
  미구현 메서드는 UnsupportedOperationException

Fake 선택 기준:
  여러 테스트에서 재사용 + 상태 연결이 필요 → Fake
  부수효과 검증 + 한 번만 사용 → Mock

두 방식의 공존:
  비즈니스 로직 → Fake로 빠른 단위 테스트
  데이터 접근 계층 → @DataJpaTest로 통합 테스트
  Contract Test로 두 계층의 계약 동일성 보장
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 시나리오에서 Mock과 Fake 중 어느 것이 더 적합한가? 이유를 설명하라.

```
시나리오 A: 쿠폰 서비스
  쿠폰 저장, 조회, 사용 처리 (상태 변화)
  여러 테스트에서 반복 사용

시나리오 B: SMS 발송 서비스
  외부 SMS API 호출
  발송 여부 검증이 목적
  실제 SMS를 보내면 안 됨

시나리오 C: 실시간 환율 조회 서비스
  특정 테스트에서만 사용
  환율을 고정값으로 제공하는 것이 목적
```

**Q2.** `InMemoryOrderRepository`에서 `findByUserId()`를 아래처럼 잘못 구현했다고 하자. Contract Test가 있다면 이 버그를 잡을 수 있는가?

```java
@Override
public List<Order> findByUserId(Long userId) {
    return new ArrayList<>(store.values()); // userId 필터링 빠뜨림!
}
```

**Q3.** Fake Repository와 `@DataJpaTest` + 실제 DB 방식의 트레이드오프를 비교하고, 두 방식을 함께 사용하는 전략을 설명하라.

> 💡 **해설**
>
> **Q1.** 시나리오 A (쿠폰 서비스): **Fake.** 저장-조회-사용이 연결된 상태 변화가 핵심이고 여러 테스트에서 재사용된다. `InMemoryCouponRepository`를 만들어 실제처럼 동작시키는 것이 복잡한 Mock Stub보다 낫다. 시나리오 B (SMS 발송): **Mock.** 외부 API 부수효과(발송 여부)를 검증하는 것이 목적이므로 `verify(smsSender).send(...)` 방식이 직접적이다. Fake SMS 서비스도 가능하지만 Mock이 더 단순하다. 시나리오 C (환율 조회): **Stub.** 특정 테스트에서만 쓰이고, 고정값을 제공하는 것이 목적이다. `when(rateApi.getRate("USD", "KRW")).thenReturn(1_300.0)`으로 충분하다.
>
> **Q2.** Contract Test에 `다른_사용자_주문은_조회되지_않는다` 같은 테스트가 있다면 잡을 수 있다. `repository.save(orderOf(userId=1L)); repository.save(orderOf(userId=2L)); assertThat(repository.findByUserId(1L)).hasSize(1)` — 잘못된 Fake는 두 개를 반환하므로 `InMemoryOrderRepositoryTest`가 실패한다. 이것이 Contract Test의 핵심 가치다. Contract Test가 없다면 이 버그를 발견하지 못하고, 도메인 테스트 전체가 잘못된 Fake 위에서 돌아가게 된다.
>
> **Q3.** Fake Repository: 수 ms 단위의 속도, 외부 의존 없음, 격리된 단위 테스트 가능. 단점: 실제 DB와 동작 차이(JPA 쿼리, 유니크 제약, 트랜잭션, N+1) 검증 불가. `@DataJpaTest` + 실제 DB: 실제 동작 보장, DB 제약조건과 JPA 동작 검증 가능. 단점: 초 단위 속도, DB 설정 필요. 함께 쓰는 전략: 비즈니스 로직은 Fake로 빠르게 검증(단위 테스트), 데이터 접근 계층(`OrderRepository` 구현체)은 `@DataJpaTest`로 별도 검증(통합 테스트). Contract Test로 두 계층이 같은 계약을 지키는지 보장한다. 이것이 테스트 피라미드의 계층 분리와 그대로 일치한다.

---

<div align="center">

**[⬅️ 이전: Verify Wisely](./06-verify-wisely.md)**

</div>
