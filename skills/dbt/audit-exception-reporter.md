<skill_definition name="audit_exception_reporter">
  <metadata>
    <version>1.0.0</version>
    <capability>PENDING_MAP Alerting, Orphan Notification, Restatement Lineage Reporting</capability>
    <target_engine>DuckDB (PoC) native SQL | dbt test framework</target_engine>
    <owning_agent>financial-audit-pro</owning_agent>
    <second_brain_ref>.github/context-cache/RULES.md § PENDING_MAP Threshold | RULES.md § Zero Data Loss Policy</second_brain_ref>
    <sourced_from>Extracted from financial-audit-pro.agent.md inline blocks: orphan_notification + audit_lineage_reporting</sourced_from>
  </metadata>

  <description>
    This skill generates the exception and reporting surface for the `financial-audit-pro` agent:

    1. **PENDING_MAP Orphan Notification**: SQL alerts to identify OBLIGOR_ID = 'PENDING_MAP'
       records in the fact table that are linked to active (IS_CURRENT = TRUE) rows.
       Emits [WARNING] when orphan percentage exceeds the canonical 5.0% threshold
       (read from RULES.md — do NOT hardcode a different value).

    2. **Audit Lineage Reporting**: SQL queries summarising ATTR_HASH mismatches (restatements)
       from 2-year reconciliation loads to verify REPORT_DATE accuracy and identify
       which COMP_ID / ATTR_ID combinations were restated.

    THRESHOLD (from RULES.md § PENDING_MAP Threshold — NON-NEGOTIABLE):
      pending_map_alert_threshold_pct = 5.0
    FORBIDDEN: Any threshold other than 5.0 in this skill file is a DRIFT defect.
  </description>

  <canonical_facts>
    <fact id="EXC-001">PENDING_MAP threshold: 5.0%. Emit [WARNING] if orphan_pct > 5.0. Source: RULES.md § PENDING_MAP Threshold.</fact>
    <fact id="EXC-002">IS_CURRENT filter: IS_CURRENT = TRUE. FORBIDDEN: IS_CURRENT = 1</fact>
    <fact id="EXC-003">NEVER DELETE or TRUNCATE PENDING_MAP records. They exist for remediation, not deletion.</fact>
    <fact id="EXC-004">Triage grain: (COMP_HK, VENDOR_ID, INGESTION_ID, REPORT_DATE)</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. PENDING_MAP Orphan Notification View
    Surfaces all IS_CURRENT = TRUE records where OBLIGOR_ID = 'PENDING_MAP',
    with full triage context for the operations team:

    ```sql
    -- models/vw_pending_map_orphans.sql
    -- Raw orphan triage layer — all unmapped IS_CURRENT records
    WITH raw_orphans AS (
        SELECT
            f.COMP_HK,
            f.COMP_ID,
            c.COMP_NAME,
            f.VENDOR_ID,
            f.INGESTION_ID,
            f.REPORT_DATE,
            f.ATTR_ID,
            m.ATTR_NAME,
            f.ATTR_VALUE,
            f.SOURCE_TIMESTAMP,
            f.EFF_FROM
        FROM {{ ref('financial_observations_fact') }} f
        LEFT JOIN {{ ref('company_dim') }}            c  ON c.COMP_HK  = f.COMP_HK
        LEFT JOIN {{ ref('attr_metadata_dim') }}      m  ON m.ATTR_ID  = f.ATTR_ID
        WHERE f.IS_CURRENT  = TRUE
          AND f.OBLIGOR_ID  = 'PENDING_MAP'
    ),
    -- Health metric layer — per (VENDOR_ID, REPORT_DATE) group
    health_metrics AS (
        SELECT
            f.VENDOR_ID,
            f.REPORT_DATE,
            COUNT(*) FILTER (WHERE f.OBLIGOR_ID = 'PENDING_MAP')          AS orphan_count,
            COUNT(*)                                                        AS total_is_current,
            ROUND(
                COUNT(*) FILTER (WHERE f.OBLIGOR_ID = 'PENDING_MAP')
                * 100.0 / NULLIF(COUNT(*), 0),
            2)                                                             AS orphan_pct,
            CASE
                WHEN ROUND(
                    COUNT(*) FILTER (WHERE f.OBLIGOR_ID = 'PENDING_MAP')
                    * 100.0 / NULLIF(COUNT(*), 0),
                2) > 5.0
                THEN '[WARNING] PENDING_MAP exceeds 5.0% threshold'
                ELSE '[OK]'
            END AS alert_status
            -- 5.0 sourced from RULES.md § PENDING_MAP Threshold — do NOT change this value
        FROM {{ ref('financial_observations_fact') }} f
        WHERE f.IS_CURRENT = TRUE
        GROUP BY f.VENDOR_ID, f.REPORT_DATE
    )
    -- Final output: orphan detail joined with health signal
    SELECT
        o.*,
        h.orphan_count,
        h.total_is_current,
        h.orphan_pct,
        h.alert_status
    FROM raw_orphans   o
    JOIN health_metrics h
      ON  h.VENDOR_ID   = o.VENDOR_ID
      AND h.REPORT_DATE = o.REPORT_DATE
    ORDER BY h.orphan_pct DESC, o.REPORT_DATE DESC
    -- [ORACLE_PORT]: COUNT(*) FILTER (WHERE ...) → COUNT(CASE WHEN ... THEN 1 END)
    -- [HIVE_PORT]:   COUNT(*) FILTER → SUM(CASE WHEN ... THEN 1 ELSE 0 END)
    ```

    ### 2. PENDING_MAP dbt Singular Test (Alert Gate)
    A dbt singular test that fails the pipeline when the 5.0% threshold is breached:

    ```sql
    -- tests/assert_pending_map_below_threshold.sql
    -- Fails when any (VENDOR_ID, REPORT_DATE) group exceeds 5.0% PENDING_MAP.
    SELECT
        VENDOR_ID,
        REPORT_DATE,
        ROUND(
            COUNT(*) FILTER (WHERE OBLIGOR_ID = 'PENDING_MAP')
            * 100.0 / NULLIF(COUNT(*), 0),
        2) AS orphan_pct,
        '[WARNING] PENDING_MAP exceeds 5.0% threshold' AS violation_reason
    FROM {{ ref('financial_observations_fact') }}
    WHERE IS_CURRENT = TRUE
    GROUP BY VENDOR_ID, REPORT_DATE
    HAVING ROUND(
        COUNT(*) FILTER (WHERE OBLIGOR_ID = 'PENDING_MAP')
        * 100.0 / NULLIF(COUNT(*), 0),
    2) > 5.0
    ```

    ### 3. Audit Lineage Report — Restatement Detection
    Identifies ATTR_HASH mismatches between load batches for the same grain,
    surfacing which records were restated and across which REPORT_DATE pair:

    ```sql
    -- models/vw_restatement_audit_report.sql
    WITH closed_records AS (
        -- Historical (expired) records — candidates for restatement detection
        SELECT
            COMP_HK,
            COMP_ID,
            PERIOD_ID,
            ATTR_ID,
            ATTR_VALUE,
            ATTR_HASH,
            REPORT_DATE         AS original_report_date,
            EFF_FROM            AS original_eff_from,
            EFF_TO              AS expiry_timestamp,
            INGESTION_ID        AS original_ingestion_id
        FROM {{ ref('financial_observations_fact') }}
        WHERE IS_CURRENT = FALSE
          AND EFF_TO IS NOT NULL
    ),
    current_records AS (
        SELECT
            COMP_HK,
            COMP_ID,
            PERIOD_ID,
            ATTR_ID,
            ATTR_VALUE,
            ATTR_HASH,
            REPORT_DATE         AS restated_report_date,
            EFF_FROM            AS restatement_timestamp,
            INGESTION_ID        AS restatement_ingestion_id
        FROM {{ ref('financial_observations_fact') }}
        WHERE IS_CURRENT = TRUE
    )
    SELECT
        c.COMP_HK,
        c.COMP_ID,
        c.PERIOD_ID,
        c.ATTR_ID,
        h.original_report_date,
        c.restated_report_date,
        h.ATTR_VALUE                                AS original_value,
        c.ATTR_VALUE                                AS restated_value,
        h.ATTR_HASH                                 AS original_hash,
        c.ATTR_HASH                                 AS restated_hash,
        h.expiry_timestamp                          AS restatement_effective_ts,
        h.original_ingestion_id,
        c.restatement_ingestion_id,
        'ATTR_HASH mismatch — restatement detected between load batches' AS audit_note
    FROM current_records  c
    JOIN closed_records   h
      ON  h.COMP_HK   = c.COMP_HK
      AND h.PERIOD_ID  = c.PERIOD_ID
      AND h.ATTR_ID    = c.ATTR_ID
      AND h.ATTR_HASH != c.ATTR_HASH   -- Hash divergence = restatement
    ORDER BY h.expiry_timestamp DESC
    ```

    ### 4. Brain Update on Findings
    After running the above reports, emit `[BRAIN-UPDATE-PENDING]` markers:
    ```
    [BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: Step 4 complete — FINANCIAL_AUDIT_PRO_v1_READY]
    [BRAIN-UPDATE-PENDING: RULES.md: PENDING_MAP_THRESHOLD: Observed orphan_pct = X.X% for VENDOR_ID/REPORT_DATE]
    [BRAIN-UPDATE-PENDING: BRAIN.md: AUDIT_FINDINGS: Restatements detected for COMP_IDs: <list>; ATTR_IDs: <list>]
    ```

  </instruction_set>

</skill_definition>
