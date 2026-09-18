# Playbook: Brute-Force Authentication

**Status:** Documented. **Not implemented in Shuffle, not exercised.**
**Trigger:** Brute-force detection — see [`../detection/brute-force.md`](../detection/brute-force.md)

## Shuffle workflow

```
Alert (Wazuh webhook)
   │
   ▼
Extract: srcip · srcuser · agent · failure count · window
   │
   ▼
Is srcip external and routable?  ──no──▶ tag internal, skip external lookup
   │ yes
   ▼
ENRICHMENT (automatic, no approval)
   ├── IP reputation
   ├── Geolocation / ASN
   ├── Seen in lab before?
   ├── Does target account exist? Privilege level?
   └── Any SUCCESSFUL auth from this source in the window?
   │
   ▼
RISK ASSESSMENT
   ├── Success after failures? ──────▶ severity HIGH, escalate immediately
   ├── Privileged account targeted? ─▶ raise severity
   ├── Multiple accounts? ───────────▶ password spraying
   └── Poor reputation source? ──────▶ raise confidence
   │
   ▼
Present enriched alert to analyst
   │
   ▼
════════ APPROVAL GATE ════════
   │
   ▼
ANALYST DECISION
   ├── False positive ──▶ document, feed tuning
   ├── Monitor ─────────▶ heighten monitoring, document
   └── Respond ─────────▶ approved actions below
   │
   ▼
DOCUMENTATION (always, every path)
```

## Analyst decision criteria

| Evidence | Action |
|---|---|
| Internal source, single account, small count | Likely user error — verify, document, close |
| Internal source, regular automated interval | Likely stale service credential — identify the service |
| External, background scanning volume, no success | Expected internet noise — document, close |
| External, sustained, targeted | Investigate; consider blocking |
| **Any success following failures** | **Treat as potential compromise — full investigation** |
| Multiple accounts from one source | Password spraying — investigate all targeted accounts |

## Response actions

| Action | Approval | Notes |
|---|---|---|
| Document | None | Always |
| Notify account owner | None | |
| Increase monitoring on target | None | |
| **Block source IP** | **Required** | May be NAT/CGNAT — could block many legitimate users |
| **Disable targeted account** | **Required** | Denies service to a legitimate user |
| **Force password reset** | **Required** | |
| **Revoke active sessions** | **Required** | Do this if any success occurred |

**No blocking action executes automatically.** The workflow prepares it, attaches the context, and waits.

## If authentication succeeded

Escalate to High immediately and treat as potential compromise:

1. Revoke active sessions for the account (approval required)
2. Force credential reset (approval required)
3. Pull **everything** that account did after the successful authentication
4. Check for persistence — new keys, new users, scheduled tasks, startup entries
5. Check for lateral movement from the host
6. Assess data access
7. Follow [`incident-response-process.md`](incident-response-process.md) through eradication and recovery

The successful login is the start of the investigation, not the end of it.

## Evidence to capture

Full authentication log for the window · source IP, ports, UTC timestamps · target accounts and outcomes · any successful authentication · all subsequent session activity · enrichment results with query time · the alert with rule ID.

**Capture before remediating.**

## Post-incident

Was the threshold right? Did enrichment help or add noise? Was the source internet background scanning that should be baselined? Should this account have been exposed at all? Feed answers to [`../detection/detection-engineering.md`](../detection/detection-engineering.md).
