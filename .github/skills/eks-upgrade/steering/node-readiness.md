# Fargate Data-Plane Readiness

## Purpose

Assess the parts of a Fargate-only EKS data plane that an upgrade can actually affect.
Fargate capacity, the Fargate runtime, kubelet version, operating system image, and
container runtime are AWS-managed. Do not apply EC2 node-group, AMI, containerd,
Karpenter, or node-drain checks to a Fargate-only cluster.

## Fargate Detection

Confirm the cluster's compute model before running this section:

1. Describe the cluster and record `resourcesVpcConfig.subnetIds` and
   `resourcesVpcConfig.securityGroupIds`.
2. List Fargate profiles:

   ```bash
   aws eks list-fargate-profiles --cluster-name <cluster> --region <region>
   aws eks describe-fargate-profile --cluster-name <cluster> \
     --fargate-profile-name <profile> --region <region>
   ```

3. List managed node groups only as a detection check. If any are present, the cluster
   is hybrid rather than Fargate-only and the EC2 node-readiness process is required for
   those node groups:

   ```bash
   aws eks list-nodegroups --cluster-name <cluster> --region <region>
   ```

4. Inspect nodes only to identify hybrid compute or unexpected capacity. An empty or
   near-empty node list is expected for Fargate-only clusters; it is not evidence that
   Fargate is unhealthy.

## Checks to Execute

### F1 — Profile Inventory and Lifecycle

For every profile, record:

- profile name and status; anything other than `ACTIVE` is a high-priority finding;
- pod execution role ARN;
- subnets assigned to the profile;
- namespace and label selectors;
- whether selectors overlap or leave platform workloads unmatched.

An `ACTIVE` profile is necessary but not sufficient. A workload can still remain
Pending when no profile selector matches it, when the profile's subnets lack IPs, or
when the pod execution role cannot pull images or write required logs.

### F2 — Fargate Scheduling Coverage

Build a table of platform workloads and compare each namespace/label pair with every
Fargate profile selector. Include the six registered platform controller families and
any AWS-managed add-ons that run as pods.

```bash
kubectl get deployments,statefulsets,daemonsets -A -o wide
kubectl get pods -A --field-selector=status.phase=Pending -o wide
```

Flag:

- Pending pods with scheduling events naming a missing Fargate profile;
- platform workloads with no matching selector;
- overlapping selectors that make placement ambiguous;
- DaemonSets that cannot run on Fargate. DaemonSets are not a Fargate replacement
  pattern and must be treated as hybrid/unsupported until verified.

These are workload or platform findings, not node-version findings.

### F3 — Subnet Capacity for Fargate ENIs

Fargate tasks consume VPC IP addresses. Check every subnet used by the cluster and
every Fargate profile:

```bash
aws ec2 describe-subnets --subnet-ids <subnet-id-1> <subnet-id-2> ... \
  --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount,CIDR:CidrBlock}' \
  --output table
```

Use the following conservative interpretation:

| Condition | Severity | Meaning |
|---|---|---|
| Any profile subnet has fewer than 5 available IPs | MEDIUM | Fargate task placement and control-plane ENI placement have little headroom |
| Cluster control-plane subnets collectively have fewer than 5 available IPs | CRITICAL / blocker | EKS may reject the control-plane upgrade |
| A profile has only one usable AZ | MEDIUM | Loss of that AZ or subnet removes placement resilience |

The collective control-plane threshold is the only subnet hard blocker. Do not reuse
EC2 node surge or VPC CNI warm-pool calculations: Fargate has no customer-managed node
surge and no node-group rolling update.

### F4 — Pod-Level Data-Plane Signals

Use the workload-risk checks for readiness probes, resource requests, PDBs, graceful
shutdown, and Pending/Failed pods. These are the meaningful upgrade risks for Fargate
because pods are recreated rather than customer-managed nodes being drained.

Also inspect recent pod events and conditions:

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pods -A -o json
```

Do not infer a Fargate runtime or AMI upgrade action from `status.nodeInfo`; AWS owns
those details.

## Fargate-Only Result

When there are no managed or self-managed node groups and no Karpenter-managed nodes:

- Category 3 / node readiness is **N/A**, with zero deduction;
- do not create a node-group summary, AL2 finding, containerd finding, kubelet-skew
  finding, or Karpenter finding;
- report the Fargate profile table, profile coverage, Pending pods, and subnet capacity
  as the data-plane evidence;
- state explicitly that a high score covers only the checks that were assessable. It
  does not certify AWS-managed Fargate capacity or unobserved scheduling failures.

## Hybrid Cluster Boundary

If any managed or self-managed node group is present, stop treating the cluster as
Fargate-only. Assess Fargate profiles with this file and assess the EC2 data plane with
a separate node-group procedure. Do not silently merge the two scoring models.
