<prompt_template name="generate_financial_snapshot">
<context>
I have a new seed file: {{seed_name}}.csv. It contains financial line items with a primary key of {{pk}}.
</context>

<task>
1. Identify the primary key and financial columns.
2. Generate a dbt Snapshot using the 'A/D' flag logic.
3. Use strategy='check' on the [flag_field] and [amount] columns.
</task>

<output_constraint>
Ensure the snapshot is configured for the 'snapshots' schema and uses DuckDB-optimized syntax.
</output_constraint>
</prompt_template>