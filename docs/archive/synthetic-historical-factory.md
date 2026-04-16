<skill_definition id="SKL-MOCK-02">
    <name>Synthetic History Factory</name>
    <capability>
        Generates multi-state CSV seeds (Day 1, Day 2, Day 3) to validate that dbt snapshots 
        accurately capture SCD Type 2 changes, point-in-time (PIT) accuracy, and record 
        invalidation (dbt_valid_to)[cite: 32, 40].
    </capability>
    
    <chain_of_thought_pattern>
        <step index="1">
            <goal>Identify Change Vectors</goal>
            <logic>Determine which attributes are 'Type 1' (overwrite) vs 'Type 2' (historical) based on the business grain.</logic>
        </step>
        <step index="2">
            <goal>Construct Day 1 (Baseline)</goal>
            <logic>Create an initial state with diverse records (New, Null-heavy, and Edge-case strings).</logic>
        </step>
        <step index="3">
            <goal>Construct Day 2 (The Mutation)</goal>
            <logic>Modify specific fields to trigger 'check' or 'timestamp' strategies. Include 1 update, 1 new record, and 1 unchanged record to test idempotency[cite: 7, 32].</logic>
        </step>
        <step index="4">
            <goal>Verification Logic</goal>
            <logic>Predict the expected row count in the final snapshot table (Expected = Day 1 records + Day 2 updates + Day 2 new)[cite: 15, 52].</logic>
        </step>
    </chain_of_thought_pattern>

    <audit_echo_trigger>
        "Synthetic History generated: 3 states defined. 
        Day 1: Baseline. Day 2: Value Change. Day 3: Logical Delete. 
        Ready to execute 'dbt snapshot' tests"[cite: 32, 53].
    </audit_echo_trigger>
</skill_definition>