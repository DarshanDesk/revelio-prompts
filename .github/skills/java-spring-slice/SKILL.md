---
name: java-spring-slice
description: "Implements Spring slice test architecture for focused layer testing. Activates when writing @WebMvcTest, @DataJpaTest, or Spring Security test configurations. Does not cover full @SpringBootTest integration tests."
---

# 🛠️ Skill: Spring-Slice-Architect

<metadata>
  id: JQS-SKILL-001
  goal: Optimize Test Context & Speed
  standard: SonarQube Maintainability
</metadata>

## 📖 Description
Determines the narrowest possible Spring Test Slice for a given class to prevent full ApplicationContext loads, ensuring tests are fast, isolated, and maintainable.

<instruction_set>
1. **Analyze SUT (System Under Test):** - If class has `@RestController`: Use `@WebMvcTest(ClassName.class)`.
   - If class has `@Repository`: Use `@DataJpaTest`.
   - If class has `@Service` or logic-only: Use pure `JUnit 5` with `@ExtendWith(MockitoExtension.class)`.
2. **Infrastructure Mocking:**
   - For `@WebMvcTest`, automatically include `MockMvc` autoconfiguration.
   - For `@DataJpaTest`, include `TestEntityManager`.
3. **Security Context:** - If Spring Security is present, generate `@WithMockUser` annotations to satisfy Security pillar rules.
</instruction_set>

<output_rules>
- NEVER use `@SpringBootTest` unless explicitly requested for integration testing.
- ALWAYS use constructor injection for mocks.
- Ensure `@Tag("unit")` or `@Tag("slice")` is added for CI/CD filtering.
</output_rules>

<example_grounding>
Input: `AccountController.java`
Output:
```java
@WebMvcTest(AccountController.class)
class AccountControllerTest {
    @Autowired
    private MockMvc mockMvc;
    @MockBean
    private AccountService accountService;
}
```
</example_grounding>