# dbt Unified Master Plan — All Agents + Second Brain

> **Date**: 11 April 2026
> **Scope**: All 4 agents in `agents/dbt/` + `agents/dbt/custom_instructions.md` + full Second Brain architecture
> **Based on**: `dbt_plan.md`, `dbt-plan-logic-pro.md`, `dbt-plan-synthesizer.md`, `dbt-plan-audit-pro.md`
> **New findings in this document**: INTER-AGENT-COMM-001, COLUMN-DRIFT escalation to suite-scope (custom_instructions.md §5)

```
<echo>DBT_UNIFIED_MASTER_PLAN_v1_READY</echo>
```

---

## 1. Executive Summary

Four agents have been audited using the PROMPT_ARCHITECT methodology. This document consolidates all findings, introduces the inter-agent communication model, and provides a single phased delivery plan (Phases 0–4) covering 22 action items across 22 files.

**Critical blocker**: `financial-audit-pro.agent.md` and `custom_instructions.md §5` both reference `VALID_FROM_DT` / `VALID_TO_DT` — columns that **do not exist** in the DDL. Canonical schema defines `EFF_FROM` / `EFF_TO`. This is a runtime-breaking suite-wide COLUMN-DRIFT. Phase 0 fixes this before any other work proceeds.

**Second blocker**: `.context/data_model_brain.json` is referenced in all 5 files (4 agents + orchestration hub) — this path has never been created. The ReACT `Learn` stage is a no-op suite-wide. The Second Brain at `.github/context-cache/` replaces it entirely.

---

## 2. Consolidated Audit Findings (16 Findings)

| ID | Severity | File | Finding | Impact |
|----|----------|------|---------|--------|
| PHANTOM-001 | CRITICAL | All 5 files | `.context/data_model_brain.json` — path never created; brain is phantom | ReACT Learn stage is a no-op suite-wide |
| COLUMN-DRIFT-001 | CRITICAL | `financial-audit-pro.agent.md` | `VALID_FROM_DT` / `VALID_TO_DT` in perceive stage and rationale — columns do not exist in DDL | Runtime failure on every audit query |
| COLUMN-DRIFT-002 | CRITICAL | `custom_instructions.md §5` | `VALID_FROM_DT` / `VALID_TO_DT` — same drift in orchestration hub | Suite-wide COLUMN-DRIFT confirmed |
| XML-MALFORM-001 | HIGH | `risk-data-synthesizer.agent.md` | `mock_realtime_updates` `<logic>` block closes with `</skill>` instead of `</logic>` | Agent XML is malformed; may break parsers |
| SKILL-GAP-001 | HIGH | `dbt-logic-pro.agent.md` | 6 inline `<logic>` blocks — no external skill files | Domain knowledge trapped in agent; no reuse |
| SKILL-GAP-002 | HIGH | `risk-data-synthesizer.agent.md` | 3 inline `<logic>` blocks — no external skill files | Domain knowledge trapped in agent; no reuse |
| SKILL-GAP-003 | HIGH | `financial-audit-pro.agent.md` | 6 inline `<logic>` blocks — no external skill files | Domain knowledge trapped in agent; no reuse |
| ECHO-001 | HIGH | `dbt-logic-pro.agent.md` | Missing `<echo>` tag | Multi-agent chain cannot be end-to-end verified |
| ECHO-002 | HIGH | `risk-data-synthesizer.agent.md` | Missing `<echo>` tag | Multi-agent chain cannot be end-to-end verified |
| ECHO-003 | HIGH | `financial-audit-pro.agent.md` | Missing `<echo>` tag | Multi-agent chain cannot be end-to-end verified |
| TYPE-DRIFT-001 | MEDIUM | `dbt-logic-pro.agent.md` | `IS_CURRENT=1` integer — DDL defines `IS_CURRENT BOOLEAN` | Wrong filter type; queries may fail |
| TYPE-DRIFT-002 | MEDIUM | `financial-audit-pro.agent.md` | `IS_CURRENT=1` integer — DDL defines `IS_CURRENT BOOLEAN` | Wrong filter type; queries may fail |
| DRIFT-001 | MEDIUM | `risk-schema-architect.agent.md` | `0.1%` threshold — canonical is `5.0%` (50× discrepancy) | Incorrect validation gate |
| DUPLICATION-001 | MEDIUM | `risk-data-synthesizer.agent.md` + `custom_instructions.md §4` | Synthesis parameters defined in both files (`500 records`, `≥5% PENDING_MAP`, `Day 0/Day N`) | Single source of truth violation |
| DOMAIN-INLINE-001 | LOW | `risk-schema-architect.agent.md` | `<second_brain_integration>` and `<design_philosophy>` inline in agent body | Agent is not a pure ReACT orchestrator |
| INTER-AGENT-COMM-001 | LOW | Suite-wide | No directed handoff protocol between agents; no prerequisite checks before invocation | Agents can be invoked out of order without warning |

---

## 3. Inter-Agent Communication Model

### 3.1 Directed 4-Step Pipeline

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    .github/context-cache/  (Second Brain)                    │
│                                                                              │
│   BRAIN.md (routing + pipeline map)                                          │
│   SCHEMA.md (DDL contracts: EFF_FROM/EFF_TO, IS_CURRENT BOOLEAN)            │
│   RULES.md (business rules: 5.0% PENDING_MAP, synthesis params)             │
└──────────┬────────────────────────┬──────────────────────────────────────────┘
           │ READ                   │ READ                  │ READ        │ READ
           ▼                        ▼                        ▼             ▼
┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐    ┌──────────────────────┐
│  Step 1         │    │  Step 2             │    │  Step 3         │    │  Step 4              │
│  risk-schema-   │───►│  risk-data-         │───►│  dbt-logic-pro  │───►│  financial-audit-pro │
│  architect      │    │  synthesizer        │    │                 │    │                      │
│                 │    │                     │    │                 │    │                      │
│ READ:           │    │ READ:               │    │ READ:           │    │ READ:                │
│   SCHEMA.md     │    │   BRAIN.md (routing)│    │   SCHEMA.md     │    │   SCHEMA.md          │
│   RULES.md      │    │   RULES.md §Synth   │    │   RULES.md      │    │   RULES.md           │
│                 │    │                     │    │                 │    │                      │
│ WRITE:          │    │ WRITE:              │    │ WRITE:          │    │ WRITE:               │
│   [BRAIN-UPDATE │    │   [BRAIN-UPDATE     │    │   [BRAIN-UPDATE │    │   [BRAIN-UPDATE      │
│    -PENDING:    │    │    -PENDING:        │    │    -PENDING:    │    │    -PENDING:         │
│    schema]      │    │    seed stats]      │    │    logic]       │    │    audit findings]   │
│                 │    │                     │    │                 │    │                      │
│ ECHO:           │    │ ECHO:               │    │ ECHO:           │    │ ECHO:                │
│ RISK_SCHEMA_    │    │ RISK_DATA_          │    │ DBT_LOGIC_PRO_  │    │ FINANCIAL_AUDIT_PRO_ │
│ ARCHITECT_v1_   │    │ SYNTHESIZER_v1_     │    │ v1_READY        │    │ v1_READY             │
│ READY           │    │ READY               │    │                 │    │                      │
└─────────────────┘    └─────────────────────┘    └─────────────────┘    └──────────────────────┘
```

### 3.2 Agent Brain READ/WRITE Profiles

| Agent | Brain Files Read | Brain Update Pattern | Echo Tag |
|-------|-----------------|---------------------|----------|
| `risk-schema-architect` | `SCHEMA.md`, `RULES.md` | `[BRAIN-UPDATE-PENDING: DDL: <change>]` | `RISK_SCHEMA_ARCHITECT_v1_READY` |
| `risk-data-synthesizer` | `BRAIN.md` (routing), `RULES.md §Data Synthesis` | `[BRAIN-UPDATE-PENDING: SEED_STATS: <value>]` | `RISK_DATA_SYNTHESIZER_v1_READY` |
| `dbt-logic-pro` | `SCHEMA.md` (EFF_FROM/EFF_TO), `RULES.md` (thresholds) | `[BRAIN-UPDATE-PENDING: LOGIC: <pattern>]` | `DBT_LOGIC_PRO_v1_READY` |
| `financial-audit-pro` | `SCHEMA.md` (column authority), `RULES.md` (5.0% threshold) | `[BRAIN-UPDATE-PENDING: AUDIT: <finding>]` | `FINANCIAL_AUDIT_PRO_v1_READY` |

### 3.3 Copilot Read-Only Session Protocol

GitHub Copilot cannot write to files during a session. Any agent that would normally update the brain must emit the `[BRAIN-UPDATE-PENDING]` marker in its response. The user applies these manually after the session.

**Protocol format**:
```
[BRAIN-UPDATE-PENDING: <BRAIN_FILE>: <SECTION>: <VALUE>]
```

**Examples**:
```
[BRAIN-UPDATE-PENDING: SCHEMA.md: SCD2_Columns: Added obligor_segment EFF_FROM 2026-04-11]
[BRAIN-UPDATE-PENDING: RULES.md: PENDING_MAP_THRESHOLD: Confirmed 5.0% for CREDIT_RISK domain]
[BRAIN-UPDATE-PENDING: BRAIN.md: PIPELINE_STATUS: Step 1 complete — RISK_SCHEMA_ARCHITECT_v1_READY]
```

### 3.4 Out-of-Order Invocation Guard (Prerequisite Check Table)

| Agent | May Be Invoked When | Prerequisite Echo Required |
|-------|--------------------|-----------------------------|
| `risk-schema-architect` | Always (first in chain) | None |
| `risk-data-synthesizer` | After Step 1 is confirmed | `RISK_SCHEMA_ARCHITECT_v1_READY` visible in session |
| `dbt-logic-pro` | After Step 2 is confirmed | `RISK_DATA_SYNTHESIZER_v1_READY` visible in session |
| `financial-audit-pro` | After Step 3 is confirmed | `DBT_LOGIC_PRO_v1_READY` visible in session (implies Steps 1–2 also complete) |

> **Enforcement**: Each agent's `<perceive>` stage must verify the prerequisite echo before proceeding. If the predecessor echo is absent, the agent must emit a blocking warning and halt.

---

## 4. Second Brain Architecture

### 4.1 File Structure

```
.github/
├── context-cache/
│   ├── BRAIN.md        (~80 lines)  — Pipeline routing index + navigation map + [BRAIN-UPDATE-PENDING] log
│   ├── SCHEMA.md       (~150 lines) — DDL contracts, canonical column names, type definitions
│   └── RULES.md        (~150 lines) — Business rules, thresholds, synthesis parameters
├── copilot-instructions.md          — Copilot entry point; routes to second brain
└── instructions/
    └── dbt.instructions.md          — Path-scoped: applyTo: "**/*.sql, **/models/**, **/seeds/**, **/tests/**"

AGENTS.md                            — Repo root; cross-tool (Copilot + Claude Code + Cursor)
```

### 4.2 SCHEMA.md Canonical Facts (Non-Negotiable)

| Field | Canonical Value | Source Authority |
|-------|----------------|-----------------|
| SCD2 start column | `EFF_FROM` | `skills/dbt/bitemporal-ddl-generator.md` lines 27–28 |
| SCD2 end column | `EFF_TO` | `skills/dbt/bitemporal-ddl-generator.md` lines 27–28 |
| Currency flag type | `IS_CURRENT BOOLEAN DEFAULT TRUE` | `skills/dbt/bitemporal-ddl-generator.md` |
| Currency flag filter | `IS_CURRENT = TRUE` | `skills/dbt/gold-view-designer.md` |
| EAV value column | `ATTR_VALUE VARCHAR(4000)` | `skills/dbt/bitemporal-ddl-generator.md` |

> **COLUMN-DRIFT guard**: `SCHEMA.md § SCD2 Column Name Authority` is the single source of truth. Any occurrence of `VALID_FROM_DT`, `VALID_TO_DT`, or `IS_CURRENT = 1` anywhere in the suite is a defect.

### 4.3 RULES.md Canonical Facts (Non-Negotiable)

| Rule | Canonical Value | Source Authority |
|------|----------------|-----------------|
| PENDING_MAP threshold | `5.0%` | `skills/dbt/pending-map-exception-tracker.md` |
| Zero Data Loss pattern | `COALESCE(obligor_id, 'PENDING_MAP')` | `skills/dbt/bitemporal-ddl-generator.md` |
| Hash function | `MD5(concat(...))` native — no dbt-utils | `skills/dbt/scd2-incremental-engine.md` (to be created) |
| Seed record count | `500 records` minimum | `RULES.md §Data Synthesis Parameters` (moved from agent) |
| Day 0 / Day N pattern | Seed structure: Day 0 = initial load, Day N = delta | `RULES.md §Data Synthesis Parameters` (moved from agent) |

### 4.4 Model-Agnostic Tooling Principle

No tool-specific syntax. The Second Brain files at `.github/` are native VS Code / GitHub mechanisms:
- **GitHub Copilot**: reads `.github/copilot-instructions.md` and `.github/instructions/*.instructions.md` automatically
- **Claude Code**: reads `AGENTS.md` at repo root automatically
- **Cursor**: reads `.cursor/rules/` (symlink or copy from `.github/instructions/`)

No `@imports`, no `CLAUDE.md`, no auto-memory keywords. Same markdown files, consumed by all three tools.

---

## 5. Per-Agent Change Summary

### 5.1 `risk-schema-architect.agent.md` — Addendum Plan

**Fixes required**:
1. Remove `<second_brain_integration>` block (phantom brain path + inline domain knowledge)
2. Remove `<design_philosophy>` block (inline domain — belongs in BRAIN.md)
3. Replace phantom `.context/data_model_brain.json` → `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
4. Fix `0.1%` threshold → `5.0%` (DRIFT-001)
5. Add `[BRAIN-UPDATE-PENDING]` annotation to Learn stage

**Echo tag**: Already present — `<echo>RISK_SCHEMA_ARCHITECT_v1_READY</echo>` ✅

**Skill files**: Already external (3 `skill_file` references) — no skill work needed ✅

**Ref**: `dbt_plan.md` / `dbt-plan-architect.md`

---

### 5.2 `dbt-logic-pro.agent.md` — Two-Layer Refactor

**Layer 1 — Strip domain knowledge**:
1. Remove phantom brain path; add `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
2. Add prerequisite echo check to `<perceive>`: halt if `RISK_DATA_SYNTHESIZER_v1_READY` not confirmed
3. Add `<echo>DBT_LOGIC_PRO_v1_READY</echo>` (ECHO-001)

**Layer 2 — Inline → skill files** (SKILL-GAP-001):

| Inline Block | New Skill File | Key Canonical Facts |
|-------------|----------------|---------------------|
| `is_incremental()` surrogate key + SCD2 open-record logic | `skills/dbt/scd2-incremental-engine.md` | `IS_CURRENT = TRUE` (BOOLEAN); `EFF_TO` = high date |
| Hash-and-delete MERGE pattern | `skills/dbt/hash-and-delete-handler.md` | `MD5(concat(...))` native — no `dbt-utils` |
| EAV pipeline processing | `skills/dbt/eav-pipeline-optimizer.md` | `ATTR_VALUE VARCHAR(4000)`; single column |

4. Replace all 6 `<logic>` blocks with `<capability skill_file="skills/dbt/[new-file].md" />`
5. Fix `IS_CURRENT=1` → `IS_CURRENT = TRUE` in all logic references (TYPE-DRIFT-001)

**Ref**: `dbt-plan-logic-pro.md`

---

### 5.3 `risk-data-synthesizer.agent.md` — Consolidate + Fix

**Fixes required**:
1. Fix XML malformation: `mock_realtime_updates` closing `</skill>` → `</logic>` (XML-MALFORM-001)
2. Remove phantom brain path; add `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
3. Add prerequisite echo check to `<perceive>`: halt if `RISK_SCHEMA_ARCHITECT_v1_READY` not confirmed
4. Add `<echo>RISK_DATA_SYNTHESIZER_v1_READY</echo>` (ECHO-002)

**Inline → skill files** (SKILL-GAP-002): 3 → 1 consolidation:

| Inline Blocks (3) | Consolidated Skill File |
|------------------|------------------------|
| `generate_realistic_records`, `generate_eav_attributes`, `mock_realtime_updates` | `skills/dbt/synthetic-data-factory.md` |

5. Replace all 3 `<logic>` blocks with `<capability skill_file="skills/dbt/synthetic-data-factory.md" />`
6. Remove synthesis parameters from agent body — canonical definition moves to `RULES.md § Data Synthesis Parameters` (DUPLICATION-001)

**Ref**: `dbt-plan-synthesizer.md`

---

### 5.4 `financial-audit-pro.agent.md` — Critical Column Fix + Refactor

**Priority fixes (CRITICAL — must go first)**:
1. Replace `VALID_FROM_DT` → `EFF_FROM` in perceive stage and rationale (COLUMN-DRIFT-001)
2. Replace `VALID_TO_DT` → `EFF_TO` in perceive stage and rationale (COLUMN-DRIFT-001)
3. Fix `IS_CURRENT = 1` → `IS_CURRENT = TRUE` (TYPE-DRIFT-002)
4. Add `<perceive>` mandate: load `SCHEMA.md § SCD2 Column Name Authority` before generating any test SQL

**Structural fixes**:
5. Remove phantom brain path; add `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
6. Add prerequisite echo check to `<perceive>`: halt if `DBT_LOGIC_PRO_v1_READY` not confirmed
7. Add `<echo>FINANCIAL_AUDIT_PRO_v1_READY</echo>` (ECHO-003)

**Inline → skill files** (SKILL-GAP-003): 6 → 3 consolidation:

| Inline Blocks | New Skill File | Key Canonical Facts |
|--------------|----------------|---------------------|
| `generate_schema_tests`, `generate_custom_data_tests` | `skills/dbt/audit-test-generator.md` | EFF_FROM/EFF_TO; IS_CURRENT BOOLEAN |
| `validate_scd2_integrity`, `check_temporal_overlap` | `skills/dbt/scd2-integrity-validator.md` | No overlapping EFF_FROM–EFF_TO windows per surrogate key |
| `generate_pending_map_alerts`, `generate_audit_report` | `skills/dbt/audit-exception-reporter.md` | 5.0% PENDING_MAP threshold (from RULES.md) |

8. Replace all 6 `<logic>` blocks with `<capability skill_file="skills/dbt/[new-file].md" />`

**Ref**: `dbt-plan-audit-pro.md`

---

### 5.5 `custom_instructions.md` — Suite Orchestration Hub Fix

**Three targeted changes**:

| Section | Current (Defect) | Fix |
|---------|-----------------|-----|
| §3 | `.context/data_model_brain.json` — phantom path | Replace with `.github/context-cache/BRAIN.md` (PHANTOM-001) |
| §4 | Synthesis parameters duplicated: `500 records`, `≥5%`, `Day 0/Day N` | Remove; replace with pointer to `RULES.md § Data Synthesis Parameters` (DUPLICATION-001) |
| §5 | `Manage VALID_FROM_DT, VALID_TO_DT, and IS_CURRENT manually` | Replace with `EFF_FROM`, `EFF_TO`; clarify `IS_CURRENT BOOLEAN = TRUE` (COLUMN-DRIFT-002) |

> §5 is the escalation point: COLUMN-DRIFT is confirmed **suite-wide** because `custom_instructions.md` is the orchestration hub consumed by all 4 agents.

---

## 6. Phased Delivery Plan (Phases 0–4, 22 Action Items)

### Phase 0 — Emergency Fixes (Risk-Free, Unblocks Suite)
> **Rationale**: These fixes are non-structural edits to existing files. They resolve the two CRITICAL blockers (COLUMN-DRIFT-SUITE, PHANTOM-001). No new files created. Safe to execute immediately.

| # | Action | File | Defect ID |
|---|--------|------|-----------|
| 1 | Fix `§3`: Replace `.context/data_model_brain.json` → `.github/context-cache/BRAIN.md` | `custom_instructions.md` | PHANTOM-001 |
| 2 | Fix `§4`: Remove duplicate synthesis parameters; add pointer to `RULES.md § Data Synthesis Parameters` | `custom_instructions.md` | DUPLICATION-001 |
| 3 | Fix `§5`: Replace `VALID_FROM_DT` / `VALID_TO_DT` → `EFF_FROM` / `EFF_TO`; clarify `IS_CURRENT BOOLEAN` | `custom_instructions.md` | COLUMN-DRIFT-002 |
| 4 | Fix perceive stage and rationale: Replace `VALID_FROM_DT` / `VALID_TO_DT` → `EFF_FROM` / `EFF_TO`; fix `IS_CURRENT=1` → `IS_CURRENT = TRUE` | `financial-audit-pro.agent.md` | COLUMN-DRIFT-001, TYPE-DRIFT-002 |

---

### Phase 1A — Create Second Brain (3 New Files)
> **Rationale**: The brain must exist before agents can reference it. Build before modifying agents.

| # | Action | File | Notes |
|---|--------|------|-------|
| 5 | Create `BRAIN.md`: 4-step pipeline routing map, agent READ profiles, `[BRAIN-UPDATE-PENDING]` log section, navigation index | `.github/context-cache/BRAIN.md` | ~80 lines |
| 6 | Create `SCHEMA.md`: DDL contracts, SCD2 Column Name Authority (`EFF_FROM`/`EFF_TO`), `IS_CURRENT BOOLEAN`, `ATTR_VALUE VARCHAR(4000)`, `COALESCE(obligor_id, 'PENDING_MAP')` | `.github/context-cache/SCHEMA.md` | ~150 lines |
| 7 | Create `RULES.md`: Business rules, `5.0%` PENDING_MAP threshold, `§ Data Synthesis Parameters` (500 records, Day 0/Day N), Zero Data Loss rules | `.github/context-cache/RULES.md` | ~150 lines |

---

### Phase 1B — Create 7 New Skill Files
> **Rationale**: Skill files must exist before agents reference them via `skill_file=`. Build before Phase 3 agent refactors.

| # | Action | File | Sourced From (Inline Logic In) |
|---|--------|------|-------------------------------|
| 8 | Create `scd2-incremental-engine.md`: `is_incremental()` SCD2 open-record logic, surrogate key upsert | `skills/dbt/scd2-incremental-engine.md` | `dbt-logic-pro.agent.md` (2 inline blocks) |
| 9 | Create `hash-and-delete-handler.md`: Hash-and-delete MERGE pattern, `MD5(concat(...))` native | `skills/dbt/hash-and-delete-handler.md` | `dbt-logic-pro.agent.md` (2 inline blocks) |
| 10 | Create `eav-pipeline-optimizer.md`: EAV processing pipeline, `ATTR_VALUE VARCHAR(4000)` single column | `skills/dbt/eav-pipeline-optimizer.md` | `dbt-logic-pro.agent.md` (2 inline blocks) |
| 11 | Create `synthetic-data-factory.md`: Realistic record generation, EAV attribute generation, delta update mocking | `skills/dbt/synthetic-data-factory.md` | `risk-data-synthesizer.agent.md` (3 inline blocks, consolidated) |
| 12 | Create `audit-test-generator.md`: Schema tests, custom data quality tests, EFF_FROM/EFF_TO column contract tests | `skills/dbt/audit-test-generator.md` | `financial-audit-pro.agent.md` (2 inline blocks) |
| 13 | Create `scd2-integrity-validator.md`: SCD2 integrity checks, temporal overlap detection, no-gap validation | `skills/dbt/scd2-integrity-validator.md` | `financial-audit-pro.agent.md` (2 inline blocks) |
| 14 | Create `audit-exception-reporter.md`: PENDING_MAP alerts (5.0% from RULES.md), audit report generation | `skills/dbt/audit-exception-reporter.md` | `financial-audit-pro.agent.md` (2 inline blocks) |

---

### Phase 2 — Create Tooling Layer (3 New Files)
> **Rationale**: The tooling layer makes the Second Brain discoverable by GitHub Copilot, Claude Code, and Cursor without tool-specific syntax.

| # | Action | File | Notes |
|---|--------|------|-------|
| 15 | Create `copilot-instructions.md`: Routes to second brain, defines 4-step pipeline, echo tag prerequisites | `.github/copilot-instructions.md` | Copilot auto-loads this file |
| 16 | Create `dbt.instructions.md`: Path-scoped rules, `applyTo: "**/*.sql, **/models/**, **/seeds/**, **/tests/**"`, SCD2 column contract enforcement | `.github/instructions/dbt.instructions.md` | Copilot applies to matching paths |
| 17 | Create `AGENTS.md`: Repo root, cross-tool (Copilot + Claude Code + Cursor), 4-step pipeline, echo prerequisites, `[BRAIN-UPDATE-PENDING]` protocol | `AGENTS.md` | agentsmd spec |

---

### Phase 3 — Refactor All 4 Agent Files (Atomic — Treat as Single PR)
> **Rationale**: All 4 agents must be updated together. A partial refactor leaves the pipeline inconsistent. Phase 1A (brain) and Phase 1B (skills) must complete before this phase.

| # | Action | File | Key Changes |
|---|--------|------|-------------|
| 18 | Refactor `risk-schema-architect`: Remove inline domain knowledge, fix brain reference, fix `0.1%` → `5.0%`, add `[BRAIN-UPDATE-PENDING]` to Learn | `agents/dbt/risk-schema-architect.agent.md` | DOMAIN-INLINE-001, DRIFT-001, PHANTOM-001 |
| 19 | Refactor `dbt-logic-pro`: Remove 6 inline logic blocks → 3 skill refs, add echo tag, fix IS_CURRENT TYPE, fix brain ref, add prerequisite check | `agents/dbt/dbt-logic-pro.agent.md` | ECHO-001, SKILL-GAP-001, TYPE-DRIFT-001, PHANTOM-001 |
| 20 | Refactor `risk-data-synthesizer`: Fix XML malform, remove 3 inline logic blocks → 1 skill ref, add echo tag, remove duplicate synthesis params, fix brain ref, add prerequisite check | `agents/dbt/risk-data-synthesizer.agent.md` | ECHO-002, SKILL-GAP-002, XML-MALFORM-001, DUPLICATION-001, PHANTOM-001 |
| 21 | Refactor `financial-audit-pro`: Remove 6 inline logic blocks → 3 skill refs, add echo tag, add SCHEMA.md load mandate to perceive, fix brain ref, add prerequisite check | `agents/dbt/financial-audit-pro.agent.md` | ECHO-003, SKILL-GAP-003, PHANTOM-001 (COLUMN-DRIFT done in Phase 0) |

---

### Phase 4 — Suite Closure
> **Rationale**: Update persistent memory with new anti-pattern entries. These must be added AFTER all fixes are implemented so that the log records confirmed resolutions.

| # | Action | File | Notes |
|---|--------|------|-------|
| 22 | Add `LOG-003` (XML-MALFORM pattern: `</skill>` inside logic blocks) and `LOG-004` (COLUMN-DRIFT: `VALID_FROM_DT`/`VALID_TO_DT` — always use `SCHEMA.md § SCD2 Column Name Authority`) | `agents/reviewer/LEARNING_LOG.md` | Next entries after LOG-001 and LOG-002 |

---

## 7. Complete 22-File Inventory

### Files to Create (13 New Files)

| File | Phase | Purpose |
|------|-------|---------|
| `.github/context-cache/BRAIN.md` | 1A | Pipeline routing index, navigation map, [BRAIN-UPDATE-PENDING] log |
| `.github/context-cache/SCHEMA.md` | 1A | DDL contracts, SCD2 column name authority |
| `.github/context-cache/RULES.md` | 1A | Business rules, thresholds, synthesis parameters |
| `.github/copilot-instructions.md` | 2 | Copilot entry point; brain routing |
| `.github/instructions/dbt.instructions.md` | 2 | Path-scoped Copilot rules for dbt files |
| `AGENTS.md` | 2 | Repo root; cross-tool (Copilot + Claude Code + Cursor) |
| `skills/dbt/scd2-incremental-engine.md` | 1B | SCD2 is_incremental() logic |
| `skills/dbt/hash-and-delete-handler.md` | 1B | Hash-and-delete MERGE pattern |
| `skills/dbt/eav-pipeline-optimizer.md` | 1B | EAV pipeline processing |
| `skills/dbt/synthetic-data-factory.md` | 1B | CSV seed generation (all 3 logic blocks consolidated) |
| `skills/dbt/audit-test-generator.md` | 1B | Schema + data quality test generation |
| `skills/dbt/scd2-integrity-validator.md` | 1B | SCD2 integrity + temporal overlap checks |
| `skills/dbt/audit-exception-reporter.md` | 1B | PENDING_MAP alerts + audit reporting |

### Files to Modify (6 Files)

| File | Phase | Changes |
|------|-------|---------|
| `agents/dbt/custom_instructions.md` | 0 | §3 brain pointer, §4 dedup, §5 column names |
| `agents/dbt/financial-audit-pro.agent.md` | 0 + 3 | Phase 0: column names; Phase 3: full refactor |
| `agents/dbt/risk-schema-architect.agent.md` | 3 | Remove inline domain, fix threshold, fix brain ref |
| `agents/dbt/dbt-logic-pro.agent.md` | 3 | Echo, skill refs, type fix, brain ref, prerequisite check |
| `agents/dbt/risk-data-synthesizer.agent.md` | 3 | Echo, skill ref, XML fix, dedup, brain ref, prerequisite check |
| `agents/reviewer/LEARNING_LOG.md` | 4 | Add LOG-003 + LOG-004 |

### Files Existing and Correct — No Changes (3 Files)

| File | Status | Notes |
|------|--------|-------|
| `skills/dbt/bitemporal-ddl-generator.md` | ✅ Canonical authority | Defines EFF_FROM/EFF_TO, IS_CURRENT BOOLEAN; do not modify |
| `skills/dbt/gold-view-designer.md` | ✅ Correct patterns | IS_CURRENT = TRUE; do not modify |
| `skills/dbt/pending-map-exception-tracker.md` | ✅ Canonical authority | 5.0% threshold; do not modify |

---

## 8. Suite-Wide Verification Checklist (22 Items)

After all phases complete, verify the following before closing the implementation:

```
[ ] 1.  `find . -name "*.md" | xargs grep -l "data_model_brain.json"` → 0 results (PHANTOM-001 resolved)
[ ] 2.  `.github/context-cache/BRAIN.md` exists and contains 4-step pipeline routing map
[ ] 3.  `.github/context-cache/SCHEMA.md` exists and defines EFF_FROM / EFF_TO as SCD2 Column Name Authority
[ ] 4.  `.github/context-cache/RULES.md` exists and defines 5.0% PENDING_MAP threshold and § Data Synthesis Parameters
[ ] 5.  `custom_instructions.md §3` points to `.github/context-cache/BRAIN.md` (not `.context/`)
[ ] 6.  `custom_instructions.md §4` removed; contains only pointer to `RULES.md § Data Synthesis Parameters`
[ ] 7.  `custom_instructions.md §5` uses EFF_FROM / EFF_TO and IS_CURRENT BOOLEAN (not IS_CURRENT integer)
[ ] 8.  `financial-audit-pro.agent.md`: `grep -c "VALID_FROM_DT\|VALID_TO_DT"` → 0
[ ] 9.  `risk-schema-architect.agent.md`: echo tag `RISK_SCHEMA_ARCHITECT_v1_READY` present
[ ] 10. `dbt-logic-pro.agent.md`: echo tag `DBT_LOGIC_PRO_v1_READY` present
[ ] 11. `risk-data-synthesizer.agent.md`: echo tag `RISK_DATA_SYNTHESIZER_v1_READY` present
[ ] 12. `financial-audit-pro.agent.md`: echo tag `FINANCIAL_AUDIT_PRO_v1_READY` present
[ ] 13. `risk-data-synthesizer.agent.md`: `grep -c "</skill>"` inside logic blocks → 0 (XML-MALFORM-001 resolved)
[ ] 14. All 4 agents: `grep -c "<logic>"` → 0 (all inline blocks replaced with skill_file refs)
[ ] 15. 7 new skill files created under `skills/dbt/` and each referenced by the correct agent via `skill_file=`
[ ] 16. `dbt-logic-pro.agent.md`: `grep -c "IS_CURRENT=1\|IS_CURRENT = 1"` → 0 (TYPE-DRIFT-001 resolved)
[ ] 17. `financial-audit-pro.agent.md`: `grep -c "IS_CURRENT=1\|IS_CURRENT = 1"` → 0 (TYPE-DRIFT-002 resolved)
[ ] 18. `AGENTS.md` exists at repo root with 4-step pipeline, echo prerequisites, and [BRAIN-UPDATE-PENDING] protocol
[ ] 19. `.github/copilot-instructions.md` exists and routes to `.github/context-cache/` second brain
[ ] 20. `.github/instructions/dbt.instructions.md` exists with `applyTo: "**/*.sql, **/models/**, **/seeds/**, **/tests/**"`
[ ] 21. `agents/reviewer/LEARNING_LOG.md` contains LOG-003 (XML-MALFORM) and LOG-004 (COLUMN-DRIFT / VALID_FROM_DT pattern)
[ ] 22. `[BRAIN-UPDATE-PENDING]` protocol is documented in `.github/context-cache/BRAIN.md` for Copilot read-only sessions
```

---

## 9. Architecture Consolidated Cons

| Con | Severity | Mitigation |
|-----|----------|------------|
| Copilot cannot write to files — `[BRAIN-UPDATE-PENDING]` is a workaround, not an automated solution | Medium | Protocol is explicit; user applied manually after session; documented in BRAIN.md |
| 3-file Second Brain requires discipline to keep canonical — risk of drift if edits happen outside the process | Medium | SCHEMA.md is the single source of truth; column names validated by verification checklist item 3 |
| 7 new skill files increase surface area and navigation overhead | Low | BRAIN.md navigation index provides routing; agents reference by `skill_file=` path (unambiguous) |
| Directed 4-step pipeline is single-threaded — no parallel agent invocation | Low | Financial risk data has dependency order by design; Steps 1→4 are logically sequential |
| Out-of-order invocation guard relies on user reading the echo in session — no hard enforcement | Low | AGENTS.md and each agent's `<perceive>` stage document the prerequisite; checklist verifies |
| Model-agnostic design means no tool-specific optimisations (e.g., no Claude Code's extended thinking) | Low | Accepted tradeoff for portability; `.github/` native mechanisms work across all three tools |

---

## 10. Reference Map to Individual Plans

| Plan File | Scope | Key Findings Introduced |
|-----------|-------|------------------------|
| `dbt_plan.md` / `dbt-plan-architect.md` | `risk-schema-architect.agent.md` | PHANTOM-001, DRIFT-001, DOMAIN-INLINE-001, Second Brain architecture |
| `dbt-plan-logic-pro.md` | `dbt-logic-pro.agent.md` | ECHO-001, SKILL-GAP-001 (6 blocks), TYPE-DRIFT-001 |
| `dbt-plan-synthesizer.md` | `risk-data-synthesizer.agent.md` | ECHO-002, SKILL-GAP-002 (3→1), XML-MALFORM-001, DUPLICATION-001 |
| `dbt-plan-audit-pro.md` | `financial-audit-pro.agent.md` | ECHO-003, SKILL-GAP-003 (6→3), COLUMN-DRIFT-001, TYPE-DRIFT-002 |
| **`dbt-master-plan.md` (this document)** | **All agents + orchestration hub** | **COLUMN-DRIFT-002 (suite-scope), INTER-AGENT-COMM-001, unified phased delivery** |

---

```
<echo>DBT_UNIFIED_MASTER_PLAN_v1_READY</echo>
```
