---
description: "Generate JUnit 5 test scaffolding for Java services with Mockito mocks and AssertJ assertions."
---

<orchestration_request>
  <agent_context>
    Initialize: JavaQualitySentinel (JQS)
    Goal: Generate high-hygiene unit tests for the targeted source code.
    Standard: SonarQube Quality Model + 90% Branch Coverage.
  </agent_context>

  <active_skills>
    - JQS-SKILL-001: Spring-Slice-Architect (Determine Test Isolation)
    - JQS-SKILL-002: Mock-Dependency-Injector (Setup Collaborator Mocks)
    - JQS-SKILL-003: Branch-Exhaustion-Generator (Target 90% Coverage)
  </active_skills>

  <input_data>
    <source_file>${file}</source_file>
    <project_context>Spring Boot 3, Java 17, AssertJ</project_context>
  </input_data>

  <execution_logic>
    1. READ <source_file> and identify all logical branches (if/else, switch, Optional).
    2. INVOKE JQS-SKILL-001 to select the narrowest Spring Test Slice.
    3. INVOKE JQS-SKILL-002 to generate the @Mock and @InjectMocks infrastructure.
    4. INVOKE JQS-SKILL-003 to write @Test methods for every identified branch.
    5. AUDIT the output against the "Self-Correction Hook" criteria.
  </execution_logic>

  <non_negotiable_standards>
    - **Hygiene Gate:** Reject any code with < 90% branch coverage.
    - **Technical Stack:** Java 17, Spring Boot 3, AssertJ, Mockito.
    - **SonarQube Compliance:** Zero Blocker/Critical smells (Reliability & Security focus).
  </non_negotiable_standards>

  <output_format>
    <thinking>
      Show the branch mapping and estimated coverage calculation here.
    </thinking>
    <hygiene_report>
      Summarize: Coverage %, Mocks used, and Sonar pillars addressed.
    </hygiene_report>
    <java_test_suite>
      [The final hygienic test code goes here]
    </java_test_suite>
  </output_format>
</orchestration_request>