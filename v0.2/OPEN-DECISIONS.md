# v0.2 open decisions

Status: v0.2.0-draft. **Not part of the specification.** This file exists
for the review of this draft and is deleted before v0.2.0 is published.

An earlier revision of this draft listed eleven items. Four rulings
collapsed most of them, because they turned out to be questions about
what kind of document this is rather than questions about the method.

---

## Settled

### The specification is the method, plus a small normative core

The spec describes claim-driven development. It is deliberately broader
than any implementation of it, and it is not a schema to validate
against field by field.

It pins a short list of things two tools must agree on (`claims` with
`name`/`statement`/`verdict`/`route`/`authored`, plus `grammar`,
`identity`, `lineage`) and explicitly leaves everything else to the
implementation. See `record-schema.md`, "What this document requires,
and what it leaves alone."

This settled six of the original eleven outright. Whether a tool emits
`identity.source_available`, what it calls its lineage keys beyond the
spec version, whether concepts and references and dependency records are
top-level sections or live under `meta`, whether it carries a record-shape
version: **all implementation-defined**. Recording those facts is
encouraged and the spec names spellings for them, but a tool that does it
differently is conformant.

### Claim identity belongs to the grammar

Whether two tools agree that they checked "the same claim", how a claim
is canonically rendered, whether alternative spellings collapse, and
whether there is a fingerprint at all: **not this document's business.**
A claim's text is written in a grammar, and the grammar is what knows how
to read it, the same way a domain's internal structure is already the
grammar's business rather than the spec's.

A grammar shared between tools has to answer those questions in its own
documentation. The spec says only that it must.

This settled the item with the most cross-implementation consequence, by
moving it to where it can actually be answered. "Does a conditional
claim's statement carry its premise" is now decided once by each grammar
for every tool using it, rather than by a specification that has never
seen that grammar. What the spec still requires is that the premise is
not silently **lost**, which is a statement about honesty, not about
rendering.

### The epistemic rule is normative; the fields illustrating it are not

`cdd.md`, "Say what you know, and no more" states the general rule: a
tool never reports a stronger epistemic state than it has a basis for,
and not knowing is spelled out rather than defaulted to a confident
value.

`identity.pure` is now a worked example of that rule rather than a
required field. A tool that cannot analyse a function and reports it as
pure is violating the rule, which matters more than whether it emits that
particular key.

### An acceptance is defined by its principle, not its shape

The spec requires that a human decision about a claim records who decided
and when, is bound to the version of code it was made about, and lapses
when that version moves. Both the scalar form and a richer object form
satisfy that. What neither may do is carry a decision forward onto code
nobody agreed to.

---

## Still open

### B1. What `claims[].authored` holds

The one survivor from the original list, because `authored` is in the
normative core: if two tools disagree about what it contains, the core
does not do its job.

- **This document says** an origin reference: a file path, a person, an
  agent, `geo.gc_distance:decorator:L142`. The point is that a falsified
  claim is traceable back to whoever wrote it, the way a failing test
  points at the file and line that defined it.
- **mathema writes** a *surface kind* here, from a closed vocabulary
  (`docstring`, `decorator`, `declared`, `types`, `suggested`, `builtin`,
  `ad_hoc`), and writes the origin reference in its declared layer
  instead, in exactly the format this document's example shows.

Both are useful and they are not the same fact. "Which file and line" and
"what kind of surface" answer different questions.

Options:

1. **`authored` is the origin reference**, as written. mathema moves its
   existing declared-layer tag into the verified record. The surface kind,
   if it wants to keep it, goes beside it or under `meta`.
2. **`authored` is whatever traces the claim**, origin reference or
   surface kind, tool's choice. Weakest: a reader can no longer rely on
   getting back to the source.
3. **Two fields**: `authored` for the origin reference (core, required)
   and something like `authored_surface` for the kind (optional).

**Ruling** (tick one, in PR #1's description):

- [ ] **B1.1** `authored` is the origin reference; mathema moves its
      declared-layer tag into the verified record
- [ ] **B1.2** `authored` is either; tool's choice
- [ ] **B1.3** two fields: `authored` (origin, core) and
      `authored_surface` (kind, optional)

### A3. The spelling of the spec-version key

Minor, but it is inside the core, so it has to be one or the other. This
document says `lineage.spec_version`; mathema writes
`lineage.CDD_spec_version`.

Arguments each way: `spec_version` is what the published v0.1.0 says and
what any existing reader would look for; `CDD_spec_version` is less
ambiguous in a record that may carry several version fields from
different layers.

**Ruling** (tick one, in PR #1's description):

- [ ] **A3.1** `lineage.spec_version`; mathema moves
- [ ] **A3.2** `lineage.CDD_spec_version`; the spec moves

### S1. Six documents, or one

v0.1.0 is six documents. This draft keeps the split so the diff is
reviewable per concern. Consolidation has been raised before and never
ruled, and is a separate question from any of the content above.

**Ruling** (tick one, in PR #1's description):

- [ ] **S1.1** keep the six documents
- [ ] **S1.2** consolidate to one
- [ ] **S1.3** decide after the content is agreed

---

## Owed on the implementation side, once the above is ruled

Not spec questions, listed so they are not lost. Each is mathema
breaking a rule this document states, rather than a disagreement about
what the rule should be.

- `identity.pure` is `true` for a function whose source could not be
  read, which violates "Say what you know, and no more."
- Re-verification rebuilds the claim list from current declared sources
  without merging forward from the previous record, so deleting a claim
  from an authoring surface drops its verified history, including
  retained falsifications that `cdd.md` says survive.
- A same-name, different-statement conflict is resolved by preferring
  whichever version was read last, where `record-schema.md` requires it
  to be surfaced as an error.
