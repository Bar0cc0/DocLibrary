# PostgreSQL Cheat Sheet

https://www.postgresql.org/docs/current/index.html  
https://www.postgresql.org/docs/current/app-psql.html

## Table of Contents
1. [Installation and Setup](#1-installation-and-setup)  

2. [Database Cluster Management](#2-database-cluster-management-linux)  
	2.1. [Create and Manage Clusters](#create-and-manage-clusters)  
	2.2. [Systemd service management](#systemd-service-management-linux)  
	2.3. [System Control](#system-control)  
	2.4. [Utilities](#utilities)

3. [Command Line Interface (psql)](#3-command-line-interface-psql)

4. [Configuration, Backup and Security](#4-configuration-backup-and-security)  
	4.1. [Configuration](#configuration)  
	4.2. [Backup and Restore](#backup-and-restore)  
	4.3. [Security](#security)

5. [Data Types](#5-data-types)  
	5.1. [JSONB Data Type](#jsonb-data-type)

6. [Data Manipulation](#6-data-manipulation)  
	6.1. [Merge Operations](#merge-operations)  
	6.2. [Bulk Insert](#bulk-insert)  
	6.3. [Concurrent Access Control](#concurrent-access-control)

7. [Programming](#7-programming)    
	7.1. [Error Handling](#error-handling)  
	7.2. [Functions](#functions)  
	7.3. [Stored Procedures](#stored-procedures)  
	7.4. [Triggers](#triggers)  
	7.5. [PL/Python](#pl-python)  
	7.6. [Inheritance](#inheritance)  

8. [Table Access Optimization](#8-table-access-optimization)   
	8.1. [Indexing](#indexing)  
	8.2. [Clustering](#clustering)
	8.3. [Partitioning](#partitioning)  
	8.4. [Tablespace](#tablespace)  

9. [High Availability and Replication](#9-high-availability-and-replication)
	9.1. [Logical Replication](#logical-replication)  
	9.2. [Interprocess Communication](#interprocess-communication)

10. [Testing](#10-testing)  
	10.1. [Unit Tests](#unit-tests)  
	10.2. [Assertions](#assertions)

11. [Additional Tools and Resources](#11-additional-tools-and-resources)


## 1. Installation and Setup

```bash
# Install PostgreSQL and dependencies (Linux)
sudo apt update
sudo apt install postgresql postgresql-contrib postgresql-server-dev-all
sudo apt-get install psycopg2 asyncpg

# Windows (using Chocolatey)
# Or download the installer from https://www.postgresql.org/download/windows/
choco install postgresql --version <version>

# .NET Client Libraries
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version <version>

# Check if PostgreSQL is running
sudo service postgresql status     # Linux
Get-Service postgresql*            # Windows

# Start/Stop PostgreSQL service
sudo service postgresql start|stop      # Linux
Start-Service|Stop-Service postgresql*  # Windows

# Add PostgreSQL to system PATH
# Linux
echo 'export PATH=$PATH:/usr/lib/postgresql/<version>/bin' >> ~/.bashrc
source ~/.bashrc
# Windows
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\PostgreSQL\<version>\bin", "User")

# Remove PostgreSQL
sudo apt remove --purge postgresql-<version> postgresql-client-<version> # Linux
choco uninstall postgresql --version <version> # Windows
```

## 2. Database Cluster Management (Linux)

### Create and Manage Clusters
```bash
# List all PostgreSQL clusters
pg_lsclusters

# Create a new PostgreSQL cluster
pg_createcluster <version> <cluster_name>

# Drop a PostgreSQL cluster
pg_dropcluster <version> <cluster_name>

# Start/Stop/Restart a PostgreSQL cluster
pg_ctlcluster <version> <cluster_name> start|stop|restart
```

### Systemd service management (Linux)
```bash
# Service Unit file (e.g., at /etc/systemd/system/postgresql.service)
[Unit]
Description=PostgreSQL database server
Documentation=man:postgres(1)
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=postgres
ExecStart=/usr/local/pgsql/bin/postgres -D /usr/local/pgsql/data
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=infinity # default is 90 seconds but a server performing crash recovery takes longer (hence set to infinity)

[Install]
WantedBy=multi-user.target
```
### System Control
```bash
# Run as service (Linux)
sudo systemctl enable postgresql
sudo systemctl start postgresql
sudo systemctl status postgresql

# Windows service management
sc query postgresql-x64-15
sc start postgresql-x64-15
sc stop postgresql-x64-15
```

### Utilities
```bash
# Create/drop a database
createdb -U postgres -h localhost [-p 5432 -T <template_db>] <db_name>
dropdb -U postgres -h localhost -p 5432 <db_name>


```

## 3. Command Line Interface (psql)

```bash
# Connect to a database
psql -h <host> -U <username> -d <db_name> -p <port> -c "<command>"

# List databases, tables, and objects
\l                  # List all databases
\dt                 # List all tables in current db
\du                 # List all users
\dx                 # List installed extensions
\dn                 # List all schemas
\df                 # List all functions
\di                 # List all indexes
\dv                 # List all views
\dp                 # List all table privileges

# Navigation and connection
\c <db_name>        # Connect to a database
\d <table_name>     # Show table structure
\q                  # Exit PostgreSQL
\?                  # Help

# Import/Export
\i <file>           # Execute SQL from file
\o <file>           # Send output to file
\copy <table> to 'file.csv' with csv header    # Export to CSV
\copy <table> from 'file.csv' with csv header  # Import from CSV

# Formatting and Variables
\x                  # Toggle expanded display
\set <var> <value>  # Set a variable
\echo :<var>        # Print variable value

# System info
\conninfo           # Show connection information
show config_file;   # Show config file location
```

## 4. Configuration, Backup and Security
### Configuration
```bash
# Get PostgreSQL configuration file path
psql -U postgres -c "SHOW config_file;"

# Edit configuration files
sudo nano /etc/postgresql/<version>/main/postgresql.conf  # Main config
sudo nano /etc/postgresql/<version>/main/pg_hba.conf      # Authentication

# Allow local connections in pg_hba.conf
# Add the following line:
local all postgres peer

# Set environment variables for connection
# Linux
export PGDATABASE=postgres PGUSER=postgres PGPASSWORD=password
# Windows
$env:PGDATABASE="postgres"; $env:PGUSER="postgres"; $env:PGPASSWORD="password"

# Update Airflow connection
airflow connections add 'postgres_default' \
    --conn-type 'postgres' \
    --conn-host 'localhost' \
    --conn-login 'postgres' \
    --conn-password 'your_secure_password' \
    --conn-port 5432 \
    --conn-schema 'postgres'
```
### Backup and Restore
```bash
# Backup and Restore
# Logical backup
pg_dump -U postgres -h localhost -d <db_name> > backup.sql
pg_dump -U postgres -h localhost -d <db_name> --format=c --blobs --verbose --file=backup.dump
pg_dump -U postgres -h localhost -d <db_name> --schema-only > schema.sql
pg_dump -U postgres -h localhost -d <db_name> --data-only > data.sql

# Physical backup
pg_basebackup -h localhost -D /path/to/backup -U postgres -P --wal-method=stream --format=tar

# Restore from backup
psql -U postgres -h localhost -d <db_name> -f backup.dump -X --set ON_ERROR_STOP=on 
pg_restore -U postgres -d <db_name> backup.dump
pg_restore -h localhost -U postgres -d <db_name> /path/to/backup/backup_dump.tar

# Transfer data between Clusters
pg_dumpall -p 5432 | psql -d postgres -p 5433

# check if the secondary server is in hot standby mode
SHOW in_hot_standby;

# Check if the primary server is in recovery mode
SHOW recovery_target;

# Check replication status
SELECT * FROM pg_stat_replication;
```

### Security
```bash
# Create/drop a user
createuser -U postgres -h localhost -p 5432 --interactive <username>
dropuser -U postgres -h localhost -p 5432 <username>
```
```sql
# Role Management
CREATE ROLE r_postgres SUPERUSER;
CREATE ROLE r_postgres LOGIN CREATEDB CREATEROLE REPLICATION PASSWORD 'secure_password' CONNECTION LIMIT int;
ALTER ROLE r_postgres SET default_transaction_isolation TO 'read committed';
GRANT ALL PRIVILEGES ON DATABASE postgres TO r_postgres;
REVOKE ALL PRIVILEGES ON DATABASE postgres FROM r_postgres;

# Dropping role procedure
REASSIGN OWNED BY doomed_role TO successor_role;
DROP OWNED BY doomed_role;
DROP ROLE doomed_role;
```
### SSL Configuration
```bash
# Enable SSL in postgresql.conf
ssl = on
# Set SSL certificate and key paths
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
# Restart PostgreSQL service to apply changes
```

## 5. Data Types

### JSONB Data Type
```sql
-- Create a table with a JSONB column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB
);

-- Insert JSON data
INSERT INTO products (name, attributes)
VALUES
    ('Product 1', '{"color": "red", "size": "M", "tags": ["new", "sale"]}');

-- Query the tablel; @> operator = 'contains'
SELECT * FROM products WHERE attributes @> '{"color": "red"}';

-- jsonpath matching; @? operator = 'matches'
SELECT * FROM products 
WHERE attributes @? '$.color == "red"'; -- $ = json root being queried

SELECT * FROM products 
WHERE attributes @? '$.tags[*] == "sale"'; -- matches any item in array 'tags' with value 'sale'

-- Query nested JSONB data; -> operator = 'get'
SELECT name, attributes->'color' AS color
FROM products;

-- Update a JSONB field
UPDATE products
SET attributes = jsonb_set(attributes, '{size}', '"M"')
WHERE id = 1;

-- Delete a key from the JSONB column
UPDATE products
SET attributes = attributes - 'size'
WHERE id = 1;

-- Add a new key-value pair to the JSONB column
UPDATE products
SET attributes = attributes || '{"brand": "Brand A"}'
WHERE id = 2;

-- Create GIN indexes on the JSONB column
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
CREATE INDEX idx_products_attributes_color ON products USING GIN ((attributes->'tags'));
```

## 6. Data Manipulation

### Merge Operations
```sql
-- UPSERT (INSERT with ON CONFLICT)
INSERT INTO {schema}.{table_name} (column1, column2, ...) VALUES (%s, %s, ...)
-- If a conflict (duplicate key violation) occurs on the specified key column(s), the existing row is updated instead of inserting a new one.
ON CONFLICT (key_columns)
-- Each non-key column is updated with the corresponding value from the attempted insert, accessed through the `EXCLUDED` table reference.
DO UPDATE SET 
	non_key_column1 = EXCLUDED.non_key_column1,
	non_key_column2 = EXCLUDED.non_key_column2,
	...;
-- Variant: Ignores the insert if a conflict occurs on the specified key column(s).
ON CONFLICT (key_columns) 
DO NOTHING

-- MERGE (PostgreSQL 15+)
MERGE INTO {schema}.{table_name} AS target
USING (VALUES (%s, %s, ...)) AS source (column1, column2, ...)
ON target.key_column = source.key_column
WHEN MATCHED THEN
	UPDATE SET 
		non_key_column1 = source.non_key_column1,
		non_key_column2 = source.non_key_column2,
		...;
WHEN NOT MATCHED THEN
	INSERT (column1, column2, ...)
	VALUES (source.column1, source.column2, ...);
-- Variant: Ignores the insert if a conflict occurs on the specified key column(s).
WHEN NOT MATCHED THEN
	DO NOTHING;

```
In Read Committed isolation level:
- INSERT with an ON CONFLICT DO UPDATE clause => guarantees that either the INSERT or UPDATE will occur (unless an error occurs)
- MERGE with both INSERT and UPDATE => does not guarantee that either INSERT or UPDATE will occur if the row is concurrently modified by another transaction which entails that the join condition fails

### Bulk Insert
```sql
-- Bulk insert from CSV
COPY {schema}.{table_name} [(column1, column2, ...)]
FROM '/path/to/file.csv'
WITH (
	FORMAT csv, 
	HEADER true, 
	DELIMITER ',', 
	NULL 'NULL', 
	ON_ERROR ignore, 
	LOG_VERBOSITY verbose
);
-- On completion, the COPY command returns the number of rows copied
```

### Concurrent Access Control
#### Isolation Levels

| Isolation Level | Data Visibility | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Special Characteristics |
|-----------------|-----------------|-------------|----------------------|---------------|-------------------------|
| **Read Committed** (default) | Snapshot at start of transaction; can see own uncommitted changes | Prevented | Possible | Possible | Most permissive level |
| **Repeatable Read** | Only data committed before transaction began | Prevented | Prevented | Possible | Does not see changes by other transactions after start |
| **Serializable** | Only data committed before transaction began | Prevented | Prevented | Prevented | Behaves like single-threaded transaction; can lead to serialization failures if concurrent transactions conflict |

```sql
-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;    -- Default
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Use READ ONLY to avoid unnecessary locks
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY;
-- Use READ WRITE if you need to modify data
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE READ WRITE;

-- Row-level locking
-- Acquire an exclusive lock on a row for updates/deletes/selects by other transactions until the transaction is committed
SELECT * FROM accounts WHERE user_id = 123 FOR UPDATE;  -- Exclusive lock
SELECT * FROM accounts WHERE user_id = 123 FOR SHARE;   -- Shared lock
```

### Advanced Querying / Algorithms
```sql
-- Recursive CTEs (with cycle detection)
-- Use cases: graph traversal / tree structures
WITH RECURSIVE employee_tree AS (
	-- Base case
	SELECT id, name, manager_id, ARRAY[id] AS path
	FROM employees
	WHERE id = $1  -- starting employee (value passed as a parameter)

	UNION ALL
	-- Recursive case
	
	SELECT e.id, e.name, e.manager_id, et.path || e.id
	FROM employees e
	
	-- Join with the previous level of the hierarchy
	JOIN employee_tree et ON e.manager_id = et.id
	
	-- cycle prevention
	WHERE NOT e.id = ANY(et.path) -- Prevent cycles in the hierarchy 
);


-- Recursive CTE: Render breadcrumb navigation (Electronics > Computers > Laptops)
WITH RECURSIVE category_path AS (
	SELECT id, name, parent_id, name::TEXT AS path
	FROM categories
	WHERE parent_id IS NULL

	UNION ALL

	SELECT c.id, c.name, c.parent_id, cp.path || ' > ' || c.name
	FROM categories c
	JOIN category_path cp ON c.parent_id = cp.id
);


-- Dynamic feature extraction
-- Use cases: extracting elements from JSONB arrays (e.g., flatten tags) on GIN indexed columns
CREATE INDEX idx_products_tags ON products USING GIN (tags);
SELECT p.id, p.name, tag.value AS tag
FROM products p,
    LATERAL jsonb_array_elements_text(p.tags::jsonb) AS tag
WHERE tag.value = $1;  -- filter by tag (value passed as a parameter)

-- LEFT JOIN LATERAL 
SELECT c.id, c.name, c.email, o.id AS order_id, o.total_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT id, total_amount
    FROM orders
    WHERE o.customer_id = c.id
    ORDER BY created_at DESC
    LIMIT 1		-- Prevents nested loop joins
) o ON true; 	-- Prevents NULL values to be returned

```

## 7. Programming

### Error Handling
```sql
BEGIN
    -- SQL statements
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Duplicate key error: %', SQLERRM;
    WHEN foreign_key_violation THEN
        RAISE NOTICE 'Foreign key error: %', SQLERRM;
    WHEN OTHERS THEN
        RAISE NOTICE 'Unknown error: %', SQLERRM;
END;
```
- `unique_violation` - For duplicate key errors
- `foreign_key_violation` - For FK constraint violations
- `not_null_violation` - For NULL in NOT NULL columns
- `check_violation` - For CHECK constraint failures
- `division_by_zero` - For arithmetic errors
- `SQLSTATE 'xxxxx'` - Using specific error codes
- `OTHERS` - Catch-all for any unhandled exceptions


### Functions
```sql
CREATE OR REPLACE FUNCTION transfer_funds(
    sender_id INT, 
    receiver_id INT, 
    amount DECIMAL
) RETURNS BOOLEAN AS $$
BEGIN
    -- Attempt the transfer
    UPDATE accounts SET balance = balance - amount 
    WHERE user_id = sender_id;
    
    UPDATE accounts SET balance = balance + amount 
    WHERE user_id = receiver_id;
    
    RETURN TRUE;
EXCEPTION
	WHEN SQLSTATE '23505' THEN -- unique_violation
		RAISE NOTICE 'Transfer failed due to duplicate transaction';
		RETURN FALSE;
    WHEN check_violation THEN
        RAISE NOTICE 'Insufficient funds in sender account';
		RETURN FALSE;
    WHEN OTHERS THEN
        RAISE NOTICE 'Transaction failed: %', SQLERRM;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;
-- Call the function
SELECT transfer_funds(123, 456, 100.00);
-- Note: Use PERFORM to call a function without a returning value
```

### Stored Procedures
```sql
CREATE OR REPLACE PROCEDURE process_order(
    p_order_id INT,
    p_customer_id INT,
    p_payment_method VARCHAR,
    OUT p_status VARCHAR,
    OUT p_message VARCHAR
) LANGUAGE plpgsql AS $$
DECLARE
    v_total_amount DECIMAL(10,2);
    v_current_stock INT;
    v_item_id INT;
    v_quantity INT;
    v_product_name VARCHAR(100);
    v_customer_email VARCHAR(100);
    v_order_items CURSOR FOR 
        SELECT item_id, quantity FROM order_items WHERE order_id = p_order_id;
    v_payment_successful BOOLEAN := FALSE;
BEGIN
    -- Initialize output parameters
    p_status := 'ERROR';
    p_message := 'Processing failed for unknown reason';
    
    -- Get order total
    SELECT SUM(oi.quantity * p.price) INTO v_total_amount
    FROM order_items oi
    JOIN products p ON oi.item_id = p.id
    WHERE oi.order_id = p_order_id;
    
    -- Get customer email
    SELECT email INTO v_customer_email
    FROM customers
    WHERE id = p_customer_id;
    
    -- Begin transaction
    BEGIN
        -- Check inventory for each item
        OPEN v_order_items;
        LOOP
            FETCH v_order_items INTO v_item_id, v_quantity;
            EXIT WHEN NOT FOUND;
            
            -- Get current stock
            SELECT stock, name INTO v_current_stock, v_product_name
            FROM products
            WHERE id = v_item_id;
            
            -- Check if enough stock
            IF v_current_stock < v_quantity THEN
                p_message := 'Insufficient stock for product: ' || v_product_name;
                CLOSE v_order_items;
                RETURN;
            END IF;
            
            -- Update inventory
            UPDATE products
            SET stock = stock - v_quantity
            WHERE id = v_item_id;
            
        END LOOP;
        CLOSE v_order_items;
        
        -- Process payment
        IF p_payment_method = 'CREDIT_CARD' THEN
            -- Simulate payment processing
            v_payment_successful := process_credit_card_payment(p_customer_id, v_total_amount);
        ELSIF p_payment_method = 'BANK_TRANSFER' THEN
            v_payment_successful := process_bank_transfer(p_customer_id, v_total_amount);
        ELSIF p_payment_method = 'PAYPAL' THEN
            v_payment_successful := process_paypal_payment(p_customer_id, v_total_amount);
        ELSE
            RAISE EXCEPTION 'Unsupported payment method: %', p_payment_method;
        END IF;
        
        IF NOT v_payment_successful THEN
            RAISE EXCEPTION 'Payment processing failed';
        END IF;
        
        -- Update order status
        UPDATE orders
        SET status = 'PAID', 
            payment_date = CURRENT_TIMESTAMP
        WHERE id = p_order_id;
        
        -- Send confirmation email
        PERFORM send_order_confirmation_email(v_customer_email, p_order_id, v_total_amount);
        
        -- Set success output
        p_status := 'SUCCESS';
        p_message := 'Order processed successfully. Total: $' || v_total_amount;
        
        -- Commit transaction
        COMMIT;
        
    EXCEPTION
        WHEN OTHERS THEN
            -- Rollback transaction on any error
            ROLLBACK;
            p_status := 'ERROR';
            p_message := 'Order processing failed: ' || SQLERRM;
    END;
END;
$$;

-- Call procedure
DO $$
DECLARE
    v_status VARCHAR;
    v_message VARCHAR;
BEGIN
    CALL process_order(1001, 5432, 'CREDIT_CARD', v_status, v_message);
    
    RAISE NOTICE 'Status: %, Message: %', v_status, v_message;
END;
$$;
```

### Triggers
```sql
-- Create a function that will be called by the trigger
CREATE OR REPLACE FUNCTION notify_product_update()
RETURNS TRIGGER AS $$
BEGIN
	-- 'NEW' refers to the new row being inserted or updated
	-- 'OLD' refers to the existing row before the insert/update
	NEW.updated_at := CURRENT_TIMESTAMP;
    PERFORM pg_notify('product_updates', 'Product ' || NEW.id || ' has been updated at ' || NEW.updated_at);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create a trigger that calls the function on UPDATE
CREATE TRIGGER product_update_trigger
    AFTER UPDATE ON products            -- [BEFORE, AFTER] [INSERT, UPDATE, DELETE, TRUNCATE] ON {schema}.{table_name}
    FOR EACH ROW						-- or FOR EACH STATEMENT
    WHEN (OLD.* IS DISTINCT FROM NEW.*) -- Only if the row has changed; OLD/NEW are only accessible for EACH ROW triggers
    EXECUTE FUNCTION notify_product_update();
```

### PL/Python
#### Prerequisites for using PL/Python: 
1. Install the PostgreSQL Python extension (`postgresql-plpython3`)
2. Enable the language in your database: `CREATE EXTENSION plpython3u;`
3. Install required Python packages in the server's Python environment

```bash
# Install PL/Python extension
sudo apt-get install postgresql-plpython3-<version>  # Linux

# Enable PL/Python in the database
sudo -u postgres psql -c "CREATE EXTENSION plpython3u;"

# Install pandas and numpy in the PostgreSQL Python environment
sudo -u postgres psql -c "SELECT plpython3u_install('pandas');"
sudo -u postgres psql -c "SELECT plpython3u_install('numpy');"
```

#### Example
```sql
-- Create Python function
CREATE OR REPLACE FUNCTION py_analyze_data(
    table_name TEXT,
    column_name TEXT
) RETURNS TABLE(metric TEXT, value FLOAT) AS $$
import pandas as pd
import numpy as np

# Get data from PostgreSQL
plan = plpy.prepare("SELECT $1 FROM $2", ["TEXT", "TEXT"])
result = plpy.execute(plan, [column_name, table_name])

try:
    # Process with pandas
    data = pd.Series([row[column_name] for row in result])

    # Perform statistical analysis
    metrics = [
        ("count", float(len(data))),
        ("mean", float(data.mean())),
        ("median", float(data.median())),
        ("std_dev", float(data.std())),
        ("min", float(data.min())),
        ("max", float(data.max())),
        ("range", float(data.max() - data.min())),
        ("first_quartile", float(np.percentile(data, 25))),
        ("third_quartile", float(np.percentile(data, 75)))
    ]
    
    # Log function execution
    plpy.notice(f"Analysis completed at {datetime.now()} - {len(data)} records processed")
    
    return metrics
    
except Exception as e:
    plpy.error(f"Python error in data analysis: {str(e)}")

$$ LANGUAGE plpython3u;

-- Call the Python function
SELECT * FROM py_analyze_data('my_table', 'my_column');
```

### Inheritance
Limitations: 
- does not propagate data from INSERTs or UPDATEs in the parent table to child tables
- all constraints on the parent table are inherited by child tables but foreign key constraints, unique constraint, and indexes are not (they must be defined on each child table)
- maintenance and tuning (e.g., REINDEX, VACUUM) only works on individual, physical tables and do not support recursing over inheritance hierarchies

```sql
CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name VARCHAR(100) NOT NULL,
	price DECIMAL(10, 2) NOT NULL
);

-- Create a child table that inherits all fields from the parent table and adds a new one
CREATE TABLE digital_products (
	file_size INT NOT NULL
) INHERITS (products);

SELECT * FROM ONLY products;	-- Returns only rows from the parent table
SELECT * FROM products; 		-- Returns rows from both parent and child tables

SELECT p.tableoid, p.name FROM products p 				-- Returns the OID (object internal identifier) of the table where each row comes from
SELECT p.tableoid::regclass, p.name FROM products p; 	-- Returns the name of the table where each row comes from
SELECT r.relname, p.name 		-- Returns the name of the table where each row comes from; equivalent to the previous query
	FROM products p, pg_class r -- pg_class contains information about tables, indexes, sequences, etc.
	WHERE p.tableoid = r.oid; 	
```

## 8. Table Access Optimization

### Indexing
Optimize queries that contain WHERE clauses, JOIN conditions, or ORDER BY clauses.

B-tree: sequential scan + sorting => requires a temporary file on disk => storage and I/O overhead
other index types: no sorting, just a lookup => still requires a temporary file on disk but less storage and I/O overhead

```sql
-- B-tree (default index type) for equality and range queries (<, <=, =, >=, >)
CREATE INDEX idx_products_name ON products (name);

-- Covering index (B-tree only)
CREATE INDEX idx_products_name_price ON products INCLUDE (name, price);

-- GiST index for nearest neighbor searches on geometric data types
CREATE INDEX idx_products_location ON products USING GIST (location);

-- GIN index for full-text search or JSONB data
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);

-- Hash index for equality comparisons
CREATE INDEX idx_products_id ON products USING HASH (id);

-- BRIN index for range queries
CREATE INDEX idx_products_created_at ON products USING BRIN (created_at);

--  Reindex a specific index
REINDEX INDEX idx_products_name;

--  Reindex a table to rebuild all indexes
REINDEX TABLE products;
```

### Clustering
```sql
-- Cluster a table using an index = physically reorder data to match the index order
CLUSTER products USING idx_products_name;

-- Rebuild the index after clustering
REINDEX INDEX idx_products_name;
```


### Partitioning
When the table is large and most of the heavily accessed rows are in a specific range of values.
Allows for old data removal without disrupting the physical ordering.

```sql
-- Enable pruning to skip scanning partitions that do not match the query conditions.
SET enable_partition_pruning = on;

-- Create a partitioned table BY RANGE
CREATE TABLE sales (
	id SERIAL PRIMARY KEY,
	sale_date DATE NOT NULL,
	amount DECIMAL(10, 2) NOT NULL
) PARTITION BY RANGE (sale_date); -- BY LIST, HASH, or RANGE

-- Create partitions for specific range
CREATE TABLE sales_2023 PARTITION OF sales
	FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
CREATE TABLE sales_2024 PARTITION OF sales
	FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Create a default partition to catch all unmatched rows (otherwise an error is raised)
CREATE TABLE sales_default PARTITION OF sales DEFAULT;

-- Insert data into the partitioned table
INSERT INTO sales (sale_date, amount) VALUES ('2023-06-15', 100.00);
INSERT INTO sales (sale_date, amount) VALUES ('2024-06-15', 100.00);

-- Update data in a specific partition
-- Fails if updates the partition key (sale_date) to a value outside the partition range 
-- => deletes the row from the partition and inserts it into the correct partition
UPDATE sales_2023 SET amount = 150.00 WHERE sale_date = '2023-06-15';

-- PostgreSQL does not support global indexes on partitioned tables
-- => Create an index on the matching partition
CREATE INDEX idx_sales_2023_date ON sales_2023 (sale_date);

-- Archiving partitions
-- Detach a partition
ALTER TABLE sales DETACH PARTITION sales_2023;
-- Upload to an archive bucket (e.g., S3)
pg_dump -t sales_2023 -Fc -f sales_2023.dump; -- -Fc creates a custom-format backup (compressed, restorable)
aws s3 cp sales_2023.dump s3://your-archive-bucket/sales_2023.dump 
--or to a data lake:
COPY sales_2023 TO '/path/to/sales_2023.parquet' WITH (FORMAT 'parquet', COMPRESSION 'gzip'); 
-- Remove the partition from the database
DROP TABLE sales_2023;

-- Querying a table using on-the-fly partitioning
SELECT * OVER (PARTITION BY sale_date) FROM sales; -- Window function to calculate aggregates over partitions

-- Vacuum all partitions
DO $$
DECLARE
    partition TEXT;
BEGIN
    FOR partition IN
        SELECT tablename FROM pg_tables
        WHERE tablename LIKE 'sales_%'
    LOOP
        EXECUTE format('VACUUM ANALYZE %I', partition);
    END LOOP;
END $$;

-- List all partitions of a partitioned table
SELECT inhrelid::regclass AS partition_name
FROM pg_inherits
WHERE inhparent = 'sales'::regclass;

-- Check partitioning type and bounds
SELECT relname AS partition_name, relkind AS partition_type, pg_get_expr(relpartbound, oid) AS partition_bound
FROM pg_class
JOIN pg_partitioned_table ON partrelid = oid
WHERE relname = 'sales' OR relname LIKE 'sales_%';

-- Check partitioning statistics
SELECT relname AS partition_name,
	   pg_size_pretty(pg_total_relation_size(oid)) AS partition_size,
	   n_live_tup AS row_count
FROM pg_class
WHERE relname LIKE 'sales_%';

-- Explain query plan for partitioned tables; partition filter = number of partitions that were scanned => parallel execution?
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM sales WHERE sale_date BETWEEN '2023-01-01' AND '2024-12-31';

-- Multi-level partitioning
-- Create a partitioned table by RANGE on sale_date ('sales') and then by LIST on region
CREATE TABLE sales_2023_region1 PARTITION OF sales
	FOR VALUES FROM ('2023-01-01') TO ('2024-01-01') 
	PARTITION BY LIST (region);
CREATE TABLE sales_2023_region1_north PARTITION OF sales_2023_region1
	FOR VALUES IN ('CA');

```

### Tablespace
```sql
-- Create a tablespace
CREATE TABLESPACE my_tablespace LOCATION '/path/to/tablespace';

-- Create a table in the tablespace
CREATE TABLE my_table (
	id SERIAL PRIMARY KEY,
	name VARCHAR(100)
) TABLESPACE my_tablespace;

-- Move an existing table to a tablespace
ALTER TABLE my_table SET TABLESPACE my_tablespace;

-- List all tablespaces
SELECT * FROM pg_tablespace;

-- Drop a tablespace
DROP TABLESPACE my_tablespace;

-- Check the size of a tablespace
SELECT pg_size_pretty(pg_tablespace_size('my_tablespace')) AS tablespace_size;

-- Check the size of a table in a tablespace
SELECT pg_size_pretty(pg_total_relation_size('my_table')) AS table_size;
```




## 9. High Availability and Replication
See: https://www.postgresql.org/docs/17/high-availability.html
	 https://www.postgresql.org/docs/17/hot-standby.html

Load Balancing through replication (NB: hot standby servers can only be used for read queries):
- **Logical Replication**: synchronous (sent to standby before commit => slower but no risk of data loss if primary fails)
- **Trigger-based Replication**: asynchronous (sent to standby after commit => faster but risk of data loss if primary fails)

Also:
- **Partitioning** (using indexes)
- **Sharding** (Multiple-Server Parallel Query Execution)

### Logical Replication
Hot Standby for synchronous replication (publication/subscription)
```sql
-- Setup, see: https://www.postgresql.org/docs/17/logical-replication-quick-setup.html

-- Create roles for replication
CREATE ROLE myuser WITH REPLICATION;
CREATE ROLE admin WITH SUPERUSER;

-- Create tables on the primary server (publisher)
CREATE TABLE my_table_1 (id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE my_table_2 (id SERIAL PRIMARY KEY, data INT);
INSERT INTO my_table_1 (data) VALUES ('Initial data');
INSERT INTO my_table_2 (data) VALUES (42);

-- Create tables on the standby server (subscriber): name and structure must match the primary server
-- Note: The tables must be created on the subscriber before creating the subscription
CREATE TABLE my_table_1 (id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE my_table_2 (id SERIAL PRIMARY KEY, data INT);

-- Create a publication on the primary server
CREATE PUBLICATION my_publication_1 FOR TABLE my_table_1; -- publish all operations (insert, update, delete) on my_table_1
CREATE PUBLICATION my_publication_2 FOR TABLE my_table_2 (id, data) -- publish only specific columns
	WITH (publish = 'insert, update, delete') -- only publish insert, update, delete operations
	WHERE (data > 10); -- publish only rows where data > 10

-- Create a subscription on the standby server
CREATE SUBSCRIPTION my_subscription_1 
	CONNECTION 'host=primary_host port=5432 dbname=mydb user=myuser password=mypassword' 
	PUBLICATION my_publication_1;
CREATE SUBSCRIPTION my_subscription_2 
	CONNECTION 'host=primary_host port=5432 dbname=mydb user=myuser password=mypassword' 
	PUBLICATION my_publication_2;
	
-- Check the status of the replication on the primary server
SELECT * FROM pg_stat_replication; 

-- Check the status of the subscription on the subscriber
SELECT * FROM pg_stat_subscription;

-- Check the status of the WAL receiver process on the subscriber
SELECT * FROM pg_stat_wal_receiver;

-- Drop the subscription
DROP SUBSCRIPTION my_subscription_1;
DROP SUBSCRIPTION my_subscription_2;


-- Slot Management
-- Automatic but in some cases (e.g., failover) need to manually create a slot on the primary server (publisher) 
-- and enable hot standby on the standby server (subscriber)
-- Enable logical replication on the primary server (publisher)
ALTER SYSTEM SET wal_level = logical; -- set to 'logical' to enable logical replication
ALTER SYSTEM SET max_replication_slots = 4; -- set the maximum number of replication slots
ALTER SYSTEM SET max_wal_senders = 4; -- set the maximum number of WAL sender processes
-- Restart the PostgreSQL server for changes to take effect

-- Create a replication slot on the primary server
SELECT * FROM pg_create_logical_replication_slot('my_slot', 'pgoutput');
-- Enable hot standby on the standby server (subscriber)
ALTER SYSTEM SET hot_standby = on; -- set to 'on' to enable hot standby
ALTER SYSTEM SET max_replication_slots = 4; -- set the maximum number of replication slots
ALTER SYSTEM SET max_logical_replication_workers = 4; -- set the maximum number of logical replication workers
-- Restart the PostgreSQL server for changes to take effect
```

### Interprocess Communication
```sql
-- Listen for notifications
LISTEN my_channel;

-- Send notification
NOTIFY my_channel, 'Data has been updated';

-- Create notification trigger
CREATE OR REPLACE FUNCTION notify_data_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('data_change', 'Table ' || TG_TABLE_NAME || ' changed');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER data_change_trigger
AFTER INSERT OR UPDATE OR DELETE ON my_table
FOR EACH ROW EXECUTE FUNCTION notify_data_change();
```


## 10. Testing

### Unit Tests
```sql
-- Unit test for a function
CREATE OR REPLACE FUNCTION test_my_function(query_to_test text, expected_output text)
RETURNS void AS $$
DECLARE
    result text;
BEGIN
    result := EXEC query_to_test;
    IF result != 'expected_output' THEN
        RAISE EXCEPTION 'Test failed: %', result;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Unit test for an assertion
CREATE TABLE test_results (
	test_name TEXT,
	result TEXT,
	message TEXT,
	executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE OR REPLACE FUNCTION assert_equals(test_name TEXT, actual INT, expected INT, msg TEXT)
RETURNS VOID AS $$
BEGIN
    INSERT INTO test_results (test_name, result, message)
    VALUES (
        test_name,
        CASE WHEN actual = expected THEN 'PASS' ELSE 'FAIL' END,
        FORMAT('Expected: %s, Got: %s. %s', expected, actual, msg)
    );
END;
$$ LANGUAGE plpgsql;

-- Run the unit test
DO $$
BEGIN
    PERFORM test_my_function('SELECT my_function(''input'')', 'expected_output');
	PERFORM assert_equals('Test My Function', my_function('input'), 'expected_output', 'Function did not return expected output');
	RAISE NOTICE 'Unit test completed successfully';
END;
$$;

```

### Assertions
```sql
-- Assert that all orders have a valid customer
SELECT CASE
  WHEN NOT EXISTS (
    SELECT 1
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.id
    WHERE c.id IS NULL
  )
  THEN 'PASS' ELSE 'FAIL'
END AS test_result;

-- Assert expected vs actual match exactly
SELECT 'FAIL' AS test_result
WHERE EXISTS (
  (SELECT id FROM actual_table)
  EXCEPT
  (SELECT id FROM expected_table)
);

-- Assert that only one user is marked as admin
SELECT 'PASS' AS test_result
FROM users
HAVING COUNT(*) FILTER (WHERE is_admin = TRUE) = 1;

-- Assert that no NULL values exist in a column
IF EXISTS (
  SELECT 1 FROM customers WHERE email IS NULL
)
  THROW 50001, 'Test failed: NULL emails found', 1;
```



## 11. Additional Tools and Resources

### PostgreSQL Tools
- **pgAdmin**: Web-based administration tool
- **pgBadger**: Log analyzer
- **pgBouncer**: Connection pooler
- **Prometheus**: Metrics collection for monitoring (and Grafana for visualization)
- **barman**: Backup and recovery manager
- **pgtune**: PostgreSQL performance tuning

### Extensions
```bash
# Enable extensions in postgresql.conf
shared_preload_libraries = 'pg_stat_statements, pg_partman, pg_repack, pgcrypto'
```
```sql
-- Enable extensions in the database
CREATE EXTENSION IF NOT EXISTS pg_stat_statements; -- Query statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_kcache;     -- Kernel cache statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_monitor;    -- Query performance monitoring
CREATE EXTENSION IF NOT EXISTS pg_hint_plan;       -- Query optimization hints

CREATE EXTENSION IF NOT EXISTS postgis;            -- Spatial data
CREATE EXTENSION IF NOT EXISTS pg_trgm;            -- Trigram matching for text search
CREATE EXTENSION IF NOT EXISTS timescaledb;        -- Time-series data
CREATE EXTENSION IF NOT EXISTS pgcrypto;           -- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS parquet_fdw;        -- pg-parquet foreign data wrapper

CREATE EXTENSION IF NOT EXISTS citus;              -- Distributed database (sharding and parallel query execution)

CREATE EXTENSION IF NOT EXISTS pg_partman;         -- Partition management
CREATE EXTENSION IF NOT EXISTS pg_repack;          -- Repack tables and indexes
CREATE EXTENSION IF NOT EXISTS pg_prewarm;         -- Prewarm relation cache
```

### Test Frameworks by Platform
| Database       | Unit Test Tool                                    | Integration Option                 |
| -------------- | ------------------------------------------------- | ---------------------------------- |
| **SQL Server** | [tSQLt](http://tsqlt.org)                         | SSIS test harness, pytest + pyodbc |
| **PostgreSQL** | `pgTAP`, `pytest-postgresql`                      | DBT tests, integration via Python  |
| **dbt**        | `dbt test`, `schema.yml`, `dbt-utils`             | dbt + CI/CD integration            |

