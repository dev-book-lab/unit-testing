# 04. Humble Object Pattern

> **테스트할 수 없는 경계(UI, 프레임워크, 외부 시스템)를 최대한 얇게 만들고, 로직은 그 안쪽으로 밀어넣는다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- Humble Object Pattern이 해결하는 문제는 무엇인가?
- "얇은 경계"와 "두꺼운 경계"의 차이는 무엇인가?
- Controller, Batch Job, Event Listener에서 이 패턴을 어떻게 적용하는가?

---

## 🔍 테스트할 수 없는 경계란

모든 코드가 단위 테스트하기 쉬운 것은 아닙니다. 어떤 경계는 본질적으로 테스트하기 어렵습니다.

```
테스트하기 어려운 경계:
  HTTP 요청/응답 (Controller, Servlet)
  UI 렌더링 (View, Template)
  배치 Job 실행 (JobLauncher, Step)
  이벤트 리스너 (Kafka Consumer, JMS Listener)
  스케줄러 (Scheduled, Cron)
```

이 경계들은 프레임워크와 강하게 결합되어 있어서 단위 테스트보다 통합 테스트가 더 적합합니다. 통합 테스트는 느립니다.

**Humble Object Pattern의 핵심 아이디어:** 테스트하기 어려운 경계를 최대한 얇게(humble) 만들어서, 거기에 비즈니스 로직이 없도록 한다. 로직은 경계 안쪽으로 밀어서 단위 테스트가 가능한 순수한 객체에 둔다.

---

## 😱 두꺼운 경계: 컨트롤러에 로직이 있다

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    @PostMapping
    public ResponseEntity<OrderResponse> place(@RequestBody OrderRequest request) {

        // ❌ 비즈니스 로직이 컨트롤러에 있다
        User user = userRepository.findById(request.userId())
                .orElseThrow(() -> new UserNotFoundException(request.userId()));

        if (user.grade() == Grade.VIP) {
            // VIP 할인 로직
            int discount = (int) (request.totalPrice() * 0.1);
            request = request.withDiscount(discount);
        }

        if (request.totalPrice() < 1_000) {
            return ResponseEntity.badRequest()
                    .body(OrderResponse.error("최소 주문 금액은 1,000원입니다"));
        }

        Order order = Order.from(request, user);
        orderRepository.save(order);

        return ResponseEntity.ok(OrderResponse.of(order));
    }
}
```

이 컨트롤러를 테스트하려면?

```java
// VIP 할인 로직만 테스트하고 싶은데
// @WebMvcTest를 써야 하고
// MockMvc로 HTTP 요청을 만들어야 하고
// JSON 직렬화/역직렬화가 관여하고
// 응답 status code도 확인해야 한다
```

VIP 할인 로직 하나를 검증하기 위해 HTTP 스택 전체가 개입합니다.

---

## ✨ 얇은 경계: 로직을 안쪽으로 밀기

### Step 1: 비즈니스 로직을 별도 서비스로 추출

```java
// ✅ 로직이 있는 서비스 — Spring 없이 테스트 가능
public class OrderService {

    private final UserFinder userFinder;
    private final DiscountPolicy discountPolicy;
    private final OrderRepository orderRepository;

    public OrderResult place(Long userId, int totalPrice) {
        User user = userFinder.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));

        if (totalPrice < 1_000) {
            throw new MinimumAmountException("최소 주문 금액은 1,000원입니다");
        }

        int discount = discountPolicy.calculate(user);
        Order order = Order.from(userId, totalPrice, discount);
        orderRepository.save(order);

        return OrderResult.success(order);
    }
}
```

### Step 2: 컨트롤러는 변환과 위임만 담당

```java
// ✅ 얇은 컨트롤러 — HTTP 관심사만
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;  // 주입

    @PostMapping
    public ResponseEntity<OrderResponse> place(@RequestBody OrderRequest request) {
        try {
            // 변환 → 위임 → 변환
            OrderResult result = orderService.place(
                    request.userId(),
                    request.totalPrice()
            );
            return ResponseEntity.ok(OrderResponse.of(result.order()));

        } catch (UserNotFoundException e) {
            return ResponseEntity.notFound().build();
        } catch (MinimumAmountException e) {
            return ResponseEntity.badRequest()
                    .body(OrderResponse.error(e.getMessage()));
        }
    }
}
```

컨트롤러는 HTTP 요청을 도메인 객체로 변환하고, 서비스에 위임하고, 결과를 HTTP 응답으로 변환합니다. 비즈니스 규칙이 없습니다.

### 테스트가 어떻게 바뀌는가

```java
// ✅ OrderService 단위 테스트 — HTTP 없이 빠르게
class OrderServiceTest {

    private final UserFinder stubFinder = id -> Optional.of(vipUser);
    private final DiscountPolicy vipPolicy = user -> 10;
    private final InMemoryOrderRepository fakeRepo = new InMemoryOrderRepository();
    private final OrderService service = new OrderService(stubFinder, vipPolicy, fakeRepo);

    @Test
    void VIP_회원_주문_시_10퍼센트_할인이_적용된다() {
        OrderResult result = service.place(1L, 20_000);
        assertThat(result.order().discountedPrice()).isEqualTo(18_000);
    }

    @Test
    void 최소_금액_미만_주문은_예외가_발생한다() {
        assertThatThrownBy(() -> service.place(1L, 999))
                .isInstanceOf(MinimumAmountException.class);
    }
}

// ✅ OrderController 통합 테스트 — HTTP 변환만 확인
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @MockBean OrderService orderService;

    @Test
    void 주문_성공_시_200을_반환한다() throws Exception {
        when(orderService.place(anyLong(), anyInt()))
                .thenReturn(OrderResult.success(sampleOrder));

        mockMvc.perform(post("/orders")
                        .contentType(APPLICATION_JSON)
                        .content("{\"userId\": 1, \"totalPrice\": 20000}"))
                .andExpect(status().isOk());
    }
}
```

비즈니스 로직은 빠른 단위 테스트로, HTTP 변환은 `@WebMvcTest`로 나뉩니다.

---

## 🏛️ 다른 경계에 패턴 적용하기

### Batch Job

```java
// ❌ Step에 로직이 있다
@Bean
public Step inactiveUserCleanupStep() {
    return stepBuilderFactory.get("inactiveUserCleanupStep")
            .<User, User>chunk(100)
            .reader(userItemReader())
            .processor(user -> {
                // 비즈니스 로직이 람다 안에 숨어있다
                if (user.lastLoginAt().isBefore(LocalDate.now().minusMonths(6))) {
                    user.deactivate();
                    return user;
                }
                return null;
            })
            .writer(userItemWriter())
            .build();
}

// ✅ 로직을 ItemProcessor로 추출하고, 그 안의 정책을 또 추출
public class InactiveUserProcessor implements ItemProcessor<User, User> {

    private final InactivityPolicy inactivityPolicy;

    @Override
    public User process(User user) {
        if (inactivityPolicy.isInactive(user)) {
            return user.deactivated();
        }
        return null;
    }
}

// InactivityPolicy는 단위 테스트 가능
class InactivityPolicyTest {
    @Test
    void 6개월_이상_미접속_사용자는_비활성으로_판단한다() {
        Clock fixedClock = Clock.fixed(...);
        InactivityPolicy policy = new InactivityPolicy(6, fixedClock);

        User longAbsent = aUser().withLastLoginAt(7개월전).build();
        assertThat(policy.isInactive(longAbsent)).isTrue();
    }
}
```

### Kafka Consumer (Event Listener)

```java
// ❌ 로직이 리스너에 있다
@KafkaListener(topics = "order-placed")
public void handleOrderPlaced(OrderPlacedEvent event) {
    User user = userRepository.findById(event.userId()).orElseThrow();
    String email = emailTemplateEngine.render("order-confirm", user, event.order());
    emailSender.send(user.email(), "주문 확인", email);
    pointService.accumulatePoints(user, event.order().totalPrice());
}

// ✅ 리스너는 이벤트 수신과 위임만
@KafkaListener(topics = "order-placed")
public void handleOrderPlaced(OrderPlacedEvent event) {
    orderPlacedHandler.handle(event);  // 위임
}

// 로직이 있는 핸들러 — 단위 테스트 가능
public class OrderPlacedHandler {
    private final UserFinder userFinder;
    private final EmailService emailService;
    private final PointService pointService;

    public void handle(OrderPlacedEvent event) {
        User user = userFinder.findById(event.userId()).orElseThrow();
        emailService.sendOrderConfirmation(user, event.order());
        pointService.accumulatePoints(user, event.order().totalPrice());
    }
}
```

---

## 💻 실전 적용: 얼마나 얇게 만들어야 하는가

```
컨트롤러에 남겨도 되는 것:
  요청 → 도메인 객체 변환
  예외 → HTTP 응답 코드 변환
  인증/인가 어노테이션 (@PreAuthorize)

컨트롤러에서 제거해야 하는 것:
  비즈니스 규칙 (최소 금액, 할인 계산)
  도메인 상태 변경 로직
  복잡한 조건 분기
  외부 서비스 호출 조합
```

"컨트롤러 메서드가 5줄 이상이라면 로직이 있는지 의심하라"는 경험적 기준이 있습니다.

---

## 🤔 트레이드오프

### "컨트롤러 예외 처리도 서비스로 내리면 더 깔끔하지 않은가?"

예외 처리의 책임 위치는 설계 철학에 따라 다릅니다. 서비스는 도메인 예외를 던지고, HTTP 예외 변환은 컨트롤러(또는 `@ControllerAdvice`)가 담당하는 것이 관심사 분리에 맞습니다. 서비스가 `ResponseEntity`를 알면 HTTP에 결합됩니다.

### "얇은 컨트롤러면 @WebMvcTest에서 테스트할 것이 별로 없지 않은가?"

맞습니다. 얇은 컨트롤러의 `@WebMvcTest`는 HTTP status code 변환, 요청 역직렬화, 응답 직렬화 정도만 검증합니다. 이것이 의도입니다. 비즈니스 로직 검증은 서비스 단위 테스트가 담당합니다.

---

## 📌 핵심 정리

```
Humble Object Pattern:
  테스트 불가 경계를 최대한 얇게
  로직은 경계 안쪽의 순수한 객체로

얇은 경계의 책임:
  변환 (HTTP ↔ 도메인, Event ↔ 도메인)
  위임 (서비스에 넘기기)
  프레임워크 관심사 (응답 코드, 직렬화)

로직이 경계에 있다는 신호:
  컨트롤러 메서드가 10줄 이상
  리스너 안에 if-else가 있다
  배치 Step에 비즈니스 규칙이 있다

적용 대상:
  Controller → Service
  Kafka Listener → Handler
  Batch Processor → Policy/Service
  Scheduler → Job/Service

결과:
  비즈니스 로직 → 빠른 단위 테스트
  경계 계층 → 소수의 통합 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 컨트롤러에서 단위 테스트 가능한 부분을 서비스로 추출하라. 추출 후 서비스 테스트 코드도 작성하라.

```java
@GetMapping("/users/{id}/points")
public ResponseEntity<PointSummary> getPoints(@PathVariable Long id) {
    User user = userRepository.findById(id).orElseThrow(UserNotFoundException::new);
    List<PointHistory> histories = pointRepository.findByUserId(id);

    int total = histories.stream().mapToInt(PointHistory::amount).sum();
    int expiringSoon = histories.stream()
            .filter(h -> h.expiresAt().isBefore(LocalDate.now().plusDays(30)))
            .mapToInt(PointHistory::amount)
            .sum();

    return ResponseEntity.ok(new PointSummary(total, expiringSoon));
}
```

**Q2.** Kafka Consumer를 통합 테스트할 때(`@EmbeddedKafka`)와 Humble Object로 분리해서 단위 테스트할 때의 장단점을 비교하라.

**Q3.** 스케줄러(`@Scheduled`)가 있는 클래스를 Humble Object 패턴으로 리팩터링하라.

```java
@Component
public class PointExpiryScheduler {

    @Scheduled(cron = "0 0 0 * * *")
    public void expirePoints() {
        LocalDate today = LocalDate.now();
        List<Point> expiredPoints = pointRepository.findByExpiresAtBefore(today);
        expiredPoints.forEach(p -> {
            p.expire();
            pointRepository.save(p);
            eventPublisher.publish(new PointExpired(p.userId(), p.amount()));
        });
    }
}
```

> 💡 **해설**

**Q1.**

추출 기준: `PointSummary` 계산 로직(`total`, `expiringSoon` 계산)이 비즈니스 로직이다. `LocalDate.now()` 현재 날짜 의존성도 문제다.

추출 후 서비스:

```java
public class PointSummaryService {
    private final UserFinder userFinder;
    private final PointHistoryReader historyReader;
    private final Clock clock;

    public PointSummary summarize(Long userId) {
        userFinder.findById(userId).orElseThrow(UserNotFoundException::new);
        List<PointHistory> histories = historyReader.findByUserId(userId);

        int total = histories.stream().mapToInt(PointHistory::amount).sum();
        LocalDate threshold = LocalDate.now(clock).plusDays(30);
        int expiringSoon = histories.stream()
            .filter(h -> h.expiresAt().isBefore(threshold))
            .mapToInt(PointHistory::amount)
            .sum();

        return new PointSummary(total, expiringSoon);
    }
}
```

서비스 테스트: `Clock.fixed()`로 "30일 내 만료" 경계값 테스트 포함.

**Q2.**

`@EmbeddedKafka`: 실제 Kafka 프로토콜 검증, 직렬화/역직렬화, offset 관리 등 인프라 동작 검증. 단점: 수 초~수십 초, 설정 복잡.

Humble Object 단위 테스트: `handler.handle(event)` 직접 호출 — ms 단위, Kafka 없이 비즈니스 로직만 빠르게 검증. 단점: Kafka 메시지 형식, 역직렬화 오류를 잡지 못함.

함께 쓰는 전략: 핸들러 로직은 단위 테스트, Kafka 컨슈머 설정은 최소한의 `@EmbeddedKafka` 통합 테스트로 검증.

**Q3.**

분리 결과:

```java
@Component
public class PointExpiryScheduler {
    private final PointExpiryJob job;

    @Scheduled(cron = "0 0 0 * * *")
    public void expirePoints() {
        job.execute();
    }
}
```

```java
public class PointExpiryJob {
    private final PointRepository pointRepository;
    private final EventPublisher eventPublisher;
    private final Clock clock;

    public void execute() {
        LocalDate today = LocalDate.now(clock);
        List<Point> expired = pointRepository.findByExpiresAtBefore(today);
        expired.forEach(p -> {
            p.expire();
            pointRepository.save(p);
            eventPublisher.publish(new PointExpired(p.userId(), p.amount()));
        });
    }
}
```

`PointExpiryJob`은 `Clock` 주입으로 경계값 테스트 가능. `PointExpiryScheduler`는 `job.execute()`만 호출하므로 테스트가 거의 필요 없다.

---

<div align="center">

**[⬅️ 이전: Interface Segregation](./03-interface-segregation.md)** | **[다음: Pure Functions First ➡️](./05-pure-functions-first.md)**

</div>
