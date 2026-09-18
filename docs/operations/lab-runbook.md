# Lab Runbook — PLANNED

> **PLANNED. The lab does not exist. This procedure applies once it is deployed.**

The day-to-day operating procedure. Follow it every session, including the short ones.

## Lifecycle

```
BUILD ──▶ TEST ──▶ OPERATE ──▶ STOP ──▶ (repeat) ──▶ TEARDOWN
                                 ▲                      │
                                 └──────────────────────┘
```

**STOP is the normal end state between sessions**, not teardown. Teardown is for when the lab is finished with for an extended period.

## Start of session

```
□ 1. Check the cost dashboard FIRST — before starting anything
      Anything unexpected since last session? Investigate before adding to it.
□ 2. Start instances:  wazuh-aio → shuffle → target-01
□ 3. Wait for SSM to register all three (they are unreachable until it does)
□ 4. Apply pending patches via SSM Patch Manager
□ 5. Verify Wazuh stack healthy
□ 6. Verify Shuffle containers healthy
      — if Opensearch failed, check vm.max_map_count persistence first
□ 7. Verify all agents reporting — a silent agent is the failure that hides everything else
□ 8. Confirm CloudTrail and flow logs still delivering
□ 9. Open dashboards via SSM port forwarding
```

**[DESIGN DECISION]** The cost check comes first, not last. Checking at the end of a session means discovering a problem after another session's worth of charges.

## Accessing dashboards

```bash
aws ssm start-session --target <instance-id> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["<port>"],"localPortNumber":["<local>"]}'
```

Then browse to `localhost:<local>`. **Never open an inbound port instead**, not even briefly — see [`../security/network-security.md`](../security/network-security.md).

## During the session

| Activity | Reference |
|---|---|
| Writing a detection | [`../detection/detection-engineering.md`](../detection/detection-engineering.md) |
| Validating a detection | [`../testing/test-cases.md`](../testing/test-cases.md) |
| Triaging an alert | [`../incident-response/incident-response-process.md`](../incident-response/incident-response-process.md) |
| Running a simulation | [`../detection/malware-simulation.md`](../detection/malware-simulation.md) |

### Before any simulation

```
□ Scheduled and documented in advance, so the alerts it generates are attributable
□ Target is a disposable lab host
□ Technique is non-destructive and reversible
□ Nothing is directed at any system outside the lab
```

## End of session

```
□ 1. Record what was done, and results, with dates
□ 2. Commit detection rules and workflow exports to git
      — inspect workflow exports for embedded credentials before committing
□ 3. STOP ALL INSTANCES  ← the single largest cost lever
□ 4. Confirm all three show "stopped", not "stopping"
□ 5. Confirm no Elastic IP was allocated during the session
□ 6. Check the cost dashboard again
```

**[VERIFIED FACT]** Stopping an instance ends compute charges but not EBS charges: volumes are "charged by the amount of GB you provision per month until you release the storage" ([Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)).

**Step 4 matters.** An instance that failed to stop bills all night. "Stopping" is not "stopped".

## Weekly

```
□ Cost dashboard reviewed against the budget
□ Any resource outside the Project tag investigated
□ Instances confirmed stopped
□ No unattached EBS volume, snapshot or Elastic IP
□ Patch status reviewed
```

## Monthly

```
□ Access review — IAM identities, permissions, MFA
□ Confirm zero access keys exist (expected answer: zero)
□ Teardown plan re-read and confirmed still accurate
□ TeardownAfter tags reviewed
□ Detection tuning log reviewed
```

## Emergency stop

If cost spikes, exposure is suspected, or behaviour is not understood:

```
1. STOP ALL COMPUTE IMMEDIATELY — question afterwards
2. Revoke any access keys that exist
3. Check CloudTrail for what actually happened
4. Then decide: rollback a phase, or full teardown
```

The environment is disposable. A runaway bill or an ongoing compromise is not.

## Never, in any session

- Open an inbound security group rule "temporarily"
- Use the root account for routine work
- Create a long-lived access key
- Commit a credential, account ID or real IP address
- Run simulation against any system outside the lab
- Leave instances running overnight
- Allocate an Elastic IP
- Record a result that was not observed

## Sources

- [Amazon EBS pricing](https://aws.amazon.com/ebs/pricing/)
- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
