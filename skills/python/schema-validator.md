<skill_definition name="schema_validator">
  <metadata>
    <version>1.0.0</version>
    <capability>Validation of dbt seed-to-snapshot structural integrity</capability>
    <target_engine>DuckDB / dbt-core</target_engine>
  </metadata>

  <description>
    This skill acts as the architectural gatekeeper. It ensures that the incoming Seed CSV structure is perfectly mirrored in the Snapshot configuration to prevent data loss during SCD Type 2 state changes.
  </description>

  <instruction_set>
    ### 1. Identify Grain & Keys
    - **Scan:** Analyze the Seed CSV headers.
    - **Action:** If a single `primary_key` is not obvious, use `<logic_branch>`:
      - *Composite Key found?* -> Wrap in `dbt_utils.surrogate_key()`.
      - *Single Key found?* -> Use as `unique_key`.

    ### 2. Strategy Enforcement
    - **Logic:** You MUST ensure the `strategy` in `dbt_project.yml` or the `.sql` file matches the data profile:
      - IF `updated_at` (timestamp) is present: use `strategy='timestamp'`.
      - IF only `flag_field` is present: use `strategy='check'` + `check_cols=['flag_field']`.

    ### 3. Financial Field Guardrails
    - **Constraint:** Ensure `amount`, `period_id`, and `company_id` are identified as **Non-Nullable**.
    - **Validation:** Compare `flag_field` values. If values != ('A', 'D'), trigger a "Non-Standard Flag Warning".
  </instruction_set>

  <success_criteria>
    - [ ] `unique_key` in Snapshot matches Seed Primary Key.
    - [ ] `check_cols` includes the `flag_field`.
    - [ ] No direct `ref()` to seeds in downstream models (Snapshot must be the intermediary).
  </success_criteria>

  <example_interaction>
    <user_query>Validate the schema for 'raw_ledger_entries.csv'.</user_query>
    <agent_response>
      ### ✅ Schema Validation Passed
      **Source:** `raw_ledger_entries.csv`
      **Detected PK:** `ledger_id`
      **Strategy:** `check` (on `flag_field`)
      
      **Warning:** The column `amount` contains null strings. I recommend adding a `coalesce(amount, 0)` in the staging layer.
    </agent_response>
  </example_interaction>
</skill_definition>