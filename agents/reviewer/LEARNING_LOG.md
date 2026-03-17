<learning_log_metadata>
    <agent_id>PROMPT_ARCHITECT_V2</agent_id>
    <last_updated>2026-03-17</last_updated>
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
</knowledge_base>

<global_anti_patterns>
    <item>Unstructured "Wall of Text" instructions without XML/Markdown headers.</item>
    <item>Vague tone requirements (e.g., "Make it sound professional") without specific vocabulary constraints.</item>
    <item>Omitting Input/Output JSON schemas for data-heavy tasks.</item>
</global_anti_patterns>

<echo>LEARNING_LOG_SCHEMA_v1_LOADED</echo>