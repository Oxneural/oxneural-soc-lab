# Network Security — PLANNED

> **PLANNED. No network control has been implemented.**

Implementation detail for the network design in [`../architecture/network-architecture.md`](../architecture/network-architecture.md).

## The governing rule

**No security group in this lab permits inbound traffic from any internet CIDR, on any port, for any length of time.**

Not "restricted to my home IP". Not "just while I set it up". None.

Why the stricter rule rather than an IP allowlist:

| IP allowlist | Zero inbound |
|---|---|
| Breaks when the ISP changes the address | Nothing to break |
| Usual fix is widening the rule | No rule to widen |
| One typo exposes the service | No exposure path exists |
| Access not individually authenticated | Every session is IAM-authenticated and logged |

**[DESIGN DECISION]** The stricter rule is also the easier one to live with. There is no operational pressure to loosen it, because nothing depends on it being loose.

## Access path

**[VERIFIED FACT]** Session Manager carries no additional charge on EC2 ([AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)).

```
Analyst ──▶ AWS Systems Manager ──▶ SSM agent (outbound tunnel) ──▶ instance
             IAM + MFA, CloudTrail-logged
```

Dashboards are reached by SSM port forwarding to `localhost`. Nothing binds to a public interface.

## Rules permitted inside the lab

Intra-lab rules reference **source security groups, never CIDRs**. A source-security-group rule cannot be satisfied by any address outside the VPC, so it cannot accidentally become internet-facing.

| From | To | Port | Purpose |
|---|---|---|---|
| `sg-target` | `sg-wazuh` | Wazuh agent ports | Telemetry |
| `sg-shuffle` | `sg-wazuh` | Wazuh API | Alert context lookup |
| `sg-target` | `sg-target` | Scenario-specific | Lateral-movement simulation |

## Egress

**[DESIGN DECISION]** Restricted rather than open. The lab needs HTTPS to AWS APIs, package repositories and one threat intelligence endpoint.

| From | Destination | Port | Purpose |
|---|---|---|---|
| All | `0.0.0.0/0` | 443 | SSM, packages, threat intelligence |

Everything else denied. An outbound rule permitting all ports is a data-exfiltration path and, in a detection lab, an unmonitored blind spot.

## Services that must never be reachable

**[VERIFIED FACT]** Shuffle listens on 3001, 3443, 5001 and 9200 ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)).

| Port | Service | Consequence if exposed |
|---|---|---|
| **9200** | Opensearch | **Data disclosure.** Unauthenticated search indexes are a well-documented way self-hosted stacks leak. |
| 5001 | Shuffle backend API | Workflow manipulation |
| 3001 / 3443 | Shuffle UI | Administrative access |
| Wazuh dashboard | Wazuh | Full visibility into the environment |
| 1514 / 1515 | Wazuh agent | Log injection, false telemetry |

None is exposed in this design. All are reached over SSM.

## Network segmentation

**[DESIGN DECISION]** Single subnet, segmentation by security group rather than by subnet.

Honest reasoning: at three instances, subnet-level segmentation adds routing complexity and, if done with private subnets, a NAT gateway — with its continuous charge — for no meaningful additional control. Security groups are stateful instance-level firewalls and provide the isolation that matters at this scale.

**[ASSUMPTION]** This remains appropriate at this size. If the lab grows to include an untrusted or deliberately vulnerable host, revisit — that host belongs in its own subnet with no route to the SIEM.

## Isolation from everything else

- No VPC peering
- No VPN connection
- No Transit Gateway
- No connection to any personal, corporate or client network
- No shared credentials with any other environment

**Simulated attack traffic never leaves the lab VPC. No third-party system is scanned, probed or connected to.**

## Containment capability

For the isolate-host response action: replace the instance's security group with a quarantine group having **no rules at all** — no inbound, no outbound.

**[DESIGN DECISION]** Isolation is achieved by changing the security group, not by terminating the instance. The host stays alive and inspectable. Termination destroys the volatile evidence that explains how it was compromised.

**This action requires human approval.** See [`../incident-response/incident-response-process.md`](../incident-response/incident-response-process.md).

**[OPEN QUESTION]** Whether the quarantine group should permit SSM egress so the host remains reachable for forensics. Retaining access is operationally valuable but is a live channel on a suspect host. Decide before Phase 6 and record the reasoning.

## Verification

Before declaring deployment complete:

- [ ] Every security group listed; inbound rules from internet CIDRs: **zero**
- [ ] Port scan of each instance's public IP from outside AWS returns **no open ports**
- [ ] Wazuh dashboard not reachable from the internet
- [ ] Shuffle UI and port 9200 not reachable from the internet
- [ ] SSM session establishes successfully
- [ ] Port forwarding to each dashboard works
- [ ] Every rule has a description

The external port scan is against the lab's **own** instances — OxNeural-owned resources — and no other address.

## Sources

- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
