# CDD spec changelog

Versioned independently of any implementation. A record's
`lineage.CDD_spec_version` field states which version it conforms to.
(v0.1.0 spelled this `lineage.spec_version`; see the 0.2.0 entry.)

## 0.2.0 (2026-09-16)

Drafted from experience implementing v0.1.0. The largest change is one of
scope: v0.2 states the method and pins only a small normative core,
leaving the rest to implementations. Nothing from v0.1.0 is removed or
repointed; several things it required become recommended instead.

**Scope**

- **A small normative core.** Two tools must agree on `claims`
  (`name`, `statement`, `verdict`, `route`, `authored`), `grammar`,
  `identity` (`form`, `sig`) and `lineage` (the spec version).
  Everything else is implementation-defined. Fields named elsewhere in
  these documents are recommended spellings for facts a tool may want to
  record, not a checklist it is measured against.
- **Claim identity belongs to the grammar.** Canonical rendering,
  alternative spellings, and whether a claim has a fingerprint at all are
  the grammar's to define and publish, not this specification's, for the
  same reason a domain's internal structure already was.

**Method**

- **Say what you know, and no more**: the general epistemic rule, stated
  once. A tool never reports a stronger epistemic state than it has a
  basis for, and not knowing is spelled out rather than defaulted to a
  confident value. Most of the rest of this document is that rule applied
  to a case.
- **A falsification requires an executed witness**, and a retained
  counterexample is replayed as a pin on every later adjudication.
- **Additive-only vocabularies**: a released value is never renamed and
  never repointed.

**Vocabulary**

- `unknown` (adjudication settled nothing, or the claim has not been
  checked) and `invalidated` (was supported, no longer is) join the
  ladder; `skipped` narrows to attempted-and-blocked-with-a-reason, which
  is what it was sharing with `unknown`.
- A standard stance fold (`supported`, `refuted`, `blocked`,
  `undecided`), so tools counting verdicts produce comparable numbers.
- The colon subtype convention, classified at the first colon, and the
  rule that `route` names the mechanism that actually decided, so cascade
  values never reach a record.

**Claims and records**

- **Premises**: a claim true only under a side condition states it rather
  than being narrowed or dropped, and a premise naming another claim caps
  the evidence at the weakest link.
- **Typed claim dependencies** (`kind`, `ref`, `requires`), with `claim`
  and `function-form` as the well-known kinds.
- **Claim membership is append-only**: an adjudicated claim leaves the
  live list only by being superseded, retained as a discovery, or marked
  historical, and is retained in every case.
- **Acceptance** is defined by its principle: it records who decided and
  when, binds to the version of code it was about, and lapses when that
  version moves. The shape is the tool's.
- **Freshness composes**: `form` alone does not establish that a claim is
  current, because behaviour also depends on what a function calls and
  reads.
- **`authored` becomes an object**, so one field serves both a small
  project and a regulated one. Only `authored.surface` is required, on
  the reasoning that it is the single fact a checking tool cannot fail
  to have; `ref`, `by`, `at`, `commit` and a `reviewed` event list are
  recorded when a tool has them, and a tool may add its own keys. The
  declared layer may now carry the part only the author knows, and a
  stated author is never overwritten by a checker. A v0.1.0 bare string
  reads as `{ref: <string>}`.
- **`lineage.spec_version` becomes `lineage.CDD_spec_version`**, spelled
  out because a record routinely carries version fields from several
  layers and which specification it follows should not be the ambiguous
  one.
- `lineage.commit` and `claims[].condition` are named as optional fields.

## 0.1.0 (2026-08-01)

First published version.

- The CDD loop: state, implement or generate, verify, diagnose, retain as
  new knowledge, accept.
- The claim tuple (quantifiers, law, domain, tolerance, evidence route)
- The record schema: authoring and verified-record shapes, with `grammar`,
  `domain`, `route`, `verdict`, `meta`, and `authored` fields.
- The evidence-verdict vocabulary (`proven`, `holds`, `documented`,
  `falsified`, `skipped`).
