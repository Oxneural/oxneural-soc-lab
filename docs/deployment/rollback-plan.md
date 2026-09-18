# Rollback Plan — PLANNED

> **No deployment has occurred. Nothing to roll back.**

Rollback is not teardown. **Teardown** removes the lab permanently ([`teardown-plan.md`](teardown-plan.md)). **Rollback** returns a specific phase to its previous known-good state so deployment can be retried.

## Principle

**[DESIGN DECISION]** Every deployment phase is independently reversible, and its rollback is tested when the phase is built — not when it is needed.

A phase whose rollback has never been exercised is an assumption. The first time you find out a rollback does not work should not be while the thing is broken.

## Rollback by phase

### Phase 1 — Network

**Trigger:** wrong CIDR, misconfigured routing, a security group rule that should not exist.

```
1. Confirm no compute exists in the subnet
2. Delete security groups (default group cannot be deleted; that is expected)
3. Delete route table associations, then route tables
4. Detach and delete the internet gateway
5. Delete the subnet
6. Delete the VPC
7. Verify: filter the console by tag Project=oxneural-soc-lab → zero results
```

**Cost exposure:** negligible — these components are free.
**Trap:** a leftover network interface blocks subnet deletion. Find and remove it rather than forcing.

### Phase 2 — Logging

**Trigger:** events not arriving, wrong scope, unexpected volume.

```
1. Stop and delete the CloudTrail trail
2. Delete the VPC Flow Log
3. Decide on the S3 data:
   - Reconfiguring and retrying → keep the bucket, purge the objects
   - Abandoning → empty the bucket, then delete it
4. Verify no further objects arrive after deletion
```

**Cost exposure:** LOW — S3 storage only.
**Trap:** deleting the trail does not delete the objects it already wrote. They keep billing until removed.

### Phase 3 — Wazuh

**Trigger:** failed installation, stack unhealthy, resource exhaustion.

```
1. Capture the installation log and the failure output BEFORE destroying anything
   — the evidence is the point of the rollback
2. Terminate the wazuh-aio instance
3. DELETE ITS EBS VOLUME — verify it is gone
4. Retry from a clean instance
```

**[DESIGN DECISION]** Rebuild rather than repair a failed Wazuh installation. The all-in-one assistant is designed to run on a clean host; repairing a half-installed stack takes longer and leaves uncertainty about what state it is actually in.

**Cost exposure:** MEDIUM — orphaned EBS volumes are the risk. Verify deletion, do not assume it.

**If it fails at the documented sizing:** the host runs at Wazuh's published minimum (4 vCPU / 8 GiB — [`../architecture/decisions.md`](../architecture/decisions.md) §3), so do not assume undersizing. Capture the installer log and diagnose the actual cause before changing the instance type.

### Phase 4 — Agents and ingestion

**Trigger:** events not parsing, agent not enrolling.

```
1. Uninstall the agent from the target host
2. Remove the agent registration on the manager
3. Revert ingestion configuration to the last known-good file
4. Retry
```

**Cost exposure:** none. No rebuild required — this phase is configuration only.

### Phase 5 — Detection content

**Trigger:** rule not firing, or firing on everything.

```
1. Revert the rule file from git
2. Restart the Wazuh manager
3. Confirm the previously working rules still fire (regression check)
```

**[DESIGN DECISION]** Detection content is version-controlled, so rollback is a `git revert` rather than an archaeology exercise. This is the main practical reason detection-as-code is worth the discipline.

**Cost exposure:** none.

### Phase 6 — Shuffle

**Trigger:** containers failing, webhook not received, Opensearch not starting.

```
1. docker compose down
2. Check vm.max_map_count — the most common cause of Opensearch failing to start
3. If unrecoverable: terminate the instance, DELETE ITS EBS VOLUME, rebuild
```

**[VERIFIED FACT]** Shuffle requires `vm.max_map_count=262144` ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)). If it was set with `sysctl -w` and not made persistent, it is lost on reboot and Opensearch will not start — check this before concluding anything is broken.

**Cost exposure:** MEDIUM if the instance is rebuilt — verify volume deletion.

### Phase 7 — Playbooks

**Trigger:** workflow misbehaving, or an approval gate not holding.

```
1. Disable the workflow in Shuffle (do not delete — keep it for diagnosis)
2. Revert the workflow export from git
3. Re-import
4. RE-TEST THE APPROVAL GATE before re-enabling
```

**A workflow that failed to halt at its approval gate is not re-enabled until that specific behaviour has been retested and confirmed.** This is the one rollback with a safety consequence rather than a cost one.

## Emergency rollback

If cost spikes unexpectedly, exposure is suspected, or something is behaving in a way that is not understood:

```
1. STOP ALL COMPUTE IMMEDIATELY        — question afterwards
2. Revoke any access keys that exist
3. Check CloudTrail for what actually happened
4. THEN decide: rollback a phase, or full teardown
```

Acting first and diagnosing second is correct here. The environment is disposable; a runaway bill or an ongoing compromise is not.

## Rollback log

| Date | Phase | Reason | Actions | Verified clean | Cost impact |
|---|---|---|---|---|---|
| | | | | | |

## Sources

- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
- [Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html)
