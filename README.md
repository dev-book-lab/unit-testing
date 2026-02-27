<div align="center">

# 🧪 Unit Testing: Principles to Practice

**"테스트를 작성하는 것과, 올바른 테스트를 작성하는 것은 다르다"**

<br/>

> *"테스트가 없는 코드는 레거시다. 하지만 잘못된 테스트가 있는 코드는 더 위험한 레거시다"*

안티패턴 Before부터 올바른 패턴 After까지  
**왜 이렇게 작성해야 하는가** 라는 질문으로 단위 테스트의 원리를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![JUnit5](https://img.shields.io/badge/JUnit-5-25A162?style=flat-square&logo=junit5&logoColor=white)](https://junit.org/junit5/)
[![Docs](https://img.shields.io/badge/Docs-44개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

단위 테스트에 관한 자료는 많습니다. 하지만 대부분은 **"어떻게 작성하는가"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "테스트는 AAA 패턴으로 작성하세요" | AAA가 깨지는 순간, 어떤 냄새가 나는가 |
| "Mock을 사용하세요" | Stub, Spy, Fake 중 이 상황에서 무엇을 써야 하는가 |
| "커버리지를 높이세요" | 커버리지 90%인데 왜 버그가 생기는가 |
| "테스트하기 좋은 설계를 하세요" | 테스트가 어렵다는 신호가 설계 문제를 어떻게 가리키는가 |
| 이론 나열 | Before 코드 + After 코드 + 왜 달라야 하는지 이유 실측 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Testing Fundamentals](https://img.shields.io/badge/🔹_Fundamentals-What_Is_a_Unit-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./testing-fundamentals/01-what-is-a-unit.md)
[![Good Tests](https://img.shields.io/badge/🔹_Good_Tests-Single_Assert_Principle-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./anatomy-of-good-tests/01-single-assert-principle.md)
[![Mocking](https://img.shields.io/badge/🔹_Mocking-Test_Doubles_Taxonomy-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./mocking-strategies/01-test-doubles-taxonomy.md)
[![Testable Design](https://img.shields.io/badge/🔹_Testable_Design-DI_for_Testability-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./testable-design/01-dependency-injection-for-testability.md)
[![Integration](https://img.shields.io/badge/🔹_Integration-Integration_Test_Scope-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./integration-testing/01-integration-test-scope.md)
[![Anti-patterns](https://img.shields.io/badge/🔹_Anti--patterns-Test_Logic_in_Production-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./test-anti-patterns/01-test-logic-in-production.md)
[![Advanced](https://img.shields.io/badge/🔹_Advanced-Mutation_Testing-4CAF50?style=for-the-badge&logo=junit5&logoColor=white)](./advanced-topics/01-mutation-testing.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 테스팅 기초 (Testing Fundamentals)

> **핵심 질문:** "좋은 테스트"와 "있는 테스트"는 무엇이 다른가?

<details>
<summary><b>단위 테스트가 존재하는 이유와 올바른 첫걸음 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. What Is a Unit?](./testing-fundamentals/01-what-is-a-unit.md) | "단위"의 정의가 팀마다 다른 이유, 고전파 vs 런던파 관점 차이 |
| [02. The Test Pyramid](./testing-fundamentals/02-the-test-pyramid.md) | Unit / Integration / E2E 비율의 근거, 피라미드가 뒤집히면 생기는 일 |
| [03. AAA Pattern](./testing-fundamentals/03-aaa-pattern.md) | Arrange-Act-Assert 구조가 깨지는 신호, Given-When-Then과의 차이 |
| [04. Test Naming Conventions](./testing-fundamentals/04-test-naming-conventions.md) | `testSave()` vs `save_whenDuplicate_throwsException()` — 이름이 문서가 되는 방법 |
| [05. Coverage Myths](./testing-fundamentals/05-coverage-myths.md) | 커버리지 100%인데 버그가 생기는 이유, 커버리지가 측정하지 못하는 것 |
| [06. FIRST Principles](./testing-fundamentals/06-first-principles.md) | Fast / Isolated / Repeatable / Self-validating / Timely — 각 원칙이 깨졌을 때의 증상 |

</details>

<br/>

### 🔹 좋은 테스트의 해부 (Anatomy of Good Tests)

> **핵심 질문:** 테스트 코드도 코드다. 어떻게 읽기 쉽고 유지보수하기 쉽게 만드는가?

<details>
<summary><b>가독성 높고 신뢰할 수 있는 테스트를 만드는 구체적 기법 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Single Assert Principle](./anatomy-of-good-tests/01-single-assert-principle.md) | 단일 단언 원칙의 의미, 여러 단언이 필요할 때 어떻게 분리하는가 |
| [02. Test Data Builders](./anatomy-of-good-tests/02-test-data-builders.md) | 복잡한 픽스처를 Builder 패턴으로 정리하는 방법, 가독성 변화 비교 |
| [03. Boundary Testing](./anatomy-of-good-tests/03-boundary-testing.md) | 경계값 분석, Off-by-One 에러를 테스트로 잡는 방법 |
| [04. Parameterized Tests](./anatomy-of-good-tests/04-parameterized-tests.md) | `@ParameterizedTest`, `@CsvSource`, `@MethodSource` — 중복 없이 케이스 확장 |
| [05. Fixtures & SetUp](./anatomy-of-good-tests/05-fixtures-and-setup.md) | `@BeforeEach`의 올바른 범위, 공유 픽스처가 만드는 숨겨진 결합 |
| [06. Meaningful Assertions](./anatomy-of-good-tests/06-meaningful-assertions.md) | 실패 메시지를 읽으면 원인을 알 수 있는 단언 작성법, AssertJ 활용 |

</details>

<br/>

### 🔹 목킹 전략 (Mocking Strategies)

> **핵심 질문:** Stub, Mock, Fake — 이 세 가지를 구분하지 못하면 테스트가 거짓말을 한다

<details>
<summary><b>테스트 대역의 올바른 선택과 Mockito 실전 활용 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Test Doubles Taxonomy](./mocking-strategies/01-test-doubles-taxonomy.md) | Dummy / Stub / Spy / Mock / Fake 5가지 분류, 언제 무엇을 선택하는가 |
| [02. Stub vs Mock](./mocking-strategies/02-stub-vs-mock.md) | 상태 검증 vs 행동 검증의 트레이드오프, 잘못된 Mock 사용이 만드는 취약한 테스트 |
| [03. Mockito Best Practices](./mocking-strategies/03-mockito-best-practices.md) | `@Mock` / `@InjectMocks` / `@Captor` 올바른 사용, `when().thenReturn()` vs `doReturn()` |
| [04. Partial Mocking with Spy](./mocking-strategies/04-partial-mocking-with-spy.md) | `@Spy`가 필요한 상황과 함정, Spy가 필요하다면 설계를 의심하라 |
| [05. Argument Matchers](./mocking-strategies/05-argument-matchers.md) | `any()` 남용이 만드는 거짓 통과, `ArgumentCaptor`로 정확하게 검증하기 |
| [06. Verify Wisely](./mocking-strategies/06-verify-wisely.md) | `verify()`를 언제 써야 하고 언제 쓰면 안 되는가, 과도한 검증의 문제 |
| [07. Fakes over Mocks](./mocking-strategies/07-fakes-over-mocks.md) | In-memory Repository가 Mockito Repository보다 나은 경우, Fake의 장단점 |

</details>

<br/>

### 🔹 테스트 가능한 설계 (Testable Design)

> **핵심 질문:** 테스트가 어렵다는 느낌은 설계가 나쁘다는 신호인가?

<details>
<summary><b>테스트 용이성을 높이는 설계 원칙과 리팩터링 패턴 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DI for Testability](./testable-design/01-dependency-injection-for-testability.md) | 의존성 주입이 테스트에 미치는 영향, 생성자 주입 vs 필드 주입의 테스트 비용 차이 |
| [02. Avoiding Static Methods](./testable-design/02-avoiding-static-methods.md) | 정적 메서드가 테스트를 망치는 이유, `PowerMock` 없이 해결하는 방법 |
| [03. Interface Segregation](./testable-design/03-interface-segregation.md) | 작은 인터페이스가 Stub을 쉽게 만드는 이유, ISP와 테스트 용이성의 관계 |
| [04. Humble Object Pattern](./testable-design/04-humble-object-pattern.md) | UI / 외부 시스템과 로직을 분리해 테스트 가능한 영역을 최대화하는 방법 |
| [05. Pure Functions First](./testable-design/05-pure-functions-first.md) | 부수효과 없는 로직의 테스트 용이성, 도메인 모델을 순수하게 유지하는 전략 |
| [06. Ports and Adapters](./testable-design/06-ports-and-adapters.md) | Hexagonal Architecture에서 테스트 경계를 어떻게 그리는가 |

</details>

<br/>

### 🔹 통합 테스팅 (Integration Testing)

> **핵심 질문:** 단위 테스트가 모두 통과했는데 왜 시스템이 망가지는가?

<details>
<summary><b>경계를 가로지르는 테스트 — 데이터베이스, API, 스프링 컨텍스트 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Integration Test Scope](./integration-testing/01-integration-test-scope.md) | 무엇을 통합 테스트로 검증해야 하는가, 단위 테스트와의 역할 분리 |
| [02. Database Testing with Testcontainers](./integration-testing/02-database-testing-testcontainers.md) | H2 대신 실제 DB로 테스트하기, Testcontainers 설정과 비용 |
| [03. REST API Testing](./integration-testing/03-rest-api-testing.md) | `MockMvc` vs `WebTestClient` vs `RestAssured` 선택 기준과 비교 |
| [04. Spring Context Slicing](./integration-testing/04-spring-context-slicing.md) | `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`의 범위와 올바른 선택 |
| [05. Test Transaction Management](./integration-testing/05-test-transaction-management.md) | `@Transactional` 테스트의 롤백 함정, 실제 커밋을 검증해야 할 때 |
| [06. Contract Testing](./integration-testing/06-contract-testing.md) | Consumer-Driven Contracts, Pact로 서비스 간 계약을 테스트하는 방법 |

</details>

<br/>

### 🔹 테스트 안티패턴 (Test Anti-patterns)

> **핵심 질문:** 테스트가 있는데도 코드 변경이 두렵다면 — 무엇이 잘못된 것인가?

<details>
<summary><b>신뢰를 갉아먹는 7가지 안티패턴과 리팩터링 방법 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Test Logic in Production](./test-anti-patterns/01-test-logic-in-production.md) | `if (isTest)` 분기, 테스트 전용 생성자 — 테스트가 프로덕션을 오염시키는 패턴 |
| [02. Sleepy Tests](./test-anti-patterns/02-sleepy-tests.md) | `Thread.sleep()`으로 비동기를 검증하는 문제, `Awaitility`로 안전하게 대체하기 |
| [03. Flickering Tests](./test-anti-patterns/03-flickering-tests.md) | 불규칙하게 실패하는 테스트의 4가지 원인과 각각의 해결 전략 |
| [04. Overspecified Tests](./test-anti-patterns/04-overspecified-tests.md) | 구현 세부사항을 검증하는 테스트, 리팩터링할 때마다 테스트가 깨지는 이유 |
| [05. Test Code Duplication](./test-anti-patterns/05-test-code-duplication.md) | 반복되는 `setUp`, 반복되는 단언 — DRY가 테스트에서 의미하는 것 |
| [06. Hidden Test Dependencies](./test-anti-patterns/06-hidden-test-dependencies.md) | 테스트 실행 순서에 의존하는 테스트, 공유 상태가 만드는 시한폭탄 |
| [07. Assertion Roulette](./test-anti-patterns/07-assertion-roulette.md) | 실패 시 어떤 단언이 틀렸는지 알 수 없는 테스트, SoftAssertions 활용 |

</details>

<br/>

### 🔹 고급 주제 (Advanced Topics)

> **핵심 질문:** 테스트를 "잘 쓴다"는 것을 넘어, 테스트가 설계를 이끌 수 있는가?

<details>
<summary><b>뮤테이션 테스팅, 속성 기반 테스팅, TDD — 테스트의 다음 단계 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Mutation Testing](./advanced-topics/01-mutation-testing.md) | PIT(PITest)로 테스트 품질 측정하기, 커버리지가 잡지 못하는 결함 찾기 |
| [02. Property-Based Testing](./advanced-topics/02-property-based-testing.md) | `jqwik`으로 경계를 자동 탐색, 예제 기반 테스트와 속성 기반 테스트의 차이 |
| [03. Architecture Testing](./advanced-topics/03-architecture-testing.md) | ArchUnit으로 레이어 의존성 규칙을 테스트로 강제하는 방법 |
| [04. TDD Workflow](./advanced-topics/04-tdd-workflow.md) | Red → Green → Refactor 사이클의 실전 리듬, 언제 테스트를 먼저 쓰고 언제 나중에 쓰는가 |
| [05. Test-Driven Design](./advanced-topics/05-test-driven-design.md) | 테스트를 먼저 쓰면 설계가 어떻게 달라지는가, "테스트 가능성 = 설계 품질"의 의미 |
| [06. Testing Legacy Code](./advanced-topics/06-testing-legacy-code.md) | 테스트 없는 코드에 테스트를 추가하는 순서, Seam 개념과 Characterization Test |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 단위 테스트를 처음 배우는 개발자 / 테스트 습관을 만들고 싶은 분 (2~3주)</b></summary>

<br/>

**Week 1 — 테스트의 본질 이해**
```
what-is-a-unit
the-test-pyramid
aaa-pattern
test-naming-conventions
```

**Week 2 — 올바른 테스트 작성**
```
single-assert-principle
meaningful-assertions
test-doubles-taxonomy
stub-vs-mock
```

**Week 3 — 안티패턴 제거**
```
overspecified-tests
flickering-tests
hidden-test-dependencies
test-code-duplication
```

</details>

<details>
<summary><b>🔵 테스트를 쓰고 있지만 신뢰가 가지 않는 개발자 (4~6주)</b></summary>

<br/>

```
anatomy-of-good-tests 전체
→ mocking-strategies 전체 (Stub, Mock, Fake 완전히 구분)
→ test-anti-patterns 전체 (내 테스트에서 냄새 찾기)
→ testable-design 전체 (설계부터 바꾸기)
```

</details>

<details>
<summary><b>🔴 테스트가 설계를 이끄는 수준을 목표로 하는 개발자 (2~3개월)</b></summary>

<br/>

```
전체 순서대로 + 예제 코드 직접 리팩터링
→ advanced-topics 챕터로 마무리
→ 실제 프로젝트에 mutation testing 적용
→ legacy code에 characterization test 추가하며 체화
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 패턴이 중요한가** | 이 패턴이 없을 때 생기는 실제 문제 |
| 😱 **안티패턴 (Before)** | 흔히 쓰는 잘못된 코드 + 무엇이 문제인지 |
| ✨ **올바른 패턴 (After)** | 개선된 코드 + 왜 이것이 더 나은지 |
| 💻 **실전 적용** | 복사해서 바로 쓸 수 있는 예제, JUnit5 + AssertJ + Mockito |
| 🤔 **트레이드오프** | 이 패턴의 한계와 적용하지 말아야 할 상황 |
| 📌 **핵심 정리** | 한 화면 요약 |

---

## 🙏 Reference

- [Unit Testing: Principles, Practices, and Patterns — Vladimir Khorikov](https://www.manning.com/books/unit-testing)
- [Growing Object-Oriented Software, Guided by Tests — Freeman, Pryce](http://www.growing-object-oriented-software.com/)
- [Test-Driven Development: By Example — Kent Beck](https://www.oreilly.com/library/view/test-driven-development/0321146530/)
- [Working Effectively with Legacy Code — Michael Feathers](https://www.oreilly.com/library/view/working-effectively-with/0131177052/)
- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"테스트를 작성하는 것과, 올바른 테스트를 작성하는 것은 다르다"*

</div>
