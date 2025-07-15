# SQL Server ADMINISTRATION CHEATSHEET

[0. SQLCMD Utility (ODBC)](#0-sqlcmd-utility-odbc)  
[1. TSQL Basics](#1-tsql-basics)  
[2. Backup & Restore](#3-backup--restore)  
[3. User Management](#4-user-management)  
[4. Monitoring & Optimization (SSMS)](#5-monitoring--optimization)  
[5. Automation](#6-automation-bat-scripts)  


## 0. SQLCMD Utility (ODBC)
```shell
# checks sqlcmd(ODBC) version if installed
sqlcmd "-?"	

# Start sqlcmd interative mode 
sqlcmd
(cmdlets: EXIT quit, 
			GO buffer TSQL statements,
			KILL <process_id>,
			:exit(select <return_value>)
			:connect <server>\,<instance>
			:reset clear cache, 
			:setvar <var_name> <value>,
			:r <sqlScript_path>
)

# Run script
sqlcmd i- C:\MyFolder\MyScript.sql

# Connect to a named instance specifying input and output files
sqlcmd -S <myServerOrLocalhostName>[\<instanceName>] -i <MyScript.sql> -o <MyOutput.rpt>
	
# use Entra ID (=Azure AD) authentication
sqlcmd -S Target_DB_or_DW.testsrv.database.windows.net -G -U bob@contoso.com -P MyAzureADPassword -l 30

# Run query
sqlcmd -d AdventureWorks2022 -Q "SELECT FirstName, LastName FROM Person.Person WHERE LastName LIKE 'Whi%';"
sqlcmd -Q "SELECT * FROM AdventureWorks2022.Person.Person" -o MyOutput.txt

# Create a new SQL Server instance (=container) using latest version of SQL Server 2022, then restore Wide World Importers sample database
sqlcmd create mssql --accept-eula --tag 2022-latest --using https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Full.bak

# List connection strings to be used to connect to the instance that was created
sqlcmd config connection-strings

# Delete instance
sqlcmd delete
```


## 1. TSQL Basics
### Syntax
```sql
/* GOOD PRACTICES */
-- Pragmas
SET NOCOUNT ON;
SET ANSI_NULLS ON;
SET QUOTED_IDENTIFIER ON;

-- always specify a table's schema; dbo = database owner := default
INSERT INTO dbo.Customers
	-- enclose column names in square brackets
	([CustomerId],[Name],[Location],[Email])
VALUES 
	-- add N before strings
	(1, N'some_name', N'some_location', N'some_email');
``` 
### Stored procedures
```sql
-- Execute TSQL stored procedure
EXEC sp_rename 'table_name.old_name', 'new_name', 'COLUMN';	
EXEC sp_who;

-- User defined sp
CREATE OR ALTER PROCEDURE Schema.usp_SomeName
	@SomeVar NVARCHAR(500) = NULL
AS
BEGIN
	...
END;
EXEC Schema.usp_SomeName;
```

## 2. CATALOGS
```sql
/* SQL Server = master system_tables & system views */
USE master;
SELECT table_name, engine 
	FROM INFORMATION_SCHEMA.TABLES;
SELECT table_name, (data_length + index_length)/1024 
	AS 'Data size (kB)' 
	FROM INFORMATION_SCHEMA.TABLES;
```
```shell
sqlcmd -S ServerName -A -Q "SELECT session_id, blocking_session_id FROM `sys.dm_exec_requests` WHERE blocking_session_id <> 0;"
```

**USE msdb**
The msdb database is one of SQL Server's critical system databases that stores information used by SQL Server Agent for scheduling alerts and jobs, as well as backup and restore history information. When creating, modifying, or managing SQL Server Agent jobs, you must first connect to the msdb database since it contains all the tables and stored procedures related to job scheduling and execution.



## 3. BACKUP / RESTORE
```sql
-- Backup files format rules:
-- Database files 	:= .BAK
-- Log files 		:= .TRN
	
USE master;
BACKUP DATABASE [$(db)] TO DISK='$(bakfile)';
```


## 4. USER MANAGEMENT
### Configure access on databases objects 
```sql
-- Create login to grant access to SQL Server 
CREATE LOGIN <computer_name>\<user_name>
	FROM WINDOWS 
	WITH DEFAULT_DATABASE = <database_name>;

-- Create user & grant access to specific db
USE <DB>;
CREATE USER <user_name> 
	WITH PASSWORD 'user_password'
	FOR LOGIN <computer_name>\<user_name>;

-- RBAC (schema.db.security.roles.databases_roles)
CREATE ROLE <role_name>;
GRANT CONNECT ON DATABASE <db_name> TO <role_name>;
GRANT USAGE ON SCHEMA schema_name TO <role_name>;

DROP USER <user_name>;
DROP LOGIN [<computer_name>\<user_name>];
```

### Manage permissions
```sql
-- Set privileges 
GRANT SELECT, INSERT, DELETE, UPDATE ON ALL TABLES IN SCHEMA schema_name TO 'user/group/role';
GRANT SELECT ON db.table TO <user-role_name>;  -- grant permission for specific cmds: select, insert, update, delete 
GRANT CREATE TABLE TO <user-role_name>;

GRANT CREATE PROCEDURE TO <user-role_name>;
GRANT VIEW ON db.proc TO <user-role_name>;     -- allow reading procedure but not run/write 
GRANT EXECUTE ON db.proc TO <user-role_name>;  -- allow to run procedure 
GRANT ALTER ON db.proc TO <user-role_name>;    -- allow to change procedure 

-- Revoke privileges
REVOKE CONNECT TO 'public/guest';           -- disable by-default permissions
REVOKE SELECT ON table_name FROM user_a;
REVOKE <role_name> FROM user_a;
DENY   ...                                  -- overwrite any previous or linked permissions


-- Example:
/****************************************************************
Setup:
SchemaX owner = DBO
SchemaX.T1 owner = User1
SchemaX.T2 owner = User1
SchemaY owner = User1
*****************************************************************/
CREATE USER User1 WITHOUT LOGIN ;
CREATE USER User2 WITHOUT LOGIN ;
GO
CREATE SCHEMA SchemaX;
GO
CREATE SCHEMA SchemaY AUTHORIZATION User1;
GO
CREATE TABLE [SchemaX].[T1] (
	[vendorID] [varchar](255) Not NULL, 
	[totalAmount] [float] Not NULL,    
	[puYear] [int] NULL 
);
CREATE TABLE [SchemaX].[T2] (
	[vendorID] [varchar](255) Not NULL,    
	[totalAmount] [float] Not NULL,    
	[puYear] [int] NULL
);
GO
ALTER AUTHORIZATION ON OBJECT::SchemaX.[T1] TO User1;
ALTER AUTHORIZATION ON OBJECT::SchemaX.[T2] TO User1;

/*****************************************************************************
For user2 to create a MV in SchemaY on SchemaX.T1 and SchemaX.T2, user2 needs:
1. CREATE VIEW permission in the database
2. REFERENCES permission on the schema1
3. SELECT permission on base table T1, T2  
4. ALTER permission on SchemaY
******************************************************************************/
GRANT CREATE VIEW to User2;
GRANT REFERENCES ON SCHEMA::SchemaX to User2;  
GRANT SELECT ON OBJECT::SchemaX.T1 to User2; 
GRANT SELECT ON OBJECT::SchemaX.T2 to User2;
GRANT ALTER ON SCHEMA::SchemaY to User2; 
GO
EXECUTE AS USER = 'User2';  
GO
CREATE materialized VIEW [SchemaY].MV_by_User2 with(distribution=round_robin) 
AS 
	select A.vendorID, sum(A.totalamount) as S, Count_Big(*) as T 
	from [SchemaX].[T1] A
	inner join [SchemaX].[T2] B on A.vendorID = B.vendorID group by A.vendorID ;
GO
revert;
GO
```

### Encryption
```sql
SET @key_str = SHA2('My secret passphrase', 512);                   -- hash key 
USE db_name;
ALTER TABLE table_name MODIFY COLUMN col_name varbinary(255);       -- change col type to binary bytes
UPDATE table_name SET col_name = AES_ENCRYPT(col_name, @key_str);   -- encrypt data using AES protocol 
SELECT cast(AES_DECRYPT(col_name, @key_str) as char(255)) FROM table_name; -- decrypts
```




## 5. MONITORING / OPTIMIZATION 
[https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/database-console-commands?view=sql-server-ver16]


### Statistics for query optimization
```sql
DBCC SHOW_STATISTICS ('database_name', 'statistics_name')
```
### Maintenance w/ DBCC
```sql
-- update space usage information in sys.allocation_units and sys.partitions catalog views
DBCC DROPUSAGE(<database_name>)

-- Clear all cache
DBCC DROPCLEANBUFFERS
DBCC freeproccache
```

### Profiling
```sql
EXEC sp_spaceused
		[ @objname = ] N'objname',			-- qualified | nonqualified name of the table, indexed view, or queue
											-- if @objname isn't specified, results are returned for the whole database
		[ @updateusage = ] N'TRUE|FALSE',	-- update space usage information
		
		[ @oneresultset = ] "1",		 	-- verbosity params
		[ @include_total_xtp_storage = ] "1";
```
```sql
-- Check consistency
DBCC CHECKDB(<database_name>)
DBCC CHECKTABLE(<table_name>)
DBCC CHECKCATALOG(<database_name>)
```

```sql
/* Profiling w/ extended events */
	
-- TSQL ex1: Create an event session
IF EXISTS (SELECT *
	FROM sys.server_event_sessions
	WHERE name = 'session_name')
BEGIN
	-- attempting to create a session with the same name causes an error
	DROP EVENT SESSION session_name
		ON SERVER;
END
GO
CREATE EVENT SESSION [session_name]
	ON SERVER 	-- for SQL Server and Azure SQL Managed Instance
				-- ON DATABASE for Azure SQL Database
	ADD EVENT sqlserver.sql_statement_completed	
	(
		ACTION(sqlserver.sql_text)
		WHERE
		( [sqlserver].[like_i_sql_unicode_string]([sqlserver].[sql_text], N'%SELECT%HAVING%')
		)
	)
	ADD TARGET package0.event_file 
	(SET filename=N'C:\temp\session_name_Target.xel');
GO

-- TSQL ex2: Start/stop session
ALTER EVENT SESSION [session_name]
	ON SERVER
	STATE = START; -- STOP;

-- TSQL ex3: Return all possible events & their descriptions
SELECT obj1.name AS [XEvent-name],
	col2.name AS [XEvent-column],
	obj1.description AS [Descr-name],
	col2.description AS [Descr-column]
FROM sys.dm_xe_objects AS obj1
INNER JOIN sys.dm_xe_object_columns AS col2
	ON col2.object_name = obj1.name
ORDER BY obj1.name,
	col2.name;
GO

-- TSQL ex3: Discover available events in packages
SELECT 
	p.name AS [Package-Name],
	o.object_type,
	o.name AS [Object-Name],
	o.description AS [Object-Descr],
	p.guid AS [Package-Guid]
FROM sys.dm_xe_packages AS p
INNER JOIN sys.dm_xe_objects AS o
	ON p.guid = o.package_guid
WHERE o.object_type = 'event' --'action'  --'target'
	AND p.name LIKE '%'
	AND o.name LIKE '%sql%'
ORDER BY p.name,
	o.object_type,
	o.name;

-- TSQL ex4: Read event session data as XML
SELECT object_name,
	file_name,
	file_offset,
	event_data,
	'CLICK_NEXT_CELL_TO_BROWSE_XML RESULTS!' AS [CLICK_NEXT_CELL_TO_BROWSE_XML_RESULTS],
	CAST(event_data AS XML) AS [event_data_XML]
FROM sys.fn_xe_file_target_read_file(
	'C:\Temp\YourSession_data.xel', NULL, NULL, NULL
);
```

## 6. AUTOMATION (bat scripts)





