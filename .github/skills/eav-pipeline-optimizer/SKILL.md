---
name: eav-pipeline-optimizer
description: "Implements OBLIGOR mapping reconciliation and ATTR_VALUE pass-through for the EAV pipeline. Activates when handling LEFT JOIN obligor bridging, COALESCE PENDING_MAP sentinel, or EAV attribute routing. Does not cover DDL generation."
metadata:
  version: "1.0.0"
compatibility: "DuckDB (PoC) | Oracle | Hive (portability)"
---

<skill_definition name="eav_pipeline_optimizer">
  <metadata>
    <second_brain_ref>.github/context-cache/SCHEMA.md § EAV Column Contract | RULES.md § Zero Data Loss Policy</second_brain_ref>
  </metadata>


  <canonical_facts>
    <fact id="EAV-001">ATTR_VALUE is VARCHAR(4000). Single column. NEVER create typed split columns.</fact>
    <fact id="EAV-002">OBLIGOR_ID fallback: COALESCE(c.OBLIGOR_ID, 'PENDING_MAP') — applied at model layer, not DDL.</fact>
    <fact id="EAV-003">Zero Data Loss: Records must load even when OBLIGOR mapping is absent. PENDING_MAP is the sentinel.</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. OBLIGOR Mapping Reconciliation Pattern
    Apply a LEFT JOIN from the staging table to the Internal App DB mapping table.
    The COALESCE ensures Zero Data Loss when no mapping exists:

    ```sql
    WITH mapped_records AS (
        SELECT
            src.COMP_HK,
            src.COMP_ID,
            src.VENDOR_ID,
            src.PERIOD_ID,
            src.ATTR_ID,
            src.ATTR_VALUE,           -- VARCHAR(4000): do NOT cast here
            src.REPORT_DATE,
            src.INGESTION_ID,
            src.SOURCE_TIMESTAMP,
            src.SOURCE_TRACE_ID,
            src.PRIORITY_RANK,
            COALESCE(map.OBLIGOR_ID, 'PENDING_MAP') AS OBLIGOR_ID
            -- COALESCE is the Zero Data Loss contract — never filter out unmapped records
        FROM {{ ref('sp_financial_stage') }} src
        LEFT JOIN {{ ref('obligor_mapping') }} map
          ON  map.COMP_ID   = src.COMP_ID
          AND map.VENDOR_ID = src.VENDOR_ID
    )
    SELECT * FROM mapped_records
    ```

    > **NON-NEGOTIABLE**: Use LEFT JOIN — never INNER JOIN. An INNER JOIN silently drops
    > unmapped records, violating the Zero Data Loss policy.

    ### 2. PENDING_MAP Audit Emission
    After the mapping step, emit a `[BRAIN-UPDATE-PENDING]` marker if the PENDING_MAP
    percentage is at or near the 5.0% threshold:

    ```sql
    -- In the Learn stage, compute and surface the orphan percentage:
    SELECT
        VENDOR_ID,
        REPORT_DATE,
        COUNT(*) FILTER (WHERE OBLIGOR_ID = 'PENDING_MAP') AS orphan_count,
        COUNT(*)                                            AS total_count,
        ROUND(
            COUNT(*) FILTER (WHERE OBLIGOR_ID = 'PENDING_MAP') * 100.0
            / NULLIF(COUNT(*), 0),
        2) AS orphan_pct
    FROM mapped_records
    GROUP BY VENDOR_ID, REPORT_DATE
    ```

    Emit: `[BRAIN-UPDATE-PENDING: RULES.md: PENDING_MAP_THRESHOLD: Observed orphan_pct for this load]`

    ### 3. EAV Join Optimization Patterns
    EAV tables grow wide in row count due to the long format. Apply these patterns:

    **3a. Filter before joining (push-down predicate)**:
    ```sql
    -- Apply date and vendor filters on the stage CTE before the OBLIGOR join
    WITH filtered_stage AS (
        SELECT *
        FROM {{ ref('sp_financial_stage') }}
        WHERE REPORT_DATE >= '{{ var("min_report_date") }}'
          AND VENDOR_ID    = '{{ var("vendor_id") }}'
    )
    -- Join the filtered set — avoids full EAV table scan during mapping
    ```

    **3b. ATTR_ID filtering for incremental loads**:
    ```sql
    -- In is_incremental() blocks, scope the source to only the attrs arriving in this load:
    WHERE ATTR_ID IN (SELECT DISTINCT ATTR_ID FROM {{ source_cte }})
    ```

    **3c. DuckDB EXPLAIN usage** (PoC diagnostics):
    ```sql
    EXPLAIN SELECT * FROM financial_observations_fact WHERE IS_CURRENT = TRUE;
    -- Inspect for sequential scans on large EAV tables; consider ORDER BY EFF_FROM
    -- for DuckDB's micro-partition pruning
    ```

    ### 4. ATTR_VALUE Handling Rules
    ```sql
    -- CORRECT: Pass ATTR_VALUE as VARCHAR through the EAV model layer
    src.ATTR_VALUE AS ATTR_VALUE    -- no cast — casting happens in vw_financial_latest_gold

    -- FORBIDDEN: Splitting into typed columns
    CAST(src.ATTR_VALUE AS DECIMAL(18,2)) AS ATTR_VALUE_NUM  -- VIOLATION

    -- FORBIDDEN: Implicit cast in SCD2 hash
    md5(concat(..., CAST(ATTR_VALUE AS INTEGER), ...))        -- VIOLATION: cast before hash
    ```

    ### 5. Portability Notes for Large EAV Joins
    | Engine | Optimization Pattern |
    |--------|---------------------|
    | DuckDB | ORDER BY EFF_FROM for micro-partition pruning; columnar storage handles EAV well |
    | Oracle | Partition by VENDOR_ID + REPORT_DATE; use COMPRESS for EAV repetition |
    | Hive   | Partition by REPORT_DATE and VENDOR_ID; use ORC with predicate pushdown |

  </instruction_set>

</skill_definition>
