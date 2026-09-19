# Final AWS Architecture — PLANNED

> **PLANNED ARCHITECTURE. No AWS resource has been created. Nothing in this document is deployed.**

**Classification:** OxNeural Internal Project · **Phase:** 1 — Design · **Status:** Awaiting approval

## Evidence labelling

Every substantive statement in this design carries one of four labels:

| Label | Meaning |
|---|---|
| **[VERIFIED FACT]** | Confirmed against official AWS, Wazuh or Shuffle documentation, cited |
| **[DESIGN DECISION]** | A choice made here, with the reasoning stated |
| **[ASSUMPTION]** | Believed true, not yet confirmed; must be validated before or during deployment |
| **[OPEN QUESTION]** | Not decided; requires an answer before deployment |

---

## 1. Design goals, in priority order

1. **Safety** — no exposure of administrative interfaces, no credentials in source control, no activity against third-party systems
2. **Cost containment** — the smallest footprint that demonstrates the capability; nothing always-on that need not be
3. **Demonstrable capability** — a full detection-to-response workflow, not a screenshot of a dashboard
4. **Reversibility** — every resource removable, verifiably, by a written procedure

Where these conflict, the higher one wins. Cost beats convenience; safety beats both.

## 2. The architecture in one picture

```
                    ANALYST (you)
                         │
                         │  AWS SSM Session Manager
                         │  (outbound-initiated; NO inbound ports)
                         ▼
        ┌────────────────────────────────────────────┐
        │  AWS ACCOUNT — dedicated lab                │
        │                                            │
        │  ┌──────────────────────────────────────┐  │
        │  │ VPC 10.0.0.0/16                      │  │
        │  │                                      │  │
        │  │  Public subnet 10.0.1.0/24           │  │
        │  │   ┌────────────────┐                 │  │
        │  │   │ EC2: wazuh-aio │◀── agent ──┐    │  │
        │  │   │ manager+index  │            │    │  │
        │  │   │ +dashboard     │            │    │  │
        │  │   └───────┬────────┘            │    │  │
        │  │           │ webhook             │    │  │
        │  │           ▼                     │    │  │
        │  │   ┌────────────────┐     ┌──────┴──┐ │  │
        │  │   │ EC2: shuffle   │     │ EC2:    │ │  │
        │  │   │ SOAR (docker)  │     │ target  │ │  │
        │  │   └───────┬────────┘     └─────────┘ │  │
        │  │           │                          │  │
        │  │  SG: ZERO inbound rules on all three │  │
        │  └───────────┼──────────────────────────┘  │
        │              │ egress only                 │
        │              ▼                             │
        │      Threat intelligence API               │
        │                                            │
        │  LOGGING SOURCES                           │
        │   CloudTrail ──▶ S3 ──┐                    │
        │   VPC Flow Logs ─▶ S3 ─┼──▶ pulled by Wazuh│
        │   Host/auth logs ─────▶ Wazuh agent        │
        │                                            │
        │  GUARDRAILS                                │
        │   AWS Budgets · CloudWatch billing alarm   │
        │   IAM least privilege · MFA · tagging      │
        └────────────────────────────────────────────┘
```

Detailed diagram specification: [`diagram-specification.md`](diagram-specification.md)

## 3. The two decisions that shape everything else

### 3.1 No inbound ports. Access is via SSM Session Manager

**[VERIFIED FACT]** AWS Systems Manager Session Manager is offered "at no additional charge" for Amazon EC2 instances ([AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)).

**[DESIGN DECISION]** No security group in this lab has **any** inbound rule. Not SSH from a home IP, not HTTPS to the Wazuh dashboard. Access is exclusively through Session Manager, which works by the SSM agent making an *outbound* connection to AWS.

Why this matters more than it first appears:

- The requirement "do not expose Wazuh or Shuffle administrative interfaces publicly" is satisfied structurally, not by a firewall rule that can be loosened "just for a minute"
- A home IP allowlist breaks whenever the ISP changes the address, and the usual fix is widening the rule
- The Wazuh and Shuffle dashboards are reached by **SSM port forwarding** to localhost — the browser connects to `localhost`, not to the internet
- Session Manager access is itself IAM-controlled, MFA-gated and logged in CloudTrail — which also makes analyst access a detectable event

This removes the single most common way a home lab becomes an incident.

### 3.2 No NAT gateway

**[VERIFIED FACT]** NAT Gateway is charged both hourly and per gigabyte processed: "you are charged for each 'NAT Gateway-hour' that your gateway is provisioned and available. Data processing charges apply for each gigabyte processed through the NAT gateway" ([Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)).

**[DESIGN DECISION]** No NAT gateway. Instances sit in a public subnet with an auto-assigned public IPv4 address for *outbound* traffic only (package updates, threat intelligence lookups, SSM). Inbound remains impossible because no security group permits it.

A NAT gateway is the most common cause of a surprise bill on a small AWS lab — it bills continuously whether or not the lab is being used. The alternative, a private subnet with SSM VPC interface endpoints, also carries hourly charges per endpoint and needs three of them.

**[VERIFIED FACT]** Public IPv4 addresses carry an hourly charge ([Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)).
**[DESIGN DECISION]** Use auto-assigned public IPs rather than Elastic IPs. An auto-assigned address is released when the instance stops; an unattached Elastic IP keeps billing. The address changes between sessions, which costs nothing here because nothing ever connects inbound.

## 4. Compute

**[VERIFIED FACT]** Wazuh documents, for a single-host all-in-one deployment monitoring 1–25 agents with 90 days of retention: **4 vCPU, 8 GiB RAM, 50 GB storage** ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)).

**[VERIFIED FACT]** Shuffle documents "a minimum of **4Gb of RAM** available. More RAM = better." CPU and disk are not stated ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)).

| Instance | Role | Proposed size | Rationale |
|---|---|---|---|
| `wazuh-aio` | Wazuh manager + indexer + dashboard | **4 vCPU / 8 GiB** | Wazuh's documented minimum for 1–25 agents — **LOCKED** |
| `shuffle` | Shuffle SOAR (Docker) | 2 vCPU / 4 GiB | Meets the documented RAM minimum |
| `target-01` | Telemetry source and simulation target | Smallest viable | Generates host logs; disposable |

**[DESIGN DECISION] LOCKED: 4 vCPU / 8 GiB.** An earlier draft proposed 2 vCPU as a documented deviation, arguing that ~3 agents is far below the 25-agent basis of Wazuh's table. That reasoning was not unsound, but it was the wrong trade for a first deployment: a deviation from documented minimums makes every subsequent failure ambiguous — configuration, rule, or undersized host? Stability beats a small saving on a first build, and the instance is stopped between sessions anyway.

Full reasoning: [`decisions.md`](decisions.md) §3.

**[DESIGN DECISION] LOCKED: region `ap-south-1` (Mumbai)** and **base AMI Ubuntu Server 24.04 LTS** on all three instances. See [`decisions.md`](decisions.md) §1 and §2. Exact instance types are selected at deployment against live availability; no price is committed here — see [`../deployment/cost-model.md`](../deployment/cost-model.md).

**Option B, if cost proves tighter than expected:** collapse `wazuh-aio` and `shuffle` onto one larger instance. It is cheaper to stop and start one instance than two, but it places the SOAR platform and the SIEM in the same failure and compromise domain — which undercuts part of what this lab exists to demonstrate. Recommended only if the two-instance model proves unaffordable in practice.

## 5. Storage

**[VERIFIED FACT]** EBS is "charged by the amount of GB you provision per month until you release the storage" ([Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)). Provisioned, not used — and until released, not until detached.

**[DESIGN DECISION]** gp3 for all volumes, at baseline performance. **[VERIFIED FACT]** gp3 includes a free baseline of 3,000 IOPS and 125 MB/s, charged only beyond that ([EBS pricing](https://aws.amazon.com/ebs/pricing/)). Provisioning nothing above baseline keeps the volume billed on capacity alone.

The practical consequence, stated plainly because it is the second most common cause of a surprise lab bill: **stopping an instance does not stop its EBS volume billing.** Only deleting the volume does.

## 6. Logging sources

**[VERIFIED FACT]** CloudTrail delivers "one copy of your ongoing management events to your Amazon S3 bucket for free by creating trails"; additional copies are charged ([AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)).

**[DESIGN DECISION]** Exactly one management-events trail, to S3, in the free allowance. No data events — they are charged per event and the lab has no use for S3 object-level auditing. No CloudTrail Insights.

| Source | Provides | Destination |
|---|---|---|
| CloudTrail (1 trail, management events) | API calls, console sign-in, IAM changes | S3, pulled by Wazuh |
| VPC Flow Logs | Accepted/rejected connections | S3, pulled by Wazuh |
| Wazuh agent | Host auth, process, file integrity | Wazuh manager directly |

**[VERIFIED FACT]** The VPC pricing page does not itself publish Flow Logs pricing; flow log delivery is billed through the destination service's ingestion charges.
**[DESIGN DECISION]** Deliver flow logs to **S3 rather than CloudWatch Logs**, and scope them deliberately rather than logging every interface at full verbosity. Flow log volume is the one logging cost here that scales with activity rather than sitting flat.

**[DESIGN DECISION]** S3 lifecycle expiry on both log prefixes, set to the retention window in the deployment plan. Without lifecycle rules, log storage grows forever and quietly becomes the largest line on the bill.

## 7. Services deliberately NOT used

Stating these prevents them creeping in later:

| Service | Why not |
|---|---|
| NAT Gateway | Continuous hourly + per-GB charge; avoidable by design (§3.2) |
| Elastic IP | Bills while unattached; auto-assigned IP suffices |
| Load balancer | Nothing needs load balancing or public ingress |
| Amazon GuardDuty | Useful in production; overlaps what this lab exists to build by hand, and adds cost |
| AWS Security Hub / Detective | Same reasoning |
| Amazon OpenSearch Service (managed) | Wazuh's bundled indexer is included; a managed cluster adds significant always-on cost |
| RDS | Nothing needs a managed relational database |
| Multi-AZ / HA | A lab does not need availability guarantees |
| AWS Config | Charged per configuration item; not needed for these detection scenarios |
| CloudTrail data events / Insights | Charged per event; no lab requirement |

The discipline here is the point: enterprise-grade is not the same as appropriate.

## 8. Related documents

[`network-architecture.md`](network-architecture.md) · [`security-architecture.md`](security-architecture.md) · [`data-flow.md`](data-flow.md) · [`diagram-specification.md`](diagram-specification.md)
[`../deployment/aws-prerequisites.md`](../deployment/aws-prerequisites.md) · [`../deployment/cost-model.md`](../deployment/cost-model.md) · [`../deployment/deployment-plan.md`](../deployment/deployment-plan.md) · [`../deployment/rollback-plan.md`](../deployment/rollback-plan.md) · [`../deployment/teardown-plan.md`](../deployment/teardown-plan.md)
[`../security/iam-design.md`](../security/iam-design.md) · [`../security/network-security.md`](../security/network-security.md) · [`../security/secrets-management.md`](../security/secrets-management.md) · [`../security/logging-and-monitoring.md`](../security/logging-and-monitoring.md)

## 9. Sources

- [Wazuh quickstart — all-in-one requirements](https://documentation.wazuh.com/current/quickstart.html)
- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
- [AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)
- [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)
- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
