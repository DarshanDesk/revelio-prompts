<skill_definition>
    <metadata>
        <name>ContradictionMatrix</name>
        <type>Logical_Consistency_Validator</type>
        <version>1.1.0</version>
        <description>Identifies mutually exclusive instructions or conflicting constraints within a prompt.</description>
    </metadata>

    <conflict_categories>
        <category id="TONE_VS_FORMAT">
            <description>Conflict between the persona's voice and the required output structure.</description>
            <example>Persona: "Friendly Assistant" vs. Output: "Strict JSON Schema with no prose."</example>
        </category>
        <category id="BREADTH_VS_DEPTH">
            <description>Conflict between length constraints and detail requirements.</description>
            <example>Rule: "Be extremely concise" vs. Instruction: "Explain every edge case in depth."</example>
        </category>
        <category id="RAG_VS_CREATIVITY">
            <description>Conflict between grounding data and creative freedom.</description>
            <example>Rule: "Only use provided context" vs. Instruction: "Speculate on future trends not in the text."</example>
        </category>
    </conflict_categories>

    <validation_algorithm>
        <step>1. Extract all 'Constraints' and 'Instructions' into a temporary array.</step>
        <step>2. Perform a pair-wise comparison of every rule against the output schema.</step>
        <step>3. Cross-reference against the 'LEARNING_LOG.md' for known logic failures.</step>
        <step>4. Assign a 'Consistency Score' (0-100%).</step>
    </validation_algorithm>

    <error_reporting_schema>
        <conflict_report>
            <severity>CRITICAL | WARNING | INFO</severity>
            <conflict_pair>
                <instruction_a>Source text of first rule</instruction_a>
                <instruction_b>Source text of contradicting rule</instruction_b>
            </conflict_pair>
            <resolution_strategy>How to merge or prioritize these rules.</resolution_strategy>
        </conflict_report>
    </error_reporting_schema>

    <echo>CONTRADICTION_MATRIX_SKILL_v1_READY</echo>
</skill_definition>