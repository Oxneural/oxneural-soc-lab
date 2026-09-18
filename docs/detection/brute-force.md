# Detection: Brute-Force Authentication Activity

**Status:** Documented. **Not implemented, not validated.**
**ATT&CK:** T1110 Brute Force · T1110.001 Password Guessing · T1110.003 Password Spraying

## 1. Detection objective

Identify repeated failed authentication attempts indicating credential guessing against lab hosts or the AWS console, and distinguish them from ordinary user error.

Both patterns matter and they look different:

- **Brute force** — many attempts against one account
- **Password spraying** — few attempts across many accounts, deliberately staying under per-account lockout thresholds

A rule that only counts failures per account will miss spraying entirely.

## 2. Data source

| Source | Provides | Collected by |
|---|---|---|
| Linux `/var/log/auth.log` or `secure` | SSH authentication | Wazuh agent |
| Windows Security Event Log 4625 | Failed logon | Wazuh agent |
| AWS CloudTrail `ConsoleLogin` | Console sign-in failures | CloudTrail → Wazuh |

**Confirm collection before writing logic.** A rule cannot fire on telemetry that is not arriving.

## 3. Detection logic

Wazuh correlates by source IP and user over a time window. Two complementary rules:

**A — Brute force (per-account)**
Failed authentication, same source IP, same target account, threshold N within window W.

**B — Password spraying (cross-account)**
Failed authentication, same source IP, **distinct** target accounts exceeding M within a longer window.

**C — Success after failures**
Successful authentication from a source IP that produced failures within the preceding window. **This is the highest-value rule of the three** — it is the one that indicates the attempt may have worked.

### Threshold justification

Thresholds are set after baselining normal failure rates in the lab, and recorded here with the date measured. No numbers are stated yet, because nothing has been measured. Setting them now would be inventing them.

## 4. Expected alert

```
Rule:        Brute-force authentication — multiple failures from single source
Severity:    Medium  (High if followed by success)
Source IP:   <srcip>
Target:      <srcuser> on <agent>
Count:       <n> failures in <window>
Window:      <start> – <end> UTC
```

## 5. Analyst investigation

1. **Is the source internal or external?** Internal points to a misconfigured service or saved credential far more often than an attacker.
2. **Did any attempt succeed?** Search successful authentications from that source across the full window, not just after the alert.
3. **One account or many?** Distinguishes brute force from spraying.
4. **Does the account exist?** Failures against non-existent accounts indicate enumeration.
5. **Is the timing human or automated?** Precise regular intervals indicate tooling.
6. **What is the source's reputation and geography?** Enrichment — context, not verdict.
7. **If successful: what happened next?** Commands run, IAM calls made, data accessed. **This is the question that determines whether it is an incident.**

## 6. Enrichment

Automatic, via Shuffle: IP reputation, geolocation and ASN, whether the source is previously seen in the lab, whether the target account exists and its privilege level, recent successful logins for that account.

## 7. Severity considerations

**Raises severity:** success after failures · targeting privileged or root accounts · source with poor reputation · sustained or distributed attempts · console rather than host.

**Lowers severity:** known internal source · single account with a plausible typo pattern · immediately after a password change · known service account with a stale credential.

## 8. False-positive considerations

Common and expected:

- Users mistyping passwords, especially after a change
- Service accounts with expired cached credentials — often produces a perfectly regular automated-looking pattern
- Backup or monitoring agents with stale credentials
- A misconfigured script retrying in a loop
- Internet-facing SSH receiving constant untargeted background scanning — **expected, not an incident on its own**

The last is why "success after failures" carries the real signal.

## 9. Response

| Action | Approval |
|---|---|
| Investigate and document | None |
| Enrich with threat intelligence | None — automatic |
| Notify the account owner | None |
| **Block source IP** | **Required** |
| **Disable the account** | **Required** |
| **Force password reset** | **Required** |
| Escalate a confirmed compromise | Per playbook |

Blocking an address automatically is how a NAT gateway carrying legitimate users gets blocked. The gate exists for that reason.

Playbook: [`../incident-response/brute-force-playbook.md`](../incident-response/brute-force-playbook.md)

## 10. Validation

Generate failed authentications from a controlled lab host against a lab target, at a rate crossing the threshold. Then repeat with a success following failures to validate rule C.

Recorded in [`../testing/test-cases.md`](../testing/test-cases.md). **All simulation is against OxNeural lab systems only.**

## 11. Evidence to capture

Raw authentication log entries covering the full window · source IP, ports, timestamps in UTC · target accounts and outcomes · any successful authentication from the source · subsequent session activity if success occurred · the alert itself with rule ID and severity.

**Capture before remediating.** Rebuilding the host destroys the answer to how it was compromised.
