# 🛠️ Skill: Branch-Exhaustion-Generator

<metadata>
  id: JQS-SKILL-002
  target_standard: SonarQube Reliability & 90% Coverage
  frameworks: [JUnit 5, AssertJ, Mockito]
</metadata>

## 📖 Description
Analyzes Java method logic to identify every logical path.

<instruction_set>
  1. **Identify Decision Points:** Scan for `if`, `else`, `switch`, `case`, and `ternary` operators.
  2. **Exception Mapping:** For every method call that `throws`, create a test case that induces that exception using `doThrow()`.
  3. **Null-Safety:** Generate a test case for every nullable input parameter.
</instruction_set>

<output_schema>
  - A @ParameterizedTest if multiple values trigger the same branch.
  - A @DisplayName that explicitly mentions the SonarQube Rule being satisfied.
</output_schema>

<example_grounding>
  Input: `if (user == null) throw new IllegalArgumentException();`
  Output: 
  @Test
  @DisplayName("Should throw IAE when user is null (Reliability Rule)")
  void validateUser_Null_ThrowsException() { ... }
</example_grounding>