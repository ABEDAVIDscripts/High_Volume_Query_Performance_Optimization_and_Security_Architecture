## High-Volume Query Performance Optimization and Surity Architecture

<br>

### Project Overview

This production case study simulates a real-world PostgreSQL system operating at scale, where a high-volume relational database already exists in production and the engineer’s responsibility is to:

- Diagnose query performance bottlenecks
- Apply targeted, evidence-based SQL optimizations
- Design and enforce enterprise-grade database security
- Validate improvements using execution plans and role-based testing <br>

Rather than designing a schema from scratch, this project intentionally begins with performance and security debt, reflecting how databases evolve in real organizations.


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
<img  height="350" alt="Work Flow" src="https://github.com/user-attachments/assets/bf72d312-c2af-4c31-aa13-72bf1f0e72ce" />

<br>
<br>
<br>

## Implementation Phases
### Environment Context (Assumed Baseline)

This case study assumes an operational PostgreSQL environment with existing schemas and production data already loaded. Database setup and connectivity are treated as baseline operational knowledge and are intentionally excluded from the core documentation. <br>

The project begins at the point where real-world database engineering work typically starts: improving an inherited system under load.

<br>
<br>

<br>

### Phase 1. Inherited Schema & Baseline State

#### Context
The database contains normalized production tables supporting financial operations:
- core.users: customer records (PII)
- core.cards: payment card data (PCI-sensitive)
- core.transactions — high-volume fact table (13M+ rows)

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
This phase defines the performance baseline and justifies subsequent optimization decisions. The goal was to:
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

Result:
```
"table_name"	    "total_size"
"transactions"	    "1654 MB"
"transactions"	    "1503 MB"
"cards"		        "920 kB"
"cards"		        "736 kB"
"users"		        "368 kB"
"users"		        "280 kB"
```

The output below shows table sizes from both the staging and core schemas. <BR>
The transactions table is significantly larger than all other tables, exceeding 1.5GB in size. This confirms it as the primary source of query latency and the main focus for baseline performance benchmarking and subsequent optimization.

<br>

#### 2. Identification of Realistic Query Patterns
Before executing benchmarks, common query patterns were identified based on realistic organizational usage rather than synthetic tests. <br>
These patterns represent how analysts, finance teams, operations, and support teams would interact with the data in a production environment. <br>

**Identified Query Categories:**
| Query Category       | Typical Use Case                |
| :------------------- | :------------------------------ |
| Date-range filtering | Monthly and daily reports       |
| User-based lookups   | Customer support investigations |
| Aggregations         | Financial and trend analysis    |
| Error monitoring     | Operational and fraud checks    |

<br>

#### 3. Measure Execution Plans
**Query 1: Date-Range Transaction Filtering** <br>
Use case: Retrieve transactions within a specific date range. <br>

```sql
EXPLAIN ANALYZE
SELECT *
FROM core.transactions
WHERE transaction_date >= '2019-01-01'
AND transaction_date < '2019-02-01';
```

**Observed Execution Behavior & Performance:**
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

**Observed Execution Behavior & Performance:** <br>
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


**Observed Execution Behavior & Performance** <br>
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


**Observed Execution Behavior & Performance:** <br>
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

**Observed Execution Behavior & Performance:** <br>
- Parallel Sequential Scan on transactions
- ~4.4M rows scanned, ~211K rows returned
- No supporting index on nullable errors column
- Rows removed by filter: ~4.36M
- Execution Time: ~3.59 seconds

<BR>
<BR>
<BR>

### Phase 3. Targeted Performance Optimization

Optimization Strategy

Baseline analysis showed all high-cost queries relied on Parallel Sequential Scans over core.transactions, resulting in multi-second execution times due to full-table reads.
