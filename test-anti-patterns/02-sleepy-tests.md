# 02. Sleepy Tests

> **`Thread.sleep(2000)`은 충분히 기다리는 게 아니라 — 어떤 환경에서는 너무 짧고, 어떤 환경에서는 너무 길다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- `Thread.sleep()`으로 비동기를 검증할 때 어떤 문제가 생기는가?
- `Awaitility`가 `Thread.sleep()`보다 나은 이유는 무엇인가?
- 비동기 코드를 동기적으로 테스트할 수 있는 설계 방법은 무엇인가?

---

## 🔍 Thread.sleep()의 두 가지 실패 방식

```java
@Test
void 주문_생성_시_이메일이_발송된다() throws InterruptedException {
    orderService.place(aCommand().build()); // 비동기로 이메일 발송

    Thread.sleep(2000); // 2초 대기

    verify(emailSender).send(any(), any()); // 검증
}
```

이 테스트는 두 방식으로 실패합니다.

**실패 방식 1: Flaky** — CI 서버가 느리거나 부하가 많으면 2초 안에 이메일 발송이 완료되지 않습니다. 테스트가 간헐적으로 실패합니다. sleep을 3초로 늘리고, 또 실패하면 5초로 늘리는 악순환이 시작됩니다.

**실패 방식 2: 느림** — 이메일 발송이 실제로 100ms에 완료되어도 항상 2초를 기다립니다. 이런 테스트가 50개라면 100초를 sleep에 낭비합니다.

```
Thread.sleep()의 딜레마:
  너무 짧으면 → Flaky (환경에 따라 간헐적 실패)
  너무 길면  → 테스트 스위트 전체가 느려짐
  "적당한" 값은 존재하지 않음
```

---

## 😱 실제 코드에서 나타나는 패턴

### 패턴 1: 단순 sleep

```java
// ❌
@Test
void 배치_처리_완료_후_상태_변경() throws InterruptedException {
    batchService.processAsync(orders);
    Thread.sleep(5000); // "5초면 충분하겠지"
    assertThat(orderRepository.countByStatus(PROCESSED)).isEqualTo(10);
}
```

### 패턴 2: sleep이 숨겨진 반복 대기

```java
// ❌ sleep을 루프로 감싸도 본질이 같다
@Test
void 이벤트_처리_대기() throws InterruptedException {
    eventPublisher.publish(new OrderPlaced(order));

    for (int i = 0; i < 10; i++) {
        Thread.sleep(500);
        if (eventRepository.count() > 0) break; // 처리됐으면 break
    }

    assertThat(eventRepository.count()).isGreaterThan(0);
}
```

루프를 도입했지만 최대 대기 시간(5초)과 폴링 간격(500ms)을 직접 관리해야 합니다. 오류 메시지도 명확하지 않습니다.

### 패턴 3: CountDownLatch로 개선했지만 여전히 timeout 의존

```java
// ⚠️ CountDownLatch: 더 나아졌지만 여전히 timeout 설정이 필요
@Test
void 이벤트_처리() throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(1);
    doAnswer(invocation -> { latch.countDown(); return null; })
        .when(eventHandler).handle(any());

    eventPublisher.publish(new OrderPlaced(order));

    boolean completed = latch.await(5, TimeUnit.SECONDS);
    assertThat(completed).isTrue(); // timeout이면 false
    verify(eventHandler).handle(any());
}
```

`CountDownLatch`는 특정 조건을 기다리는 데 유용하지만, `await()` timeout을 여전히 수동으로 설정해야 하고, 실패 메시지가 불명확합니다.

---

## ✨ Awaitility — 조건 기반 대기

Awaitility는 폴링 방식으로 조건이 충족될 때까지 기다립니다. timeout을 설정하지만, 조건이 충족되면 즉시 다음으로 넘어갑니다.

```kotlin
// build.gradle.kts
testImplementation("org.awaitility:awaitility:4.2.0")
```

### 기본 사용

```java
import static org.awaitility.Awaitility.await;

@Test
void 주문_생성_시_이메일이_발송된다() {
    orderService.place(aCommand().build());

    await()
        .atMost(5, SECONDS)           // 최대 5초 대기
        .pollInterval(100, MILLISECONDS) // 100ms마다 조건 확인
        .untilAsserted(() ->
            verify(emailSender).send(
                eq("user@example.com"),
                contains("주문이 접수됐습니다")
            )
        );
}
```

조건이 50ms에 충족되면 50ms에 끝납니다. 5초가 지나도 충족되지 않으면 명확한 메시지로 실패합니다.

### Repository 상태 기반 대기

```java
@Test
void 배치_처리_완료_후_상태_변경() {
    batchService.processAsync(orders);

    await()
        .atMost(10, SECONDS)
        .until(() -> orderRepository.countByStatus(PROCESSED) == 10);

    // 추가 검증
    List<Order> processed = orderRepository.findByStatus(PROCESSED);
    assertThat(processed).allMatch(o -> o.processedAt() != null);
}
```

### 이벤트 발행 검증

```java
@Test
void Kafka_이벤트_처리_완료() {
    kafkaTemplate.send("order-placed", new OrderPlacedEvent(orderId));

    await()
        .atMost(Duration.ofSeconds(10))
        .pollDelay(Duration.ofMillis(200))   // 첫 확인까지 200ms 지연
        .pollInterval(Duration.ofMillis(500)) // 이후 500ms마다
        .untilAsserted(() -> {
            Order order = orderRepository.findById(orderId).orElseThrow();
            assertThat(order.status()).isEqualTo(PROCESSING);
        });
}
```

### 실패 메시지 커스터마이징

```java
await()
    .atMost(5, SECONDS)
    .alias("이메일 발송 대기")  // 실패 시 메시지에 포함
    .untilAsserted(() ->
        verify(emailSender).send(any(), any())
    );

// 실패 시 출력:
// ConditionTimeoutException: 이메일 발송 대기
// Condition was not fulfilled within 5 seconds
```

---

## 💻 더 나은 설계: 비동기를 동기로 만들기

Awaitility는 좋은 도구지만, 비동기 코드 자체를 동기적으로 테스트할 수 있게 설계하면 더 빠르고 안정적입니다.

### 전략 1: 이벤트 핸들러를 분리해서 동기 테스트

```java
// ❌ Kafka Consumer 자체를 테스트
@Test
void Kafka_이벤트_처리() {
    kafkaTemplate.send("order-placed", event);
    await().atMost(10, SECONDS).untilAsserted(/* 검증 */);
}

// ✅ 핸들러 로직을 분리해서 동기 테스트 (Humble Object 패턴)
@Test
void 이벤트_핸들러_로직() {
    OrderPlacedHandler handler = new OrderPlacedHandler(fakeRepo, mockEmailSender);

    handler.handle(new OrderPlacedEvent(orderId, userId)); // 동기 호출

    verify(mockEmailSender).send(any(), any()); // 즉시 검증
}
```

`@KafkaListener`는 메시지 수신 후 `handler.handle()`에 위임합니다. 핸들러 로직은 동기 단위 테스트로, Kafka 연동은 `@EmbeddedKafka` 최소 통합 테스트로 분리합니다.

### 전략 2: 테스트에서 동기 실행기 주입

```java
// 비동기 실행기를 주입받는 설계
public class NotificationService {
    private final Executor executor;
    private final EmailSender emailSender;

    public void sendAsync(User user, String message) {
        executor.execute(() -> emailSender.send(user.email(), message));
    }
}
```

```java
// 테스트: 동기 실행기 주입
@Test
void 알림_발송() {
    Executor syncExecutor = Runnable::run; // 즉시 동기 실행
    NotificationService service = new NotificationService(syncExecutor, mockEmailSender);

    service.sendAsync(user, "주문이 접수됐습니다");

    // Thread.sleep 없이 즉시 검증
    verify(mockEmailSender).send(eq("user@example.com"), contains("주문"));
}
```

---

## 🤔 트레이드오프

### "Awaitility도 결국 polling인데 테스트가 느려지지 않는가?"

조건이 충족되는 즉시 다음으로 넘어가므로 `Thread.sleep()`처럼 고정 시간을 낭비하지 않습니다. `pollInterval`을 너무 짧게 설정하면 CPU를 낭비하고, 너무 길게 설정하면 반응이 느립니다. `100ms~500ms` 정도가 일반적인 균형점입니다.

### "비동기 코드는 비동기로 테스트해야 현실적이지 않은가?"

비동기 동작 자체(메시지 큐, 스레드 풀)는 별도 통합 테스트에서 검증합니다. 비즈니스 로직은 동기 단위 테스트로 빠르게 검증합니다. 두 가지를 하나의 테스트에서 동시에 검증하려는 것이 `Thread.sleep()`을 유발합니다.

---

## 📌 핵심 정리

```
Thread.sleep()의 문제:
  너무 짧으면 Flaky
  너무 길면 테스트 스위트가 느려짐
  "적당한" 값이 환경마다 다름

Awaitility:
  조건 충족 즉시 통과 → 불필요한 대기 없음
  timeout 초과 시 명확한 실패 메시지
  untilAsserted: AssertJ/Mockito verify 사용 가능
  until: 단순 조건 boolean 반환

설정 기준:
  atMost: 충분히 넉넉하게 (5~10초)
  pollInterval: 100ms~500ms (CPU 낭비 vs 반응성)
  pollDelay: 비동기 작업이 즉시 시작되지 않을 때

더 나은 설계:
  핸들러 로직 분리 → 동기 단위 테스트
  Executor 주입 → 테스트에서 동기 Executor 사용
  비동기 인프라 → 최소한의 통합 테스트

Awaitility 도입 시점:
  동기화가 불가피한 통합 테스트
  @EmbeddedKafka, 스케줄러, 배치 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 테스트를 Awaitility로 개선하고, 실패 메시지를 명확하게 만들어라.

```java
@Test
void 주문_처리_완료() throws InterruptedException {
    orderProcessor.processAsync(orderId);
    Thread.sleep(3000);
    Order order = orderRepository.findById(orderId).orElseThrow();
    assertThat(order.status()).isEqualTo(COMPLETED);
}
```

**Q2.** 비동기 `@Scheduled` 배치가 매 분 실행된다. 이 배치가 특정 조건에서 동작하는지 테스트하려면 어떻게 해야 하는가? `Thread.sleep(60000)` 없이 해결하는 방법을 설명하라.

**Q3.** `CountDownLatch`와 `Awaitility`를 각각 어떤 상황에서 선택하는가? `CountDownLatch`가 더 적합한 경우를 하나 제시하라.

> 💡 **해설**

**Q1.**

```java
@Test
void 주문_처리_완료() {
    orderProcessor.processAsync(orderId);

    await()
        .alias("주문 처리 완료 대기")
        .atMost(10, SECONDS)
        .pollInterval(200, MILLISECONDS)
        .untilAsserted(() -> {
            Order order = orderRepository.findById(orderId).orElseThrow();
            assertThat(order.status())
                .as("주문 %d의 상태가 COMPLETED여야 한다", orderId)
                .isEqualTo(COMPLETED);
        });
}
```

개선 포인트: `Thread.sleep(3000)` 제거로 평균 실행 시간 단축. `atMost(10, SECONDS)`로 충분한 여유 확보. `alias()`와 `as()`로 실패 시 명확한 메시지 제공.

**Q2.**

`@Scheduled` 배치를 Humble Object로 분리한다.

```java
// Scheduler: 얇은 경계
@Component
public class OrderCleanupScheduler {
    private final OrderCleanupJob job;

    @Scheduled(cron = "0 * * * * *")
    public void run() {
        job.execute();
    }
}

// Job: 실제 로직 — 동기 테스트 가능
public class OrderCleanupJob {
    public void execute() {
        // 실제 배치 로직
    }
}
```

```java
// 테스트: Job을 직접 호출
@Test
void 만료_주문_정리() {
    // given: 만료 조건 데이터 준비
    orderRepository.save(expiredOrder());

    // when: Job을 동기로 직접 실행
    job.execute();

    // then: 즉시 검증
    assertThat(orderRepository.findExpired()).isEmpty();
}
```

스케줄러 트리거 자체는 `@Scheduled` 통합 테스트에서 별도로 검증하고, 비즈니스 로직은 Job 단위 테스트에서 동기로 검증한다.

**Q3.**

`Awaitility`가 더 적합한 경우: 조건이 DB 상태, Mock 호출 횟수 등 외부에서 관찰 가능한 것일 때. 여러 검증이 필요할 때. `untilAsserted`로 AssertJ 단언을 그대로 사용할 수 있어 실패 메시지가 명확.

`CountDownLatch`가 더 적합한 경우: 정확히 N번의 이벤트가 발생했음을 보장해야 할 때. 예를 들어 Kafka Consumer가 정확히 5개의 메시지를 처리했는지 확인할 때.

```java
// CountDownLatch가 적합한 예
CountDownLatch processed = new CountDownLatch(5);
kafkaConsumer.setOnMessage(msg -> processed.countDown());

// 5개 메시지 전송
for (int i = 0; i < 5; i++) {
    kafkaTemplate.send("topic", message(i));
}

// 정확히 5번 처리될 때까지 대기
assertThat(processed.await(10, SECONDS)).isTrue();
```

`Awaitility.until(() -> processedCount.get() == 5)`도 가능하지만, 카운트를 원자적으로 감소시키는 latch 의미론이 명확할 때는 `CountDownLatch`가 더 직관적이다.

---

<div align="center">

**[⬅️ 이전: Test Logic in Production](./01-test-logic-in-production.md)** | **[다음: Flickering Tests ➡️](./03-flickering-tests.md)**

</div>
