# Plan: Domain-Agnostic `dbt-logic-pro` + External Second Brain

**TL;DR**: Yes — the same Second Brain approach applies, and is even MORE justified here. `dbt-logic-pro` fails two checks that `risk-schema-architect` passed: it has no echo tag, and it embeds all skill logic inline rather than referencing external files. The refactor requires **two layers** instead of one.

> This document is an addendum to `dbt_plan.md`. Phases, file inventory, and architectural decisions from that plan carry forward. Only the `dbt-logic-pro`-specific findings and steps are recorded here.

---

## Pre-Flight Audit Findings (PROMPT_ARCHITECT Checks)

**Architectural Status: `FAILED`** *(more severe than `risk-schema-architect` which was `INCONSISTENT`)*

| ID | Severity | Issue | Fix |
|---|---|---|---|
| ECHO-001 | `[CRITICAL]` | No `<echo>` tag — zero context-grounding signal | Add `<echo>DBT_LOGIC_PRO_v1_READY</echo>` |
| PHANTOM-001 | `[CRITICAL]` | Same `.context/data_model_brain.json` phantom path (repeat of risk-schema-architect) | Replaced by `.github/context-cache/BRAIN.md` pointer |
| SKILL-GAP-001 | `[CRITICAL]` | All 6 capabilities have inline `<logic>` blocks — no `skill_file` references, no external skill files exist | Create 3 skill files in `skills/dbt/`; consolidate 6 inline skills into 3 focused files |
| TYPE-DRIFT-001 | `[WARN]` | `IS_CURRENT=1` (integer) in learn stage vs `IS_CURRENT BOOLEAN` in DDL spec | Correct to `IS_CURRENT = TRUE` |
| CONTRADICTION-001 | `[WARN]` | Persona claims "heavily dependent on Second Brain" yet brain is phantom — strongest dependency, weakest foundation | Resolved by brain externalisation |

---

## Two-Layer Refactor (what makes this agent different)

`risk-schema-architect` needed **1 layer** of refactoring: strip domain knowledge, add brain pointer.

`dbt-logic-pro` needs **2 layers**:

**Layer 1 — Same as `risk-schema-architect`** (domain knowledge → Second Brain)
- Remove `<second_brain_integration>` block, `<coding_philosophy>` / `<rationale>`, inline domain rules
- Add `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
- Fix `IS_CURRENT=1` → `IS_CURRENT = TRUE`
- Fix `output_format` which still hardcodes `.context/data_model_brain.json` — replace with `.github/context-cache/BRAIN.md`
- Add echo tag

**Layer 2 — New, specific to `dbt-logic-pro`** (inline logic → external skill files)

The 6 inline capabilities consolidate into **3 external skill files**, following the same pattern as `risk-schema-architect`'s existing skills:

| Inline Skills (current) | Consolidated into | New skill file |
|---|---|---|
| `implement_priority_merge` + `manage_scd2_incremental` | SCD2 state machine + priority resolution | `skills/dbt/scd2-incremental-engine.md` |
| `native_hashing_macros` + `hard_delete_detection` | Hash logic + delete detection | `skills/dbt/hash-and-delete-handler.md` |
| `mapping_reconciliation` + `performance_tuning` | PENDING_MAP join + EAV optimisation | `skills/dbt/eav-pipeline-optimizer.md` |

After Layer 2, the agent's `<capabilities>` block mirrors `risk-schema-architect` — 3 entries, each with `skill_file="skills/dbt/..."`, no inline logic.

---

## Phase 1 Addition — Create Skill Files *(parallel with BRAIN/SCHEMA/RULES creation)*

1. Create `skills/dbt/scd2-incremental-engine.md` — Sections:
   - SCD2 manual merge pattern (MERGE/INSERT logic for EFF_FROM/EFF_TO/IS_CURRENT)
   - `is_incremental()` block structure for DuckDB/dbt
   - `ROW_NUMBER()` priority resolution (Rank 1=Analyst Override > Rank 2=S&P Incremental > Rank 3=S&P Full)
   - Echo tag: `<echo>SCD2_INCREMENTAL_ENGINE_v1_READY</echo>`

2. Create `skills/dbt/hash-and-delete-handler.md` — Sections:
   - Portable Jinja MD5/SHA256 macros (`md5(concat(col1, '|', col2))`) compatible with DuckDB, Oracle, and Hive
   - Anti-join pattern for Full Feed hard-delete detection (trigger `VALID_TO_DT` updates)
   - `ATTR_HASH` change detection logic (drives SCD2 insert/expire decision)
   - Echo tag: `<echo>HASH_AND_DELETE_HANDLER_v1_READY</echo>`

3. Create `skills/dbt/eav-pipeline-optimizer.md` — Sections:
   - PENDING_MAP `LEFT JOIN` / `COALESCE` reconciliation pattern
   - Zero Data Loss contract (`COALESCE(obligor_id, 'PENDING_MAP')`)
   - DuckDB clustering / sort keys for EAV at scale
   - Portability notes (Oracle distribution keys, Hive bucketing)
   - Echo tag: `<echo>EAV_PIPELINE_OPTIMIZER_v1_READY</echo>`

---

## Phase 3 Addition — Agent Refactor *(depends on Phase 1)*

4. Modify `agents/dbt/dbt-logic-pro.agent.md`:
   - **Remove**: `<second_brain_integration>` block, `<rationale>` / `<coding_philosophy>` section, all inline `<logic>` blocks from all 6 capabilities
   - **Keep**: `<persona>`, `<react_framework_instructions>` structure, `<output_format>`
   - **Fix**: `IS_CURRENT=1` → `IS_CURRENT = TRUE` in learn stage thinking block
   - **Fix**: `output_format` hardcoded `.context/data_model_brain.json` → `.github/context-cache/BRAIN.md`
   - **Add**: `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>`
   - **Add**: `<echo>DBT_LOGIC_PRO_v1_READY</echo>` as terminal tag
   - **Refactor** `<capabilities>` from 6 inline `<logic>` skills to 3 external `skill_file` references

---

## Updated Full File Inventory (both agents)

| File | Action | Phase |
|---|---|---|
| `agents/dbt/risk-schema-architect.agent.md` | Modify — strip domain, add `<second_brain_ref>` | 3 (original) |
| `agents/dbt/dbt-logic-pro.agent.md` | Modify — strip domain + inline logic, add echo tag + skill refs | 3 (addendum) |
| `agents/dbt/custom_instructions.md` | Modify — replace hardcoded brain block with pointer | 4 (original) |
| `skills/dbt/scd2-incremental-engine.md` | **Create** | 1 (addendum) |
| `skills/dbt/hash-and-delete-handler.md` | **Create** | 1 (addendum) |
| `skills/dbt/eav-pipeline-optimizer.md` | **Create** | 1 (addendum) |
| `skills/dbt/pending-map-exception-tracker.md` | Verify threshold is 5.0% — no change needed | 0 (original) |
| `.github/copilot-instructions.md` | Create | 2 (original) |
| `.github/instructions/dbt.instructions.md` | Create | 2 (original) |
| `.github/context-cache/BRAIN.md` | Create | 1 (original) |
| `.github/context-cache/SCHEMA.md` | Create | 1 (original) |
| `.github/context-cache/RULES.md` | Create | 1 (original) |
| `AGENTS.md` | Create | 2 (original) |

---

## Verification Checklist (dbt-logic-pro specific)

1. Grep agent file for `IS_CURRENT=1` — must return zero results post-refactor
2. Grep agent file for `.context/data_model_brain.json` — must return zero results
3. Grep agent file for `ROW_NUMBER`, `COALESCE`, `MERGE`, `MD5`, `anti-join` — none should remain (all moved to skill files)
4. ECHO check: `<echo>DBT_LOGIC_PRO_v1_READY</echo>` must be present as terminal tag
5. Count `<capabilities>` entries — must equal 3 (down from 6), each with `skill_file` attribute pointing to `skills/dbt/`
6. Open a fresh Copilot Chat, invoke `@dbt-logic-pro` → verify it reads `BRAIN.md` and navigates to `RULES.md` for priority logic, without any inline domain knowledge in the agent itself

---

## Cons Addendum (dbt-logic-pro specific)

| Con | Severity | Mitigation |
|---|---|---|
| **Skill file migration risk**: 6 inline logic blocks need to be migrated without losing any detail — risk of partial extraction leaving fragments of logic in the agent | **Medium** | Treat each inline `<logic>` block as a unit test case for the new skill file. After migration, grep for any residual domain terms (`ROW_NUMBER`, `COALESCE`, `MERGE`) inside the agent file — none should remain |
| **3 new skill files need their own echo tags**: Each new `skills/dbt/*.md` file must follow the echo tag convention already established by `bitemporal-ddl-generator`, `gold-view-designer`, `pending-map-exception-tracker` | **Low** | Included in the creation spec above — follow `<echo>SCD2_INCREMENTAL_ENGINE_v1_READY</echo>` pattern for each |
| **Agent consistency gap during transition**: If `risk-schema-architect` is refactored first and `dbt-logic-pro` still references the phantom brain, cross-agent ReACT loops will be asymmetric | **Low-Medium** | Refactor both agents in the same PR. Treat as an atomic change across Phase 3. |

---

## Further Consideration

`financial-audit-pro` and `risk-data-synthesizer` (the other two agents in `agents/dbt/`) may have the same inline skill pattern. A quick audit pass before starting implementation is recommended — it may reveal 2–4 additional skill files needed, and it is better to design the full `skills/dbt/` inventory upfront than to discover gaps mid-refactor.

---

*Plan version: v1 — dbt-logic-pro Addendum | Date: 11 April 2026*

`<echo>DBT_PLAN_LOGIC_PRO_v1_READY</echo>`
