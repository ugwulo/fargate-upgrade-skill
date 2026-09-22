# Fargate EKS Upgrade Readiness Skill

A GitHub Copilot Agent Skill for read-only EKS upgrade assessments, tailored to Fargate-first clusters and a Kubernetes 1.35 target.

The skill reports:

- control-plane version, support lifecycle, and one-minor upgrade-path validity;
- Fargate profile status, selectors, subnet capacity, and Pending pod signals;
- AWS-managed add-on versions and target compatibility;
- live compatibility evidence for Argo CD, AWS Load Balancer Controller, External Secrets, ACK, Grafana/Alloy Operator, and KEDA;
- deprecated APIs, target-version breaking changes, workload risks, and AWS Upgrade Insights;
- an auditable score, blockers, evidence, and a recommended upgrade sequence.
- a read-only comparison between live cluster/Fargate profile values and checked-in IaC.

## Read-Only Boundary

The skill only runs AWS `describe`/`list` operations and Kubernetes read/list queries. It never executes upgrade, deployment, patch, drain, or remediation commands. Commands in reports are recommendations for an engineer to run through the approved change process.

## Install and Use

The skill is stored at `.github/skills/eks-upgrade/SKILL.md`, which is the Copilot workspace skill location. In VS Code, ask Copilot to run an EKS upgrade readiness assessment and provide the target version, normally `1.35`.

The assessment discovers the cluster and region instead of accepting hard-coded cluster names. Configure AWS credentials and a region before starting. AWS CLI and `kubectl` must be able to perform the read operations listed in the skill.

## Fargate Scope

For a Fargate-only cluster, node groups, AMIs, AL2, containerd, kubelet skew, Karpenter, and node draining are not scored. The skill instead validates Fargate profile lifecycle and selector coverage, profile subnet IP capacity, pod placement, controller health, webhooks, and workload resilience.

If managed or self-managed node groups are present, the cluster is reported as hybrid. The Fargate checks still apply, but EC2 node readiness requires a separate approved procedure rather than being silently treated as Fargate.

## Compatibility Evidence

`.github/skills/eks-upgrade/data/oss_addon_registry.json` is deliberately limited to the six service-owned controller families. It stores identifiers and authoritative URLs only; compatibility is fetched live from each project's documentation. Unknown or ambiguous evidence is reported as `UNKNOWN_VERIFIABLE`.

## Runbook Boundary

The skill produces an assessment and a suggested execution order. The organization's final runbook should own environment rollout order, source-controlled configuration reconciliation, change approvals, backup/restore decisions, and rollback planning. Velero can be evaluated as part of workload/data recovery planning, but a Velero backup does not roll back the EKS control plane or AWS-managed Fargate runtime.

## HTML Reports

After a Markdown report is generated, convert it with:

```bash
python3 .github/skills/eks-upgrade/tools/md_to_html.py <report>.md
```
