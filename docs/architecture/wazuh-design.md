# Wazuh Design — PLANNED

> **PLANNED. Wazuh is not installed. No component exists.**

## The decision: single-node, all-in-one

**[DESIGN DECISION]** **Option A — single-node all-in-one.** Manager, indexer and dashboard on one EC2 instance.

### Why

**[VERIFIED FACT]** Wazuh publishes single-host requirements for an all-in-one deployment:

| Agents | CPU | RAM | Storage (90 days) |
|---|---|---|---|
| **1–25** | 4 vCPU | 8 GiB | 50 GB |
| 25–50 | 8 vCPU | 8 GiB | 100 GB |
| 50–100 | 8 vCPU | 8 GiB | 200 GB |

The quickstart installation assistant "deploys all three central components (server, indexer, dashboard) on a single host, suitable for monitoring approximately 100 endpoints" ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)).

This lab will run roughly three agents. A multi-node deployment would cost three or more times as much, add cluster coordination and failure modes, and demonstrate nothing this lab is about. **Wazuh's own documentation places a single host at up to ~100 endpoints — thirty times what this lab needs.**

A multi-node cluster is the right answer when you must demonstrate high availability or handle production log volume. Neither applies. Choosing it here would be selecting enterprise-grade over appropriate, which the brief explicitly warns against.

**[OPEN QUESTION]** If the lab later needs to demonstrate indexer clustering or manager failover, that is a separate, scoped exercise — not a reason to over-build now.

## Components

| Component | Role |
|---|---|
| **Wazuh server (manager)** | Receives agent data, decodes, normalises, evaluates rules, generates alerts |
| **Wazuh indexer** | Stores and indexes alerts and events; search backend |
| **Wazuh dashboard** | Analyst interface — triage, investigation, visualisation |
| **Wazuh agent** | On each monitored host: auth logs, process telemetry, file integrity |

## Sizing

**[DESIGN DECISION]** `wazuh-aio` proposed at 2 vCPU / 8 GiB — RAM requirement met in full, vCPU below the documented figure.

The documented 4 vCPU is specified for **25 agents with 90 days of retention**. This lab runs ~3 agents with a far shorter window — roughly two orders of magnitude less indexing work. RAM, which is what causes hard failures in the indexer, is met exactly.

**This is a recorded deviation from documented minimums, not a claim that Wazuh runs fine at half spec.** Acceptance criterion AC-03 tests it explicitly. If indexing lags or the dashboard is unusable, the remedy is one instance size up — not tuning around a resource shortfall.

**[VERIFIED FACT]** Storage: 50 GB for the 1–25 agent / 90-day profile. **[DESIGN DECISION]** Retention shorter than 90 days, so 50 GB gp3 is ample headroom.

## Installation

**[VERIFIED FACT]** Quickstart method ([Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)):

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

**[VERIFIED FACT]** Supported OS includes Amazon Linux 2023, Ubuntu 22.04/24.04 and RHEL 8/9, on x86_64 or ARM64.

**[OPEN QUESTION]** Base AMI not fixed. Amazon Linux 2023 ships the SSM agent preinstalled, which matters because SSM is the only access path — an instance without a working SSM agent is unreachable by design. Ubuntu LTS is more commonly documented in Wazuh community material. **Decide before Phase 3; verify the installer version current at deployment rather than assuming 4.14.**

## Data sources

| Source | Method | Scenario |
|---|---|---|
| Host auth logs | Agent | Brute force |
| Process telemetry | Agent | Malware behaviour |
| File integrity monitoring | Agent | Malware behaviour, persistence |
| CloudTrail | AWS module, S3 | Unauthorized IAM |
| VPC Flow Logs | AWS module, S3 | Suspicious network |

**[OPEN QUESTION]** AWS module ingestion via S3 polling or SQS notifications. Polling is the default assumption — simpler, no extra resource, no extra IAM permission. Confirm at deployment.

## Access

**No inbound port is open to the dashboard.** Reached by SSM port forwarding:

```bash
aws ssm start-session --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["<dashboard-port>"],"localPortNumber":["<local>"]}'
```

Browser connects to `localhost`. See [`network-architecture.md`](network-architecture.md).

**[DESIGN DECISION]** Installer-generated credentials are changed immediately, before the host is considered deployed. Verified at the Phase 3 checkpoint rather than assumed.

## Alert flow

```
Event ──▶ agent / AWS module ──▶ decoder ──▶ normalised fields
      ──▶ rule evaluation ──▶ alert (id, rule, severity, raw event)
      ──▶ indexer (searchable)
      ──▶ webhook ──▶ Shuffle SOAR
```

## Analyst workflow

1. Alert appears on the dashboard, already enriched by Shuffle
2. Triage — validate, scope, classify
3. Investigate — timeline from indexed events
4. Disposition recorded: True Positive · False Positive · Benign True Positive · Duplicate · Insufficient Data
5. Response, through the approval gate where disruptive
6. Documentation, and tuning feedback to the rule that fired

See [`../detection/detection-engineering.md`](../detection/detection-engineering.md).

## Operational notes

- **Stop the instance when not in use** — the largest cost lever in the lab
- **[VERIFIED FACT]** EBS is "charged by the amount of GB you provision per month until you release the storage" — stopping the instance does not stop volume charges ([Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/))
- Confirm the stack comes up healthy after a stop/start cycle, not only after first install — this is tested in acceptance criteria
- Index growth monitored; retention enforced

## Backup

**[DESIGN DECISION]** Configuration is the asset; event data is not.

| Asset | Backup |
|---|---|
| Detection rules and decoders | **Git** — the real backup |
| Wazuh configuration | Exported to git, credentials stripped |
| Alert and event data | **Not backed up.** Reproducible by re-running test cases. |
| Incident documentation | Git |

An EBS snapshot before a risky change is reasonable; **snapshots are deleted afterwards** because they bill for the storage they consume. Routine snapshotting of a lab is paying to keep data that can be regenerated.

## Sources

- [Wazuh quickstart — requirements and installation](https://documentation.wazuh.com/current/quickstart.html)
- [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)
