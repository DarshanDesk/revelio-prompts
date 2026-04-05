<skill_definition name="gold_view_designer">
  <metadata>
    <version>1.0.0</version>
    <capability>Gold-Layer View Design for Current Bi-Temporal Financial Observations</capability>
    <target_engine>DuckDB (PoC) | Oracle | Hive (portability)</target_engine>
    <owning_agent>risk-schema-architect</owning_agent>
    <output_view>vw_financial_latest_gold</output_view>
  </metadata>

  <description>
    This skill generates `vw_financial_latest_gold` — the authoritative reporting surface
    over the financial_observations_fact table. It filters to IS_CURRENT records,
    excludes PENDING_MAP orphans from the gold layer, and performs dynamic attribute
    value casting based on attr_metadata_dim.DATA_TYPE.

    The view joins all 5 tables to produce a wide, human-readable record with typed
    financial values, company context, and period dimension enrichment.
  </description>

  <design_rules>
    <rule id="CURRENT_RECORD_FILTER">
      Use the PHYSICAL IS_CURRENT column as the primary filter. DO NOT use EFF_TO IS NULL
      as the authoritative filter — this would create a CONTRADICTION with the DDL contract.
      Correct pattern: WHERE f.IS_CURRENT = TRUE
    </rule>
    <rule id="PENDING_MAP_EXCLUSION">
      The gold view MUST exclude PENDING_MAP records. These are analytically unreliable
      until mapped to a confirmed OBLIGOR_ID.
      Correct pattern: AND f.OBLIGOR_ID != 'PENDING_MAP'
      Note: A PENDING_MAP monitoring view is handled separately by the exception_mgmt skill.
    </rule>
    <rule id="DYNAMIC_CASTING">
      ATTR_VALUE is stored as VARCHAR. The view MUST cast it to the appropriate type
      using attr_metadata_dim.DATA_TYPE as the lookup key.
      Casting strategy:
        'DECIMAL'  -> CAST(f.ATTR_VALUE AS DECIMAL(18,2))
        'INTEGER'  -> CAST(f.ATTR_VALUE AS INTEGER)
        'DATE'     -> CAST(f.ATTR_VALUE AS DATE)
        'VARCHAR'  -> f.ATTR_VALUE (no cast needed)
      Use CASE WHEN for DuckDB compatibility. Add a safe-cast guard using TRY_CAST
      where available, or wrap in error handling for Oracle/Hive.
    </rule>
    <rule id="PRIORITY_RESOLUTION">
      When multiple sources (Analyst, S&P Inc, S&P Full) have IS_CURRENT = TRUE for
      the same grain (COMP_ID + PERIOD_ID + ATTR_ID), the view MUST apply
      PRIORITY_RANK logic. The row with the lowest PRIORITY_RANK wins.
      A ROW_NUMBER() window function applied in the CTE layer eliminates duplicates
      before the final SELECT.
    </rule>
  </design_rules>

  <instruction_set>
    ### 1. Identify the View Grain
    - The grain of `vw_financial_latest_gold` is: (COMP_ID, PERIOD_ID, ATTR_ID) — one row per attribute per company per period.
    - Resolve multi-source conflicts using PRIORITY_RANK (see PRIORITY_RESOLUTION rule above).

    ### 2. CTE Architecture
    Build the view as a named CTE chain for readability and portability:
    ```
    current_facts      -> Filter financial_observations_fact to IS_CURRENT = TRUE
                          and OBLIGOR_ID != 'PENDING_MAP'
    priority_resolved  -> Apply ROW_NUMBER() to select PRIORITY_RANK = 1 winner
                          per (COMP_ID, PERIOD_ID, ATTR_ID) grain
    typed_values       -> Join attr_metadata_dim, apply CASE-based dynamic cast
    final_gold         -> Join company_dim, period_dim, vendor_dim for enrichment
    ```

    ### 3. Column Selection for the Gold View
    The final SELECT should include (in this order):
    - Identity  : COMP_ID, COMP_NAME (from company_dim), OBLIGOR_ID
    - Period    : PERIOD_ID, REPORT_DATE, PERIOD_TYPE, FISCAL_YEAR, FISCAL_QUARTER
    - Attribute : ATTR_ID, ATTR_NAME (from attr_metadata_dim), ATTR_CATEGORY, IS_RATIO_FLAG
    - Value     : ATTR_VALUE_RAW (original VARCHAR), ATTR_VALUE_TYPED (cast result)
    - Source    : VENDOR_ID, VENDOR_NAME (from vendor_dim), PRIORITY_RANK, SOURCE_TIMESTAMP
    - Audit     : INGESTION_ID, REPORT_DATE

    ### 4. Portability Annotations
    - DuckDB uses `TRY_CAST`. Oracle uses `TO_NUMBER / TO_DATE`. Hive uses `CAST`.
    - Add `-- [ORACLE_PORT]` and `-- [HIVE_PORT]` comment blocks below the primary cast CASE.
  </instruction_set>

  <view_template>
    ```sql
    -- vw_financial_latest_gold
    -- Gold-layer view: current records, no PENDING_MAP, priority-resolved, dynamically cast.
    -- [[ PLACEHOLDER: Replace table names if schema prefix is required (e.g., risk.financial_observations_fact) ]]

    CREATE VIEW vw_financial_latest_gold AS

    WITH current_facts AS (
        SELECT *
        FROM financial_observations_fact
        WHERE IS_CURRENT = TRUE                  -- Physical IS_CURRENT column (authoritative)
          AND OBLIGOR_ID != 'PENDING_MAP'        -- Exclude unmapped entities from gold layer
    ),

    priority_resolved AS (
        SELECT *,
               ROW_NUMBER() OVER (
                   PARTITION BY COMP_ID, PERIOD_ID, ATTR_ID
                   ORDER BY PRIORITY_RANK ASC, SOURCE_TIMESTAMP DESC
               ) AS rn
        FROM current_facts
    ),

    typed_values AS (
        SELECT
            pr.*,
            amd.ATTR_NAME,
            amd.ATTR_CATEGORY,
            amd.DATA_TYPE,
            amd.IS_RATIO_FLAG,
            -- Dynamic cast based on attr_metadata_dim.DATA_TYPE
            CASE amd.DATA_TYPE
                WHEN 'DECIMAL'  THEN TRY_CAST(pr.ATTR_VALUE AS VARCHAR)  -- preserve raw; cast below
                WHEN 'INTEGER'  THEN TRY_CAST(pr.ATTR_VALUE AS VARCHAR)
                WHEN 'DATE'     THEN TRY_CAST(pr.ATTR_VALUE AS VARCHAR)
                ELSE pr.ATTR_VALUE
            END AS ATTR_VALUE_RAW,
            -- [[ PLACEHOLDER: Confirm TRY_CAST availability in your DuckDB version (>=0.8) ]]
            CASE amd.DATA_TYPE
                WHEN 'DECIMAL'  THEN TRY_CAST(pr.ATTR_VALUE AS DECIMAL(18,2))::VARCHAR
                WHEN 'INTEGER'  THEN TRY_CAST(pr.ATTR_VALUE AS INTEGER)::VARCHAR
                WHEN 'DATE'     THEN TRY_CAST(pr.ATTR_VALUE AS DATE)::VARCHAR
                ELSE pr.ATTR_VALUE
            END AS ATTR_VALUE_TYPED
            -- [ORACLE_PORT]: Replace TRY_CAST with TO_NUMBER(ATTR_VALUE) / TO_DATE(ATTR_VALUE, 'YYYY-MM-DD')
            --                Wrap in CASE WHEN REGEXP_LIKE(ATTR_VALUE, '^-?[0-9.]+$') for safe casting
            -- [HIVE_PORT]  : Use CAST(ATTR_VALUE AS DOUBLE) for DECIMAL; standard CAST for others
        FROM priority_resolved pr
        JOIN attr_metadata_dim amd ON pr.ATTR_ID = amd.ATTR_ID
        WHERE pr.rn = 1  -- Priority-resolved: one row per (COMP_ID, PERIOD_ID, ATTR_ID)
    ),

    final_gold AS (
        SELECT
            -- Company Context
            tv.COMP_ID,
            cd.COMP_NAME,
            tv.OBLIGOR_ID,
            -- Period Context
            tv.PERIOD_ID,
            pd.REPORT_DATE,
            pd.PERIOD_TYPE,
            pd.FISCAL_YEAR,
            pd.FISCAL_QUARTER,
            -- Attribute Context
            tv.ATTR_ID,
            tv.ATTR_NAME,
            tv.ATTR_CATEGORY,
            tv.IS_RATIO_FLAG,
            -- Value (Raw + Typed)
            tv.ATTR_VALUE_RAW,
            tv.ATTR_VALUE_TYPED,
            -- Source Lineage
            tv.VENDOR_ID,
            vd.VENDOR_NAME,
            tv.PRIORITY_RANK,
            tv.SOURCE_TIMESTAMP,
            -- Audit
            tv.INGESTION_ID,
            tv.EFF_FROM,
            tv.EFF_TO,
            tv.IS_CURRENT
        FROM typed_values tv
        JOIN company_dim  cd ON tv.COMP_ID   = cd.COMP_ID   AND cd.IS_CURRENT = TRUE
        JOIN period_dim   pd ON tv.PERIOD_ID = pd.PERIOD_ID
        JOIN vendor_dim   vd ON tv.VENDOR_ID = vd.VENDOR_ID AND vd.IS_CURRENT = TRUE
    )

    SELECT * FROM final_gold;
    ```
  </view_template>

  <second_brain_update_contract>
    After generating the view, update `.context/data_model_brain.json` with:
    ```json
    {
      "views": {
        "vw_financial_latest_gold": {
          "status": "GENERATED",
          "grain": ["COMP_ID", "PERIOD_ID", "ATTR_ID"],
          "current_record_filter": "IS_CURRENT = TRUE (physical column)",
          "pending_map_excluded": true,
          "priority_resolution": "ROW_NUMBER() on PRIORITY_RANK ASC, SOURCE_TIMESTAMP DESC",
          "casting_strategy": "CASE WHEN on attr_metadata_dim.DATA_TYPE",
          "pending_placeholders": ["<list of [[ PLACEHOLDER: ... ]] items unresolved>"]
        }
      }
    }
    ```
  </second_brain_update_contract>

  <success_criteria>
    - [ ] View filters on IS_CURRENT = TRUE (physical column), NOT EFF_TO IS NULL.
    - [ ] PENDING_MAP records excluded: AND OBLIGOR_ID != 'PENDING_MAP'.
    - [ ] ROW_NUMBER() CTE applied for priority resolution before final SELECT.
    - [ ] Dynamic cast uses CASE WHEN on attr_metadata_dim.DATA_TYPE.
    - [ ] Oracle and Hive portability annotations present as inline comments.
    - [ ] Second Brain updated with view grain and casting strategy.
  </success_criteria>

  <echo>GOLD_VIEW_DESIGNER_v1_READY</echo>
</skill_definition>
