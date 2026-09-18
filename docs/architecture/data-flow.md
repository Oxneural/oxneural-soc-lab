# Data Flow

**Status:** Design. Nothing described here is running.

## End-to-end flow

```
Event occurs (auth attempt, API call, network connection)
        │
        ▼
Captured by source          CloudTrail | VPC Flow Logs | Wazuh agent
        │
        ▼
Transported to Wazuh manager                (TLS)
        │
        ▼
Decoded and normalised      → common field names, parsed timestamps
        │
        ▼
Rule evaluation             → match? severity? 
        │
        ├── no match ──▶ stored for retention window, searchable
        │
        ▼
Alert generated             → alert ID, rule ID, severity, raw event retained
        │
        ▼
Forwarded to Shuffle SOAR                   (webhook / API)
        │
        ▼
Enrichment                  → IP reputation, IAM context, asset context
        │
        ▼
Enriched alert presented to analyst
        │
        ▼
Analyst triage              → disposition recorded
        │
        ├── False Positive ──────▶ tuning feedback ──▶ rule updated
        ├── Benign True Positive ─▶ documented, baseline updated
        └── True Positive ───────▶ investigation
                                        │
                                        ▼
                              [APPROVAL GATE] ──▶ response action
                                        │
                                        ▼
                              Evidence captured, incident documented
```

## Field normalisation

Detection logic depends on consistent field names across sources. Without normalisation, a rule written against CloudTrail cannot correlate with a rule written against host logs.

| Concept | Normalised field |
|---|---|
| Source address | `srcip` |
| Destination address | `dstip` |
| User or principal | `srcuser` |
| Event action | `action` |
| Outcome | `status` |
| Timestamp (UTC) | `timestamp` |

## Time

**All timestamps normalised to UTC at ingestion.** Sources are NTP-synchronised. Unsynchronised clocks make correlation unreliable and timelines indefensible — this is the single most common cause of an investigation reaching the wrong conclusion.

## Data volume

Not stated. Volume will be measured during deployment and reported then, with the measurement method. Estimating it now and publishing the estimate would be inventing a figure.

## Retention intent

| Data | Intent |
|---|---|
| Raw logs | Short window sufficient for investigation; cost-bounded |
| Alerts | Longer than raw logs — alerts are small |
| Incident documentation | Retained for the life of the lab |

Exact periods are set in the deployment plan once storage cost is measured.

## What is never in this pipeline

No client data. No third-party data. No personal data. No production telemetry from any other environment. The lab generates its own activity, synthetically.

## Failure modes to watch

An unmonitored pipeline fails silently and everything downstream looks healthy:

- Agent stops reporting → host goes dark, no alert fires, absence looks like calm
- Log destination fills or permission changes → ingestion stops
- Webhook to SOAR fails → alerts generate but never reach enrichment
- Clock drift → correlation breaks

Each of these gets a health check defined in [`../testing/validation-plan.md`](../testing/validation-plan.md). **Monitoring the monitoring is part of the design, not an afterthought.**
