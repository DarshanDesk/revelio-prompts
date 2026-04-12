# GitHub Copilot — Financial Risk dbt/DuckDB PoC

> **Scope**: All agents, SQL models, dbt seeds, and test files in this repository.
> **Second Brain**: `.github/context-cache/` — read before generating any SQL, DDL, or test code.
> **Tool**: This file is auto-loaded by GitHub Copilot for every session scoped to this repository.

---

## 1. Second Brain Load Order

Before generating any SQL, DDL, or test code, Copilot must load and honour these three files:

| Priority | File                               | Purpose                                                |
|----------|------------------------------------|--------------------------------------------------------|
| 1        | `.github/context-cache/SCHEMA.md`  | **Column name authority** — EFF_FROM/EFF_TO, IS_CURRENT BOOLEAN, ATTR_VALUE |
| 2        | `.github/context-cache/RULES.md`   | **Business rules** — 5.0% PENDING_MAP threshold, synthesis parameters, hash function |
| 3        | `.github/context-cache/BRAIN.md`   | **Pipeline routing** — 4-step chain, echo tags, [BRAIN-UPDATE-PENDING] protocol |

> **COLUMN-DRIFT enforcement**: `SCHEMA.md § SCD2 Column Name Authority` is the single source of truth.
> Occurrences of `VALID_FROM_DT`, `VALID_TO_DT`, or `IS_CURRENT = 1` in any generated code are **defects**.

---

## 2. Directed 4-Step Agent Pipeline

This repository uses a **sequential 4-agent ReACT pipeline** for financial risk data modelling:

```
Step 1: risk-schema-architect  →  Step 2: risk-data-synthesizer
Step 3: dbt-logic-pro          →  Step 4: financial-audit-pro
```

Agent configuration files are in `agents/dbt/`. The orchestration hub is `agents/dbt/custom_instructions.md`.

### Agent Routing (use the correct agent for the task)

| Task Type                                       | Invoke Agent              |
|-------------------------------------------------|---------------------------|
| DDL, dimensional modelling, schema design        | `risk-schema-architect`   |
| CSV seed generation, synthetic test data         | `risk-data-synthesizer`   |
| `is_incremental()` logic, SCD2 state machine, MERGE pattern | `dbt-logic-pro` |
| Data quality tests, PENDING_MAP alerts, audit reporting | `financial-audit-pro` |

---

## 3. Echo Tag Prerequisites (Out-of-Order Guard)

Each agent must verify its predecessor's echo tag before proceeding.
If the predecessor echo is missing in the session, emit `[BLOCKING-WARNING]` and halt.

| Agent                   | May Proceed When                         | Required Echo Tag               |
|-------------------------|------------------------------------------|---------------------------------|
| `risk-schema-architect` | Always (first in chain)                  | None required                   |
| `risk-data-synthesizer` | After Step 1 output confirmed            | `RISK_SCHEMA_ARCHITECT_v1_READY`  |
| `dbt-logic-pro`         | After Step 2 output confirmed            | `RISK_DATA_SYNTHESIZER_v1_READY`  |
| `financial-audit-pro`   | After Step 3 output confirmed            | `DBT_LOGIC_PRO_v1_READY`          |

---

## 4. Non-Negotiable Code Constraints

### SCD2 Column Contract (SCHEMA.md authority)
```sql
-- CORRECT (DuckDB):
EFF_FROM    TIMESTAMP   -- SCD2 validity start (inclusive)
EFF_TO      TIMESTAMP   -- SCD2 validity end (exclusive; NULL = open record)
IS_CURRENT  BOOLEAN     -- Filter: WHERE IS_CURRENT = TRUE

-- FORBIDDEN (cause runtime failure):
-- VALID_FROM_DT, VALID_TO_DT, dbt_valid_from, dbt_valid_to
-- WHERE IS_CURRENT = 1  (wrong type for DuckDB BOOLEAN)
```

### Hash Function (RULES.md authority)
```sql
-- CORRECT: native SQL only
md5(concat(VENDOR_ID, '|', COMP_ID))                              AS COMP_HK
md5(concat(COMP_HK, '|', PERIOD_ID, '|', ATTR_ID, '|', ATTR_VALUE)) AS ATTR_HASH

-- FORBIDDEN: dbt-utils is STRICTLY FORBIDDEN in this repository
-- {{ dbt_utils.generate_surrogate_key([...]) }}  ← VIOLATION
```

### Zero Data Loss (RULES.md authority)
```sql
-- ALWAYS use PENDING_MAP sentinel — NEVER filter or drop unmapped records during load
COALESCE(c.OBLIGOR_ID, 'PENDING_MAP') AS OBLIGOR_ID
```

### PENDING_MAP Alert Threshold (RULES.md authority)
- Canonical threshold: **5.0%** per `(VENDOR_ID, REPORT_DATE)`
- Any value other than `5.0` in alert logic is DRIFT-001

---

## 5. Skill File Registry

All domain logic is externalised. Reference by `skill_file=` path — never inline domain code in agent files.

| Skill File                                          | Covers                                         |
|-----------------------------------------------------|------------------------------------------------|
| `skills/dbt/scd2-incremental-engine.md`             | `is_incremental()` SCD2 state machine, surrogate keys |
| `skills/dbt/hash-and-delete-handler.md`             | Hard-delete anti-join, priority MERGE via ROW_NUMBER |
| `skills/dbt/eav-pipeline-optimizer.md`              | OBLIGOR mapping reconciliation, ATTR_VALUE pass-through |
| `skills/dbt/synthetic-data-factory.md`              | 6 CSV seed files, Day 0/Day N, PENDING_MAP injection |
| `skills/dbt/audit-test-generator.md`                | Null checks, ratio range, regex casting, idempotency |
| `skills/dbt/scd2-integrity-validator.md`            | IS_CURRENT uniqueness, temporal overlap, continuity |
| `skills/dbt/audit-exception-reporter.md`            | PENDING_MAP orphan view, 5.0% gate, restatement lineage |
| `skills/dbt/bitemporal-ddl-generator.md`            | DDL authority — EFF_FROM/EFF_TO definitions (do not modify) |
| `skills/dbt/gold-view-designer.md`                  | Gold view patterns — IS_CURRENT = TRUE filters (do not modify) |
| `skills/dbt/pending-map-exception-tracker.md`       | 5.0% threshold authority (do not modify) |

---

## 6. ReACT Framework (All Agents)

Every agent response follows the **ReACT** loop:

1. **Perceive** — Read Second Brain; verify prerequisite echo; load `SCHEMA.md § SCD2 Column Name Authority`
2. **Reason** — Use `<think>` tags for chain-of-thought; explain design choices explicitly
3. **Act** — Generate code, DDL, or CSV; enforce all constraints from §4
4. **Learn** — Emit `[BRAIN-UPDATE-PENDING: <BRAIN_FILE>: <SECTION>: <VALUE>]` for any new facts discovered

---

## 7. [BRAIN-UPDATE-PENDING] Protocol

GitHub Copilot cannot write to files during a session. Emit markers; the user applies them manually.

**Format**:
```
[BRAIN-UPDATE-PENDING: <BRAIN_FILE>: <SECTION>: <VALUE>]
```

**Examples**:
```
[BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: Step 1 complete — RISK_SCHEMA_ARCHITECT_v1_READY]
[BRAIN-UPDATE-PENDING: SCHEMA.md: SCD2_Columns: Added obligor_segment EFF_FROM 2026-04-11]
[BRAIN-UPDATE-PENDING: RULES.md: PENDING_MAP_THRESHOLD: Confirmed 5.0% for CREDIT_RISK domain]
```

---

## 8. Output Format

All agent responses must include:
- `<think>` section for the chain-of-thought reasoning process
- Clear header identifying the active **Agent ID** (e.g., `## [risk-schema-architect]`)
- Code blocks with DuckDB-compatible SQL and `-- [ORACLE_PORT]` / `-- [HIVE_PORT]` portability annotations
- A **Second Brain Update** section for any `[BRAIN-UPDATE-PENDING]` markers
- Terminal `<echo>` tag confirming the agent's completion state (e.g., `<echo>RISK_SCHEMA_ARCHITECT_v1_READY</echo>`)
