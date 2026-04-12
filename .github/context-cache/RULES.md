# Second Brain — Business Rules, Thresholds & Synthesis Parameters

> **Version**: 1.0.0 | **Created**: 2026-04-11 | **Source Authorities**: `skills/dbt/pending-map-exception-tracker.md` · `skills/dbt/bitemporal-ddl-generator.md` · `agents/dbt/custom_instructions.md`
> **Consumed by**: All 4 agents. `risk-data-synthesizer` must read `§ Data Synthesis Parameters`. `financial-audit-pro` must read `§ PENDING_MAP Threshold`.

---

## § PENDING_MAP Threshold

> **Source authority**: `skills/dbt/pending-map-exception-tracker.md` — ALERT_THRESHOLD rule

| Parameter                              | Canonical Value | Scope                                  |
|----------------------------------------|-----------------|----------------------------------------|
| `exception_mgmt.pending_map_alert_threshold_pct` | `5.0`  | Per `(VENDOR_ID, REPORT_DATE)` combination |

**Alert logic** (from `pending-map-exception-tracker.md`):
```sql
orphan_count  = COUNT(*) WHERE OBLIGOR_ID = 'PENDING_MAP' AND IS_CURRENT = TRUE
total_count   = COUNT(*) WHERE IS_CURRENT = TRUE
orphan_pct    = ROUND(orphan_count * 100.0 / NULLIF(total_count, 0), 2)
alert_status  = CASE WHEN orphan_pct > 5.0 THEN '[WARNING]' ELSE '[OK]' END
```

> **DRIFT GUARD**: Any threshold value other than `5.0` in any agent or skill file is a defect (DRIFT-001).

---

## § Zero Data Loss Policy

> **Source authority**: `skills/dbt/bitemporal-ddl-generator.md` — OBLIGOR_ID Bridge + PENDING_MAP rules

- Records **must load** even when the internal OBLIGOR mapping is unavailable
- The `PENDING_MAP` sentinel value is applied in the **dbt model layer** via:
  ```sql
  COALESCE(c.OBLIGOR_ID, 'PENDING_MAP') AS OBLIGOR_ID
  ```
- **NEVER** generate a `DELETE` or `TRUNCATE` for PENDING_MAP records — they exist for remediation
- The `vw_unmapped_entities` view is the remediation surface; the gold view excludes them

---

## § Priority Ranking

| Rank | Source               | Description                                  |
|------|----------------------|----------------------------------------------|
| 1    | Analyst Override     | Highest priority — manual correction         |
| 2    | S&P Incremental Feed | Delta updates; supersede Full Feed           |
| 3    | S&P Full Feed        | Baseline load; lowest priority               |

**Conflict resolution**: When multiple sources have `IS_CURRENT = TRUE` for the same grain `(COMP_ID, PERIOD_ID, ATTR_ID)`, the record with the lowest `PRIORITY_RANK` integer wins. Resolved via `ROW_NUMBER()` in the gold view CTE layer.

---

## § Hash Function Authority

| Usage            | Function                                        | Prohibition                  |
|------------------|-------------------------------------------------|------------------------------|
| Surrogate keys   | `md5(concat(VENDOR_ID, '\|', COMP_ID))`         | No `dbt-utils` — FORBIDDEN   |
| Change detection | `md5(concat(COMP_HK, '\|', PERIOD_ID, '\|', ATTR_ID, '\|', ATTR_VALUE))` as `ATTR_HASH` | No `dbt-utils` — FORBIDDEN |

> **Enforcement**: `{{ dbt_utils.generate_surrogate_key([...]) }}` is a **VIOLATION** of the zero-dependency rule. Use native SQL `md5(concat(...))` in all DuckDB models.

---

## § Zero Dependency Rule

- `dbt-utils` package: **STRICTLY FORBIDDEN**
- Any other external dbt package: **STRICTLY FORBIDDEN**
- Use **native DuckDB SQL** for: hashing, regex, window functions, casting, date arithmetic
- Use **native Jinja** (no macros from packages) for: model conditionals, `is_incremental()` checks
- DuckDB-specific functions permitted (e.g., `regexp_matches`, `TRY_CAST`, `epoch_ms`)
- For Oracle/Hive portability, add inline `-- [ORACLE_PORT]` / `-- [HIVE_PORT]` comments

---

## § Data Synthesis Parameters

> **Moved here from `risk-data-synthesizer.agent.md` body** (DUPLICATION-001 resolution).
> `risk-data-synthesizer` reads this section during `<perceive>`. Do not redefine in any agent file.

| Parameter                     | Canonical Value            | Notes                                                  |
|-------------------------------|----------------------------|--------------------------------------------------------|
| Minimum seed record count     | `500 records`              | Upper bound: within DuckDB/Git size limits             |
| PENDING_MAP injection rate    | `≥ 5%` of records          | Must match PENDING_MAP alert threshold for test utility|
| Analyst Override records      | `≥ 2 records`              | Minimum to test priority logic                         |
| Seed versioning               | V1 (Day 0) + V2 (Day N)   | V1 = initial full load; V2 = delta for SCD2 time travel|
| ID consistency requirement    | Cross-file                 | Company, Period, Attribute IDs must match across all 4 S&P files and the mapping file |

**Day 0 / Day N seed structure**:
- **Day 0 (V1)**: Initial full load — establishes baseline `IS_CURRENT = TRUE` records with `EFF_FROM = load_timestamp`
- **Day N (V2)**: Delta/incremental load — triggers SCD2 close on changed records (`IS_CURRENT = FALSE`, `EFF_TO = delta_timestamp`) and opens new current records
- V1 and V2 together test the complete SCD2 "Time Travel" lifecycle

---

## § Audit Completeness Mandate

Every transformation output **must** preserve these columns from source to target:

| Column          | Type          | Enforcement               |
|-----------------|---------------|---------------------------|
| `INGESTION_ID`  | `VARCHAR(64)` | Must not be NULL or dropped|
| `REPORT_DATE`   | `DATE`        | Must not be NULL or dropped|
| `SOURCE_TRACE_ID` | `VARCHAR(64)` | Must not be NULL or dropped|

---

## § SCD2 State Machine Rules

| State Transition         | Action                                                          |
|--------------------------|-----------------------------------------------------------------|
| New record (no match)    | INSERT with `IS_CURRENT = TRUE`, `EFF_FROM = now()`, `EFF_TO = NULL` |
| Changed record (ATTR_HASH mismatch) | Close old: `IS_CURRENT = FALSE`, `EFF_TO = now()`. Open new: `IS_CURRENT = TRUE`, `EFF_FROM = now()`, `EFF_TO = NULL` |
| Unchanged record (ATTR_HASH match) | No action — idempotent full-feed reload produces zero new rows |
| Deleted record (not in source) | Close: `IS_CURRENT = FALSE`, `EFF_TO = now()` — NO physical DELETE |

---

## § PENDING_MAP Triage Grain

When surfacing orphans for remediation, the operational triage grain is:

```
(COMP_HK, VENDOR_ID, INGESTION_ID, REPORT_DATE)
```

This gives the operations team the exact batch and company identifier to locate and resolve the mapping gap in the Internal App DB.

---

## § DuckDB PoC Constraints

| Constraint               | Value / Pattern                              |
|--------------------------|----------------------------------------------|
| String type              | `VARCHAR(N)` — annotate length always        |
| Date/timestamp type      | `DATE` for business dates; `TIMESTAMP` for wall-clock |
| Regex function           | `regexp_matches(col, pattern)` — DuckDB native |
| Safe casting             | `TRY_CAST(col AS type)` — DuckDB native      |
| Target engine (PoC)      | DuckDB                                       |
| Portability target       | Oracle / Hive (annotate with inline comments)|
