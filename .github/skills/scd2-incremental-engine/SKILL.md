---
name: scd2-incremental-engine
description: "Implements the SCD2 is_incremental() state machine and surrogate key logic. Activates when handling open-record detection, EFF_FROM/EFF_TO close logic, or COMP_HK/ATTR_HASH generation. Does not cover DDL generation or CSV seed creation."
metadata:
  version: "1.0.0"
compatibility: "DuckDB (PoC) | Oracle | Hive (portability)"
---

<skill_definition name="scd2_incremental_engine">
  <metadata>
    <second_brain_ref>.github/context-cache/SCHEMA.md § SCD2 Column Name Authority</second_brain_ref>
  </metadata>


  <canonical_facts>
    <fact id="COL-001">SCD2 start column: EFF_FROM (TIMESTAMP). FORBIDDEN: VALID_FROM_DT, dbt_valid_from</fact>
    <fact id="COL-002">SCD2 end column: EFF_TO (TIMESTAMP). FORBIDDEN: VALID_TO_DT, dbt_valid_to</fact>
    <fact id="COL-003">Current flag: IS_CURRENT BOOLEAN. Filter: IS_CURRENT = TRUE. FORBIDDEN: IS_CURRENT = 1</fact>
    <fact id="HASH-001">Surrogate key: md5(concat(VENDOR_ID, '|', COMP_ID)) — native SQL only. NO dbt-utils.</fact>
    <fact id="HASH-002">Change hash: md5(concat(COMP_HK, '|', PERIOD_ID, '|', ATTR_ID, '|', ATTR_VALUE)) AS ATTR_HASH</fact>
  </canonical_facts>

  <instruction_set>
    ### 1. Surrogate Key Generation
    Always derive COMP_HK inline using native SQL before any JOIN or INSERT:
    ```sql
    md5(concat(VENDOR_ID, '|', COMP_ID)) AS COMP_HK
    -- [ORACLE_PORT]: LOWER(RAWTOHEX(DBMS_CRYPTO.HASH(UTL_RAW.CAST_TO_RAW(VENDOR_ID||'|'||COMP_ID), 2)))
    -- [HIVE_PORT]:   md5(concat(VENDOR_ID, '|', COMP_ID))
    ```

    ### 2. ATTR_HASH Change Detection
    Derive ATTR_HASH for every incoming source record:
    ```sql
    md5(concat(
        COALESCE(COMP_HK, 'PENDING_MAP'), '|',
        COALESCE(CAST(PERIOD_ID AS VARCHAR), ''), '|',
        COALESCE(ATTR_ID, ''), '|',
        COALESCE(ATTR_VALUE, '')
    )) AS ATTR_HASH
    ```
    - Use `COALESCE` guards to prevent hash instability from NULLs
    - ATTR_VALUE is VARCHAR — never cast before hashing

    ### 3. is_incremental() Block Structure
    ```sql
    {% if is_incremental() %}

        -- Step A: Identify records to expire (ATTR_HASH changed)
        UPDATE {{ this }}
        SET
            IS_CURRENT = FALSE,
            EFF_TO     = CURRENT_TIMESTAMP
        WHERE IS_CURRENT = TRUE
          AND COMP_HK IN (
              SELECT src.COMP_HK
              FROM {{ source_cte }} src
              JOIN {{ this }} tgt
                ON  tgt.COMP_HK   = src.COMP_HK
                AND tgt.PERIOD_ID  = src.PERIOD_ID
                AND tgt.ATTR_ID    = src.ATTR_ID
                AND tgt.IS_CURRENT = TRUE
                AND tgt.ATTR_HASH != src.ATTR_HASH  -- Changed record
          );

        -- Step B: Insert new open records for changed + net-new rows
        INSERT INTO {{ this }}
        SELECT
            src.*,
            CURRENT_TIMESTAMP AS EFF_FROM,
            NULL              AS EFF_TO,
            TRUE              AS IS_CURRENT
        FROM {{ source_cte }} src
        LEFT JOIN {{ this }} tgt
          ON  tgt.COMP_HK   = src.COMP_HK
          AND tgt.PERIOD_ID  = src.PERIOD_ID
          AND tgt.ATTR_ID    = src.ATTR_ID
          AND tgt.IS_CURRENT = TRUE
        WHERE tgt.COMP_HK IS NULL           -- Net-new: no current record exists
           OR tgt.ATTR_HASH != src.ATTR_HASH;  -- Changed: hash mismatch

    {% else %}
        -- Full refresh: all records are current at load time
        SELECT
            src.*,
            CURRENT_TIMESTAMP AS EFF_FROM,
            NULL              AS EFF_TO,
            TRUE              AS IS_CURRENT
        FROM {{ source_cte }} src
    {% endif %}
    ```

    ### 4. Idempotency Contract
    A full-feed re-run with identical source data MUST produce zero new rows.
    This is guaranteed by the ATTR_HASH equality check in Step B — if ATTR_HASH
    is identical, the `LEFT JOIN ... WHERE tgt.ATTR_HASH != src.ATTR_HASH` condition
    excludes the record from re-insertion.

    ### 5. Unchanged Record Pass-Through
    Records where ATTR_HASH is identical to the existing IS_CURRENT = TRUE record
    require NO action: neither UPDATE nor INSERT. The `is_incremental()` block above
    correctly skips them by construction.

    ### 6. State Transition Summary
    | Incoming Record | Action |
    |-----------------|--------|
    | New (no match in target) | INSERT: EFF_FROM=now(), EFF_TO=NULL, IS_CURRENT=TRUE |
    | Changed (ATTR_HASH mismatch) | UPDATE old: IS_CURRENT=FALSE, EFF_TO=now(). INSERT new: EFF_FROM=now(), EFF_TO=NULL, IS_CURRENT=TRUE |
    | Unchanged (ATTR_HASH match) | No action — idempotent |
    | Deleted (absent from source) | Handled by hash-and-delete-handler.md anti-join pattern |
  </instruction_set>

  <portability_notes>
    - DuckDB: `CURRENT_TIMESTAMP` — returns TIMESTAMP
    - Oracle: `SYSDATE` or `SYSTIMESTAMP`; IS_CURRENT = 0/1 (NUMBER(1,0))
    - Hive: `CURRENT_TIMESTAMP`; IS_CURRENT = 0/1 (TINYINT)
    - Add `-- [ORACLE_PORT]` and `-- [HIVE_PORT]` comments for each dialect difference
  </portability_notes>

</skill_definition>
