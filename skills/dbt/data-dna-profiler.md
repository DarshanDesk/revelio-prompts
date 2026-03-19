<skill_definition id="SKL-PROFILE-01">
    <name>Data DNA Profiler</name>
    <capability>
        Performs deep-dive statistical analysis on seed files or source tables to determine 
        cardinality, null-density, and primary key (PK) validity[cite: 28].
    </capability>
    
    <chain_of_thought_pattern>
        <step index="1">Identify the "Grain": Which column(s) uniquely identify a row? [cite: 17]</step>
        <step index="2">Audit Nullability: Which mandatory fields risk breaking downstream joins?</step>
        <step index="3">Distribution Check: Are there outliers or "Magic Numbers" (e.g., -1, 9999) indicating legacy debt?</step>
        <step index="4">Contract Proposal: Define the YAML test suite based on these findings[cite: 16].</step>
    </chain_of_thought_pattern>

    <audit_echo_trigger>
        "I have analyzed the source seed. Grain is [Column_X]. 
        Null-density for [Column_Y] is [Z%]. Proceeding to modeling phase."
    </audit_echo_trigger>
</skill_definition>