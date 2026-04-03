---
name: pgskipper-context
description: Verify pgskipper-operator installation and detect Helm vs ArgoCD deployment model — run after kubernetes-context
---

# pgskipper-operator Context Verification

Run this skill after the `kubernetes-context` skill to verify pgskipper-operator-specific prerequisites: Patroni CRD presence and deployment model detection.

## Step 1: Verify pgskipper-operator is Deployed

```bash
kubectl get crd patronicores.netcracker.com
kubectl get crd patroniservices.netcracker.com
```

If CRDs are not found, pgskipper-operator is not installed in this cluster. Report this and stop — all PostgreSQL skills in this package require the operator.

## Step 2: Detect Deployment Model (Helm vs ArgoCD)

```bash
kubectl get applications --all-namespaces 2>/dev/null | grep -iE 'patroni|postgres'
```

**Interpret**:
- ArgoCD Applications found managing PostgreSQL Helm releases → **all remediation must go through Git + ArgoCD**, not `helm upgrade`. Record this and carry it through every subsequent remediation step.
- No ArgoCD found → standard direct Helm remediation applies.

## Output

Append to the `kubernetes-context` summary:

4. **CRDs**: present or missing
5. **Deployment model**: Helm-direct or ArgoCD-managed

Full example summary:
> Context: `prod-cluster`, namespace: `postgres-prod`, permissions: OK, CRDs: present, deployment model: ArgoCD-managed.
