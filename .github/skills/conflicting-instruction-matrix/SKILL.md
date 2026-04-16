---
name: conflicting-instruction-matrix
description: "Detects conflicting instructions across agent configuration files. Activates when auditing agent configs for instruction contradictions or drift detection."
---

<skill_definition>
    <metadata>
        <name>ConflictingInstructionMatrix</name>
        <type>Hierarchical_Consistency_Auditor</type>
        <version>1.2.0</version>
        <description>Detects quantitative and qualitative discrepancies between Parent (Agent) and Child (Skill/Prompt) instructions.</description>
    </metadata>

    <audit_vectors>
        <vector id="NUMERICAL_DRIFT">
            <description>Identifying mismatched thresholds, percentages, or counts.</description>
            <logic>If Agent[Metric_X] != Skill[Metric_X], flag the discrepancy.</logic>
            <example>Agent: "Maintain 80% coverage" vs. Skill: "Enforce 95% coverage."</example>
        </vector>
        <vector id="ROLE_OVERSHADOWING">
            <description>When a skill attempts to redefine a persona already set by the Agent.</description>
            <logic>If Skill[Persona] contradicts Agent[Persona], prioritize the Agent file.</logic>
        </vector>
        <vector id="SCOPE_CREEP">
            <description>When a skill requests access to tools or data not authorized in the Agent manifest.</description>
        </vector>
    </audit_vectors>

    <execution_steps>
        <step>1. **Cross-File Ingestion**: Read the `agent.md` file and all referenced `SKILL.md` files.</step>
        <step>2. **Entity Extraction**: Identify all numerical constants (%, tokens, counts) across all files.</step>
        <step>3. **Conflict Identification**: Run a parity check. Highlight any values where the Skill is more or less restrictive than the Agent.</step>
        <step>4. **Hierarchy Resolution**: Apply the "Law of the Parent": The Agent file is the Source of Truth unless the Skill is explicitly marked as an "Override."</step>
    </execution_steps>

    <output_schema>
        <conflict_alert>
            <file_a>agent.md</file_a>
            <file_b>skill_name.md</file_b>
            <discrepancy_type>QUANTITATIVE_MISMATCH</discrepancy_type>
            <finding>"Agent defines coverage at 80%, but Skill enforces 95%."</finding>
            <reconciliation_action>Synchronize Skill to 80% or update Agent to 95% to maintain a unified goal.</reconciliation_action>
        </conflict_alert>
    </output_schema>

    <echo>CONFLICTING_INSTRUCTION_MATRIX_v1_READY</echo>
</skill_definition>