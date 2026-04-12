<skill_definition name="scd2_integrity_validator">
  <metadata>
    <version>1.0.0</version>
    <capability>SCD2 Temporal Integrity Checks — Overlap Detection, IS_CURRENT Uniqueness, Gap Validation</capability>
    <target_engine>DuckDB (PoC) native SQL | dbt test framework</target_engine>
    <owning_agent>financial-audit-pro</owning_agent>
    <second_brain_ref>.github/context-cache/SCHEMA.md § SCD2 Column Name Authority</second_brain_ref>
    <sourced_from>Extracted from financial-audit-pro.agent.md inline blocks: temporal_overlap_detection + casting_guardrails (SCD2-specific validation aspects)</sourced_from>
  </metadata>

  <description>
    This skill generates native DuckDB SQL tests (no dbt-utils) to validate the SCD2
    temporal integrity of `financial_observations_fact`:

    1. **IS_CURRENT Uniqueness**: No more than one IS_CURRENT = TRUE row per grain
       (COMP_HK, PERIOD_ID, ATTR_ID).

    2. **Temporal Non-Contradiction (Overlap Detection)**: No two records for the same grain
       have overlapping EFF_FROM — EFF_TO windows.

    3. **EFF_FROM / EFF_TO Ordering**: EFF_FROM must precede EFF_TO on all closed records.

    COLUMN CONTRACT (loaded from SCHEMA.md § SCD2 Column Name Authority — MUST be loaded in Perceive):
      EFF_FROM    — TIMESTAMP — inclusive start of validity (SCD2 start)
      EFF_TO      — TIMESTAMP — exclusive end; NULL on IS_CURRENT = TRUE records
      IS_CURRENT  — BOOLEAN  — TRUE = authoritative current record
    FORBIDDEN: VALID_FROM_DT, VALID_TO_DT, IS_CURRENT = 1
  </description>

  <canonical_facts>
    <fact id="SCD2-VALID-001">IS_CURRENT = TRUE must be unique per grain (COMP_HK, PERIOD_ID, ATTR_ID). Multiple TRUE rows = broken SCD2.</fact>
    <fact id="SCD2-VALID-002">EFF_TO must be NULL on every IS_CURRENT = TRUE record (open record contract).</fact>
    <fact id="SCD2-VALID-003">No two records for the same grain may have overlapping EFF_FROM — EFF_TO windows (Temporal Non-Contradiction).</fact>
    <fact id="SCD2-VALID-004">EFF_FROM must be strictly less than EFF_TO on all closed records (EFF_TO IS NOT NULL).</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. IS_CURRENT Uniqueness Test
    Returns rows where more than one IS_CURRENT = TRUE record exists for the same grain — the most fundamental SCD2 integrity check:

    ```sql
    -- tests/assert_is_current_unique_per_grain.sql
    -- Fails if any grain has duplicate IS_CURRENT = TRUE records.
    WITH current_duplicates AS (
        SELECT
            COMP_HK,
            PERIOD_ID,
            ATTR_ID,
            COUNT(*) AS current_count
        FROM {{ ref('financial_observations_fact') }}
        WHERE IS_CURRENT = TRUE
        GROUP BY COMP_HK, PERIOD_ID, ATTR_ID
        HAVING COUNT(*) > 1
    )
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        current_count,
        'Multiple IS_CURRENT = TRUE rows for same grain — SCD2 integrity violated' AS violation_reason
    FROM current_duplicates
    ```

    ### 2. Open Record Contract Test
    Every IS_CURRENT = TRUE record must have EFF_TO = NULL (open temporal window):

    ```sql
    -- tests/assert_open_record_eff_to_null.sql
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        EFF_FROM,
        EFF_TO,
        'IS_CURRENT = TRUE but EFF_TO is not NULL — open record contract violated' AS violation_reason
    FROM {{ ref('financial_observations_fact') }}
    WHERE IS_CURRENT = TRUE
      AND EFF_TO IS NOT NULL
    ```

    ### 3. Temporal Overlap Detection Test
    Identifies overlapping EFF_FROM — EFF_TO windows for the same grain.
    Uses a self-join with DuckDB window functions:

    ```sql
    -- tests/assert_no_temporal_overlap.sql
    -- Fails if any two records for the same grain have overlapping validity windows.
    WITH ordered_records AS (
        SELECT
            COMP_HK,
            PERIOD_ID,
            ATTR_ID,
            EFF_FROM,
            EFF_TO,
            LEAD(EFF_FROM) OVER (
                PARTITION BY COMP_HK, PERIOD_ID, ATTR_ID
                ORDER BY EFF_FROM ASC
            ) AS next_eff_from
        FROM {{ ref('financial_observations_fact') }}
    )
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        EFF_FROM,
        EFF_TO,
        next_eff_from,
        'Temporal overlap detected: EFF_TO > next record EFF_FROM for same grain' AS violation_reason
    FROM ordered_records
    WHERE EFF_TO IS NOT NULL                   -- Only check closed records
      AND EFF_TO > next_eff_from               -- Closed window extends past next record's start
      AND next_eff_from IS NOT NULL
    -- [ORACLE_PORT]: LEAD() syntax is identical; ensure TIMESTAMP comparison is valid
    -- [HIVE_PORT]:   LEAD() supported in Hive 0.11+; TIMESTAMP comparison applies
    ```

    ### 4. EFF_FROM < EFF_TO Ordering Test
    All closed records (EFF_TO IS NOT NULL) must have EFF_FROM strictly before EFF_TO:

    ```sql
    -- tests/assert_eff_from_before_eff_to.sql
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        EFF_FROM,
        EFF_TO,
        'EFF_FROM >= EFF_TO on closed record — temporal ordering violated' AS violation_reason
    FROM {{ ref('financial_observations_fact') }}
    WHERE EFF_TO IS NOT NULL
      AND EFF_FROM >= EFF_TO
    ```

    ### 5. Closed Record Completeness Test
    Every IS_CURRENT = FALSE record must have a non-NULL EFF_TO (properly closed):

    ```sql
    -- tests/assert_closed_records_have_eff_to.sql
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        EFF_FROM,
        IS_CURRENT,
        'IS_CURRENT = FALSE but EFF_TO is NULL — closed record not properly expired' AS violation_reason
    FROM {{ ref('financial_observations_fact') }}
    WHERE IS_CURRENT = FALSE
      AND EFF_TO IS NULL
    ```

    ### 6. SCD2 Timeline Continuity Test (2-Year Reconciliation)
    For a given grain, verify the temporal windows form a gapless chain:

    ```sql
    -- tests/assert_scd2_timeline_continuity.sql
    -- Warning: informational — gaps may be legitimate if data was not reported in a period.
    -- Flag for review rather than hard failure.
    WITH ordered_records AS (
        SELECT
            COMP_HK,
            PERIOD_ID,
            ATTR_ID,
            EFF_FROM,
            EFF_TO,
            LAG(EFF_TO) OVER (
                PARTITION BY COMP_HK, PERIOD_ID, ATTR_ID
                ORDER BY EFF_FROM ASC
            ) AS prev_eff_to
        FROM {{ ref('financial_observations_fact') }}
    )
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        EFF_FROM,
        prev_eff_to,
        'Gap in SCD2 timeline: EFF_FROM does not immediately follow previous EFF_TO' AS violation_reason
    FROM ordered_records
    WHERE prev_eff_to IS NOT NULL
      AND EFF_FROM != prev_eff_to   -- Gap exists between prev close and this open
    ```

  </instruction_set>

</skill_definition>
