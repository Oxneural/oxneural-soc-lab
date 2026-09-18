# Architecture Overview

**Status:** Design. **No component described here is deployed.**

## Purpose

An isolated AWS environment producing realistic security telemetry, a SIEM that collects and detects on it, a SOAR platform that enriches and orchestrates response, and a documented analyst workflow around both.

## Logical pipeline

```
┌──────────────────────────────────────────────────────────┐
│ 1. SOURCE — isolated AWS lab account                     │
│    IAM · EC2 · VPC · S3                                  │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 2. LOG COLLECTION                                        │
│    CloudTrail (API/control plane)                        │
│    VPC Flow Logs (network)                               │
│    Wazuh agent (host, auth, file integrity)              │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 3. DETECTION — Wazuh                                     │
│    Decoding → normalisation → rule evaluation → alert    │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 4. ENRICHMENT — Shuffle SOAR                             │
│    IP reputation · IAM context · asset context           │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 5. INVESTIGATION — SOC analyst                           │
│    Triage → timeline → scope → impact → disposition      │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 6. RESPONSE — [APPROVAL GATE] then action                │
│    Block · disable · isolate · monitor · close            │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ 7. DOCUMENTATION                                         │
│    Disposition · evidence · timeline · tuning feedback   │
└──────────────────────────────────────────────────────────┘
```

## The stages, and why they are separated

**Log collection** is the only stage that touches source systems. Its job is completeness and integrity, not interpretation. Gaps here are invisible later — a detection cannot fire on telemetry that was never collected.

**Detection** turns events into alerts. Its output is an assertion that something may warrant attention, with a severity and a defined response. Detection quality is measured by whether alerts are actionable, not by how many fire.

**Enrichment** attaches context automatically so the analyst does not spend their first ten minutes doing lookups. Enrichment never decides; it informs.

**Investigation** is the human stage. The analyst establishes what actually happened before deciding what it means, builds a timeline, scopes the affected assets and identities, and records a disposition.

**Response** is where action is taken — and where the approval gate sits. Every potentially disruptive action requires human authorisation. Automating containment without a gate is how a false positive becomes an outage.

**Documentation** closes the loop: the disposition, the evidence, and the tuning feedback that improves the detection that fired.

## Feedback loop

Investigation outcomes feed back into detection. A rule producing benign true positives gets tuned, not disabled — a disabled rule looks like coverage on a report and provides none. Tuning changes are recorded with reason and date.

## Design constraints

| Constraint | Consequence |
|---|---|
| Single lab account, isolated | No cross-account or multi-tenant patterns demonstrated |
| Cost-controlled | Minimal always-on footprint; components stopped when not in use |
| Non-destructive simulation only | No real malware, no exploit code |
| Documentation before deployment | Architecture reviewed before resources are created |

## What this architecture does not demonstrate

Stated plainly so it is not inferred: production scale, high availability, multi-region resilience, 24/7 staffed operations, enterprise log volumes, or multi-tenant separation. It is a lab.

## Related

[`aws-architecture.md`](aws-architecture.md) · [`data-flow.md`](data-flow.md) · [`../deployment/deployment-plan.md`](../deployment/deployment-plan.md)
