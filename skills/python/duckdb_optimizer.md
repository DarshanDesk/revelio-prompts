<skill_definition name="duckdb_optimizer">
  <metadata>
    <version>1.0.0</version>
    <capability>DuckDB-Specific Performance Tuning & CSV Handling</capability>
    <optimization_target>In-memory processing & Type Casting</logic_pattern>
  </metadata>

  <description>
    This skill optimizes dbt models for the DuckDB engine. It focuses on 
    efficiently reading Seed CSVs, managing memory during large financial 
    aggregations, and ensuring strict data typing to prevent floating-point 
    errors in financial reporting.
  </description>

  <instruction_set>
    ### 1. Optimized CSV Ingestion
    - **Action:** When referencing seeds, suggest the use of `read_csv_auto`.
    - **Logic:** In DuckDB, explicit type definitions in the `read_csv` function prevent "Type Inference" overhead. 
    - **Instruction:** If the seed is >100MB, suggest setting `parallel=True` and `all_varchar=True` followed by explicit casting in the staging layer.

    ### 2. Financial Precision (Decimal vs. Float)
    - **Rule:** NEVER use `FLOAT` for currency or line-item amounts.
    - **Action:** Force `DECIMAL(18, 2)` or `HUGEINT` (for micro-cents) during casting to ensure DuckDB maintains exact precision during `SUM()` operations.

    ### 3. Join Optimization
    - **Action:** Promote the use of the `USING` clause for joins on `line_item_id` or `company_id` to keep the codebase clean and the join plan efficient.
    - **Filter Pushdown:** Ensure that the `flag_field != 'D'` filter is applied *before* any complex joins to reduce the working set size in memory.

    ### 4. Direct Parquet Export (Optional)
    - **Action:** If the user is building a "Mart," suggest materialized as `external` (Parquet) to take advantage of DuckDB's fast columnar writes for downstream BI tools.
  </instruction_set>

  <performance_patterns>
    <pattern name="efficient_casting">
      ```sql
      -- Prefer this:
      SELECT 
          id,
          CAST(amount AS DECIMAL(18,2)) as amount_gold
      FROM {{ ref('seed_file') }}
      ```
    </pattern>
  </performance_patterns>

  <success_criteria>
    - [ ] Seed reads utilize DuckDB's parallel processing where applicable.
    - [ ] Financial columns are explicitly cast to `DECIMAL`.
    - [ ] Joins are structured to minimize memory spill-to-disk.
  </success_criteria>
</skill_definition>