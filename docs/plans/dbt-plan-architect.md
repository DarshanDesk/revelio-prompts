# Plan: Domain-Agnostic Agent + External Second Brain

**TL;DR**: Refactor `risk-schema-architect` to a pure ReACT orchestrator (no embedded domain knowledge), and externalize all domain knowledge into a 3-file Second Brain at `.github/context-cache/`. The brain is plain markdown — readable by any AI tool in VS Code (GitHub Copilot, Cursor, Claude Code, Codeium, etc.) without any tool-specific wiring beyond what already exists in `.github/`.

---

## Pre-Flight Audit Findings (PROMPT_ARCHITECT Checks)

| ID | Severity | Issue | Fix |
|---|---|---|---|
| DRIFT-001 | `[CRITICAL]` | Orphan threshold: `0.1%` in agent vs `5.0%` in skill | Canonical value moves to `RULES.md` in Second Brain |
| PHANTOM-001 | `[CRITICAL]` | `.context/data_model_brain.json` referenced but folder does not exist — ReACT "Learn" is a no-op | Replaced by `.github/context-cache/` |
| GAP-001 | `[WARN]` | No other agent reads the shared "Second Brain" — it's write-only | Cross-agent pointer added via `copilot-instructions.md` and `AGENTS.md` |

---

## Phase 0 — Prerequisite Fixes *(unblocks everything)*

1. Reconcile threshold drift: `5.0%` is the correct operational value (sourced from skill). Remove `0.1%` from the agent — it moves exclusively to `RULES.md`.
2. Formally retire `.context/data_model_brain.json` reference throughout the workspace. The new Second Brain replaces it.

---

## Phase 1 — Second Brain: 3-File Structure *(parallel, independent)*

The 7 original files collapse into **3 purpose-built files** inside `.github/context-cache/`. This reduces cognitive load, context budget, and maintenance surface while preserving all knowledge:

```
.github/
  context-cache/
    BRAIN.md      ← Index + domain overview + echo registry       (~80 lines)
    SCHEMA.md     ← DDL contracts + glossary + column conventions  (~150 lines)
    RULES.md      ← Business rules + thresholds + vendor + mapping (~150 lines)
```

**Why 3, not 7?**

| Original 7 files | Consolidated into |
|---|---|
| `00_INDEX.md` | → `BRAIN.md` (leads with index) |
| `01_domain-brain.md` | → `BRAIN.md` (domain overview) |
| `02_schema-registry.md` | → `SCHEMA.md` |
| `03_entity-glossary.md` | → `SCHEMA.md` (appended section) |
| `04_data-vendors.md` + `05_priority-config.md` | → `RULES.md` |
| `06_alert-thresholds.md` + `07_mapping-rules.md` | → `RULES.md` |

**Loading strategy** — designed for context efficiency:
- `BRAIN.md` always loaded first (lightweight, ~80 lines). Contains a summary table that lets an AI agent decide whether `SCHEMA.md` or `RULES.md` is needed without loading both
- `SCHEMA.md` loaded on-demand for DDL/modeling tasks (path-specific instruction)
- `RULES.md` loaded on-demand for operational/audit tasks (path-specific instruction)

3. Create `BRAIN.md` — Sections: `## Navigation Map`, `## Domain Overview` (EAV rationale, SCD2 design philosophy, why bi-temporal), `## Echo Registry` (all echo tags in one place), `## Agent Routing` (which agent handles what)
4. Create `SCHEMA.md` — Sections: `## 5-Table DDL Contracts` (table names, column list, types, portability comments), `## Hash Key Formulas` (COMP_HK MD5 formula), `## Column Conventions` (UPPER_SNAKE_CASE, zero `dbt_*`, audit columns), `## Sentinel Values & Glossary` (IS_CURRENT, PENDING_MAP, ATTR_VALUE single-column rule)
5. Create `RULES.md` — Sections: `## Vendor Feed Rules` (S&P Full/Inc, 4 Long file types, CDC flags A/U/D), `## Priority Matrix` (1=Analyst Override > 2=Inc > 3=Full, ROW_NUMBER pattern), `## Alert Thresholds` (**canonical**: `pending_map_alert_threshold_pct = 5.0%`), `## Mapping Rules` (COALESCE PENDING_MAP fallback, Zero Data Loss contract, 5-step remediation path)

---

## Phase 2 — Model-Agnostic Tool Compatibility Layer *(parallel with Phase 1)*

The tooling layer uses **only VS Code / GitHub Copilot native mechanisms** — no CLAUDE.md, no tool-specific APIs.

```
.github/
  copilot-instructions.md                     ← Repo-wide entry point (all tools)
  instructions/
    dbt.instructions.md                        ← Path-specific (SQL/dbt files only)
AGENTS.md                                      ← Cross-tool agent routing standard
```

6. Create `.github/copilot-instructions.md` — Content: pointer to `context-cache/BRAIN.md`, instruction to load brain before responding, agent routing table (mirrors `custom_instructions.md` agentic flow). **This is the single hook** that makes the Second Brain visible to Copilot without any attachment. Works with VS Code Copilot, GitHub.com Copilot, Copilot code review.
7. Create `.github/instructions/dbt.instructions.md` — Frontmatter: `applyTo: "**/*.sql,**/models/**,**/seeds/**,**/snapshots/**"`. Content: auto-injects `SCHEMA.md` + `RULES.md` when Copilot is working on dbt/SQL files. *No manual `#file:` attachment needed.*
8. Create `AGENTS.md` at repo root — Cross-tool standard (compatible with GitHub Copilot, Claude Code, Cursor, and any tool following the `agentsmd` spec). Contains: agent roster, routing rules, Second Brain pointer. This is the **tool-agnostic** equivalent of the agent routing table.

> **Model Agnosticism note**: `.github/copilot-instructions.md` is read by GitHub Copilot. `AGENTS.md` at root is read by Claude Code, Cursor, and other tools natively. The `context-cache/` files are plain markdown — any AI tool can read them. No tool-specific syntax (`@imports`, auto memory, slash commands) is used anywhere in the Second Brain files themselves.

---

## Phase 3 — Agent Refactor *(depends on Phase 1)*

9. Modify `agents/dbt/risk-schema-architect.agent.md`:
   - **Remove**: entire `<second_brain_integration>` block (hardcoded `.context/data_model_brain.json`), `<design_philosophy>` section, inline domain rationale
   - **Keep**: `<persona>`, `<react_framework_instructions>`, `<capabilities>` (skill refs), `<output_format>`, `<echo>` tag
   - **Add**: `<second_brain_ref>.github/context-cache/BRAIN.md</second_brain_ref>` tag — single pointer to the brain
   - **Add** to `<stage id="perceive">`: "Load `BRAIN.md` before analysis. Navigate to `SCHEMA.md` or `RULES.md` as needed."
   - **Add** to `<stage id="learn">`: "Update `.github/context-cache/` files directly when rules change. In read-only sessions (Copilot inline), flag the drift as `[BRAIN-UPDATE-PENDING]` in the response instead."

---

## Phase 4 — Cross-Agent Alignment *(depends on Phase 1)*

10. Update `agents/dbt/custom_instructions.md` — Replace the hardcoded "Second Brain Protocol" block with a pointer to `.github/context-cache/BRAIN.md`. All 4 agents (schema-architect, dbt-logic-pro, financial-audit-pro, risk-data-synthesizer) share the same single brain reference.

---

## File Inventory

| File | Action | Phase |
|---|---|---|
| `agents/dbt/risk-schema-architect.agent.md` | Modify — strip domain, add `<second_brain_ref>` | 3 |
| `agents/dbt/custom_instructions.md` | Modify — replace hardcoded brain block with pointer | 4 |
| `skills/dbt/pending-map-exception-tracker.md` | Verify threshold is 5.0% — no change needed | 0 |
| `.github/copilot-instructions.md` | Create | 2 |
| `.github/instructions/dbt.instructions.md` | Create | 2 |
| `.github/context-cache/BRAIN.md` | Create | 1 |
| `.github/context-cache/SCHEMA.md` | Create | 1 |
| `.github/context-cache/RULES.md` | Create | 1 |
| `AGENTS.md` | Create | 2 |

---

## Verification Checklist

1. Open a fresh Copilot Chat in VS Code (no `#file:` attachments) → ask *"what is the PENDING_MAP alert threshold?"* → must answer `5.0%` sourced from `copilot-instructions.md`
2. Open any `.sql` file in the workspace → ask Copilot a schema question → verify it references `SCHEMA.md` via the path-specific instruction (no manual attachment)
3. Ask `@risk-schema-architect generate DDL` → verify the agent response uses `BRAIN.md` as its knowledge source, not inline persona text
4. Grep workspace for `0.1%` — must return zero results post-refactor
5. Grep workspace for `.context/data_model_brain.json` — must return zero results post-refactor
6. ECHO check: `<echo>RISK_SCHEMA_ARCHITECT_v1_READY</echo>` must still be present in the refactored agent

---

## Architectural Decisions

- **`.github/context-cache/` over `.context/`** — sits within `.github/` which is the conventional repo tooling directory, and Copilot's instruction system already has a loading mechanism for this location
- **3 brain files over 7** — knowledge density is higher per file, cross-references are fewer, context budget per session is lower
- **`AGENTS.md` at repo root over tool-specific wiring** — the `agentsmd` standard is already recognised by GitHub Copilot, Claude Code, and Cursor natively
- **No `CLAUDE.md`, no `@imports` syntax, no auto-memory** — all tooling hooks use only VS Code / `.github/` native mechanisms to remain model-agnostic

---

## Cons — Honest Assessment

| Con | Severity | Mitigation |
|---|---|---|
| **Copilot is READ-ONLY**: `<stage id="learn">` cannot write to the Second Brain during a Copilot chat session — the ReACT loop is incomplete | **High** | Flag as `[BRAIN-UPDATE-PENDING]` in response. Human reviews and commits changes to `context-cache/`. Copilot code review will surface the delta. |
| **No native auto-load for context-cache**: `.github/context-cache/` files are NOT auto-loaded — Copilot only reads them if wired via `copilot-instructions.md` or attached with `#file:` | **High** | Fully resolved by Phase 2 wiring. Without `copilot-instructions.md`, the brain is invisible. |
| **Context window budget**: `copilot-instructions.md` + `BRAIN.md` + `SCHEMA.md` + skill file can approach Copilot's effective instruction budget. Verbosity kills adherence. | **Medium** | Each brain file capped at 150 lines. `BRAIN.md` summary table acts as a router so both `SCHEMA.md` and `RULES.md` don't always load simultaneously. |
| **Not a true OpenWolf replacement**: OpenWolf-style tools offer semantic search over memory, structured upserts, deduplication, and embeddings. Flat markdown is a Tier 1 approximation — no semantic retrieval. | **Medium** | Accept consciously. Markdown wins on: zero dependencies, git-tracked, IDE-native, human-editable, works offline. For this PoC stage it is the right tier. |
| **No structured mutable state**: The "Learn" stage cannot write structured JSON to the brain. State drift between sessions is manual. | **Low-Medium** | Defer mutable state to a Tier 2 layer (e.g., a future `data_model_brain.json` generated by a scheduled dbt run and committed to git by CI). Brain files remain the immutable, human-curated truth. |
| **Maintenance overhead**: Every schema/rule change requires a PR to `context-cache/` in addition to the skill/agent update | **Low** | This overhead EXISTS today — the drift audit proved it. Externalising makes it intentional and git-auditable instead of silent. |

---

*Plan version: v2 — Model Agnostic, 3-File Second Brain | Date: 11 April 2026*

`<echo>DBT_SECOND_BRAIN_PLAN_v2_READY</echo>`
