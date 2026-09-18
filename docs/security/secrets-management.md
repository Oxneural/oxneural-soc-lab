# Secrets Management — PLANNED

> **PLANNED. No secret, credential, key or token has been created for this lab.**

## The rule

**No secret is ever committed to this repository.** Not in code, configuration, comments, commit messages, test fixtures, documentation examples, screenshots, workflow exports or infrastructure state.

## What counts as a secret here

| Category | Examples |
|---|---|
| AWS | Access key IDs, secret access keys, session tokens, **account IDs** |
| Wazuh | API credentials, agent enrolment password, dashboard admin password |
| Shuffle | Admin credentials, API keys, webhook URLs, app authentication |
| Threat intelligence | Provider API keys |
| Infrastructure | SSH private keys, TLS private keys, certificates |
| Environmental | Real IP addresses, hostnames, ARNs, instance IDs |

**[DESIGN DECISION]** AWS account IDs and instance IDs are treated as secrets in this repository. They are not credentials, but publishing them tells an attacker exactly what to target and offers nothing in return.

## The strongest control: don't create secrets

**[DESIGN DECISION]** The lab is designed so that most secrets never exist.

| Conventional approach | This design | Secret eliminated |
|---|---|---|
| SSH key pair for access | SSM Session Manager | Private key |
| Access keys on instances | IAM instance roles | Long-lived AWS keys |
| Bastion host credentials | No bastion | Bastion credentials |
| Dashboard exposed with a strong password | Not exposed at all | Password becomes non-critical |

**[VERIFIED FACT]** Session Manager carries no additional charge on EC2 ([AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)).

A secret that does not exist cannot be leaked, rotated incorrectly, or committed by accident. This is worth more than any storage mechanism.

## Secrets that will exist

| Secret | Where it lives | Rotation |
|---|---|---|
| Wazuh dashboard admin password | Generated at install; stored in the operator's password manager | On suspicion; at rebuild |
| Wazuh API credentials | On the Wazuh host; referenced by Shuffle via environment variable | On suspicion; at rebuild |
| Shuffle admin credentials | Created at first login; operator's password manager | On suspicion |
| Threat intelligence API key | `.env` on the Shuffle host, git-ignored | Per provider policy |

**[VERIFIED FACT]** Shuffle has no default username or password: "Shuffle doesn't have a default username and password" — credentials are set at first login ([Shuffle install guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)). **[DESIGN DECISION]** Set them immediately on first start, before any other configuration.

**[DESIGN DECISION]** All installer-generated default credentials are changed before the host is considered deployed. Phase checkpoints in [`../deployment/deployment-plan.md`](../deployment/deployment-plan.md) verify this explicitly rather than assuming it.

## Storage

**[DESIGN DECISION]** Environment variables from a git-ignored `.env`, with a committed `.env.example` holding placeholders only.

**[OPEN QUESTION]** AWS Secrets Manager or SSM Parameter Store (SecureString) would be the better practice. Parameter Store standard parameters are free, which makes it attractive; Secrets Manager charges per secret. Deferred to Phase 6 rather than decided now, because at this scale the `.env` approach is adequate and adds no moving parts. **If the number of secrets grows beyond a handful, move to Parameter Store rather than accumulating `.env` files.**

## `.gitignore` coverage

Already committed and in force:

```
.env  .env.*  !.env.example
*.pem  *.key  *.ppk  *_rsa  *_ed25519  *.p12  *.pfx
credentials  credentials.*  .aws/  aws-credentials*
*.tfstate  *.tfstate.*  *.tfvars  !*.tfvars.example
shuffle-export-*.json
```

**[DESIGN DECISION]** Terraform state is git-ignored because it can contain secrets in plaintext, and `*.tfvars` with it. Only `.example` variants are ever committed.

**[DESIGN DECISION]** Shuffle workflow exports are git-ignored by default. They can embed API keys and webhook URLs. An export is committed only after being opened and inspected — never on the assumption that it is clean.

## Repository controls

Enabled and verified on this repository:

- **Secret scanning** — detects committed credentials
- **Push protection** — blocks the push before the secret reaches GitHub
- `.gitignore` as above

Push protection is the one that actually matters: it prevents the exposure rather than reporting it afterwards.

## Documentation placeholders

Use these, never a real value:

| Instead of | Write |
|---|---|
| Real AWS account ID | `YOUR_AWS_ACCOUNT_ID` |
| Real access key | `AKIAIOSFODNN7EXAMPLE` (AWS's own documentation example) |
| Real public IP | `198.51.100.10` (TEST-NET-2) or `203.0.113.10` (TEST-NET-3) |
| Real private IP | `10.0.1.10` |
| Real hostname | `example.com` |
| Real API key | `YOUR_API_KEY_HERE` |

## If a secret is committed

```
1. ROTATE OR REVOKE THE CREDENTIAL IMMEDIATELY
   — this is the only step that actually fixes anything
2. Notify the founder
3. Then deal with git history
```

**Removing the commit does not undo the exposure.** Forks, clones, caches and GitHub's event stream persist. Treat the credential as compromised from the moment it was pushed, not from when it was noticed.

For an AWS key specifically: revoke it, then check CloudTrail for what it did between exposure and revocation. That second step is the one people skip.

## Pre-commit check

Before every commit:

- [ ] `git diff` read in full, not skimmed
- [ ] No credential, key, token or password
- [ ] No AWS account ID, instance ID or ARN
- [ ] No real public IP or hostname
- [ ] No `.env`, `.tfstate` or `.tfvars` staged
- [ ] Workflow exports opened and inspected, not assumed clean
- [ ] Screenshots sanitised

## Sources

- [AWS Systems Manager pricing](https://aws.amazon.com/systems-manager/pricing/)
- [Shuffle installation guide](https://github.com/Shuffle/Shuffle/blob/main/.github/install-guide.md)
