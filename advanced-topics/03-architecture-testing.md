# 03. Architecture Testing

> **"도메인 레이어가 인프라를 의존하면 안 된다"는 규칙은 코드 리뷰에서만 지킬 수 없다 — 테스트로 강제해야 한다**

---

## 🎯 핵심 질문

이 문서를 읽고 나면 아래 질문에 답할 수 있어야 합니다.

- ArchUnit은 어떤 문제를 해결하는가?
- 레이어 의존성 규칙을 테스트로 어떻게 표현하는가?
- 패키지 구조와 어노테이션 규칙을 어떻게 강제하는가?

---

## 🔍 아키텍처 규칙이 깨지는 방식

```
팀 내 합의된 아키텍처 규칙:
  "도메인 레이어는 인프라 레이어를 의존해서는 안 된다"
  "서비스는 다른 서비스를 직접 주입받으면 안 된다"
  "@Transactional은 서비스 레이어에만 붙여야 한다"

실제로 일어나는 일:
  개발자 A: Order 엔티티에서 JpaRepository를 직접 참조
  개발자 B: OrderService에서 UserService를 new로 생성
  개발자 C: Controller에 @Transactional 추가

코드 리뷰에서 모든 PR을 잡을 수 없다.
빌드가 통과해도 아키텍처는 조용히 무너진다.
```

ArchUnit은 이 규칙들을 JUnit 테스트로 표현해서, 빌드 시점에 자동으로 검사합니다.

---

## 🏛️ ArchUnit 설정

```kotlin
// build.gradle.kts
testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")
```

```java
// 기본 구조
@AnalyzeClasses(packages = "com.example.order") // 검사 대상 패키지
class ArchitectureTest {

    @ArchTest
    static final ArchRule 규칙 = /* 규칙 정의 */;
}
```

---

## 🏛️ 레이어 의존성 규칙

### 클래식 레이어드 아키텍처 강제

```java
@AnalyzeClasses(packages = "com.example")
class LayeredArchitectureTest {

    @ArchTest
    static final ArchRule 레이어드_아키텍처 =
        layeredArchitecture()
            .consideringAllDependencies()
            .layer("Controller").definedBy("..controller..")
            .layer("Service").definedBy("..service..")
            .layer("Repository").definedBy("..repository..")
            .layer("Domain").definedBy("..domain..")

            .whereLayer("Controller").mayOnlyBeAccessedByLayers("Controller")
            .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller", "Service")
            .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
            .whereLayer("Domain").mayOnlyBeAccessedByLayers("Controller", "Service", "Repository");
}
```

### Hexagonal Architecture 강제

```java
@AnalyzeClasses(packages = "com.example.order")
class HexagonalArchitectureTest {

    @ArchTest
    static final ArchRule 도메인은_인프라를_모른다 =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
                .resideInAnyPackage(
                    "..adapter..",     // JPA 어댑터
                    "..infrastructure..", // Kafka, Redis
                    "org.springframework.data.."  // Spring Data
                );

    @ArchTest
    static final ArchRule 애플리케이션은_어댑터를_모른다 =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
                .resideInAPackage("..adapter..");

    @ArchTest
    static final ArchRule 인바운드_어댑터는_유스케이스만_호출한다 =
        classes()
            .that().resideInAPackage("..adapter.in..")
            .should().onlyDependOnClassesThat()
                .resideInAnyPackage(
                    "..adapter.in..",
                    "..application.port.in..",
                    "..domain..",
                    "java..",
                    "org.springframework.."
                );
}
```

---

## 🏛️ 어노테이션 규칙

```java
@AnalyzeClasses(packages = "com.example")
class AnnotationRuleTest {

    // @Transactional은 서비스 레이어에만
    @ArchTest
    static final ArchRule Transactional은_서비스에만 =
        noClasses()
            .that().resideOutsideOfPackage("..service..")
            .and().areNotAnnotatedWith(SpringBootApplication.class)
            .should().beAnnotatedWith(Transactional.class)
            .because("@Transactional은 서비스 레이어에만 적용해야 한다");

    // @Repository는 repository 패키지에만
    @ArchTest
    static final ArchRule Repository_어노테이션_위치 =
        classes()
            .that().areAnnotatedWith(Repository.class)
            .should().resideInAPackage("..repository..")
            .because("@Repository 클래스는 repository 패키지에 있어야 한다");

    // Controller는 반드시 @RestController 또는 @Controller
    @ArchTest
    static final ArchRule Controller_어노테이션_필수 =
        classes()
            .that().resideInAPackage("..controller..")
            .and().haveSimpleNameEndingWith("Controller")
            .should().beAnnotatedWith(RestController.class)
                .orShould().beAnnotatedWith(Controller.class);
}
```

---

## 🏛️ 네이밍 규칙

```java
@AnalyzeClasses(packages = "com.example")
class NamingConventionTest {

    @ArchTest
    static final ArchRule 서비스_클래스_네이밍 =
        classes()
            .that().resideInAPackage("..service..")
            .and().areAnnotatedWith(Service.class)
            .should().haveSimpleNameEndingWith("Service");

    @ArchTest
    static final ArchRule Repository_인터페이스_네이밍 =
        classes()
            .that().areAssignableTo(JpaRepository.class)
            .should().haveSimpleNameEndingWith("Repository");

    @ArchTest
    static final ArchRule UseCase_인터페이스_네이밍 =
        classes()
            .that().resideInAPackage("..port.in..")
            .and().areInterfaces()
            .should().haveSimpleNameEndingWith("UseCase");

    @ArchTest
    static final ArchRule DTO_네이밍 =
        classes()
            .that().resideInAPackage("..dto..")
            .should().haveSimpleNameEndingWith("Request")
                .orShould().haveSimpleNameEndingWith("Response")
                .orShould().haveSimpleNameEndingWith("Dto");
}
```

---

## 🏛️ 순환 의존성 금지

```java
@AnalyzeClasses(packages = "com.example")
class CyclicDependencyTest {

    @ArchTest
    static final ArchRule 순환_의존성_없음 =
        slices()
            .matching("com.example.(*)..")
            .should().beFreeOfCycles();
}
```

이 규칙은 `order` 패키지 → `user` 패키지 → `order` 패키지 같은 순환을 탐지합니다.

---

## 💻 실전 적용: 커스텀 아키텍처 규칙

### 서비스가 다른 서비스를 직접 주입받지 않는 규칙

```java
@ArchTest
static final ArchRule 서비스_간_직접_주입_금지 =
    noClasses()
        .that().resideInAPackage("..service..")
        .and().haveSimpleNameEndingWith("Service")
        .should().dependOnClassesThat()
            .resideInAPackage("..service..")
            .and().haveSimpleNameEndingWith("Service")
        .because("서비스는 다른 서비스를 직접 주입받지 말고 " +
                 "UseCase 인터페이스를 통해 호출해야 한다");
```

### 도메인 객체가 Spring 의존 없음

```java
@ArchTest
static final ArchRule 도메인은_스프링_미의존 =
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
            .resideInAnyPackage(
                "org.springframework..",
                "javax.persistence..",
                "jakarta.persistence.."
            )
        .because("도메인 객체는 순수 Java여야 한다");
```

### 규칙 무시(예외) 처리

```java
@ArchTest
static final ArchRule 도메인은_인프라_미의존 =
    noClasses()
        .that().resideInAPackage("..domain..")
        .and().areNotAnnotatedWith(Entity.class)      // JPA Entity는 예외
        .and().areNotAnnotatedWith(Embeddable.class)  // JPA Embeddable은 예외
        .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");
```

---

## 🤔 트레이드오프

### "ArchUnit 테스트가 너무 엄격하면 개발이 불편해지지 않는가?"

규칙을 점진적으로 도입합니다. 처음부터 모든 규칙을 강제하면 팀의 저항이 생깁니다. 가장 중요한 규칙(도메인 → 인프라 의존 금지)부터 시작하고, 팀이 동의한 규칙을 하나씩 추가합니다.

기존 코드에서 규칙을 위반하는 부분은 `@ArchIgnore`로 잠시 제외하고, 리팩터링 이슈를 생성해서 점진적으로 해결합니다.

```java
@ArchIgnore(reason = "레거시 코드 — 이슈 #456에서 리팩터링 예정")
@ArchTest
static final ArchRule 아직_적용_안된_규칙 = ...;
```

### "IDE에서 이미 경고해주지 않는가?"

IDE 경고는 개발자가 직접 확인해야 하며, CI 빌드에서 강제되지 않습니다. ArchUnit은 빌드가 실패하도록 강제하므로 규칙 위반이 배포되는 것을 막습니다.

---

## 📌 핵심 정리

```
ArchUnit이 해결하는 문제:
  아키텍처 규칙이 코드 리뷰에서만 검증됨
  빌드가 통과해도 레이어 의존성이 무너짐
  규칙이 문서에만 있고 코드에서 강제되지 않음

주요 검증 영역:
  레이어 의존성 (도메인 → 인프라 금지)
  패키지 간 의존 방향
  어노테이션 사용 위치 (@Transactional)
  네이밍 컨벤션 (Service, Repository, UseCase)
  순환 의존성

핵심 DSL:
  noClasses().that()...should()      : 금지 규칙
  classes().that()...should()        : 허용/요구 규칙
  layeredArchitecture()              : 레이어 의존성 전체
  slices().beFreeOfCycles()          : 순환 의존성
  resideInAPackage("..service..")    : 패키지 매칭

도입 전략:
  가장 중요한 규칙부터 시작
  기존 위반은 @ArchIgnore + 이슈 등록
  팀 합의 후 규칙 추가
  because()로 규칙의 이유 명시
```

---

## 🤔 생각해볼 문제

**Q1.** 아래 패키지 구조에서 "OrderService가 UserRepository를 직접 참조하면 안 된다"는 규칙을 ArchUnit으로 표현하라.

```
com.example.order.service.OrderService
com.example.user.repository.UserRepository
com.example.user.port.UserFinder (인터페이스)
```

**Q2.** `slices().beFreeOfCycles()` 규칙이 탐지하는 순환 의존성의 예시를 하나 만들고, 이를 어떻게 해소하는지 설명하라.

**Q3.** ArchUnit 테스트를 CI 파이프라인에 포함시킬 때 주의해야 할 점은 무엇인가? 특히 레거시 코드베이스에 처음 도입할 때의 전략을 설명하라.

> 💡 **해설**

**Q1.**

```java
@ArchTest
static final ArchRule OrderService는_UserRepository_직접_참조_금지 =
    noClasses()
        .that().haveFullyQualifiedName("com.example.order.service.OrderService")
        .should().dependOnClassesThat()
            .haveFullyQualifiedName("com.example.user.repository.UserRepository")
        .because("OrderService는 UserRepository를 직접 참조하지 말고 " +
                 "UserFinder 인터페이스를 통해 접근해야 한다");
```

더 일반적으로 order 패키지가 user repository 패키지 전체에 의존하지 않도록:

```java
@ArchTest
static final ArchRule order_service는_user_repository_미의존 =
    noClasses()
        .that().resideInAPackage("com.example.order.service..")
        .should().dependOnClassesThat()
            .resideInAPackage("com.example.user.repository..")
        .because("서비스 간 경계는 Port 인터페이스(UserFinder)를 통해 유지한다");
```

**Q2.**

순환 의존성 예시:

```
com.example.order.service.OrderService
  → com.example.payment.service.PaymentService (결제 처리)
  
com.example.payment.service.PaymentService
  → com.example.order.service.OrderService (주문 상태 업데이트)
```

`order.service` → `payment.service` → `order.service` 순환.

해소 방법:

① **이벤트 기반**: `PaymentService`가 직접 `OrderService`를 호출하는 대신, `PaymentCompleted` 이벤트를 발행하고 `OrderService`가 이벤트를 구독한다.

② **인터페이스 추출**: `order.port` 패키지에 `OrderStatusUpdater` 인터페이스를 만들고 `PaymentService`는 이 인터페이스에만 의존한다.

③ **공통 도메인 패키지**: 양쪽에서 모두 필요한 개념을 `common.domain` 패키지로 추출한다.

**Q3.**

레거시 코드베이스 도입 전략:

**1단계: 현황 파악** — 먼저 ArchUnit을 `@ArchIgnore` 없이 실행해서 현재 위반 수를 파악한다. 위반이 수백 개라면 바로 CI에 포함하면 빌드가 모두 실패한다.

**2단계: 우선순위 결정** — 모든 위반을 한 번에 고치지 말고, 새로운 코드에서의 위반 방지를 1순위로 한다.

**3단계: 점진적 강제** — 기존 위반을 `@ArchIgnore("레거시 — 이슈 #N")`으로 임시 제외하고 CI에 포함한다. 새 코드에서의 위반은 즉시 빌드 실패로 차단된다.

```java
// 일시적 예외 처리 패턴
@ArchTest
static final ArchRule 도메인_인프라_독립 =
    noClasses()
        .that().resideInAPackage("..domain..")
        .and().areNotAnnotatedWith(LegacyViolation.class) // 마이그레이션 중인 클래스 제외
        .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");
```

**4단계: 이슈 트래킹** — `@ArchIgnore`된 항목마다 이슈를 생성하고, 스프린트마다 일정 수를 해소하는 계획을 세운다. `@ArchIgnore`가 영구화되지 않도록 관리한다.

주의할 점: 규칙을 너무 많이 추가하면 팀이 `@ArchIgnore`를 습관적으로 사용하게 된다. 핵심 규칙(5개 이내)부터 시작하는 것이 현실적이다.

---

<div align="center">

**[⬅️ 이전: Property-Based Testing](./02-property-based-testing.md)** | **[다음: TDD Workflow ➡️](./04-tdd-workflow.md)**

</div>
