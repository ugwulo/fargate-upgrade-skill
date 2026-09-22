# Workload Risks

## Purpose
Assess workload resilience during the upgrade process. These are not upgrade blockers but affect the safety and smoothness of the upgrade.

## CRITICAL: Systematic Enumeration Rule

You MUST follow this process to avoid miscounting. Do NOT count from memory.

### Step A: Build the Master Workload Table

Before checking ANY risk, build a single table of ALL workloads in non-system namespaces.

**Non-system namespaces to EXCLUDE:** kube-system, kube-public, kube-node-lease, karpenter,
amazon-cloudwatch, amazon-guardduty, aws-observability.

**Workload types to INCLUDE:** Deployments, StatefulSets, DaemonSets.

**How to build the table:**
1. List ALL Deployments across all namespaces
2. List ALL StatefulSets across all namespaces
3. List ALL DaemonSets across all namespaces
4. Filter out workloads in system namespaces listed above
5. For EACH remaining workload, extract from its spec:
   - `name`, `namespace`, `kind` (Deployment/StatefulSet/DaemonSet)
   - `replicas` (for Deployments/StatefulSets; DaemonSets run on all nodes)
   - `strategy.type` (Deployments only: RollingUpdate or Recreate)
   - For EACH container: `readinessProbe` (present/absent), `livenessProbe` (present/absent),
     `resources.requests.cpu` (value or absent), `resources.requests.memory` (value or absent)

**Output format — you MUST produce this table before proceeding:**

```
| # | Name | Kind | NS | Replicas | Strategy | Probes | Requests | Notes |
|---|------|------|----|----------|----------|--------|----------|-------|
| 1 | app-a | Deployment | default | 3 | RollingUpdate | ✅ readiness+liveness | ✅ cpu+mem | |
| 2 | app-b | Deployment | default | 1 | Recreate | ❌ none | ❌ none | single-replica, recreate |
| 3 | mon-agent | DaemonSet | default | N/A | N/A | ❌ none | ✅ cpu+mem | |
```

### Step B: Check Each Risk Against the Table

Walk through each check below. For every finding, reference the row number from the table.
This prevents miscounting and ensures no workload is missed.

## Checks to Execute

### 6.1 — Single Replica Deployments and StatefulSets

**Why this matters:** Node drains during upgrade will cause downtime for single-replica workloads.

**How to check:** From the master table, filter for `kind IN (Deployment, StatefulSet) AND replicas == 1`. StatefulSets are collected in the master table (Step A above) and are scored identically to single-replica Deployments in `report-generation.md` — do NOT restrict this check to Deployments only, or single-replica StatefulSets will be collected but never scored.

**Rating:** Each match = HIGH severity (3 pts in score).

### 6.2 — Missing Pod Disruption Budgets

**Why this matters:** Without PDBs, node drain can evict all pods simultaneously.

**How to check:**
1. List PodDisruptionBudgets across all namespaces
2. From the master table, filter for `kind == Deployment AND replicas > 1` in non-system namespaces
3. Cross-reference: which multi-replica deployments have NO matching PDB?
4. Check for **drain-blocking PDBs** (see 6.2b below)

**IMPORTANT:** Only flag missing PDBs for workloads with replicas > 1. A PDB on a single-replica
deployment is meaningless — do NOT flag single-replica workloads for missing PDBs.

**Rating:** Each missing PDB on multi-replica deployment = MEDIUM severity (1 pt).

### 6.2b — Drain-Blocking PDBs (upgrade stall risk)

**Why this matters:** A PDB that allows zero disruptions can prevent voluntary pod
replacement while the control plane or a controller rollout is in progress. On Fargate,
there is no customer-managed node drain; the risk is stalled pod replacement or a
controller rollout that cannot make progress.

**How to check:**
1. For each PDB found in step 6.2, inspect. A PDB is **drain-blocking** if it currently
   permits zero voluntary disruptions — expressed in any of these equivalent forms:
   - `status.disruptionsAllowed` == 0 (authoritative runtime signal; prefer this when present), OR
   - `spec.maxUnavailable` == 0, OR
   - `spec.minAvailable` >= total replicas of the target workload
2. If the PDB is drain-blocking (any one of the above) AND the target workload has pods
   running on nodes that will be drained during the upgrade → flag it. The three conditions
   are equivalent expressions of the same "0 disruptions allowed" state — do NOT count a
   single PDB more than once.

**Report message (use this exact framing):**

> **⚠️ PDB may stall Fargate pod replacement**
>
> `<pdb-name>` in namespace `<ns>` currently allows 0 disruptions for `<workload-name>`.
> During a rollout or voluntary disruption, this PDB cannot be satisfied. On Fargate, the
> replacement pod may remain Pending or the rollout may stop until another pod is healthy.
>
> **Before upgrading:**
> 1. Verify sufficient Fargate profile subnet capacity exists for replacement pods
> 2. Consider temporarily relaxing the PDB: `kubectl patch pdb <name> -n <ns> -p '{"spec":{"maxUnavailable":1}}'`
> 3. Or ensure the workload has enough replicas spread across multiple nodes
>
> **If you skip this:** a controller or workload rollout may stall and require manual
> intervention. This does not by itself block the EKS control plane update.

**Rating:** Each drain-blocking PDB = MEDIUM severity (2 pts).

**This is NOT a hard blocker** because:
- The control plane upgrade itself will succeed
- The issue is most visible during pod replacement or controller rollout
- It can be resolved mid-upgrade by patching the PDB
- But it WILL cause significant delay and potential manual intervention if not addressed

### 6.3 — Missing Health Probes

**Why this matters:** Without readiness probes, traffic is sent to pods before they're ready.

**How to check:** From the master table, filter for workloads where ANY container is missing
a `readinessProbe`. Count ALL workload types (Deployments, StatefulSets, AND DaemonSets).

**Rating:** Each workload missing probes = MEDIUM severity (1 pt).

### 6.4 — Missing Resource Requests

**Why this matters:** Without resource requests, pods can't be properly rescheduled during node drains.

**How to check:** From the master table, filter for workloads where ANY container is missing
`resources.requests.cpu` OR `resources.requests.memory`.

**IMPORTANT:** Check the ACTUAL spec data. Do NOT assume a workload has or lacks requests
without verifying. If the deployment spec shows `requests: {cpu: "100m", memory: "128Mi"}`,
that workload HAS requests — do not flag it.

**Rating:** Each workload missing requests = MEDIUM severity (1 pt).

### 6.5 — Recreate Update Strategy

**Why this matters:** Recreate strategy causes full downtime during any rollout.

**How to check:** From the master table, filter for `kind == Deployment AND strategy == Recreate`.

**Rating:** Each match = HIGH severity (3 pts in score).

### 6.6 — Graceful Shutdown Configuration

**Why this matters:** Without preStop hooks, there's a race condition during node drain.

**How to check:**
1. From the master table, identify workloads exposed via Services (especially LoadBalancer type)
2. Check if those workloads have `lifecycle.preStop` hooks
3. Check `terminationGracePeriodSeconds`

**Externally-facing** = a workload backed by a LoadBalancer-type Service OR an Ingress (these receive external traffic and are sensitive to abrupt pod termination); ClusterIP-only workloads are NOT externally-facing.

**Rating:** Missing preStop on externally-facing workloads = MEDIUM severity (1 pt).
**Scoring home:** Category 6 (Workload Risks) MEDIUM — counted in the report-generation.md Category 6 pseudocode, subject to the 4-pt MEDIUM sub-cap.

### Fargate-only cluster caveat

**On a Fargate-only cluster**, `kubectl get nodes` may be empty or show only transient
Fargate nodes. Category 3 (node readiness) is **N/A** and deducts 0 because AWS owns the
runtime. The data-plane assessment relies on Fargate profile selectors, subnet capacity,
Pending pod events, and pod-level signals such as preStop hooks, PDBs, and probes. When
reporting, state that a high score reflects only those observable signals.

## Step C: Compile Findings with Row References

After all checks, produce a findings list that references the master table row numbers:

```
| Finding | Severity | Workloads (by row #) | Count |
|---------|----------|---------------------|-------|
| Single replica | HIGH | #2, #7 | 2 |
| Recreate strategy | HIGH | #2, #5 | 2 |
| Missing probes | MEDIUM | #2, #3, #5, #6, #8 | 5 |
| Missing requests | MEDIUM | #2, #6 | 2 |
```

This makes the count verifiable. If the count doesn't match the listed row numbers, something is wrong.

## Score Impact

> **Canonical scoring is defined in `steering/report-generation.md` §Category 6 (Workload Risks).**

| Finding | Deduction |
|---------|-----------|
| High-severity workload risk (single replica, Recreate) | 3 pts each (sub-cap 8) |
| Medium-severity workload risk (missing probes, requests, PDBs) | 1 pt each (sub-cap 4) |
| Drain-blocking PDB (disruptionsAllowed == 0) | 2 pts each (sub-cap 4) |
| Max category | 10 pts |
