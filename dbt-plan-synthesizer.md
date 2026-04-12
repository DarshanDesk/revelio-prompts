# Plan: Domain-Agnostic `risk-data-synthesizer` + External Second Brain

**TL;DR**: Yes — same Second Brain approach applies, and this agent's refactor is the **most contained** of the three (3 skills → 1 file, no type drift, cleaner ReACT structure). The unique finding here is an XML malformation and a duplication between the agent and `custom_instructions.md` that must be resolved before the Second Brain can be the single source of truth for synthesis parameters.

> This document is an addendum to `dbt_plan.md` and `dbt-plan-logic-pro.md`. Phases, file inventory, and architectural decisions from those plans carry forward. Only the `risk-data-synthesizer`-specific findings and steps are recorded here.

---

## Pre-Flight Audit Findings (PROMPT_ARCHITECT Checks)

**Architectural Status: `INCONSISTENT`**
*(Less severe than `dbt-logic-pro`'s `FAILED` — skill gap is smaller, but the XML malformation and duplication are new unique failure modes)*

| ID | Severity | Issue | Fix |
|---|---|---|---|
| ECHO-002 | `[CRITICAL]` | No `<echo>` tag — zero grounding signal; third agent in suite missing this | Add `<echo>RISK_DATA_SYNTHESIZER_v1_READY</echo>` |
| PHANTOM-001 | `[CRITICAL]` | Same `.context/data_model_brain.json` phantom path — systemic failure confirmed across all 3 agents | Replaced by `.github/context-cache/BRAIN.md` |
| XML-MALFORM-001 | `[WARN]` | `mock_realtime_updates`: `<logic>` block closes with `</skill>` instead of `</logic>` — malformed XML | Fix closing tag to `</logic>` in refactor |
| DUPLICATION-001 | `[WARN]` | `500 records`, `PENDING_MAP ≥5%`, `Day 0/Day N` pattern hardcoded in agent AND in `custom_instructions.md` | Both removed; single canonical definition in `RULES.md` |
| SKILL-GAP-002 | `[WARN]` | 3 inline `<logic>` blocks — no external skill files | 3 → 1 external skill file (all serve same purpose) |
| INLINE-DOMAIN-001 | `[WARN]` | Seed parameters, file names, date formats hardcoded in ReACT `reason` stage | Moved to `RULES.md` under `## Data Synthesis Parameters` |

---

## What Makes This Agent Different From the Previous Two

| Dimension | risk-schema-architect | dbt-logic-pro | risk-data-synthesizer |
|---|---|---|---|
| Echo tag | ✅ present | ❌ missing | ❌ missing |
| Phantom brain | ❌ broken ref | ❌ broken ref | ❌ broken ref |
| Inline skills | 0 (uses `skill_file`) | 6 inline | 3 inline |
| Skill consolidation | n/a | 6 → 3 files | **3 → 1 file** |
| XML malformation | none | none | **`</skill>` instead of `</logic>`** |
| Param duplication | none | none | **Duplicated in `custom_instructions.md`** |
| Domain knowledge type | Schema / DDL | SQL logic / SCD2 | **Test data parameters** |

The domain knowledge in this agent is qualitatively different — it is **test data generation parameters** (volumes, IDs, edge case percentages, file names), not schema specs or business logic. These live naturally in `RULES.md` under a dedicated `## Data Synthesis Parameters` section that does not yet exist.

---

## Phase 1 Addition — Single Skill File *(parallel with BRAIN/SCHEMA/RULES creation)*

**3 inline skills → 1 external skill file** (all three serve one cohesive purpose):

| Inline Skills (current) | Consolidated into | New skill file |
|---|---|---|
| `generate_long_format_seeds` + `mock_realtime_updates` + `reconciliation_scenario_design` | Full synthetic data factory — seed generation, update simulation, restatement scenarios | `skills/dbt/synthetic-data-factory.md` |

1. Create `skills/dbt/synthetic-data-factory.md` — Sections:
   - `## Long-Format Seed Structure` — cross-file grain (Company/Period/Attribute), referential integrity rules across all 6 CSV files, ATTR_VALUE as STRING constraint
   - `## Temporal Scenario Design` — Day 0 (Initial Load) vs Day N (Reconciliation) pattern, how to alter `ATTR_VALUE` to trigger `ATTR_HASH` change detection in `dbt-logic-pro`
   - `## Analyst Override Simulation` — higher priority timestamps, `Updated_By` signatures, Priority Rank 1 signal format
   - `## Edge Case Injections` — PENDING_MAP scenario (deliberately omit `OBLIGOR_ID`), Ratio restatement records, `'D'` flag (hard-delete) records for anti-join testing
   - Echo tag: `<echo>SYNTHETIC_DATA_FACTORY_v1_READY</echo>`

---

## `RULES.md` Addition — `## Data Synthesis Parameters`

This section does **not yet exist** in the planned `RULES.md`. It must be added to resolve DUPLICATION-001:

2. Add to `.github/context-cache/RULES.md`: `## Data Synthesis Parameters`
   - Seed volume: `~500 records` (git-friendly for DuckDB)
   - PENDING_MAP injection rate: `≥5%` of records (canonical — matches `custom_instructions.md` and resolves the `0.1%` vs `5.0%` historical drift)
   - Required edge case injections: ≥2 Analyst Override records, ≥1 Ratio restatement (V1 + V2), ≥1 hard-delete (`'D'` flag) record
   - Output file names: `vendor_seed.csv`, `period_seed.csv`, `data_seed.csv`, `ratio_seed.csv`, `analyst_updates.csv`, `obligor_mapping.csv`
   - Date format standard: `YYYY-MM-DD` (DuckDB DATE compatible)
   - ID consistency rule: `COMP_ID` values must match across all 6 files

---

## Phase 3 Addition — Agent Refactor *(depends on Phase 1)*

3. Modify `agents/dbt/risk-data-synthesizer.agent.md`:
   - **Remove**: `<second_brain_integration>` block, `<rationale>` / `<design_philosophy>` section, all inline `<logic>` blocks from 3 capabilities, hardcoded file names from `act` stage, hardcoded parameters from `reason` stage
   - **Keep**: `<persona>`, `<react_framework_instructions>` structure, `<output_format>`
   - **Fix**: `</skill>` closing tag on `mock_realtime_updates` capability → `</logic>`
   - **Fix**: `output_format` hardcoded `.context/data_model_brain.json` → `.github/context-cache/BRAIN.md`
   - **Add**: `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
   - **Add**: `<echo>RISK_DATA_SYNTHESIZER_v1_READY</echo>` as terminal tag
   - **Refactor** `<capabilities>`: 3 inline skills → 1 entry with `skill_file="skills/dbt/synthetic-data-factory.md"`
   - **Add** to `<stage id="perceive">`: "Load `BRAIN.md`. Navigate to `RULES.md` → `## Data Synthesis Parameters` for seed volumes, file names, and edge case injection rates."

---

## Phase 4 Addition — `custom_instructions.md` Deduplication *(depends on Phase 1)*

4. Update `agents/dbt/custom_instructions.md` — Remove the `## 4. Data Synthesis & Seed Requirements` section that duplicates synthesis parameters (lines 26–34). Replace with a pointer to `RULES.md → ## Data Synthesis Parameters`. This is in addition to the brain pointer update already planned in `dbt_plan.md`.

---

## Updated Full File Inventory (all three agents)

| File | Action | Phase |
|---|---|---|
| `agents/dbt/risk-schema-architect.agent.md` | Modify | 3 (original) |
| `agents/dbt/dbt-logic-pro.agent.md` | Modify | 3 (dbt-logic-pro plan) |
| `agents/dbt/risk-data-synthesizer.agent.md` | Modify | 3 (this plan) |
| `agents/dbt/custom_instructions.md` | Modify — brain pointer + deduplicate synthesis params | 4 |
| `skills/dbt/scd2-incremental-engine.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/hash-and-delete-handler.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/eav-pipeline-optimizer.md` | Create | 1 (dbt-logic-pro plan) |
| `skills/dbt/synthetic-data-factory.md` | **Create** | 1 (this plan) |
| `skills/dbt/pending-map-exception-tracker.md` | Verify threshold is 5.0% — no change needed | 0 |
| `.github/copilot-instructions.md` | Create | 2 (original) |
| `.github/instructions/dbt.instructions.md` | Create | 2 (original) |
| `.github/context-cache/BRAIN.md` | Create | 1 (original) |
| `.github/context-cache/SCHEMA.md` | Create | 1 (original) |
| `.github/context-cache/RULES.md` | Create (+ new `## Data Synthesis Parameters` section) | 1 (original + this plan) |
| `AGENTS.md` | Create | 2 (original) |

---

## Verification Checklist (risk-data-synthesizer specific)

1. Grep agent for `.context/data_model_brain.json` — must return zero results
2. Grep agent for `500 records`, `Day 0`, `Day N`, `vendor_seed.csv` — must return zero results (all moved to `RULES.md`)
3. XML validation: confirm `mock_realtime_updates` skill closes properly with `</logic>` then `</skill>`
4. ECHO check: `<echo>RISK_DATA_SYNTHESIZER_v1_READY</echo>` present as terminal tag
5. `<capabilities>` count = 1 entry with `skill_file="skills/dbt/synthetic-data-factory.md"`
6. Grep `custom_instructions.md` for `500 records`, `PENDING_MAP` percentage — must show only a pointer to `RULES.md`, not a duplicate definition

---

## Cons Addendum (risk-data-synthesizer specific)

| Con | Severity | Mitigation |
|---|---|---|
| **Duplication removal risk**: Deleting synthesis parameters from `custom_instructions.md` removes a safety net — if `RULES.md` is not loaded by an agent, it silently generates unconstrained seeds | **Medium** | `copilot-instructions.md` must explicitly reference `RULES.md` for the `risk-data-synthesizer` agent context. Add a guard note in `BRAIN.md` navigation map: "Synthesizer agents MUST load RULES.md" |
| **Single skill file reduces discoverability**: 3 → 1 means the skill file is longer. If it exceeds 150 lines, context efficiency degrades | **Low** | Cap at 150 lines by keeping each section to pattern + example only, no exhaustive column lists (those are in `SCHEMA.md`) |
| **XML malformation has been live**: The `</skill>` bug may have caused silent parsing inconsistencies in past sessions — historical outputs from this agent may have had incomplete capability activation | **Low** | Document in `LEARNING_LOG.md` as LOG-003 so it's tracked as a known past failure mode |

---

## Further Consideration

This is the third `agents/dbt/` agent audited. The only remaining one is `financial-audit-pro`. Based on the pattern confirmed across all three agents:
- All will have the phantom brain path
- All will be missing the echo tag
- `financial-audit-pro` likely has inline skills (consistent with `dbt-logic-pro` and `risk-data-synthesizer`)

A pre-implementation audit of `financial-audit-pro` before any files are created would complete the full inventory and ensure the `RULES.md` thresholds section captures all data quality metrics in one pass — particularly the audit thresholds that `financial-audit-pro` is responsible for enforcing.

---

*Plan version: v1 — risk-data-synthesizer Addendum | Date: 11 April 2026*

`<echo>DBT_PLAN_SYNTHESIZER_v1_READY</echo>`
