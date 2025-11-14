# Mid-Term--Food-Inspections
Contents
Data Profiling Observations	
Alteryx transformations before loading into databricks	
Data Loading: From Alteryx to Databricks	
Data Modelling and Medallion Architecture	
Bronze Layer: Raw Data Ingestion with Streaming	
Silver Layer: Data Validation and Business Rules	
Gold Layer	
Testing SCD 2 on restaurant_dim	
Visualization	

Alteryx transformations before loading into databricks

1. Embedded Newline Characters in Location Field
Issue: Dallas "Lat Long Location" field contained embedded newlines, causing coordinates to split into separate rows
  "125 W CAMP WISDOM RD
  (32.662836999, -96.824488024)"
Solution: Used Formula tool with Replace(Replace([Lat Long Location], CharFromInt(10), " "), CharFromInt(13), " ") to remove line feed and carriage return characters
2. ZIP Code Format Issues
Issue: Chicago ZIP codes appeared as floats (60659.0) instead of strings
Solution: Applied RegEx tool to remove ".0" suffix: regexp_replace([zip], "\.0$", "")
3. Input Data Tool Configuration
Set delimiter to \t (tab) for TSV files
Checked "First Row Contains Field Names"
Important: Enabled "Allow Newlines in Fields" option to prevent row splitting
4. Column Name Cleanup
Used Dynamic Rename or Formula tool to replace spaces with underscores
Converted special characters in column names (e.g., "License #" to "license")
5. Data Cleansing Tool Usage
Applied "Remove Line Breaks" option specifically for the Location field
Used to clean embedded newlines without complex formulas
These Alteryx-specific transformations ensured the tab-delimited data was properly parsed and ready for downstream processing.


Data Loading: From Alteryx to Databricks

Overview
The data pipeline leveraged Alteryx for initial data preparation and cleansing of the restaurant inspection TSV files, with the processed output being written as Parquet files. These files were then loaded into Databricks volumes as raw tables, establishing the foundation for subsequent medallion architecture transformations.
Parquet File Generation in Alteryx
After completing the data cleansing steps including handling embedded newlines, standardizing column names, and addressing the ZIP code formatting issues which Alteryx was configured to output the processed data as Parquet files. The columnar storage format was selected for its compression efficiency and compatibility with Databricks' distributed processing engine. Both Chicago and Dallas datasets were output as separate Parquet files, maintaining their distinct schemas while ensuring clean, database-friendly column names.
Volume Storage and Raw Schema Design
The Parquet files were loaded directly into Databricks volumes without any hierarchical directory structure, following a straightforward raw table approach. The data was ingested into a simple schema structure:
/Volumes/main/raw/ 
This flat structure aligned with the raw layer philosophy of minimal transformation, preserving the data as close to its Alteryx-processed state as possible.
Data Modelling and Medallion Architecture

Bronze Layer: Raw Data Ingestion with Streaming
Purpose
The Bronze layer serves as the initial ingestion point for raw restaurant inspection data, preserving the original data while adding minimal metadata for tracking and deduplication. This layer uses streaming tables with Change Data Feed (CDF) enabled for real-time processing capabilities.
Architecture Design
Data Sources
Chicago: food_inspection.raw.chicago_inspections
Dallas: food_inspection.raw.dallas_inspections
Both sources contain pre-cleaned data from Alteryx, stored as raw tables in Databricks with database-friendly column names.
Key Features
Streaming Ingestion 
Utilizes Spark Structured Streaming with readChangeFeed option
Enables real-time processing of new inspection records
Supports incremental updates and late-arriving data
Deduplication Strategy 
Creates SHA-256 hash of all non-metadata columns
Uses 1-hour watermark for streaming deduplication
Prevents duplicate records from downstream processing
Metadata Enrichment 
_ingestion_timestamp: Tracks when records entered the pipeline
_source_file: Identifies data origin (set to "raw_table")
Preserves data lineage for auditing
Change Data Feed Handling 
Removes reserved CDF columns before processing
Ensures clean data flow to Silver layer
Maintains compatibility with streaming operations


Silver Layer: Data Validation and Business Rules
Purpose
The Silver layer transforms and validates the Bronze data, applying business rules and data quality expectations to create trusted datasets ready for analytical processing. This layer implements streaming tables with comprehensive data quality checks using Delta Live Tables expectations.
Data Quality Framework
The Silver layer leverages DLT's expectation framework with @dlt.expect_all_or_drop decorators, ensuring only high-quality records proceed to downstream processing. Failed records are automatically dropped, maintaining data integrity.
Chicago Silver Processing
Validation Rules Applied
Required Fields Validation 
dba_name cannot be null
inspection_date and inspection_type must exist
results field must be populated
ZIP Code Standardization 
Removes .0 suffix from float representation
Validates exactly 5-digit format
Handles the Chicago-specific float storage issue
Violation Processing 
Ensures violations field is not empty
Counts violations by splitting pipe-delimited text
Enforces minimum of 1 violation per inspection
Score Derivation 
Implements Chicago-specific scoring logic: 
Pass → 90
Pass w/ Conditions → 80
Fail → 70
No Entry → 0
Others → NULL
Dallas Silver Processing
Validation Rules
Basic Data Quality 
Restaurant name, date, and type validation
ZIP code must match 5-digit pattern
Inspection scores bounded between 0-100
Violation Aggregation 
Creates 25 binary flags for violation presence
Aggregates total violation count across all columns
Handles sparse violation matrix efficiently
Business Rule Enforcement 
High Score Limit: Scores ≥90 limited to maximum 3 violations
Critical Violation Check: Searches all memo fields for "Urgent" or "Critical" keywords
PASS Validation: Prevents PASS result (score ≥70) with critical violations
Metadata Enhancement 
Adds has_critical_violation boolean flag
Creates derived_score (same as inspection_score for Dallas)
Maintains source tracking with source_city
Technical Implementation Details
Streaming Capabilities
Both tables use dlt.read_stream() for incremental processing
CDC columns automatically dropped to prevent conflicts
Maintains real-time data freshness


Gold Layer 
Overview
The Gold layer implements a dimensional data warehouse using a star schema design pattern with natural keys instead of surrogate keys. This layer consolidates data from both Chicago and Dallas restaurant inspection systems into a unified analytical model optimized for business intelligence and reporting.
Architecture Pattern
Schema Type: Star Schema with Bridge Tables
Key Strategy: Natural Keys (business keys) instead of surrogate keys
Processing Mode: Streaming tables using Delta Live Tables (DLT)
SCD Implementation: Type 2 (Slowly Changing Dimensions) for restaurant dimension
Change Tracking: Delta Change Data Feed (CDF) enabled on most tables
Dimension Tables


Bridge Tables
1. bridge_inspection_violation
Purpose: Many-to-many relationship between inspections and violations (normalized)
Natural Key: bridge_id_nk_pk (format: [inspection_id_nk]_[violation_id_nk])
Key Attributes:
inspection_id_nk: Links to fact_inspection
violation_id_nk: Links to dim_violation
data_workflow_name: Source city
Data Extraction:
Chicago: Exploded from pipe-delimited violations field
Dallas: Flattened from 25 violation_description columns
Update Pattern: Streaming with 2-hour watermark and deduplication on composite key

2. bridge_inspection_violation_detail
Purpose: Many-to-many relationship with additional violation context
Natural Key: bridge_detail_id_nk_pk (format: [inspection_id_nk]_[violation_id_nk])
Key Attributes:
inspection_id_nk: Links to fact_inspection
violation_id_nk: Links to dim_violation
violation_code: Denormalized code for convenience
violation_comment: Inspector comments/notes
data_workflow_name: Source city
Data Extraction:
Chicago: Comments extracted using regex pattern -\s*Comments:\s*(.+)$
Dallas: Sourced from violation_memo columns (violation_memo_1 through violation_memo_25)
Update Pattern: Streaming with 2-hour watermark and deduplication on composite key

Design Decisions
Natural Keys vs Surrogate Keys
Rationale: Natural keys are used throughout to:
Maintain referential integrity in streaming scenarios
Enable idempotent processing and reprocessing
Simplify joins without key lookup overhead
Preserve business meaning in keys




