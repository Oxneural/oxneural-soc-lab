# Security Policy

**Classification:** OxNeural Internal Project. This is a lab, not a production or client environment.

## Reporting a vulnerability

Report privately — **never as a public issue**. Use **Security → Report a vulnerability** on this repository, or contact [Muzaffar Moosa Shaikh](https://www.linkedin.com/in/cyber-shaikh/).

Acknowledgement target: 5 business days. This is a small team, stated honestly; we will tell you if something will take longer.

## Secrets — the absolute rule

**No secret is ever committed to this repository.** Not in code, configuration, comments, commit messages, test fixtures, documentation examples, screenshots or Terraform state.

Specifically never committed:

- AWS access key IDs and secret access keys
- AWS account IDs
- Session tokens, API keys, webhook URLs
- SSH private keys, certificates, `.pem` / `.ppk` files
- Wazuh API credentials, agent enrolment passwords
- Shuffle API keys, workflow exports containing credentials
- Threat intelligence API keys
- Real IP addresses, hostnames or ARNs from any environment

### How secrets are handled instead

- Environment variables, loaded from a local `.env` that is git-ignored
- A committed `.env.example` with placeholder values only
- AWS credentials via IAM roles and short-lived credentials, not long-lived keys
- GitHub encrypted secrets for anything CI needs
- Documentation uses placeholders: `YOUR_AWS_ACCOUNT_ID`, `10.0.0.0/16`, `198.51.100.0/24` (TEST-NET-2), `example.com`

### If a secret is committed

1. **Rotate or revoke the credential immediately.** This is the only step that fixes anything.
2. Notify the founder.
3. Then deal with history.

Removing the commit does not undo the exposure — forks, clones, caches and GitHub's own event stream persist. Treat it as compromised from the moment it was pushed.

Secret scanning and push protection are enabled on this repository.

## AWS security baseline

| Control | Requirement |
|---|---|
| Account isolation | Dedicated lab account, separate from anything else |
| Root account | MFA enabled; not used for daily work; no access keys |
| IAM users | MFA required; least privilege; no wildcard admin for routine work |
| Access keys | Avoided in favour of roles; where unavoidable, rotated and scoped |
| CloudTrail | Enabled in all regions, including IAM activity |
| Security groups | No `0.0.0.0/0` on SSH, RDP, or any management port — source-restricted to known addresses only |
| Encryption | At rest for S3 and EBS; TLS in transit |
| Public exposure | No public S3 buckets; no unnecessary public IPs |
| Tagging | Every resource tagged for cost tracking and teardown |

## Isolation

The lab runs in its own VPC in its own account. Simulated attack traffic stays inside it. There is no connectivity to any production, client, personal or third-party network.

## Authorized testing only

All testing in this lab is against systems owned and controlled by OxNeural for that purpose.

**No testing is ever conducted against any third-party system**, whatever the provocation, curiosity or apparent harmlessness. Unauthorised testing is a criminal offence in most jurisdictions, including India under the Information Technology Act.

Attack simulation is limited to controlled, non-destructive techniques. This repository contains **no malware, no ransomware, no exploit code and no destructive payloads**, and none will be added. Malware behaviour is simulated using benign, industry-standard artefacts such as the EICAR test string and documented, reversible technique emulation.

## Logging and evidence

- Logs and captured evidence stay in the lab; nothing is committed to this repository
- Retention is defined in the deployment plan
- Time is synchronised across sources — unsynchronised clocks make timelines unreliable
- Screenshots, if ever added, are sanitised: no account IDs, no real addresses, no keys

## Response actions

Automated workflows in this lab use **approval gates** before any potentially disruptive action — isolating a host, disabling an identity, blocking an address. Automation gathers context and prepares the action; a human decides. This is the same discipline the lab exists to practise.

## Cost and safe teardown

Cost is a security property here: an abandoned lab is an unmonitored, unpatched, internet-exposed environment as well as a bill. Billing alerts are configured before deployment, and the teardown procedure is written before the first resource is created.

See [`docs/deployment/cost-control.md`](docs/deployment/cost-control.md) and [`docs/deployment/teardown-plan.md`](docs/deployment/teardown-plan.md).
