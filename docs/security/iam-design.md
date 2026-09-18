# IAM Design — PLANNED

> **PLANNED. No IAM user, role, policy or access key has been created.**

## Principle

Document permissions rather than granting `AdministratorAccess` and moving on. In a cloud environment identity is the perimeter: an attacker with valid credentials does not need an exploit, they need a policy attachment.

## Identity model

```
┌─────────────────────────────────────────────────────┐
│ ROOT                                                │
│  MFA enabled · NO access keys · break-glass only    │
│  Used for: account recovery, billing preferences,   │
│            closing the account. Nothing else.       │
└──────────────────────┬──────────────────────────────┘
                       │ creates, then steps back
                       ▼
┌─────────────────────────────────────────────────────┐
│ lab-admin  (human, MFA required)                    │
│  Builds and tears down the lab.                     │
│  Broad but NOT AdministratorAccess.                 │
└──────────────────────┬──────────────────────────────┘
                       │ creates
                       ▼
┌─────────────────────────────────────────────────────┐
│ lab-operator  (human, MFA required)                 │
│  Day-to-day operation. Start/stop instances,        │
│  open SSM sessions, read logs.                      │
│  CANNOT create IAM identities or delete logging.    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ INSTANCE ROLES  (no humans, no keys)                │
│  role-wazuh    · role-shuffle · role-target         │
└─────────────────────────────────────────────────────┘
```

## Root account

**[DESIGN DECISION]**

| Control | Setting |
|---|---|
| MFA | Enabled, hardware or authenticator app |
| Access keys | **None.** Delete any that exist. |
| Daily use | Never |
| Recovery codes | Stored offline |
| Monitoring | **Any root activity raises a critical alert** — see [`../detection/unauthorized-iam-activity.md`](../detection/unauthorized-iam-activity.md) rule E |

Root is for account recovery, billing preferences and closing the account. Nothing else ever.

## `lab-admin`

**Purpose:** build and tear down infrastructure.

**[DESIGN DECISION]** Broad but bounded. Specifically **not** `AdministratorAccess`, because the whole point of the exercise is to demonstrate that scoping is practical.

Permissions in scope:

| Area | Actions |
|---|---|
| EC2 | Instances, volumes, security groups, VPC, subnets, route tables, internet gateways |
| IAM | Create/manage the lab roles and policies named here |
| S3 | Create and manage the lab log bucket |
| CloudTrail | Create, configure and read the trail |
| CloudWatch | Alarms and log groups |
| SSM | Session Manager, Patch Manager, documents |
| Budgets / Cost Explorer | Read and configure |

**Explicitly denied, even for the admin:**

- Deleting or stopping CloudTrail logging — **[DESIGN DECISION]** an explicit `Deny` on `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`. If logging genuinely needs to be removed at teardown, that is a deliberate root-level act, not something a working identity can do by accident or under an attacker's control. An explicit `Deny` cannot be overridden by any `Allow`.
- Modifying the billing alarm or budget
- Creating IAM users outside the lab naming convention

**[OPEN QUESTION]** IAM Identity Center (SSO) is the better modern practice over long-lived IAM users. For a single-operator lab it adds setup overhead. Decision: start with an IAM user plus MFA; revisit if more than one person operates the lab.

## `lab-operator`

**Purpose:** run the lab day to day without the ability to reshape it.

| Allowed | Denied |
|---|---|
| Start / stop / describe instances | Create or delete IAM identities |
| Start SSM sessions and port forwarding | Modify security groups |
| Read CloudTrail, flow logs, S3 log objects | Delete logs or trails |
| Read CloudWatch alarms | Create access keys |
| Read cost data | Terminate instances |

**[DESIGN DECISION]** `lab-operator` deliberately **cannot terminate instances or modify security groups.** These are the two actions that most easily break the lab or silently open it up; they belong to the build identity, not the operating one. Day-to-day work does not need them.

## Instance roles — no keys, ever

**[DESIGN DECISION]** No EC2 instance receives an access key. Every instance uses an IAM role. There is no long-lived credential on any host to steal, and nothing that can be committed to git.

### `role-wazuh`

| Permission | Why | Scope |
|---|---|---|
| `AmazonSSMManagedInstanceCore` (AWS managed) | Session Manager and Patch Manager | AWS managed policy |
| `s3:GetObject`, `s3:ListBucket` | Read CloudTrail and flow log objects for ingestion | **Only** the lab log bucket ARN, only the relevant prefixes |

Nothing else. Wazuh reads logs; it does not need to write to S3, touch EC2, or read IAM.

**[DESIGN DECISION] LOCKED: S3 bucket polling.** No SQS queue is created, so no SQS permission is granted.

**[VERIFIED FACT]** Wazuh's CloudTrail documentation specifies, following least privilege, read-only access of `s3:GetObject` and `s3:ListBucket`, with `s3:DeleteObject` added only if Wazuh is to delete logs ([Wazuh — CloudTrail](https://documentation.wazuh.com/current/cloud-security/amazon/services/supported-services/cloudtrail.html)).

**[DESIGN DECISION]** `s3:DeleteObject` is **deliberately excluded**. Retention is handled by S3 lifecycle rules. Granting delete on the audit-log bucket to the component that reads it would let a compromise of Wazuh destroy the evidence trail — the exact outcome the CloudTrail tampering detection exists to catch.

See [`../architecture/decisions.md`](../architecture/decisions.md) §5.

### `role-shuffle`

| Permission | Why | Scope |
|---|---|---|
| `AmazonSSMManagedInstanceCore` | Session Manager, Patch Manager | AWS managed policy |

Nothing else.

**[DESIGN DECISION]** Shuffle gets **no AWS API permissions at all** in the initial build. It orchestrates and enriches; every AWS-side response action passes through a human approval gate and is executed by an identity with the necessary permission, not by the SOAR platform itself.

This is deliberate and worth stating plainly: granting a SOAR platform standing permission to disable identities or modify security groups creates exactly the automated-containment risk the approval gates exist to prevent. **If a future response action genuinely requires an AWS API call, it gets its own narrowly scoped role, reviewed separately.**

### `role-target`

| Permission | Why |
|---|---|
| `AmazonSSMManagedInstanceCore` | Session Manager, Patch Manager |

Nothing else. This host is a telemetry source and a simulation target. It has no reason to touch any AWS API.

## Access keys

**[DESIGN DECISION]** No long-lived access keys are created for any identity in this lab.

- Instances use roles
- Human CLI access uses short-lived credentials, MFA-backed
- If a key ever becomes genuinely necessary, it is documented here first, scoped minimally, given an expiry, and rotated on a schedule

**[VERIFIED FACT]** Access key creation is itself a monitored event — `CreateAccessKey` is a detection in [`../detection/unauthorized-iam-activity.md`](../detection/unauthorized-iam-activity.md) rule B, because creating a key is a standard persistence mechanism.

## MFA

Required on root, `lab-admin` and `lab-operator`. No exceptions.

**[DESIGN DECISION]** Policies requiring MFA for sensitive actions are preferred over trusting that MFA was used at sign-in.

## Naming convention

```
Users:     lab-admin · lab-operator
Roles:     role-wazuh · role-shuffle · role-target
Policies:  policy-<component>-<purpose>
```

Anything outside this convention is, by definition, not part of the lab and warrants investigation.

## Review

| Item | Cadence |
|---|---|
| Unused permissions (IAM Access Analyzer / last-accessed data) | Monthly while active |
| Access keys in existence | Monthly — the expected answer is zero |
| MFA on every human identity | Monthly |
| Full IAM cleanup | At teardown |

## Teardown

Order matters — see [`../deployment/teardown-plan.md`](../deployment/teardown-plan.md):

1. Detach and delete instance roles and their policies
2. Delete `lab-operator`, then `lab-admin`
3. Confirm no access key exists anywhere in the account
4. Root retains MFA — root is not deleted

## Related

[`network-security.md`](network-security.md) · [`secrets-management.md`](secrets-management.md) · [`logging-and-monitoring.md`](logging-and-monitoring.md)
