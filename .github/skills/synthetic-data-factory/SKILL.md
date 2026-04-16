---
name: synthetic-data-factory
description: "Generates 6 CSV seed files with Day 0 and Day N structure and PENDING_MAP injection. Activates when seeding vendor_dim, company_dim, financial_observations_fact, attr_metadata_dim, or period_dim tables. Does not cover dbt model logic."
metadata:
  version: "1.0.0"
compatibility: "DuckDB seeds (CSV) | dbt /seeds folder"
---

<skill_definition name="synthetic_data_factory">
  <metadata>
    <second_brain_ref>.github/context-cache/RULES.md § Data Synthesis Parameters | SCHEMA.md § EAV Column Contract</second_brain_ref>
  </metadata>


  <canonical_facts>
    <fact id="SEED-001">ATTR_VALUE column type in CSV: plain string VALUES. Do not pre-cast to numeric. DuckDB reads them as VARCHAR.</fact>
    <fact id="SEED-002">OBLIGOR_ID is nullable in the mapping CSV — leave blank to trigger PENDING_MAP COALESCE in the model layer.</fact>
    <fact id="SEED-003">All date fields: ISO 8601 format (YYYY-MM-DD) for DuckDB DATE compatibility.</fact>
    <fact id="SEED-004">Referential integrity: COMP_ID values must appear in vendor_seed, period_seed, data_seed, ratio_seed, AND obligor_mapping within the same Day 0/Day N scenario.</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. Seed File Inventory
    Generate all 6 seed files per scenario (Day 0 and Day N):

    | File | dbt seed path | Purpose |
    |------|--------------|---------|
    | `vendor_seed.csv` | `seeds/vendor_seed.csv` | S&P Vendor master — anchors all records |
    | `period_seed.csv` | `seeds/period_seed.csv` | Fiscal period definitions |
    | `data_seed.csv` | `seeds/data_seed.csv` | Financial data items (Long format EAV) |
    | `ratio_seed.csv` | `seeds/ratio_seed.csv` | Calculated ratio attributes (Long format EAV) |
    | `analyst_updates.csv` | `seeds/analyst_updates.csv` | Internal Analyst Overrides (PRIORITY_RANK = 1) |
    | `obligor_mapping.csv` | `seeds/obligor_mapping.csv` | Internal App DB OBLIGOR_ID mapping |

    ### 2. Long-Format (EAV) CSV Structure
    All data/ratio CSVs must use the Long format — one row per attribute value:
    ```csv
    COMP_ID,VENDOR_ID,PERIOD_ID,ATTR_ID,ATTR_VALUE,REPORT_DATE,SOURCE_TIMESTAMP,INGESTION_ID,SOURCE_TRACE_ID,PRIORITY_RANK
    SP001,SP_GLOBAL,FY2024Q4,TOTAL_ASSETS,"1250000.00",2025-01-15,2025-01-15 08:00:00,ING_001,TRC_001,3
    SP001,SP_GLOBAL,FY2024Q4,TOTAL_LIABILITIES,"850000.00",2025-01-15,2025-01-15 08:00:00,ING_001,TRC_001,3
    SP002,SP_GLOBAL,FY2024Q4,TOTAL_ASSETS,"780000.00",2025-01-15,2025-01-15 08:00:00,ING_001,TRC_002,3
    ```
    - ATTR_VALUE is always a quoted string
    - No numeric types in CSV headers — DuckDB infers via CAST in models

    ### 3. PENDING_MAP Injection Pattern
    To inject ≥ 5% PENDING_MAP records, include COMP_IDs in data_seed.csv that have
    **no corresponding row** in obligor_mapping.csv:
    ```csv
    -- In data_seed.csv: include COMP_ID = 'SP_UNMAPPED_001'
    SP_UNMAPPED_001,SP_GLOBAL,FY2024Q4,TOTAL_ASSETS,"500000.00",2025-01-15,...

    -- In obligor_mapping.csv: deliberately OMIT 'SP_UNMAPPED_001'
    -- The model's COALESCE(map.OBLIGOR_ID, 'PENDING_MAP') will produce 'PENDING_MAP'
    ```
    Count: with 500 records minimum, include ≥ 25 PENDING_MAP-triggering rows (5%).

    ### 4. Analyst Override Mocking (PRIORITY_RANK = 1)
    Generate ≥ 2 Analyst Override records in `analyst_updates.csv` with:
    - Same COMP_ID, PERIOD_ID, ATTR_ID grain as an existing S&P Full Feed record
    - Higher (more recent) SOURCE_TIMESTAMP to break ties within rank
    - Different ATTR_VALUE to trigger conflict resolution
    ```csv
    COMP_ID,VENDOR_ID,PERIOD_ID,ATTR_ID,ATTR_VALUE,REPORT_DATE,SOURCE_TIMESTAMP,INGESTION_ID,SOURCE_TRACE_ID,PRIORITY_RANK,UPDATED_BY
    SP001,INTERNAL,FY2024Q4,TOTAL_ASSETS,"1260000.00",2025-01-20,2025-01-20 14:00:00,ANA_001,TRC_ANA_001,1,analyst@firm.com
    SP003,INTERNAL,FY2024Q4,NET_INCOME,"95000.00",2025-01-20,2025-01-20 14:05:00,ANA_001,TRC_ANA_002,1,analyst@firm.com
    ```

    ### 5. Day 0 / Day N Versioning (SCD2 Time Travel Trigger)
    **Day 0 (V1) — Initial Full Load**:
    - All records are new; `is_incremental()` = FALSE path runs
    - All inserted records: EFF_FROM = load_timestamp, EFF_TO = NULL, IS_CURRENT = TRUE
    - Obligor_mapping includes all COMPs except the deliberate PENDING_MAP ones

    **Day N (V2) — Delta / Incremental Load**:
    Modify ≥ 3 records from V1 to trigger SCD2 close-and-reopen:
    ```csv
    -- In data_seed_v2.csv: update ATTR_VALUE for COMP_ID SP001, ATTR_ID TOTAL_ASSETS
    SP001,SP_GLOBAL,FY2024Q4,TOTAL_ASSETS,"1275000.00",2025-03-01,...   ← changed value triggers ATTR_HASH diff
    ```
    Expected SCD2 outcome for each changed record:
    - V1 record: IS_CURRENT = FALSE, EFF_TO = Day_N_timestamp
    - V2 record: IS_CURRENT = TRUE,  EFF_FROM = Day_N_timestamp, EFF_TO = NULL

    ### 6. Referential Integrity Checklist
    Before finalising seeds, verify:
    - [ ] Every COMP_ID in `data_seed` exists in `vendor_seed`
    - [ ] Every PERIOD_ID in `data_seed` exists in `period_seed`
    - [ ] Every ATTR_ID in `data_seed` exists in `attr_metadata_dim` (or seed it separately)
    - [ ] ≥ 5% of COMP_IDs in `data_seed` are absent from `obligor_mapping` (PENDING_MAP trigger)
    - [ ] ≥ 2 records exist in `analyst_updates` with matching grain to `data_seed`
    - [ ] V1 and V2 share the same COMP_ID, PERIOD_ID, ATTR_ID set for the changed records

    ### 7. Synthetic Data Trace (Brain Update)
    After generating seeds, emit a `[BRAIN-UPDATE-PENDING]` marker documenting the Comp_IDs
    used, to enable other agents to validate against the same reference:
    ```
    [BRAIN-UPDATE-PENDING: BRAIN.md: SEED_STATS: Day0=500 records; PENDING_MAP_COMPs: SP_UNMAPPED_001..025; Analyst_COMPs: SP001,SP003; Day_N_changed: SP001-TOTAL_ASSETS, SP003-NET_INCOME, SP005-DEBT_RATIO]
    ```

  </instruction_set>

</skill_definition>
