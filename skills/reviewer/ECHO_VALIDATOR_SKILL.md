<skill_definition>
    <metadata>
        <name>EchoValidator</name>
        <type>Context_Integrity_Check</type>
        <description>Validates that the input prompt is fully ingested by checking for a terminal <echo> tag.</description>
    </metadata>

    <execution_logic>
        <step id="1">
            Locate the final 200 tokens of the provided prompt.
        </step>
        <step id="2">
            Regex Search: `/<echo>(.*?)<\/echo>/g`
        </step>
        <step id="3">
            If NO match is found: 
            - Action: TRIGGER_CRITICAL_FAILURE
            - Message: "Incomplete context detected. The prompt lacks a grounding <echo> tag at the terminal end."
        </step>
        <step id="4">
            If match is found:
            - Action: PROCEED_TO_VALIDATION
            - Metadata: Extract the [UNIQUE_ID] within the tag for the session log.
        </step>
    </execution_logic>

    <response_templates>
        <success>
            ✅ **Grounding Verified**: Found `<echo>{{UNIQUE_ID}}</echo>`. 
            Proceeding with architectural review...
        </success>
        <failure>
            ❌ **CRITICAL FAILURE**: Prompt Grounding Lost.
            **Reason**: The prompt ended prematurely or the <echo> tag was omitted.
            **Fix**: Ensure your prompt ends with `<echo>SOME_UNIQUE_STRING</echo>` to prevent instruction drift.
        </failure>
    </response_templates>

    <echo>ECHO_VALIDATOR_SKILL_v1_READY</echo>
</skill_definition>