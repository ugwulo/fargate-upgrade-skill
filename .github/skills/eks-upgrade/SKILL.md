---
name: eks-upgrade-check
description: "Assess Fargate-focused EKS cluster upgrade readiness - run read-only checks, validate platform controller compatibility, calculate a readiness score, and generate a report for a target Kubernetes version. Use for EKS upgrade readiness, deprecated APIs, Fargate profiles, add-on compatibility, support lifecycle, or control plane upgrade planning."
---

# EKS Upgrade Readiness Skill

## Overview

This skill assesses a live, Fargate-first EKS cluster's readiness for a Kubernetes version upgrade. It connects via AWS CLI and kubectl, checks control-plane support, Fargate profiles and subnet capacity, platform workloads, deprecated APIs, AWS upgrade insights, and the six registered platform controller families. It produces a readiness score and a detailed report with prioritized remediation steps and pre-filled commands.

This skill is laser-focused on **upgrade safety** — answering the question: "Is it safe to upgrade this cluster to the next version?"

> **Read-only / assessment-only — hard rule.** This skill ONLY inspects the cluster; it
> MUST NOT modify it. Every `aws`, `kubectl`, and MCP call it issues must be a read/list/describe
> operation. NEVER run mutating verbs (`apply`, `create`, `delete`, `patch`, `edit`, `replace`,
> `annotate`, `label`, `set`, `scale`, `cordon`, `drain`, `update-*`, `--force`, etc.), and NEVER
> execute a remediation snippet. Any mutating command embedded in a steering file is a
> **recommendation for the user to run themselves** — surface it as text, do not execute it.

## What Gets Assessed

| # | Section | Key Checks |
|---|---------|------------|
| 01 | Version Validation | Upgrade path validity, version skew policy, support status |
| 02 | Breaking Changes | Version-specific API removals, behavioral changes, resource impact |
| 03 | Deprecated API Detection | Live scan of cluster resources for deprecated/removed APIs |
| 04 | Platform Compatibility | AWS-managed add-ons plus Argo CD, AWS Load Balancer Controller, External Secrets, ACK, Grafana/Alloy Operator, and KEDA |
| 05 | Fargate Data Plane | Fargate profile lifecycle, selector coverage, Pending pods, subnet IP capacity |
| 06 | Workload Risks | Single replicas, missing PDBs, health probes, resource requests |
| 07 | AWS Upgrade Insights | Official EKS pre-upgrade checks and recommendations |
| 08 | Upgrade Plan | Pre-filled CLI commands, step-by-step upgrade sequence |
| 09 | Configuration Alignment | Read-only comparison of live versions/profiles with checked-in IaC |

## Readiness Score

The skill calculates a weighted readiness score:

| Category | Max Deduction | Rationale |
|----------|--------------|-----------|
| Breaking Changes | 25 pts | Highest risk — can break apps |
| Deprecated APIs | 20 pts | Actionable, fixable pre-upgrade |
| Fargate Data Plane (profiles + subnet IPs) | 20 pts | Can block control-plane upgrade or prevent task placement |
| Unsupported Version | 15 pts | No security patches, urgent upgrade needed |
| Add-on Compatibility | 15 pts | Critical > optional add-ons |
| Workload Risks | 10 pts | Best-practice, not blockers |
| AWS Upgrade Insights | 10 pts | Official AWS checks |
| Behavioral Changes | 5 pts | Version-specific workload impact |

**Hard Blocker Override:** If any hard blocker is detected (e.g., a critical platform
controller is incompatible or degraded, cluster subnets collectively cannot place control-plane
ENIs, a required Fargate profile is not ACTIVE, or the cluster is not ACTIVE), the score is capped at ≤ 59% (NOT READY)
regardless of other findings. See `steering/report-generation.md` for the full list.

**Score Interpretation:**
- 90-100: **READY** — Safe to proceed
- 80-89: **GOOD** — Minor issues, can proceed with caution
- 70-79: **FAIR** — Several issues need attention first
- 60-69: **RISKY** — Significant issues, not recommended yet
- 0-59: **NOT READY** — Critical blockers, must resolve first

## Prerequisites

1. **AWS credentials configured** — `aws configure` or `~/.aws/credentials` with EKS access
2. **kubectl access** to the target cluster (for Kubernetes API queries)
3. **Required AWS Permissions:**
   - `eks:DescribeCluster`, `eks:ListClusters`, `eks:ListFargateProfiles`, `eks:DescribeFargateProfile`, `eks:ListNodegroups`
   - `eks:ListAddons`, `eks:DescribeAddon`, `eks:DescribeAddonVersions`, `eks:ListInsights`, `eks:DescribeInsight`
   - `ec2:DescribeSubnets`

### MCP Server Setup

This skill uses two MCP servers, both pre-configured in `.mcp.json` at the project root:

- `awslabs.eks-mcp-server` — connects to your EKS cluster
- `awslabs.aws-documentation-mcp-server` — looks up AWS documentation during assessment

Enable both servers in the Copilot/MCP host when available. If MCP servers are not available, the skill falls back to AWS CLI and kubectl commands.

### Configuration

The skill uses your existing AWS credentials. No additional configuration needed if `aws eks list-clusters` works from your terminal.

To use a specific profile or region, set environment variables:
```bash
export AWS_PROFILE=your-profile-name
export AWS_REGION=your-region
```

### Getting Started

Invoke the skill: `/eks-upgrade-check`

Or simply ask: *"Run an EKS upgrade readiness assessment"*

The skill will discover your clusters, ask which one to assess and what target version, then run the full assessment.

---

## Assessment Workflow

### Step 0: Pre-flight

**Action 1 — List clusters (test connectivity & discover clusters)**

Run `aws eks list-clusters` to discover available clusters.

> **Region caveat.** `aws eks list-clusters` is **region-scoped** (it lists only the current/`--region`
> region) and returns **names only, not regions**. An empty result means "no clusters in this region,"
> NOT "no clusters in the account" — before treating zero clusters as terminal, confirm the intended
> region (`echo $AWS_REGION`) and, if the region is ambiguous, list the likely regions. Any "name +
> region" shown to the user pairs the returned name with the region actually queried.

- ✅ Success → Show the cluster list. Ask which cluster to assess. If only one cluster, confirm it.
- ❌ Failure → STOP. Do NOT retry more than once. Show:

> **Cannot access EKS clusters.** Try these steps:
> 1. Check that AWS credentials are configured: `aws sts get-caller-identity`
> 2. Check your region: `aws eks list-clusters --region <region>`
> 3. Check that the configured MCP servers are enabled in VS Code

Wait for the user to resolve the issue.

**Action 2 — Describe the selected cluster**

Run `aws eks describe-cluster --name <cluster>` and show: cluster name, Kubernetes version, platform version, region, status, account ID.

> **Account ID hygiene:** the account ID (from `aws sts get-caller-identity` / the cluster ARN) is sensitive. If the report will be shared outside the account, mask or omit the account ID before sharing.

**Action 2b — Validate cluster status**

Check the `status` field from the cluster description. If status is NOT `ACTIVE`:
- **CREATING/UPDATING/DELETING** → STOP. Show: "Cluster is currently in `<status>` state. The EKS API will reject an upgrade request. Wait for the operation to complete, then re-run this assessment."
- **FAILED** → STOP. Show: "Cluster is in FAILED state. This is a hard blocker — the cluster must be recovered before an upgrade can be attempted. Contact AWS Support if the cluster is stuck in FAILED."

Do NOT proceed with the assessment if cluster status is not ACTIVE. This is a hard blocker (see report-generation.md).

Cluster status gates the whole assessment. For Fargate, profile status and selector coverage gate data-plane readiness; there is no customer-managed node rotation to wait for. If managed node groups are present, classify the cluster as hybrid and assess those groups separately.

**Action 3 — Validate permissions (AWS + Kubernetes)**

**3a — AWS API preflight.** After describing the cluster, verify key AWS permissions by attempting:
1. `aws eks list-fargate-profiles --cluster-name <cluster>`
2. `aws eks list-addons --cluster-name <cluster>`
3. `aws eks describe-addon-versions --kubernetes-version <current>` (add-on compatibility — `addon-compatibility.md` marks this a MUST-run read)
4. `aws eks list-insights --cluster-name <cluster>`
5. `aws ec2 describe-subnets --subnet-ids <cluster subnet ids>` (Fargate/control-plane subnet-IP input)

`eks:DescribeCluster` / `eks:DescribeNodegroup` / `eks:DescribeAddon` / `eks:DescribeInsight` are
exercised implicitly by the assessment steps themselves; the probes above cover the list/describe
reads that gate scoring inputs.

**3b — Kubernetes RBAC preflight.** The high-weight assessment categories read Kubernetes objects,
not just AWS APIs. Verify cluster read access with `kubectl auth can-i` before scanning:

```bash
kubectl auth can-i list deployments -A          # workloads (workload-risks, deprecated-apis)
kubectl auth can-i list daemonsets -A           # workloads
kubectl auth can-i list statefulsets -A         # workloads
kubectl auth can-i list validatingwebhookconfigurations   # webhooks (breaking-changes)
kubectl auth can-i list mutatingwebhookconfigurations     # webhooks
kubectl auth can-i list horizontalpodautoscalers -A       # HPA (deprecated-apis)
```

If `kubectl auth can-i` itself errors (not a clean yes/no), treat the read as denied.

**Denied-read discipline (same for the AWS and Kubernetes preflights).** If any probe above
returns `AccessDenied` (AWS) or `no` (kubectl) → surface exactly which read is denied and the IAM
action or RBAC verb/resource needed, then ask the user whether to (a) fix the permission and
re-run the probe, or (b) continue with a **partial assessment**. A denied read is NOT a hard stop
and NOT a silent 0: the affected category is reported UNKNOWN / not-scored and listed in
`## Unassessed`, per `steering/report-generation.md`. A partial assessment can NEVER yield an
uncaveated READY — the headline verdict carries the partial marker and is capped below READY.
The guarantee this preflight gives extends only to the reads it actually probes.

**Action 4 — Determine target version**

Ask: *"Your cluster is on v[current]. The next version is v[current+1]. Shall I assess upgrade readiness to v[current+1]?"*

If the user specifies a version more than 1 minor version ahead, explain that EKS requires one-version-at-a-time upgrades and show the required path (e.g., 1.29 → 1.30 → 1.31 → 1.32). Offer to assess the first hop.

**Action 5 — Confirm and proceed**

### Steps 1-8: Run Assessment

Read each steering file in order from `${SKILL_DIR}/steering/`. For each section:
1. Read the steering file
2. Execute the checks described in it using AWS CLI and kubectl commands
3. Collect findings with severity ratings

**Steering file loading guide:**

| User Request | Steering File(s) |
|---|---|
| Full upgrade assessment | ALL files in order |
| Version / upgrade path | `steering/version-validation.md` |
| Breaking changes / API removals | `steering/breaking-changes.md` |
| Deprecated APIs | `steering/deprecated-apis.md` |
| Platform controller compatibility | `steering/addon-compatibility.md` |
| Fargate profiles / data plane | `steering/node-readiness.md` |
| Workload risks / PDB / probes | `steering/workload-risks.md` |
| AWS Insights | `steering/upgrade-insights.md` |
| Generate report | `steering/report-generation.md` |
| Source-controlled configuration drift | `steering/configuration-drift.md` |

### Step 9: Calculate Score & Generate Report

Read `${SKILL_DIR}/steering/report-generation.md` and produce the report.

---

## Tool Usage Rules

1. **Do NOT call any tools when this skill is first activated.** Wait for the user to ask.
2. **Do NOT hardcode or guess cluster names.** Always discover by listing first.
3. **Do NOT retry a failed command more than once.**
4. **Always read the relevant steering file before executing checks for that section.**
5. **Use `aws` CLI and `kubectl` for cluster queries.** If MCP servers are available, prefer them for EKS operations.

## Data Files

- **Platform Controller Registry:** `${SKILL_DIR}/data/oss_addon_registry.json` — identifiers and authoritative upstream URLs for the six service-owned controllers. This file does NOT contain compatibility data. Compatibility is always verified live via the registry's `compatibility_url` and `releases_url` fields. If a controller is not in the registry or the upstream source is unreachable, report UNKNOWN — never guess.
- **HTML Converter:** `${SKILL_DIR}/tools/md_to_html.py` — converts markdown reports to HTML

## Report Output

- **Markdown:** `EKS-Upgrade-Assessment-<cluster>-<current>-to-<target>-<YYYY-MM-DD>-<HHMM>.md`
- **HTML:** Run `python3 ${SKILL_DIR}/tools/md_to_html.py <report>.md` to convert

Do NOT generate HTML manually. Always use the conversion script.
