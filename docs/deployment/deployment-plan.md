# Deployment Plan — PLANNED

> **Status: not started. No AWS resource has been created.**
> **This plan does not execute until the explicit instruction "APPROVE AWS DEPLOYMENT" is given.**

Phased, with a checkpoint after each. Nothing proceeds until the current phase is verified working, documented, and proven reversible.

Acceptance criteria: [`../testing/acceptance-criteria.md`](../testing/acceptance-criteria.md) · Rollback: [`rollback-plan.md`](rollback-plan.md)

---

## Phase 0 — Prerequisites

Complete [`aws-prerequisites.md`](aws-prerequisites.md) in full.

**Critical ordering:** the budget, billing alarm and teardown plan exist **before** the first chargeable resource. Configuring alerts afterwards means the first thing they can warn about is a bill already incurred.

**Checkpoint:** every prerequisite ticked · region selected and recorded · `[OPEN QUESTION]` items resolved or explicitly accepted.

---

## Phase 1 — Network foundation

Create VPC, subnet, route table, internet gateway, security groups. **No compute.**

- Security groups created with **zero inbound rules**
- Every rule carries a description
- All resources tagged with the five required tags

**Checkpoint (AC-10 to AC-13):**

- Network exists and is tagged
- Security group rules reviewed line by line — inbound from internet CIDRs: zero
- **Phase torn down and rebuilt once, to prove rollback works before anything depends on it**

Free-tier components; cost exposure negligible.

---

## Phase 2 — Logging

Enable CloudTrail (one management trail, all regions, to S3) and VPC Flow Logs (to S3). S3 bucket encrypted, public access blocked, lifecycle expiry applied at creation.

**Checkpoint (AC-20 to AC-23):**

- **A deliberate test action appears in CloudTrail.** Observing an event, not confirming the configuration looks correct.
- Flow log objects landing in S3
- Lifecycle rules active from day one
- Exactly one trail; no data events, no Insights

---

## Phase 3 — Wazuh

Launch `wazuh-aio` with `role-wazuh`. Run the installation assistant. Change every generated credential. Confirm the stack is healthy.

**[VERIFIED FACT]** `curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a` ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)). Confirm the current version at deployment rather than assuming 4.14.

**Checkpoint (AC-30 to AC-34):**

- Stack healthy
- **All default credentials changed**
- Dashboard reachable via SSM port forwarding **and confirmed unreachable from the internet by testing from outside AWS**
- **AC-33: performance acceptable at the documented 4 vCPU / 8 GiB**, with a baseline recorded for any later, evidence-based downsizing experiment
- Healthy after a full stop/start cycle

---

## Phase 4 — Agents and ingestion

Launch `target-01`. Install and enrol the Wazuh agent. Configure the AWS module for CloudTrail and flow logs. Verify parsing.

**Checkpoint (AC-40 to AC-44):**

- Events from all three sources visible and correctly parsed
- Timestamps UTC and consistent across sources
- **AC-44: stop the agent and confirm an alert fires.** A host going dark must be detected, not read as calm. This is the single most important check in the phase.

---

## Phase 5 — Detection content

Implement detections one at a time. Each is validated before the next is added.

**Checkpoint (AC-50 to AC-54):**

- Each rule fires on its positive test case
- **Each rule stays silent on its negative test case**
- False positives observed and documented, or a stated reason none was
- Every rule has a response action
- Thresholds set from an observed baseline, and the baseline recorded

Detections are committed to git as they are validated, so rollback is `git revert`.

---

## Phase 6 — Shuffle SOAR

Launch `shuffle` with `role-shuffle`. Install via Docker. **Set `vm.max_map_count` persistently** in `/etc/sysctl.d/`, not with `sysctl -w` alone. Set admin credentials at first login. Connect the Wazuh webhook. Build enrichment workflows.

**Checkpoint (AC-60 to AC-65):**

- Admin credentials set
- UI reachable via SSM only; **9200 confirmed unreachable from the internet**
- An alert reaches Shuffle and returns enriched
- **AC-63: `vm.max_map_count` survives a reboot**
- **AC-64: approval gate halts the workflow** — verified by triggering a disruptive action and confirming it stops. Reading the workflow is not the test.
- **AC-65: enrichment failure does not suppress the alert** — invalidate the API key and confirm the alert still reaches the analyst

---

## Phase 7 — End-to-end validation

Run every scenario start to finish: trigger → detect → enrich → triage → approve → respond → document.

**Checkpoint (AC-70 to AC-75):** every scenario completes and produces a written incident record. Timings and observations recorded with method and date.

---

## Phase 8 — Documentation

Update the repository with **what was actually built, actually measured and actually observed** — including what did not work and where this plan was wrong.

**Checkpoint (AC-100 to AC-104):**

- No claim unsupported by a recorded observation
- No metric without its measurement method
- Diagram title updated from PLANNED; undeployed components still dashed
- **Documentation corrected wherever deployment proved it wrong**
- Limitations written honestly

---

## Operating discipline throughout

- **Stop compute at the end of every session** — the largest cost lever
- Check the cost dashboard at the start of each session, not the end
- Never open an inbound rule, not even temporarily
- Record results after they happen, never in advance

See [`../operations/lab-runbook.md`](../operations/lab-runbook.md).

## Rollback

Every phase is independently reversible. **If a phase cannot be cleanly torn down, that is a finding** — fix the teardown procedure before proceeding. See [`rollback-plan.md`](rollback-plan.md).

## Related

[`aws-prerequisites.md`](aws-prerequisites.md) · [`cost-model.md`](cost-model.md) · [`teardown-plan.md`](teardown-plan.md) · [`rollback-plan.md`](rollback-plan.md) · [`../architecture/final-aws-architecture.md`](../architecture/final-aws-architecture.md)

## Sources

- [Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)
- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
