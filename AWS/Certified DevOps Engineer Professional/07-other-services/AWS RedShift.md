
Amazon Redshift is a fully managed **data warehouse** — a relational database optimised for analytical queries (OLAP) rather than transactional workloads (OLTP). It stores data in columnar format and is designed for running complex queries across very large datasets.

---

## Architecture

```
Leader node
  Receives queries from clients
  Builds query execution plan
  Coordinates parallel execution
        ↓
Compute nodes (1 to 128)
  Execute query slices in parallel
  Store actual data in columnar format
  Return results to leader node
        ↓
Leader node aggregates and returns results
```

Each compute node is divided into **slices** — parallel processing units. Data is distributed across slices for parallel execution.

---

## Node Types

|Type|Use for|
|---|---|
|RA3|Managed storage — scales compute and storage independently. Recommended|
|DC2|Dense compute — SSD storage, fixed compute/storage ratio. Legacy|

**RA3 is the modern choice** — you scale compute independently of storage. Data stored in S3-backed Redshift Managed Storage (RMS).

---

## Redshift Spectrum

Query data directly in S3 without loading it into Redshift:

```
Redshift cluster
        ↓
CREATE EXTERNAL SCHEMA pointing at S3
        ↓
Query runs — Redshift pushes computation to
Spectrum layer (separate from compute nodes)
        ↓
Spectrum reads S3 files in parallel
        ↓
Results returned to Redshift for final aggregation
```

**Supported formats:** Parquet, ORC, JSON, CSV, Avro, Ion, TSV

**Best for:** Historical data, infrequent queries, petabyte-scale data that would be expensive to load

**Key advantage:** Data stays in S3 — no storage cost in Redshift, no loading time

---

## Distribution Styles

How data is distributed across compute node slices — critical for query performance:

|Style|How it works|Use when|
|---|---|---|
|AUTO|Redshift chooses automatically|Default — let Redshift decide|
|EVEN|Rows distributed round-robin|No clear join/filter column|
|KEY|Rows with same key value go to same slice|Frequently joined on this column|
|ALL|Full copy of table on every node|Small dimension tables joined to large fact tables|

Choosing the right distribution key reduces data movement between nodes during joins — the biggest performance lever.

---

## Sort Keys

Define the order rows are stored on disk — allows Redshift to skip blocks that don't match query filters:

|Type|How it works|Use when|
|---|---|---|
|Compound|Sorts by columns in order listed|Most common — filter on leading columns|
|Interleaved|Equal weight to all sort key columns|Filter on any column independently|

---

## Loading Data

**COPY command** — primary method for bulk loading:

sql

```sql
COPY my_table
FROM 's3://my-bucket/data/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
FORMAT AS PARQUET;
```

- Loads in parallel across all compute nodes
- Supports S3, DynamoDB, EMR, SSH
- Much faster than INSERT statements
- Compresses automatically

**INSERT** — for small row counts only — never for bulk loading

---

## Unloading Data

**UNLOAD command** — export query results to S3:

sql

```sql
UNLOAD ('SELECT * FROM sales WHERE year = 2025')
TO 's3://my-bucket/export/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
FORMAT AS PARQUET;
```

---

## Redshift Serverless

No cluster to manage — Redshift automatically provisions and scales capacity:

- Pay per compute second used
- Scales to zero when idle
- Good for intermittent workloads
- No node type selection needed

---

## Security

**Encryption:**

- At rest — AES-256, KMS or CloudHSM
- In transit — SSL

**Network:**

- Deploy in VPC
- Enhanced VPC Routing — forces all COPY/UNLOAD traffic through VPC (not internet)
- VPC endpoint for private access

**Access control:**

- IAM for cluster management
- Database users/groups for data access
- Row-level security available

---

## Key Integrations

|Service|Integration|
|---|---|
|S3|COPY to load, UNLOAD to export, Spectrum to query in place|
|Glue Data Catalog|Spectrum uses Glue catalog for external table metadata|
|QuickSight|Direct connector for dashboards|
|Kinesis Firehose|Stream data directly into Redshift|
|DMS|Migrate existing databases to Redshift|
|Lambda|Custom processing via UDFs|

---

## Redshift vs Other Services

|Scenario|Service|
|---|---|
|Complex analytics on structured data at scale|Redshift|
|Ad-hoc SQL on S3 files, serverless|Athena|
|Real-time search and dashboards|OpenSearch|
|Simple key-value or document queries|DynamoDB|
|OLTP transactional workload|RDS / Aurora|
|Large-scale ETL and ML|EMR|
|Historical petabyte data, infrequent queries|Redshift Spectrum|

---

## Exam Traps

|Wrong assumption|Reality|
|---|---|
|Use COPY for petabyte historical data|Use Spectrum — COPY stores data twice|
|Redshift has Read Replicas|NO — no Read Replicas, no Global Database|
|Redshift cross-region replication|Snapshot copy only — not continuous|
|INSERT for bulk loading|Always use COPY — INSERT is slow|
|Redshift Spectrum needs data in Redshift|NO — queries S3 directly|
|CFN Mappings for dynamic passwords|NO — use Parameters section|
|Redshift is good for OLTP|NO — designed for OLAP analytical queries|
|Enhanced VPC Routing optional for security|Enable it to prevent S3 traffic going over internet|