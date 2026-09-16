# Devin Ask-Mode Prompt: Oracle Stored Procedure → Spring Boot DAO Migration

## How to use this
Devin performs better with a **two-phase interaction** rather than one giant prompt:
1. **Phase 1 – Discovery/Plan prompt** (below) → let Devin analyze and propose a plan, review it yourself.
2. **Phase 2 – Execution prompt** (below) → only after you approve the plan.

This avoids Devin guessing at transaction boundaries or table relationships it hasn't actually inspected.

---

## PHASE 0 — Inventory & Coverage-Guaranteed Deep-Dive Analysis (run BEFORE Phase 1)

**Why this phase exists:** at ~29–30 SPs/Functions, with one file bundling 8 procedures together, Devin will tend to skim rather than deep-analyze — volume plus bundled files causes silent coverage gaps (exactly what you're seeing). This phase produces no plan and no code; its only job is a complete, independently-verified catalog of every unit in scope, so nothing gets lost before planning even starts.

```
ROLE
You are performing a pre-migration source inventory and deep-dive audit for an
Oracle PL/SQL modernization project. This phase produces NO plan and NO code —
only a verified, complete catalog of every procedure/function in scope.

CONTEXT
The migration scope spans approximately 29-30 stored procedures and functions
across [N] .sql files. Note specifically: [FILE_NAME_WITH_8_SPS] contains 8
distinct procedures bundled in a single file. Treat every individually named
CREATE OR REPLACE PROCEDURE / FUNCTION as its own analysis unit regardless of
how many share a physical file.

STEP 1 — Build the Inventory Manifest (do this first, before any deep analysis)
Scan every .sql file in [source directory/paths] and produce a markdown table:
| # | Unit Name | Type (SP/Function) | File | Start Line | End Line | Called By | Calls Out To |
Populate "Called By" / "Calls Out To" only where explicitly evident from source
(direct CALL/EXEC or function invocation) — mark "TBD - confirm in deep dive"
if uncertain rather than guessing.

STEP 2 — Verify Manifest Completeness
Independently re-scan all files for the patterns `CREATE OR REPLACE PROCEDURE`,
`CREATE OR REPLACE FUNCTION`, and any package-based procedure/function
declarations. Cross-check the count against Step 1. If counts don't match,
find and add the missing units before proceeding. State the final count
explicitly (e.g., "Confirmed: 30 units found, 30 units in manifest").

STEP 3 — Chunked Deep-Dive (process in batches of 4-5 units, not all at once)
For EACH unit in the manifest, in batches, extract and present:
  a. Full parameter list (name, direction IN/OUT/INOUT, PL/SQL type)
  b. Every table touched, with operation type (SELECT/INSERT/UPDATE/DELETE/MERGE)
  c. Every nested SP/function call made, with parameters passed
  d. Cursors, bulk operations (BULK COLLECT/FORALL), dynamic SQL (EXECUTE IMMEDIATE)
  e. Exception handling: every WHEN clause, RAISE_APPLICATION_ERROR code+message,
     custom exception declarations
  f. Any COMMIT/ROLLBACK/SAVEPOINT statements — flag explicitly, with line number
  g. Business/validation logic in plain English (the real-world rule being
     enforced, not just "checks if X is null")
Present each batch's findings before moving to the next batch — do not defer
detail to a final summary, and do not compress multiple units' detail into a
shared paragraph.

STEP 4 — Special Handling for Bundled Files
For [FILE_NAME_WITH_8_SPS] specifically: confirm each of the 8 procedures was
analyzed with the SAME level of detail as standalone-file procedures in Step 3.
List their names explicitly and confirm none were skipped or silently merged
into a single combined analysis.

STEP 5 — Coverage Reconciliation (mandatory closing step)
Produce a final checklist: every unit from the Step 1/2 manifest, marked
[Analyzed ✓] or [Gap - reason]. Zero gaps are acceptable for sign-off; if any
gap exists, go back and close it before ending this phase.

OUTPUT FILES FOR THIS PHASE
- INVENTORY_MANIFEST.md (Step 1-2)
- DEEP_DIVE_ANALYSIS.md (Step 3-4, one clearly-headed section per unit, same
  structure for every unit — no unit gets a shorter treatment than another)
- COVERAGE_CHECKLIST.md (Step 5)

Do not proceed to migration planning until I confirm these three documents are
complete and accurate.
```

---

## PHASE 1 — Discovery & Migration Plan Prompt

```
ROLE
You are acting as a senior Spring Boot architect performing a legacy Oracle PL/SQL
to Java migration. Do NOT write final implementation code yet. Your job right now
is analysis and planning only.

CONTEXT
I need to migrate an Oracle stored procedure named [PARENT_SP_NAME] into a
DAO-based, transactional Spring Boot REST API. This procedure is a DML orchestrator:
it calls out to [N] other stored/nested procedures ([LIST_CHILD_SP_NAMES]) and
performs INSERT/UPDATE/DELETE across multiple tables ([LIST_TABLE_NAMES]).

Source artifacts attached/available in repo:
- [path to parent SP .sql file]
- [paths to child SP .sql files]
- [DDL / table schema scripts, if available]
- [any existing Java entity classes or DB migration scripts already in this repo]

TASKS FOR THIS PHASE
1. Parse the parent SP and every nested SP it calls. Build a call graph showing
   invocation order (sequential vs conditional vs looped calls).
2. For each SP (parent + children), extract and list:
   - Input parameters (IN/OUT/INOUT) and their PL/SQL types
   - Tables touched, and whether each touch is SELECT, INSERT, UPDATE, or DELETE
   - Any cursors, bulk operations (FORALL/BULK COLLECT), or dynamic SQL
   - Business validation logic (NULL checks, existence checks, raised exceptions,
     custom error codes via RAISE_APPLICATION_ERROR)
   - Any commit/rollback/savepoint statements inside the SP (flag these explicitly —
     they must be translated into Spring @Transactional boundaries, not left as-is)
3. Identify shared state between parent and child SPs (e.g., temp tables, package
   variables, global context, sys_context usage) — these are migration risk points.
4. Propose a target Java package/class structure using this pattern:
   - Controller → Service (transactional boundary) → DAO (one per aggregate/table
     group) → Mapper/Entity
   - Map each nested SP to either (a) a private method within the orchestrating
     Service, or (b) its own Service if it's independently reusable elsewhere —
     state your recommendation and why for each.
5. Propose the transaction strategy:
   - Where does @Transactional go (should almost always be the outer
     orchestrating service method, propagation = REQUIRED)
   - How will partial failure be handled (does the original SP do partial commits
     or is it all-or-nothing?)
   - Savepoint-equivalent needs, if any (Spring supports nested tx via
     PROPAGATION_NESTED — flag if the source SP logic actually requires this)
6. Flag any Oracle-specific constructs that don't map 1:1 to JPA/JDBC and need a
   manual decision: sequences (NEXTVAL), triggers relied upon, packages/records,
   ref cursors, autonomous transactions, DBMS_* calls.
7. Output a structured migration plan as markdown, including:
   - Call graph diagram (mermaid or ASCII)
   - Table-to-DAO mapping table
   - Proposed class list with responsibilities
   - Transaction boundary diagram
   - Open questions / assumptions you need me to confirm before you generate code

Do not generate the Spring Boot code yet. Stop after the plan and wait for my
confirmation.
```

---

## PHASE 1.5 — Comprehensive Modernization Documentation Suite (after Phase 0 + Phase 1, before Phase 2 execution)

This phase turns the raw analysis into the six persisted, reviewable documents you asked for — each its own markdown file, so it can be reviewed/versioned independently and later linked from code for traceability. It deliberately reuses Phase 0/Phase 1 outputs as its only source of truth, rather than letting Devin re-derive facts loosely from memory of the SQL.

```
ROLE
You are producing the complete pre-implementation documentation package for the
[PARENT_SP_NAME] modernization. Use INVENTORY_MANIFEST.md, DEEP_DIVE_ANALYSIS.md,
and the Phase 1 migration plan as your source of truth — do not re-derive facts
from scratch or introduce information not already established in those
documents. Where something is uncertain, mark it as an open question rather
than assuming.

Produce the following as SEPARATE markdown files:

1) MASTER_ANALYSIS.md
   Consolidate and formally document what Phase 1 already produced, so it is a
   durable artifact rather than chat output:
   - Full call graph (mermaid `flowchart` or `graph TD`), covering all ~30 units
   - Table-to-DAO mapping table (Table | Operations | Owning DAO | Used By Units)
   - Proposed class list with a one-line responsibility per class
   - Transaction boundary diagram (mermaid, showing @Transactional scope(s),
     propagation type, and where COMMIT/ROLLBACK equivalents occur)
   - Consolidated open questions/assumptions list (carry forward Phase 0/1
     items + add any new ones found while writing this doc)

2) COMPLEXITY_ASSESSMENT.md
   Score every unit from the manifest using this rubric (apply it consistently
   and show the scoring per unit — don't just assign a label with no working):
   - Tables touched: 1-2 = Low, 3-5 = Medium, 6+ = High
   - Nested calls made: 0 = Low, 1-3 = Medium, 4+ = High
   - Control flow: linear = Low, conditional branching = Medium, loops/cursors/
     recursive calls = High
   - Dynamic SQL / EXECUTE IMMEDIATE present: adds one complexity tier
   - Exception/error-handling paths: <2 = Low, 2-5 = Medium, 6+ = High
   Combine into an overall complexity rating (Low/Medium/High) per unit, with a
   one-line rationale (e.g. "High — touches 7 tables, 2 nested cursors, dynamic
   SQL for audit logging"). Present as a markdown table sorted High→Low, since
   this should double as a migration-sequencing risk indicator.

3) FUNCTIONAL_RESPONSIBILITY_CATALOG.md
   For every unit, two levels of description:
   - Summary (1-2 sentences, business language, no technical jargon) — this is
     what a business analyst or product owner should be able to read
   - Detailed responsibility (bullet points covering: trigger condition/when
     it's called, preconditions/validations enforced, every side effect —
     every table write, every downstream call triggered, every notification/
     log/audit entry — and failure conditions with their business meaning, not
     just the raw Oracle error code)

4) TECHNICAL_MODERNIZATION_PLAN.md
   The rationale-backed technical plan, distinct from the raw class list in
   MASTER_ANALYSIS.md:
   - Target architecture decision and why (layered DAO vs JPA vs hybrid;
     justify per complexity tier if the approach should differ between simple
     and complex units)
   - Migration sequencing recommendation: which units to migrate first and why
     (e.g. low-complexity/leaf-level units first to de-risk, or grouped by
     business domain — state which strategy you're proposing and why)
   - Transaction strategy rationale (why REQUIRED vs REQUIRES_NEW vs NESTED for
     each transactional boundary identified)
   - Data consistency/rollback strategy given the original SPs' commit behavior
   - Risk register: top risks (e.g. "Unit X relies on package-level global
     state shared with Unit Y — no direct Java equivalent, needs a redesign
     decision") with a proposed mitigation per risk
   - Effort/complexity-informed grouping for phased delivery (which units can
     ship in the same release safely vs which have cross-dependencies that
     force them together)

5) TRANSACTION_DATA_FLOW_DIAGRAMS.md
   For each distinct end-to-end business transaction (i.e. each entry point a
   caller actually invokes, traced through all nested units it triggers),
   produce a mermaid `sequenceDiagram` or `flowchart` showing:
   - Order of unit invocation (sequential, conditional, looped)
   - Data read from which tables at each step
   - Data written to which tables at each step
   - Points where a failure in a downstream unit affects upstream state
   One diagram per business transaction, not one mega-diagram — name each
   diagram after the business operation it represents, not the SP name alone.

6) PROPOSED_CLASS_DIAGRAM.md
   A mermaid `classDiagram` for the target Spring Boot structure:
   - Controllers, Services, DAOs, Entities/DTOs, exception classes
   - Relationships (composition/dependency arrows) matching the class list from
     MASTER_ANALYSIS.md and the sequencing from TECHNICAL_MODERNIZATION_PLAN.md
   - Annotate which classes correspond to which original SP/Function names in
     a legend or note block, so traceability back to the source is preserved

CROSS-CHECK BEFORE FINALIZING
Confirm every one of the ~30 units from INVENTORY_MANIFEST.md appears in
MASTER_ANALYSIS.md (call graph + table mapping), COMPLEXITY_ASSESSMENT.md, and
FUNCTIONAL_RESPONSIBILITY_CATALOG.md. Report any unit missing from any of
these three and fix it before considering this phase complete.

These six documents, together with the Phase 0 outputs, are the complete input
package for the Phase 2 execution prompt. Do not begin Phase 2 until I've
reviewed and approved these.
```

---

## PHASE 2 — Execution Prompt (after you approve the plan)

```
ROLE
Continue as the Spring Boot architect. The migration plan for [PARENT_SP_NAME]
is approved (see plan above / attached). Now implement it.

REQUIRED INPUT DOCUMENTS (read all of these before writing any code)
INVENTORY_MANIFEST.md, DEEP_DIVE_ANALYSIS.md, MASTER_ANALYSIS.md,
COMPLEXITY_ASSESSMENT.md, FUNCTIONAL_RESPONSIBILITY_CATALOG.md,
TECHNICAL_MODERNIZATION_PLAN.md, TRANSACTION_DATA_FLOW_DIAGRAMS.md, and
PROPOSED_CLASS_DIAGRAM.md. Treat these as the authoritative source of truth for
behavior — do not re-derive logic from the raw SQL if it conflicts with what's
documented; instead, flag the conflict to me.

TECH STACK / CONVENTIONS
- Spring Boot [version], Java [version]
- Persistence: [Spring Data JPA | JdbcTemplate | MyBatis — state which; if the
  logic is heavily set-based DML, prefer JdbcTemplate/NamedParameterJdbcTemplate
  over JPA entity-by-entity saves for performance parity with the original SP]
- Build tool: [Maven|Gradle]
- Package base: [com.company.project]
- Follow existing repo conventions in [reference package/module] for naming,
  exception handling, and layering — inspect it before generating new code.

IMPLEMENTATION REQUIREMENTS
1. Controller
   - One REST endpoint corresponding to the original SP's business operation
     (not one endpoint per nested SP). Use a clear resource-oriented path.
   - Request/response DTOs mirroring the SP's IN/OUT parameters. Validate inputs
     with Bean Validation annotations matching the SP's NULL/format checks.

2. Service layer (transactional boundary)
   - Single @Transactional method (propagation = REQUIRED) orchestrating calls
     to child logic in the exact order derived from the call graph in the plan.
   - Re-implement each nested SP's business/validation logic explicitly in Java
     — do not just wrap the nested SPs as JDBC callable statement calls unless
     I've explicitly said to keep them as DB-side calls for this migration pass.
   - Translate RAISE_APPLICATION_ERROR / custom error codes into a defined
     exception hierarchy (e.g., BusinessRuleException, extending a common
     ApiException) with the same error codes/messages preserved, mapped to
     appropriate HTTP status via a @ControllerAdvice.
   - If the source SP had partial commits/savepoints, replicate that behavior
     using nested transactions or explicit compensation logic — flag this in
     code comments since it changes atomicity guarantees.

3. DAO layer
   - One DAO per table or tightly-coupled table group (per the plan's mapping).
   - Use NamedParameterJdbcTemplate/JPA repository methods that mirror the exact
     WHERE/SET clauses and join conditions from the original DML — do not
     "simplify" business predicates without flagging the change to me.
   - Batch operations from the SP (FORALL, bulk INSERT/UPDATE) must use JDBC
     batch updates, not row-by-row loops, to preserve performance.
   - Preserve any explicit locking (SELECT FOR UPDATE) used in the original SP.

4. Sequences/IDs
   - Map Oracle SEQUENCE.NEXTVAL usage to the equivalent JPA @SequenceGenerator
     or a dedicated DAO method — do not switch to IDENTITY/auto-increment
     unless I confirm that's acceptable.

5. Idempotency & concurrency
   - Since this SP does multi-table DML, add optimistic locking (@Version) or
     explicit row-versioning where the original schema supports it, and note
     where it doesn't.

6. Testing
   - Unit tests for the Service (mock DAOs) covering: happy path, each business
     validation failure branch, and each nested-SP-equivalent step in isolation.
   - Integration test using Testcontainers (Oracle XE or a compatible image) or
     an in-memory equivalent if schema allows, exercising the full flow against
     real DDL, asserting on final table state across ALL affected tables —
     not just the primary one.
   - Include at least one test proving transactional rollback works (force a
     failure in the "last" nested step and assert nothing committed).

7. Documentation
   - Javadoc on the Service method mapping each code block back to the
     originating SP name/section for traceability during review/sign-off.
   - A short MIGRATION_NOTES.md entry: original SP name, Java entry point,
     behavioral differences (if any), and open risks.

DELIVERABLE FORMAT
- Show the full call graph → class mapping once more before writing code.
- Then generate code file by file (Controller, DTOs, Service, exceptions, DAOs,
  tests), pausing after the Service layer for my review before continuing to
  DAOs and tests.

CONSTRAINTS
- Do not silently change business logic, default values, null-handling, or
  ordering of operations from the original SP. If you believe something should
  change, state it separately as a "recommended deviation" and wait for
  approval — do not bake it in silently.
```

---

## Quick-reference tips at scale (~30 units, bundled multi-SP files)

- **The core failure mode you hit** — Devin skimming a large file set — is almost always a *coverage* problem, not a *capability* problem. Fix it by forcing an explicit, self-verified inventory (Phase 0, Steps 1-2) before any real analysis starts. An unverified count is how units silently disappear.
- **Batch, don't dump.** Asking for detail on 30 units in one shot reliably produces shrinking detail toward the end. Explicit batches of 4-5 (Phase 0, Step 3) with a "present as you go" instruction keeps quality flat across the set.
- **Bundled files need an explicit callout.** Devin will treat a file as one unit of work by default. Naming the 8-SP file directly and requiring a per-procedure confirmation (Phase 0, Step 4) is what prevents it from being analyzed as a single blob.
- **Make coverage falsifiable.** "Analyze everything" is not verifiable; a checklist of named units marked ✓/Gap (Phase 0, Step 5; cross-check in Phase 1.5) is. Always ask for the reconciliation artifact, not just the narrative claim that everything was covered.
- **Documentation should cite its source docs, not re-derive.** Phase 1.5 explicitly tells Devin to treat Phase 0/1 outputs as ground truth. Without this, it's common for the "complexity assessment" or "class diagram" pass to quietly re-read the SQL and introduce small inconsistencies with the earlier analysis.

## Quick-reference tips for this migration specifically

- **Nested SP calls**: Devin will default to "flatten everything into one service" — explicitly tell it whether shared child SPs (called from multiple parents elsewhere in your codebase) should become their own reusable `@Service` beans instead of being inlined.
- **Multi-table DML**: Explicitly ask for a table-to-DAO mapping table before code gen — this is the #1 place Devin conflates unrelated tables into one "God DAO."
- **Transactions**: Always name the propagation level explicitly in the prompt (`REQUIRED` vs `REQUIRES_NEW` vs `NESTED`) — don't let Devin infer it, since Oracle's implicit autonomous-transaction/commit behavior inside SPs doesn't map cleanly.
- **Iterate per-SP if the graph is deep**: if the parent SP calls 5+ children, consider running Phase 1 once for the whole tree, but running Phase 2 incrementally per child-SP-turned-class, reviewing each before moving to the next — reduces the chance of compounding errors across a large diff.
