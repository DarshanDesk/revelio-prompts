---
applyTo: "**/*.sql, **/models/**, **/seeds/**, **/tests/**"
---

# dbt / SQL File Instructions — Financial Risk dbt/DuckDB PoC

> **Second Brain**: `.github/context-cache/SCHEMA.md` is the **column name authority** for all SQL and DDL generated in matched paths.
> Load `SCHEMA.md § SCD2 Column Name Authority` before writing any column reference, filter predicate, or DDL statement.

---

## 1. SCD2 Column Name Contract (Non-Negotiable)

Every SQL file in `models/`, `tests/`, or `seeds/` must use the canonical column names from `SCHEMA.md`.

| Role                | Canonical Column | Type (DuckDB) | Forbidden Alternatives                        |
|---------------------|------------------|---------------|-----------------------------------------------|
| SCD2 validity start | `EFF_FROM`       | `TIMESTAMP`   | `VALID_FROM_DT`, `dbt_valid_from`, `start_dt` |
| SCD2 validity end   | `EFF_TO`         | `TIMESTAMP`   | `VALID_TO_DT`, `dbt_valid_to`, `end_dt`       |
| Currency flag       | `IS_CURRENT`     | `BOOLEAN`     | `is_active`, `current_flag` (INTEGER)         |

**Mandatory filter pattern** in all `WHERE` clauses and Gold view CTEs:
```sql
WHERE IS_CURRENT = TRUE     -- DuckDB
-- [ORACLE_PORT]: WHERE IS_CURRENT = 1
-- [HIVE_PORT]:  WHERE IS_CURRENT = 1
```

> **FORBIDDEN** — these patterns are runtime defects in DuckDB:
> - `WHERE IS_CURRENT = 1` — wrong type; DuckDB BOOLEAN requires `= TRUE`
> - Column aliases `VALID_FROM_DT` or `VALID_TO_DT` — do not alias to forbidden names
> - `WHERE EFF_TO IS NULL` as the **sole** current-record filter (use `IS_CURRENT = TRUE` as primary predicate)

---

## 2. Hash Key Rules (No dbt-utils)

```sql
-- CORRECT: native DuckDB SQL
md5(concat(VENDOR_ID, '|', COMP_ID))                                         AS COMP_HK
md5(concat(COMP_HK, '|', PERIOD_ID, '|', ATTR_ID, '|', ATTR_VALUE))         AS ATTR_HASH

-- FORBIDDEN: any dbt-utils macro in this repository
-- {{ dbt_utils.generate_surrogate_key([...]) }}   ← VIOLATION — zero dependency rule
```

Hash key storage type: `VARCHAR(64)` on all tables.

---

## 3. EAV Column Contract

```sql
-- ATTR_VALUE is always VARCHAR(4000) — never split into typed columns
ATTR_VALUE  VARCHAR(4000)   -- raw storage; cast in Gold view via DATA_TYPE lookup

-- Casting pattern (Gold view only):
TRY_CAST(f.ATTR_VALUE AS DECIMAL(18,2))   -- DECIMAL
TRY_CAST(f.ATTR_VALUE AS INTEGER)          -- INTEGER
TRY_CAST(f.ATTR_VALUE AS DATE)             -- DATE
-- [ORACLE_PORT]: Use TO_NUMBER() / TO_DATE() with error handling
-- [HIVE_PORT]:  Use CAST() without TRY_CAST; add REGEXP guard
```

---

## 4. PENDING_MAP Sentinel (Zero Data Loss)

```sql
-- ALWAYS apply COALESCE — NEVER drop unmapped records
COALESCE(c.OBLIGOR_ID, 'PENDING_MAP')  AS OBLIGOR_ID

-- NEVER: INNER JOIN on obligor mapping (drops PENDING_MAP records)
-- ALWAYS: LEFT JOIN + COALESCE
```

Alert threshold: **5.0%** orphans per `(VENDOR_ID, REPORT_DATE)`. Source: `RULES.md § PENDING_MAP Threshold`.

---

## 5. Naming Conventions

| Rule                 | Pattern                  | Example                          |
|----------------------|--------------------------|----------------------------------|
| Column names         | `UPPER_SNAKE_CASE`       | `COMP_HK`, `ATTR_VALUE`          |
| No dbt_* prefixes    | Never prefix with `dbt_` | `dbt_scd_id` is a **VIOLATION**  |
| VARCHAR length       | Always annotate          | `VARCHAR(64)` not `VARCHAR`      |
| DuckDB surrogate PKs | `VARCHAR(64)` for hash   | `COMP_HK VARCHAR(64)`            |

---

## 6. Portability Annotations

For every DuckDB-specific construct, add inline portability comments:

```sql
IS_CURRENT  BOOLEAN      -- [ORACLE_PORT]: NUMBER(1,0)  | [HIVE_PORT]: TINYINT
EFF_FROM    TIMESTAMP    -- [ORACLE_PORT]: DATE          | [HIVE_PORT]: TIMESTAMP
TRY_CAST(x AS DECIMAL)   -- [ORACLE_PORT]: TO_NUMBER()  | [HIVE_PORT]: CAST (no TRY variant)
regexp_matches(col, pat)  -- [ORACLE_PORT]: REGEXP_LIKE  | [HIVE_PORT]: col RLIKE pat
```

Use `-- [ORACLE_PORT]` block comments immediately below primary DuckDB DDL for alternate dialect.

---

## 7. SCD2 State Machine (Quick Reference)

| Transition            | SQL Pattern                                                                |
|-----------------------|----------------------------------------------------------------------------|
| New record            | `INSERT ... IS_CURRENT = TRUE, EFF_FROM = now(), EFF_TO = NULL`           |
| Changed (hash diff)   | `UPDATE SET IS_CURRENT = FALSE, EFF_TO = now()` then INSERT new open row  |
| Unchanged (hash same) | No action; full-feed reload must be idempotent                            |
| Hard delete           | `UPDATE SET IS_CURRENT = FALSE, EFF_TO = now()` — NO physical DELETE      |

Full state machine logic: `skills/dbt/scd2-incremental-engine.md`

---

## 8. Audit Column Mandate

Every model output must carry these columns — never drop or alias to NULL:

```sql
INGESTION_ID    VARCHAR(64)    -- NOT NULL
REPORT_DATE     DATE           -- NOT NULL
SOURCE_TRACE_ID VARCHAR(64)    -- NOT NULL
```

---

## 9. dbt Test File Standards

- Use `assert_` prefix for all custom test SQL files (e.g., `assert_is_current_unique_per_grain.sql`)
- Return rows = **test failure**; zero rows = **test pass**
- IS_CURRENT in `accepted_values` test must use `[true, false]` (boolean) not `[0, 1]`
- Reference skill files for test templates:
  - Data quality: `skills/dbt/audit-test-generator.md`
  - SCD2 integrity: `skills/dbt/scd2-integrity-validator.md`
  - PENDING_MAP gate: `skills/dbt/audit-exception-reporter.md`
