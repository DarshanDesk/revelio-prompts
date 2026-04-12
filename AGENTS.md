# AGENTS.md — Financial Risk dbt/DuckDB PoC

> **Cross-tool**: This file is consumed by **GitHub Copilot**, **Claude Code**, and **Cursor**.
> **Scope**: All agents, models, seeds, and test files in this repository.
> **Second Brain**: `.github/context-cache/` — read before any code generation, DDL, or test authoring.

---

## Quick-Start for New Sessions

1. Read `.github/context-cache/SCHEMA.md` — column name authority (EFF_FROM/EFF_TO/IS_CURRENT BOOLEAN)
2. Read `.github/context-cache/RULES.md` — thresholds (5.0% PENDING_MAP), synthesis parameters, hash rules
3. Read `.github/context-cache/BRAIN.md` — pipeline routing, echo tags, [BRAIN-UPDATE-PENDING] protocol
4. Identify the active step in the pipeline (§2 below) and invoke the appropriate agent

---

## 1. Repository Purpose

This repository contains a **4-agent ReACT pipeline** for validating a Bi-Temporal SCD Type 2 EAV data model
for financial risk data ingestion (S&P Global feed). Target runtime is **DuckDB** (PoC) with Oracle/Hive portability.

**5-Table Schema** (dependency order):
1. `vendor_dim` — Root anchor
2. `attr_metadata_dim` — Attribute metadata + DATA_TYPE
3. `period_dim` — Fiscal period definitions
4. `company_dim` — Company master; OBLIGOR_ID nullable (PENDING_MAP bridge)
5. `financial_observations_fact` — EAV SCD2 fact; owns EFF_FROM/EFF_TO/IS_CURRENT

---

## 2. Directed 4-Step Agent Pipeline

Agents **must** run in this order. Invoking out of order without the predecessor echo emits a `[BLOCKING-WARNING]`.

```
┌────────────────────────────────────────────────────────────────────┐
│            .github/context-cache/  (Second Brain)                  │
│   SCHEMA.md · RULES.md · BRAIN.md                                  │
└───┬─────────────────┬─────────────────┬─────────────────┬──────────┘
    │ READ            │ READ            │ READ            │ READ
    ▼                 ▼                 ▼                 ▼
┌──────────┐    ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐
│ Step 1   │───►│ Step 2       │─►│ Step 3       │─►│ Step 4          │
│ risk-    │    │ risk-data-   │  │ dbt-logic-   │  │ financial-      │
│ schema-  │    │ synthesizer  │  │ pro          │  │ audit-pro       │
│ architect│    │              │  │              │  │                 │
└──────────┘    └──────────────┘  └──────────────┘  └─────────────────┘
```

### Agent Responsibilities

| Step | Agent                   | Responsibility                                           | Config File                               |
|------|-------------------------|----------------------------------------------------------|-------------------------------------------|
| 1    | `risk-schema-architect` | DDL generation, dimensional modelling, schema design     | `agents/dbt/risk-schema-architect.agent.md` |
| 2    | `risk-data-synthesizer` | CSV seed generation, 6 seed files, Day 0/Day N structure | `agents/dbt/risk-data-synthesizer.agent.md` |
| 3    | `dbt-logic-pro`         | `is_incremental()` SCD2 logic, priority MERGE, dbt models | `agents/dbt/dbt-logic-pro.agent.md`       |
| 4    | `financial-audit-pro`   | Data quality tests, PENDING_MAP alerts, audit reporting  | `agents/dbt/financial-audit-pro.agent.md` |

**Orchestration hub**: `agents/dbt/custom_instructions.md` — loaded by all 4 agents.

---

## 3. Echo Tag Prerequisites (Out-of-Order Guard)

Every agent's `<perceive>` stage **must** verify the predecessor echo before proceeding.
If the required echo is absent in the session context, the agent must emit `[BLOCKING-WARNING]` and halt.

| Agent                   | Required Predecessor Echo          | Agent's Own Echo Tag                |
|-------------------------|------------------------------------|-------------------------------------|
| `risk-schema-architect` | None — always first in chain       | `RISK_SCHEMA_ARCHITECT_v1_READY`    |
| `risk-data-synthesizer` | `RISK_SCHEMA_ARCHITECT_v1_READY`   | `RISK_DATA_SYNTHESIZER_v1_READY`    |
| `dbt-logic-pro`         | `RISK_DATA_SYNTHESIZER_v1_READY`   | `DBT_LOGIC_PRO_v1_READY`            |
| `financial-audit-pro`   | `DBT_LOGIC_PRO_v1_READY`           | `FINANCIAL_AUDIT_PRO_v1_READY`      |

**Blocking warning format**:
```
[BLOCKING-WARNING]: Prerequisite echo <ECHO_TAG> not found in session context.
Cannot proceed with <AGENT_NAME> until Step <N> is confirmed.
Invoke <PREDECESSOR_AGENT> first and confirm: <ECHO_TAG>
```

---

## 4. Non-Negotiable Code Constraints

### SCD2 Column Contract

```sql
-- CANONICAL (DuckDB — single source of truth: SCHEMA.md § SCD2 Column Name Authority)
EFF_FROM    TIMESTAMP   -- Inclusive start; never NULL
EFF_TO      TIMESTAMP   -- Exclusive end; NULL = open/current record
IS_CURRENT  BOOLEAN     -- DEFAULT TRUE on new insert; FALSE on close
                        -- Filter: WHERE IS_CURRENT = TRUE

-- FORBIDDEN — these column names/patterns are defects:
-- VALID_FROM_DT   VALID_TO_DT   dbt_valid_from   dbt_valid_to
-- WHERE IS_CURRENT = 1   (type mismatch — DuckDB BOOLEAN requires TRUE/FALSE)
```

### Hash Function (Native SQL — No External Packages)

```sql
-- CORRECT
md5(concat(VENDOR_ID, '|', COMP_ID))                                      AS COMP_HK
md5(concat(COMP_HK, '|', PERIOD_ID, '|', ATTR_ID, '|', ATTR_VALUE))      AS ATTR_HASH

-- FORBIDDEN — zero-dependency rule
-- {{ dbt_utils.generate_surrogate_key([...]) }}
-- Any import from dbt-utils or other dbt packages
```

### Zero Data Loss

```sql
-- Unmapped obligors MUST load with PENDING_MAP sentinel — never filter or drop
COALESCE(c.OBLIGOR_ID, 'PENDING_MAP') AS OBLIGOR_ID
-- Use LEFT JOIN on obligor mapping; never INNER JOIN (INNER JOIN silently drops orphans)
```

### PENDING_MAP Alert Threshold

- Canonical: **5.0%** orphans per `(VENDOR_ID, REPORT_DATE)` — source: `RULES.md § PENDING_MAP Threshold`
- Any value other than `5.0` is DRIFT-001

---

## 5. Skill File Registry

Domain logic is externalised into skill files. Agents reference via `skill_file=` — never inline.

| Skill File                                     | Covers                                                    | Used By           |
|------------------------------------------------|-----------------------------------------------------------|-------------------|
| `skills/dbt/scd2-incremental-engine.md`        | `is_incremental()` SCD2 open-record logic, surrogate keys | `dbt-logic-pro`   |
| `skills/dbt/hash-and-delete-handler.md`        | Hard-delete anti-join, priority MERGE (ROW_NUMBER)        | `dbt-logic-pro`   |
| `skills/dbt/eav-pipeline-optimizer.md`         | OBLIGOR reconciliation, ATTR_VALUE pass-through           | `dbt-logic-pro`   |
| `skills/dbt/synthetic-data-factory.md`         | 6 CSV seed files, Day 0/Day N, PENDING_MAP injection      | `risk-data-synthesizer` |
| `skills/dbt/audit-test-generator.md`           | Null checks, ratio range, regex casting, idempotency      | `financial-audit-pro` |
| `skills/dbt/scd2-integrity-validator.md`       | IS_CURRENT uniqueness, temporal overlap, continuity       | `financial-audit-pro` |
| `skills/dbt/audit-exception-reporter.md`       | PENDING_MAP orphan view, 5.0% gate, restatement lineage   | `financial-audit-pro` |
| `skills/dbt/bitemporal-ddl-generator.md`       | DDL authority — EFF_FROM/EFF_TO (do not modify)           | `risk-schema-architect` |
| `skills/dbt/gold-view-designer.md`             | Gold view IS_CURRENT = TRUE patterns (do not modify)      | `risk-schema-architect` |
| `skills/dbt/pending-map-exception-tracker.md`  | 5.0% threshold canonical source (do not modify)           | `financial-audit-pro` |

---

## 6. ReACT Framework (All Agents)

Every agent operates on the **Perceive → Reason → Act → Learn** cycle:

| Stage    | Action                                                                                   |
|----------|------------------------------------------------------------------------------------------|
| Perceive | Read Second Brain files; verify predecessor echo; load `SCHEMA.md § SCD2 Column Name Authority` |
| Reason   | Use `<think>` tags for chain-of-thought; explain design choices before generating code   |
| Act      | Generate code/DDL/CSV; enforce all constraints from §4; annotate DuckDB-specific syntax  |
| Learn    | Emit `[BRAIN-UPDATE-PENDING]` markers for any new facts discovered during the session    |

---

## 7. [BRAIN-UPDATE-PENDING] Protocol

AI tools **cannot** write to files during a session. Emit markers in the response; the user applies them manually.

**Format**:
```
[BRAIN-UPDATE-PENDING: <BRAIN_FILE>: <SECTION>: <VALUE>]
```

| Brain File   | When to Update                                        | Example Value                                |
|--------------|-------------------------------------------------------|----------------------------------------------|
| `BRAIN.md`   | Pipeline step completed; new routing fact discovered  | `PIPELINE_STATUS: Step 2 complete — RISK_DATA_SYNTHESIZER_v1_READY` |
| `SCHEMA.md`  | New column added or type confirmed                    | `SCD2_Columns: Added obligor_segment 2026-04-11` |
| `RULES.md`   | Threshold or parameter confirmed for a domain         | `PENDING_MAP_THRESHOLD: Confirmed 5.0% for CREDIT_RISK domain` |

---

## 8. Output Format (All Agents)

Every agent response must follow this structure:

```
## [<agent-name>]

<think>
... chain-of-thought reasoning ...
</think>

--- code blocks with DuckDB SQL ---
-- [ORACLE_PORT]: alternate dialect
-- [HIVE_PORT]:   alternate dialect

### Second Brain Update
[BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: ...]

<echo>AGENT_ECHO_TAG_v1_READY</echo>
```

---

## 9. Cross-Tool Compatibility Notes

| Tool            | How This Repository is Loaded                                              |
|-----------------|----------------------------------------------------------------------------|
| GitHub Copilot  | Auto-loads `.github/copilot-instructions.md` and `.github/instructions/*.instructions.md` |
| Claude Code     | Auto-reads `AGENTS.md` at repository root                                  |
| Cursor          | Consumes `.github/instructions/` or `.cursor/rules/` (symlink if needed)  |

No tool-specific syntax is used in any configuration file. All three tools read standard Markdown.
The Second Brain at `.github/context-cache/` is plain Markdown — readable by all tools without modification.
