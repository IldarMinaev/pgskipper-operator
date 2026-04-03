---
name: postgresql-troubleshooting-sql
description: Shared SQL scripts for PostgreSQL health, performance, replication, storage, connections, locks, and configuration analysis — used by troubleshooting skills via psql stdin
---

# PostgreSQL Troubleshooting — SQL Scripts

Shared SQL scripts used by the `pgskipper-operator` skills via psql stdin redirect.

This `_sql/` directory is deployed as a sibling to the other skill directories. After `apm install`, the SQL files are located at `<SKILLS_DIR>/_sql/<script>.sql` where `<SKILLS_DIR>` depends on your AI agent runtime (e.g., `.claude/skills/` for Claude, `.github/skills/` for Copilot).

## Contents

| Script | Purpose |
|--------|---------|
| [health_check.sql](health_check.sql) | Connections, replication lag, dead tuples, long queries, locks, WAL, database sizes, health summary |
| [performance.sql](performance.sql) | Slow queries (pg_stat_statements), active queries, connection states, cache hit ratio, table I/O, unused indexes, vacuum health |
| [replication.sql](replication.sql) | Streaming replication lag, replication slots, WAL statistics, archive status, inactive slot warnings |
| [connections.sql](connections.sql) | Connection pool usage, idle-in-transaction, per-client stats, max_connections headroom |
| [storage.sql](storage.sql) | Database and table sizes, bloat indicators, WAL accumulation, PVC-level usage hints |
| [locks.sql](locks.sql) | Lock trees, blocked queries, advisory locks, deadlock-prone patterns |
| [bloat_estimation.sql](bloat_estimation.sql) | Table and index bloat estimation using pgstattuple-compatible heuristics |
| [configuration.sql](configuration.sql) | Key PostgreSQL configuration parameters with current values and recommendations |
| [memory_requirements.sql](memory_requirements.sql) | Memory configuration vs pod limit comparison — detects overcommit risk |
| [memory_intensive_queries.sql](memory_intensive_queries.sql) | Queries spilling to disk via temp files — candidates for work_mem tuning |

## Usage

Scripts are executed via psql stdin redirect from the skill steps:

```bash
# SQL_DIR is set in each skill's context step:
SQL_DIR=$(find . -maxdepth 5 -type d -name '_sql' 2>/dev/null | head -1)

kubectl exec -i -n <NAMESPACE> $MASTER_POD -- \
  env PGPASSWORD="$(kubectl get secret -n <NAMESPACE> postgres-credentials \
    -o jsonpath='{.data.password}' | base64 -d)" \
  psql -U postgres -d postgres -f /dev/stdin < "$SQL_DIR/health_check.sql"
```
