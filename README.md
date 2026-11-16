# rds-audit-lake

Collect audit logs from Amazon RDS and Amazon Aurora into Amazon S3, and
query them with Amazon Athena.

> **Status:** Design phase. No release is available yet. See
> [docs/design.md](docs/design.md).

## Why

Database audit logs are usually kept for a long time and queried rarely, for
example during an incident investigation or a compliance request. Publishing
them to CloudWatch Logs is often the most expensive part of keeping them.

rds-audit-lake pulls rotated audit log files through the RDS API and writes
them to S3 as compressed objects in a documented layout. You pay for S3
storage and for the Athena queries you run, with no always-on compute.

## Features (v1)

- Supports Aurora MySQL (Advanced Auditing) and RDS for MySQL (MariaDB Audit
  Plugin)
- Collects from every instance in an Aurora cluster, including readers added
  later
- Collects only rotated files, so a partially written file is never stored
- Stores the original log lines unmodified, compressed with zstd
- Uses deterministic S3 keys, so retries never create duplicates
- Never drops a line: lines that cannot be assigned to a record go to a
  quarantine prefix
- Publishes a versioned [raw layout](docs/design.md#5-raw-layout-v1) that other
  tools can consume
- Ships Athena table definitions with partition projection

## Architecture

```text
EventBridge Scheduler
        |
        v
rds-audit-lake (Lambda) ---> DynamoDB (checkpoints)
        |
        |  RDS API: DescribeDBLogFiles, downloadCompleteLogFile
        v
S3: raw/v1/source=<id>/dt=YYYY-MM-DD/hour=HH/*.log.zst
        |
        v
Athena
```

See [docs/design.md](docs/design.md) for the collection algorithm,
checkpoints, error handling and the raw layout specification.

## Supported sources

| Engine | Audit feature | Status |
| --- | --- | --- |
| Aurora MySQL | Advanced Auditing | Planned for v1 |
| RDS for MySQL | MariaDB Audit Plugin | Planned for v1 |
| RDS for PostgreSQL / Aurora PostgreSQL | pgaudit | Future |

## Distribution

- Prebuilt `linux/arm64` Lambda packages on GitHub Releases
- A reference Terraform configuration in `examples/terraform/`. You can use it
  as a module, but it is an example rather than a versioned module

## Related projects

- **rds-audit-lake-iceberg** (planned, experimental): converts the raw layout
  into Apache Iceberg tables on Amazon S3 Tables

## License

[MIT](LICENSE)
