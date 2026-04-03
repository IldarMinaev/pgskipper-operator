---
applyTo: "**"
---

## Skill trigger: `postgresql-health-check`

Invoke the `postgresql-health-check` skill when the user reports or asks about:
- Patroni cluster status, timeline, or leader issues
- PostgreSQL pod restarts, CrashLoopBackOff, or OOMKilled
- Replication lag or replica not streaming
- General "is the database healthy?" or "check the cluster" requests
- Pod resource usage (CPU/memory) for PostgreSQL pods
- PVC status for database volumes
- Failover or switchover events
- Cluster members showing unexpected states (e.g., replica stuck in "starting")
