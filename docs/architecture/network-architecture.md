# Network Architecture — PLANNED

> **PLANNED ARCHITECTURE. No VPC, subnet, security group or route table has been created.**

## Topology

```
                    ┌─────────────────────────────────┐
                    │  VPC  10.0.0.0/16               │
                    │                                 │
                    │  ┌───────────────────────────┐  │
                    │  │ Public subnet 10.0.1.0/24 │  │
                    │  │ (single AZ)               │  │
                    │  │                           │  │
                    │  │  wazuh-aio    (sg-wazuh)  │  │
                    │  │  shuffle      (sg-shuffle)│  │
                    │  │  target-01    (sg-target) │  │
                    │  └─────────────┬─────────────┘  │
                    │                │                │
                    │        ┌───────▼────────┐       │
                    │        │ Route table    │       │
                    │        │ 0.0.0.0/0 ─▶IGW│       │
                    │        └───────┬────────┘       │
                    │                │                │
                    │        ┌───────▼────────┐       │
                    │        │ Internet GW    │       │
                    │        └───────┬────────┘       │
                    └────────────────┼────────────────┘
                                     │  EGRESS ONLY
                                     ▼
                    AWS SSM · package repos · threat intel API
```

## Why "public subnet" does not mean "publicly reachable"

**[DESIGN DECISION]** The subnet is public in the routing sense — it has a route to an internet gateway — because the instances need outbound access for SSM, package updates and threat intelligence lookups.

They are not *reachable* from the internet, because **no security group permits any inbound traffic**. A route out does not create a path in: reachability requires both a route and a permitting rule, and this design removes the second.

This achieves what a private subnet plus NAT gateway would achieve, without the continuous NAT charge. See [`final-aws-architecture.md`](final-aws-architecture.md) §3.2.

## Addressing

| Component | CIDR | Notes |
|---|---|---|
| VPC | `10.0.0.0/16` | RFC1918; generous costs nothing |
| Public subnet | `10.0.1.0/24` | Single AZ — a lab needs no AZ redundancy |
| *(reserved)* | `10.0.2.0/24` | Unallocated, for a future private subnet |

**[ASSUMPTION]** `10.0.0.0/16` does not overlap any network the operator may later need to reach. Validate before deployment.

**[DESIGN DECISION]** Single availability zone. Multi-AZ adds cross-AZ data transfer charges and demonstrates nothing relevant here.

## Security groups

Three groups, one per role. **Every group starts with an empty inbound rule set.**

### `sg-wazuh`

| Direction | Rule | Reason |
|---|---|---|
| Inbound | **(none from any CIDR)** | Access via SSM only; dashboard via SSM port forwarding |
| Inbound | Agent ports, source = `sg-target` | See open question below |
| Outbound | HTTPS 443 → `0.0.0.0/0` | SSM, package repositories, Wazuh updates |

**[OPEN QUESTION]** Wazuh agents connect to the manager on TCP 1514, with 1515 for enrolment. The rule will reference **`sg-target` as the source security group, never a CIDR** — meaning nothing outside the lab can satisfy it. Recorded as open only because the exact port set must be confirmed against Wazuh documentation at deployment time.

### `sg-shuffle`

| Direction | Rule | Reason |
|---|---|---|
| Inbound | **(none)** | UI reached by SSM port forwarding only |
| Outbound | HTTPS 443 → `0.0.0.0/0` | SSM, Docker images, threat intelligence APIs |
| Outbound | → `sg-wazuh` | Query the Wazuh API for alert context |

**[VERIFIED FACT]** Shuffle listens on 3001 (HTTP), 3443 (HTTPS), 5001 (backend REST API) and 9200 (Opensearch) ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)).

**None of these is exposed to the internet.** Port 9200 in particular must never be reachable — an exposed search index is a data-disclosure incident, and a well-known way self-hosted stacks leak.

### `sg-target`

| Direction | Rule | Reason |
|---|---|---|
| Inbound | **(none from any CIDR)** | |
| Inbound | From `sg-target` only | Lateral-movement simulation, scoped to the lab |
| Outbound | HTTPS 443 → `0.0.0.0/0` | SSM, package updates |
| Outbound | → `sg-wazuh` on agent port | Send telemetry |

### `sg-quarantine` — **LOCKED**

Applied to isolate a host during incident response. It **replaces** the instance's security groups; it does not add to them.

| Direction | Rule | Reason |
|---|---|---|
| **Inbound** | **None. Zero rules.** | Total inbound containment; no lateral movement into the host |
| **Outbound** | **TCP 443 only** | The minimum that keeps SSM alive for forensic access |
| Outbound | Nothing else — no 80, no 53, no intra-lab | Cuts lateral movement out and every non-443 channel |

**[VERIFIED FACT]** Session Manager requires outbound HTTPS 443 to `ssm`, `ssmmessages` and `ec2messages` regional endpoints, and requires no inbound port ([Session Manager prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)).

**[DESIGN DECISION]** A quarantine group with no rules at all would be stronger containment — and would sever SSM, leaving the analyst able only to terminate the host, which destroys the volatile evidence explaining how it was compromised. **Residual risk accepted and recorded:** 443 egress is a viable C2 channel, mitigated here by the traffic remaining visible in VPC Flow Logs and the "attacker" being a controlled simulation. The hardened alternative — SSM VPC interface endpoints with egress restricted to them — is documented in [`decisions.md`](decisions.md) §4 and not adopted because three endpoints carry continuous hourly charges.

Applying quarantine is a containment action and **requires human approval**.

## Rules that will never exist in this lab

- `0.0.0.0/0` inbound, on any port, for any duration — "temporarily" included
- Inbound 22 (SSH) — Session Manager replaces it entirely
- Inbound 3389 (RDP)
- Inbound to the Wazuh dashboard
- Inbound 3001 / 3443 / 5001 / **9200** to Shuffle
- Any rule without a description

**[DESIGN DECISION]** Every rule carries a description explaining why it exists. An undocumented rule outlives its reason, and nobody dares remove it.

## Reaching the dashboards

```
Analyst workstation
   │  aws ssm start-session --target <instance-id> \
   │     --document-name AWS-StartPortForwardingSession \
   │     --parameters '{"portNumber":["<port>"],"localPortNumber":["<local>"]}'
   ▼
AWS Systems Manager   (IAM-authenticated, MFA, CloudTrail-logged)
   ▼
SSM agent on instance (outbound-initiated tunnel)
   ▼
Dashboard bound to 127.0.0.1 — never to a public interface
```

The browser connects to `localhost`. Nothing listens on a public interface at any point.

## Egress

**[DESIGN DECISION]** Outbound restricted to HTTPS 443 plus the intra-lab flows above, rather than left wide open. The lab needs AWS APIs, package repositories and one threat intelligence endpoint.

**[OPEN QUESTION]** Whether to tighten egress to specific prefix lists. Deferred — meaningful maintenance burden for modest gain when there is no inbound path at all.

## Flow logging

**[DESIGN DECISION]** VPC Flow Logs at VPC level, delivered to S3 with lifecycle expiry. Flow log volume scales with traffic and is the one logging cost that is not flat; scope is set deliberately and reviewed in the first week.

## Sources

- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
