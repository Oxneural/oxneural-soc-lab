# infrastructure/

Infrastructure-as-code for the lab environment.

**Status: empty. No AWS resource has been created.** The design lives in [`../docs/architecture/aws-architecture.md`](../docs/architecture/aws-architecture.md).

## Planned layout

```
infrastructure/
├── network/       VPC, subnets, security groups
├── compute/       Wazuh, Shuffle, target hosts
├── logging/       CloudTrail, VPC Flow Logs, S3
└── iam/           roles and policies
```

## Rules for content in this directory

- **Never commit state files.** `*.tfstate` can contain secrets in plaintext and is git-ignored. Likewise `*.tfvars` — commit `*.tfvars.example` with placeholders instead.
- **No credentials, account IDs or real addresses** in any committed file.
- **No `0.0.0.0/0` on any management port.** Source-restricted access only. Every security group rule carries a description explaining why it exists.
- **Least privilege** in every IAM policy. No wildcard administrative policy for routine operation.
- **Tag everything** — `Project`, `Owner`, `Environment`, `CostCentre`, `TeardownAfter`. Tagging is what makes teardown verifiable and cost attributable.
- **Teardown must work.** Every phase is individually destroyable. If it cannot be cleanly torn down, that is a finding to fix before proceeding.

Cost controls and billing alerts are configured **before** any chargeable resource exists. See [`../docs/deployment/cost-control.md`](../docs/deployment/cost-control.md) and [`../docs/deployment/prerequisites.md`](../docs/deployment/prerequisites.md).
