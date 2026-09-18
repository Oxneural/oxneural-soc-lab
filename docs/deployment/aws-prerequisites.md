# AWS Prerequisites — PLANNED

> **Every item below must be complete before the first AWS resource is created.**

Supersedes the generic [`prerequisites.md`](prerequisites.md) for AWS specifics.

## 1. Account

- [ ] Dedicated AWS account for this lab, isolated from all other use
- [ ] Root: **MFA enabled**, recovery codes stored offline
- [ ] Root: **no access keys** exist (delete any that do)
- [ ] Root not used for daily work — break-glass only
- [ ] Administrative IAM identity created, with MFA
- [ ] Billing access confirmed from the admin identity

## 2. Region selection

**[DESIGN DECISION]** Choose one region and stay in it. Resources scattered across regions are the most common cause of a teardown that misses something.

Selection criteria, in order:

| Criterion | Consideration |
|---|---|
| **Latency** | Nearest region to the operator — the dashboard is used interactively over an SSM tunnel |
| **Price** | **[VERIFIED FACT]** AWS pricing varies by region. Compare the actual candidates on the [EC2 pricing page](https://aws.amazon.com/ec2/pricing/) on the day. |
| **Service availability** | Confirm every service in the design exists in the chosen region |
| **Data residency** | No personal or client data is involved, so this does not constrain the choice |

**[DESIGN DECISION] LOCKED: `ap-south-1` (Mumbai).**

The dashboards are used interactively over an SSM tunnel and the operator is in Mumbai, so latency is felt on every click. Lower-cost regions exist, but at three small instances stopped between sessions the saving is a fraction of an already small bill, set against a permanently worse interactive experience. Single-region discipline also reduces the chance of a resource being launched — and later missed at teardown — somewhere else.

Full reasoning: [`../architecture/decisions.md`](../architecture/decisions.md) §1.

- [ ] **Confirm in the console** that every service in this design is available in `ap-south-1` — recorded as an assumption, not a verified fact, because the live services grid could not be read programmatically

All resources are created in `ap-south-1` and nowhere else. **Teardown verification still checks every region.**

## 3. Cost controls — before any chargeable resource

**This section is not optional and does not come later.**

- [ ] AWS Budgets: monthly budget with a hard figure, alerts at 50% / 80% / 100%
- [ ] CloudWatch billing alarm with email notification
- [ ] Cost Explorer enabled
- [ ] Tagging scheme agreed and written down
- [ ] Billing alert email address confirmed as one that is actually read

A lab without billing alerts is a bill waiting to be discovered.

## 4. Tagging scheme

Applied to **every** resource at creation:

```
Project      = oxneural-soc-lab
Owner        = <owner>
Environment  = lab
CostCentre   = internal
TeardownAfter= <YYYY-MM-DD>
```

Tagging is what makes cost attributable and teardown verifiable. Untagged resources are how labs become permanent.

## 5. Local environment

- [ ] AWS CLI v2 installed
- [ ] **Session Manager plugin installed** — required for `aws ssm start-session`; without it there is no access path at all
- [ ] CLI profile configured for the lab account **only**
- [ ] `.env` created from `.env.example`; confirmed git-ignored
- [ ] `git status` confirms no credential file is tracked

**[DESIGN DECISION]** No SSH key pair is created. Access is exclusively via Session Manager, so there is no private key to protect, lose or accidentally commit.

**[DESIGN DECISION] LOCKED: base AMI Ubuntu Server 24.04 LTS** on all three instances — one patch baseline, one package manager. **[VERIFIED FACT]** SSM Agent is preinstalled on Ubuntu Server 24.04 LTS ([AMIs with SSM Agent preinstalled](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)); AWS notes the preinstalled version may not be current, so it is updated on first boot. See [`../architecture/decisions.md`](../architecture/decisions.md) §2.

## 6. Service quota checks

- [ ] Default VPC quota sufficient (the lab needs one)
- [ ] EC2 instance limits allow the planned instance types in the chosen region
- [ ] Elastic IP quota — not needed, none will be allocated

**[ASSUMPTION]** Default quotas on a new account are sufficient for three small instances. Verify before Phase 2; a new account occasionally has low initial limits.

## 7. Knowledge prerequisites

- [ ] Wazuh architecture understood — manager, indexer, dashboard, agent
- [ ] **[VERIFIED FACT]** Wazuh all-in-one installation assistant reviewed: `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a` ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html))
- [ ] Shuffle installation reviewed — Docker-based ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md))
- [ ] **[VERIFIED FACT]** Shuffle requires `sudo sysctl -w vm.max_map_count=262144`; must be made persistent or Opensearch fails on reboot
- [ ] SSM Session Manager and port forwarding understood
- [ ] CloudTrail event structure understood
- [ ] IAM policy evaluation understood

## 8. Design approval

- [ ] [`../architecture/final-aws-architecture.md`](../architecture/final-aws-architecture.md) reviewed and approved
- [ ] [`cost-model.md`](cost-model.md) reviewed and approved
- [ ] [`teardown-plan.md`](teardown-plan.md) reviewed — **written and approved before build**
- [ ] [`rollback-plan.md`](rollback-plan.md) reviewed
- [ ] [`../security/iam-design.md`](../security/iam-design.md) reviewed
- [ ] All **[OPEN QUESTION]** items resolved or explicitly accepted

## 9. Authorisation and ethics

- [ ] Confirmed: all activity is against OxNeural-owned lab resources only
- [ ] Confirmed: **no third-party system will be touched**
- [ ] Confirmed: simulation is non-destructive; no real malware, no exploit code
- [ ] Confirmed: AWS acceptable use policy understood

## Sign-off

Deployment does not begin until every box above is ticked.

| Item | Confirmed by | Date |
|---|---|---|
| Prerequisites complete | | |
| Region selected and recorded | | |
| Cost controls live | | |
| Teardown plan approved | | |
| Design approved | | |

## Sources

- [Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Amazon EC2 pricing](https://aws.amazon.com/ec2/pricing/)
