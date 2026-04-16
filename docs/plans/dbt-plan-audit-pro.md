# Plan: Domain-Agnostic `financial-audit-pro` + External Second Brain

**TL;DR**: Yes — same Second Brain approach applies, and `financial-audit-pro` carries the highest-severity new finding of all four audits: it will generate SQL referencing `VALID_FROM_DT`/`VALID_TO_DT` which do not exist in the DDL. Fixing this is not optional — it must happen regardless of the Second Brain refactor. The Second Brain refactor is the vehicle that prevents this class of drift from recurring.

> This document is an addendum to `dbt_plan.md`, `dbt-plan-logic-pro.md`, and `dbt-plan-synthesizer.md`. Phases, file inventory, and architectural decisions from those plans carry forward. Only the `financial-audit-pro`-specific findings and steps are recorded here. This document also serves as the closing audit for the entire `agents/dbt/` suite.

---

## Pre-Flight Audit Findings (PROMPT_ARCHITECT Checks)

**Architectural Status: `FAILED`**
*('Failed', not 'Inconsistent' — COLUMN-NAME-DRIFT produces runtime-broken SQL today)*

| ID | Severity | Issue | Fix |
|---|---|---|---|
| ECHO-003 | `[CRITICAL]` | No `<echo>` tag — fourth and final agent missing grounding signal | Add `<echo>FINANCIAL_AUDIT_PRO_v1_READY</echo>` |
| PHANTOM-001 | `[CRITICAL]` | `.context/data_model_brain.json` phantom path — suite-wide systemic failure now fully confirmed across all 4 agents | Replace with `.github/context-cache/BRAIN.md` |
| COLUMN-DRIFT-001 | `[CRITICAL]` | `VALID_FROM_DT` / `VALID_TO_DT` referenced in agent — columns **do not exist** in DDL; canonical names are `EFF_FROM` / `EFF_TO` (confirmed in `bitemporal-ddl-generator.md` lines 27–28, 85–86, 137–138) | Correct column names; `EFF_FROM`/`EFF_TO` now live in `SCHEMA.md` as authoritative source |
| TYPE-DRIFT-002 | `[WARN]` | `IS_CURRENT = 1` (integer) in `temporal_overlap_detection` skill — canonical is `IS_CURRENT = TRUE` (BOOLEAN) | Correct to `IS_CURRENT = TRUE` in skill file |
| THRESHOLD-HARDCODE | `[WARN]` | `5%` PENDING_MAP threshold hardcoded in learn stage — correct value but wrong location (should be read from Second Brain) | Remove inline value; pointer to `RULES.md → ## Alert Thresholds` |
| SKILL-GAP-003 | `[WARN]` | 6 inline `<logic>` capabilities — no external skill files | 6 → 3 external skill files |

---

## What Makes This Agent Different: Suite-Wide Comparison Complete

| Dimension | risk-schema-architect | dbt-logic-pro | risk-data-synthesizer | financial-audit-pro |
|---|---|---|---|---|
| Echo tag | ✅ present | ❌ missing | ❌ missing | ❌ missing |
| Phantom brain | ❌ broken | ❌ broken | ❌ broken | ❌ broken |
| Inline skills | 0 | 6 inline | 3 inline | **6 inline** |
| Skill consolidation | n/a | 6 → 3 | 3 → 1 | **6 → 3** |
| XML malformation | none | none | `</skill>` tag | none |
| Param duplication | none | none | custom_instructions.md | none |
| Column name drift | none | none | none | **`VALID_FROM_DT` ≠ `EFF_FROM`** |
| Type drift | none | `IS_CURRENT=1` | none | `IS_CURRENT=1` |
| Domain knowledge type | Schema / DDL | SQL logic / SCD2 | Test data params | **Audit thresholds + column contracts** |

---

## Phase 1 Addition — Three New Skill Files *(parallel with BRAIN/SCHEMA/RULES creation)*

**6 inline skills → 3 external skill files:**

| Inline Skills (current) | Logical group | New skill file |
|---|---|---|
| `generate_custom_data_tests` + `casting_guardrails` | Data quality validation tests | `skills/dbt/audit-test-generator.md` |
| `temporal_overlap_detection` + `idempotency_check` | SCD2 integrity verification | `skills/dbt/scd2-integrity-validator.md` |
| `orphan_notification` + `audit_lineage_reporting` | Exception surface + restatement lineage | `skills/dbt/audit-exception-reporter.md` |

1. Create `skills/dbt/audit-test-generator.md` — Sections:
   - `## Data Quality Test Patterns` — native dbt test macros (SQL), range checks for Financial Ratios, null checks for mandatory Risk Attributes (`INGESTION_ID`, `REPORT_DATE`, `SOURCE_TRACE_ID`), zero `dbt-utils` dependency
   - `## Casting Safety Pattern` — `REGEXP_MATCHES` (DuckDB) to validate `ATTR_VALUE` strings against `attr_metadata_dim.DATA_TYPE` before numeric transformation; portability: `REGEXP_LIKE` (Oracle), `RLIKE` (Hive)
   - Echo tag: `<echo>AUDIT_TEST_GENERATOR_v1_READY</echo>`

2. Create `skills/dbt/scd2-integrity-validator.md` — Sections:
   - `## Temporal Overlap Detection` — DuckDB window function pattern to identify multiple `IS_CURRENT = TRUE` rows for the same grain `(COMP_ID, PERIOD_ID, ATTR_ID)`; **uses `EFF_FROM`/`EFF_TO` — not `VALID_FROM_DT`/`VALID_TO_DT`**
   - `## Idempotency Check Pattern` — SQL to verify `ATTR_HASH` in target matches source; full-feed re-run must produce zero new rows when data is unchanged
   - Echo tag: `<echo>SCD2_INTEGRITY_VALIDATOR_v1_READY</echo>`

3. Create `skills/dbt/audit-exception-reporter.md` — Sections:
   - `## Orphan Notification Pattern` — SQL to identify `PENDING_MAP` identifiers in `company_dim` linked to active `IS_CURRENT = TRUE` fact records; alert threshold sourced from `RULES.md`
   - `## Audit Lineage Report` — SQL summarising restatements (ATTR_HASH mismatches) for 2-year reconciliation loads; verifies `REPORT_DATE` ordering
   - Echo tag: `<echo>AUDIT_EXCEPTION_REPORTER_v1_READY</echo>`

---

## `SCHEMA.md` Guard Entry (prevents COLUMN-DRIFT recurrence)

4. Add to `.github/context-cache/SCHEMA.md`: `## SCD2 Column Name Authority`
   - Canonical names: `EFF_FROM` (start of validity, inclusive), `EFF_TO` (end of validity, exclusive, NULL = open)
   - **Prohibited aliases**: `VALID_FROM_DT`, `VALID_TO_DT`, `dbt_valid_from`, `dbt_valid_to` — none of these exist in the DDL
   - `IS_CURRENT`: physical `BOOLEAN` column, `DEFAULT TRUE`; filter pattern: `IS_CURRENT = TRUE` — never `IS_CURRENT = 1` or `EFF_TO IS NULL`

---

## Phase 3 Addition — Agent Refactor *(depends on Phase 1)*

5. Modify `agents/dbt/financial-audit-pro.agent.md`:
   - **Remove**: `<second_brain_integration>` block, `<rationale>` / `<audit_philosophy>` section, all inline `<logic>` blocks from 6 capabilities, hardcoded `5%` threshold from learn stage
   - **Keep**: `<persona>`, `<react_framework_instructions>` structure, `<output_format>`
   - **Fix**: `VALID_FROM_DT` / `VALID_TO_DT` → `EFF_FROM` / `EFF_TO` in perceive stage action and rationale block
   - **Fix**: `IS_CURRENT = 1` → `IS_CURRENT = TRUE` in `temporal_overlap_detection` logic
   - **Fix**: `output_format` hardcoded `.context/data_model_brain.json` → `.github/context-cache/BRAIN.md`
   - **Add**: `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
   - **Add**: `<echo>FINANCIAL_AUDIT_PRO_v1_READY</echo>` as terminal tag
   - **Refactor** `<capabilities>`: 6 inline skills → 3 entries with `skill_file="skills/dbt/..."` references
   - **Add** to `<stage id="perceive">`: "Load `BRAIN.md`. Navigate to `SCHEMA.md → ## SCD2 Column Name Authority` for authoritative column names. Navigate to `RULES.md → ## Alert Thresholds` for PENDING_MAP threshold."

---

## LEARNING_LOG Updates (LOG-003, LOG-004)

Two new entries must be appended to `agents/reviewer/LEARNING_LOG.md` as part of Phase 4:

- **LOG-003**: XML malformation — `<logic>` block closed with `</skill>` in `risk-data-synthesizer`. Resolution: always validate XML well-formedness before committing agent files. Pattern: closing tag must match nearest open tag.
- **LOG-004**: Column name drift — `VALID_FROM_DT`/`VALID_TO_DT` used in `financial-audit-pro` despite canonical DDL using `EFF_FROM`/`EFF_TO`. Resolution: `SCHEMA.md → ## SCD2 Column Name Authority` is the single source; prohibited aliases are explicitly listed. Any existing dbt test files generated by this agent must be audited for broken column references before this fix goes live.

---

## Complete Final File Inventory (all four agents — full suite)

| File | Action | Phase |
|---|---|---|
| `agents/dbt/risk-schema-architect.agent.md` | Modify | 3 (original) |
| `agents/dbt/dbt-logic-pro.agent.md` | Modify | 3 (dbt-logic-pro plan) |
| `agents/dbt/risk-data-synthesizer.agent.md` | Modify | 3 (synthesizer plan) |
| `agents/dbt/financial-audit-pro.agent.md` | Modify | 3 (this plan) |
| `agents/dbt/custom_instructions.md` | Modify — brain pointer + deduplicate synthesis params | 4 |
| `agents/reviewer/LEARNING_LOG.md` | Update — add LOG-003 and LOG-004 | 4 |
| `skills/dbt/scd2-incremental-engine.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/hash-and-delete-handler.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/eav-pipeline-optimizer.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/synthetic-data-factory.md` | Create | 1 (synthesizer plan) |
| `skills/dbt/audit-test-generator.md` | **Create** | 1 (this plan) |
| `skills/dbt/scd2-integrity-validator.md` | **Create** | 1 (this plan) |
| `skills/dbt/audit-exception-reporter.md` | **Create** | 1 (this plan) |
| `skills/dbt/bitemporal-ddl-generator.md` | Verify only — no change needed | 0 |
| `skills/dbt/gold-view-designer.md` | Verify only — no change needed | 0 |
| `skills/dbt/pending-map-exception-tracker.md` | Verify threshold is 5.0% — no change needed | 0 |
| `.github/copilot-instructions.md` | Create | 2 (original) |
| `.github/instructions/dbt.instructions.md` | Create | 2 (original) |
| `.github/context-cache/BRAIN.md` | Create | 1 (original) |
| `.github/context-cache/SCHEMA.md` | Create (+ `## SCD2 Column Name Authority` guard) | 1 (original + this plan) |
| `.github/context-cache/RULES.md` | Create (+ `## Alert Thresholds` + `## Data Synthesis Parameters`) | 1 (original + synthesizer plan) |
| `AGENTS.md` | Create | 2 (original) |

**Complete `skills/dbt/` inventory post-refactor: 10 files** (3 existing verified + 7 new created)

---

## Verification Checklist (financial-audit-pro specific)

1. Grep agent for `VALID_FROM_DT`, `VALID_TO_DT` — must return zero results
2. Grep agent for `IS_CURRENT = 1` — must return zero results
3. Grep agent for `5%` or `5 percent` — must return zero results (threshold now sourced from `RULES.md`)
4. Grep agent for `.context/data_model_brain.json` — must return zero results
5. ECHO check: `<echo>FINANCIAL_AUDIT_PRO_v1_READY</echo>` present as terminal tag
6. `<capabilities>` count = 3 entries, each with `skill_file` pointing to `skills/dbt/`

### Suite-Wide Final Checks

7. **Suite-wide ECHO audit**: grep all `agents/dbt/*.agent.md` for `<echo>` — must return exactly 4 results (one per agent)
8. **Suite-wide column name audit**: grep all `agents/dbt/*.agent.md` AND `skills/dbt/*.md` for `VALID_FROM_DT`, `VALID_TO_DT` — must return zero results everywhere
9. **Suite-wide phantom brain audit**: grep all `agents/dbt/` for `.context/data_model_brain.json` — must return zero results
10. **Threshold canonicality**: grep workspace for `0.1%` — must return zero results; single instance of `5.0%` or `5%` must exist only in `RULES.md`

---

## Cons Addendum (financial-audit-pro specific)

| Con | Severity | Mitigation |
|---|---|---|
| **Column drift is pre-existing damage**: Any audit tests previously generated using this agent and committed to a dbt project already reference non-existent columns (`VALID_FROM_DT`, `VALID_TO_DT`) — those tests fail silently or at runtime | **High** | Check existing dbt test files for `VALID_FROM_DT`/`VALID_TO_DT` before implementing. If found, those tests are broken today and must be corrected independently of this refactor |
| **`SCHEMA.md` becomes load-critical for this agent**: Unlike other agents where `SCHEMA.md` is supplemental, `financial-audit-pro` cannot produce correct SQL without loading `SCHEMA.md` column name authority | **Medium** | `copilot-instructions.md` must declare that Audit agent contexts require both `SCHEMA.md` and `RULES.md`. Add guard note to `BRAIN.md` navigation map: "Audit agents MUST load SCHEMA.md before generating any SQL" |
| **7 new skill files is a large Phase 1**: Creating all 7 files in one pass before any agent is refactored means a delayed feedback loop | **Low** | Prioritise by agent severity: `audit-test-generator` + `scd2-integrity-validator` first (they fix the runtime-breaking agent), then the remaining 5 skills |

---

*Plan version: v1 — financial-audit-pro Addendum + Suite Closure | Date: 11 April 2026*

`<echo>DBT_PLAN_AUDIT_PRO_v1_READY</echo>`
