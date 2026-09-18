# Deployment Plan

**Status:** Not started. **No resource has been created.**

Phased, with a checkpoint after each phase. Nothing proceeds to the next phase until the current one is verified working and documented.

## Phase 0 — Prerequisites

Complete [`prerequisites.md`](prerequisites.md) in full. Billing alerts and the teardown plan exist before anything chargeable is created.

**Checkpoint:** every prerequisite box ticked.

## Phase 1 — Network foundation

Create the VPC, subnets, route tables, internet gateway and security groups. No compute yet.

- Security groups source-restricted; no `0.0.0.0/0` on management ports
- Every rule carries a description
- All resources tagged

**Checkpoint:** network exists, security group rules reviewed line by line, teardown of this phase tested — create it, destroy it, confirm nothing orphaned.

## Phase 2 — Logging

Enable CloudTrail (all regions, to S3, with lifecycle expiry) and VPC Flow Logs. Confirm events are actually landing.

**Checkpoint:** a deliberate test action — for example an IAM read — appears in CloudTrail. Logging that has not been observed working is an assumption.

## Phase 3 — Wazuh

Deploy the Wazuh server. Secure the dashboard behind source restriction. Change all default credentials. Confirm the stack is healthy.

**Checkpoint:** dashboard reachable only from the permitted source; default credentials confirmed changed; services healthy after a reboot.

## Phase 4 — Agents and ingestion

Install the Wazuh agent on the target host. Configure CloudTrail and VPC Flow Log ingestion. Verify normalisation.

**Checkpoint:** events from all three sources visible and correctly parsed; timestamps in UTC and consistent across sources.

## Phase 5 — Detection content

Implement the documented detections one at a time. Each is validated against its test case before the next is added.

**Checkpoint:** every rule has fired in a test; results recorded in [`../testing/test-cases.md`](../testing/test-cases.md); false positives observed and documented.

## Phase 6 — Shuffle SOAR

Deploy Shuffle. Connect the Wazuh alert webhook. Build enrichment workflows.

**Checkpoint:** an alert reaches Shuffle and returns enriched. **Approval gates confirmed present on every disruptive action** — verified by attempting one and confirming it halts for approval.

## Phase 7 — Playbooks and end-to-end validation

Run each scenario end to end: trigger → detect → enrich → triage → respond → document.

**Checkpoint:** every documented scenario completes end to end; timings and observations recorded with method and date.

## Phase 8 — Documentation and evidence

Update this repository with what was actually built, actually measured and actually observed — including what did not work.

**Checkpoint:** no claim in the repository that is not supported by a recorded observation.

## Operating discipline

- **Stop compute when not in use.** Running a lab overnight teaches nothing and bills continuously.
- Check the cost dashboard weekly.
- Re-read the teardown plan monthly; confirm it still matches reality.

## Rollback

Every phase is individually reversible. If a phase cannot be cleanly torn down, that is a finding — document it and fix the teardown procedure before proceeding.

## Recording results

Deployment results, measurements and problems go in this repository **after** they happen, with dates. Nothing is written up in advance.
