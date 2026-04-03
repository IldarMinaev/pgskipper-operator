---
name: postgresql-sql-runner
description: Run read-only SQL queries against PostgreSQL via kubectl exec on the Patroni master pod
---

# PostgreSQL SQL Runner

## Purpose

Execute read-only SQL queries against a PostgreSQL database running in Kubernetes with Patroni. This skill provides a safe, repeatable framework for running diagnostic SQL.

## Prerequisites

- `kubectl` with exec permissions on the target cluster
- Namespace where PostgreSQL is deployed (default: `postgres`)

Invoke the `patroni-reference` skill before proceeding — it contains data directory paths and SQL access patterns. Also see the `pg-credential-handling` skill for credential patterns.

> **🔒 SECURITY**: Never expose passwords in command output. Always use inline credential retrieval: `env PGPASSWORD="$(kubectl get secret ... | base64 -d)"`. Never run `kubectl get secret` separately — it displays the password. See the `pg-credential-handling` skill for detailed patterns.

## Prerequisites

Before proceeding:
1. Invoke the `kubernetes-context` skill to verify cluster access and resolve `<NAMESPACE>` (default: `postgres`).
2. Invoke the `pgskipper-context` skill to verify CRD presence and detect deployment model.

## Step 1: Discover PostgreSQL Pods

```bash
# Find Patroni pods
kubectl get pods -n <NAMESPACE> -l app=patroni -o wide
```

If no pods found with `app=patroni`, try:
```bash
kubectl get pods -n <NAMESPACE> | grep -E 'pg-|patroni'
```

## Step 2: Identify the Master Pod

```bash
# Method 1: Use pgtype label
kubectl get pods -n <NAMESPACE> -l pgtype=master -o name

# Method 2: Use patronictl
kubectl exec -n <NAMESPACE> <any-patroni-pod> -- patronictl -c /patroni/pg_node.yml list
```

From `patronictl list`, the pod with Role=Leader is the master.

## Step 3: Credential Handling

**IMPORTANT**: Do NOT retrieve passwords separately. Use inline retrieval in each command as shown below.

See the `pg-credential-handling` skill for detailed credential handling patterns.

## Step 4: Execute SQL

### Direct query
```bash
kubectl exec -n <NAMESPACE> <MASTER_POD> -- env PGPASSWORD="$(kubectl get secret -n <NAMESPACE> postgres-credentials -o jsonpath='{.data.password}' | base64 -d)" psql -U postgres -d postgres -c "<SQL_QUERY>"
```

### From a SQL file (pipe via stdin)
```bash
kubectl exec -i -n <NAMESPACE> <MASTER_POD> -- env PGPASSWORD="$(kubectl get secret -n <NAMESPACE> postgres-credentials -o jsonpath='{.data.password}' | base64 -d)" psql -U postgres -d postgres -f /dev/stdin < <SQL_FILE>
```

### With expanded output for wide tables
```bash
kubectl exec -n <NAMESPACE> <MASTER_POD> -- env PGPASSWORD="$(kubectl get secret -n <NAMESPACE> postgres-credentials -o jsonpath='{.data.password}' | base64 -d)" psql -U postgres -d postgres -x -c "<SQL_QUERY>"
```

## Safety Rules

1. **Read-only**: Only run SELECT, SHOW, and read-only functions
2. **Never run DDL/DML** (CREATE, ALTER, DROP, INSERT, UPDATE, DELETE) without explicit user approval
3. **Use LIMIT** on potentially large result sets
4. **Set statement_timeout** for expensive queries:
   ```sql
   SET statement_timeout = '30s';
   ```

## Step 5: Interpret Results

After executing SQL, interpret the output:
- Flag CRITICAL/WARNING statuses
- Provide human-readable explanations
- Suggest follow-up queries if needed
- Reference the appropriate troubleshooting skill for deeper investigation

## Common Issues and Remediation

1. **"No master pod found"**: Check Patroni cluster status — may be in failover
2. **"Permission denied"**: Verify the `postgres-credentials` secret exists and has correct password
3. **"Connection refused"**: PostgreSQL may not be running — check pod logs
4. **"Could not connect to server"**: Pod may be in CrashLoopBackOff — check `kubectl describe pod`
5. **Timeout**: Query may be too expensive — add `SET statement_timeout` or simplify the query
