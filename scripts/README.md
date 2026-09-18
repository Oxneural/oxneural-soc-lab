# scripts/

Helper and validation tooling. Python.

**Status: empty.** No scripts written yet.

## Planned purpose

- Generating synthetic activity for detection validation
- Pipeline health checks (agent heartbeat, log delivery, time sync)
- Enrichment helpers
- Lab setup and teardown assistance

## Rules for content in this directory

- **No credentials.** Configuration from environment variables, loaded from a git-ignored `.env`. A committed `.env.example` holds placeholders only.
- **No hard-coded account IDs, ARNs, IP addresses or hostnames.**
- **Simulation scripts target lab systems only.** Any script generating activity includes a comment stating its scope and the authorisation it operates under.
- **No destructive capability.** No script deletes data, terminates resources or modifies production-style configuration without explicit confirmation.
- **No malware, no exploit code.** Simulation uses benign artefacts such as the EICAR test string and reversible technique emulation.
- PEP 8, type hints on public functions, dependencies pinned.

See [`../SECURITY.md`](../SECURITY.md).
