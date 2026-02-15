# 📋 Universal Audit Specification
**Status:** MANDATORY | **Enforcement:** NON-SKIPBABLE

<post_execution_hook id="PQS-HYGIENE-GATE-01">
  <metadata>
    <type>Lifecycle-Enforced-Gate</type>
    <assurance_level>100% Mandatory</assurance_level>
    <target_metrics>90% Coverage / 100% Pass Rate</target_metrics>
  </metadata>
<mandatory_hook_logic>
  <enforcement_level>SYSTEM_MANDATORY_BYPASS_PROTECTION_ENABLED</enforcement_level>

  <phase id="1_STATIC_AUDIT">
    <thinking>
      - Analyze the AST (Abstract Syntax Tree) or logical structure of the source file.
      - Identify all conditional branches (if, else, try, catch, match).
      - Cross-reference these branches against the generated test coverage.
      - Mathematically determine if the 90% threshold is met.
    </thinking>
    <action>Verify branch exhaustion vs. Source Code.</action>
    <bypass_protection>
      - If coverage < 90%, the "Pass" flag is programmatically disabled.
      - The Agent is forbidden from proceeding to Phase 2 until 90% is achieved.
    </bypass_protection>
  </phase>

  <phase id="2_DYNAMIC_HEALING">
    <thinking>
      - Identify the correct OS-agnostic and language-agnostic command (pytest, mvn test).
      - Anticipate potential environment failures (async loops, mock timeouts).
      - Prepare to parse the stderr for specific traceback patterns.
    </thinking>
    <action>Execute tests and auto-repair functional failures.</action>
    <bypass_protection>
      - Any non-zero exit code from the test runner triggers an immediate REPAIR loop.
      - The "Complete" status is withheld until all tests report "PASSED."
    </bypass_protection>
  </phase>

  <phase id="3_TELEMETRY">
    <thinking>
      - Aggregate the data: input prompt, methods targeted, and coverage deltas.
      - Format the audit summary into the standardized JSON schema.
      - Verify the write-path to .github/logs/ is accessible.
    </thinking>
    <action>Persist Audit Summary to .github/logs/.</action>
    <bypass_protection>
      - The Agent cannot terminate the session until a valid JSON log is persisted.
      - Missing audit metadata will result in a "Log Failure" and session restart.
    </bypass_protection>
  </phase>

  <global_bypass_rules>
    <rule>NO_OPTIONAL_EXECUTION: This hook must execute for every 'test-gen' task.</rule>
    <rule>NO_USER_BYPASS: The developer cannot override the 90% gate via prompt instructions.</rule>
    <rule>LOG_INTEGRITY: If the log is not written, the code generation is considered 'Failed'.</rule>
  </global_bypass_rules>
</mandatory_hook_logic>
</post_execution_hook>