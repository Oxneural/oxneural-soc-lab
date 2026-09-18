# Security Architecture — PLANNED

> **PLANNED ARCHITECTURE. No security control described here has been implemented.**

## Threat model for the lab itself

A security lab is a target. It holds credentials, runs internet-connected services, and is operated intermittently — which is precisely the profile of a system that gets compromised and not noticed.

| Threat | Control |
|---|---|
| Administrative interface exposed and brute-forced | **Zero inbound rules**; access via SSM only |
| Credentials committed to GitHub | `.gitignore`, secret scanning, push protection, no long-lived keys |
| Lab left running and forgotten | Billing alarm, budget alerts, `TeardownAfter` tag, stop-when-idle discipline |
| Root account compromise | MFA, no root access keys, root unused for daily work |
| Over-permissioned identity | Least privilege; no `AdministratorAccess` on working identities |
| Simulation escaping the lab | Single isolated VPC, no peering, no VPN, disposable targets |
| Unpatched internet-facing host | SSM Patch Manager; hosts stopped when not in use |
| Log tampering hiding activity | CloudTrail monitored for `StopLogging` / `DeleteTrail` |

## Defence layers

```
 1. ACCOUNT      Dedicated lab account · MFA · no root daily use · budgets
 2. IDENTITY     Least-privilege IAM · instance roles · no long-lived keys
 3. NETWORK      Zero inbound rules · restricted egress · single isolated VPC
 4. ACCESS       SSM Session Manager · IAM-gated · CloudTrail-logged
 5. HOST         Patch Manager · file integrity monitoring · disposable targets
 6. DATA         Encryption at rest and in transit · lifecycle expiry · no real data
 7. DETECTION    CloudTrail · flow logs · host telemetry → Wazuh
 8. RESPONSE     Shuffle orchestration with mandatory human approval gates
 9. GOVERNANCE   Tagging · cost monitoring · written teardown
```

No layer is load-bearing alone. The design assumes any one of them can fail.

## Encryption

**[DESIGN DECISION]**

| Data | Control |
|---|---|
| EBS volumes | Encrypted at rest, AWS-managed keys — a customer-managed KMS key adds cost and no meaningful benefit at lab scale |
| S3 log buckets | Server-side encryption; public access blocked at bucket **and** account level |
| Wazuh agent → manager | TLS |
| Wazuh dashboard, Shuffle UI | HTTPS, reached over the SSM tunnel |
| SSM sessions | Encrypted in transit by AWS |

**[ASSUMPTION]** Default EBS encryption can be enabled account-wide, so no volume is ever created unencrypted by accident. Confirm at deployment.

## Access control

**[VERIFIED FACT]** Session Manager carries no additional charge on EC2 instances ([AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)).

**[DESIGN DECISION]** Session Manager is the only interactive access path. At no additional cost this gives:

- No inbound ports, therefore no SSH brute-force surface
- No SSH key pair to manage, lose, or accidentally commit
- IAM-controlled, MFA-gated access
- **Every session recorded in CloudTrail** — analyst access becomes a detectable event, which is itself useful in a detection lab

Full identity design: [`../security/iam-design.md`](../security/iam-design.md)

## Patching

**[VERIFIED FACT]** Patch Manager carries no additional charge on EC2 instances ([AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)).

**[DESIGN DECISION]** Patch baseline applied at session start rather than on a fixed schedule, because instances are stopped between sessions and a scheduled window would require the lab to be running.

## Simulation containment

- All simulation stays inside the single lab VPC
- No VPC peering, no VPN, no Transit Gateway, no connection to any other network
- Target hosts are disposable and rebuilt after malware-behaviour exercises
- **No real malware, no exploit code, no destructive payloads** — EICAR and reversible technique emulation only
- **No activity of any kind against any third-party system**

See [`../detection/malware-simulation.md`](../detection/malware-simulation.md).

## Monitoring the monitoring

**[DESIGN DECISION]** The lab watches its own controls. Specifically detected:

- CloudTrail `StopLogging` or `DeleteTrail` — **critical**
- MFA device deactivation — **critical**
- Root account usage — **critical**
- Security group modified to add an inbound rule — high
- IAM policy attachment granting broad permissions — high
- **Wazuh agent silence** — a host gone quiet looks identical to a host with nothing happening

That last is the failure mode most likely to matter and least likely to be noticed.

## Explicit non-goals

Stated so they are never assumed:

- This is **not** a production security architecture
- It does **not** demonstrate multi-account governance, SCPs or landing zones
- It provides **no** high availability or disaster recovery
- It is **not** a 24/7 monitored environment
- It holds **no** client, personal or production data, and never will

## Sources

- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
