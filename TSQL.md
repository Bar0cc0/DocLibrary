# T-SQL & ETL Comprehensive Cheatsheet

[1. Setup & Configuration](#1-setup--configuration)  
[2. Variables & Data Types](#2-variables--data-types)  
[3. Built-in Functions](#3-built-in-functions)  
[4. Table Operations](#4-table-operations)  
[5. Temporary Tables & Variables](#5-temporary-tables--variables)  
[6. Data Manipulation Techniques](#6-data-manipulation-techniques)  
[7. Advanced Query Techniques](#7-advanced-query-techniques)  
[8. Error Handling & Transactions](#8-error-handling--transactions)  
[9. ETL Operations](#9-etl-operations)  
[10. Performance Optimization](#10-performance-optimization)  
[11. Resilience & Monitoring](#11-resilience--monitoring)  
[12. System Management](#12-system-management)  

## 1. Setup & Configuration

```sql
SET NOCOUNT ON;
SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;
GO

-- Enable advanced options
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

-- Enable xp_cmdshell for external commands
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Enable distributed queries
EXEC sp_configure 'Ad Hoc Distributed Queries', 1; 
RECONFIGURE;
```

## 2. Variables & Data Types

### Local & Environment Variables
```sql
SELECT @@VERSION; -- must be first statement in batch; else, use EXEC sp_x
DECLARE @TxtLocalVar VARCHAR(20) 
SELECT @TxtLocalVar = 'Hello world.'
PRINT 'My local var as of ' + CAST(GETDATE() AS CHAR(20)) + 'is ' + @TxtLocalVar;
```

### External Variables from PowerShell
```powershell
# Powershell
function Invoke-SqlCmd {
    param (
        [string]$Script,
        [hashtable]$Variables = @{},
    )
}
Invoke-SqlCmd -Script MyScript.sql -Variables @{
    Var1 = 'Something' 
    Var2 = $ShellScopedVariable
}
```

```sql
-- MyScript.sql
DECLARE @VarOne VARCHAR(100) = N'$(Var1)';
DECLARE @VarTwo VARCHAR(100) = $(Var2);
```

### Type Casting
```sql
-- Casting using CAST or CONVERT
SELECT GETDATE() AS UnconvertedDateTime,
    CAST(GETDATE() AS NVARCHAR(30)) AS UsingCast,
    CONVERT(NVARCHAR(30), GETDATE(), 126) AS UsingConvertTo_ISO8601;

-- Evaluate @ColValue type and cast corresponding @ColName accordingly
IF EXISTS (
    SELECT TRY_CAST(@ColValue AS SMALLINT) WHERE TRY_CAST(@ColValue AS SMALLINT) IS NOT NULL
    ) SET @ColDataType = 'SMALLINT'
```

## 3. Built-in Functions

### String Functions
```sql
-- Using strings patterns and ranges
SELECT SUBSTRING(p.col_name, 1, 25) AS alias;
SELECT * FROM table_name WHERE col_name LIKE '%pattern%';
                                    BETWEEN val AND val;
                                    IN (val, val);
```

### Date Functions
```sql
-- Current date/time
SELECT GETDATE(), SYSDATETIME(), SYSUTCDATETIME();

-- Date parts
SELECT 
    YEAR(date), 
    MONTH(date), 
    DAY(date),
    DATEPART(WEEKDAY, date),
    DATEPART(WEEK, date);

-- Date formatting
SELECT 
    FORMAT(date, 'yyyy-MM-dd'),
    CONVERT(VARCHAR, date, 112); -- yyyymmdd
```

## 4. Table Operations

### Table Creation Best Practices
```sql
-- Create table with constraints and computed columns
CREATE TABLE TableName (
    ID INT IDENTITY(1,1) PRIMARY KEY,
    Column1 VARCHAR(50) NOT NULL,
    Column2 DECIMAL(18,2) NOT NULL,
    -- Computed column
    DerivedColumn AS (Column1 + Column2) PERSISTED,
    -- Check constraint
    CONSTRAINT CK_ColumnCheck CHECK (Column2 >= 0),
    -- Foreign key
    FOREIGN KEY (Column1) REFERENCES OtherTable(OtherColumn)
);

-- Create table with temporal support
CREATE TABLE TableWithHistory (
    ID INT PRIMARY KEY,
    Data VARCHAR(100),
    EffectiveStartDate DATETIME2 GENERATED ALWAYS AS ROW START,
    EffectiveEndDate DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (EffectiveStartDate, EffectiveEndDate)
)
WITH (
    SYSTEM_VERSIONING = ON (
        HISTORY_TABLE = TableHistory,
        DATA_CONSISTENCY_CHECK = ON
    )
);
```

### Index Creation
```sql
-- Basic nonclustered index
CREATE NONCLUSTERED INDEX IX_TableName_Column
    ON TableName(ColumnName);

-- Clustered index
CREATE CLUSTERED INDEX CIX_TableName_Column
    ON TableName(ColumnName);

-- Composite index
CREATE INDEX IX_TableName_Composite
    ON TableName(Column1, Column2);

-- Covering index with included columns
CREATE INDEX IX_TableName_Covering
    ON TableName(KeyColumn)
    INCLUDE (Column1, Column2);

-- Filtered index
CREATE INDEX IX_TableName_Filtered
    ON TableName(Column1)
    WHERE Column2 > 100;
```

## 5. Temporary Tables & Variables

```sql
-- Create temporary table
CREATE TABLE #TempTable (
    ID INT PRIMARY KEY,
    Value VARCHAR(50)
);

-- Table variable
DECLARE @TableVar TABLE (
    ID INT,
    Value VARCHAR(50)
);

-- Example usage
-- Declare variables
DECLARE @FileName NVARCHAR(255);
DECLARE @TableName NVARCHAR(255);

-- Create a temporary table to store file names
IF OBJECT_ID('tempdb..#FileList') IS NOT NULL
    DROP TABLE #FileList;

CREATE TABLE #FileList (ID INT IDENTITY(1,1), FileName NVARCHAR(255));

-- Create a temporary mapping table of CSV files to table names
IF OBJECT_ID('tempdb..#FileTableMapping') IS NOT NULL
    DROP TABLE #FileTableMapping;

CREATE TABLE #FileTableMapping (FileName NVARCHAR(255), TableName NVARCHAR(255));
INSERT INTO #FileTableMapping (FileName, TableName)
VALUES 
    ('telco_customer_churn.csv', 'Customers'),
    ('ecommerce_dataset_updated.csv', 'Transactions');    

-- Cleanup
DROP TABLE #FileList;
DROP TABLE #FileTableMapping;
```

## 6. Data Manipulation Techniques

### MERGE Statement (Upsert)
```sql
-- MERGE for update/insert operations
MERGE INTO TargetTable AS target
USING (SELECT ID, Value FROM SourceTable) AS source
ON (target.ID = source.ID)
WHEN MATCHED THEN
    UPDATE SET target.Value = source.Value
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ID, Value) VALUES (source.ID, source.Value);

-- More complex example with joins
MERGE Production.FactMarket AS target
USING (
    SELECT 
        d.DateID,
        sm.MarketSegment,
        sm.Region,
        sm.MarketValue
    FROM Staging.Market sm
    JOIN Production.DimDate d ON CONVERT(DATE, sm.MarketDate) = d.FullDate
) AS source
ON (target.DateID = source.DateID 
    AND target.MarketSegment = source.MarketSegment
    AND target.Region = source.Region)

-- Update existing records
WHEN MATCHED THEN
    UPDATE SET 
        MarketValue = source.MarketValue,
        ModifiedDate = GETDATE()
        
-- Insert new records
WHEN NOT MATCHED BY TARGET THEN
    INSERT (
        DateID, MarketSegment, Region, MarketValue, LoadDate
    )
    VALUES (
        source.DateID, source.MarketSegment, source.Region, source.MarketValue, GETDATE()
    );
```

### BULK INSERT
```sql
-- Bulk insert from CSV
BULK INSERT TargetTable
FROM 'C:\path\to\file.csv'
WITH (
    FORMAT = 'CSV',
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    TABLOCK,
    ERRORFILE = 'C:\path\to\errors.txt',
    MAXERRORS = 10
);

-- Dynamic path example
BULK INSERT #TelcoData
FROM @FilePath
WITH (
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    TABLOCK
);
```

## 7. Advanced Query Techniques

### CROSS APPLY
```sql
/* 
STRING_SPLIT splits the comma-delimited string in the HeaderRow column into individual values 
CROSS APPLY creates a row for each split value, transforming the single comma-separated string into multiple rows
*/
SET @COL_COUNTER_H = (
    SELECT * FROM #ColumnHeaders CROSS APPLY STRING_SPLIT(#ColumnHeaders.HeaderRow, ',')
);
```

### OPENROWSET
When using OPENROWSET with CSV files, SQL Server requires you to define all columns 
up to the highest one you need to access, even if you don't use them all.
The reason is:
- Positional parsing: SQL Server reads CSV files by position, not by column names
- Sequential reading: The parser needs to know how to identify each column from left to right
- No partial column definitions: You can't tell SQL Server to "skip to column 9" - it needs to know about columns 1-8 too
- So even though we only need columns 9-15 for the Locations data, we have to define columns 1-8 as well 
  so SQL Server knows which columns to map to our col9, col10, etc. variables.

```sql
-- Enable Ad Hoc Queries for OPENROWSET
IF NOT EXISTS (SELECT * FROM sys.configurations 
                WHERE name = 'Ad Hoc Distributed Queries' AND value_in_use = 1)
BEGIN
    EXEC sp_configure 'show advanced options', 1;  
    RECONFIGURE;
    EXEC sp_configure 'Ad Hoc Distributed Queries', 1;
    RECONFIGURE;
END;
GO

FROM OPENROWSET(
    BULK ''' + @FilePath + ''',
    FORMAT = ''CSV'',
    FIRSTROW = 2,
    FIELDTERMINATOR = '','',
    ROWTERMINATOR = ''\n''
) AS Source(
    col1 VARCHAR(100), col2 VARCHAR(100), col3 VARCHAR(100), col4 VARCHAR(100),   
    col5 VARCHAR(100), col6 VARCHAR(100), col7 VARCHAR(100), col8 VARCHAR(100),  
    Country VARCHAR(50), State VARCHAR(50), City VARCHAR(50),   
    ZipCode VARCHAR(20), Latitude DECIMAL(9,6), Longitude DECIMAL(9,6), Population INT
)  
```

### Dynamic SQL
```sql
-- Create and execute dynamic SQL
DECLARE @SQL NVARCHAR(MAX);
SET @SQL = N'SELECT * FROM ' + @TableName + ' WHERE ID = @ID';

EXEC sp_executesql 
    @SQL, 
    N'@ID INT', 
    @ID = 123;
```

### Window Functions
```sql
-- Row number
SELECT 
    Column1,
    ROW_NUMBER() OVER (ORDER BY Column2) AS RowNum
FROM TableName;

-- Partitioning
SELECT 
    Column1,
    ROW_NUMBER() OVER (PARTITION BY Column1 ORDER BY Column2) AS RowInGroup
FROM TableName;
```

### Cursor Pattern
```sql
-- Cursor example
DECLARE cursor_name CURSOR FOR 
    SELECT Column1, Column2 FROM TableName;

OPEN cursor_name;
FETCH NEXT FROM cursor_name INTO @Variable1, @Variable2;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Process data
    -- ...
    
    FETCH NEXT FROM cursor_name INTO @Variable1, @Variable2;
END

CLOSE cursor_name;
DEALLOCATE cursor_name;
```

## 8. Error Handling & Transactions

### Basic Transaction Pattern
```sql
-- Basic transaction pattern
BEGIN TRY
    IF @@TRANCOUNT = 0
        BEGIN TRANSACTION;
    
    -- SQL statements here
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    -- Error handling
END CATCH;

-- Check transaction state
IF XACT_STATE() = -1 
    ROLLBACK TRANSACTION;
```

### Comprehensive Error Handling: TRY...CATCH
```sql
DECLARE @SQL NVARCHAR(MAX);
DECLARE @ErrorMessage NVARCHAR(4000);
DECLARE @ErrorSeverity INT;
DECLARE @ErrorState INT;

BEGIN TRY
    -- Your SQL statement here
    SET @SQL = N'MERGE INTO...';
    EXEC sp_executesql @SQL;
END TRY
BEGIN CATCH
    SELECT 
        @ErrorMessage = ERROR_MESSAGE(),
        @ErrorSeverity = ERROR_SEVERITY(),
        @ErrorState = ERROR_STATE();
    
    -- Log the error
    INSERT INTO ETLLog 
        (ProcedureName, StepName, Status, ErrorMessage)
    VALUES 
        ('ProcedureName', @CurrentStep, 'Failed', @ErrorMessage);
    
    RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
END CATCH
```

### Catching return codes
```sql
SET @Cmd = 'bcp schemaName.tableName out path/filename.bcp -T -n';
EXEC @ReturnCode = xp_cmdshell @Cmd;

IF @ReturnCode <> 0
	RAISERROR('BCP export failed with return code %d', 16, 1, @ReturnCode);
```

### Error Logging
```sql
-- Create error log table if not exists
IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'ErrorLog' AND schema_id = SCHEMA_ID('TelcoChurnQ3'))
BEGIN
    CREATE TABLE TelcoChurnQ3.ErrorLog (
        ErrorLogID INT IDENTITY(1,1) PRIMARY KEY,
        ErrorTime DATETIME DEFAULT GETDATE(),
        ErrorNumber INT,
        ErrorSeverity INT,
        ErrorState INT,
        ErrorProcedure NVARCHAR(128),
        ErrorLine INT,
        ErrorMessage NVARCHAR(4000),
        Operation NVARCHAR(200)
    );
END

-- In your CATCH block
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    INSERT INTO TelcoChurnQ3.ErrorLog
        (ErrorNumber, ErrorSeverity, ErrorState, 
        ErrorProcedure, ErrorLine, ErrorMessage, Operation)
    VALUES
        (ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(), 
        ERROR_PROCEDURE(), ERROR_LINE(), ERROR_MESSAGE(), 'Locations Import');
        
    -- Optionally rethrow the error
    RAISERROR('Error importing Locations data. See error log for details.', 16, 1);
END CATCH
```

## 9. ETL Operations

### ETL Logging Patterns
```sql
-- Log ETL step start
INSERT INTO ETLLog (ProcedureName, StepName, Status)
VALUES ('ProcedureName', 'StepName', 'Started');

-- Log ETL step completion
INSERT INTO ETLLog 
    (ProcedureName, StepName, Status, RowsProcessed, ExecutionTimeSeconds)
VALUES 
    ('ProcedureName', 'StepName', 'Success', 
    @RowsProcessed, DATEDIFF(SECOND, @StepStartTime, GETDATE()));

-- Log ETL error
INSERT INTO ETLLog 
    (ProcedureName, StepName, Status, ErrorMessage)
VALUES 
    ('ProcedureName', 'StepName', 'Failed', @ErrorMessage);
```

### Managing Primary/Foreign Keys
- **Identity columns**: Remove identity columns from INSERT statements so SQL Server handles auto-increment
- **Reference data**: Use MERGE to handle reference data like locations, contract types, etc.
- **Foreign keys**: Use JOINs to lookup foreign key values (e.g., LocationID, InternetServiceID)
- **Schema name**: Use TelcoChurnQ3 schema consistently (assuming that's your schema)
- **Error prevention**: Add WHERE conditions for columns that might be NULL

```sql
-- Import Order:

-- Independent/References tables (No FK)
MERGE INTO TelcoChurnQ3.Locations AS target
USING (
    SELECT DISTINCT col9 AS Country, col10 AS State, col11 AS City
    FROM OPENROWSET(
        /*...*/
    ) AS Source(col1, col2, col3, col4, col5, col6, col7, col8, col9, col10, col11)
) AS source
ON target.Country = source.Country AND target.State = source.State AND target.City = source.City
WHEN NOT MATCHED THEN
    INSERT (Country, State, City)
    VALUES (source.Country, source.State, source.City);

-- Tables containing the main PK (e.g. CustomerID) and FK to refs looked up
INSERT INTO TelcoChurnQ3.CustomersInfo (CustomerID, Gender, Age, LocationID)
SELECT s.col1, s.col2, s.col3, l.LocationID
FROM OPENROWSET(
        /*...*/
) AS s(col1, col2, col3)
JOIN TelcoChurnQ3.Locations l 
ON l.Country = s.col9 AND l.State = s.col10 AND l.City = s.col11 AND l.ZipCode = s.col12;

-- Dependent tables with Fk joined to lookup tables
INSERT INTO TelcoChurnQ3.CustomerContracts (
    CustomerID, ContractTypeID, OfferType)
SELECT 
    s.col1, c.ContractTypeID, s.col20
FROM OPENROWSET(
    /*...*/
) AS s(col1, col2, col3, col4, col5, col6, col7, col8, col9, col10,
		col11, col12, col13, col14, col15, col16, col17, col18, col19, col20)
JOIN TelcoChurnQ3.ContractTypes c ON c.ContractTypeName = s.col20
```

### File Operations
```sql
-- Check if file exists
DECLARE @FileExists INT;
EXEC master.dbo.xp_fileexist 'C:\path\to\file.csv', @FileExists OUTPUT;

-- Execute operating system commands
EXEC xp_cmdshell 'dir C:\path\to\files';

-- Catching return codes
SET @Cmd = 'bcp schemaName.tableName out path/filename.bcp -T -n';
EXEC @ReturnCode = xp_cmdshell @Cmd;

IF @ReturnCode <> 0
    RAISERROR('BCP export failed with return code %d', 16, 1, @ReturnCode);
```

## 10. Performance Optimization

### Batch Processing for Large Tables
```sql
-- Add to Production.usp_ExecuteETL
DECLARE @BatchSize INT = 100000; -- Configurable
DECLARE @MaxID INT, @MinID INT, @CurrentID INT;

SELECT @MinID = MIN(LoanID), @MaxID = MAX(LoanID) FROM Staging.Loan;
SET @CurrentID = @MinID;

WHILE @CurrentID <= @MaxID
BEGIN
    -- Process batch
    INSERT INTO Production.FactLoan (...)
    SELECT ...
    FROM Staging.Loan
    WHERE LoanID BETWEEN @CurrentID AND @CurrentID + @BatchSize - 1;
    
    SET @CurrentID = @CurrentID + @BatchSize;
    
    -- Log batch progress
    INSERT INTO Production.Log (TableName, Status)
    VALUES ('FactLoan', 'Processed batch through ID ' + CAST(@CurrentID AS VARCHAR));
    
    -- Add checkpoints for restartability
    UPDATE Staging.ETLControl SET LastProcessedID = @CurrentID WHERE TableName = 'Loan';
END
```

### Strategic Table Hints
```sql
-- Add NOLOCK for read-only staging data
FROM Staging.Loan WITH (NOLOCK)
JOIN Production.DimDate WITH (NOLOCK) ON CONVERT(DATE, sl.LoanDate) = d.FullDate

-- Use TABLOCK for bulk insert performance
INSERT INTO Production.FactLoan WITH (TABLOCK) (...)
```

### Cached Date Lookups
```sql
-- Create a memory-optimized temp table for date lookups
CREATE TABLE #DateLookup (
    FullDate DATE PRIMARY KEY,
    DateID INT NOT NULL
);

-- Populate with dates from the staging data
INSERT INTO #DateLookup (FullDate, DateID)
SELECT DISTINCT 
    CONVERT(DATE, sl.LoanDate) AS FullDate,
    d.DateID
FROM Staging.Loan sl WITH (NOLOCK)
JOIN Production.DimDate d WITH (NOLOCK) ON 
    CONVERT(DATE, sl.LoanDate) = d.FullDate
OPTION (RECOMPILE);

-- Create clustered index for efficient lookups
CREATE CLUSTERED INDEX IX_DateLookup ON #DateLookup(FullDate);

-- Use this in your joins
-- JOIN #DateLookup d ON CONVERT(DATE, sl.LoanDate) = d.FullDate
```

### Query Parallelism Control
```sql
-- Control parallelism for different ETL steps
-- For dimension loading (single-threaded often better for small dimensions)
OPTION (MAXDOP 1)

-- For fact table loading (multi-threaded better for large tables)
OPTION (MAXDOP 4) 
```

### Columnstore Compression & Partition Switching
```sql
-- Add at the end of ETL process for better compression and analytics performance
SET @CurrentStep = 'ColumnstoreOptimization';
SET @StepStartTime = GETDATE();

-- Force columnstore compression for new rows
ALTER INDEX CCI_FactLoan ON Production.FactLoan REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);

INSERT INTO Production.Log 
    (TableName, Status, ExecutionTimeSeconds)
VALUES 
    ('FactLoan', 'Columnstore Compressed', 
    DATEDIFF(SECOND, @StepStartTime, GETDATE()));

-- Example partition switch (if using table partitioning)
-- This allows for minimal-logging bulk operations

-- For partitioned tables, add code like:
SET @CurrentStep = 'PartitionSwitching';
SET @StepStartTime = GETDATE();

-- Create a staging table with identical structure to the target partition
CREATE TABLE Production.FactLoan_Stage (
    -- Same structure as FactLoan
) ON [partition_scheme_name](partition_column_value);

-- Load data into staging table
INSERT INTO Production.FactLoan_Stage WITH (TABLOCK) (...)
SELECT ... FROM #StagedLoans;

-- Switch the partition
ALTER TABLE Production.FactLoan_Stage SWITCH TO Production.FactLoan PARTITION $partition_number;

INSERT INTO Production.Log 
    (TableName, Status, ExecutionTimeSeconds)
VALUES 
    ('FactLoan', 'Partition Switched', 
    DATEDIFF(SECOND, @StepStartTime, GETDATE()));
```


## 11. Resilience & Monitoring

### Checkpoint/Restart Capability
```sql
CREATE TABLE Production.ETLCheckpoint (
    ETLStepName VARCHAR(100) PRIMARY KEY,
    LastSuccessfulRun DATETIME,
    LastBatchID UNIQUEIDENTIFIER,
    LastPosition INT,
    Status VARCHAR(20)
);

-- In your ETL procedure:
IF EXISTS (SELECT 1 FROM Production.ETLCheckpoint WHERE ETLStepName = @CurrentStep AND Status = 'In Progress')
BEGIN
    -- Restart logic here
    SELECT @BatchID = LastBatchID FROM Production.ETLCheckpoint WHERE ETLStepName = @CurrentStep;
END

-- After each step:
UPDATE Production.ETLCheckpoint 
SET LastSuccessfulRun = GETDATE(), LastBatchID = @BatchID, Status = 'Complete'
WHERE ETLStepName = @CurrentStep;
```

### Deadlock Retry Logic
```sql
CREATE PROCEDURE Production.usp_ExecuteWithRetry
    @SQL NVARCHAR(MAX),
    @MaxRetries INT = 3,
    @DelayMS INT = 500
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @Success BIT = 0;
    
    WHILE @RetryCount < @MaxRetries AND @Success = 0
    BEGIN
        BEGIN TRY
            EXEC sp_executesql @SQL;
            SET @Success = 1;
        END TRY
        BEGIN CATCH
            IF ERROR_NUMBER() = 1205 -- Deadlock victim
            BEGIN
                SET @RetryCount = @RetryCount + 1;
                WAITFOR DELAY '00:00:00.500';  -- 500ms delay
                CONTINUE;
            END
            ELSE
                THROW;
        END CATCH
    END
    
    IF @Success = 0
        RAISERROR('Maximum retries exceeded for operation', 16, 1);
END;
```

### ETL Performance Dashboard
```sql
CREATE VIEW Production.vw_ETLPerformance AS
SELECT 
    CONVERT(DATE, l.Date) AS ExecutionDate,
    l.TableName,
    AVG(l.ExecutionTimeSeconds) AS AvgExecutionTime,
    MAX(l.ExecutionTimeSeconds) AS MaxExecutionTime,
    MIN(l.ExecutionTimeSeconds) AS MinExecutionTime,
    AVG(l.RowsAffected) AS AvgRowsProcessed,
    COUNT(CASE WHEN l.Status LIKE '%Failed%' THEN 1 END) AS FailureCount
FROM Production.Log l
GROUP BY CONVERT(DATE, l.Date), l.TableName;
```

## 12. System Management

### System Queries
```sql
-- Check if object exists
IF OBJECT_ID('TableName') IS NOT NULL
    DROP TABLE TableName;

-- Check if procedure exists
IF EXISTS (SELECT 1 FROM sys.procedures WHERE name = 'ProcName')
    DROP PROCEDURE ProcName;

-- Get list of tables
SELECT name FROM sys.tables WHERE schema_id = SCHEMA_ID('SchemaName');
```

### I/O Resource Governor Configuration
```sql
-- Create separate resource pool for ETL operations
-- Run these as a separate script under admin credentials:

USE master;
GO

-- Create Resource Pool for ETL
CREATE RESOURCE POOL ETL_Pool
    WITH (
        MIN_CPU_PERCENT = 50,  -- Minimum CPU %
        MAX_CPU_PERCENT = 80,  -- Maximum CPU %
        MIN_MEMORY_PERCENT = 50, -- Minimum Memory %
        MAX_MEMORY_PERCENT = 80  -- Maximum Memory %
    );

-- Create Workload Group for ETL
CREATE WORKLOAD GROUP ETL_Group
    WITH (
        IMPORTANCE = HIGH,
        REQUEST_MAX_MEMORY_GRANT_PERCENT = 50,
        REQUEST_MAX_CPU_TIME_SEC = 0,   -- No limit
        REQUEST_MEMORY_GRANT_TIMEOUT_SEC = 60,
        MAX_DOP = 4,
        GROUP_MAX_REQUESTS = 0  -- No limit
    )
    USING ETL_Pool;

-- Create Classifier Function
CREATE FUNCTION dbo.ETL_Classifier()
RETURNS SYSNAME
WITH SCHEMABINDING
AS
BEGIN
    DECLARE @WorkloadGroup SYSNAME;
    
    IF APP_NAME() LIKE '%ETL%' OR APP_NAME() LIKE '%SSIS%'
        SET @WorkloadGroup = 'ETL_Group';
    ELSE
        SET @WorkloadGroup = 'default';
        
    RETURN @WorkloadGroup;
END;

-- Register the classifier function
ALTER RESOURCE GOVERNOR WITH (CLASSIFIER_FUNCTION = dbo.ETL_Classifier);
ALTER RESOURCE GOVERNOR RECONFIGURE;
GO


-- To use this in your ETL process:
-- 1. Set application name at the start of your ETL connection
DECLARE @AppName VARCHAR(128) = 'ETL_FinancialRisk_' + CONVERT(VARCHAR(10), GETDATE(), 112);
SET CONTEXT_INFO 0x0;
SET CONTEXT_INFO @AppName;
```

### Data Lineage Tracking
```sql
CREATE TABLE Production.DataLineage (
    LineageID INT IDENTITY(1,1) PRIMARY KEY,
    SourceSystem VARCHAR(100),
    SourceObject VARCHAR(100),
    SourceColumns VARCHAR(MAX),
    TargetSchema VARCHAR(100),
    TargetTable VARCHAR(100),
    TargetColumns VARCHAR(MAX),
    TransformationLogic VARCHAR(MAX),
    BatchID UNIQUEIDENTIFIER,
    LoadDate DATETIME DEFAULT GETDATE()
);
```