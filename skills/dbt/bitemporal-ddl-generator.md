<skill_definition name="bitemporal_ddl_generator">
  <metadata>
    <version>1.0.0</version>
    <capability>Bi-Temporal EAV DDL Generation for Financial Risk Schemas</capability>
    <target_engine>DuckDB (PoC) | Oracle | Hive (portability)</target_engine>
    <owning_agent>risk-schema-architect</owning_agent>
    <design_pattern>Long-Format EAV + SCD Type 2 + Bi-Temporal</design_pattern>
  </metadata>

  <description>
    This skill generates DDL for the 5-table Bi-Temporal EAV schema used for S&P
    financial risk data ingestion. It enforces the design philosophy of the
    risk-schema-architect: Technology Agnosticism (no dbt_* columns), Unified Storage
    (single ATTR_VALUE VARCHAR column), and Vendor & Obligor Authority anchoring.

    IMPORTANT: Column templates below use known column names from the Second Brain
    (dbt-logic-pro alignment). When actual S&P source column definitions are available,
    replace all [[ PLACEHOLDER: ... ]] annotations with confirmed column names/types.
  </description>

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
    - **Check:** Read the Second Brain (`.context/data_model_brain.json`) to confirm
      no schema has already been generated for this table. Avoid regenerating if stable.
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

  <ddl_templates>
    <template name="vendor_dim">
    ```sql
    -- [[ PLACEHOLDER: Confirm VENDOR_NAME length and whether SOURCE_SYSTEM is an enum or free-text ]]
    CREATE TABLE vendor_dim (
        VENDOR_ID        VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        VENDOR_NAME      VARCHAR(255)  NOT NULL,   -- Oracle: VARCHAR2(255) | Hive: STRING
        SOURCE_SYSTEM    VARCHAR(100),             -- Oracle: VARCHAR2(100) | Hive: STRING
        EFF_FROM         TIMESTAMP     NOT NULL,   -- Oracle: DATE          | Hive: TIMESTAMP
        EFF_TO           TIMESTAMP,                -- Oracle: DATE          | Hive: TIMESTAMP
        IS_CURRENT       BOOLEAN       NOT NULL DEFAULT TRUE, -- Oracle: NUMBER(1,0) | Hive: TINYINT
        INGESTION_ID     VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        PRIMARY KEY (VENDOR_ID)
    );
    ```
    </template>

    <template name="attr_metadata_dim">
    ```sql
    -- [[ PLACEHOLDER: Confirm ATTR_CATEGORY values (e.g., RATIO, FINANCIAL, DESCRIPTOR) ]]
    CREATE TABLE attr_metadata_dim (
        ATTR_ID          VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        ATTR_NAME        VARCHAR(255)  NOT NULL,   -- Oracle: VARCHAR2(255) | Hive: STRING
        ATTR_CATEGORY    VARCHAR(100),             -- Oracle: VARCHAR2(100) | Hive: STRING
        DATA_TYPE        VARCHAR(50),              -- Oracle: VARCHAR2(50)  | Hive: STRING
                                                  -- Expected values: 'DECIMAL', 'INTEGER', 'VARCHAR', 'DATE'
        IS_RATIO_FLAG    BOOLEAN       DEFAULT FALSE, -- Oracle: NUMBER(1,0) | Hive: TINYINT
        PRIMARY KEY (ATTR_ID)
    );
    ```
    </template>

    <template name="period_dim">
    ```sql
    -- [[ PLACEHOLDER: Confirm PERIOD_TYPE values and whether FISCAL_QUARTER is 1-4 or 'Q1'-'Q4' ]]
    CREATE TABLE period_dim (
        PERIOD_ID        VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        REPORT_DATE      DATE          NOT NULL,   -- Oracle: DATE          | Hive: DATE
        PERIOD_TYPE      VARCHAR(20)   NOT NULL,   -- Oracle: VARCHAR2(20)  | Hive: STRING
                                                  -- Expected values: 'ANNUAL', 'QUARTERLY'
        FISCAL_YEAR      INTEGER,                  -- Oracle: NUMBER(4,0)   | Hive: INT
        FISCAL_QUARTER   INTEGER,                  -- Oracle: NUMBER(1,0)   | Hive: TINYINT
                                                  -- NULL for ANNUAL periods
        PRIMARY KEY (PERIOD_ID)
    );
    ```
    </template>

    <template name="company_dim">
    ```sql
    -- [[ PLACEHOLDER: Confirm whether COMP_HK is pre-computed on ingest or generated by this layer ]]
    -- [[ PLACEHOLDER: Confirm the Internal App DB join key for OBLIGOR_ID mapping ]]
    CREATE TABLE company_dim (
        COMP_ID          VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        COMP_HK          VARCHAR(64)   NOT NULL,   -- Hash key: md5(VENDOR_ID || '|' || COMP_ID)
                                                  -- Oracle: VARCHAR2(64)  | Hive: STRING
        VENDOR_ID        VARCHAR(64)   NOT NULL,   -- FK -> vendor_dim      | Oracle: VARCHAR2(64)
        OBLIGOR_ID       VARCHAR(64),              -- FK -> Internal App DB (nullable; PENDING_MAP in model layer)
                                                  -- Oracle: VARCHAR2(64)  | Hive: STRING
        COMP_NAME        VARCHAR(255),             -- Oracle: VARCHAR2(255) | Hive: STRING
        EFF_FROM         TIMESTAMP     NOT NULL,   -- Oracle: DATE          | Hive: TIMESTAMP
        EFF_TO           TIMESTAMP,                -- Oracle: DATE          | Hive: TIMESTAMP
        IS_CURRENT       BOOLEAN       NOT NULL DEFAULT TRUE, -- Oracle: NUMBER(1,0) | Hive: TINYINT
        INGESTION_ID     VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING
        PRIMARY KEY (COMP_ID, COMP_HK)
    );
    ```
    </template>

    <template name="financial_observations_fact">
    ```sql
    -- [[ PLACEHOLDER: Confirm PRIORITY_RANK domain values: 1=Analyst Override, 2=S&P Inc, 3=S&P Full ]]
    -- [[ PLACEHOLDER: Confirm ATTR_VALUE max length; 4000 assumed for S&P long-format data ]]
    CREATE TABLE financial_observations_fact (
        -- Surrogate / Hash Key
        ATTR_HASH        VARCHAR(64)   NOT NULL,   -- SCD2 change-detection hash (COMP_HK || ATTR_ID || ATTR_VALUE)
                                                  -- Oracle: VARCHAR2(64)  | Hive: STRING

        -- Foreign Keys (EAV grain)
        COMP_HK          VARCHAR(64)   NOT NULL,   -- FK -> company_dim.COMP_HK
        VENDOR_ID        VARCHAR(64)   NOT NULL,   -- FK -> vendor_dim.VENDOR_ID
        COMP_ID          VARCHAR(64)   NOT NULL,   -- FK -> company_dim.COMP_ID
        PERIOD_ID        VARCHAR(64)   NOT NULL,   -- FK -> period_dim.PERIOD_ID
        ATTR_ID          VARCHAR(64)   NOT NULL,   -- FK -> attr_metadata_dim.ATTR_ID

        -- Obligor Bridge (nullable; PENDING_MAP COALESCE handled in dbt model layer)
        OBLIGOR_ID       VARCHAR(64),              -- Oracle: VARCHAR2(64)  | Hive: STRING

        -- Unified EAV Value Column (Single String Storage)
        ATTR_VALUE       VARCHAR(4000),            -- Oracle: VARCHAR2(4000) | Hive: STRING
                                                  -- DESIGN RULE: Cast in view layer via attr_metadata_dim.DATA_TYPE

        -- Priority & Source
        PRIORITY_RANK    INTEGER       NOT NULL,   -- 1=Analyst, 2=S&P Incremental, 3=S&P Full
                                                  -- Oracle: NUMBER(1,0) | Hive: TINYINT
        SOURCE_TIMESTAMP TIMESTAMP,               -- Oracle: DATE          | Hive: TIMESTAMP

        -- Bi-Temporal SCD Type 2 Columns
        EFF_FROM         TIMESTAMP     NOT NULL,   -- Oracle: DATE          | Hive: TIMESTAMP
        EFF_TO           TIMESTAMP,                -- NULL = open/current record
                                                  -- Oracle: DATE          | Hive: TIMESTAMP
        IS_CURRENT       BOOLEAN       NOT NULL DEFAULT TRUE, -- PHYSICAL FLAG (authoritative)
                                                  -- Oracle: NUMBER(1,0) | Hive: TINYINT

        -- Audit Columns
        REPORT_DATE      DATE          NOT NULL,   -- Oracle: DATE          | Hive: DATE
        INGESTION_ID     VARCHAR(64)   NOT NULL,   -- Oracle: VARCHAR2(64)  | Hive: STRING

        PRIMARY KEY (ATTR_HASH, EFF_FROM)
    );
    ```
    </template>
  </ddl_templates>

  <second_brain_update_contract>
    After generating DDL, you MUST update `.context/data_model_brain.json` with:
    ```json
    {
      "schema_version": "<ISO timestamp of generation>",
      "tables": {
        "<table_name>": {
          "status": "GENERATED | PLACEHOLDER_PENDING",
          "primary_key": "<column(s)>",
          "scd2_columns": ["EFF_FROM", "EFF_TO", "IS_CURRENT"],
          "hash_keys": ["<COMP_HK formula>", "<ATTR_HASH formula>"],
          "pending_placeholders": ["<list of [[ PLACEHOLDER: ... ]] items still unresolved>"]
        }
      }
    }
    ```
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

  <echo>BITEMPORAL_DDL_GENERATOR_v1_READY</echo>
</skill_definition>
