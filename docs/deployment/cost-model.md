# Cost Model — PLANNED

> **No AWS resource exists. No charge has been incurred.**

## How to read this document

**No dollar figures appear here.** AWS pricing varies by region and changes; publishing a number I have not verified for the chosen region on the day of deployment would be inventing a metric. Every resource is instead given a **cost category** and a link to the official pricing page, which is the source of truth.

| Category | Meaning |
|---|---|
| **FREE** | No charge for this usage pattern, per official AWS documentation |
| **LOW-COST** | Small, predictable, flat charge |
| **VARIABLE** | Scales with usage — the ones to watch |
| **POTENTIALLY EXPENSIVE** | Can become the largest line on the bill if left unattended |

## Resource-by-resource

### EC2 instances — `wazuh-aio`, `shuffle`, `target-01`

| | |
|---|---|
| **Why required** | Wazuh all-in-one, Shuffle SOAR, and a telemetry source. Without compute there is no lab. |
| **Minimum viable** | RAM-led sizing: Wazuh documents 8 GiB, Shuffle documents 4 GiB minimum. See [`../architecture/final-aws-architecture.md`](../architecture/final-aws-architecture.md) §4 for the sizing decision and its recorded deviation. |
| **Cost category** | **POTENTIALLY EXPENSIVE** if left running continuously; **LOW-COST** under stop-when-idle discipline |
| **Can be stopped?** | **Yes — and this is the single largest cost lever in the lab.** Stopped instances incur no compute charge. |
| **Ongoing charges?** | Compute stops when stopped. **Attached EBS volumes do not.** |
| **Teardown** | Terminate instance, then explicitly delete the EBS volume |
| **Source** | [Amazon EC2 pricing](https://aws.amazon.com/ec2/pricing/) |

### EBS volumes (gp3)

| | |
|---|---|
| **Why required** | Root and data storage. Wazuh documents 50 GB for the 1–25 agent profile. |
| **Minimum viable** | gp3 at baseline performance. **[VERIFIED FACT]** gp3 includes 3,000 IOPS and 125 MB/s free; charges apply only beyond that baseline. Provision nothing above it. |
| **Cost category** | **LOW-COST**, but **relentless** |
| **Can be stopped?** | **No.** There is no "stopped" state for a volume. |
| **Ongoing charges?** | **Yes — continuously.** "Volume storage for all EBS volume types is charged by the amount of GB you provision per month **until you release the storage**." |
| **Teardown** | **Delete the volume explicitly.** Volumes commonly survive instance termination and keep billing — this is the second most common cause of a lingering lab bill. |
| **Source** | [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/) |

### VPC, subnet, route table, internet gateway, security groups

| | |
|---|---|
| **Why required** | Network foundation |
| **Cost category** | **FREE** for these components themselves |
| **Ongoing charges?** | No |
| **Teardown** | Delete after compute is removed — order matters |
| **Source** | [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/) |

### NAT Gateway — **NOT USED**

| | |
|---|---|
| **Why excluded** | **[VERIFIED FACT]** Charged per NAT-Gateway-hour provisioned **and** per gigabyte processed. It bills continuously whether or not the lab is in use. |
| **Cost category** | **POTENTIALLY EXPENSIVE** — the most frequent cause of a surprise bill on a small AWS lab |
| **Design response** | Eliminated entirely. See [`../architecture/final-aws-architecture.md`](../architecture/final-aws-architecture.md) §3.2. |
| **Source** | [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/) |

### Public IPv4 addresses

| | |
|---|---|
| **Why required** | Outbound access for SSM, package updates and threat intelligence |
| **Cost category** | **LOW-COST** |
| **Ongoing charges?** | **[VERIFIED FACT]** Public IPv4 addresses carry an hourly charge |
| **Design response** | **Auto-assigned addresses, not Elastic IPs.** An auto-assigned address is released when the instance stops; an unattached Elastic IP keeps billing precisely because it is idle. The address changes between sessions, which costs nothing because nothing ever connects inbound. |
| **Teardown** | Released automatically with the instance |
| **Source** | [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/) |

### S3 — CloudTrail and flow log destination

| | |
|---|---|
| **Why required** | Durable destination for CloudTrail and VPC Flow Logs, read by Wazuh |
| **Minimum viable** | One bucket, two prefixes, lifecycle expiry on both |
| **Cost category** | **LOW-COST** with lifecycle rules; **VARIABLE** without them |
| **Ongoing charges?** | Yes — storage, requests, and growth over time |
| **Teardown** | **Empty the bucket, then delete it.** A non-empty bucket cannot be deleted. |
| **Source** | [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/) |

### CloudTrail — one management-events trail

| | |
|---|---|
| **Why required** | Primary detection source for the unauthorised-IAM scenario |
| **Minimum viable** | Exactly one trail, management events only |
| **Cost category** | **FREE** for this configuration |
| **Ongoing charges?** | **[VERIFIED FACT]** "You can deliver one copy of your ongoing management events to your Amazon S3 bucket for free by creating trails." Additional trails and data events are charged. S3 storage still applies. |
| **Deliberately excluded** | Second trail, data events, CloudTrail Insights — all charged per event, none needed |
| **Teardown** | Delete the trail before deleting the bucket |
| **Source** | [AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/) |

### VPC Flow Logs

| | |
|---|---|
| **Why required** | Network telemetry for the suspicious-network-activity scenario |
| **Minimum viable** | VPC-level, delivered to **S3 rather than CloudWatch Logs**, scope set deliberately |
| **Cost category** | **VARIABLE** — billed through the destination's ingestion and storage charges, and scaling with traffic |
| **Ongoing charges?** | Yes, proportional to traffic volume |
| **Design response** | This is the one logging cost that is not flat. Scope reviewed in the first week of operation; lifecycle expiry applied. |
| **Teardown** | Delete the flow log, then the destination data |
| **Source** | [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/) · [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) |

### AWS Systems Manager — Session Manager and Patch Manager

| | |
|---|---|
| **Why required** | The sole access path, and patching |
| **Cost category** | **FREE** on EC2 |
| **Ongoing charges?** | **[VERIFIED FACT]** Both are documented as "No additional charges for usage on Amazon EC2 instances." |
| **Note** | The free position applies to EC2. Hybrid and multicloud nodes are treated differently — not relevant to this lab, but worth knowing before reusing this design elsewhere. |
| **Teardown** | Nothing to remove; the instance role goes with IAM cleanup |
| **Source** | [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/) |

### CloudWatch — billing alarm and log groups

| | |
|---|---|
| **Why required** | The billing alarm is the lab's financial safety net |
| **Minimum viable** | One billing alarm. Avoid CloudWatch Logs as a flow-log destination — S3 is cheaper for this pattern. |
| **Cost category** | **LOW-COST** for one alarm; **VARIABLE** if log ingestion is added |
| **Teardown** | Delete alarms and log groups |
| **Source** | [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/) |

### AWS Budgets

| | |
|---|---|
| **Why required** | Threshold alerts before the bill, not after |
| **Cost category** | **FREE / LOW-COST** — a small number of budgets is free |
| **Teardown** | **Keep this until last.** It is what catches anything the teardown missed. |
| **Source** | [AWS Budgets pricing](https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/) |

### IAM

**FREE.** No charge for users, roles or policies.

## Summary by category

| Category | Resources |
|---|---|
| **FREE** | VPC, subnet, route table, IGW, security groups, IAM, CloudTrail (1 management trail), SSM Session Manager, SSM Patch Manager |
| **LOW-COST** | EBS volumes, public IPv4, S3 with lifecycle, CloudWatch billing alarm, AWS Budgets |
| **VARIABLE** | VPC Flow Logs, S3 growth, CloudWatch Logs if used |
| **POTENTIALLY EXPENSIVE** | EC2 left running continuously · NAT Gateway (**excluded by design**) · Elastic IPs left unattached (**excluded by design**) |

## The three rules that actually control the bill

1. **Stop instances at the end of every session.** Largest lever by a wide margin.
2. **Delete EBS volumes at teardown.** They survive instance termination and bill silently.
3. **Keep the budget and billing alarm in place** — including after teardown, until the bill reaches zero.

## Cost log

To be completed from **actual bills**, never from projections.

| Month | Actual cost | Notes |
|---|---|---|
| | | |

## Sources

- [Amazon EC2 pricing](https://aws.amazon.com/ec2/pricing/)
- [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
- [AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
- [AWS Budgets pricing](https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/)
