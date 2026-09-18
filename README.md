# OxNeural Cloud SOC Lab

**Classification:** OxNeural Internal Project
**Status:** Active Development — documentation and design phase
**Deployment state:** No AWS resources have been deployed. Nothing in this repository is running.

> **This is an OxNeural internal security engineering and learning project. It is not a client deployment.**

---

## 1. Project Overview

A cloud-based security operations lab used to design, build and validate detection and response capability end to end: log collection from AWS, detection in Wazuh, automated enrichment and response orchestration in Shuffle, and documented analyst investigation.

The purpose is to develop and evidence practical SOC engineering capability — detection logic, triage procedure, response playbooks and the documentation that makes them operable by someone else.

Everything here is built against synthetic activity in an isolated lab account. No client data, no client systems, no production environment.

## 2. Objectives

- Build a working log pipeline from AWS services into a SIEM
- Write original detection content for a defined set of scenarios, with documented logic and expected false positives
- Orchestrate enrichment and response in a SOAR platform, with human approval gates on anything disruptive
- Produce incident response playbooks that a second analyst could follow without the author present
- Validate every detection against a documented test case before calling it done
- Operate the whole thing within a controlled, tagged, tear-downable AWS footprint with billing alerts

Explicit non-objective: this lab does not demonstrate production scale, high availability, or multi-tenant operations, and will not be presented as doing so.

## 3. Architecture

```
AWS account (isolated lab)
        │
        ├── AWS IAM, EC2, VPC, S3 — lab resources
        │
        ▼
  Security events and logs
  (CloudTrail, VPC Flow Logs, OS/auth logs via Wazuh agent)
        │
        ▼
      Wazuh  ──────────────  log collection, normalisation, retention
        │
        ▼
  Detection and alerting  ──  rules, decoders, severity
        │
        ▼
    Shuffle SOAR  ─────────  orchestration
        │
        ├──▶ Threat intelligence / enrichment (IP reputation, context lookup)
        │
        ├──▶ [APPROVAL GATE] ──▶ Response action
        │
        ▼
  SOC analyst investigation
        │
        ▼
      Documentation and evidence
```

Full detail: [`docs/architecture/architecture-overview.md`](docs/architecture/architecture-overview.md) · [`docs/architecture/aws-architecture.md`](docs/architecture/aws-architecture.md) · [`docs/architecture/data-flow.md`](docs/architecture/data-flow.md)

## 4. Technology Stack

| Layer | Technology | Role |
|---|---|---|
| Cloud | AWS | Lab environment |
| Identity | AWS IAM | Least-privilege access, and a detection data source |
| Audit logging | AWS CloudTrail | API and control-plane activity |
| Network logging | VPC Flow Logs | Network telemetry |
| SIEM | Wazuh | Collection, normalisation, detection, alerting |
| Endpoint telemetry | Wazuh agent | Host and authentication logs |
| SOAR | Shuffle | Enrichment and response orchestration |
| Threat intelligence | Public IP reputation sources | Alert enrichment |
| Scripting | Python | Validation and helper tooling |
| Documentation | Markdown | This repository |

Technologies are listed because they are in the design. A component is only described as deployed once it actually is.

## 5. SOC Capabilities Demonstrated

- Log collection and normalisation across cloud and host sources
- Detection engineering: rule authoring, tuning, false-positive analysis
- Alert triage with recorded dispositions
- Incident investigation and timeline construction
- Enrichment with threat intelligence
- Response orchestration with approval gates
- Evidence handling and incident documentation
- Least-privilege IAM design and monitoring of identity activity

## 6. Detection Scenarios

| # | Scenario | Documentation |
|---|---|---|
| 1 | Brute-force authentication activity | [`docs/detection/brute-force.md`](docs/detection/brute-force.md) |
| 2 | Unauthorized IAM activity | [`docs/detection/unauthorized-iam-activity.md`](docs/detection/unauthorized-iam-activity.md) |
| 3 | Suspicious network activity | [`docs/detection/suspicious-network-activity.md`](docs/detection/suspicious-network-activity.md) |
| 4 | Malware behaviour — controlled simulation | [`docs/detection/malware-simulation.md`](docs/detection/malware-simulation.md) |
| 5 | Suspicious IP reputation | [`docs/threat-intelligence/ip-reputation.md`](docs/threat-intelligence/ip-reputation.md) |

Method and standards: [`docs/detection/detection-engineering.md`](docs/detection/detection-engineering.md)

## 7. Incident Response Scenarios

| Playbook | Documentation |
|---|---|
| Process and severity model | [`docs/incident-response/incident-response-process.md`](docs/incident-response/incident-response-process.md) |
| Brute force | [`docs/incident-response/brute-force-playbook.md`](docs/incident-response/brute-force-playbook.md) |
| Unauthorized IAM activity | [`docs/incident-response/unauthorized-iam-playbook.md`](docs/incident-response/unauthorized-iam-playbook.md) |
| Malware — isolated lab | [`docs/incident-response/malware-lab-playbook.md`](docs/incident-response/malware-lab-playbook.md) |

## 8. Threat Intelligence Workflow

Alert fires → indicator extracted → reputation and context lookup → risk assessed against lab baseline → enrichment attached to the alert → analyst decides.

Intelligence **enriches** a decision; it does not make one. A single reputation hit is not a verdict — shared infrastructure, stale feeds and NAT egress all produce misleading hits. See [`docs/threat-intelligence/ip-reputation.md`](docs/threat-intelligence/ip-reputation.md).

## 9. IAM and Security Considerations

- Dedicated lab AWS account, isolated from anything else
- MFA on the root account; root not used for daily work
- Least-privilege IAM roles; no long-lived access keys where a role will do
- Security groups restricted to known source addresses — never `0.0.0.0/0` on management ports
- CloudTrail enabled and monitored, including on IAM itself
- No credentials, keys or account identifiers in this repository, ever

Full policy: [`SECURITY.md`](SECURITY.md)

## 10. Deployment Approach

Documentation and architecture first, deployment second. Nothing is deployed until the design has been reviewed and the teardown procedure exists.

[`docs/deployment/prerequisites.md`](docs/deployment/prerequisites.md) · [`docs/deployment/deployment-plan.md`](docs/deployment/deployment-plan.md)

## 11. Testing Approach

Every detection is validated against a documented test case before it is considered complete. A rule that has never fired in a test is an untested assumption.

[`docs/testing/validation-plan.md`](docs/testing/validation-plan.md) · [`docs/testing/test-cases.md`](docs/testing/test-cases.md)

## 12. Teardown and Cost Control

An abandoned AWS lab bills indefinitely. The teardown procedure is written before the first resource is created, and billing alerts are configured before anything is deployed.

[`docs/deployment/teardown-plan.md`](docs/deployment/teardown-plan.md) · [`docs/deployment/cost-control.md`](docs/deployment/cost-control.md)

## 13. Security Considerations

All activity in this lab is against systems owned and controlled by OxNeural for this purpose. No testing is conducted against any third-party system. Attack simulation is limited to controlled, non-destructive techniques within the isolated lab. See [`SECURITY.md`](SECURITY.md).

## 14. Project Status

| Area | Status |
|---|---|
| Architecture and design documentation | Active Development |
| AWS environment | **Not deployed** |
| Wazuh deployment | **Not deployed** |
| Shuffle deployment | **Not deployed** |
| Detection content | Documented; not yet implemented or validated |
| SOAR playbooks | Documented; not yet implemented |
| Test results | **None — nothing has been tested yet** |
| Screenshots | **None — nothing to screenshot yet** |

No metrics, results or evidence are presented because none exist. They will be added only when measured, with method and date.

## 15. Future Improvements

- Container and Kubernetes telemetry
- Additional cloud log sources (GuardDuty, Config)
- MITRE ATT&CK coverage mapping with honest gap reporting
- Detection-as-code with CI validation
- Automated lab provisioning and teardown

Each remains a **Future / Planned Capability** until built.

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Standards: [OxNeural Handbook](https://github.com/Oxneural/oxneural-handbook).

## Licence

[MIT](LICENSE)

---

<sub>OxNeural — Knowledge • Action • Service · Ilm • Amal • Khidmat</sub>
