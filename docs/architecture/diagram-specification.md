# Architecture Diagram Specification

> **Every diagram produced from this specification must carry the title suffix "— PLANNED ARCHITECTURE" until deployment actually occurs.** No diagram may depict a resource as existing when it does not.

## Purpose

A drawing specification, so any diagram of this lab is consistent regardless of who draws it or in which tool.

## Rules

1. **Label state honestly.** Title reads `OxNeural Cloud SOC Lab — PLANNED ARCHITECTURE`. When components are deployed, the title changes and undeployed components keep a dashed border.
2. **Dashed border = not deployed.** Solid = deployed and verified. At Phase 1, everything is dashed.
3. **No invented detail.** No instance IDs, no account numbers, no real IP addresses, no resource ARNs. RFC1918 and placeholder labels only.
4. **Show the approval gate.** Any diagram including the response path must draw the human approval gate explicitly. A diagram implying automated containment misrepresents the design.
5. **Show that there is no inbound path.** The absence of inbound access is the defining property of this architecture; a diagram that does not convey it has failed.

## Layers, top to bottom

### Layer 0 — Analyst

Single actor box: `SOC Analyst (authorised operator)`. One arrow down, labelled `SSM Session Manager — outbound-initiated, IAM + MFA, CloudTrail-logged`.

**Draw this arrow as entering the AWS boundary without crossing any inbound port.** Annotate: `No inbound ports open`.

### Layer 1 — AWS account boundary

Outer container: `AWS Account — dedicated lab`. Corner badges: `MFA enforced` · `AWS Budgets` · `CloudWatch billing alarm` · `CloudTrail enabled`.

### Layer 2 — VPC

Container inside the account: `VPC 10.0.0.0/16 (single AZ)`, containing one subnet box `Public subnet 10.0.1.0/24`.

Annotate the subnet: `Public route for EGRESS only — no inbound rule on any security group`.

### Layer 3 — Compute

Three instance boxes inside the subnet, each showing its security group:

| Box | Label | Security group annotation |
|---|---|---|
| `wazuh-aio` | Wazuh manager + indexer + dashboard | `sg-wazuh · 0 inbound from internet` |
| `shuffle` | Shuffle SOAR (Docker) | `sg-shuffle · 0 inbound from internet` |
| `target-01` | Telemetry source / simulation target | `sg-target · 0 inbound from internet` |

### Layer 4 — Logging sources

A band feeding upward into `wazuh-aio`:

```
CloudTrail (1 management trail) ──▶ S3 ──┐
VPC Flow Logs ───────────────────▶ S3 ──┼──▶ Wazuh ingestion
Host / auth / FIM (Wazuh agent) ────────┘
```

Label the S3 box: `S3 — encrypted, public access blocked, lifecycle expiry`.

### Layer 5 — Detection

`Wazuh: decode → normalise → rule evaluation → alert`. Output arrow labelled `Alert (webhook)`.

### Layer 6 — SOAR

`Shuffle workflow`, with a side branch to `Threat intelligence — IP reputation, ASN, geo` and a return arrow labelled `Enrichment attached`.

### Layer 7 — The approval gate

**Draw as a distinct, visually prominent element — a diamond or a heavy horizontal bar spanning the response path.** Label:

```
════ HUMAN APPROVAL GATE ════
No disruptive action executes automatically
```

Nothing may bypass it on the diagram. If a line goes around it, the diagram is wrong.

### Layer 8 — Response

`Response action: block · disable · isolate · monitor · close`, reached only through the gate.

### Layer 9 — Documentation

`Evidence capture · disposition · incident record`, with a **feedback arrow returning to Layer 5** labelled `tuning feedback`. The loop is part of the design, not decoration.

## Egress annotation

One arrow from the subnet outward: `HTTPS 443 egress only — AWS APIs, package repositories, threat intelligence`. Annotate: `No NAT gateway · No Elastic IP · No load balancer`.

## Style

| Element | Convention |
|---|---|
| AWS account | Outermost container, solid border |
| VPC | Nested container |
| Subnet | Nested within VPC |
| Instances | Rounded rectangles |
| Data stores | Cylinders |
| Approval gate | Diamond or heavy bar, highest visual weight |
| Not deployed | **Dashed border** |
| Log flow | Dotted arrows |
| Control flow | Solid arrows |

Monochrome-legible: it must survive being printed in black and white or pasted into a proposal. Meaning is carried by shape and label, never by colour alone.

## Formats

**[DESIGN DECISION]** Diagram source is committed as text — Mermaid or Graphviz — not as a binary image only. Text diffs; a PNG does not, and a diagram that cannot be reviewed in a pull request drifts from reality within weeks.

Rendered images go in `screenshots/` under the sanitisation rules in that directory's README.

## Before committing any diagram

- [ ] Title carries `— PLANNED ARCHITECTURE` (until deployment)
- [ ] Undeployed components dashed
- [ ] No account ID, instance ID, ARN or real IP address
- [ ] Approval gate present and unbypassable
- [ ] "No inbound ports" conveyed
- [ ] Legible in monochrome
- [ ] Source committed alongside any rendered image
