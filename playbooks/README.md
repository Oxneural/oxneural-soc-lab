# playbooks/

Shuffle SOAR workflow exports.

**Status: empty.** No workflow has been built. The design for each playbook lives in [`../docs/incident-response/`](../docs/incident-response/).

## Planned layout

```
playbooks/
├── brute-force/
├── unauthorized-iam/
└── suspicious-ip/
```

## Rules for content in this directory

- **Strip every credential before export.** Shuffle workflow exports can contain API keys, webhook URLs and authentication tokens. **Open the JSON and check before committing — never assume the export is clean.**
- **Approval gates are mandatory** on any action that blocks an address, disables an identity, isolates or rebuilds a host, or reverts a configuration. Automation gathers context and prepares the action; a human decides.
- **No destructive action executes automatically.** This is a design decision, not a limitation.
- **Documented before committed.** Each workflow corresponds to a playbook document.
- **Validated before committed** — including a test that confirms the approval gate actually halts the workflow.

See [`../docs/incident-response/incident-response-process.md`](../docs/incident-response/incident-response-process.md).
