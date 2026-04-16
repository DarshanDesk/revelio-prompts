---
name: hash-and-delete-handler
description: "Implements hard-delete anti-join detection and priority MERGE via ROW_NUMBER. Activates when detecting deleted source rows, deduplicating on ATTR_HK, or constructing MERGE statements. Does not cover DDL generation or SCD2 open-record logic."
metadata:
  version: "1.0.0"
compatibility: "DuckDB (PoC) | Oracle | Hive (portability)"
---

<skill_definition name="hash_and_delete_handler">
  <metadata>
    <second_brain_ref>.github/context-cache/SCHEMA.md § SCD2 Column Name Authority | RULES.md § Priority Ranking</second_brain_ref>
  </metadata>


  <canonical_facts>
    <fact id="DELETE-001">Logical delete only: SET IS_CURRENT = FALSE, EFF_TO = CURRENT_TIMESTAMP. NEVER use DELETE or TRUNCATE on PENDING_MAP or expired SCD2 records.</fact>
    <fact id="DELETE-002">EFF_TO column name: EFF_TO (TIMESTAMP). FORBIDDEN: VALID_TO_DT</fact>
    <fact id="PRIORITY-001">Priority ranking (RULES.md): 1=Analyst Override, 2=S&amp;P Incremental, 3=S&amp;P Full Feed. Lowest integer wins.</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. Hard-Delete Detection Pattern (Full Feed Only)
    During a `Full Feed` load, attributes absent from the source file indicate logical deletes.
    Use an anti-join to identify them:

    ```sql
    -- Identify IS_CURRENT records that are absent from the incoming Full Feed
    WITH absent_records AS (
        SELECT tgt.COMP_HK, tgt.PERIOD_ID, tgt.ATTR_ID
        FROM {{ this }} tgt
        WHERE tgt.IS_CURRENT = TRUE
          AND tgt.VENDOR_ID  = '{{ vendor_id }}'     -- scope to this vendor's feed
          AND tgt.REPORT_DATE = '{{ report_date }}'  -- scope to this load's report date
          -- Anti-join: not present in source
          AND NOT EXISTS (
              SELECT 1
              FROM {{ source_cte }} src
              WHERE src.COMP_HK  = tgt.COMP_HK
                AND src.PERIOD_ID = tgt.PERIOD_ID
                AND src.ATTR_ID   = tgt.ATTR_ID
          )
    )
    -- Logically expire the absent records
    UPDATE {{ this }}
    SET
        IS_CURRENT = FALSE,
        EFF_TO     = CURRENT_TIMESTAMP
    WHERE IS_CURRENT = TRUE
      AND (COMP_HK, PERIOD_ID, ATTR_ID) IN (
          SELECT COMP_HK, PERIOD_ID, ATTR_ID FROM absent_records
      );
    -- [ORACLE_PORT]: ROW tuple comparison may need AND-chained conditions.
    -- [HIVE_PORT]:   ACID tables required for UPDATE; use INSERT OVERWRITE pattern instead.
    ```

    > **ZERO DATA LOSS**: Expired records remain queryable with IS_CURRENT = FALSE and
    > EFF_FROM / EFF_TO forming a closed temporal window. They are never purged.

    ### 2. Full-Feed Trigger Guard
    Only execute the hard-delete anti-join when the load type is `FULL_FEED`.
    Use a Jinja conditional or a configuration variable:

    ```sql
    {% if var('load_type', 'INCREMENTAL') == 'FULL_FEED' %}
        -- Execute absent_records CTE + UPDATE above
    {% endif %}
    ```

    ### 3. Priority-Based Conflict Resolution
    Before writing to the target table, deduplicate by the canonical grain
    `(COMP_ID, PERIOD_ID, ATTR_ID)` using ROW_NUMBER():

    ```sql
    WITH priority_resolved AS (
        SELECT
            *,
            ROW_NUMBER() OVER (
                PARTITION BY COMP_ID, PERIOD_ID, ATTR_ID
                ORDER BY
                    PRIORITY_RANK     ASC,   -- 1=Analyst wins
                    SOURCE_TIMESTAMP  DESC   -- latest timestamp breaks ties within same rank
            ) AS rn
        FROM {{ source_union_cte }}  -- union of all source feeds for this load
    )
    SELECT * EXCLUDE(rn)
    FROM priority_resolved
    WHERE rn = 1
    -- [ORACLE_PORT]: Replace SELECT * EXCLUDE(rn) with explicit column list minus rn.
    -- [HIVE_PORT]:   Use QUALIFY rn = 1 if available; otherwise wrap in outer SELECT.
    ```

    ### 4. Source Union for Multi-Feed Loads
    When multiple feeds arrive in the same load window (e.g., S&P Full + Analyst Override),
    union them before priority resolution:

    ```sql
    WITH source_union_cte AS (
        SELECT *, 3 AS PRIORITY_RANK FROM {{ ref('sp_full_feed_stage') }}
        UNION ALL
        SELECT *, 2 AS PRIORITY_RANK FROM {{ ref('sp_incremental_stage') }}
        UNION ALL
        SELECT *, 1 AS PRIORITY_RANK FROM {{ ref('analyst_override_stage') }}
    )
    ```

    ### 5. Execution Order Within a Load Cycle
    ```
    Step 1 → priority_resolved CTE    (deduplicate incoming by PRIORITY_RANK)
    Step 2 → is_incremental() block   (expire changed, insert new — see scd2-incremental-engine.md)
    Step 3 → hard-delete anti-join     (expire absent records — Full Feed only)
    ```
    > Step 3 must follow Step 2 to avoid expiring records that were just re-inserted
    > as part of the incremental merge.

  </instruction_set>

  <portability_notes>
    - DuckDB: Supports tuple IN ((col1, col2)) syntax and `SELECT * EXCLUDE(col)`
    - Oracle: Use explicit column list; tuple comparison in WHERE needs AND-chaining
    - Hive: ACID table required for UPDATE/DELETE; for non-ACID, use INSERT OVERWRITE full-partition reload pattern
  </portability_notes>

</skill_definition>
