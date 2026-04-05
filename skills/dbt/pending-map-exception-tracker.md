<skill_definition name="pending_map_exception_tracker">
  <metadata>
    <version>1.0.0</version>
    <capability>PENDING_MAP Orphan Tracking & Unmapped Entity Exception Management</capability>
    <target_engine>DuckDB (PoC) | Oracle | Hive (portability)</target_engine>
    <owning_agent>risk-schema-architect</owning_agent>
    <output_view>vw_unmapped_entities</output_view>
  </metadata>

  <description>
    This skill generates `vw_unmapped_entities` — the exception surface for tracking
    financial_observations_fact records where OBLIGOR_ID could not be resolved to an
    Internal App DB entity and fell back to the sentinel value 'PENDING_MAP'.

    It provides: raw orphan counts, triage enrichment (VENDOR, INGESTION_ID, REPORT_DATE),
    a percentage-of-total health metric, and a threshold-based alert signal. The view
    is the monitoring contract between the risk-schema-architect and the
    financial-audit-pro agent.
  </description>

  <design_rules>
    <rule id="PENDING_MAP_SENTINEL">
      'PENDING_MAP' is the COALESCE fallback applied in the dbt model layer (not DDL).
      This skill reads OBLIGOR_ID = 'PENDING_MAP' as the canonical unmapped signal.
      DO NOT create a separate flag column for unmapped status.
    </rule>
    <rule id="ZERO_DATA_LOSS">
      The PENDING_MAP pattern exists to enforce Zero Data Loss — records must load even
      when the internal OBLIGOR mapping is unavailable. This view surfaces them for
      remediation, NOT for deletion.
      NEVER generate a DELETE or TRUNCATE statement for PENDING_MAP records.
    </rule>
    <rule id="ALERT_THRESHOLD">
      If PENDING_MAP records exceed 5% of total IS_CURRENT records for a given
      VENDOR_ID + REPORT_DATE combination, emit a [WARNING] status in the view.
      This threshold is configurable via the Second Brain parameter:
        `exception_mgmt.pending_map_alert_threshold_pct` (default: 5.0)
    </rule>
    <rule id="TRIAGE_GRAIN">
      The triage grain is: (COMP_HK, VENDOR_ID, INGESTION_ID, REPORT_DATE).
      This gives the operations team the exact batch and company identifier needed
      to locate and resolve the mapping gap in the Internal App DB.
    </rule>
  </design_rules>

  <instruction_set>
    ### 1. Two-Layer View Architecture
    Build `vw_unmapped_entities` in two layers:
    ```
    Layer 1 — raw_orphans     : All IS_CURRENT records where OBLIGOR_ID = 'PENDING_MAP'
    Layer 2 — health_metrics  : Aggregate orphan count vs. total IS_CURRENT count
                                per (VENDOR_ID, REPORT_DATE); compute alert threshold
    ```
    Present BOTH layers in a UNION-style or separate CTEs with a final SELECT joining them.

    ### 2. Triage Columns (Mandatory in raw_orphans)
    Include these columns to enable operational remediation:
    - COMP_HK          (hash — unique identifier to locate the company in vendor feed)
    - COMP_ID          (S&P internal company identifier)
    - COMP_NAME        (from company_dim, for human readability)
    - VENDOR_ID        (which vendor sourced this record)
    - INGESTION_ID     (which load batch introduced the orphan)
    - REPORT_DATE      (business date — helps narrow to specific S&P file)
    - ATTR_ID          (attribute that is orphaned — may indicate a specific feed section)
    - ATTR_NAME        (from attr_metadata_dim)
    - ATTR_VALUE       (the raw value that failed to map)
    - SOURCE_TIMESTAMP (when the record was received)

    ### 3. Health Metric Calculation
    For each (VENDOR_ID, REPORT_DATE) group:
    ```sql
    orphan_count     = COUNT(*) WHERE OBLIGOR_ID = 'PENDING_MAP' AND IS_CURRENT = TRUE
    total_count      = COUNT(*) WHERE IS_CURRENT = TRUE
    orphan_pct       = ROUND(orphan_count * 100.0 / NULLIF(total_count, 0), 2)
    alert_status     = CASE WHEN orphan_pct > [threshold] THEN '[WARNING]' ELSE '[OK]' END
    ```
    - [[ PLACEHOLDER: Read `exception_mgmt.pending_map_alert_threshold_pct` from Second Brain;
         default to 5.0 if not set ]]

    ### 4. Portability Annotations
    - DuckDB: `ROUND(x, 2)` and `NULLIF`. Oracle: `ROUND(x, 2)` and `NULLIF` (compatible).
    - Hive: `ROUND(x, 2)` compatible; use `COALESCE(total_count, 1)` instead of NULLIF for safety.
    - Add `-- [ORACLE_PORT]` and `-- [HIVE_PORT]` comments on divergent lines.
  </instruction_set>

  <view_template>
    ```sql
    -- vw_unmapped_entities
    -- Exception tracker: PENDING_MAP orphan triage and health monitoring.
    -- [[ PLACEHOLDER: Replace table names with schema-prefixed names if required ]]
    -- [[ PLACEHOLDER: Set alert threshold from Second Brain or hardcode default 5.0 ]]

    CREATE VIEW vw_unmapped_entities AS

    WITH raw_orphans AS (
        -- All current records that failed OBLIGOR_ID mapping
        SELECT
            f.COMP_HK,
            f.COMP_ID,
            cd.COMP_NAME,
            f.VENDOR_ID,
            vd.VENDOR_NAME,
            f.INGESTION_ID,
            f.REPORT_DATE,
            f.ATTR_ID,
            amd.ATTR_NAME,
            f.ATTR_VALUE,
            f.SOURCE_TIMESTAMP,
            f.EFF_FROM,
            f.PRIORITY_RANK
        FROM financial_observations_fact f
        LEFT JOIN company_dim       cd  ON f.COMP_ID   = cd.COMP_ID   AND cd.IS_CURRENT = TRUE
        LEFT JOIN vendor_dim        vd  ON f.VENDOR_ID = vd.VENDOR_ID AND vd.IS_CURRENT = TRUE
        LEFT JOIN attr_metadata_dim amd ON f.ATTR_ID   = amd.ATTR_ID
        WHERE f.OBLIGOR_ID = 'PENDING_MAP'
          AND f.IS_CURRENT = TRUE
    ),

    health_metrics AS (
        -- Aggregate orphan rate per vendor + report date
        SELECT
            f.VENDOR_ID,
            f.REPORT_DATE,
            COUNT(CASE WHEN f.OBLIGOR_ID = 'PENDING_MAP' THEN 1 END)   AS orphan_count,
            COUNT(*)                                                     AS total_current_count,
            ROUND(
                COUNT(CASE WHEN f.OBLIGOR_ID = 'PENDING_MAP' THEN 1 END) * 100.0
                / NULLIF(COUNT(*), 0),                                  -- Oracle/Hive: NULLIF compatible
                2
            )                                                            AS orphan_pct,
            -- Alert threshold: flag if orphan_pct > 5.0 (configurable via Second Brain)
            CASE
                WHEN ROUND(
                    COUNT(CASE WHEN f.OBLIGOR_ID = 'PENDING_MAP' THEN 1 END) * 100.0
                    / NULLIF(COUNT(*), 0), 2
                ) > 5.0
                THEN '[WARNING]: PENDING_MAP exceeds 5% threshold'
                ELSE '[OK]'
            END AS alert_status
            -- [ORACLE_PORT]: NULLIF is Oracle-compatible. No changes needed.
            -- [HIVE_PORT]  : Replace NULLIF(COUNT(*), 0) with GREATEST(COUNT(*), 1) for Hive safety.
        FROM financial_observations_fact f
        WHERE f.IS_CURRENT = TRUE
        GROUP BY f.VENDOR_ID, f.REPORT_DATE
    )

    -- Final output: triage rows enriched with health metrics
    SELECT
        -- Orphan triage detail
        o.COMP_HK,
        o.COMP_ID,
        o.COMP_NAME,
        o.VENDOR_ID,
        o.VENDOR_NAME,
        o.INGESTION_ID,
        o.REPORT_DATE,
        o.ATTR_ID,
        o.ATTR_NAME,
        o.ATTR_VALUE,
        o.SOURCE_TIMESTAMP,
        o.EFF_FROM,
        o.PRIORITY_RANK,
        -- Health metrics for this vendor + date
        hm.orphan_count,
        hm.total_current_count,
        hm.orphan_pct,
        hm.alert_status
    FROM raw_orphans o
    JOIN health_metrics hm
      ON o.VENDOR_ID   = hm.VENDOR_ID
     AND o.REPORT_DATE = hm.REPORT_DATE
    ORDER BY hm.orphan_pct DESC, o.VENDOR_ID, o.INGESTION_ID;
    ```
  </view_template>

  <remediation_guidance>
    When `vw_unmapped_entities` returns rows, the recommended remediation path is:
    1. **Identify** the COMP_HK and COMP_ID from the view output.
    2. **Locate** the entity in the Internal App DB using the COMP_ID as a search key.
    3. **Update** the `company_dim` table to set OBLIGOR_ID to the resolved value.
    4. **Re-run** the dbt incremental model (financial_observations_fact) to propagate
       the corrected OBLIGOR_ID — the IS_CURRENT SCD2 row will be updated via ATTR_HASH.
    5. **Monitor** `vw_unmapped_entities` post-run to confirm orphan_count drops to 0
       for the resolved COMP_HK.

    DO NOT delete PENDING_MAP rows. The SCD2 history must be preserved for audit.
  </remediation_guidance>

  <second_brain_update_contract>
    After generating the view, update `.context/data_model_brain.json` with:
    ```json
    {
      "views": {
        "vw_unmapped_entities": {
          "status": "GENERATED",
          "triage_grain": ["COMP_HK", "VENDOR_ID", "INGESTION_ID", "REPORT_DATE"],
          "alert_threshold_pct": 5.0,
          "zero_data_loss_enforced": true,
          "pending_placeholders": ["<list of [[ PLACEHOLDER: ... ]] items unresolved>"]
        }
      },
      "exception_mgmt": {
        "pending_map_alert_threshold_pct": 5.0
      }
    }
    ```
  </second_brain_update_contract>

  <success_criteria>
    - [ ] View reads OBLIGOR_ID = 'PENDING_MAP' as the canonical unmapped signal.
    - [ ] Zero DELETE or TRUNCATE statements generated for PENDING_MAP records.
    - [ ] IS_CURRENT = TRUE filter applied to both orphan detection and total count CTEs.
    - [ ] orphan_pct calculated correctly using NULLIF to guard against division by zero.
    - [ ] alert_status emits '[WARNING]' when orphan_pct > 5.0 (default threshold).
    - [ ] Oracle and Hive portability annotations present as inline comments.
    - [ ] Second Brain updated with threshold and triage grain.
  </success_criteria>

  <echo>PENDING_MAP_EXCEPTION_TRACKER_v1_READY</echo>
</skill_definition>
