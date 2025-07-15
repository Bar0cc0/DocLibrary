# SCHEMA DESIGN AND DATA MODELING

[1. Normalization](#1-normalization)  
[2. Dim-Facts Model](#2-dim-facts-model-star--snowflake-schemas)  
[3. Indexing Strategy](#3-indexing-strategy)  
[4. Staging Tables](#4-understanding-staging-tables-for-data-loading)  
[5. Data Validation](#5-data-validation-through-database-constraints)  
[6. Data Lineage Tracking](#6-data-lineage-in-staging-vs-production-tables)


## 1. Normalization

**Benefits**
- Reduces redundancy (avoids duplicate data)
- Improves data integrity (ensures consistency)
- Simplifies data maintenance (easier updates, insertions, deletions)
- Enhances query performance (by avoiding unnecessary large scans)

**Forms**
- 1NF: remove repeating groups 
- 2NF: remove partial dependencies (e.g., all non-key attributes depend on the whole primary key)
- 3NF: remove transitive dependencies (e.g., non-key attributes depend only on the primary key)
- BCNF: remove any remaining anomalies (e.g., every determinant is a candidate key)



## 2. Dim-Facts Model (star & snowflake schemas)

**Benefits**
- Optimized for Reporting & BI (dims allow easy filtering and grouping)
- Reduces Redundancy
- Improves Query Performance
- Scalable

**Forms**
1. Fact Tables (Transactional Data)
- Stores measurable, quantitative data (e.g., Sales, Revenue, Quantity Sold).
- Contains foreign keys that link to dimension tables.
- Often large and grows over time.

2. Dimension Tables (Descriptive Data)
- Contain descriptive attributes related to the fact data.
- Are denormalized to optimize query performance.
- Usually smaller than the fact table.
---
**Slow Changing Dimensions**  
Type 2 Slowly Changing Dimension (SCD) is a data warehousing technique that preserves the historical changes to dimension attributes over time. Unlike Type 1 SCD (which simply overwrites old values), Type 2 SCD maintains a complete history by creating new records whenever tracked attributes change, allowing for accurate historical analysis and point-in-time reporting.

Type 2 SCD is ideal for dimensions where:
- History matters for analysis (e.g., customer employment changes)
- Time-specific reporting is required
- Audit trails of changes are necessary
- Accuracy of historical metrics is important


## 3. Indexing strategy

**Strategy**
- Clustered indexes on primary/identity columns
- Non-clustered indexes on foreign keys and common filter columns
- Covering indexes for frequent query patterns
- Filtered indexes for specific data subsets

**Implementation Considerations**
- Implement indexes incrementally - Start with the most critical ones
- Monitor index usage - Use sys.dm_db_index_usage_stats to identify unused indexes
- Validate query performance - Test with actual workloads
- Consider columnstore indexes - For large aggregate fact tables in SQL Server 2016+
- Schedule regular index maintenance - Reorganize or rebuild based on fragmentation level

**Types**
1. Clustered Index
- Structure: Data is physically stored in the order of the index. Each table can have only one clustered index because it defines the physical order.
- Use Case: When you frequently query using ranges or sort on a particular column (e.g., dates, IDs).
	```sql
	CREATE CLUSTERED INDEX IX_Sales_OrderDate
	ON Sales(OrderDate);
	```

2. Non-Clustered Index
- Structure: A separate structure from the data, storing a pointer to the actual row. You can have multiple non-clustered indexes on a table.
- Use Case: For frequent lookups on columns not part of the clustered index.
	```sql
	CREATE NONCLUSTERED INDEX IX_Customers_Email
	ON Customers(Email);
	```

3. Unique Index
- Structure: Enforces uniqueness on a column or combination of columns.
- Use Case: Ensure no duplicate values exist (e.g., email addresses).
	```sql
	CREATE UNIQUE NONCLUSTERED INDEX IX_Users_Email
	ON Users(Email);
	```

4. Composite (Multi-Column) Index
- Structure: An index on multiple columns. Useful when queries filter on multiple fields.
- Use Case: Queries involving WHERE or ORDER BY on multiple columns.
	```sql
	CREATE NONCLUSTERED INDEX IX_Sales_CustomerProduct
	ON Sales(CustomerID, ProductID);
	```

5. Covering Index
- Structure: Includes all columns needed by a query, avoiding lookups in the main table.
- Use Case: Optimize SELECT queries where all columns fit in the index.
	```sql
	CREATE NONCLUSTERED INDEX IX_Orders_Covering
	ON Orders(OrderDate)
	INCLUDE (CustomerID, TotalAmount);
	```

6. Filtered Index
- Structure: A specialized non-clustered index on a subset of rows.
- Use Case: Useful for indexing sparse data or specific conditions.
	```sql
	CREATE NONCLUSTERED INDEX IX_Sales_ActiveOrders
	ON Sales(Status)
	WHERE Status = 'Active';
	```

8. Columnstore Index
- Structure: Stores data in compressed, columnar format (instead of rows).
- Use Case: Best for analytical workloads (OLAP) with large datasets.
	```sql
	CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales
	ON Sales;
	```

**Choosing the Right Index for Optimization**
- Clustered Index: Use for primary access (e.g., OrderID).
- Non-Clustered Index: Use for secondary access (e.g., Email).
- Composite Index: Use for multi-column filters (e.g., CustomerID, ProductID).
- Covering Index: Use for read-heavy queries where you want to avoid lookups.
- Filtered Index: Use for queries on a subset (e.g., Status = 'Active').
- Columnstore Index: Use for analytical workloads on large datasets.



## 4. Understanding Staging Tables for Data Loading

Staging tables are intermediate database tables used in ETL (Extract, Transform, Load) processes that act as a temporary landing zone for your raw data before it's transformed and loaded into your dimensional model.

### Why Use Staging Tables That Match CSV Formats Exactly?
1. Direct Load Without Transformation: When staging tables mirror your CSV structure exactly (same column names, same data types), you can use bulk import utilities without any intermediate transformation:
	```sql
	BULK INSERT StarSchema.Staging_Loan
	FROM 'D:\Projects\PortefolioWebsite\Projects\FinancialRisk\Datasets\Cooked\Loan_cooked.csv'
	WITH (
		FORMAT = 'CSV',
		FIRSTROW = 2, -- Skip header row
		FIELDTERMINATOR = ',',
		ROWTERMINATOR = '\n',
		TABLOCK
	);
	```
2. Validation Before Transformation: You can validate the raw data (check for duplicates, missing values, data type issues) before transforming it into your star schema.
3. Error Handling: Problems can be identified and fixed at the staging level without affecting the production tables.
4. Incremental Loading Support: You can compare staging data with existing warehouse data to identify new/changed records.

### How to Create Staging Tables for Your CSVs

```sql
	-- Example matching Loan_cooked.csv structure exactly
	CREATE TABLE StarSchema.Staging_Loan (
		CustomerID INT,
		LoanID NVARCHAR(50),
		LoanDate DATE,
		LoanAmount DECIMAL(18,2),
		LoanType NVARCHAR(50),
		InterestRate FLOAT,
		LoanDuration INT,
		MonthlyPayment DECIMAL(18,2),
		DelayedPayments INT,
		-- All other columns exactly as they appear in the CSV
		-- Including maintaining the same case-sensitivity and naming
	);

	-- Example matching Fraud_cooked.csv structure
	CREATE TABLE StarSchema.Staging_Fraud (
		TransactionID INT,
		CustomerID INT,
		TransactionDate DATETIME,
		TransactionAmount DECIMAL(18,2),
		DistanceFromHome FLOAT,
		DistanceFromLastTransaction FLOAT,
		IsOnlineTransaction BIT,
		-- All other columns exactly as in the CSV
	);
```

### Implementation Process
1. Analyze CSV Structure: Use OPENROWSET to examine your CSV structure:
	```sql
	SELECT TOP 5 *
	FROM OPENROWSET(BULK 'D:\Projects\PortefolioWebsite\Projects\FinancialRisk\Datasets\Cooked\Loan_cooked.csv',
		FORMAT = 'CSV', FIRSTROW = 2) WITH (col1 VARCHAR(MAX)) AS t;
	```
2. Create Staging Tables: With exact column names and appropriate data types
3. Build ETL Pipeline:
	```sql
	-- Step 1: Truncate staging tables before each load
	TRUNCATE TABLE StarSchema.Staging_Loan;

	-- Step 2: Load CSV data into staging
	BULK INSERT StarSchema.Staging_Loan FROM 'path\to\Loan_cooked.csv' WITH (...);

	-- Step 3: Transform and load from staging to dimension/fact tables
	INSERT INTO StarSchema.DimCustomer (...)
	SELECT DISTINCT CustomerID, ... 
	FROM StarSchema.Staging_Loan;

	INSERT INTO StarSchema.FactLoan (...)
	SELECT l.CustomerID, d.DateID, ...
	FROM StarSchema.Staging_Loan l
	JOIN StarSchema.DimDate d ON CONVERT(DATE, l.LoanDate) = d.FullDate;
	```

## 5. Data Validation Through Database Constraints
Adding constraints to your staging tables provides an automatic validation layer that catches data issues early in your ETL process. This is an essential component of data quality management.
By implementing these validation constraints, you ensure that only clean, accurate data makes it past the staging area into your production star schema, dramatically improving overall data quality and reducing downstream issues.

### Types of Data Validation Constraints
1. CHECK Constraints for Value Ranges
CHECK constraints validate that data falls within acceptable business ranges:
	```sql
	-- Example for Loan staging table
	CREATE TABLE StarSchema.Staging_Loan (
		CustomerID INT,
		LoanID NVARCHAR(50),
		LoanAmount DECIMAL(18,2) CONSTRAINT CK_Loan_Amount CHECK (LoanAmount > 0),
		InterestRate FLOAT CONSTRAINT CK_Loan_Rate CHECK (InterestRate BETWEEN 0 AND 30),
		LoanDuration INT CONSTRAINT CK_Loan_Duration CHECK (LoanDuration > 0),
		Age INT CONSTRAINT CK_Customer_Age CHECK (Age BETWEEN 18 AND 100),
		CreditScore INT CONSTRAINT CK_Credit_Score CHECK (CreditScore BETWEEN 300 AND 850)
	);

	-- Example for Market data validation
	CREATE TABLE StarSchema.Staging_Market (
		DateID DATE,
		OpenValue DECIMAL(18,2) CONSTRAINT CK_Market_Open CHECK (OpenValue >= 0),
		CloseValue DECIMAL(18,2) CONSTRAINT CK_Market_Close CHECK (CloseValue >= 0),
		VIX FLOAT CONSTRAINT CK_VIX_Range CHECK (VIX BETWEEN 0 AND 100),
		-- Other columns...
	);

	-- Complex validation for fraud detection
	ALTER TABLE StarSchema.Staging_Fraud
	ADD CONSTRAINT CK_Valid_Transaction_Distance
		CHECK (
			(DistanceFromHome >= 0) AND
			(DistanceFromLastTransaction >= 0) AND
			(TransactionAmount > 0) AND
			(RatioToMedianTransactionAmount > 0)
		);
	```

2. Temporal Validation Constraints
	```sql
	-- Date range validation
	ALTER TABLE StarSchema.Staging_Loan
	ADD CONSTRAINT CK_Valid_Loan_Date
		CHECK (LoanDate BETWEEN '2000-01-01' AND GETDATE());

	-- Transaction time validation
	ALTER TABLE StarSchema.Staging_Fraud
	ADD CONSTRAINT CK_Valid_Transaction_Time
		CHECK (
			CAST(TransactionDate AS DATE) <= GETDATE() AND
			DATEPART(YEAR, TransactionDate) >= 2010
		);
	```
3. Pattern Validation
	```sql
	-- Validate email format
	ALTER TABLE StarSchema.Staging_Customer
	ADD CONSTRAINT CK_Valid_Email
		CHECK (Email LIKE '%_@__%.__%');

	-- Validate phone number format
	ALTER TABLE StarSchema.Staging_Customer
	ADD CONSTRAINT CK_Valid_Phone
		CHECK (Phone LIKE '[0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]' 
			OR Phone LIKE '([0-9][0-9][0-9]) [0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]');
	```
	4. Advanced Validation Using Computed Columns
	```sql
	-- Calculate loan-to-value ratio and validate
	ALTER TABLE StarSchema.Staging_Loan
	ADD PropertyValue DECIMAL(18,2),
		LoanToValueRatio AS (CASE WHEN PropertyValue > 0 
								THEN LoanAmount / PropertyValue 
								ELSE NULL END) PERSISTED,
		CONSTRAINT CK_Valid_LTV CHECK (LoanToValueRatio <= 0.95);
	```

### Implementing Validation in Your ETL Process
1. With BULK INSERT (basic validation):
	```sql
	-- Use ERRORFILE option to capture rejected rows
	BULK INSERT StarSchema.Staging_Loan
	FROM 'D:\Projects\PortefolioWebsite\Projects\FinancialRisk\Datasets\Cooked\Loan_cooked.csv'
	WITH (
		FORMAT = 'CSV',
		FIRSTROW = 2,
		FIELDTERMINATOR = ',',
		ROWTERMINATOR = '\n',
		TABLOCK,
		ERRORFILE = 'D:\Projects\PortefolioWebsite\Projects\FinancialRisk\Logs\loan_errors.txt',
		MAXERRORS = 100 -- Continue after up to 100 errors
	);
	```

2. With Stored Procedure (advanced validation):
	```sql
	CREATE PROCEDURE StarSchema.usp_ValidateLoadLoanData
		@FilePath NVARCHAR(500),
		@ErrorLogPath NVARCHAR(500)
	AS
	BEGIN
		SET NOCOUNT ON;
		
		-- First truncate the staging table
		TRUNCATE TABLE StarSchema.Staging_Loan;
		
		-- Create temporary error table
		CREATE TABLE #ValidationErrors (
			RowNumber INT,
			ErrorMessage VARCHAR(500),
			RawData VARCHAR(MAX)
		);
		
		-- Load data with TRY/CATCH
		BEGIN TRY
			BULK INSERT StarSchema.Staging_Loan
			FROM @FilePath
			WITH (
				FORMAT = 'CSV',
				FIRSTROW = 2,
				FIELDTERMINATOR = ',',
				ROWTERMINATOR = '\n'
			);
		END TRY
		BEGIN CATCH
			INSERT INTO #ValidationErrors VALUES (0, ERROR_MESSAGE(), 'Bulk insert failed');
		END CATCH
		
		-- Additional validation queries
		INSERT INTO #ValidationErrors
		SELECT ROW_NUMBER() OVER (ORDER BY CustomerID), 'Invalid loan amount (zero or negative)', 
			CONCAT(CustomerID, ', ', LoanID, ', ', LoanAmount)
		FROM StarSchema.Staging_Loan
		WHERE LoanAmount <= 0;
		
		-- Additional validation for interest rate
		INSERT INTO #ValidationErrors
		SELECT ROW_NUMBER() OVER (ORDER BY CustomerID), 'Interest rate out of range', 
			CONCAT(CustomerID, ', ', LoanID, ', ', InterestRate)
		FROM StarSchema.Staging_Loan
		WHERE InterestRate < 0 OR InterestRate > 30;
		
		-- Export error report
		IF EXISTS (SELECT 1 FROM #ValidationErrors)
		BEGIN
			-- Write errors to output file
			DECLARE @sql NVARCHAR(MAX);
			SET @sql = 'bcp "SELECT * FROM #ValidationErrors" queryout "' 
					+ @ErrorLogPath + '" -c -t, -T -S ' + @@SERVERNAME;
			
			EXEC xp_cmdshell @sql;
			
			-- Return error count
			SELECT COUNT(*) AS ErrorCount FROM #ValidationErrors;
		END
		ELSE
		BEGIN
			SELECT 0 AS ErrorCount;
		END
		
		-- Clean up
		DROP TABLE #ValidationErrors;
	END;
	```

### Post-Validation Quality Checks
After constraints catch the most obvious errors, run more sophisticated checks:
```sql
-- Check for outliers in loan data
SELECT 
	'InterestRate Outlier' AS IssueType,
	COUNT(*) AS RecordCount,
	AVG(InterestRate) AS AvgValue,
	CustomerID,
	LoanID
FROM StarSchema.Staging_Loan
WHERE InterestRate > (
	SELECT AVG(InterestRate) + 3 * STDEV(InterestRate)
	FROM StarSchema.Staging_Loan
)
GROUP BY CustomerID, LoanID;

-- Validate unique business keys
SELECT 
	CustomerID, 
	LoanID, 
	COUNT(*) AS DuplicateCount
FROM StarSchema.Staging_Loan
GROUP BY CustomerID, LoanID
HAVING COUNT(*) > 1;
```


## 6. Data Lineage in Staging vs. Production Tables

```sql
-- Data lineage columns
DataSourceFile NVARCHAR(255) NULL,
LoadBatchID UNIQUEIDENTIFIER NULL,
LoadDate DATETIME NULL DEFAULT GETDATE(),
LastUpdated DATETIME NULL
```

1. Lineage at Both Layers (enterprise standard)
	- Staging Lineage: Source tracking, initial ingestion metadata
	- Production Lineage: Transformation tracking, business process metadata

2. Minimal Staging / Full Production Lineage
	- Staging: Temporary landing zone with minimal metadata
	- Production: Complete audit trail and lineage information

**When to Use Each Approach**  
Lineage at both layers is beneficial when:
- Regulatory Requirements: You need full traceability from source to final models
- Complex Transformations: Troubleshooting data quality issues requires source tracking
- Extended Staging Retention: Your staging data persists beyond the immediate load
- Production Rollback: You might need to revert to staging data

However, if staging is truly temporary (truncated after each load), then Option 2 (minimal staging lineage) would be more efficient.



## Best Practices for ETL Sequences
- Process Dimensions Before Facts: Load dimension tables before fact tables
- Incremental Processing: Implement change data capture for efficient loads
- Parallel Processing Where Possible: Load independent dimensions concurrently
- Transaction Management: Use transactions for atomicity
- Comprehensive Logging: Track execution time, row counts, and errors
- Idempotence: Ensure procedures can run multiple times with same result
- Restart Capability: Allow ETL to restart from point of failure
