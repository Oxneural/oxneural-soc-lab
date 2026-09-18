# detections/

Detection content — Wazuh rules and decoders.

**Status: empty.** Nothing has been implemented. The design for each detection lives in [`../docs/detection/`](../docs/detection/); implemented rules will land here once written and validated.

## Planned layout

```
detections/
├── rules/          custom Wazuh rules (XML)
├── decoders/       custom decoders (XML)
└── mappings/       ATT&CK technique mappings
```

## Rules for content in this directory

- **Original or properly attributed.** Written here, or used under an open-source licence with attribution. Never copied from a client environment, a previous employer or a commercial ruleset.
- **Documented before merged.** Every rule has a corresponding document in `../docs/detection/` containing all eleven required sections.
- **Validated before merged.** Every rule has a test case in [`../docs/testing/test-cases.md`](../docs/testing/test-cases.md) and has actually fired in a test.
- **No real data.** No client, employer or third-party log samples, addresses or hostnames — in rules, comments or test fixtures.
- **Thresholds justified.** A number with no stated reason gets defended forever because nobody remembers it was a guess.

See [`../docs/detection/detection-engineering.md`](../docs/detection/detection-engineering.md).
