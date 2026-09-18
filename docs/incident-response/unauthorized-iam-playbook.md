# Playbook: Unauthorized IAM Activity

**Status:** Documented. **Not implemented in Shuffle, not exercised.**
**Trigger:** IAM detection — see [`../detection/unauthorized-iam-activity.md`](../detection/unauthorized-iam-activity.md)

## Shuffle workflow

```
Alert (CloudTrail via Wazuh)
   │
   ▼
Extract: actor · event · target identity · srcip · outcome · time
   │
   ▼
EVENT VALIDATION
   ├── Did it succeed, or was it AccessDenied?
   ├── Is the actor a known automation principal?
   └── Is this within a declared change window?
   │
   ▼
IAM CONTEXT (automatic, no approval)
   ├── Full policy document — what exactly was granted
   ├── Actor's normal activity baseline
   ├── Was MFA used?
   ├── Source IP reputation, geolocation, seen before?
   ├── ALL events in the same session
   └── Any security-control tampering in the session?
   │
   ▼
RISK ASSESSMENT
   ├── Logging or MFA disabled? ──────▶ CRITICAL, escalate immediately
   ├── Root account used? ────────────▶ CRITICAL
   ├── Admin/wildcard policy granted? ▶ HIGH
   ├── Access key created by another? ▶ HIGH — persistence mechanism
   ├── Trust policy modified? ────────▶ HIGH — check for external principal
   └── Unfamiliar source IP? ─────────▶ raise confidence
   │
   ▼
Present enriched alert to analyst
   │
   ▼
════════ APPROVAL GATE ════════
   │
   ▼
ANALYST DECISION
   ├── Authorised change ──▶ document, baseline the principal, close
   ├── Unclear ────────────▶ contact actor, hold
   └── Unauthorised ───────▶ approved actions below
   │
   ▼
DOCUMENTATION (always)
```

## Analyst decision criteria

| Evidence | Action |
|---|---|
| Known admin, known IP, declared change window | Authorised — document and close |
| Known CI/automation role, matching normal pattern | Authorised — baseline it to reduce future noise |
| `AccessDenied` outcome | Did not succeed — still investigate why it was attempted |
| Unfamiliar source IP, no declared change | Investigate urgently |
| **Logging disabled or MFA removed** | **Critical — assume compromise until disproved** |
| **Root account activity** | **Critical — root should be effectively unused** |
| Access key created for another identity | Treat as persistence until explained |

## Response actions

| Action | Approval | Notes |
|---|---|---|
| Investigate, pull session history | None | |
| Enrich | None | Automatic |
| Contact the actor to confirm intent | None | **Often resolves it in one message** |
| Increase CloudTrail monitoring | None | |
| **Revoke access key** | **Required** | May break a running system |
| **Detach policy / revert change** | **Required** | Destroys evidence of what was granted — capture first |
| **Disable IAM identity** | **Required** | |
| **Revoke active sessions** | **Required** | |
| **Re-enable logging** | **Required — but do it immediately** | |

## If compromise is confirmed

Order matters:

1. **Re-enable logging first** if it was disabled — you are blind until it is back
2. Revoke the actor's sessions and keys (approval)
3. Revert unauthorised permission changes (after capturing the policy documents)
4. Pull the complete CloudTrail history for the principal, across all regions
5. Identify how access was obtained — trace to the authentication event
6. Check for other persistence: new users, new keys, modified trust policies, new roles
7. Assess what resources and data were accessed
8. Rotate every credential the actor could have reached
9. Follow [`incident-response-process.md`](incident-response-process.md) through eradication and recovery

**Check every region.** IAM is global but resources are regional; an attacker creating resources will often do so where nobody is looking.

## Evidence to capture

Full CloudTrail JSON for the event · complete session history for the principal · policy documents before and after · the authentication event that created the session · MFA status · source IP, user agent, UTC timestamps · every resource created or accessed in the session.

**Capture before reverting.** Reverting removes the record of what was granted.

## Post-incident

How was the credential obtained? Should this identity have held this permission at all? Was the change window process followed? Is there a legitimate automation principal that should be baselined? Should a preventive guardrail — an SCP or permission boundary — exist so this cannot happen rather than being detected after it does?

**Detection catching a privilege escalation is good. Prevention making it impossible is better.**
