# Second Brain — DDL Contracts & Schema Authority

> **Version**: 1.0.0 | **Created**: 2026-04-11 | **Source Authorities**: `skills/dbt/bitemporal-ddl-generator.md` · `skills/dbt/gold-view-designer.md`
> **COLUMN-DRIFT GUARD**: This file is the single source of truth for all column names and types.
> Any occurrence of `VALID_FROM_DT`, `VALID_TO_DT`, or `IS_CURRENT = 1` in any agent, skill, or instruction file is a **defect**.

---

## § SCD2 Column Name Authority

> **Agents: Load this section during `<perceive>` BEFORE generating any SQL or DDL.**

| Column       | Canonical Name  | Type (DuckDB)  | Type (Oracle)    | Type (Hive)  | Notes                                        |
|--------------|-----------------|----------------|------------------|--------------|----------------------------------------------|
| SCD2 start   | `EFF_FROM`      | `TIMESTAMP`    | `DATE`           | `TIMESTAMP`  | Inclusive start of validity period           |
| SCD2 end     | `EFF_TO`        | `TIMESTAMP`    | `DATE`           | `TIMESTAMP`  | Exclusive end of validity; NULL = open record|
| Currency flag| `IS_CURRENT`    | `BOOLEAN`      | `NUMBER(1,0)`    | `TINYINT`    | Physical column. Default FALSE on insert     |

**Non-negotiable filter pattern**:
```sql
WHERE IS_CURRENT = TRUE            -- DuckDB / standard SQL
-- Oracle: WHERE IS_CURRENT = 1
-- Hive:   WHERE IS_CURRENT = 1
```

> **FORBIDDEN patterns** (cause runtime failure):
> - `WHERE IS_CURRENT = 1` in DuckDB context
> - `WHERE EFF_TO IS NULL` as primary current-record filter (contradicts DDL contract)
> - Column names `VALID_FROM_DT`, `VALID_TO_DT`, `dbt_valid_from`, `dbt_valid_to`

---

## § EAV Column Contract

| Column       | Canonical Name    | Type               | Notes                                                         |
|--------------|-------------------|--------------------|---------------------------------------------------------------|
| EAV value    | `ATTR_VALUE`      | `VARCHAR(4000)`    | Single unified storage column. NEVER split into typed columns |
| EAV metadata | `DATA_TYPE`       | `VARCHAR(64)`      | Lookup in `attr_metadata_dim`; drives downstream CAST logic   |

**Casting pattern** (from `gold-view-designer.md`):
```sql
CASE attr.DATA_TYPE
    WHEN 'DECIMAL'  THEN TRY_CAST(f.ATTR_VALUE AS DECIMAL(18,2))
    WHEN 'INTEGER'  THEN TRY_CAST(f.ATTR_VALUE AS INTEGER)
    WHEN 'DATE'     THEN TRY_CAST(f.ATTR_VALUE AS DATE)
    ELSE f.ATTR_VALUE  -- VARCHAR: no cast needed
END AS ATTR_VALUE_TYPED
-- [ORACLE_PORT]: Use TO_NUMBER / TO_DATE with error handling
-- [HIVE_PORT]:   Use CAST without TRY_CAST; wrap in CASE + REGEXP
```

---

## § Hash Key Conventions

| Key         | Derivation Formula                             | Storage Type   | Notes                          |
|-------------|------------------------------------------------|----------------|--------------------------------|
| `COMP_HK`   | `md5(concat(VENDOR_ID, '\|', COMP_ID))`        | `VARCHAR(64)`  | Use native SQL — NO dbt-utils  |
| `ATTR_HASH` | `md5(concat(COMP_HK, '\|', PERIOD_ID, '\|', ATTR_ID, '\|', ATTR_VALUE))` | `VARCHAR(64)` | Change detection hash |

> **FORBIDDEN**: `{{ dbt_utils.generate_surrogate_key([...]) }}` — violates zero-dependency rule.

---

## § Audit Columns (Mandatory on All Fact/Dim Tables)

| Column             | Type           | Purpose                                          |
|--------------------|----------------|--------------------------------------------------|
| `INGESTION_ID`     | `VARCHAR(64)`  | UUID or sequence ID for the load batch           |
| `REPORT_DATE`      | `DATE`         | Business date of the source S&P file             |
| `SOURCE_TIMESTAMP` | `TIMESTAMP`    | Wall-clock time the source record was received   |
| `SOURCE_TRACE_ID`  | `VARCHAR(64)`  | End-to-end lineage reference                     |

---

## § 5-Table Schema Overview

> Generation order is dependency-safe. Always create in this sequence.

| Order | Table                       | Type      | Key Relationships                              |
|-------|-----------------------------|-----------|------------------------------------------------|
| 1     | `vendor_dim`                | Dimension | Root anchor — no FK dependencies              |
| 2     | `attr_metadata_dim`         | Dimension | Independent; defines ATTR_ID + DATA_TYPE       |
| 3     | `period_dim`                | Dimension | Independent; defines PERIOD_ID + fiscal axes   |
| 4     | `company_dim`               | Dimension | References `vendor_dim`; OBLIGOR_ID nullable   |
| 5     | `financial_observations_fact` | Fact (EAV SCD2) | References all 4 dims; owns EFF_FROM/EFF_TO/IS_CURRENT |

---

## § Naming Rules (Non-Negotiable)

- **Case**: `UPPER_SNAKE_CASE` for all column names — Oracle/Hive compatibility requirement
- **No dbt_* prefixes**: `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from` are **VIOLATIONS**
- **VARCHAR lengths**: Always annotate length — `VARCHAR(N)` in DuckDB, `VARCHAR2(N)` in Oracle, `STRING` in Hive
- **Surrogate keys**: `VARCHAR(64)` for hash-based PKs; `BIGINT` for sequence-based PKs

---

## § OBLIGOR_ID Bridge Contract

- `OBLIGOR_ID` is **nullable** in `company_dim` — the column exists but mapping may be absent
- The `PENDING_MAP` fallback is applied in the **dbt model layer**, NOT in DDL:
  ```sql
  COALESCE(c.OBLIGOR_ID, 'PENDING_MAP') AS OBLIGOR_ID
  ```
- `company_dim.OBLIGOR_ID` must never have a NOT NULL constraint

---

## § Cross-Engine Portability Annotations

For every DuckDB-specific construct, include inline comments:
```sql
BOOLEAN    -- Oracle: NUMBER(1,0) | Hive: TINYINT
TIMESTAMP  -- Oracle: DATE        | Hive: TIMESTAMP
TRY_CAST   -- Oracle: TO_NUMBER() with exception | Hive: CAST (no TRY variant)
```

Use `-- [ORACLE_PORT]` and `-- [HIVE_PORT]` comment blocks immediately below primary DDL for alternate dialect versions.

---

## § Gold View Contract (`vw_financial_latest_gold`)

View grain: **(COMP_ID, PERIOD_ID, ATTR_ID)** — one row per attribute per company per period after priority resolution.

| Rule                     | Pattern                                              |
|--------------------------|------------------------------------------------------|
| Current record filter    | `WHERE f.IS_CURRENT = TRUE` (physical column)        |
| PENDING_MAP exclusion    | `AND f.OBLIGOR_ID != 'PENDING_MAP'`                  |
| Multi-source conflict    | `ROW_NUMBER() OVER (PARTITION BY COMP_ID, PERIOD_ID, ATTR_ID ORDER BY PRIORITY_RANK ASC)` |
| CTE chain order          | `current_facts` → `priority_resolved` → `typed_values` → `final_gold` |
