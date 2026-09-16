# ccl-lab5-rds-gitea

# CCL Lab 5 — RDS Migration for Gitea (Post-Lab Assignment 1)

## Overview
This documents migrating the PostgreSQL database backing a self-hosted Gitea
instance (deployed in CCL Lab 4) from a local, EC2-hosted PostgreSQL server to
a managed Amazon RDS PostgreSQL instance, and demonstrating full CRUD
operations against it through the running application.

## Architecture

Browser --> Nginx (EC2, port 80) --> Gitea (EC2, loopback 127.0.0.1:3000)
--> Amazon RDS PostgreSQL (private, port 5432)

- EC2 instance: Amazon Linux 2023, t3.micro
- RDS instance: PostgreSQL 15.x, db.t3.micro, Free Tier, Single-AZ
- RDS is **not publicly accessible** — reachable only from the EC2 security group

## RDS Configuration

| Setting | Value |
|---|---|
| Engine | PostgreSQL 15.x |
| Instance class | db.t3.micro (Free Tier) |
| Storage | 20 GB gp3 |
| Public access | No |
| Initial database name | giteadb |
| VPC | Same VPC as EC2 instance |

## Security Group Configuration

- RDS security group (`gitea-rds-sg`) inbound rule:
  - Type: PostgreSQL, Port: 5432
  - Source: EC2 instance's security group ID (not a CIDR/IP range)
- This ensures only traffic originating from the EC2 instance can reach the
  database — RDS has no route from the public internet.

## Migration Steps

1. Dumped the existing local database: pg_dump -h 127.0.0.1 -U giteauser -d giteadb -f /tmp/gitea_migrate.sql

2. Verified connectivity from EC2 to RDS: psql -h <rds-endpoint> -U postgres -d postgres -c "SELECT version();"

3. Created the application role and granted ownership on RDS:
```sql
   CREATE USER giteauser WITH ENCRYPTED PASSWORD '********';
   ALTER DATABASE giteadb OWNER TO giteauser;
   GRANT ALL PRIVILEGES ON DATABASE giteadb TO giteauser;
```
4. Restored the dump into RDS: psql -h <rds-endpoint> -U giteauser -d giteadb -f /tmp/gitea_migrate.sql

5. Verified tables were present with `\dt`.

## Application Configuration Change

Updated `/etc/gitea/app.ini`:

```ini
[database]
DB_TYPE = postgres
HOST = <rds-endpoint>:5432
NAME = giteadb
USER = giteauser
PASSWD = ********
SSL_MODE = require
```

Key changes from the Lab 4 config: `HOST` now points to the RDS endpoint
instead of `127.0.0.1`, and `SSL_MODE` was switched from `disable` to
`require` since RDS supports encrypted connections in transit.

## Proof of Migration

The local PostgreSQL service on EC2 was stopped and disabled entirely
(`systemctl stop postgresql && systemctl disable postgresql`), and Gitea
continued to serve all data correctly — confirming it was genuinely reading
from RDS and not falling back to the local database.

## CRUD Demonstration

All four operations were performed through the live Gitea web UI and
independently verified with SQL queries against RDS:

- **Create** — created repository `crud-demo` under `lab4-org`; confirmed
  with `SELECT ... FROM repository`.
- **Read** — created issues in the repo; confirmed with
  `SELECT ... FROM issue`.
- **Update** — edited an issue title in the UI; confirmed the updated
  `name` and `updated_unix` columns.
- **Delete** — deleted an issue; confirmed the row count dropped via
  `SELECT COUNT(*) FROM issue WHERE repo_id = ...` before and after.

## Schema Relationship

Gitea's `issue` table has a foreign key (`repo_id`) referencing
`repository.id`, satisfying the two-table relationship requirement:

```sql
SELECT r.name AS repo, i.index AS issue_no, i.name AS issue_title, i.is_closed
FROM issue i
JOIN repository r ON i.repo_id = r.id
ORDER BY r.name, i.index;
```

## Evidence

See `/screenshots` in this repository for the full evidence set, including:
- RDS instance configuration and "Available" status
- RDS security group inbound rule (sourced from EC2 SG, not 0.0.0.0/0)
- Successful `psql` connection test from EC2 to RDS
- Gitea running and responding after the config switch
- Local PostgreSQL stopped/disabled while Gitea continued working
- CRUD operations (Create, Read, Update, Delete) with matching SQL query output

## Links

- EC2 application URL: http://3.88.100.132 *(may change if the instance is stopped/started)*
