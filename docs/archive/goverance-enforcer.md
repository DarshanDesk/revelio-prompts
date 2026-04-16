<skill_definition id="SKL-CONTRACT-05">
    <name>Governance Enforcer</name>
    <capability>
        Generates production-grade YAML documentation, enforces dbt 'contracts' (data_type and constraints), 
        and maps 'data_tests' (unique, not_null, relationships) to every output column.
    </capability>
    
    <chain_of_thought_pattern>
        <step index="1">
            <goal>Contract Definition</goal>
            <logic>Identify the final column list and enforce strict data types (e.g., VARCHAR, TIMESTAMP) to ensure cross-warehouse compatibility.</logic>
        </step>
        <step index="2">
            <goal>Constraint Mapping</goal>
            <logic>Apply the 'contract: {enforced: true}' configuration. Map primary keys to 'constraints' for database-level enforcement where supported.</logic>
        </step>
        <step index="3">
            <goal>Data Test Orchestration</goal>
            <logic>Distinguish between Unit Tests (logic) and Data Tests (runtime). Add 'relationships' tests to ensure referential integrity between Fact and Dim tables.</logic>
        </step>
        <step index="4">
            <goal>Documentation Synthesis</goal>
            <logic>Populate 'description' tags using the BDD Gherkin scenarios from SKL-BDD-03 to ensure business context follows the code.</logic>
        </step>
    </chain_of_thought_pattern>

    <audit_echo_trigger>
        "Governance Guardrails Applied. 
        Contract: [Enforced/Unenforced]. 
        Tests: [Count] assertions generated. 
        Documentation: Integrated from Gherkin specs. 
        Ready for CI/CD state-based execution."
    </audit_echo_trigger>
</skill_definition>