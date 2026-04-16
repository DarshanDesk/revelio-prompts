<learning_log_metadata>
    <agent_id>PROMPT_ARCHITECT_V2</agent_id>
    <last_updated>2026-04-11</last_updated>
    <description>
        This log serves as the persistent memory for the Prompt Reviewer Agent. 
        It tracks failed prompts (Anti-Patterns) and successful refactors (Gold Standard).
    </description>
</learning_log_metadata>

<knowledge_base>
    <entry id="LOG-001" timestamp="2026-03-17T10:00:00Z">
        <issue_type>Instruction Contradiction</issue_type>
        <problematic_prompt>
            "Generate a detailed 2000-word report but keep it under 2 paragraphs."
        </problematic_prompt>
        <resolution_pattern>
            Prioritize specific constraints (paragraph count) over qualitative descriptors (detailed). 
            Refactored to: "Generate a high-density 2-paragraph summary covering [X, Y, Z] key points."
        </resolution_pattern>
        <status>RESOLVED</status>
    </entry>

    <entry id="LOG-002" timestamp="2026-03-17T14:30:00Z">
        <issue_type>Missing Echo Command</issue_type>
        <problematic_prompt>
            [User submitted a 4000-token prompt without an <echo> tag at the end]
        </problematic_prompt>
        <resolution_pattern>
            The Agent failed to process the final constraints due to context window saturation. 
            LEARNING: Always enforce the <echo> tag to ensure the tail-end of the prompt is grounded.
        </resolution_pattern>
        <status>ENFORCED</status>
    </entry>

    <entry id="LOG-003" timestamp="2026-04-11T00:00:00Z">
        <issue_type>XML-MALFORM — Mismatched Closing Tag Inside Logic Block</issue_type>
        <defect_id>XML-MALFORM-001</defect_id>
        <affected_file>agents/dbt/risk-data-synthesizer.agent.md</affected_file>
        <problematic_pattern>
            A &lt;logic&gt; block was closed with &lt;/skill&gt; instead of &lt;/logic&gt;:
            &lt;skill id="mock_realtime_updates"&gt;
                &lt;logic&gt;Generate 'Analyst' CSVs...&lt;/skill&gt;  &lt;!-- WRONG: should be &lt;/logic&gt; --&gt;
            &lt;/skill&gt;
        </problematic_pattern>
        <resolution_pattern>
            Phase 3 refactor eliminated the defect entirely by replacing all inline &lt;logic&gt; blocks
            with external skill_file= references. The consolidated skill is now:
            &lt;skill id="synthetic_data_factory" skill_file="skills/dbt/synthetic-data-factory.md" /&gt;
            LEARNING: When authoring XML-wrapped agents, every &lt;logic&gt; open tag must close with
            &lt;/logic&gt; — never with &lt;/skill&gt;. Run `grep -c "&lt;logic&gt;"` post-edit to verify
            zero inline logic blocks remain. Prefer externalising logic to skill files over inline blocks
            to avoid this class of malformation entirely.
        </resolution_pattern>
        <prevention_rule>
            Any agent file with inline &lt;logic&gt; blocks is a candidate for XML-MALFORM defects.
            ENFORCE: `grep -c "&lt;logic&gt;"` → 0 on all agent files as a suite health check.
        </prevention_rule>
        <status>RESOLVED — Phase 3 (2026-04-11)</status>
    </entry>

    <entry id="LOG-004" timestamp="2026-04-11T00:00:00Z">
        <issue_type>COLUMN-DRIFT — Phantom Column Names in Agent SQL Logic</issue_type>
        <defect_id>COLUMN-DRIFT-001 / COLUMN-DRIFT-002</defect_id>
        <affected_files>
            agents/dbt/financial-audit-pro.agent.md (COLUMN-DRIFT-001 — CRITICAL)
            agents/dbt/custom_instructions.md (COLUMN-DRIFT-002 — suite-wide escalation)
        </affected_files>
        <problematic_pattern>
            Both files referenced SCD2 validity columns that do not exist in the canonical DDL:
            FORBIDDEN: VALID_FROM_DT, VALID_TO_DT
            FORBIDDEN: IS_CURRENT = 1  (integer comparison against a BOOLEAN column)
            These names appeared in perceive stage actions, rationale blocks, and audit logic.
            Impact: Runtime failure on every audit query — columns resolve to NULL or error.
        </problematic_pattern>
        <canonical_authority>
            Single source of truth: .github/context-cache/SCHEMA.md § SCD2 Column Name Authority
            CORRECT column names (DuckDB):
                EFF_FROM    TIMESTAMP  — SCD2 validity start (inclusive)
                EFF_TO      TIMESTAMP  — SCD2 validity end (exclusive; NULL = open record)
                IS_CURRENT  BOOLEAN    — Filter: WHERE IS_CURRENT = TRUE
        </canonical_authority>
        <resolution_pattern>
            Phase 0 (emergency): replaced VALID_FROM_DT/VALID_TO_DT → EFF_FROM/EFF_TO
            and IS_CURRENT = 1 → IS_CURRENT = TRUE in all perceive/rationale blocks.
            Phase 1A: SCHEMA.md created as the permanent suite-wide column name authority.
            Phase 3: All agent skill_file= references now inherit the column contract from
            the skill files, which enforce EFF_FROM/EFF_TO/IS_CURRENT BOOLEAN throughout.
            LEARNING: Before authoring any SQL or DDL in an agent file, load
            SCHEMA.md § SCD2 Column Name Authority. Run:
            `grep -c "VALID_FROM_DT\|VALID_TO_DT"` → 0 on all agent files.
            `grep -c "IS_CURRENT = 1\|IS_CURRENT=1"` → 0 on all agent files (DuckDB context).
        </resolution_pattern>
        <prevention_rule>
            ENFORCE: SCHEMA.md § SCD2 Column Name Authority is the single source of truth.
            Any occurrence of VALID_FROM_DT, VALID_TO_DT, dbt_valid_from, dbt_valid_to,
            or IS_CURRENT = 1 in any agent, skill, or instruction file is a DEFECT.
            Add this grep to every suite health check run.
        </prevention_rule>
        <status>RESOLVED — Phase 0 + Phase 3 (2026-04-11)</status>
    </entry>
</knowledge_base>

<global_anti_patterns>
    <item>Unstructured "Wall of Text" instructions without XML/Markdown headers.</item>
    <item>Vague tone requirements (e.g., "Make it sound professional") without specific vocabulary constraints.</item>
    <item>Omitting Input/Output JSON schemas for data-heavy tasks.</item>
</global_anti_patterns>

<echo>LEARNING_LOG_SCHEMA_v1_LOADED</echo>