# CDD spec changelog

Versioned independently of any implementation. A record's
`lineage.spec_version` field states which version it conforms to.

## 0.1.0 (2026-08-01)

First published version.

- The CDD loop: state, implement or generate, verify, diagnose, retain as
  new knowledge, accept.
- The claim tuple (quantifiers, law, domain, tolerance, evidence route)
- The record schema: authoring and verified-record shapes, with `grammar`,
  `domain`, `route`, `verdict`, `meta`, and `authored` fields.
- The evidence-verdict vocabulary (`proven`, `holds`, `documented`,
  `falsified`, `skipped`).
