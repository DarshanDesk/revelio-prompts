---
name: bitemporal-ddl-generator
description: "Serves as the DDL authority for the 5-table EAV schema. Activates when generating EFF_FROM/EFF_TO/IS_CURRENT BOOLEAN DDL or DuckDB column definitions. Does not cover SCD2 merge logic or view patterns."
metadata:
  version: "1.0.0"
compatibility: "DuckDB (PoC) | Oracle | Hive (portability)"
---

<skill_definition name="bitemporal_ddl_generator">
  <metadata>
    <design_pattern>Long-Format EAV + SCD Type 2 + Bi-Temporal</design_pattern>
  </metadata>


  <column_conventions>
    <convention name="hash_keys">
      MD5/SHA256 surrogate keys stored as VARCHAR(64).
      COMP_HK = md5(concat(VENDOR_ID, '|', COMP_ID)) — derive using native SQL, NOT dbt_utils.
    </convention>
    <convention name="scd2_columns">
      EFF_FROM    : Start of validity (inclusive). DuckDB: TIMESTAMP. Oracle: DATE. Hive: TIMESTAMP.
      EFF_TO      : End of validity (exclusive, NULL = open). DuckDB: TIMESTAMP. Oracle: DATE. Hive: TIMESTAMP.
      IS_CURRENT  : Physical flag for current record. DuckDB: BOOLEAN. Oracle: NUMBER(1,0). Hive: TINYINT.
                    Authoritative definition: IS_CURRENT = TRUE (physical column, NOT derived from EFF_TO IS NULL).
    </convention>
    <convention name="audit_columns">
      INGESTION_ID    : UUID or sequence ID for the load batch. VARCHAR(64).
      REPORT_DATE     : Business date of the source file. DATE.
      SOURCE_TIMESTAMP: Wall-clock time of source record. TIMESTAMP.
    </convention>
    <convention name="naming_rules">
      - ZERO dbt_* prefix columns (no dbt_scd_id, dbt_updated_at, dbt_valid_from, etc.)
      - Use UPPER_SNAKE_CASE for all column names (Oracle/Hive compatibility)
      - VARCHAR length annotations: VARCHAR(N). Oracle: VARCHAR2(N). Hive: STRING (annotate inline).
    </convention>
  </column_conventions>

  <instruction_set>
    ### 1. Validate Design Philosophy Before Generating DDL
    - **Check:** Load `.github/context-cache/SCHEMA.md § SCD2 Column Name Authority` to confirm
      no schema conflict exists for this table. Check `.github/context-cache/BRAIN.md` for pipeline
      status. Avoid regenerating DDL that is already marked GENERATED in the pipeline log.
    - **Enforcement:** If `dbt_*` column names appear in the user's request, flag as
      [VIOLATION] and substitute with the technology-agnostic equivalent.

    ### 2. Table Generation Order (Dependency-Safe)
    Always generate DDL in this order to respect foreign key dependencies:
    ```
    1. vendor_dim
    2. attr_metadata_dim
    3. period_dim
    4. company_dim          (references vendor_dim for OBLIGOR bridge)
    5. financial_observations_fact  (references all 4 dims)
    ```

    ### 3. DDL Template Rules
    - **Primary Keys:** Use `VARCHAR(64)` for hash-based PKs. Use `BIGINT` for sequences.
    - **Portability Annotations:** For every DuckDB-specific type, add an inline comment:
      `-- Oracle: <equivalent> | Hive: <equivalent>`
    - **IS_CURRENT:** Always a physical column. Default to FALSE on insert; set TRUE only
      for the authoritative current record.
    - **ATTR_VALUE:** Must be a single `VARCHAR(4000)` column. Do NOT split into typed columns.
      Rationale: S&P long-format data is sparse; unified storage prevents schema explosion.
    - **OBLIGOR_ID Bridge:** In company_dim, OBLIGOR_ID is nullable. The PENDING_MAP fallback
      (`COALESCE(OBLIGOR_ID, 'PENDING_MAP')`) belongs in the dbt model layer, NOT in DDL.

    ### 4. Dialect-Specific Placeholder Strategy
    When Oracle/Hive equivalents differ significantly, include a commented-out alternate DDL
    block directly below each DuckDB table using the prefix `-- [ORACLE_PORT]` or `-- [HIVE_PORT]`.
  </instruction_set>

  <ddl_templates ref=".github/skills/bitemporal-ddl-generator/references/ddl-templates.md">
    Full CREATE TABLE statements for all 5 tables (vendor_dim, attr_metadata_dim, period_dim,
    company_dim, financial_observations_fact) with DuckDB/Oracle/Hive portability annotations
    and [[ PLACEHOLDER ]] markers are in the references file above. Load that file when
    generating DDL output.
  </ddl_templates>

  <second_brain_update_contract>
    After generating DDL, emit the following `[BRAIN-UPDATE-PENDING]` markers for manual
    application to `.github/context-cache/BRAIN.md` and `.github/context-cache/SCHEMA.md`:
    ```
    [BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: Step 1 complete — RISK_SCHEMA_ARCHITECT_v1_READY]
    [BRAIN-UPDATE-PENDING: SCHEMA.md: SCD2_Columns: <table_name> EFF_FROM/EFF_TO/IS_CURRENT BOOLEAN — GENERATED <ISO timestamp>]
    [BRAIN-UPDATE-PENDING: SCHEMA.md: HASH_KEYS: <table_name> COMP_HK=md5(concat(...)), ATTR_HASH=md5(concat(...))]
    [BRAIN-UPDATE-PENDING: SCHEMA.md: PLACEHOLDER_STATUS: <list of [[ PLACEHOLDER: ... ]] items still unresolved>]
    ```
    Note: GitHub Copilot cannot write to files during a session.
    The user must apply these markers manually after the session.
  </second_brain_update_contract>

  <success_criteria>
    - [ ] DDL generated in dependency-safe order (vendor_dim first, financial_observations_fact last).
    - [ ] Zero dbt_* column names present in any generated DDL.
    - [ ] IS_CURRENT is a physical BOOLEAN column with DEFAULT TRUE (not derived from EFF_TO IS NULL).
    - [ ] ATTR_VALUE is a single VARCHAR column (no type-split columns).
    - [ ] All DuckDB-specific types carry inline Oracle/Hive comments.
    - [ ] OBLIGOR_ID is nullable in DDL; PENDING_MAP logic deferred to model layer.
    - [ ] Second Brain updated with schema_version and pending_placeholders list.
  </success_criteria>

</skill_definition>
