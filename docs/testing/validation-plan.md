# Validation Plan

**Status:** Planned. **No validation has been performed.**

## Principle

**A detection that has never fired in a test is an assumption, not a control.**

Validation answers three questions: does it fire when it should, does it stay quiet when it should, and does the analyst receive something they can act on.

## Scope

Every layer, because a failure anywhere makes everything downstream look healthy:

| Layer | Validates |
|---|---|
| Collection | Events arrive, complete and parsed |
| Detection | Rules fire correctly; thresholds behave |
| Enrichment | Context attaches; failures degrade gracefully |
| Alerting | Analyst receives an actionable alert |
| Response | Approval gates hold; actions work |
| End-to-end | The whole chain, scenario by scenario |

## Method per detection

1. **Precondition** — confirm the data source is arriving
2. **Baseline** — record normal for the metric involved
3. **Negative test** — activity below threshold: confirm **no** alert
4. **Positive test** — activity crossing threshold: confirm alert
5. **Content check** — severity, fields, and enrichment correct
6. **Response check** — approval gate present on disruptive actions
7. **Record** — outcome, date, observations, in [`test-cases.md`](test-cases.md)

**The negative test matters as much as the positive one.** A rule that fires on everything is as useless as one that never fires, and it is more damaging, because it trains people to ignore alerts.

## Pipeline health checks

Silent failure is the real risk — an agent stops reporting and the console looks calm.

| Check | Confirms | Cadence |
|---|---|---|
| Agent heartbeat | All agents reporting | Continuous |
| CloudTrail delivery | Events landing in S3 | Daily |
| Flow log delivery | Network telemetry arriving | Daily |
| Wazuh → Shuffle webhook | Alerts reaching SOAR | Per alert |
| Enrichment API | Service reachable; key valid | Daily |
| Time sync | Clocks aligned across sources | Daily |
| Disk / index capacity | Ingestion will not stop | Weekly |

**Absence of alerts is not evidence of absence of activity.** It is equally consistent with a broken pipeline. These checks distinguish the two.

## Failure-mode tests

Deliberately break things and confirm the failure is visible:

- Stop an agent → confirm a heartbeat alert fires
- Invalidate the enrichment API key → confirm enrichment fails gracefully and **the alert still reaches the analyst**
- Break the SOAR webhook → confirm the failure is detected, not silently swallowed
- Introduce clock drift → confirm it is detected

The enrichment test is the most important: an enrichment dependency that suppresses alerts on failure is worse than having no enrichment.

## Safety rules for all testing

- Simulation originates from and targets **OxNeural lab systems only**
- No third-party system is scanned, probed or tested — ever
- Non-destructive techniques only; no real malware, no exploit code
- Simulation hosts are disposable and rebuilt afterwards
- Testing is scheduled and documented in advance, so alerts it generates can be attributed

## Recording results

Results are recorded **after** the test, with the date and what was actually observed — including tests that failed and detections that did not work. A validation log showing only successes has not been kept honestly.

No timings, rates or performance figures appear anywhere in this repository until measured, with the method stated.

## Re-validation

Re-run after: rule changes, Wazuh or Shuffle upgrades, new log sources, and quarterly regardless. Detections rot quietly as the environment moves.

## Related

[`acceptance-criteria.md`](acceptance-criteria.md) — what "done" means per phase · [`test-cases.md`](test-cases.md) · [`../deployment/deployment-plan.md`](../deployment/deployment-plan.md)
