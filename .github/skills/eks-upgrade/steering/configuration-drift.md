# Source-Controlled Configuration Alignment

## Purpose

Compare the live, read-only EKS facts with source-controlled infrastructure definitions.
This check identifies drift; it never edits files or cluster state and it does not claim
that a missing local file proves the deployed configuration is wrong.

## Checks

1. Locate the repository's EKS definitions using read-only workspace search. Typical
   inputs are `eksctl-*.yaml`, Terraform, Helm values, and environment overlays.
2. Record the live values from `DescribeCluster` and Fargate profile descriptions:
   cluster name, Kubernetes version, region, VPC subnets, cluster security groups,
   Fargate profile names, profile subnets, selectors, execution roles, and installed
   managed add-ons.
3. Parse the checked-in definitions as structured YAML/HCL/JSON where tooling permits.
   Do not compare raw text when the same value can be represented in different formats.
4. Produce a table with one row per live value:

| Resource | Live value | Source value | Status | Evidence |
|---|---|---|---|---|
| Cluster Kubernetes version | 1.35 | 1.34 | DRIFT | cluster describe + file path |
| Fargate profile selectors | ... | ... | MATCH/DRIFT/NOT FOUND | profile describe + file path |

5. Treat a source value as `NOT FOUND` when no authoritative checked-in definition was
   found. Treat an unreadable or ambiguous source file as `UNKNOWN`, not as a match.
6. For every DRIFT or NOT FOUND row, name the exact file path and recommend a pull
   request or normal infrastructure reconciliation. The skill must not perform it.

## Minimum Acceptance Signal

The assessment passes this check only when every required live cluster-version and
Fargate-profile value is either `MATCH` or explicitly reviewed as an approved exception.
The report must include the source commit or workspace revision when available.
