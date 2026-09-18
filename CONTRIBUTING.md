# Contributing

This is an OxNeural internal project. Contributions follow the [OxNeural Handbook](https://github.com/Oxneural/oxneural-handbook).

## Before anything else

- **Security issue?** Do not open an issue. Follow [`SECURITY.md`](SECURITY.md).
- **Never commit a secret.** See the secrets section of `SECURITY.md`. Run `git diff` and read it before every commit.

## Workflow

```
main
 └─ feature/<short-description>
      └─ pull request → review → merge
```

`main` is protected: no direct pushes, no force-pushes. Branch prefixes: `feature/`, `fix/`, `docs/`, `detection/`, `playbook/`, `chore/`.

Commits follow [Conventional Commits](https://github.com/Oxneural/oxneural-handbook/blob/main/docs/standards/commit-convention.md).

## Contributing detection content

Every detection document must contain all eleven sections defined in [`docs/detection/detection-engineering.md`](docs/detection/detection-engineering.md): objective, data source, logic, expected alert, analyst investigation, enrichment, severity, false positives, response, validation, evidence to capture.

A detection is not complete until:

- [ ] It has a documented test case in [`docs/testing/test-cases.md`](docs/testing/test-cases.md)
- [ ] It has actually fired in a test, and the result is recorded
- [ ] Expected false positives are documented — "none" is almost never true
- [ ] A response action is defined; an alert with no response is noise
- [ ] The logic is original or properly attributed to its open-source origin and licence

**Never** copy detection content from a client environment, a previous employer or a commercial ruleset.

## Contributing playbooks

Any step that isolates a host, disables an identity, blocks an address or deletes anything requires an explicit **approval gate**. Automation prepares; a human decides.

Playbooks must be followable by a second analyst without the author present. If a step says "investigate", it is not a step.

## Honesty rules

These are enforced in review:

- **No metric without a stated measurement method.** No "reduced detection time by X%".
- **No claim of deployment that has not happened.** Documented ≠ deployed ≠ validated.
- **No fake screenshots, fake alerts or fake results.**
- **No real data.** Synthetic only — no client, employer or third-party logs, addresses or findings.
- Planned work is labelled planned.

## Pull requests

Keep them small. Fill in the template. State what you actually ran and what you observed. Flag your own uncertainty — it gets you a better review.

At least one approving review before merge.
