---
name: audit-test-generator
description: "Generates dbt data quality tests including null checks, ratio range tests, regex casting, and idempotency assertions. Activates when producing dbt test YAML, custom generic tests, or PENDING_MAP ratio assertions. Does not cover SCD2 overlap checks."
metadata:
  version: "1.0.0"
compatibility: "DuckDB (PoC) native SQL | dbt test framework"
---

<skill_definition name="audit_test_generator">
  <metadata>
    <second_brain_ref>.github/context-cache/SCHEMA.md § SCD2 Column Name Authority | SCHEMA.md § EAV Column Contract</second_brain_ref>
  </metadata>


  <canonical_facts>
    <fact id="TEST-001">IS_CURRENT filter in all test SQL: IS_CURRENT = TRUE. FORBIDDEN: IS_CURRENT = 1</fact>
    <fact id="TEST-002">ATTR_VALUE is VARCHAR — use REGEXP_MATCHES to validate before numeric transforms.</fact>
    <fact id="TEST-003">dbt-utils is FORBIDDEN — all test macros use native DuckDB SQL only.</fact>
    <fact id="TEST-004">Idempotency: re-running Full Feed with same source MUST produce COUNT(*) = 0 new rows against target.</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. Null Check Tests (Mandatory Audit Columns)
    Generate singular dbt tests that fail if mandatory audit columns are NULL on IS_CURRENT records:

    ```sql
    -- tests/assert_audit_columns_not_null.sql
    -- Fails if any IS_CURRENT record is missing mandatory audit metadata.
    SELECT
        COMP_HK,
        PERIOD_ID,
        ATTR_ID,
        CASE
            WHEN INGESTION_ID    IS NULL THEN 'INGESTION_ID null'
            WHEN REPORT_DATE     IS NULL THEN 'REPORT_DATE null'
            WHEN SOURCE_TRACE_ID IS NULL THEN 'SOURCE_TRACE_ID null'
        END AS violation_reason
    FROM {{ ref('financial_observations_fact') }}
    WHERE IS_CURRENT = TRUE
      AND (
          INGESTION_ID    IS NULL
          OR REPORT_DATE     IS NULL
          OR SOURCE_TRACE_ID IS NULL
      )
    -- dbt singular test: returns rows = test fails
    ```

    ### 2. Financial Ratio Range Tests
    For ATTR_IDs classified as `DATA_TYPE = 'DECIMAL'` in attr_metadata_dim,
    validate that ATTR_VALUE can be safely cast and falls within plausible financial bounds:

    ```sql
    -- tests/assert_financial_ratio_range.sql
    WITH ratio_records AS (
        SELECT
            f.COMP_HK,
            f.ATTR_ID,
            f.ATTR_VALUE,
            TRY_CAST(f.ATTR_VALUE AS DECIMAL(18,2)) AS attr_value_numeric
        FROM {{ ref('financial_observations_fact') }} f
        JOIN {{ ref('attr_metadata_dim') }}          m
          ON m.ATTR_ID    = f.ATTR_ID
         AND m.DATA_TYPE  = 'DECIMAL'
        WHERE f.IS_CURRENT = TRUE
    )
    SELECT
        COMP_HK,
        ATTR_ID,
        ATTR_VALUE,
        'TRY_CAST returned NULL — non-numeric value in DECIMAL attribute' AS violation_reason
    FROM ratio_records
    WHERE attr_value_numeric IS NULL   -- TRY_CAST failed: string is not a valid number
    -- [ORACLE_PORT]: Replace TRY_CAST with TO_NUMBER(ATTR_VALUE, '999999999.99') in EXCEPTION block
    ```

    ### 3. ATTR_VALUE Defensive Regex Casting Test
    For DECIMAL attributes, validate the raw string pattern before any cast:

    ```sql
    -- tests/assert_attr_value_numeric_format.sql
    SELECT
        f.COMP_HK,
        f.ATTR_ID,
        f.ATTR_VALUE,
        'ATTR_VALUE does not match numeric pattern' AS violation_reason
    FROM {{ ref('financial_observations_fact') }} f
    JOIN {{ ref('attr_metadata_dim') }}           m
      ON m.ATTR_ID   = f.ATTR_ID
     AND m.DATA_TYPE = 'DECIMAL'
    WHERE f.IS_CURRENT = TRUE
      AND NOT regexp_matches(f.ATTR_VALUE, '^-?[0-9]+(\.[0-9]+)?$')
    -- DuckDB: regexp_matches returns BOOLEAN
    -- [ORACLE_PORT]: REGEXP_LIKE(ATTR_VALUE, '^-?[0-9]+(\.[0-9]+)?$')
    -- [HIVE_PORT]:   ATTR_VALUE RLIKE '^-?[0-9]+(\\.[0-9]+)?$'
    ```

    ### 4. Idempotency Test
    Verify that a Full Feed re-run with identical source data produces zero new inserted rows.
    This test compares pre/post row counts on IS_CURRENT records:

    ```sql
    -- tests/assert_full_feed_idempotency.sql
    -- Pre-condition: run this BEFORE and AFTER a second identical full-feed load.
    -- The delta should be zero.
    WITH current_counts AS (
        SELECT
            VENDOR_ID,
            REPORT_DATE,
            COUNT(*) AS is_current_count
        FROM {{ ref('financial_observations_fact') }}
        WHERE IS_CURRENT = TRUE
        GROUP BY VENDOR_ID, REPORT_DATE
    ),
    expected_counts AS (
        -- Expected: same counts as captured before the re-run
        -- Populate via dbt seed or variable
        SELECT
            VENDOR_ID,
            REPORT_DATE,
            COUNT(*) AS expected_count
        FROM {{ ref('idempotency_baseline_seed') }}
        GROUP BY VENDOR_ID, REPORT_DATE
    )
    SELECT
        c.VENDOR_ID,
        c.REPORT_DATE,
        c.is_current_count  AS actual_count,
        e.expected_count,
        c.is_current_count - e.expected_count AS delta,
        'Idempotency violated: Full Feed re-run produced new IS_CURRENT rows' AS violation_reason
    FROM current_counts  c
    JOIN expected_counts e
      ON e.VENDOR_ID   = c.VENDOR_ID
     AND e.REPORT_DATE = c.REPORT_DATE
    WHERE c.is_current_count != e.expected_count
    ```

    ### 5. Schema Test Declarations (schema.yml)
    Standard column-level tests for the `financial_observations_fact` model:

    ```yaml
    # models/schema.yml
    models:
      - name: financial_observations_fact
        columns:
          - name: COMP_HK
            tests:
              - not_null
              - unique:
                  where: "IS_CURRENT = TRUE AND PERIOD_ID IS NOT NULL AND ATTR_ID IS NOT NULL"
          - name: ATTR_VALUE
            tests:
              - not_null:
                  where: "IS_CURRENT = TRUE"
          - name: IS_CURRENT
            tests:
              - not_null
              - accepted_values:
                  values: [true, false]
          - name: EFF_FROM
            tests:
              - not_null
    ```

  </instruction_set>

</skill_definition>
