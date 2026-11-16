# rds-audit-lake Design

Status: Draft (v1 scope)

## 1. Overview

rds-audit-lake collects database audit logs from Amazon RDS and Amazon Aurora
and stores them in Amazon S3 so that they can be queried with Amazon Athena.
It runs as an AWS Lambda function on a schedule and pulls rotated audit log
files through the RDS API.

The project targets the following use case:

- Audit logs must be kept for a long time
- Audit logs are queried rarely (incident investigation, compliance requests)
- Publishing audit logs to CloudWatch Logs is too expensive for the volume

### 1.1 Goals

- Collect every rotated audit log file exactly once into S3, with no gaps
- Keep the original log lines unmodified
- Make the stored data queryable with Athena without extra infrastructure
- Define a stable, versioned S3 layout (the raw layout) that other tools can
  consume without depending on this codebase
- Run with minimal operational overhead (serverless, no always-on compute)

### 1.2 Non-goals (v1)

- Converting logs to Parquet or Apache Iceberg (planned as a separate project,
  see [Section 13](#13-future-work))
- Real-time delivery or alerting
- Full parsing of every field of every record
- PostgreSQL (`pgaudit`) support
- Publishing to CloudWatch Logs

## 2. Supported sources (v1)

| Engine | Audit feature | Enabled by |
| --- | --- | --- |
| Aurora MySQL | Advanced Auditing | DB cluster parameter group (`server_audit_logging`, `server_audit_events`) |
| RDS for MySQL | MariaDB Audit Plugin | Option group (`MARIADB_AUDIT_PLUGIN`) |

### 2.1 Aurora MySQL (Advanced Auditing)

Facts from the AWS documentation ([Aurora MySQL Advanced Auditing][aurora-audit]):

- Log files are named `audit/audit.log.<other identifying information>`
- Each DB instance stores its audit log on local storage, separately from other
  instances in the cluster
- Each instance writes to four log files at a time
- A log file rotates when it reaches 100 MB (not configurable)
- Entries are not in sequential order across files
- Records are CSV in UTF-8, and queries are wrapped in single quotes
- Field order: `timestamp, serverhost, username, host, connectionid, queryid,
  operation, database, object, retcode`
- `timestamp` is a Unix timestamp with microsecond precision

### 2.2 RDS for MySQL (MariaDB Audit Plugin)

Facts from the AWS documentation ([MariaDB Audit Plugin][mysql-audit]):

- Log files are stored under `/rdsdbdata/log/audit/`
- Rotation is controlled by `SERVER_AUDIT_FILE_ROTATE_SIZE` (default 1,000,000
  bytes) and `SERVER_AUDIT_FILE_ROTATIONS` (default 9)
- `SERVER_AUDIT_QUERY_LOG_LIMIT` truncates long query strings (default 1024)
- Records are CSV in UTF-8
- Field order: the same ten fields as Aurora MySQL, followed by
  `connection_type`
- `timestamp` is `YYYYMMDD HH:MI:SS` (24-hour clock, second precision)

### 2.3 Differences that affect the design

| Aspect | Aurora MySQL | RDS for MySQL |
| --- | --- | --- |
| Topology | Cluster with writer and readers, each with its own logs | Single instance |
| Concurrent files | Four per instance | One active file |
| Timestamp | Unix microseconds | `YYYYMMDD HH:MI:SS`, time zone to be verified |
| Extra fields | None | `connection_type` |
| Retention on the instance | Bounded by local storage | `SERVER_AUDIT_FILE_ROTATIONS` files |

Logs that are not collected before the instance deletes them are lost.
Collection latency must stay well below the retention window on the instance.

## 3. Architecture

```text
 EventBridge Scheduler (rate, per source)
          |
          v
 +-------------------------+        +---------------------------+
 | rds-audit-lake (Lambda) | -----> | DynamoDB: checkpoints     |
 | mode: collect           | <----- | (one item per instance)   |
 +-------------------------+        +---------------------------+
   |            |        |
   |            |        +--------> CloudWatch metrics
   |            v
   |   RDS API: DescribeDBClusters / DescribeDBInstances
   |            DescribeDBLogFiles
   |            downloadCompleteLogFile (REST, SigV4)
   v
 +--------------------------------------------------+
 | S3: raw layout v1 (zstd)                         |
 | raw/v1/source=<id>/dt=YYYY-MM-DD/hour=HH/...     |
 +--------------------------------------------------+
          |
          v
 Glue table (partition projection) --> Athena
```

One function invocation handles one source. A source is either an Aurora
cluster or a single RDS instance. Each source has its own schedule, so a slow
or failing source does not block the others.

### 3.1 Components

| Component | Responsibility |
| --- | --- |
| Scheduler | Invoke the function periodically with the source identifier |
| Discovery | Resolve the source to the list of DB instances to collect from |
| Lister | List rotated audit log files that are complete and not yet collected |
| Downloader | Download the full content of a log file |
| Splitter | Split a file into records and assign each record to an hour bucket |
| Raw writer | Compress each hour bucket with zstd and write it to S3 |
| Checkpoint store | Persist which files have been collected |
| Metrics | Emit collection lag and failure metrics |

The splitter reads only the timestamp of each record. Full parsing of every
field belongs to the conversion project.

### 3.2 Code layout

```text
cmd/rds-audit-lake/      # Lambda entry point and local CLI
internal/
  source/                # discovery for Aurora clusters and RDS instances
  logfile/               # listing, file selection, download
  record/                # record splitting and timestamp parsing per engine
  rawstore/              # raw layout v1 key builder and S3 writer
  checkpoint/            # DynamoDB checkpoint store
  collect/               # orchestration
  metrics/               # CloudWatch metrics
examples/
  terraform/             # reference deployment (not a published module)
  athena/                # table definitions and sample queries
docs/
```

AWS calls are behind small interfaces so that the splitter and the raw writer
can be tested locally without AWS.

## 4. Collection algorithm

### 4.1 Discovery

- Aurora MySQL: call `DescribeDBClusters` and collect from every member
  instance on every run, so added or removed readers are handled automatically
- RDS for MySQL: call `DescribeDBInstances` for the configured instance

Instances are processed concurrently with a configurable limit.

### 4.2 File selection

For each instance:

1. Call `DescribeDBLogFiles` with a filename filter for audit logs
2. Exclude files that may still be written to
   - Aurora MySQL: the newest file of each of the concurrent write streams
   - RDS for MySQL: the active (unrotated) file
3. Exclude files already recorded in the checkpoint
4. Sort the remaining files by `LastWritten`, oldest first

Excluding active files delays collection by up to one rotation. This is the
cost of never storing a partially written file.

How to identify the streams of an Aurora instance from the file names must be
confirmed against real file listings (see [Section 14](#14-open-questions)).

### 4.3 Download

Files are downloaded with the `downloadCompleteLogFile` REST endpoint
([Reading log file contents using REST][download-rest]):

```text
GET /v19/downloadCompleteLogFile/{DBInstanceIdentifier}/{LogFileName}
host: rds.{region}.amazonaws.com
```

- Requests are signed with SigV4 for the `rds` service
- The `/v19/` path is required. The `/v13/` path stops working after
  March 1, 2027
- After a download, the file listing is fetched again. If the file's
  `LastWritten` or size changed during the download, the download is retried

### 4.4 Splitting

1. Read the content line by line
2. A line that starts with a valid timestamp for the engine starts a new record
3. A line that does not start with a valid timestamp is appended to the
   previous record, because a query can contain line breaks
4. Assign each record to the UTC hour of its timestamp
5. Collect all records of the same hour from the same file into one bucket,
   even if they are not contiguous in the file

Rule 3 is a working assumption. How each engine writes line breaks inside
queries must be verified (see [Section 14](#14-open-questions)).

A line that cannot be assigned to any record (for example, a first line
without a timestamp) is not dropped. It is written to the quarantine prefix
with the source file name and line number, and a failure metric is emitted.

### 4.5 Write and checkpoint

For each collected file:

1. Write one S3 object per hour bucket (see [Section 5](#5-raw-layout-v1))
2. After every object of the file is written, record the file in the
   checkpoint

The checkpoint is updated only after all objects are written. A crash between
the two steps causes the file to be collected again on the next run. Because
object keys are deterministic, the second run overwrites the same objects with
the same content. Delivery is at-least-once at the file level and
effectively exactly-once at the object level.

### 4.6 Correctness requirements

The following failure modes are easy to introduce in a log collector of this
kind. Each one is a requirement with a dedicated test.

| ID | Failure mode | Requirement | Test |
| --- | --- | --- | --- |
| R1 | The deprecated `/v13/` REST path stops working after March 1, 2027, and every download fails | Use `/v19/downloadCompleteLogFile`. Build the path in one place | Unit test asserts the request path. CI rejects the string `/v13/` in non-test code |
| R2 | Records of one file are not in time order near an hour boundary (hour A, B, A). Grouping by consecutive runs creates two buckets for hour A with the same key, and the second upload overwrites the first | Group by hour across the whole file, so one file produces at most one object per hour (Section 4.4, rule 5) | Unit test with interleaved hours asserts one object per hour and the total record count |
| R3 | A query contains a line break, the continuation line has no timestamp, the whole file fails to parse, and the file is skipped and marked as collected | Treat a line without a timestamp as a continuation of the previous record. Never mark a file as collected unless all of its lines are written to the raw layout or to quarantine | Unit test with multi-line queries. Unit test asserts the checkpoint is unchanged when any write fails |
| R4 | Two keys collide, so one object overwrites another with different content | The key is unique per (instance, source file, hour) | Property test: distinct inputs never produce the same key |

A general invariant covers R2 to R4: for every collected file, the number of
lines in the file equals the number of lines written to the raw layout plus the
number written to quarantine. The collector computes both counts and fails the
file when they differ.

## 5. Raw layout v1

The raw layout is the contract between rds-audit-lake and any downstream tool.
Changes that break it require a new version prefix (`raw/v2/`).

### 5.1 Key format

```text
{prefix}/raw/v1/source={source_id}/dt={YYYY-MM-DD}/hour={HH}/{instance_id}/{last_written_ms}_{file_name}_{hash8}.log.zst
```

| Element | Value |
| --- | --- |
| `prefix` | Optional, configurable |
| `source_id` | Aurora cluster identifier or RDS instance identifier |
| `dt`, `hour` | UTC date and hour of the records in the object |
| `instance_id` | DB instance that produced the file |
| `last_written_ms` | `LastWritten` of the source file (epoch milliseconds) |
| `file_name` | Source file name with `/` replaced by `_` |
| `hash8` | First 8 hex characters of SHA-256 of the instance ID and file name |

The key depends only on the source file and the hour, so writing the same
file twice produces the same keys.

### 5.2 Object content

- Records exactly as written by the engine, separated by `\n`
- Every record in the object has a timestamp within `dt` and `hour` (UTC)
- Records keep their order from the source file
- The whole object is compressed with zstd

### 5.3 Object metadata

| Metadata | Value |
| --- | --- |
| `Content-Type` | `application/zstd` |
| `x-amz-meta-engine` | `aurora-mysql` or `mysql` |
| `x-amz-meta-source-file` | Original log file name |
| `x-amz-meta-source-last-written` | `LastWritten` of the source file |
| `x-amz-meta-record-count` | Number of records in the object |
| `x-amz-meta-collected-at` | Collection time (RFC 3339, UTC) |
| `x-amz-meta-layout-version` | `1` |

### 5.4 Quarantine

```text
{prefix}/quarantine/v1/source={source_id}/dt={collection date}/{instance_id}/{file key}.jsonl.zst
```

Each line holds the source file name, the line number, the reason, and the
raw line.

## 6. Checkpoint store

DynamoDB table with one item per instance.

| Attribute | Type | Description |
| --- | --- | --- |
| `pk` | S | `{source_id}#{instance_id}` |
| `watermark_ms` | N | All files with `LastWritten` below this value are collected |
| `collected` | M | File identities collected at or above the watermark |
| `updated_at_ms` | N | Last update time |

- Several files can share the same `LastWritten`, so a timestamp alone cannot
  track progress. The `collected` map holds the identities near the watermark
- The watermark advances when every file below it is collected, and older
  entries are removed from `collected` to stay under the 400 KB item limit
- Updates use a conditional write on `updated_at_ms` so that two concurrent
  invocations cannot overwrite each other's progress

The identity of a file is its name for Aurora MySQL. For RDS for MySQL,
rotated files may be renamed, so the identity may need to include
`LastWritten` and size (see [Section 14](#14-open-questions)).

## 7. Error handling and retries

| Error | Handling |
| --- | --- |
| Throttling, HTTP 5xx, timeouts | Retry with exponential backoff and jitter |
| File rotated during download | Retry the file from listing |
| HTTP 404 on download | Retry a limited number of times, then report the file as missing |
| HTTP 4xx other than 404, 408 and 429 | Fail immediately (permission or request error) |
| Record without a timestamp | Write to quarantine and continue |
| S3 write failure | Fail the file and leave the checkpoint unchanged |

A failed file is never recorded as collected. The function returns an error
when any file fails, so that the failure is visible in Lambda metrics and the
scheduler's retry policy applies.

## 8. Configuration

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `RAL_ENGINE` | Yes | - | `aurora-mysql` or `mysql` |
| `RAL_SOURCE_ID` | Yes | - | Cluster identifier or instance identifier |
| `RAL_BUCKET` | Yes | - | Destination bucket |
| `RAL_PREFIX` | No | empty | Key prefix |
| `RAL_CHECKPOINT_TABLE` | Yes | - | DynamoDB table name |
| `RAL_CONCURRENCY` | No | `4` | Instances processed in parallel |
| `RAL_TIME_BUDGET` | No | Lambda timeout minus 60s | Stop starting new files after this |
| `RAL_LOG_LEVEL` | No | `info` | `debug`, `info`, `warn`, `error` |

The source can also be passed in the invocation payload, so one function can
serve several sources with different schedules.

## 9. IAM permissions

| Action | Resource |
| --- | --- |
| `rds:DescribeDBClusters` | The target cluster (Aurora only) |
| `rds:DescribeDBInstances` | The target instances |
| `rds:DescribeDBLogFiles` | The target instances |
| `rds:DownloadCompleteDBLogFile` | The target instances |
| `s3:PutObject` | `{bucket}/{prefix}/raw/v1/*`, `{bucket}/{prefix}/quarantine/v1/*` |
| `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem` | The checkpoint table |
| `cloudwatch:PutMetricData` | Restricted by the `cloudwatch:namespace` condition key |
| `kms:GenerateDataKey` | Only when SSE-KMS is used |

The function does not need `s3:GetObject` or `s3:DeleteObject`.

## 10. Observability

### 10.1 Metrics

| Metric | Meaning |
| --- | --- |
| `CheckpointLagSeconds` | Now minus the newest collected `LastWritten`, per instance |
| `OldestPendingAgeSeconds` | Age of the oldest file waiting to be collected |
| `FilesCollected` | Files collected in the run |
| `FilesFailed` | Files that failed in the run |
| `QuarantinedLines` | Lines written to quarantine |
| `BytesWritten` | Compressed bytes written to S3 |

An alarm on `OldestPendingAgeSeconds` detects the risk of losing logs before
the instance deletes them.

### 10.2 Logs

The function logs file names, counts and durations as structured JSON. It
never logs record content, because records contain SQL text, user names and
client addresses.

## 11. Querying with Athena

`examples/athena/` provides a table over the raw layout with partition
projection, so new partitions need no registration.

- `dt` uses date projection and `hour` uses integer projection
- `source` uses enum projection with the configured source identifiers, or
  injected projection when queries always specify a source
- Each line is read as a single string column first, and fields are extracted
  with SQL functions. A CSV SerDe is not used until its handling of quoted
  queries containing commas, quotes and line breaks is verified

The raw layout does not depend on the Athena table definition, so the table
can be redefined at any time without rewriting data.

## 12. Security

- Encryption: SSE-S3 by default, SSE-KMS optional
- Bucket: Block Public Access and `BucketOwnerEnforced` object ownership
- Integrity: S3 Object Lock and versioning are optional, and are configured on
  the bucket by the user, not by the function
- Least privilege: the permissions in [Section 9](#9-iam-permissions) only
- Data handling: no record content in logs, metrics or error messages

## 13. Future work

- **rds-audit-lake-iceberg**: a separate, experimental project that reads the
  raw layout and writes Apache Iceberg tables (Amazon S3 Tables). It depends
  only on [Section 5](#5-raw-layout-v1)
- **PostgreSQL**: `pgaudit` output through the PostgreSQL log, whose format
  depends on `log_line_prefix`
- **Cost benchmark**: compare raw zstd, Parquet on S3 and Iceberg on S3 Tables
  with the same data and queries

## 14. Open questions

| Question | How to resolve |
| --- | --- |
| How to identify the four concurrent streams of an Aurora instance from file names | Inspect `DescribeDBLogFiles` output |
| Whether RDS for MySQL renames rotated files (`server_audit.log.1` to `.2`) | Observe file names and `LastWritten` across rotations |
| Time zone of the RDS for MySQL `timestamp` field | Compare records with known event times |
| How each engine writes line breaks, commas and quotes inside queries | Run test queries and inspect the logs |
| Whether `downloadCompleteLogFile` returns a consistent snapshot of a rotated file | Compare size and content across downloads |
| Typical file size and files per hour on busy instances | Measure in a test environment |

## 15. Distribution

- GitHub Releases: `bootstrap` for `linux/arm64` packaged as a zip, with
  checksums and build provenance
- `examples/terraform/`: a reference deployment showing every resource. It can
  be used as a module but is not versioned or published as one
- Documentation in English

## 16. References

- [Using Advanced Auditing with an Amazon Aurora MySQL DB cluster][aurora-audit]
- [MariaDB Audit Plugin support for MySQL][mysql-audit]
- [Reading log file contents using REST][download-rest]
- [Amazon Athena partition projection][athena-projection]

[aurora-audit]: https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Auditing.html
[mysql-audit]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.MySQL.Options.AuditPlugin.html
[download-rest]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/DownloadCompleteDBLogFile.html
[athena-projection]: https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html
