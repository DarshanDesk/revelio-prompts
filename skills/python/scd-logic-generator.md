<skill_definition name="scd2_logic_generator">
  <metadata>
    <version>1.1.0</version>
    <capability>Automated Generation of SCD Type 2 Snapshots and Filtered Staging Views</capability>
    <logic_pattern>Functional Delete Handling ('A'/'D' Flags)</logic_pattern>
  </metadata>

  <description>
    This skill generates the dbt code required to transform raw CSV seeds into auditable 
    financial models. it enforces the 'A' (Add/Update) and 'D' (Delete) logic 
    standard, ensuring deleted records are logically expired in the snapshot 
    and physically excluded from the reporting views.
  </description>

  <instruction_set>
    ### 1. Snapshot Block Generation
    - **Tagging:** Wrap the snapshot in standard dbt Jinja `{% snapshot %}` tags.
    - **Config Injection:** - Set `strategy='check'`.
      - Set `check_cols` to include the `flag_field` and any financial amount columns.
      - Ensure `invalidate_hard_deletes=True` is considered if the seed might physically remove rows.

    ### 2. Staging Layer Filtering (The "Golden Filter")
    - **Logic:** Generate a SQL model that references the Snapshot.
    - **Mandatory WHERE Clause:** You MUST inject the following exact syntax:
      ```sql
      WHERE dbt_valid_to IS NULL  -- Only the current version
        AND flag_field != 'D'     -- Exclude logically deleted records
      ```
    - **Column Casting:** Automatically cast financial columns (Amount, etc.) to `DECIMAL` or `NUMERIC` for DuckDB precision.

    ### 3. Audit Trail Preservation
    - **Constraint:** Do NOT allow the generation of a `DELETE` statement. 
    - **Instruction:** If the user asks to "Remove records," generate an update to the `flag_field` in the source or a filter in the view instead.
  </instruction_set>

  <code_templates>
    <template name="standard_staging">
    -- models/stg_{{source_name}}.sql
    with source as (
        select * from {{ ref('{{snapshot_name}}') }}
    ),
    filtered as (
        select 
            * from source
        where dbt_valid_to is null 
          and {{flag_col}} != 'D'
    )
    select * from filtered
    </template>
  </code_templates>

  <success_criteria>
    - [ ] Generated SQL contains `dbt_valid_to is null`.
    - [ ] Generated SQL contains `flag_field != 'D'`.
    - [ ] Snapshot strategy is correctly set to `check`.
    - [ ] Resulting DAG shows: Seed -> Snapshot -> Staging.
  </success_criteria>
</skill_definition>