# Platform Controller Compatibility

## Purpose

Validate the AWS-managed add-ons that are installed and the service-owned platform
controllers that run on Fargate. Compatibility is checked against the target Kubernetes
version using live authoritative sources. This file does not assess Karpenter, EC2 node
images, or container runtimes.

## 4.1 — AWS-Managed Add-ons

List and describe installed EKS-managed add-ons. Record name, installed version,
status, health issues, and the target-compatible versions:

```bash
aws eks list-addons --cluster-name <cluster> --region <region>
aws eks describe-addon --cluster-name <cluster> --addon-name <addon> --region <region>
aws eks describe-addon-versions --addon-name <addon> --kubernetes-version <target> \
  --query 'addons[0].addonVersions[?compatibilities[0].defaultVersion==`true`].addonVersion' \
  --output text
```

If no default version is returned, select the highest semver from the returned set. Do
not assume array position zero is newest. Apply these verdicts:

| Condition | Verdict | Severity |
|---|---|---|
| Installed version is in the target-compatible set and status is ACTIVE | COMPATIBLE | INFO |
| Compatible but behind the target default | UPDATE_RECOMMENDED | LOW |
| Not in the target-compatible set, or status is DEGRADED/FAILED | INCOMPATIBLE | HIGH |
| Read denied, failed, or partial | UNKNOWN_VERIFIABLE | MEDIUM; list under Unassessed when the read itself failed |

On Fargate, do not assume `vpc-cni`, `kube-proxy`, or node agents are present as
DaemonSets. Report only what is installed and what the cluster architecture requires.
Core DNS and storage/network integrations that actually run in the cluster still need
compatibility and health checks.

## 4.2 — Discover the Service-Owned Controllers

Inspect Deployments, StatefulSets, Pods, Helm metadata, and images:

```bash
kubectl get deployments,statefulsets -A -o json
kubectl get pods -A -o json
```

Match in this order:

1. `app.kubernetes.io/name` and `app.kubernetes.io/part-of`;
2. Helm chart labels;
3. container image repository and tag;
4. workload namespace and controller-specific labels.

The registry at `${SKILL_DIR}/data/oss_addon_registry.json` contains only:

- Argo CD;
- AWS Load Balancer Controller;
- External Secrets;
- ACK service controllers;
- Grafana Operator and Alloy Operator;
- KEDA.

For each match, record the exact project identity, namespace, controller version, image
tag, desired/ready replicas, pod status, matching Fargate profile, and identification
method. ACK, Grafana Operator, and Alloy Operator must remain distinct when the live
workload identifies them separately.

A workload in a platform namespace that looks controller-like but matches none of the
six families is `UNKNOWN_UNIDENTIFIED`, not an assumed compatible add-on. A normal user
application is not an add-on finding and remains in the workload-risk assessment.

## 4.3 — Verify Each Controller Against Its Own Source

For every identified controller:

1. Read the registry's `compatibility_url`.
2. If it lacks a definitive Kubernetes support statement, read `releases_url` for the
   installed release and target-version guidance.
3. Use web search only to locate the project's own documentation when those URLs fail.
4. Capture the source URL, the quoted support statement or matrix row, installed version,
   minimum version needed for the target, and whether an upgrade is required.
5. If the evidence is unavailable or ambiguous, use `UNKNOWN_VERIFIABLE` and do not infer
   compatibility from release recency or model knowledge.

Use exactly one verdict per controller:

| Verdict | Meaning | Score |
|---|---|---:|
| `COMPATIBLE` | Source confirms the installed version supports the target | 0 |
| `UPDATE_RECOMMENDED` | Installed version supports target but a newer supported version is advised | 1 |
| `INCOMPATIBLE` | Source says installed version does not support target | 5 and hard blocker for a required platform controller; 3 otherwise |
| `UNKNOWN_VERIFIABLE` | Controller is identified but evidence is unreachable or ambiguous | 2 |
| `UNKNOWN_UNIDENTIFIED` | Controller-shaped workload cannot be identified | 2 |

The target in this service's normal process is Kubernetes 1.35, but never substitute a
hard-coded minimum version for the live project source. Record the source date because
upstream compatibility pages change.

## 4.4 — Fargate-Specific Compatibility Checks

Compatibility is not only a version matrix. For each of the six controllers verify:

- its pods have an `ACTIVE` Fargate profile match;
- its desired and ready replica counts agree;
- it has no Pending, CrashLoopBackOff, or ImagePullBackOff pods;
- its webhooks have service endpoints and are not failing admission calls;
- its Service/Ingress integration is appropriate for Fargate;
- its resource requests fit the configured Fargate task sizes;
- any CRDs and conversion webhooks are served by the installed controller version.

A healthy version with a missing Fargate selector or broken webhook is still a platform
finding. Keep the version verdict and the runtime-health finding separate so the report
explains both causes.

## Output Contract

The report must include a controller inventory with current version, status, Fargate
profile, target minimum version, upgrade required, verdict, and source URL. Include a
separate table for unidentified controller-like workloads with kind, name, namespace,
images, labels, and why identification failed.

## Score Impact

The canonical category cap and hard-blocker rules are defined in
`steering/report-generation.md`:

- required platform controller incompatible or degraded: 5 points and hard blocker;
- other installed controller incompatible: 3 points;
- unknown verification or unidentified controller: 2 points;
- compatible but behind: 1 point;
- category maximum: 15 points.
