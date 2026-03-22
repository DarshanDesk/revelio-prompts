This document serves as the Technical Architecture Reference for the Coding Assistant AI Agent. It outlines the end-to-end design for processing S&P Global (CIQ) Private Financials using dbt, specifically focusing on the transition from raw delta-feed CSVs to a "Current Valid" Long-structure view.

Architecture Context: S&P CIQ Private Financials (SCD Type 2)
1. Perceive: Data Domain & Source Intelligence
The AI Agent must recognize the specific constraints and behaviors of the S&P CIQ Private Financials feed.

Grain of Data: The feed is "Long" (EAV model). Each row represents a single metric (DataItemID) for a specific CompanyID and FinancialPeriodID.

The Entities:

Vendor (Company): Master record; processed as Incremental.

Period: The temporal container (Annual, Quarterly). SCD Type 2.

Data: The financial values (Revenue, Net Income). SCD Type 2.

Ratio: Derived metrics provided by S&P. SCD Type 2.

Change Data Capture (CDC) Flags:

A (Add): New record.

U (Update): A financial restatement. The previous value for that period is now superseded.

D (Delete): An error correction. The data point should be treated as if it never existed in the "Current" state.

Temporal Columns: The feed provides an updated_at or timestamp column. This is the Business Effective Time and must drive the SCD logic.

2. Reasoning: Design Rationale
The architecture is designed to balance Historical Integrity (Audit) with Downstream Simplicity (Usability).

Why SCD Type 2? Financial analysts require "As-Reported" data. If S&P updates 2023 Revenue in 2025, we must keep both records to explain why historical reports changed.

The Surrogate Key (SK) Necessity: In CIQ, a financialperiodid is not unique in an SCD table because it will appear multiple times due to restatements.

Logic: SK = Hash(Natural_Key + updated_at).

The "ETL Team" Boundary: To prevent downstream "Wide-model" teams from struggling with complex SCD joins (Interval Joins), the ETL team provides a Current Valid View.

This view abstracts the dbt_valid_to logic and filters out 'D' flags, presenting a "Clean Long" state.

Snapshot Strategy: Use the check strategy on the flag and value columns to detect when S&P has issued a restatement.

3. Action: Technical Blueprint
The AI Agent should follow this implementation pattern for all four tables.

A. Folder & Materialization Layering
Seeds (/seeds): Load raw CIQ CSVs.

Staging (/models/staging): * Cast datatypes.

Generate the Hashed SK: {{ dbt_utils.generate_surrogate_key(['id', 'updated_at']) }}.

Snapshots (/snapshots): * Implement dbt snapshot logic.

Set invalidate_hard_deletes=True to handle 'D' flags.

Intermediate (/models/intermediate): * Create the Current Valid View (int_ciq_{entity}_current.sql).

B. The "Current Valid" SQL Pattern
The Agent should implement the "Current" views using this standard:

SQL
-- Materialized as View
SELECT 
    * EXCEPT(dbt_scd_id, dbt_updated_at, dbt_valid_from, dbt_valid_to)
FROM {{ ref('snp_ciq_data') }}
WHERE dbt_valid_to IS NULL  -- Captures only the latest version
  AND flag != 'D'           -- Excludes retracted/deleted records
C. Relationship Management
When joining Ratio to Data, the Agent must use the interval_join logic if querying snapshots, or simple ID joins if querying the "Current" views.

4. Feedback: Validation & Guardrails
The AI Agent must verify the following conditions to ensure the SCD logic has not "leaked" or duplicated data.

1. The Uniqueness Contract
In the Current Valid View, the combination of (Natural_Key + DataItemID) must be unique.

Test: dbt_utils.unique_combination_of_columns.

2. Chronological Integrity
For any Natural Key, the dbt_valid_from of a new version must be greater than or equal to the dbt_valid_to of the previous version.

Test: Custom SQL check for overlapping intervals.

3. CIQ Specific Edge Cases
The Orphan Check: Ensure no Data records exist for a PeriodID that doesn't exist in the Period table.

The Flag Sequence: A 'U' or 'D' flag should logically follow an 'A' flag in the snapshot history.

Hard Delete Verification: If a source row is removed or marked 'D', verify that the snapshot dbt_valid_to is populated and the record is absent from the "Current" view.

AI Agent Prompt Instruction: > "Using the S&P CIQ SCD Context, please generate the dbt code for the Data table pipeline, including the staging model with Hashed SK, the Snapshot configuration, and the Intermediate Current Valid view."