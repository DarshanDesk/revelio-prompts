<skill_definition>
    <metadata>
        <name>PatternAlignment</name>
        <type>Structural_Prompt_Optimization</type>
        <version>1.3.0</version>
        <description>Enforces industry-standard prompting patterns (CoT, Few-Shot, ReAct) based on task complexity.</description>
    </metadata>

    <pattern_library>
        <pattern id="CoT">
            <name>Chain-of-Thought</name>
            <trigger_condition>Complex logic, math, or multi-step reasoning tasks.</trigger_condition>
            <required_syntax>"Let's think step-by-step" or a <reasoning> block.</reasoning>
            <success_criteria>The model outputs its internal monologue before the final answer.</success_criteria>
        </pattern>

        <pattern id="FEW_SHOT">
            <name>Few-Shot Learning</name>
            <trigger_condition>Tasks requiring specific formatting, tone, or edge-case handling.</trigger_condition>
            <required_syntax>Minimum of 2 concrete <example> pairs (Input -> Output).</required_syntax>
            <success_criteria>Examples cover both a "Standard" case and an "Edge" case.</success_criteria>
        </pattern>

        <pattern id="REACT">
            <name>Reasoning and Acting</name>
            <trigger_condition>Agentic workflows involving tool use (e.g., searching code, reading files).</trigger_condition>
            <required_syntax>Thought: [Reasoning] -> Action: [Tool Call] -> Observation: [Result].</required_syntax>
        </pattern>
        
        <pattern id="SKELETON">
            <name>Skeleton-of-Thought</name>
            <trigger_condition>Long-form content generation (Reports, Documentation).</trigger_condition>
            <required_syntax>First generate an <outline>, then expand each section.</required_syntax>
        </pattern>
    </pattern_library>

    <alignment_logic>
        <step>1. **Complexity Assessment**: Determine if the prompt task is Atomic (Simple) or Composite (Complex).</step>
        <step>2. **Pattern Matching**: 
            - If Composite + Math/Logic -> Enforce [CoT].
            - If Structural/Schema-heavy -> Enforce [FEW_SHOT].
            - If Tool-dependent -> Enforce [REACT].
        </step>
        <step>3. **Gap Analysis**: Compare current prompt structure against the `required_syntax` of the chosen pattern.</step>
    </alignment_logic>

    <output_schema>
        <alignment_report>
            <recommended_pattern>ID of the pattern</recommended_pattern>
            <missing_elements>["list of missing syntax tags"]</missing_elements>
            <refactored_snippet>Code block showing how to inject the pattern.</refactored_snippet>
        </alignment_report>
    </output_schema>

    <echo>PATTERN_ALIGNMENT_SKILL_v1_READY</echo>
</skill_definition>