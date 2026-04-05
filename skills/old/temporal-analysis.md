<skill_definition id="SKL-SCD-04">
    <name>Temporal Architect</name>
    <capability>
        Configures dbt Snapshots with optimal strategies (check vs. timestamp) and constructs 
        Point-in-Time (PIT) join logic to align facts with historical dimension versions.
    </capability>
    
    <chain_of_thought_pattern>
        <step index="1">
            <goal>Strategy Selection</goal>
            <logic>Evaluate the source: If a reliable 'updated_at' exists, use 'timestamp'. If not, use 'check' on volatile columns[cite: 40].</logic>
        </step>
        <step index="2">
            <goal>Snapshot Configuration</goal>
            <logic>Define the snapshot block with 'dbt_utils.generate_surrogate_key' for the target unique key to ensure cross-system consistency[cite: 9, 51].</logic>
        </step>
        <step index="3">
            <goal>PIT Table Construction</goal>
            <logic>Create a 'spine' of valid timestamps. Use 'dbt_valid_from' and 'dbt_valid_to' to map fact event times to the correct dimension version[cite: 12].</logic>
        </step>
        <step index="4">
            <goal>Idempotency Audit</goal>
            <logic>Verify that re-running the PIT model on static data produces zero record drift[cite: 7].</logic>
        </step>
    </chain_of_thought_pattern>

    <audit_echo_trigger>
        "Temporal Strategy defined. Using [Timestamp/Check] strategy. 
        Surrogate key generated via dbt_utils. 
        PIT join logic mapped to align [Fact_Table] with [Dimension_Snapshot]."
    </audit_echo_trigger>
</skill_definition>