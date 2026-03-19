<skill_definition id="SKL-BDD-03">
    <name>Gherkin-to-SQL Translator</name>
    <capability>
        Parses natural language "Given/When/Then" scenarios and generates the corresponding 
        dbt 'unit_tests:' block in the model's YAML configuration.
    </capability>
    
    <chain_of_thought_pattern>
        <step index="1">
            <goal>Scenario Decomposition</goal>
            <logic>Extract the 'Given' (input seeds), 'When' (transformation logic/filter), and 'Then' (expected output grain/values).</logic>
        </step>
        <step index="2">
            <goal>Model Mapping</goal>
            <logic>Identify which dbt model (e.g., int_orders) is the target and which upstream dependencies (e.g., stg_orders) need to be mocked.</logic>
        </step>
        <step index="3">
            <goal>Edge-Case Injection</goal>
            <logic>Based on SKL-PROFILE-01, inject 1 null-check and 1 boundary-check (e.g., zero value) into the mocked input to ensure robustness.</logic>
        </step>
        <step index="4">
            <goal>YAML Synthesis</goal>
            <logic>Construct the dbt 'unit_test' syntax, ensuring the 'expect:' block matches the predicted outcome of the transformation logic.</logic>
        </step>
    </chain_of_thought_pattern>

    <audit_echo_trigger>
        "Gherkin parsed: [Scenario Name]. 
        Mocks defined for [Source_A, Source_B]. 
        Expectation: [X] rows with attributes [Y]. 
        Unit test YAML is ready for inclusion in schema.yml."
    </audit_echo_trigger>
</skill_definition>