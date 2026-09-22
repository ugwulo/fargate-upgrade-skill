# Fargate Report Generation

## Purpose

Generate an auditable readiness report for a Fargate-first EKS cluster. The report must
separate control-plane upgrade blockers from Fargate scheduling and workload risks. Do
not score node groups, AMIs, AL2, containerd, kubelet skew, or Karpenter for a
Fargate-only cluster.

## 1. Assessment State

A category whose backing AWS or Kubernetes read was denied, errored, or partial is
`UNKNOWN` and contributes no deduction. It must appear in `## Unassessed`. An empty
node list is expected for Fargate and is `N/A`, not a failed read and not a clean node
assessment.

A Fargate-only report must explicitly state:

> Node-level checks are N/A because this cluster has no customer-managed node groups.
> Fargate runtime, kubelet, operating system, and container runtime are AWS-managed.
> The score reflects observable profile, subnet, controller, and pod-level signals only.

If managed or self-managed node groups are detected, mark the report `HYBRID` and do not
apply the Fargate-only score without a separate node-group assessment.

## 2. Scoring Algorithm

Build the master finding table before calculating the score. Each finding maps to one
row below and has exactly one scoring home.

```text
score = 100

# 1 Breaking Changes, maximum 25
breaking = sum(HIGH=10, MEDIUM=4, LOW=2 per distinct applicable change type)
breaking = min(breaking, 25)

# 2 Deprecated APIs, maximum 20
# Count distinct API paths, after managedFields writer filtering.
deprecated = sum(removed-in-target=5, still-served=1 per API path)
deprecated = min(deprecated, 20)

# 3 Fargate Data Plane, maximum 20
fargate = 0
for each profile subnet with available IPs <= 15: fargate += 2
if control_plane_subnets_collectively_have_fewer_than_5_ips: fargate += 5
for each required Fargate profile not ACTIVE: fargate += 5
for each platform workload with no matching selector or persistent Pending state: fargate += 3
for each profile with only one usable AZ: fargate += 1
fargate = min(fargate, 20)

# 4 Platform Controller Compatibility, maximum 15
controller = sum(required INCOMPATIBLE/DEGRADED=5,
                 other INCOMPATIBLE=3,
                 UNKNOWN_VERIFIABLE or UNKNOWN_UNIDENTIFIED=2,
                 UPDATE_RECOMMENDED=1)
controller = min(controller, 15)

# 5 Workload Risks, maximum 10
# Use workload-risks.md's master table. HIGH sub-cap 8, MEDIUM sub-cap 4.
workload = min(min(high_points, 8) + min(medium_points, 4), 10)

# 6 AWS Upgrade Insights, maximum 10
insights = sum(ERROR=5, WARNING=2; suppress subjects already scored elsewhere)
insights = min(insights, 10)

# 7 Behavioral Changes, maximum 5
behavioral = sum(MEDIUM=2, LOW=1 for explicitly detected target changes)
behavioral = min(behavioral, 5)

# 8 Unsupported Current Version, maximum 15
unsupported = 15 if current_version_is_past_extended_support else 0

total = breaking + deprecated + fargate + controller + workload + insights + behavioral + unsupported
score = max(0, 100 - total)
```

For a Fargate-only cluster, node readiness, AL2/AMI, containerd, and Karpenter do not
produce score rows. Do not turn absence of EC2 nodes into a zero-point clean finding.

## 3. Hard Blockers

After arithmetic, cap the score at 59 when any of these is true:

1. The cluster status is not `ACTIVE`.
2. Control-plane subnets collectively have fewer than 5 available IP addresses.
3. A required Fargate profile is not `ACTIVE`.
4. A required platform controller is incompatible or degraded/failed.
5. An API removed in the target is actively used by a user-managed resource.
6. A target-version mandatory breaking change is present.

A low-IP profile subnet by itself is a MEDIUM warning unless collective control-plane
capacity is below 5. A single-AZ Fargate profile is a MEDIUM resilience warning, not a
control-plane blocker. A Pending pod is a blocker only when it is a required platform
controller or its cause prevents required upgrade validation; otherwise it is a
recommended action.

## 4. Required Report Structure

Produce the following top-level sections in exactly this order:

1. `# EKS Upgrade Readiness Assessment`
2. `## Readiness Score: ...`
3. `## Blockers`
4. `## Critical Actions`
5. `## Recommended Actions`
6. `## Informational Findings`
7. `## Unassessed`
8. `## Evidence`
9. `## Upgrade Plan`
10. `## AWS Reference Links`

Print the point-in-time caveat on every report. Add a scope caveat and a
`(partial — N categories unassessed)` marker when `## Unassessed` is non-empty. A partial
assessment cannot print READY; cap its displayed rating at GOOD.

`## Blockers` contains only the six hard-blocker classes above. Other HIGH findings go in
`## Critical Actions`; MEDIUM findings go in `## Recommended Actions`; LOW and INFO items
go in `## Informational Findings`.

## 5. Evidence Requirements

The score breakdown must include these rows:

| Category | Status | Deduction | Details |
|---|---|---:|---|
| Breaking Changes | assessed/unknown | -X | distinct applicable changes |
| Deprecated APIs | assessed/unknown | -X | API paths and writer filter |
| Fargate Data Plane | assessed/N/A/unknown | -X | profile, selector, Pending, subnet results |
| Platform Controller Compatibility | assessed/unknown | -X | six-controller inventory |
| Workload Risks | assessed/unknown | -X | names from master workload table |
| AWS Upgrade Insights | assessed/unknown | -X | insight IDs and suppression |
| Behavioral Changes | assessed/unknown | -X | explicit target changes |
| Unsupported Version | assessed/N/A/unknown | -X | support lifecycle |
| **Total** | | **-X** | **Score: XX%** |

Under `### Add-on Inventory`, include controller version, status, Fargate profile,
target minimum version, upgrade required, verdict, and source URL. Include the optional
unknown/unidentified table only when it has rows.

Under `### Fargate Profile Summary`, include profile name, status, execution role,
subnets, selectors, usable AZ count, and coverage findings.

Under `### Configuration Alignment`, include the live-versus-source table from
`configuration-drift.md`, the workspace revision when available, and every `DRIFT`,
`NOT FOUND`, or `UNKNOWN` item with its exact source path.

Under `### Workload Risk Summary`, include the complete master workload table before any
counts or findings. Cross-check every count against the listed workload names.

## 6. Upgrade Plan

The skill is assessment-only. Commands in the report are recommendations and must never
be executed by the skill. For Fargate, the plan is:

1. Resolve blockers and confirm the target exists and the one-minor upgrade path is valid.
2. Update required platform controllers and CRDs first, using each project's documented
   compatibility requirement and the normal deployment pipeline.
3. Confirm all required Fargate profiles are `ACTIVE`, selectors cover platform
   workloads, profile subnets have capacity, and required pods are Ready.
4. Upgrade the control plane with the approved change process:

   ```bash
   aws eks update-cluster-version --name <cluster> --kubernetes-version <target> --region <region>
   ```

5. Monitor the update and re-run the read-only assessment:

   ```bash
   aws eks describe-update --name <cluster> --update-id <update-id> --region <region>
   kubectl get pods -A
   kubectl get events -A --sort-by=.lastTimestamp
   ```

6. Update EKS-managed add-ons that are installed and validate controller webhooks,
   Ingress/Service behavior, External Secrets reconciliation, ACK resources, Grafana/
   Alloy telemetry, and KEDA scaling.
7. Record the live cluster version and component versions back into source-controlled
   infrastructure through the normal review process. The skill must not edit them.

Fargate has no node-group upgrade step. Do not print `update-nodegroup-version`, AMI
migration, cordon, drain, containerd, or Karpenter steps for a Fargate-only cluster.

Rollback and Velero backup strategy belong in the organization's runbook/change plan,
not in this assessment skill. The report may link to that approved procedure but must not
claim that a backup guarantees rollback of the EKS control plane or managed Fargate
runtime.

## 7. Score Reconciliation

The headline score must equal `100 - sum(capped category deductions)` unless a hard
blocker caps it at 59. Every master finding must appear in exactly one action or
informational section. Every UNKNOWN category must appear in `## Unassessed` and must
have no deduction row contribution.
