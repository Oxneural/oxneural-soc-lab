# Data Flow — PLANNED

> **PLANNED. Nothing described here is running.**

## Detection data flow

```
EVENT OCCURS
  AWS API call · console sign-in · IAM change        → CloudTrail
  Network connection accepted or rejected            → VPC Flow Logs
  Authentication · process · file change on a host   → Wazuh agent
        │
        ▼
TRANSPORT
  CloudTrail    ──▶ S3 (encrypted, lifecycle expiry)
  Flow Logs     ──▶ S3 (encrypted, lifecycle expiry)
  Host telemetry──▶ Wazuh manager over TLS
        │
        ▼
INGESTION
  Wazuh AWS module polls S3          ──┐
  Wazuh agent stream                 ──┼──▶ Wazuh manager
        │
        ▼
DECODE AND NORMALISE
  Common field names · UTC timestamps · parsed values
        │
        ▼
RULE EVALUATION
        ├── no match ──▶ stored, searchable for the retention window
        │
        ▼
ALERT GENERATED
  alert id · rule id · severity · raw event retained
        │
        ▼
INDEXED                    ──▶ analyst dashboard (via SSM tunnel)
        │
        ▼
WEBHOOK ──▶ Shuffle SOAR
```

## Incident response data flow

```
Shuffle receives alert
        │
        ▼
EXTRACT indicators — IP · user · host · process · event name
        │
        ▼
FILTER — skip RFC1918, loopback, known-good lab addresses
         BEFORE any external lookup
        │        (no internal address is ever sent to a third party)
        ▼
ENRICH  (automatic — nothing disruptive, no approval needed)
        ├── IP reputation · geolocation · ASN
        ├── Seen in the lab before?
        ├── IAM context — policy document, actor baseline, MFA used?
        └── Asset context — host role, baseline behaviour
        │
        ▼
RISK ASSESSMENT — documented severity criteria applied
        │
        ▼
ENRICHED ALERT PRESENTED TO ANALYST
        │
        ▼
╔════════════════════════════════════════════╗
║        HUMAN APPROVAL GATE                 ║
║  No disruptive action executes automatically║
╚════════════════════════════════════════════╝
        │
        ▼
ANALYST DECISION
        ├── False positive ──────▶ document ──▶ TUNING FEEDBACK ──┐
        ├── Benign true positive ▶ document, baseline updated ────┤
        ├── Monitor ─────────────▶ heightened monitoring ─────────┤
        └── Respond ─────────────▶ approved action only           │
                                        │                         │
                                        ▼                         │
                        EVIDENCE CAPTURED BEFORE REMEDIATION      │
                                        │                         │
                                        ▼                         │
                        RESPONSE EXECUTED (approved)              │
                                        │                         │
                                        ▼                         │
                        INCIDENT DOCUMENTED                       │
                                        │                         │
                                        └─────────────────────────┘
                                          feeds back to rule tuning
```

**The feedback loop is part of the design.** A detection whose false positives never reach the rule that produced them stays noisy forever.

## Where the approval gate sits, and why

Automatic, no approval: enrichment, context lookup, notification, increased monitoring — nothing that changes state.

Approval required: block an address, disable an identity or key, isolate a host, modify a security group, terminate or rebuild.

**[DESIGN DECISION]** The gate exists because enrichment is probabilistic. A reputation hit on a NAT or CGNAT address represents an entire ISP's customer base; automatic blocking on that signal converts a false positive into an outage. Automation prepares the action and attaches the context. A human decides.

**[DESIGN DECISION]** `role-shuffle` holds no AWS API permissions in the initial build, so the gate is enforced by IAM as well as by workflow design — not only by the workflow being drawn correctly. See [`../security/iam-design.md`](../security/iam-design.md).

## Field normalisation

Detection depends on consistent field names. Without it, a CloudTrail rule cannot correlate with a host rule.

| Concept | Field |
|---|---|
| Source address | `srcip` |
| Destination address | `dstip` |
| User or principal | `srcuser` |
| Event action | `action` |
| Outcome | `status` |
| Timestamp (UTC) | `timestamp` |

## Time

**All timestamps normalised to UTC at ingestion.** Sources NTP-synchronised, verified as a health check rather than assumed.

Unsynchronised clocks make correlation unreliable and timelines indefensible. It is the most common cause of an investigation reaching a confidently wrong conclusion, and it is invisible until someone tries to build a timeline.

## Data volume

**Not stated.** Volume will be measured during deployment and reported then, with the method. Publishing an estimate now would be inventing a figure.

## Retention

| Data | Intent |
|---|---|
| Raw logs in S3 | Long enough to investigate, short enough to bound cost; lifecycle expiry |
| Wazuh alerts | Longer than raw logs — alerts are small |
| Incident documentation | Life of the lab, in git |

Exact periods set once storage cost is measured. See [`../security/logging-and-monitoring.md`](../security/logging-and-monitoring.md).

## What is never in this pipeline

No client data. No third-party data. No personal data. No production telemetry from any other environment. The lab generates its own activity, synthetically.

## Failure modes

An unmonitored pipeline fails silently, and everything downstream looks healthy:

| Failure | Appears as | Detected by |
|---|---|---|
| Agent stops reporting | A quiet host | Agent heartbeat alert (AC-44) |
| S3 delivery stops | No new cloud events | Daily delivery check |
| Webhook to Shuffle fails | Alerts generated, never enriched | Per-alert delivery check |
| Enrichment API unavailable | Alerts without context | Daily API check — **must not suppress the alert** (AC-65) |
| Clock drift | Timelines that do not line up | Daily time-sync check |

**Absence of alerts is not evidence of absence of activity.** It is equally consistent with a broken pipeline. These checks are what distinguish the two — monitoring the monitoring is part of the design, not an afterthought.

## Related

[`architecture-overview.md`](architecture-overview.md) · [`final-aws-architecture.md`](final-aws-architecture.md) · [`wazuh-design.md`](wazuh-design.md) · [`shuffle-design.md`](shuffle-design.md) · [`../testing/validation-plan.md`](../testing/validation-plan.md)
