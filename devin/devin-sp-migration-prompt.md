# Devin Ask-Mode Prompt: Oracle Stored Procedure → Spring Boot DAO Migration

## How to use this
Devin performs better with a **two-phase interaction** rather than one giant prompt:
1. **Phase 1 – Discovery/Plan prompt** (below) → let Devin analyze and propose a plan, review it yourself.
2. **Phase 2 – Execution prompt** (below) → only after you approve the plan.

This avoids Devin guessing at transaction boundaries or table relationships it hasn't actually inspected.

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

## PHASE 2 — Execution Prompt (after you approve the plan)

```
ROLE
Continue as the Spring Boot architect. The migration plan for [PARENT_SP_NAME]
is approved (see plan above / attached). Now implement it.

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

## Quick-reference tips for this migration specifically

- **Nested SP calls**: Devin will default to "flatten everything into one service" — explicitly tell it whether shared child SPs (called from multiple parents elsewhere in your codebase) should become their own reusable `@Service` beans instead of being inlined.
- **Multi-table DML**: Explicitly ask for a table-to-DAO mapping table before code gen — this is the #1 place Devin conflates unrelated tables into one "God DAO."
- **Transactions**: Always name the propagation level explicitly in the prompt (`REQUIRED` vs `REQUIRES_NEW` vs `NESTED`) — don't let Devin infer it, since Oracle's implicit autonomous-transaction/commit behavior inside SPs doesn't map cleanly.
- **Iterate per-SP if the graph is deep**: if the parent SP calls 5+ children, consider running Phase 1 once for the whole tree, but running Phase 2 incrementally per child-SP-turned-class, reviewing each before moving to the next — reduces the chance of compounding errors across a large diff.
