# Logging and Monitoring — PLANNED

> **PLANNED. No logging source has been enabled.**

## Sources

| Source | Captures | Destination | Feeds |
|---|---|---|---|
| **CloudTrail** (1 management trail) | API calls, console sign-in, IAM changes, SSM sessions | S3 | Unauthorized IAM detection |
| **VPC Flow Logs** | Accepted/rejected connections, ports, byte counts | S3 | Suspicious network detection |
| **Wazuh agent** | Authentication, process, file integrity | Wazuh manager | Brute force, malware behaviour |
| **Wazuh internal** | Agent status, manager health | Wazuh manager | Pipeline health |

**[VERIFIED FACT]** CloudTrail delivers "one copy of your ongoing management events to your Amazon S3 bucket for free by creating trails"; additional trails and data events are charged ([AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)).

**[DESIGN DECISION]** Exactly one trail, management events only, all regions. No data events (charged per event, no lab use). No CloudTrail Insights (charged per event analysed).

**[DESIGN DECISION]** All-regions trail even though the lab lives in one region. An attacker creating resources will do so where nobody is looking, and a single-region trail makes that invisible. It costs nothing extra within the free allowance.

## Why S3 rather than CloudWatch Logs

**[DESIGN DECISION]** Both CloudTrail and VPC Flow Logs deliver to S3.

| | S3 | CloudWatch Logs |
|---|---|---|
| Cost profile | Storage, with lifecycle expiry | Ingestion **and** storage; ingestion is the larger charge |
| Wazuh ingestion | Native AWS module support | Requires additional plumbing |
| Retention control | Lifecycle rules | Retention settings |

Flow log volume scales with traffic, so ingestion charges are the ones that surprise people. S3 keeps this predictable.

## Retention

**[DESIGN DECISION]** Lifecycle expiry on every log prefix, set at deployment.

| Data | Intent |
|---|---|
| CloudTrail in S3 | Long enough to investigate; short enough to bound cost |
| Flow logs in S3 | Shorter — highest volume, lowest per-event value |
| Wazuh alerts | Longer than raw logs; alerts are small |
| Incident documentation | Life of the lab, in git |

**Exact periods are set when storage cost is measured, not estimated in advance.** Without lifecycle rules, log storage grows forever and quietly becomes the largest line on the bill.

## Time

**[DESIGN DECISION]** UTC everywhere. NTP synchronised on every host. Verified as a health check, not assumed.

Unsynchronised clocks make correlation unreliable and timelines indefensible. It is the most common cause of an investigation reaching a confidently wrong conclusion, and it is invisible until someone tries to build a timeline.

## Ingestion into Wazuh

```
CloudTrail ──▶ S3 ──┐
                    ├──▶ Wazuh AWS module (polling) ──▶ decode ──▶ normalise ──▶ rules
VPC Flow Logs ─▶ S3 ┘
Host telemetry ────────▶ Wazuh agent (TLS) ──────────▶ decode ──▶ normalise ──▶ rules
```

**[OPEN QUESTION]** Wazuh's AWS module supports polling S3 directly or consuming SQS notifications. **Polling is the default assumption** — simpler, no extra resource, no extra IAM permission. SQS reduces latency and API calls; adopt only if polling proves inadequate. Confirm against Wazuh documentation at deployment.

**[DESIGN DECISION]** `role-wazuh` receives `s3:GetObject` and `s3:ListBucket` scoped to the log bucket and its prefixes, and nothing else. See [`iam-design.md`](iam-design.md).

## Field normalisation

Detection depends on consistent field names across sources. Without it, a CloudTrail rule cannot correlate with a host rule.

| Concept | Field |
|---|---|
| Source address | `srcip` |
| Destination address | `dstip` |
| User or principal | `srcuser` |
| Event action | `action` |
| Outcome | `status` |
| Timestamp (UTC) | `timestamp` |

## Monitoring the monitoring

**[DESIGN DECISION]** The most dangerous failure in this system is silent. An agent stops reporting, and the console looks calm — indistinguishable from a quiet day.

| Check | Confirms | Cadence |
|---|---|---|
| **Wazuh agent heartbeat** | Every agent reporting | Continuous — alert on silence |
| CloudTrail delivery | Objects landing in S3 | Daily |
| Flow log delivery | Network telemetry arriving | Daily |
| Wazuh → Shuffle webhook | Alerts reaching SOAR | Per alert |
| Threat intelligence API | Reachable, key valid | Daily |
| Time synchronisation | Clocks aligned | Daily |
| Disk and index capacity | Ingestion will not stall | Weekly |

**Absence of alerts is not evidence of absence of activity.** It is equally consistent with a broken pipeline. These checks are what distinguish the two, and they are part of the design rather than an afterthought.

## Security control monitoring

The lab watches its own controls. These detections carry the highest severity because they indicate someone turning off the lights:

| Event | Severity |
|---|---|
| `StopLogging` / `DeleteTrail` | **Critical** |
| `DeactivateMFADevice` | **Critical** |
| Root account usage | **Critical** |
| Security group modified to add inbound | High |
| `AttachUserPolicy` / `AttachRolePolicy` granting broad permissions | High |
| `CreateAccessKey` | High |
| `UpdateAssumeRolePolicy` | High |

**[DESIGN DECISION]** `lab-admin` carries an explicit `Deny` on `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`, so the critical case is prevented as well as detected. An explicit `Deny` cannot be overridden by any `Allow`. See [`iam-design.md`](iam-design.md).

## Analyst access is itself logged

**[DESIGN DECISION]** Session Manager sessions appear in CloudTrail. This is deliberate: it means operator access to the lab is a detectable event, and provides a baseline of normal access against which anything unusual stands out.

## Cost awareness

Logging is where labs quietly overspend. Two disciplines:

1. **Lifecycle expiry on every prefix**, from the day the bucket is created
2. **Review flow log volume in the first week** and narrow scope if it is disproportionate

See [`../deployment/cost-model.md`](../deployment/cost-model.md).

## Sources

- [AWS CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)
- [Amazon VPC pricing](https://aws.amazon.com/vpc/pricing/)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/)
