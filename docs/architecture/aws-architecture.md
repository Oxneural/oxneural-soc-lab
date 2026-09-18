# AWS Architecture

**Status:** Design. **No AWS resource described here has been created.** Every item is a plan.

## Account

A dedicated AWS lab account, isolated from all other use. Root account has MFA and is not used for daily work.

## Planned resources

| Resource | Purpose | Sizing intent | Always-on |
|---|---|---|---|
| VPC | Network isolation for the lab | Single VPC, private + public subnet | Yes (no cost) |
| EC2 — Wazuh server | SIEM: manager, indexer, dashboard | Sized to the smallest instance that runs the stack reliably | No — stopped when not in use |
| EC2 — Shuffle server | SOAR orchestration | Small instance | No — stopped when not in use |
| EC2 — target host(s) | Generates telemetry; simulation target | Smallest viable | No |
| S3 | CloudTrail log destination | Lifecycle-expired | Yes (low cost) |
| CloudTrail | API and control-plane audit | All regions | Yes |
| VPC Flow Logs | Network telemetry | To CloudWatch or S3 | Yes |
| IAM roles/policies | Least-privilege access; detection source | — | Yes (no cost) |
| CloudWatch billing alarm | Cost control | — | Yes |

Instance types and exact sizing are deliberately not stated until measured during deployment. Guessing a size and publishing it is a claim, not a design.

## Network design intent

```
VPC  10.0.0.0/16              (lab — RFC1918, isolated)
 ├── Public subnet  10.0.1.0/24
 │     └── Management access, source-restricted
 └── Private subnet 10.0.2.0/24
       ├── Wazuh server
       ├── Shuffle server
       └── Target host(s)
```

Addresses shown are illustrative RFC1918 placeholders, not a deployed environment.

## Security group principles

- **No `0.0.0.0/0` on any management port.** SSH and dashboard access are restricted to a specific known source address.
- Lab-internal traffic permitted only between the specific security groups that need it, on the specific ports they need.
- Egress restricted to what the components actually require — package repositories and threat intelligence endpoints.
- Every rule carries a description explaining why it exists. An undocumented rule outlives its reason.

## IAM design intent

- Roles over long-lived access keys wherever possible
- Separate roles: administration, Wazuh service access, read-only analysis
- No wildcard administrative policy attached for routine work
- MFA on every human identity
- IAM activity is itself monitored — see [`../detection/unauthorized-iam-activity.md`](../detection/unauthorized-iam-activity.md)

## Logging design intent

| Source | Captures | Destination |
|---|---|---|
| CloudTrail | API calls, console sign-in, IAM changes | S3, ingested by Wazuh |
| VPC Flow Logs | Accepted/rejected connections | CloudWatch or S3, ingested by Wazuh |
| Wazuh agent | Authentication, process, file integrity | Wazuh manager |

Retention is set deliberately and documented in the deployment plan — long enough to investigate, short enough to control cost.

## Tagging

Every resource: `Project=oxneural-soc-lab`, `Owner=<owner>`, `Environment=lab`, `CostCentre=internal`, `TeardownAfter=<date>`.

Tagging is what makes teardown and cost attribution possible. Untagged resources are how labs become permanent bills.

## Cost posture

Compute is stopped when not in use. Always-on components are limited to those with negligible cost. Billing alarms are configured **before** any chargeable resource is created. See [`../deployment/cost-control.md`](../deployment/cost-control.md).

## Not in scope

Multi-account organisation, transit gateway, multi-region, high availability, managed SIEM services. All would be reasonable in production; none are needed to demonstrate the capability this lab is about.
