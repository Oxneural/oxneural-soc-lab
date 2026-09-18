# Detection Engineering

**Status:** Method documented. **No detection has been implemented or validated.**

## Principles

**A detection is a hypothesis about attacker behaviour, expressed in data.** It is only useful if an analyst can act on it.

1. **Every alert has a defined response.** An alert nobody knows how to handle is noise that trains people to ignore the console.
2. **Detect behaviour, not artefacts.** Indicators change in minutes; behaviour is expensive for an attacker to change.
3. **False positives are expected and documented.** A detection claiming zero false positives has usually not run against real volume.
4. **Tune, never disable.** A disabled rule shows as coverage on a report and provides none.
5. **Untested means unknown.** A rule that has never fired in a test is an assumption.
6. **Original or attributed.** Detection content is written here or used under its open-source licence with attribution. Never copied from a client, employer or commercial ruleset.

## Required structure

Every detection document in this directory contains all eleven sections:

| # | Section | Must answer |
|---|---|---|
| 1 | Detection objective | What behaviour, and why it matters |
| 2 | Data source | Which telemetry, and confirmation it is collected |
| 3 | Detection logic | The rule, with thresholds justified |
| 4 | Expected alert | What the analyst sees, with severity |
| 5 | Analyst investigation | Steps a second analyst can follow unaided |
| 6 | Enrichment | What context is added automatically |
| 7 | Severity considerations | What raises or lowers severity |
| 8 | False-positive considerations | Known benign causes |
| 9 | Response | Actions, and which need approval |
| 10 | Validation | How it is tested |
| 11 | Evidence to capture | What to preserve, before remediating |

A document missing any section is incomplete.

## Thresholds

Every threshold is justified in writing. "5 failures in 5 minutes" without a reason is a guess that will be defended forever because nobody remembers it was arbitrary.

Baseline first where possible: observe normal, then set the threshold against it, then record what normal looked like and when it was measured.

## Severity

| Severity | Meaning | Response expectation |
|---|---|---|
| Critical | Confirmed or highly likely compromise with active impact | Immediate |
| High | Strong indication of malicious activity | Prompt |
| Medium | Suspicious; requires investigation | Same day |
| Low | Notable; context-dependent | Routine review |
| Info | Recorded for correlation | No individual response |

Severity reflects demonstrated impact in this environment. Inflating severity to make a detection look important destroys trust in every other alert.

## Lifecycle

```
Hypothesis → Data check → Logic → Lab test → FP analysis
    → Document → Deploy → Observe → Tune → Review
```

A detection is "done" only after it has fired in a test, produced at least one observed false positive or a documented reason it cannot, and has a written response.

## Tuning record

Every tuning change records what changed, why, who approved it, and when. Exceptions added during an incident routinely outlive their justification.

| Date | Rule | Change | Reason | By |
|---|---|---|---|---|
| | | | | |

## ATT&CK mapping

Detections map to MITRE ATT&CK techniques where applicable. Coverage is reported **including gaps** — a coverage map showing only what is covered is marketing, not engineering.

## Scenarios

[`brute-force.md`](brute-force.md) · [`unauthorized-iam-activity.md`](unauthorized-iam-activity.md) · [`suspicious-network-activity.md`](suspicious-network-activity.md) · [`malware-simulation.md`](malware-simulation.md) · [`../threat-intelligence/ip-reputation.md`](../threat-intelligence/ip-reputation.md)
