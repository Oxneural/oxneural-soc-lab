# Prerequisites

**Complete every item before creating the first AWS resource.**

## Account and access

- [ ] Dedicated AWS lab account, isolated from all other use
- [ ] Root account MFA enabled; recovery codes stored offline
- [ ] Root account not used for daily work; no root access keys
- [ ] Administrative IAM identity created, with MFA
- [ ] Billing access confirmed

## Cost controls — before any chargeable resource

- [ ] AWS Budgets budget created with a hard monthly figure
- [ ] CloudWatch billing alarm configured with email notification
- [ ] Cost Explorer enabled
- [ ] Tagging scheme agreed (see [`../architecture/aws-architecture.md`](../architecture/aws-architecture.md))

**This section is not optional and does not come later.** A lab without billing alerts is a bill waiting to be discovered.

## Documentation — before deployment

- [ ] Architecture reviewed and approved
- [ ] **Teardown plan written and reviewed** — [`teardown-plan.md`](teardown-plan.md)
- [ ] Detection scenarios documented
- [ ] Test cases documented
- [ ] Response playbooks documented with approval gates

Writing teardown before build is deliberate. Written afterwards, it is written by someone who wants to be finished.

## Local environment

- [ ] AWS CLI installed and configured against the lab account only
- [ ] SSH key pair generated; private key stored outside any repository
- [ ] `.env` created from `.env.example`; confirmed git-ignored
- [ ] Git configured; `git status` confirms no credential file is tracked

## Network

- [ ] Static or known source IP identified for management access restriction
- [ ] Confirmed: no security group will use `0.0.0.0/0` on a management port

## Knowledge

- [ ] Wazuh architecture understood — manager, indexer, dashboard, agent
- [ ] Shuffle workflow model understood — triggers, apps, conditions
- [ ] AWS IAM policy evaluation understood
- [ ] CloudTrail event structure understood

## Authorisation and ethics

- [ ] Confirmed: all activity is against OxNeural-owned lab resources only
- [ ] Confirmed: no third-party system will be tested
- [ ] Confirmed: simulation is non-destructive; no real malware or exploit code
- [ ] Confirmed: AWS acceptable use policy understood

## Sign-off

Deployment does not begin until every box above is ticked and the architecture is approved.

| Item | Confirmed by | Date |
|---|---|---|
| Prerequisites complete | | |
| Architecture approved | | |
| Teardown plan approved | | |
