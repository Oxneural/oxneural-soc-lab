# Detection: Suspicious Network Activity

**Status:** Documented. **Not implemented, not validated.**
**ATT&CK:** T1046 Network Service Discovery · T1071 Application Layer Protocol · T1048 Exfiltration Over Alternative Protocol · T1571 Non-Standard Port

## 1. Detection objective

Identify network behaviour inconsistent with the lab's normal profile: scanning, connections to unexpected destinations, unusual ports, and outbound patterns suggesting command-and-control or data movement.

Network telemetry rarely proves anything alone. Its value is corroboration — it answers "what else was happening" during an investigation driven by another signal.

## 2. Data source

| Source | Provides |
|---|---|
| VPC Flow Logs | Accepted and rejected connections, ports, byte counts, direction |
| Wazuh agent | Host-level connection and process context |

Flow logs give the connection, not the content. They tell you a host talked to an address on a port and how much data moved — not what was said.

## 3. Detection logic

**A — Port scanning.** Single source connecting to many distinct destination ports on one host, or one port across many hosts, within a short window. Rejected connections are the strong signal.

**B — Unexpected outbound destination.** Outbound connection from a lab host to an address outside the expected set — package repositories, AWS endpoints, threat intelligence APIs.

**C — Non-standard port usage.** Outbound connections on ports the lab has no reason to use.

**D — Beaconing pattern.** Repeated outbound connections to the same destination at regular intervals with consistent, small byte counts. **Regularity is the signal** — human traffic is irregular; automation is not.

**E — Unusual outbound volume.** Outbound data transfer significantly exceeding the host's baseline.

### Threshold justification

Every one of these depends on a baseline. Thresholds are set after observing normal lab traffic for a defined period, and recorded with the date measured. Without a baseline these rules generate noise and get switched off within a week — which is the outcome principle 4 of the detection standard exists to prevent.

## 4. Expected alert

```
Rule:        Suspicious network activity — <pattern>
Severity:    Medium  (High with corroborating host or identity signal)
Source:      <srcip> (<agent>)
Destination: <dstip>:<port>
Pattern:     <scan / beacon / volume / unexpected destination>
Window:      <start> – <end> UTC
Bytes:       <count>
```

## 5. Analyst investigation

1. **Which host, and what does it legitimately do?** A monitoring agent's traffic looks like beaconing because it is beaconing.
2. **Inbound or outbound?** Outbound from an internal host is generally the more serious direction.
3. **What process owned the connection?** Host telemetry, not flow logs. **This is usually the question that resolves the alert.**
4. **Is the destination known?** Reputation, ownership, whether previously seen in the lab.
5. **Accepted or rejected?** Rejected connections indicate attempts; accepted indicate success.
6. **How much data moved, and in which direction?**
7. **What else happened on this host at the same time?** Authentication, process creation, file changes.

## 6. Enrichment

Destination IP reputation, geolocation, ASN and ownership · whether previously observed in the lab · source host role and baseline · resolved DNS where available · process and user context from the agent.

## 7. Severity considerations

**Raises severity:** outbound to a poor-reputation destination · corroborating authentication or IAM signal on the same host · sustained beaconing · large outbound transfer · connection immediately following a suspicious process · non-standard port carrying encrypted traffic.

**Lowers severity:** known update or package repository · monitoring or backup agent matching its documented pattern · activity coinciding with planned testing · rejected connections from internet background scanning.

## 8. False-positive considerations

Network detections are the noisiest category. Expect all of these:

- **Package updates and OS telemetry** — regular, outbound, to unfamiliar CDN addresses. Looks exactly like beaconing.
- **Monitoring and backup agents** — genuinely beacon by design
- **Cloud service endpoints** across wide, changing IP ranges
- **Internet background scanning** against any public IP — constant and meaningless on its own
- **NTP, DNS and health checks** — small, regular, outbound
- A vulnerability scanner or the lab's own testing

Baselining is not optional here; it is the only thing that makes these rules usable.

## 9. Response

| Action | Approval |
|---|---|
| Investigate and document | None |
| Correlate with host and identity telemetry | None |
| Enrich destination | None — automatic |
| Increase monitoring on the host | None |
| **Block destination address** | **Required** |
| **Modify security group** | **Required** |
| **Isolate host** | **Required** |
| Escalate | Per playbook |

Blocking a destination that turns out to be a package repository or an AWS endpoint breaks the environment. The gate exists for that reason.

## 10. Validation

From an authorised lab host against another lab host: a controlled port scan; a connection to a known-benign external address on a non-standard port; a scripted regular-interval outbound connection to simulate beaconing; a controlled outbound transfer.

Recorded in [`../testing/test-cases.md`](../testing/test-cases.md). **All traffic originates from and targets OxNeural lab systems, or well-known benign public endpoints. No third-party system is ever scanned.**

## 11. Evidence to capture

Flow log records for the full window · source and destination addresses, ports, protocol, byte counts, accept/reject · process and user context from the host · timestamps in UTC · host baseline for comparison · corroborating events from other sources.
