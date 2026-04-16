---
name: python-audit-reviewer
description: "Reviews Python audit patterns in dbt pipelines. Activates when reviewing Python dbt audit logic, analysing test coverage, or performing compliance checks."
metadata:
  version: "1.2.0"
---

<skill_definition name="audit_reviewer">
  <metadata>
    <scope>@workspace (Models, Snapshots, and Seeds)</scope>
  </metadata>


  <instruction_set>
    ### 1. Dependency Chain Audit
    - **Scan:** Analyze the `ref()` and `source()` calls in all files within `models/`.
    - **Violation:** If a model queries a `seed` directly (e.g., `ref('raw_csv_name')`), flag as a **Critical Architectural Violation**.
    - **Remedy:** Instruct the user to point the model to the corresponding `snapshot` or `staging` layer instead.

    ### 2. Functional Delete Leakage Check
    - **Scan:** Look for models derived from snapshots.
    - **Violation:** If the model lacks a `WHERE flag_field != 'D'` (or equivalent) filter.
    - **Risk:** This causes "Ghost Data" to appear in financial reports where deleted items are summed into totals.
    - **Remedy:** Provide the specific `WHERE` clause snippet to be injected.

    ### 3. SCD Type 2 "Current State" Validation
    - **Scan:** Ensure that models consuming snapshots include `WHERE dbt_valid_to IS NULL`.
    - **Violation:** Selecting from a snapshot without filtering for the null expiration date.
    - **Risk:** Duplicate records for the same PK (Type 2 history overlap).

    ### 4. DuckDB Optimization Review
    - **Scan:** Check for inefficient joins on large financial tables.
    - **Action:** Suggest DuckDB-specific improvements like `USING` clauses for clean joins or explicit casting to avoid float-point errors in currency.
  </instruction_set>

  <violation_reporting_format>
    ### 🚨 Audit Violation Found in `{{file_path}}`
    - **Issue:** [Description of the anti-pattern]
    - **Impact:** [Financial/Performance risk, e.g., "Double counting of deleted line items"]
    - **Fix:** ```sql
    -- Suggested Fix
    {{corrected_code_snippet}}
    ```
  </violation_reporting_format>

  <success_criteria>
    - [ ] Zero models query seeds directly.
    - [ ] 100% of snapshot-dependent models filter for `dbt_valid_to is null`.
    - [ ] 100% of snapshot-dependent models filter for `flag_field != 'D'`.
  </success_criteria>
</skill_definition>