# Acceptance Criteria — PLANNED

> **PLANNED. Nothing has been deployed or tested. Every result column is empty because no test has run.**

What "done" means for each phase. A phase is not complete until its criteria are met **and recorded** — not when it looks like it works.

**Nothing is marked Pass in advance. A pre-filled result is a fabricated result.**

---

## Phase 1 — Design (this phase)

| ID | Criterion | Met |
|---|---|---|
| AC-01 | Architecture documented with verified facts cited to official sources | ✓ |
| AC-02 | Every AWS resource has a stated cost category, stop behaviour and teardown step | ✓ |
| AC-03 | Sizing deviations recorded explicitly, with a test defined | ✓ |
| AC-04 | Teardown and rollback plans written **before** deployment | ✓ |
| AC-05 | IAM model documents permissions rather than granting AdministratorAccess | ✓ |
| AC-06 | No AWS resource created, no credential created, no secret created | ✓ |

---

## Phase 2 — Network

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-10 | VPC, subnet, route table, IGW created and tagged | | |
| AC-11 | **Every security group has zero inbound rules from any internet CIDR** | | |
| AC-12 | Every security group rule carries a description | | |
| AC-13 | Phase torn down and rebuilt cleanly — rollback proven, not assumed | | |

---

## Phase 3 — Logging

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-20 | CloudTrail trail delivering to S3; a deliberate test action appears in it | | |
| AC-21 | VPC Flow Logs delivering to S3 | | |
| AC-22 | S3 bucket: encrypted, public access blocked, lifecycle expiry applied | | |
| AC-23 | Exactly one trail exists; no data events, no Insights | | |

**AC-20 requires observing a real event, not confirming the configuration looks right.** Logging that has not been seen working is an assumption.

---

## Phase 4 — Wazuh

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-30 | Wazuh stack installed and healthy | | |
| AC-31 | **All installer-generated default credentials changed** | | |
| AC-32 | Dashboard reachable via SSM port forwarding **and NOT from the internet** | | |
| AC-33 | **Sizing deviation validated**: indexing and dashboard usable at 2 vCPU | | |
| AC-34 | Stack healthy after a full stop/start cycle | | |

**AC-33 is the recorded deviation from Wazuh's documented 4 vCPU.** If it fails, the documented remedy is one instance size up. **Record the failure — a documented deviation that proved wrong is a useful finding.**

**AC-32 is verified by attempting to reach the dashboard from outside AWS and confirming it does not respond.** Confirming the security group looks right is not the same test.

---

## Phase 5 — Ingestion

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-40 | Agent enrolled on target-01 and reporting | | |
| AC-41 | CloudTrail events ingested and parsed correctly | | |
| AC-42 | Flow log events ingested and parsed correctly | | |
| AC-43 | Timestamps in UTC and consistent across all three sources | | |
| AC-44 | **Agent silence triggers an alert** — verified by stopping the agent | | |

**AC-44 is the most important criterion in this phase.** A host going dark must be detected, not read as calm.

---

## Phase 6 — Detection

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-50 | Each detection fires on its positive test case | | |
| AC-51 | **Each detection stays silent on its negative test case** | | |
| AC-52 | Each detection documents at least one observed false positive, or a stated reason none was observed | | |
| AC-53 | Every rule has a defined response action | | |
| AC-54 | Thresholds set from an observed baseline, and the baseline recorded | | |

**AC-51 matters as much as AC-50.** A rule that fires on everything is worse than one that never fires — it trains people to ignore alerts.

Test cases: [`test-cases.md`](test-cases.md)

---

## Phase 7 — Shuffle

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-60 | Shuffle running; admin credentials set (no defaults exist) | | |
| AC-61 | UI reachable via SSM only; **9200 not reachable from the internet** | | |
| AC-62 | Wazuh alert reaches Shuffle and returns enriched | | |
| AC-63 | **`vm.max_map_count` persistent across reboot** | | |
| AC-64 | **Approval gate halts the workflow** — verified by triggering a disruptive action | | |
| AC-65 | **Enrichment failure does not suppress the alert** | | |

**AC-64 is verified by attempting an action and confirming it stops.** Reading the workflow and seeing a gate drawn in it is not the test.

**AC-65:** invalidate the API key, trigger an alert, confirm it still reaches the analyst — unenriched and flagged as such.

---

## Phase 8 — End to end

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-70 | Brute force: trigger → detect → enrich → triage → approve → respond → document | | |
| AC-71 | Unauthorized IAM: same, end to end | | |
| AC-72 | Suspicious network: same, end to end | | |
| AC-73 | Malware simulation: same, on a disposable host, host rebuilt after | | |
| AC-74 | Suspicious IP enrichment: same | | |
| AC-75 | Every scenario produces a written incident record | | |

---

## Cost acceptance

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-80 | Budget and billing alarm live **before** the first chargeable resource | | |
| AC-81 | Every resource carries all five required tags | | |
| AC-82 | No NAT gateway, no Elastic IP, no load balancer exists | | |
| AC-83 | Actual monthly cost within budget — from the bill, not a projection | | |
| AC-84 | Instances confirmed stopped at the end of every session | | |

---

## Teardown acceptance

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-90 | Console filtered by `Project=oxneural-soc-lab` returns zero resources | | |
| AC-91 | **Every region checked**, not only the one used | | |
| AC-92 | No orphaned EBS volume, snapshot, AMI or Elastic IP | | |
| AC-93 | S3 bucket emptied and deleted | | |
| AC-94 | All IAM lab identities, roles and policies removed; zero access keys | | |
| AC-95 | **Cost falls to zero over the following 48 hours** — verified in Cost Explorer | | |
| AC-96 | Budget and billing alarm retained until AC-95 confirms zero | | |

**AC-95 is the criterion that actually proves teardown worked.** A console that looks empty and a bill that keeps arriving means something was missed.

---

## Documentation acceptance

| ID | Criterion | Result | Date |
|---|---|---|---|
| AC-100 | Every claim in the repository supported by a recorded observation | | |
| AC-101 | No metric stated without its measurement method | | |
| AC-102 | Diagram title updated from PLANNED once deployed; undeployed components still dashed | | |
| AC-103 | Documentation corrected where deployment proved it wrong | | |
| AC-104 | Limitations section written honestly, including what the lab does not demonstrate | | |

**AC-103 is the one most easily skipped and most worth keeping.** Where the design turned out to be wrong, the correction is the most valuable content in the repository.

---

## Sign-off

| Phase | Criteria met | Signed | Date |
|---|---|---|---|
| 1 — Design | | | |
| 2 — Network | | | |
| 3 — Logging | | | |
| 4 — Wazuh | | | |
| 5 — Ingestion | | | |
| 6 — Detection | | | |
| 7 — Shuffle | | | |
| 8 — End to end | | | |
