# CDD spec changelog

Versioned independently of any implementation. A record's
`lineage.spec_version` field states which version it conforms to.

## 0.2.0 (unreleased, draft)

Drafted from experience implementing v0.1.0. Everything here is additive
or a narrowing of something v0.1.0 left ambiguous; no v0.1.0 field is
removed or repointed. `v0.2/OPEN-DECISIONS.md` lists the questions this
draft deliberately leaves open, and is deleted before release.

- **Verdict vocabulary**: `unknown` (adjudication settled nothing, or the
  claim has not been checked yet) and `invalidated` (a claim that was
  supported and no longer is) join the ladder. `skipped` narrows to
  attempted-and-blocked-with-a-reason, which is what it shared with
  `unknown` before.
- **Stance fold**: a standard rollup (`supported`, `refuted`, `blocked`,
  `undecided`), so tools counting verdicts produce comparable numbers.
- **Subtypes**: the colon convention (`probe:semi_analytical`,
  `skipped:unparseable`), classified at the first colon, and the rule
  that `route` names the mechanism that actually decided, so cascade
  values never appear in a record.
- **Canonical form**: a claim's identity is its text in the grammar it
  declares, not a tool's field layout. Grammars must canonicalise
  alternative spellings, and a canonical rendering must round-trip.
- **Premises**: a claim true only under a side condition states it, and
  a premise naming another claim caps the evidence at the weakest link.
- **Claim dependencies**: one typed entry (`kind`, `ref`, `requires`),
  with `claim` and `function-form` as the well-known kinds.
- **Records have memory**: claim membership is append-only, with
  `superseded`, discovery and `historical` as the three retaining exits;
  `invalidated` makes a regression explicit rather than silent.
- **Acceptance**: an object form binding a human decision to the `form`
  hash it was made about, so the decision goes stale when the code moves.
- **Freshness composition**: function-level dependencies carry a callee's
  `form` and a constant's value, so a behavioural change nothing else
  would notice still invalidates the claims that rested on it.
- **Soundness language**: a falsification requires an executed witness,
  and a retained counterexample is replayed as a pin on every later
  adjudication.
- **Additive-only vocabularies**: a released value in any open vocabulary
  is never renamed and never repointed.
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
