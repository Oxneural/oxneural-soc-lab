# Incident Response Process

**Status:** Documented. **Not exercised — no incident has occurred in this lab.**

Adapted from NIST SP 800-61 for a small lab environment. Scaled to fit; not diluted.

## Phases

```
Preparation → Detection & Analysis → Containment → Eradication → Recovery → Post-Incident
```

## 1. Preparation

Detections written and validated · playbooks documented · evidence procedure defined · approval gates configured · contacts and escalation path known · logging verified working.

**Preparation is the only phase with unlimited time.** Everything not decided here gets decided badly under pressure.

## 2. Detection and analysis

1. **Validate** — is the alert real, or a known false positive?
2. **Scope** — which hosts, accounts, data are involved?
3. **Build the timeline** — first evidence of activity, not first alert. They are rarely the same.
4. **Assess impact** — what was accessed, changed, or taken?
5. **Classify severity** (below)
6. **Record the disposition**

**Establish what happened before deciding what it means.** The most common investigation failure is forming a conclusion early and then interpreting evidence to fit it.

## 3. Containment

Short-term: stop the immediate harm — isolate, disable, block.
Long-term: keep it contained while eradication is prepared.

**Preserve evidence before containing where the situation allows.** Isolating a host may drop volatile memory; rebuilding destroys the answer to how it was compromised. Where containment and evidence conflict, containment wins on a live threat — but the choice is recorded.

**Every containment action that affects availability requires approval.** See the gates in each playbook.

## 4. Eradication

Remove the cause: persistence, unauthorised access, malicious artefacts, and the vulnerability that permitted entry. Rotate every credential that could have been exposed.

Eradication that fixes the symptom and leaves the entry path open guarantees recurrence.

## 5. Recovery

Restore from a known-good state · verify integrity · restore access with rotated credentials · **monitor closely for recurrence** — returning to normal monitoring immediately is how a partially eradicated intrusion resurfaces unnoticed.

## 6. Post-incident

Within a week, while memory is accurate:

- What happened, in sequence
- What worked in the response
- What did not
- What detection gap allowed it, and what closes it
- What playbook step was wrong or missing

**Blameless.** The purpose is a better system, not a culprit. A process that punishes reporting produces silence, not safety.

## Severity classification

| Severity | Definition | Response |
|---|---|---|
| **Critical** | Confirmed compromise with active impact; credential theft; data loss; logging disabled | Immediate, everything else stops |
| **High** | Strong indication of compromise; privilege escalation; persistence found | Prompt |
| **Medium** | Suspicious activity requiring investigation; no confirmed compromise | Same day |
| **Low** | Notable, likely benign, context needed | Routine |

Severity is reassessed as evidence arrives. Incidents move in both directions — downgrading on evidence is as correct as escalating.

## Evidence handling

- **Capture before remediating** wherever possible
- Preserve originals; work from copies
- Record what was collected, when, by whom, from where
- Timestamps in UTC throughout
- Nothing sensitive leaves the lab; nothing is committed to this repository

## Approval gates

| Action | Approval |
|---|---|
| Investigate, collect evidence, enrich | None |
| Increase monitoring | None |
| Notify | None |
| **Block address** | **Required** |
| **Disable account or key** | **Required** |
| **Isolate host** | **Required** |
| **Terminate or rebuild host** | **Required** |
| **Revert configuration** | **Required** |

Automation prepares the action and presents the context. **A human decides.** This is the discipline the lab exists to practise — and the reason a false positive here does not become an outage.

## Documentation

Every incident, regardless of outcome: what fired, what was checked, what was concluded, what was done, what evidence was kept, what changed as a result.

An incident closed without a written reason is an incident nobody can learn from.

## Playbooks

[`brute-force-playbook.md`](brute-force-playbook.md) · [`unauthorized-iam-playbook.md`](unauthorized-iam-playbook.md) · [`malware-lab-playbook.md`](malware-lab-playbook.md)
