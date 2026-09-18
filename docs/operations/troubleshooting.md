# Troubleshooting — PLANNED

> **PLANNED. Nothing is deployed. These are anticipated failure modes based on documented requirements, not observed incidents.**

Symptoms marked **[ANTICIPATED]** are predicted from official documentation. As real problems are hit, they are recorded here with what actually happened and what actually fixed it — which is more valuable than any prediction.

## First rule

**Check whether the pipeline is broken before concluding nothing is happening.** A quiet console is equally consistent with a quiet day and a dead agent. Distinguishing the two is the first diagnostic step, always.

---

## Access

### Cannot start an SSM session **[ANTICIPATED]**

SSM is the only access path — if it fails, the instance is unreachable by design. Check in order:

```
1. Is the instance running? (not "pending", not "stopping")
2. Has the SSM agent registered? Managed instance list, not the EC2 console
3. Does the instance role include AmazonSSMManagedInstanceCore?
4. Can the instance reach SSM endpoints outbound on 443?
5. Is the Session Manager plugin installed locally?
6. Does your IAM identity permit ssm:StartSession on this instance?
```

Most common cause: the instance was just started and SSM has not registered yet. Wait, then re-check.

**Do not "fix" this by opening SSH.** If SSM cannot be restored, rebuild the instance — it is disposable.

### Port forwarding connects but the dashboard does not load **[ANTICIPATED]**

```
1. Is the service actually listening on the instance? (check from an SSM shell)
2. Correct port in the forwarding parameters?
3. Local port already in use by something else?
4. Is the service bound to 127.0.0.1 as intended?
```

---

## Wazuh

### Installation assistant fails **[ANTICIPATED]**

```
1. Capture the installer log BEFORE retrying — the evidence is the point
2. Confirm the OS is on the supported list
3. Confirm RAM meets the documented 8 GiB
4. Confirm outbound 443 to packages.wazuh.com
5. Sufficient disk?
```

**[DESIGN DECISION]** Rebuild rather than repair a failed installation. The assistant expects a clean host; repairing a half-installed stack takes longer and leaves uncertainty about its actual state. See [`../deployment/rollback-plan.md`](../deployment/rollback-plan.md).

### Indexer will not start, or the dashboard is very slow **[ANTICIPATED]**

**The host runs at Wazuh's documented minimum (4 vCPU / 8 GiB), so a resource shortfall here is not an undersizing you accepted** — investigate the actual cause rather than assuming the instance is too small. See [`../architecture/decisions.md`](../architecture/decisions.md) §3.

```
1. Check memory pressure and CPU saturation
2. Check disk space — a full disk stops indexing
3. Check index size against retention intent
```

If the documented specification genuinely proves insufficient for this workload, that is a finding worth recording against AC-33 — and worth reporting upstream, because it contradicts Wazuh's published sizing.

### Agent not reporting **[ANTICIPATED]**

The most important failure in the system, because it is silent.

```
1. Is the agent service running on the host?
2. Is the agent enrolled on the manager?
3. Does sg-wazuh permit the agent port from sg-target?
4. Clock skew between agent and manager?
5. Manager service healthy?
```

**This must trigger an alert, not be discovered by chance.** If a silent agent went unnoticed, that is a finding about the monitoring, not just about the agent.

### CloudTrail or flow log events not appearing **[ANTICIPATED]**

```
1. Are objects actually landing in S3? Check the bucket directly.
2. Does role-wazuh have s3:GetObject and s3:ListBucket on that bucket and prefix?
3. Is the AWS module configured for the right bucket, prefix and region?
4. Are events parsing? Check for decoder errors, not just ingestion.
```

Distinguish "not delivered" from "delivered but not parsed" early — they have completely different fixes.

---

## Shuffle

### Opensearch will not start **[ANTICIPATED]**

**Check this first, before anything else.**

**[VERIFIED FACT]** Shuffle requires `vm.max_map_count=262144` ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)).

If it was set with `sysctl -w` and not made persistent in `/etc/sysctl.d/`, **it is lost on reboot** and Opensearch silently fails. This presents as a broken installation and is not one.

```
1. cat /proc/sys/vm/max_map_count   → expect 262144
2. If wrong: set it, make it persistent, restart containers
3. Then check container memory against the documented 4 GiB minimum
4. Then check disk space
```

### Containers unhealthy after stop/start **[ANTICIPATED]**

Most likely the same `vm.max_map_count` persistence issue. Check it before anything else.

### Webhook from Wazuh not arriving **[ANTICIPATED]**

```
1. Is Shuffle actually running and listening?
2. Does sg-shuffle permit the webhook from sg-wazuh?
3. Is the webhook URL correct in the Wazuh integration config?
4. Check the Wazuh integration logs for delivery errors
```

**An alert generated but never delivered is invisible** — the pipeline looks healthy and the analyst sees nothing. This is why webhook delivery is a monitored health check, not an assumption.

### Enrichment failing **[ANTICIPATED]**

```
1. API key valid and present in the environment?
2. Rate limit reached? (free tiers have daily caps)
3. Provider reachable — outbound 443 permitted?
4. CRITICAL: confirm the alert still reached the analyst unenriched
```

**Step 4 is the one that matters.** Enrichment failing is an inconvenience; enrichment failing *silently and suppressing the alert* is a monitoring outage. Tested as AC-65.

---

## Cost

### Unexpected charge **[ANTICIPATED]**

In order of likelihood:

```
1. Instances left running — check every region
2. EBS volumes surviving terminated instances
3. Unattached Elastic IP (should never exist in this design)
4. NAT gateway (should never exist in this design)
5. Forgotten snapshots
6. Flow log volume higher than expected
7. S3 growth without lifecycle rules
8. Resources in a region you did not intend to use
```

**Check every region, not just the one you worked in.**

If the cause is not identified within one session, stop all compute and work it out with the lab off.

---

## Recording real problems

As problems actually occur, replace or supplement the anticipated entries:

| Date | Symptom | Root cause | Fix | Prevention |
|---|---|---|---|---|
| | | | | |

**Record what actually happened, including the wrong turns.** A troubleshooting guide listing only clean diagnoses was not written from experience.

## Sources

- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)
