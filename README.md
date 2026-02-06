## High-Volume Query Performance Optimization and Database Security Architecture

<br>

### Project Overview

This production case study simulates a real-world PostgreSQL system operating at scale, where a high-volume relational database already exists in production and the engineer’s responsibility is to:

- Diagnose query performance bottlenecks
- Apply targeted, evidence-based SQL optimizations
- Design and enforce production-grade database security
- Validate improvements through execution plans and role-based testing <br>

Instead of designing a schema from scratch, this case study starts from an inherited system with performance and security debt, reflecting how databases typically evolve in real organizations.


<br>

### Key Focus Areas:
- PostgreSQL query performance engineering
- Execution plan analysis (EXPLAIN ANALYZE)
- Indexing & partitioning strategies
- Role-Based Access Control (RBAC)
- Row-Level Security (RLS)
- Audit-ready security validation

<br>

### Technology Stack
- Database: PostgreSQL 
- Admin Tool: pgAdmin 
- Data Volume: 13+ million transaction records 
- Domain: Financial operations (users, cards, payments)

<br> 

### Project Structure

<img height="350" alt="Work Flow new" src="https://github.com/user-attachments/assets/b5315265-1dd5-4f97-a9b9-fd731f699cb3" />


<br>
<br>
<br>
<br>

## Implementation Phases
### Environment Context (Assumed Baseline)

This case study assumes an operational PostgreSQL environment with existing schemas and production data already loaded. Database setup and connectivity are treated as baseline operational knowledge and are intentionally excluded from the core documentation. <br>

The case study begins at the point where real-world database engineering work typically starts: improving an inherited system under load.

<br>
<br>

<br>

### Phase 1. Inherited Schema & Baseline State
<img height="300" alt="schemas" src="https://github.com/user-attachments/assets/2f0e5508-92aa-43cc-97fe-8bee3088d77a" />

#### Context
The database schema contains normalized production tables supporting financial operations:
- core.users: customer records (PII)
- core.cards: payment card data (PCI-sensitive)
- core.transactions: high-volume fact table (13M+ rows)

<br>

#### Intentional Constraints
The system reflects common early-stage technical debt:
- No indexes beyond primary keys
- No partitioning
- No access control
- No row-level data restrictions <br>

This baseline establishes measurable performance pain and security exposure necessary to justify subsequent improvements.


<br>
<br>
<br>

### Phase 2. Baseline Performance Benchmarking
#### Objective
Establish clear, measurable evidence of performance limitations before any optimization, indexing, or tuning is applied. <br>
This phase defines the performance baseline and justifies subsequent optimization decisions. <br>
The goal was to:
- Identify workload bottlenecks
- Capture baseline execution plans
- Measure real query response times
- Document why performance is degraded

<br>

#### 1. Identify the Largest Tables
An initial storage analysis was performed to identify the primary workload drivers within the database. The following query was used to inspect total table sizes across the schemas:

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

<br>

Output:
```
"table_name"	    "total_size"
"transactions"	    "1654 MB"
"transactions"	    "1503 MB"
"cards"		        "920 kB"
"cards"		        "736 kB"
"users"		        "368 kB"
"users"		        "280 kB"
```

The result shows table sizes from both the staging and core schemas. <BR>
The transactions table is significantly larger than all other tables, exceeding 1.5GB in size. This confirms it as the primary source of query latency and the main focus for baseline performance benchmarking and subsequent optimization.

<br>

#### 2. Identification of Realistic Query Patterns
Before executing benchmarks, common query patterns were identified based on realistic organizational usage rather than synthetic tests. <br>
These patterns reflect how analysts, finance teams, and operations typically interact with production data. <br>

**Identified Query Categories:**
| Query Category       | Typical Use Case                |
| :------------------- | :------------------------------ |
| Date-range filtering | Monthly and daily reports       |
| User-based lookups   | Customer support investigations |
| Aggregations         | Financial and trend analysis    |
| Error monitoring     | Operational and fraud checks    |

<br>

#### 3. Measure Execution Plans
Representative queries were executed without indexes to capture baseline execution behavior and response times. 

<br>

**Query 1: Date-Range Transaction Filtering** <br>
Use case: Retrieve transactions within a specific date range. <br>

```sql
EXPLAIN ANALYZE
SELECT *
FROM core.transactions
WHERE transaction_date >= '2019-01-01'
AND transaction_date < '2019-02-01';
```

**Behavior Observed:**
- Parallel Sequential Scan on transactions
- ~4.4M rows scanned, ~118K rows returned
- No supporting index on transaction_date
- Rows removed by filter: ~4.39M
- Execution Time: ~11.39 seconds

<br>
<br>

**Query 2: User Transaction History** <BR>
Use case: Retrieve all transactions for a specific user <br>

```sql
EXPLAIN ANALYZE
SELECT *
FROM core.transactions
WHERE user_id = 12345;
```

**Behavior Observed:** <br>
- Parallel Sequential Scan on transactions
- ~4.4M rows scanned, 0 rows returned
- No supporting index on user_id
- Rows removed by filter: ~4.43M
- Execution Time: ~3.06 seconds

<br>
<br>

**Query 3: User Transaction History With Date Range** <br>
Use case: Retrieve transactions for a specific user within a defined date range. <br>

```sql
EXPLAIN ANALYZE
SELECT *
FROM core.transactions
WHERE user_id = 12345
  AND transaction_date >= '2019-01-01'
  AND transaction_date < '2019-02-01';
```


**Behavior Observed:** <br>
- Parallel Sequential Scan on transactions
- ~4.4M rows scanned, 0 rows returned
- No supporting index on user_id or transaction_date
- Highly selective filters (user_id + date range) applied only after full table scan
- Rows removed by filter: ~4.43M
- Execution Time: ~14.26 seconds

<br>
<br>

**Query 4: Aggregation (Finance / Analytics)** <br>
Aggregate transaction amounts by day for analytical reporting. <br>

```sql
EXPLAIN ANALYZE
SELECT
    DATE(transaction_date) AS tx_date,
    SUM(amount) AS total_amount
FROM core.transactions
GROUP BY DATE(transaction_date);
```


**Behavior Observed:** <br>
- Parallel Sequential Scan across entire table
- Partial Hash Aggregate followed by Final Group Aggregate
- In-memory sorting performed
- ~4.4M rows processed
- Result rows: 3,591 daily aggregates
- Execution Time: ~10.41 seconds

<br>
<br>

**Query 5: Error Monitoring** <br>
Identify transactions containing errors for operational monitoring. <br>

```sql
EXPLAIN ANALYZE
SELECT *
FROM core.transactions
WHERE errors IS NOT NULL;
```

**Behavior Observed:** <br>
- Parallel Sequential Scan on transactions
- ~4.4M rows scanned, ~211K rows returned
- No supporting index on nullable errors column
- Rows removed by filter: ~4.36M
- Execution Time: ~3.59 seconds

<BR>
<BR>
<BR>

### Phase 3. Targeted Performance Optimization
Baseline execution plans showed that all high-cost queries against core.transactions relied on parallel sequential scans, resulting in multi-second execution times caused by full-table reads on a high-volume dataset. <br>

Based on the observed query patterns, optimization efforts focused on high-impact, evidence-driven changes rather than speculative tuning.

<br>

#### Optimization Approach
The following principles guided all optimization decisions:
- Apply indexes only where query patterns justified them
- Prioritize high-selectivity access paths
- Use composite, expression, and partial indexes where appropriate
- Avoid redundant or premature optimization

<br>

#### Indexing Strategy
| Query Pattern            | Optimization Applied                          |
| :----------------------- | :-------------------------------------------- |
| Date-range filtering     | B-tree index on `transaction_date`            |
| User transaction lookups | Index on `user_id`                            |
| User + date filters      | Composite index `(user_id, transaction_date)` |
| Date-based aggregations  | Expression index on `DATE(transaction_date)`  |
| Error monitoring         | Partial index on `errors IS NOT NULL`         |

<br>

#### Index Creation
```sql
-- 1. Date-Range Index
CREATE INDEX idx_transactions_transaction_date
ON core.transactions (transaction_date);


-- 2. User Transaction Lookup Index 
CREATE INDEX idx_transactions_user_id
ON core.transactions (user_id);


-- 3. Composite Index (User + Date)
CREATE INDEX idx_transactions_user_date
ON core.transactions (user_id, transaction_date);


-- 4. Expression Index for Aggregations
CREATE INDEX idx_transactions_date_expression
ON core.transactions (DATE(transaction_date));


-- 5. Partial Index for Error Monitoring
CREATE INDEX idx_transactions_errors_not_null
ON core.transactions (errors)
WHERE errors IS NOT NULL;


--- After index creation, planner statistics were refreshed
ANALYZE core.transactions;
```


<br>

#### Post-Optimization Validation
All baseline queries were re-executed unchanged to ensure a direct, apples-to-apples comparison. <br>

Observed Execution Behavior & Performance: 

<br>

**Query 1: Date-Range Transaction Filtering** <br>
- Index Scan on idx_transactions_transaction_date
- Execution time reduced from ~11.4s to ~40ms (>99% improvement)

<br>

**Query 2: User Transaction History** <br>
- Bitmap Index Scan on idx_transactions_user_id
- Execution time reduced from ~3s to ~5ms

<br>

**Query 3: User Transaction History with Date Range** <br>
- Bitmap Index Scan using composite index
- Highly selective access path
- Sub-millisecond execution (~0.08ms)

<br>

**Query 4: Aggregation (Finance / Analytics)** <br>
- Parallel sequential scan retained
- Execution time ~11.7s
- Full-table aggregation required; index usage not beneficial for this workload

> This behavior is consistent with PostgreSQL’s execution model, where indexes optimize selective access rather than full-table analytical scans.

<br>

**Query 5: Error Monitoring** <br>
- Bitmap Index Scan on partial index
- Execution time ~5.0s
- Scan limited to ~211K error records instead of the full table

<br>

#### Performance Summary (Before vs After)
| Query Type       | Before | After   |
| :--------------- | :----- | :------ |
| Date range       | ~11s   | ~40ms   |
| User lookup      | ~3s    | ~5ms    |
| User + date      | ~3s    | ~0.08ms |
| Aggregation      | ~10s   | ~11.7s  |
| Error monitoring | ~3.6s  | ~5s     |


<br>

#### Engineering Takeaways
- Targeted indexing eliminated full-table scans for selective workloads
- Composite indexes produced the largest performance gains for real query patterns
- Aggregation workloads correctly remained sequential due to full data access requirements
- Partial indexing reduced scan scope for operational error monitoring
- All optimization decisions were driven by execution plan evidence rather than assumptions

<br>
<br>
<br>


### Phase 4. Scalable Data Architecture (Partitioning Strategy)
Following targeted indexing improvements in Phase 3, further optimization was required to address scan scope, scalability, and maintenance overhead on the high-volume `core.transactions` table.

<br>

#### 1. Workload & Access Pattern Summary
Analysis of executed queries and execution plans confirmed:
- core.transactions is the primary fact table (~13M+ rows)
- Workload is read-heavy, append-only
- Dominant filters are time-based (transaction_date)
- Queries heavily favor recent (hot) data
- Mixed OLTP + analytical access patterns <br>

This profile makes the table a strong candidate for time-based partitioning.

<br>

#### 2. Partitioning Design

| Design Aspect       | Decision            |
|:--------------------|:---------------------|
| Partitioning method | RANGE                |
| Partition key       | `transaction_date`   |
| Granularity         | Monthly              |

<br>

**Rationale**
- Transactions follow a natural time-series pattern
- Queries predominantly filter by date ranges
- Monthly partitions enable effective pruning without excessive object overhead
- Supports long-term archival and maintenance strategies

<br>

#### 3. Partition Creation

- Partition Table: <br>

A new partitioned table was created to mirror a real production migration pattern.

```sql
CREATE TABLE core.transactions_partition
(LIKE core.transactions INCLUDING DEFAULTS INCLUDING CONSTRAINTS)
PARTITION BY RANGE (transaction_date);
```

<br>

- Monthly Partition Creation

```sql
CREATE TABLE core.transactions_2019_01
PARTITION OF core.transactions_partition
FOR VALUES FROM ('2019-01-01') TO ('2019-02-01');

CREATE TABLE core.transactions_2019_02
PARTITION OF core.transactions_partition
FOR VALUES FROM ('2019-02-01') TO ('2019-03-01');

CREATE TABLE core.transactions_default
PARTITION OF core.transactions_partition
DEFAULT;
```

Partition boundaries follow PostgreSQL’s inclusive/exclusive range enforcement.

<br>

- Partition Structure Validation <br>
<img height="300" alt="Partition Validation" src="https://github.com/user-attachments/assets/92c20ab5-07eb-47dc-b93b-b9d005407d27" />

```sql
SELECT
    inhrelid::regclass AS partition_name,
    inhparent::regclass AS parent_table
FROM pg_inherits
WHERE inhparent = 'core.transactions_partition'::regclass;
```

<br>

#### 4. Partition-Level Indexing: <br>

To ensure efficient access within each partition, indexes were created explicitly at the partition level. <Br>

High-value indexes applied per partition:
- transaction_date
- user_id
- (user_id, transaction_date)
- Partial index on errors IS NOT NULL

```sql
-- 1. Transaction date
-- Jan 2019 
CREATE INDEX idx_transactions_2019_jan_date
ON core.transactions_2019_jan (transaction_date);

-- Feb 2019
CREATE INDEX idx_transactions_2019_feb_date
ON core.transactions_2019_feb (transaction_date);

-- Mar 2019 
CREATE INDEX idx_transactions_2019_mar_date
ON core.transactions_2019_mar (transaction_date);


-- 2. User ID
CREATE INDEX idx_transactions_2019_jan_user
ON core.transactions_2019_jan (user_id);

CREATE INDEX idx_transactions_2019_feb_user
ON core.transactions_2019_feb (user_id);

CREATE INDEX idx_transactions_2019_mar_user
ON core.transactions_2019_mar (user_id);


-- 3. (User + Date)
CREATE INDEX idx_transactions_2019_jan_user_date
ON core.transactions_2019_jan (user_id, transaction_date);

CREATE INDEX idx_transactions_2019_feb_user_date
ON core.transactions_2019_feb (user_id, transaction_date);

CREATE INDEX idx_transactions_2019_mar_user_date
ON core.transactions_2019_mar (user_id, transaction_date);



-- 4. Partial Index for Errors
CREATE INDEX idx_transactions_2019_jan_errors
ON core.transactions_2019_jan (errors)
WHERE errors IS NOT NULL;

CREATE INDEX idx_transactions_2019_feb_errors
ON core.transactions_2019_feb (errors)
WHERE errors IS NOT NULL;

CREATE INDEX idx_transactions_2019_mar_errors
ON core.transactions_2019_mar (errors)
WHERE errors IS NOT NULL;
```
<br>

<img height="300" alt="Verify Indexes Exist" src="https://github.com/user-attachments/assets/2a16a7c7-4d86-43c5-8819-0ca2444bed1c" />

- Index verification
```sql
SELECT
    tablename,
    indexname
FROM pg_indexes
WHERE schemaname = 'core'
AND tablename LIKE 'transactions_2019%';
```

<BR>

#### 5. Data Migration into Partitioned Table
To simulate a real production migration, existing data was loaded into the partitioned table after structure and indexes were in place.

```sql
INSERT INTO core.transactions_partition
SELECT *
FROM core.transactions;
```

PostgreSQL automatically routed rows into the appropriate monthly partitions based on transaction_date, with out-of-range records handled by the default partition. <br>

<br> 

- Verification
core.transactions_2019_jan image <br>
<img height="300" alt="jan partition" src="https://github.com/user-attachments/assets/13cb865c-bee6-4a97-a0ff-13e8cee3962d" /> <br>

```
SELECT * FROM core.transactions_partition;
SELECT * FROM core.transactions_2019_jan;
SELECT * FROM core.transactions_2019_feb;
SELECT * FROM core.transactions_2019_mar;
```
This step validates end-to-end partition routing behavior without modifying the original source table.


<br>

#### 6. Query Validation (Partition Pruning)
<img height="300" alt="Query Validation" src="https://github.com/user-attachments/assets/aca10bfb-43d6-425a-b473-07756aa57844" />

```SQL
EXPLAIN ANALYZE
SELECT *
FROM core.transactions_partition
WHERE transaction_date >= '2019-01-01'
  AND transaction_date < '2019-02-01'
  AND user_id = 12345;
```

**Observed behavior:**
- Partition pruning applied (January partition only)
- Composite index used
- Index scan selected
- Sub-millisecond execution time <BR>

This confirms correct interaction between partitioning and indexing.

<BR>

**Engineering Takeaways** <BR>
- Partitioning reduced scan scope beyond what indexing alone could achieve
- Monthly range partitions aligned cleanly with real query patterns
- Partition-level indexes preserved low-latency OLTP access
- Architecture supports scalable growth and operational maintenance
- Design mirrors real-world production migration practices


<br>
<br>
<br>

### Phase 5. Database Security & Access Control
Following the partitioned architecture introduced in Phase 4, this phase enforces production-grade data security through role-based access control, row-level protection, and auditable validation suitable for financial systems. <br>

The objective is to ensure access is intentional, minimal, and auditable.

<br>

#### 5.1. Security Architecture Overview
- Sensitive Asset Classification <br>

| Data Area           | Sensitivity | Rationale                           |
| ------------------- | ----------- | ----------------------------------- |
| `core.users` (PII)  | High        | Personally identifiable information |
| `core.cards` (PCI)  | High        | Cardholder data                     |
| `core.transactions` | High        | Financial activity                  |
| Error metadata      | Medium      | Fraud & operational signals         |
| Aggregated metrics  | Low         | Non-identifiable analytics          |

Sensitive tables require explicit, role-based access.

<br>

- Role Model (Design-Level)

| Role                 | Purpose                                         |
| -------------------- | ----------------------------------------------- |
| `db_admin`           | Administrative control (DDL, grants, ownership) |
| `etl_service`        | Data ingestion and transformation               |
| `analytics_engineer` | Modeling, optimization, controlled writes       |
| `analyst_readonly`   | Read-only analytics                             |
| `auditor`            | Compliance and exception review                 |

Roles represent job functions rather than users; no role has unrestricted daily access.

<br>

**Security Principles Enforced**
- Least privilege
- Separation of duties
- Defense in depth (schema, table, row)
- Secure by default

<br>
<br>

#### 5.2. Role-Based Access Control (RBAC)

```SQL

-- Administrative role
CREATE ROLE db_admin NOLOGIN;

-- ETL role
CREATE ROLE etl_service NOLOGIN;

-- Analytics engineering role
CREATE ROLE analytics_engineer NOLOGIN;

-- Read-only analyst role
CREATE ROLE analyst_readonly NOLOGIN;

-- Audit / compliance role
CREATE ROLE auditor NOLOGIN;
```

<br>
<br>

#### 5.3 Schema-Level Privileges

```sql
-- Admin
GRANT ALL PRIVILEGES ON SCHEMA staging, core TO db_admin;

-- ETL
GRANT USAGE ON SCHEMA staging TO etl_service;
GRANT INSERT, UPDATE, DELETE, SELECT ON ALL TABLES IN SCHEMA staging TO etl_service;

-- Analytics Engineer
GRANT USAGE, CREATE ON SCHEMA core TO analytics_engineer;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA core TO analytics_engineer;

-- Read-only Analyst
GRANT USAGE ON SCHEMA core TO analyst_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA core TO analyst_readonly;

-- Auditor
GRANT USAGE ON SCHEMA core TO auditor;
GRANT SELECT ON core.transactions TO auditor;
```

<br>

- Default Privileges (Future-Proofing)

```sql
-- Core schema: tables created by analytics_engineer
ALTER DEFAULT PRIVILEGES IN SCHEMA core
GRANT SELECT ON TABLES TO analyst_readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA core
GRANT SELECT ON TABLES TO auditor;

-- Staging schema: tables created by etl_service
ALTER DEFAULT PRIVILEGES IN SCHEMA staging
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO etl_service;
```

Ensures newly created tables inherit the intended access model without requiring manual re-granting.

<br>
<br>

#### 5.4. PUBLIC Access Revocation

```sql
-- Revoke schema & table access from PUBLIC
REVOKE ALL ON SCHEMA core, staging FROM PUBLIC;
REVOKE ALL ON ALL TABLES IN SCHEMA core, staging FROM PUBLIC;

-- Revoke future table access from PUBLIC
ALTER DEFAULT PRIVILEGES IN SCHEMA core
REVOKE ALL ON TABLES FROM PUBLIC;

ALTER DEFAULT PRIVILEGES IN SCHEMA staging
REVOKE ALL ON TABLES FROM PUBLIC;
```


<br>
<br>

#### 5.5. Row-Level Security (RLS)

- Enable RLS on Transactions Table
```sql
ALTER TABLE core.transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE core.transactions FORCE ROW LEVEL SECURITY;
```

<br>

- Create RLS Policies
```sql
-- Analytics Engineer: full visibility
CREATE POLICY analytics_engineer_full_access
ON core.transactions
FOR SELECT
TO analytics_engineer
USING (true);

-- Analyst: clean transactions only
CREATE POLICY analyst_readonly_filtered_access
ON core.transactions
FOR SELECT
TO analyst_readonly
USING (errors IS NULL);

-- Auditor: error records only
CREATE POLICY auditor_error_access
ON core.transactions
FOR SELECT
TO auditor
USING (errors IS NOT NULL);
```

<br>
<br>

#### 5.6. RLS on Partitioned Table

```sql
-- Enable RLS on the partitioned table
ALTER TABLE core.transactions_partition ENABLE ROW LEVEL SECURITY;

--  Analytics Engineer (full access)
CREATE POLICY analytics_full_access_partition
ON core.transactions_partition
FOR ALL
TO analytics_engineer
USING (true);

-- Analyst (no errors)
CREATE POLICY analyst_no_errors_partition
ON core.transactions_partition
FOR SELECT
TO analyst_readonly
USING (errors IS NULL);

-- Auditor (errors only)
CREATE POLICY auditor_errors_only_partition
ON core.transactions_partition
FOR SELECT
TO auditor
USING (errors IS NOT NULL);
```

<br>
<br>

#### 5.7. Security Validation & Audit Evidence

**RBAC Validation Test (Analyst Role)** <br>
<img height="300" alt="RBAC Validation_analyst" src="https://github.com/user-attachments/assets/1f7c204a-25b8-4ae5-abd0-286581a19951" />

```sql
SET ROLE analyst_readonly;

INSERT INTO core.transactions_partition (transaction_id, transaction_date)
    VALUES (999999, now());
-- Output: Permission denied

SELECT * FROM staging.transactions;
-- Output: Permission denied

RESET ROLE;
```

<br>

**RLS Validation** <br>

- Analyst Role <br>

<img height="300" alt="RLS_analyst_count" src="https://github.com/user-attachments/assets/e3f10857-eef9-4978-bb8f-1976319d36e6" />

```sql
SET ROLE analyst_readonly;

SELECT COUNT(*) FROM core.transactions_partition;
-- Output: 13094522

SELECT COUNT(*) FROM core.transactions_partition WHERE errors IS NOT NULL;
-- Output: 0

RESET ROLE;
```

<br>

- Auditor Role <br>
<img height="300" alt="RLS_auditor_count" src="https://github.com/user-attachments/assets/8e2d2c09-d03e-480e-a38c-88a0ce08130b" />

```sql
SET ROLE auditor;

SELECT COUNT(*) FROM core.transactions_partition;
-- Output: 211393

RESET ROLE;
```

<br>

- Analytic Engineer Role <br>
<img height="300" alt="RLS_Analytics Engineer_count" src="https://github.com/user-attachments/assets/01d73fd0-9a86-48c4-b783-1b4642422cdf" />

```sql
SET ROLE analytics_engineer;

SELECT COUNT(*) FROM core.transactions_partition;
-- Output: 13305915 (Full dataset)

RESET ROLE;
```

<br>
<br>

#### 5.8. Partition + Security Compatibility
<img height="300" alt="Partition + Security Compatibility" src="https://github.com/user-attachments/assets/3fbcc503-6fb7-4039-970e-70c7aece0856" />

```sql
SET ROLE analytics_engineer;

EXPLAIN ANALYZE
SELECT *
FROM core.transactions_partition
WHERE transaction_date >= '2019-01-01'
  AND transaction_date < '2019-02-01';
```

Observed:
- Partition pruning 
- RLS enforced
- No performance regression

<br>
<br>

#### 5.9. Policy & PUBLIC Audit
```sql
-- Active policies
SELECT policyname, roles, qual
FROM pg_policies
WHERE schemaname = 'core';

-- Public privilege
SELECT DISTINCT grantee, table_schema
FROM information_schema.role_table_grants
WHERE grantee = 'PUBLIC';
```

> PUBLIC access is limited to system schemas only (pg_catalog, information_schema).

<br>
<br>

#### Engineering Summary
- Role-based access enforced at schema and table levels
- Row-level security aligned with business intent
- Partitioned tables secured without breaking performance
- No PUBLIC exposure of business data
- Security validated through role simulation and audit queries <br>

This phase demonstrates production-grade PostgreSQL governance suitable for regulated, high-volume financial environments.


<br>
<br>

## Final Notes

This case study focuses on performance and security engineering in an inherited PostgreSQL system. <BR>
Schema design, data modeling, and application-layer concerns were intentionally treated as pre-existing to reflect real-world database optimization work.
