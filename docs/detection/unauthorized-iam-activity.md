# Detection: Unauthorized IAM Activity

**Status:** Documented. **Not implemented, not validated.**
**ATT&CK:** T1098 Account Manipulation · T1078.004 Valid Accounts: Cloud Accounts · T1136.003 Create Account: Cloud Account

## 1. Detection objective

Identify identity and permission changes in AWS that indicate privilege escalation or persistence establishment.

In a cloud environment, identity is the perimeter. An attacker with valid credentials does not need an exploit — they need a policy attachment. IAM changes are therefore among the highest-value events available, and among the least noisy in a stable environment.

## 2. Data source

| Source | Provides |
|---|---|
| CloudTrail | All IAM API calls, with actor, source IP, user agent, outcome |

Key events: `CreateUser` · `CreateAccessKey` · `AttachUserPolicy` · `AttachRolePolicy` · `PutUserPolicy` · `CreateRole` · `UpdateAssumeRolePolicy` · `CreateLoginProfile` · `DeactivateMFADevice` · `DeleteTrail` · `StopLogging` · `ConsoleLogin` by root.

## 3. Detection logic

Rules on CloudTrail events ingested into Wazuh:

**A — Privileged policy attachment.** Any `AttachUserPolicy` / `AttachRolePolicy` / `PutUserPolicy` where the policy grants administrative or wildcard permissions.

**B — Access key creation.** Any `CreateAccessKey`, especially where the actor differs from the target user. Long-lived keys are a persistence mechanism.

**C — New identity creation.** `CreateUser`, `CreateRole`, or `CreateLoginProfile` on an existing user (adding console access to a programmatic identity).

**D — Security control tampering.** `DeleteTrail`, `StopLogging`, `DeactivateMFADevice`, or deletion of a config rule. **Highest severity — this is an attacker turning off the lights.**

**E — Root account usage.** Any root `ConsoleLogin` or root API activity. Root should be effectively unused; any use is notable.

**F — Trust policy modification.** `UpdateAssumeRolePolicy` — quiet, powerful, and easily missed; it can grant an external principal the ability to assume a role.

### Threshold justification

IAM changes in a stable lab are rare, so these are largely single-event rules rather than threshold rules. That is deliberate: rarity is what makes them high-signal.

## 4. Expected alert

```
Rule:        Unauthorized IAM activity — <event name>
Severity:    High  (Critical for control tampering or root usage)
Actor:       <principal / assumed role>
Source IP:   <srcip>
Target:      <affected identity>
Event:       <API call>
Outcome:     Success / AccessDenied
Time:        <timestamp> UTC
```

## 5. Analyst investigation

1. **Who made the change?** Principal, assumed role, session name.
2. **Is this an expected change?** A planned administrative action, or unannounced?
3. **Where from?** Source IP and user agent. A console user agent from an unfamiliar address differs sharply from a known CI pipeline.
4. **What exactly was granted?** Read the policy document. "AdministratorAccess" and a narrow scoped policy are not the same finding.
5. **What else did this principal do?** Pull the full CloudTrail history for the session — **IAM changes rarely occur alone**.
6. **How did the principal obtain access?** Trace back to the authentication event.
7. **Was anything else changed?** Data access, resource creation, logging modification.

## 6. Enrichment

Actor's normal activity baseline · source IP reputation and geolocation · whether the source has been seen for this principal before · full policy document · other events in the same session · whether the actor used MFA.

## 7. Severity considerations

**Raises severity:** logging or MFA disabled · root usage · administrative or wildcard policy · unfamiliar source IP · outside working hours · sequence of several IAM events · trust policy modified to allow an external account.

**Lowers severity:** known administrator from a known address during a planned change window · `AccessDenied` outcome — still worth reviewing, but the action did not succeed · action performed by an approved automation role matching its normal pattern.

## 8. False-positive considerations

- Planned administrative work that was not announced — **the most common cause**; fixed by process, not by disabling the rule
- Infrastructure-as-code applying changes through a CI role
- Onboarding creating users and attaching policies legitimately
- Automated key rotation creating and deleting keys

Tune by baselining known automation principals — not by disabling the rule.

## 9. Response

| Action | Approval |
|---|---|
| Investigate and document | None |
| Retrieve full session history | None |
| Enrich | None — automatic |
| Contact the actor to confirm intent | None |
| **Revoke access key** | **Required** |
| **Detach policy / revert change** | **Required** |
| **Disable IAM identity** | **Required** |
| **Force session revocation** | **Required** |
| Escalate confirmed compromise | Per playbook |

Reverting an IAM change can break a legitimate system mid-operation. Confirm intent first where the situation allows.

Playbook: [`../incident-response/unauthorized-iam-playbook.md`](../incident-response/unauthorized-iam-playbook.md)

## 10. Validation

In the lab account, using an authorised test principal: create a test IAM user, attach a scoped policy, create and immediately delete an access key, attempt an action that will be denied. Confirm each produces the expected alert with correct actor attribution.

Recorded in [`../testing/test-cases.md`](../testing/test-cases.md). **Lab account only.**

## 11. Evidence to capture

Full CloudTrail record of the event, JSON preserved · complete session history for the principal · the policy document before and after · authentication event that created the session · MFA status · any resource created or accessed in the session.

**Preserve before reverting.** Reverting the change removes the evidence of what was granted.
