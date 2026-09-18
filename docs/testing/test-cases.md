# Test Cases

**Status:** Defined. **None executed. Every result column is empty because no test has been run.**

Results are filled in after execution, with the date and what was actually observed.

## How to read this

| Column | Meaning |
|---|---|
| Expected | What should happen |
| Result | What actually happened — blank until run |
| Date | When it was run |

**Nothing is marked Pass in advance.** A pre-filled result is a fabricated result.

---

## TC-01 — Brute force, per-account threshold

**Detection:** [`../detection/brute-force.md`](../detection/brute-force.md) rule A
**Setup:** Authorised lab host → lab target
**Steps:** Repeated failed SSH authentications against one account, crossing the configured threshold within the window.
**Expected:** One alert, severity Medium, correct source IP, target account and failure count.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-02 — Brute force, below threshold (negative test)

**Steps:** Failed authentications **below** the threshold.
**Expected:** **No alert.**

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-03 — Password spraying, cross-account

**Detection:** rule B
**Steps:** Few failed attempts across many distinct accounts from one source, each account staying below the per-account threshold.
**Expected:** Spraying alert fires even though rule A does not.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-04 — Success after failures

**Detection:** rule C — **highest-value case**
**Steps:** Failed attempts, then a successful authentication from the same source.
**Expected:** Alert at High severity, referencing both the failures and the success.

| Result | Date | Observations |
|---|---|---|
| | | |

---

## TC-05 — IAM: privileged policy attachment

**Detection:** [`../detection/unauthorized-iam-activity.md`](../detection/unauthorized-iam-activity.md) rule A
**Setup:** Lab AWS account, authorised test principal
**Steps:** Create a test IAM user; attach a policy granting broad permissions.
**Expected:** High severity alert, correct actor attribution, policy document captured in enrichment.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-06 — IAM: access key creation

**Detection:** rule B
**Steps:** Create an access key for the test user. **Delete it immediately after the test.**
**Expected:** Alert identifying actor and target identity.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-07 — IAM: denied action (negative-ish test)

**Steps:** Attempt an IAM action the principal is not permitted to perform.
**Expected:** Event recorded with `AccessDenied`; alert reflects that it did not succeed.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-08 — IAM: control tampering

**Detection:** rule D — **highest severity**
**Steps:** In the lab account, disable a non-primary CloudTrail trail. **Re-enable immediately.**
**Expected:** Critical alert.

| Result | Date | Observations |
|---|---|---|
| | | |

---

## TC-09 — Network: port scan

**Detection:** [`../detection/suspicious-network-activity.md`](../detection/suspicious-network-activity.md) rule A
**Steps:** Controlled port scan from one lab host against another lab host.
**Expected:** Scanning alert; rejected connections visible in flow logs.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-10 — Network: simulated beaconing

**Detection:** rule D
**Steps:** Scripted outbound connection to a known-benign public endpoint at a fixed interval, small consistent payload.
**Expected:** Beaconing alert after the pattern establishes.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-11 — Network: normal traffic (negative test)

**Steps:** Ordinary package update and routine lab traffic.
**Expected:** **No alert.** If this produces alerts, thresholds are wrong.

| Result | Date | Observations |
|---|---|---|
| | | |

---

## TC-12 — Host: EICAR test file

**Detection:** [`../detection/malware-simulation.md`](../detection/malware-simulation.md) rule A
**Steps:** Write the EICAR test string to a monitored path on a disposable lab host.
**Expected:** Alert on file integrity detection.
**Safety:** EICAR is a harmless standard test string. Not malware.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-13 — Host: rapid mass file modification

**Detection:** rule B
**Steps:** Encrypt a directory of **synthetic throwaway files** with a standard archiving tool, key retained. Reverse immediately.
**Expected:** Critical alert on modification rate.
**Safety:** Synthetic data only, disposable host, fully reversible, host rebuilt after.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-14 — Host: persistence artefact

**Detection:** rule C
**Steps:** Create a benign file in a startup location. Remove after the test.
**Expected:** Alert on persistence establishment.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-15 — Host: suspicious process ancestry

**Detection:** rule D
**Steps:** Spawn a shell from an unexpected parent process.
**Expected:** Alert including the full parent chain.

| Result | Date | Observations |
|---|---|---|
| | | |

---

## TC-16 — Enrichment: benign address

**Steps:** Trigger enrichment on a well-known public DNS resolver.
**Expected:** Clean reputation returned and attached.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-17 — Enrichment: private address filtered

**Steps:** Trigger enrichment on an RFC1918 address.
**Expected:** Filtered before any external query. **No internal address is sent to a third-party service.**

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-18 — Enrichment: graceful failure

**Steps:** Invalidate the enrichment API key; trigger an alert.
**Expected:** Enrichment fails, and **the alert still reaches the analyst.**
**Why this matters most:** enrichment must never suppress an alert.

| Result | Date | Observations |
|---|---|---|
| | | |

---

## TC-19 — Approval gate holds

**Steps:** Trigger a scenario whose workflow includes a blocking action.
**Expected:** Workflow **halts** and waits for human approval. No disruptive action executes automatically.

| Result | Date | Observations |
|---|---|---|
| | | |

## TC-20 — Agent failure is visible

**Steps:** Stop the Wazuh agent on a lab host.
**Expected:** Heartbeat alert fires. **Silence is detected as silence, not read as calm.**

| Result | Date | Observations |
|---|---|---|
| | | |

---

## Execution log

| Date | Tests run | Passed | Failed | Notes |
|---|---|---|---|---|
| | | | | |
