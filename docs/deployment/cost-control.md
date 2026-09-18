# Cost Control

AWS bills for what exists, not what is used. A forgotten resource costs the same as a working one.

## Before any chargeable resource

- [ ] AWS Budgets: monthly budget with a hard figure and alerts at 50%, 80%, 100%
- [ ] CloudWatch billing alarm with email notification
- [ ] Cost Explorer enabled
- [ ] Tagging scheme applied from the first resource

**This comes first.** Configuring alerts after deployment means the first thing they can warn about is a bill already incurred.

## Minimum viable footprint

Only what the lab needs to demonstrate the capability:

| Component | Necessary | Notes |
|---|---|---|
| VPC, subnets, security groups | Yes | No charge |
| Wazuh EC2 | Yes | Largest cost; stop when not in use |
| Shuffle EC2 | Yes | Stop when not in use |
| Target host EC2 | Yes | Smallest viable; stop when not in use |
| S3 for CloudTrail | Yes | Small with lifecycle expiry |
| CloudTrail (single trail) | Yes | First management trail has no charge |
| VPC Flow Logs | Yes | Charged by volume — scope deliberately |
| **NAT gateway** | **Avoid** | Hourly + data charges; frequently the largest surprise on a lab bill |
| **Elastic IP (unassociated)** | **Avoid** | Bills while unattached |
| Load balancer | No | Not needed |
| Multi-AZ / HA | No | Not needed for a lab |

## The things that actually cause surprise bills

Ranked by how often they catch people out:

1. **NAT gateway** left running — hourly charge plus per-GB processing
2. **EBS volumes** surviving terminated instances — they persist and keep billing
3. **Unassociated Elastic IPs** — charged precisely because they are idle
4. **Snapshots and AMIs** accumulating unnoticed
5. **Log data** growing without lifecycle rules
6. **Resources in another region** — invisible unless you look there
7. **Instances left running overnight and at weekends**

## Operating rules

- **Stop compute at the end of every session.** Non-negotiable.
- Set `TeardownAfter` on every resource at creation.
- Check the cost dashboard weekly.
- Investigate any unexpected charge the day it appears.
- Full teardown when the lab is not needed for an extended period.

## Tagging

Every resource, without exception:

```
Project=oxneural-soc-lab
Owner=<owner>
Environment=lab
CostCentre=internal
TeardownAfter=<YYYY-MM-DD>
```

Tagging is what makes cost attributable and teardown verifiable. Untagged resources are how labs become permanent.

## Sizing

Instance sizes are chosen during deployment based on what actually runs the stack, then recorded here with the date. **No sizing or cost figures are stated in advance** — an estimate published as a fact is an invented metric.

## Cost log

Filled in from real bills, not projections.

| Month | Actual cost | Notes |
|---|---|---|
| | | |
