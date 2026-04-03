---
name: pg-credential-handling
description: PostgreSQL credential handling patterns — inline retrieval, security rules, and common mistakes to avoid
---

# PostgreSQL Credential Handling

## PostgreSQL Credentials

### Superuser (postgres)

Stored in the `postgres-credentials` Kubernetes secret.

**✅ CORRECT Pattern** (inline retrieval — password never exposed):
```bash
kubectl exec -n <ns> <master-pod> -- env PGPASSWORD="$(kubectl get secret -n <ns> postgres-credentials -o jsonpath='{.data.password}' | base64 -d)" psql -U postgres -d postgres -c "<SQL>"
```

**Alternative** (chained commands — for multiple uses):
```bash
PGPASSWORD=$(kubectl get secret -n <ns> postgres-credentials -o jsonpath='{.data.password}' | base64 -d) && \
kubectl exec -n <ns> <master-pod> -- env PGPASSWORD="$PGPASSWORD" psql -U postgres -d postgres -c "<SQL>"
```

**❌ FORBIDDEN** (exposes password in output):
```bash
# DO NOT run this separately — it displays the password!
kubectl get secret -n <ns> postgres-credentials -o jsonpath='{.data.password}' | base64 -d
```

### Replicator User

Stored in `replicator-credentials` secret. Used internally by Patroni for streaming replication. Rarely needed for troubleshooting.

## Safety Rules

### Quick Reference

1. **Never expose passwords** in command output or reports
   - ✅ Use inline retrieval: `env PGPASSWORD="$(kubectl get secret ...)"`
   - ✅ Use chained commands: `VAR=$(...) && use $VAR`
   - ❌ NEVER run retrieval separately: `kubectl get secret ... | base64 -d`

2. **Never hardcode passwords** — always retrieve from secrets at runtime
   - ❌ `env PGPASSWORD="MYrootPWD"` (hardcoded)
   - ✅ `env PGPASSWORD="$(kubectl get secret ...)"` (retrieved)

3. **Use environment variables** (`PGPASSWORD`) rather than command-line `-W` flag
   - Prevents password from appearing in process lists

4. **Scope access**: Use read-only queries for diagnostics
   - Never run DDL/DML (CREATE, DROP, UPDATE, DELETE) without explicit user approval

5. **Clean up**: If port-forwarding was started, kill the process after use

6. **Verify before execution**: Check that commands don't expose credentials
   - Review the inline retrieval pattern before running any psql command

---

## When to Use Each Pattern

| Scenario | Pattern | Example |
|----------|---------|---------|
| Single query | Inline | `kubectl exec ... -- env PGPASSWORD="$(kubectl get secret ...)" psql -c "..."` |
| Multiple queries | Chained | `PW=$(...) && cmd1 && cmd2 && cmd3` |
| SQL from file | Inline | `kubectl exec ... -- env PGPASSWORD="$(kubectl get secret ...)" psql -f /dev/stdin < file.sql` |

---

## Additional Resources

- **`patroni-reference`** skill — Patroni-specific commands and PostgreSQL access patterns
