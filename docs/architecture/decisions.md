# Locked Decisions — Phase 1 Final

> **Status: LOCKED.** These five decisions were open at the end of Phase 1 design. They are now resolved against official documentation and are no longer open questions.
> **No AWS resource has been created. No credential exists. No money has been spent.**

**Date locked:** 18 September 2026

| # | Decision | Locked value |
|---|---|---|
| 1 | AWS region | **ap-south-1 (Mumbai)** |
| 2 | Base AMI | **Ubuntu Server 24.04 LTS** |
| 3 | Wazuh sizing | **4 vCPU / 8 GiB / 50 GB gp3** |
| 4 | Quarantine egress | **TCP 443 outbound only, zero inbound** |
| 5 | AWS log ingestion | **S3 bucket polling** (`aws-s3` wodle) |

---

## Decision 1 — AWS region: `ap-south-1` (Mumbai)

### Evidence

**[VERIFIED FACT]** AWS pricing varies by region. Every AWS pricing page used in this design presents rates per region ([EC2 pricing](https://aws.amazon.com/ec2/pricing/), [EBS pricing](https://aws.amazon.com/ebs/pricing/), [VPC pricing](https://aws.amazon.com/vpc/pricing/)).

**[VERIFIED FACT]** AWS documents a set of core services "included in all Region launches" ([AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)). The services this lab uses — EC2, EBS, S3, VPC, CloudTrail, CloudWatch, Systems Manager — are foundational services of that kind.

**[ASSUMPTION]** Every service in this design is available in `ap-south-1`. The live services grid is rendered dynamically and could not be read programmatically, so this is stated as an assumption with a confirmation step, not as a verified fact. **Confirm in the console during Phase 2 Phase 0.**

### Reasoning

**[DESIGN DECISION] `ap-south-1` is locked.**

1. **Latency is felt directly here.** The Wazuh and Shuffle dashboards are used interactively through an SSM port-forwarding tunnel. The operator is in Mumbai. Every click in a SIEM dashboard pays the round trip, and a laggy dashboard makes triage practice worse at exactly the thing this lab exists to practise.

2. **The cost difference is not material at this scale.** Lower-cost regions such as `us-east-1` exist, and for a continuously-running production fleet the delta would matter. This lab runs three small instances for a few hours a week, with compute stopped between sessions. The saving from a cheaper region is a fraction of an already small bill, set against a permanently worse interactive experience.

3. **Single-region discipline.** One region is the single most effective defence against resources being missed at teardown — a resource in a forgotten region is invisible and bills indefinitely. Choosing the region the operator naturally works in reduces the chance of accidentally launching elsewhere.

4. **No data residency constraint either way.** The lab holds no client, personal or production data, so residency does not force the choice — but neither does it argue against the local region.

**[DESIGN DECISION]** No dollar figures or regional price deltas are stated. Publishing a number not verified for the chosen region on the day would be inventing a metric. The operator confirms actual cost from the first bill, recorded in [`../deployment/cost-model.md`](../deployment/cost-model.md).

### Consequence

All resources are created in `ap-south-1` and nowhere else. **Teardown verification still checks every region** — the discipline exists precisely because mistakes happen.

---

## Decision 2 — Base AMI: Ubuntu Server 24.04 LTS

### Evidence

**[VERIFIED FACT]** Wazuh's supported operating systems for the all-in-one deployment include **Amazon Linux 2, Amazon Linux 2023, Ubuntu 16.04 / 18.04 / 20.04 / 22.04 / 24.04, RHEL 7–10 and CentOS Stream 10** ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)). **Both candidates are supported.**

**[VERIFIED FACT]** SSM Agent is preinstalled on both. AWS lists, verbatim, among AMIs with SSM Agent preinstalled: "Amazon Linux 2023 (AL2023)" and "Ubuntu Server 18.04, 20.04, 22.04 LTS, 24.04 LTS, 24.0, and 25.04" ([AMIs with SSM Agent preinstalled](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)).

**[VERIFIED FACT]** AWS notes the preinstalled version "may not be the latest available version" and recommends running the latest SSM Agent.

### Reasoning

**[DESIGN DECISION] Ubuntu Server 24.04 LTS is locked.**

The two strongest candidate criteria turned out to be **neutral**, and this is worth stating plainly because an earlier draft of this design assumed otherwise:

| Criterion | AL2023 | Ubuntu 24.04 LTS | Decisive? |
|---|---|---|---|
| Wazuh support | Supported | Supported | **No — neutral** |
| SSM Agent preinstalled | Yes | Yes | **No — neutral** |
| Security update lifecycle | Long-term AWS support | 5-year LTS standard support | No — both adequate |
| Docker for Shuffle | Available | Available | Marginal |
| Community documentation for Wazuh and Docker | Good | Broader | **Slight edge to Ubuntu** |
| One OS across all three hosts | Either works | Either works | Neutral |

**The earlier draft claimed AL2023 had an SSM advantage. That was wrong, and AWS documentation says so.** Recording the correction matters more than quietly switching.

With the technical criteria neutral, the decision rests on the remaining honest differentiator: **troubleshooting speed**. This is a learning lab. When something breaks — and Phase 2 assumes it will — the platform with the widest body of matching community material for both Wazuh and Docker costs less time to unstick. That is a real operational cost, not a popularity argument.

**Why not AL2023:** it is a legitimate choice with genuine advantages — AWS-native tuning, deterministic `dnf` updates, and first-party support. Nothing here is a criticism of it. It simply loses a close call on documentation breadth once SSM and Wazuh support proved neutral.

### Consequence

**All three instances run Ubuntu Server 24.04 LTS.** One patch baseline, one package manager, one set of commands. The SSM Agent is updated to current on first boot rather than assumed to be current.

---

## Decision 3 — Wazuh sizing: 4 vCPU / 8 GiB / 50 GB gp3

### Evidence

**[VERIFIED FACT]** Wazuh's documented single-host requirements ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)):

| Agents | CPU | RAM | Storage (90 days) |
|---|---|---|---|
| **1–25** | **4 vCPU** | **8 GiB** | **50 GB** |
| 25–50 | 8 vCPU | 8 GiB | 100 GB |
| 50–100 | 8 vCPU | 8 GiB | 200 GB |

### Reasoning

**[DESIGN DECISION] 4 vCPU / 8 GiB locked. The earlier 2 vCPU proposal is withdrawn.**

The Phase 1 design proposed 2 vCPU as a deliberate, documented deviation, on the argument that this lab runs ~3 agents rather than 25. That reasoning was not unsound, but it was the wrong trade for a **first** deployment:

- A deviation from documented minimums makes every subsequent failure ambiguous. When the indexer struggles, is it the configuration, the rule, or the undersized host? Removing that variable is worth more than the saving.
- The documented figure is what Wazuh will support and what its own troubleshooting material assumes.
- **Stability over a small saving is the correct priority for a first build.** Optimisation belongs after something is known to work, not before.

The instance is stopped between sessions regardless, so the marginal cost of the larger size applies only to hours actually in use.

**[DESIGN DECISION]** Acceptance criterion **AC-33 is rewritten**: it no longer validates a deviation. It now confirms the deployment performs acceptably at the documented specification — and, if it does, records a baseline against which a *later, evidence-based* downsizing experiment could be run.

### Locked configuration

| Item | Value | Basis |
|---|---|---|
| vCPU | 4 | Wazuh documented minimum for 1–25 agents |
| RAM | 8 GiB | Wazuh documented minimum |
| Storage | 50 GB gp3, baseline IOPS | Wazuh documented figure for 90-day retention |
| Expected agents | **3** (`wazuh-aio` self-monitoring, `shuffle`, `target-01`) | Lab design |
| Retention assumption | **Shorter than the 90 days the 50 GB figure is based on**, so storage carries substantial headroom | Design |

**[DESIGN DECISION]** Storage stays at the documented 50 GB rather than being trimmed. A full disk stops indexing, which is a silent detection outage — the failure mode this lab exists to teach people to notice. Undersizing storage to save a small amount is the wrong risk to take.

---

## Decision 4 — Quarantine security group: 443 egress only

### Evidence

**[VERIFIED FACT]** Session Manager requires **outbound HTTPS (port 443)** to three endpoints ([Session Manager prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)):

- `ssm.<region>.amazonaws.com`
- `ssmmessages.<region>.amazonaws.com`
- `ec2messages.<region>.amazonaws.com`

**[VERIFIED FACT]** The prerequisites documentation states no inbound port requirement. Session Manager works entirely on outbound-initiated connections.

### The trade-off, stated plainly

A quarantine group with **no rules at all** is the strongest containment — and it also severs SSM, making the host unreachable. The analyst can then do nothing but terminate it, which destroys the volatile evidence that explains how it was compromised. Perfect containment and zero forensics is not a win in a lab whose purpose is investigation practice.

### Reasoning

**[DESIGN DECISION] `sg-quarantine` is locked as:**

| Direction | Rule | Reason |
|---|---|---|
| **Inbound** | **None. Zero rules.** | Total inbound containment; no lateral movement into the host |
| **Outbound** | **TCP 443 only** | The minimum that keeps SSM alive for forensic access |
| Outbound | Nothing else — no 80, no 53, no intra-lab | Cuts lateral movement out, agent traffic, and every non-443 channel |

Applying it **replaces** the instance's existing security groups; it does not add to them. The host is instantly cut off from every other lab host in both directions.

### Residual risk — accepted, not hidden

**443 outbound is a viable command-and-control channel.** Leaving it open means a compromised host could, in principle, continue talking to an attacker over HTTPS.

This is accepted for a lab, for three reasons: the "attacker" is a documented simulation under the operator's control; the alternative costs the investigation; and outbound traffic from the quarantined host remains visible in VPC Flow Logs, so the channel is monitored even while open.

**[DESIGN DECISION]** The hardened alternative is documented rather than adopted: place SSM VPC interface endpoints in the VPC and restrict quarantine egress to those endpoints' private addresses, closing general internet egress entirely. This is the correct production pattern. It is not adopted here because interface endpoints carry an hourly charge per endpoint and three are required — a continuous cost for a lab used a few hours a week. **If this lab ever handles anything that is not synthetic, adopt the endpoint model.**

### Operational rule

Applying quarantine is a **containment action and requires human approval**, like every other disruptive action. Before applying it, capture volatile state where the situation allows — process list and network connections — because isolation drops what was in flight. See [`../incident-response/incident-response-process.md`](../incident-response/incident-response-process.md).

---

## Decision 5 — AWS log ingestion: S3 bucket polling

### Evidence

**[VERIFIED FACT]** Wazuh's CloudTrail documentation configures ingestion by pointing the `aws-s3` wodle at an S3 bucket ([Wazuh — CloudTrail](https://documentation.wazuh.com/current/cloud-security/amazon/services/supported-services/cloudtrail.html)):

```xml
<wodle name="aws-s3">
  <bucket type="cloudtrail">
    <name>WAZUH_AWS_BUCKET</name>
    <aws_profile>default</aws_profile>
  </bucket>
</wodle>
```

**[VERIFIED FACT]** The IAM permissions that page specifies, following least privilege:

- Read-only: `s3:GetObject`, `s3:ListBucket`
- With log deletion: adds `s3:DeleteObject`

**[VERIFIED FACT]** Wazuh polls the bucket at a configurable interval rather than receiving pushed events.

### Reasoning

**[DESIGN DECISION] S3 bucket polling is locked.**

| Criterion | S3 polling | SQS-based |
|---|---|---|
| Setup complexity | Bucket + wodle config | Adds queue, bucket notifications, queue policy |
| Additional AWS resources | **None** | SQS queue |
| IAM permissions | `s3:GetObject`, `s3:ListBucket` | Adds SQS receive/delete |
| Cost | No new resource | Another billable service |
| Home-lab suitability | **Fits exactly** | Built for volume this lab will not produce |
| Maintenance | One configuration block | Queue depth, dead-letter handling, notification config |
| Scalability | Adequate to this volume | Better at high volume |
| Teardown | One fewer resource to remove | One more thing to miss |

Polling wins on every criterion that matters at this scale. SQS solves a latency-and-volume problem this lab does not have, at the cost of an extra resource, extra permissions, extra failure modes and an extra teardown step.

### Accepted trade-off

**Ingestion latency is bounded by the poll interval.** A CloudTrail event will not appear in Wazuh the instant it occurs. This is fine for a lab that measures triage quality rather than seconds-to-detect, but it is a real property and is stated in the acceptance criteria so no one later reports a detection time that is really a poll interval.

**[DESIGN DECISION]** `role-wazuh` receives **`s3:GetObject` and `s3:ListBucket` only**, scoped to the log bucket and its prefixes. **`s3:DeleteObject` is deliberately excluded** — Wazuh has no need to delete log objects, and retention is handled by S3 lifecycle rules. Granting delete on the audit log bucket to the component that reads it would let a compromise of that component destroy the evidence trail.

### Consequence

No SQS queue is created. `role-shuffle` still receives no AWS API permissions at all.

---

## What this changes

| Document | Change |
|---|---|
| [`final-aws-architecture.md`](final-aws-architecture.md) | Region locked; sizing corrected to 4 vCPU |
| [`network-architecture.md`](network-architecture.md) | Quarantine group defined |
| [`wazuh-design.md`](wazuh-design.md) | Sizing, AMI and ingestion locked |
| [`../deployment/aws-prerequisites.md`](../deployment/aws-prerequisites.md) | Region confirmation step |
| [`../security/iam-design.md`](../security/iam-design.md) | SQS permissions question resolved; delete excluded |
| [`../security/network-security.md`](../security/network-security.md) | Quarantine egress model |
| [`../security/logging-and-monitoring.md`](../security/logging-and-monitoring.md) | Ingestion method locked |
| [`../testing/acceptance-criteria.md`](../testing/acceptance-criteria.md) | AC-33 rewritten |

## Still open — deliberately

Not part of these five, and not blocking Phase 2:

- **[OPEN QUESTION]** Wazuh → Shuffle integration: webhook push versus Shuffle polling the Wazuh API. Decided at Phase 6 when both are running.
- **[OPEN QUESTION]** Tightening general egress to specific prefix lists. Deferred; low value while there is no inbound path.

## Sources

- [Wazuh quickstart — requirements, supported OS, installation](https://documentation.wazuh.com/current/quickstart.html)
- [Wazuh — Amazon CloudTrail ingestion](https://documentation.wazuh.com/current/cloud-security/amazon/services/supported-services/cloudtrail.html)
- [AWS — Session Manager prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
- [AWS — AMIs with SSM Agent preinstalled](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)
- [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
- [Amazon EC2 pricing](https://aws.amazon.com/ec2/pricing/) · [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/) · [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
