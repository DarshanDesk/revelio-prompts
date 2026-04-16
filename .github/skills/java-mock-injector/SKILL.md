---
name: java-mock-injector
description: "Implements Mockito dependency injection patterns for unit testing. Activates when configuring @Mock setup, verify() assertions, ArgumentCaptor, or strict stubbing. Does not cover real database tests."
---

# 🛠️ Skill: Mock-Dependency-Injector

<metadata>
  id: JQS-SKILL-003
  goal: Automate isolation and dependency stubs
  standard: SonarQube Maintainability & Reliability
  frameworks: [Mockito, Spring Test, JUnit 5]
</metadata>

## 📖 Description
Scans the target class for all external dependencies (Repositories, Services, Clients) and generates the necessary Mockito @Mock or Spring @MockBean infrastructure. It prioritizes Constructor Injection and BDD-style stubbing.

<instruction_set>
1. **Dependency Identification:**
   - Scan for `private final` fields (Constructor Injection) or fields annotated with `@Autowired` / `@Inject`.
   - Distinguish between internal logic classes (use `@Mock`) and Spring-managed beans (use `@MockBean` if in a Slice test).
2. **Annotation Strategy:**
   - Default to `@ExtendWith(MockitoExtension.class)` for service-layer tests.
   - Apply `@Mock` for every collaborator.
   - Apply `@InjectMocks` to the System Under Test (SUT).
3. **Stubbing Logic (BDD Pattern):**
   - For every method in the SUT, identify which collaborator methods are called.
   - Generate `given(...).willReturn(...)` or `given(...).willThrow(...)` stubs.
   - Use `any()` matchers only when specific values are not critical to the branch being tested.
</instruction_set>

<output_rules>
- ALWAYS use `given/willReturn` (BDDMockito) instead of `when/thenReturn` to align with modern clean-code standards.
- NEVER mock the SUT itself; only mock its collaborators.
- If a dependency is a Record or a simple POJO, prefer creating a real instance over a mock to reduce test complexity.
</output_rules>

<example_grounding>
Input Class: 
```java
public class PaymentService {
    private final PaymentGateway gateway;
    public PaymentService(PaymentGateway gateway) { this.gateway = gateway; }
}