# Second Brain — Pipeline Routing Index

> **Version**: 1.0.0 | **Created**: 2026-04-11 | **Scope**: Financial Risk dbt/DuckDB PoC
> **Consumed by**: `risk-schema-architect` · `risk-data-synthesizer` · `dbt-logic-pro` · `financial-audit-pro`
> **Update protocol**: Agents cannot write during a session. Emit `[BRAIN-UPDATE-PENDING]` markers; user applies manually.

---

## 1. Navigation Index — "I need X, read Y"

| I need to know…                              | Read this file                        | Section                          |
|----------------------------------------------|---------------------------------------|----------------------------------|
| Canonical SCD2 column names (EFF_FROM/EFF_TO) | `SCHEMA.md`                          | § SCD2 Column Name Authority     |
| IS_CURRENT type and filter pattern           | `SCHEMA.md`                          | § SCD2 Column Name Authority     |
| ATTR_VALUE column type                       | `SCHEMA.md`                          | § EAV Column Contract            |
| Hash key derivation pattern                  | `SCHEMA.md`                          | § Hash Key Conventions           |
| 5-table DDL overview                         | `SCHEMA.md`                          | § 5-Table Schema Overview        |
| PENDING_MAP alert threshold                  | `RULES.md`                           | § PENDING_MAP Threshold          |
| Data synthesis seed parameters               | `RULES.md`                           | § Data Synthesis Parameters      |
| Priority ranking (Analyst/S&P Inc/Full)      | `RULES.md`                           | § Priority Ranking               |
| Zero Data Loss policy                        | `RULES.md`                           | § Zero Data Loss Policy          |
| 4-step pipeline order                        | This file (`BRAIN.md`)               | § 2 below                        |
| Prerequisite echo tags per agent             | This file (`BRAIN.md`)               | § 3 below                        |
| [BRAIN-UPDATE-PENDING] log entries           | This file (`BRAIN.md`)               | § 5 below                        |

---

## 2. Directed 4-Step Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│              .github/context-cache/  (Second Brain)                 │
│   BRAIN.md · SCHEMA.md · RULES.md                                   │
└────┬────────────────────┬───────────────────┬────────────────────┬──┘
     │ READ               │ READ              │ READ               │ READ
     ▼                    ▼                   ▼                    ▼
┌──────────────┐  ┌───────────────────┐  ┌──────────────┐  ┌─────────────────────┐
│  Step 1      │  │  Step 2           │  │  Step 3      │  │  Step 4             │
│  risk-schema │─►│  risk-data-       │─►│  dbt-logic-  │─►│  financial-audit-   │
│  -architect  │  │  synthesizer      │  │  pro         │  │  pro                │
│              │  │                   │  │              │  │                     │
│ Prereq: none │  │ Prereq: Step 1 ✓  │  │ Prereq: Step │  │ Prereq: Step 3 ✓    │
│              │  │                   │  │ 2 ✓          │  │                     │
│ ECHO:        │  │ ECHO:             │  │ ECHO:        │  │ ECHO:               │
│ RISK_SCHEMA_ │  │ RISK_DATA_        │  │ DBT_LOGIC_   │  │ FINANCIAL_AUDIT_    │
│ ARCHITECT_   │  │ SYNTHESIZER_      │  │ PRO_v1_READY │  │ PRO_v1_READY        │
│ v1_READY     │  │ v1_READY          │  │              │  │                     │
└──────────────┘  └───────────────────┘  └──────────────┘  └─────────────────────┘
```

---

## 3. Agent READ/WRITE Profiles & Prerequisite Guards

| Step | Agent                  | Brain Files Read             | Brain Update Pattern                         | Echo Tag                        | Prerequisite Echo Required         |
|------|------------------------|------------------------------|----------------------------------------------|---------------------------------|------------------------------------|
| 1    | `risk-schema-architect`  | `SCHEMA.md`, `RULES.md`      | `[BRAIN-UPDATE-PENDING: DDL: <change>]`      | `RISK_SCHEMA_ARCHITECT_v1_READY`  | None — always first in chain       |
| 2    | `risk-data-synthesizer`  | `BRAIN.md`, `RULES.md §Synth`| `[BRAIN-UPDATE-PENDING: SEED_STATS: <value>]`| `RISK_DATA_SYNTHESIZER_v1_READY`  | `RISK_SCHEMA_ARCHITECT_v1_READY`   |
| 3    | `dbt-logic-pro`          | `SCHEMA.md`, `RULES.md`      | `[BRAIN-UPDATE-PENDING: LOGIC: <pattern>]`   | `DBT_LOGIC_PRO_v1_READY`          | `RISK_DATA_SYNTHESIZER_v1_READY`   |
| 4    | `financial-audit-pro`    | `SCHEMA.md`, `RULES.md`      | `[BRAIN-UPDATE-PENDING: AUDIT: <finding>]`   | `FINANCIAL_AUDIT_PRO_v1_READY`    | `DBT_LOGIC_PRO_v1_READY`           |

> **Enforcement**: Each agent's `<perceive>` stage verifies the prerequisite echo before proceeding.
> If the predecessor echo is absent in the session, the agent must emit a **[BLOCKING-WARNING]** and halt.

---

## 4. [BRAIN-UPDATE-PENDING] Protocol

GitHub Copilot cannot write to files during a session. Agents emit markers; the user applies them manually post-session.

**Format**:
```
[BRAIN-UPDATE-PENDING: <BRAIN_FILE>: <SECTION>: <VALUE>]
```

**Examples**:
```
[BRAIN-UPDATE-PENDING: SCHEMA.md: SCD2_Columns: Added obligor_segment EFF_FROM 2026-04-11]
[BRAIN-UPDATE-PENDING: RULES.md: PENDING_MAP_THRESHOLD: Confirmed 5.0% for CREDIT_RISK domain]
[BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: Step 1 complete — RISK_SCHEMA_ARCHITECT_v1_READY]
```

---

## 5. [BRAIN-UPDATE-PENDING] Log

*Entries below are applied by the user after each session. Newest first.*

| Date | Applied By | File | Section | Value |
|------|-----------|------|---------|-------|
| —    | —          | —    | —       | —     |

---

## 6. Pipeline Status Tracker

| Step | Agent                  | Status      | Echo Confirmed         | Date       |
|------|------------------------|-------------|------------------------|------------|
| 1    | `risk-schema-architect`  | Not started | —                      | —          |
| 2    | `risk-data-synthesizer`  | Blocked     | Awaiting Step 1        | —          |
| 3    | `dbt-logic-pro`          | Blocked     | Awaiting Step 2        | —          |
| 4    | `financial-audit-pro`    | Blocked     | Awaiting Step 3        | —          |

> Update this table via `[BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: <entry>]` after each agent run.
