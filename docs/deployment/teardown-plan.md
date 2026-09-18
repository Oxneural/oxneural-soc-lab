# Teardown Plan

**Written before deployment, deliberately.** A teardown procedure written after the fact is written by someone who wants to be finished.

## Why this matters

An abandoned AWS lab is two problems: an indefinite bill, and an unmonitored, unpatched, internet-exposed environment still holding credentials and log data. The second is the more serious one.

## Order of operations

Dependencies mean order matters. Terminate compute before deleting the network it sits in.

### 1. Capture anything worth keeping

- [ ] Export detection rules and decoders
- [ ] Export Shuffle workflows — **strip API keys before saving anywhere**
- [ ] Save incident documentation and test results
- [ ] Sanitise any screenshot: no account IDs, real addresses or keys
- [ ] Confirm nothing exported contains a credential

### 2. Compute

- [ ] Terminate the Wazuh EC2 instance
- [ ] Terminate the Shuffle EC2 instance
- [ ] Terminate all target hosts
- [ ] **Delete associated EBS volumes** — volumes survive instance termination and keep billing
- [ ] Delete EBS snapshots and any custom AMIs

### 3. Network

- [ ] Release Elastic IPs — **an unassociated Elastic IP bills**
- [ ] Delete NAT gateway if one exists — a common source of unexpected cost
- [ ] Delete security groups, subnets, route tables, internet gateway
- [ ] Delete the VPC

### 4. Storage and logs

- [ ] Empty and delete the CloudTrail S3 bucket
- [ ] Delete CloudWatch log groups
- [ ] Confirm no snapshots remain

### 5. Identity

- [ ] Delete lab-specific IAM roles, policies and users
- [ ] **Rotate or delete any access key that existed**
- [ ] Remove the SSH key pair from AWS and from local storage

### 6. Verification — the step people skip

- [ ] Filter the console by tag `Project=oxneural-soc-lab`; confirm zero resources
- [ ] **Check every region**, not only the one you worked in
- [ ] Cost Explorer: confirm charges fall to zero over the following 48 hours
- [ ] Confirm the billing alarm remains in place to catch anything missed

### 7. Local cleanup

- [ ] Delete local `.env` and any credential file
- [ ] Remove the AWS CLI profile for the lab account
- [ ] Confirm no credential ever entered git history

## Partial teardown

Between working sessions, stop rather than destroy: stop EC2 instances, release unneeded Elastic IPs, leave logging and the network in place. Stopped instances do not bill for compute; their EBS volumes still do.

## Emergency teardown

If cost spikes unexpectedly or exposure is suspected:

1. Stop all compute immediately — question later
2. Revoke all access keys
3. Check CloudTrail for what actually happened
4. Then work through the full teardown above

Acting first and diagnosing second is correct here. The environment is disposable; an ongoing compromise or runaway bill is not.

## Record

| Date | Type | Performed by | Verified zero resources | Notes |
|---|---|---|---|---|
| | | | | |

## Related

[`cost-model.md`](cost-model.md) — per-resource cost categories and removal steps · [`rollback-plan.md`](rollback-plan.md) — reversing a phase rather than removing the lab · [`../testing/acceptance-criteria.md`](../testing/acceptance-criteria.md) — AC-90 to AC-96 verify teardown actually worked
