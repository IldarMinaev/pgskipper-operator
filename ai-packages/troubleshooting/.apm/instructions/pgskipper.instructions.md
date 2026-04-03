# pgskipper-operator Troubleshooting — Agent Instructions

This repository contains AI-agent skills for troubleshooting PostgreSQL databases managed by [pgskipper-operator](https://github.com/Netcracker/pgskipper-operator) in Kubernetes.

Each skill is a `SKILL.md` file that any AI agent reads and executes directly. No wrapper scripts, no test harnesses — the agent IS the execution engine.

## Default Cluster Assumptions

**Unless the user says otherwise, always assume:**

1. **PostgreSQL is managed by pgskipper-operator** — clusters are represented as `PatroniCore` and `PatroniServices` custom resources; all configuration flows through Helm charts `patroni-core` and `patroni-services`, applied either via direct `helm upgrade` or via an ArgoCD Application that manages the Helm release. Always detect PostgreSQL clusters first before asking questions about namespace or scope.
2. **Backups are via pgBackRest** managed by `postgres-backup-daemon`.

Cluster discovery must use **pgskipper-aware commands**, not generic pod listing:

```bash
# Discover PostgreSQL clusters
kubectl get patronicores --all-namespaces
kubectl get patroniservices --all-namespaces

# Discover Patroni pods (after namespace is known)
kubectl get pods -n <ns> -l app=patroni
kubectl get pods -n <ns> -l pgtype=master    # primary only

# Discover Helm releases
helm list -n <ns> --filter 'patroni'
```

**Anti-pattern**: Do NOT run `kubectl get pods -n <ns>` without a label selector to discover PostgreSQL pods — use CRD queries and label selectors instead.

## How to Use Skills

1. At the start of every session, run `kubernetes-context` then `pgskipper-context` to confirm cluster access, resolve the target namespace, and detect the deployment model (Helm vs ArgoCD)
2. When the user describes a problem, identify which skill(s) match using the skill selection guide below
3. Invoke the relevant skill by name and follow its steps
4. Invoke shared skills (`patroni-reference`, `pgskipper-architecture`, `pg-credential-handling`) as needed; use the `sql` skill for shared SQL scripts
5. If the issue spans multiple areas, combine skills (e.g., health-check + storage-check)

## Skill Selection Guide

| Symptom | Start With |
|---------|-----------|
| Operator or cluster not healthy | `pgskipper-check` → `postgresql-health-check` |
| Slow queries / high CPU | `postgresql-performance-check` |
| Disk full / storage growing | `postgresql-storage-check` |
| Connection errors / pool exhaustion | `postgresql-connection-check` |
| Backup failures | `postgresql-backup-check` |
| Error messages in logs | `postgresql-log-analyzer` |
| Missing metrics / alerts | `monitoring-check` |
| Run a diagnostic SQL query | `postgresql-sql-runner` |

## When to Ask for More Information

Before running any commands, evaluate what the user has provided:

- **Sufficient** (symptom + scope + timeline) — proceed directly to the matching skill
- **Partial** — ask: Which namespace/cluster? When did it start? What changed? Which application?
- **Vague or multi-symptom** — start with `pgskipper-check` then `postgresql-health-check` for a structured sweep

Never start investigating without knowing **what** is broken, **where** it is, and **since when**.

## Remediation Policy

**All configuration changes to operator-managed resources MUST go through the deployment tool (Helm or ArgoCD) — never via direct kubectl manipulation.**

pgskipper-operator manages PatroniCore and PatroniServices CRs (and the Kubernetes resources they own) declaratively through Helm charts. Bypassing the deployment tool creates drift, may be reverted on the next reconciliation or ArgoCD sync cycle, and removes rollback capability. Detect the deployment model before suggesting changes.

### When kubectl is acceptable

Use `kubectl` for **read-only investigation** (get, describe, logs, exec for diagnostics) and for resources that are **not** Helm/operator-managed (e.g., manually created ConfigMaps, one-off jobs, namespace-level objects outside operator scope).

### How to propose a configuration fix

When a CR parameter needs correcting, first identify the deployment model, then suggest user steps to fix via detected deployment model. To find supported configuration parameters use context7 MCP or architecture skills in this repo.

## Anti-Patterns

| Anti-Pattern | Instead |
|-------------|---------|
| Running `kubectl get pods` without label selectors | Use CRD queries and label selectors |
| `kubectl patch` / `kubectl edit` on operator-managed resources | Fix via Helm or ArgoCD |
| `helm upgrade` when ArgoCD manages releases | Update Git values + `argocd app sync` |
| Exposing passwords in command output | Use inline credential retrieval |
| Running DDL/DML without user approval | Always ask first |
