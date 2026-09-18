# Shuffle SOAR Design — PLANNED

> **PLANNED. Shuffle is not installed. No workflow exists.**

## Role

Shuffle sits between detection and decision. It receives alerts from Wazuh, attaches context automatically, and presents an enriched alert to the analyst.

**It does not decide, and it does not act on anything disruptive.** That separation is the design, not a limitation.

## Deployment

**[VERIFIED FACT]** Shuffle requires Docker and git: "Make sure you have Docker and git (for downloading) installed." Minimum RAM: "a minimum of **4Gb of RAM** available. More RAM = better." CPU and disk are not stated ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)).

**[VERIFIED FACT]** Requires `sudo sysctl -w vm.max_map_count=262144` for Opensearch.

**[DESIGN DECISION]** Make `vm.max_map_count` persistent in `/etc/sysctl.d/` rather than setting it with `sysctl -w` alone. Set only at runtime, it is lost on reboot and Opensearch silently fails to start — a failure that looks like a broken installation and is not.

**[DESIGN DECISION]** Separate EC2 instance (`shuffle`, 2 vCPU / 4 GiB), not co-located with Wazuh. Placing the SOAR platform and the SIEM in one failure and compromise domain undercuts part of what this lab exists to demonstrate. See [`final-aws-architecture.md`](final-aws-architecture.md) §4 for the cost trade-off and Option B.

## Ports — none exposed

**[VERIFIED FACT]** Shuffle listens on 3001 (HTTP), 3443 (HTTPS), 5001 (backend REST API) and 9200 (Opensearch).

**[DESIGN DECISION]** None is reachable from the internet. `sg-shuffle` has no inbound rules. The UI is reached by SSM port forwarding to `localhost`.

**Port 9200 in particular must never be reachable.** An exposed Opensearch index is a data-disclosure incident and a well-documented way self-hosted stacks leak.

**[VERIFIED FACT]** "Shuffle doesn't have a default username and password" — credentials are created at first login. **[DESIGN DECISION]** Set them immediately, before any other configuration.

## Integration with Wazuh

```
Wazuh manager
   │  alert generated
   ▼
Webhook (HTTP POST, intra-VPC)
   ▼
Shuffle workflow trigger
   │
   ├──▶ Wazuh API query for additional alert context  (sg-shuffle → sg-wazuh)
   └──▶ Threat intelligence lookup                    (HTTPS egress)
```

**[OPEN QUESTION]** Webhook versus Shuffle polling the Wazuh API. Webhook is lower latency and the documented pattern; polling is more resilient to a brief Shuffle outage. Decide at Phase 6. Either way the traffic is intra-VPC and governed by security group rules referencing source groups, never CIDRs.

**[DESIGN DECISION]** Webhook delivery failure must be **detected**, not silent. An alert generated but never delivered to enrichment is invisible — the pipeline looks healthy and the analyst sees nothing. Health check defined in [`../testing/validation-plan.md`](../testing/validation-plan.md).

## Workflow pattern

Every workflow follows the same shape:

```
TRIGGER          Wazuh alert
   ▼
EXTRACT          indicators — IP, user, host, process, event name
   ▼
FILTER           skip RFC1918 / known-good before any external lookup
   ▼
ENRICH           IP reputation · geo/ASN · lab history · IAM or asset context
   ▼             (automatic — no approval needed, nothing disruptive)
RISK ASSESS      apply documented severity criteria
   ▼
PRESENT          enriched alert to the analyst
   ▼
════ HUMAN APPROVAL GATE ════
   ▼
ANALYST DECISION  false positive · monitor · respond
   ▼
RESPOND          only approved actions
   ▼
DOCUMENT         always, on every path including false positives
```

## Approval gates

**[DESIGN DECISION]** No workflow in this lab executes a disruptive action automatically.

| Action | Automatic | Requires approval |
|---|---|---|
| Enrich with threat intelligence | ✓ | |
| Query Wazuh for context | ✓ | |
| Attach enrichment to the alert | ✓ | |
| Notify the analyst | ✓ | |
| Increase monitoring | ✓ | |
| Block an IP address | | **✓** |
| Disable an account or access key | | **✓** |
| Isolate a host | | **✓** |
| Modify a security group | | **✓** |
| Terminate or rebuild a host | | **✓** |

**Why this is not over-caution:** a reputation hit on a NAT or CGNAT address represents an entire ISP's customer base. Automatic blocking on that signal takes a false positive and turns it into an outage. Automation gathers context and prepares the action; a human decides.

**[DESIGN DECISION]** `role-shuffle` has **no AWS API permissions at all** in the initial build. Granting a SOAR platform standing permission to disable identities or change security groups creates exactly the risk the gates exist to prevent. If a response action later genuinely needs an AWS call, it gets its own narrowly scoped role, reviewed separately. See [`../security/iam-design.md`](../security/iam-design.md).

## Planned workflows

| Workflow | Trigger | Enrichment | Approved actions |
|---|---|---|---|
| **Brute force** | Repeated auth failures | IP reputation, geo/ASN, account exists?, privilege level, **any success after failures** | Block IP · disable account · force reset · revoke sessions |
| **Unauthorized IAM** | CloudTrail IAM event | Policy document, actor baseline, MFA used?, full session history | Revoke key · detach policy · disable identity · revoke sessions |
| **Suspicious IP** | Any alert with external IP | Reputation, geo/ASN, lab history | Block destination · modify security group |
| **Malware behaviour** | FIM / process telemetry | Hash reputation, process reputation, concurrent network activity | Terminate process · isolate host · remove persistence · rebuild |

Detailed playbooks: [`../incident-response/`](../incident-response/)

## Credentials

- Shuffle admin credentials → operator's password manager
- Wazuh API credentials → environment variable on the Shuffle host
- Threat intelligence API key → git-ignored `.env`

**[DESIGN DECISION]** Workflow exports are git-ignored by default and committed only after being opened and inspected. Exports can embed API keys and webhook URLs; assuming an export is clean is how keys reach public repositories. See [`../security/secrets-management.md`](../security/secrets-management.md).

## Operational notes

- Stop the instance when not in use
- Confirm containers come up healthy after a stop/start cycle — `vm.max_map_count` persistence is verified here
- **Enrichment failure must never suppress an alert.** If the threat intelligence API is unavailable, the alert still reaches the analyst, unenriched and flagged as such. An enrichment dependency that silently swallows alerts is worse than no enrichment. Tested as AC-09.

## Backup

Workflow definitions are exported to git after credential stripping. That is the backup. Execution history is not preserved — it is lab activity, reproducible from test cases.

## Sources

- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
