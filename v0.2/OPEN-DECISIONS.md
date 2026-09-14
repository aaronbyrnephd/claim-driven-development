# v0.2 open decisions

Status: v0.2.0-draft. **Not part of the specification.** This file exists
for the review of this draft and is deleted before v0.2.0 is published.

Each item below is a place where v0.1.0 and the reference implementation
(mathema) disagree, and the disagreement can be closed from either side.
Nothing here is pre-decided. Each item states what v0.1.0 says, what the
implementation does, what it costs to move each side, and leaves the
ruling open.

Where an item is referenced from the draft spec text, the text carries an
`> **OPEN (id)**` marker so no reader mistakes a draft position for a
settled one. Every one of those markers is removed when the item is
ruled.

Legend for the cost columns: *low* is a contained change, *medium* means
touching several call sites or a published surface, *high* means a
breaking change for anything already reading records.

---

## A. Live non-conformances

The implementation's own conformance suite already marks these three as
`xfail(strict=True)`, so they are known, not newly discovered.

### A1. `identity.source_available`

- **v0.1.0 says**: required. "Whether the tool had the function's source
  to analyze at all, as opposed to working from a docstring or other
  external documentation only."
- **The implementation does**: does not emit it. Its `identity` block is
  `form`, `sig`, `tier`, `pure`, `claims_fingerprint`, `integrity`.
- **If the implementation moves**: low. The information already exists
  (a doc-only record is exactly the case where source was unavailable);
  it is one key.
- **If the spec moves**: the field is dropped, and readers lose the only
  way to tell "no claim proved" from "nothing could be read in the first
  place". Note `route`/`verdict` per claim do **not** answer this, which
  is why v0.1.0 added it.

**Ruling:** [ ] implementation moves  [ ] spec moves  [ ] other

### A2. `identity.pure` when the function cannot be analysed

- **v0.1.0 says**: `pure` is `true` only when the tool has a real basis
  for saying so. "When it can't be analysed, `pure` is `null` instead of
  `false`."
- **The implementation does**: sets `pure: true` unconditionally on the
  doc-only path, with the comment "unverifiable without source; probes
  still check determinism". So a function whose source could not be read
  is reported as provably pure.
- **If the implementation moves**: low as a change, but it is a
  behavioural correction, not a rename, and anything consuming `pure` as
  a bool has to handle `null`.
- **If the spec moves**: `pure` becomes a two-valued field and the
  "unknown purity" state disappears from the schema.

This is the one item where the two answers are not equally defensible:
reporting an unanalysable function as pure asserts something the tool has
no basis for, which is the failure mode the rest of this specification
exists to prevent.

**Ruling:** [ ] implementation moves  [ ] spec moves  [ ] other

### A3. `lineage` key names and timestamp granularity

- **v0.1.0 says**: `lineage.spec_version` and `lineage.timestamp`, the
  latter a full datetime (`"2026-08-11 16:04:21"`).
- **The implementation does**: `lineage.CDD_spec_version` and
  `lineage.date`, the latter a bare date (`"2026-09-14"`). It also adds
  `lineage.commit`.
- **If the implementation moves**: low, but it rewrites a key in every
  record already on disk, so it wants a migration or a reader that
  accepts both for one version.
- **If the spec moves**: `CDD_spec_version` is arguably the clearer name
  in a record that may carry several version fields (see F1), and a bare
  date is arguably honest about the precision that matters. Either way
  the spec should say which, since two tools cannot both be right.

**Ruling:** [ ] implementation moves  [ ] spec moves  [ ] other

---

## B. Semantic divergence

Neither of these is caught by any conformance test today, because both
fields are present and only their *meaning* differs.

### B1. What `claims[].authored` holds

- **v0.1.0 says**: required, and it traces a claim "back to whoever wrote
  it, the same way a failing test points at the file and line that
  defined it". The worked examples are `geo/symspec.yaml` and
  `geo.gc_distance:decorator:L142`. The declared shape has no such field;
  the checking tool stamps it.
- **The implementation does**: writes a *surface kind* from a closed
  vocabulary in the verified layer (`docstring`, `declared`, `decorator`,
  `types`, `suggested`, `builtin`, `ad_hoc`, `unknown`), and writes the
  path-like form the spec's example shows in its *declared* layer, which
  the spec says has no such field at all. The two layers are swapped
  relative to the specification.
- **If the implementation moves**: medium. The information exists on both
  sides but is currently split across two layers.
- **If the spec moves**: v0.2 says `authored` holds either an origin
  reference or a surface kind, which weakens it: a reader can no longer
  rely on being able to get back to the file.

A third option: keep both, as `authored` (origin reference, required)
and a new `authored_surface` (kind, optional).

**Ruling:** [ ] implementation moves  [ ] spec moves  [ ] keep both  [ ] other

### B2. The shape of `claims[].accepted`

- **v0.1.0 says**: a scalar, `null` / `false` / `true` / an accepting
  identity, "only meaningful when `verdict` is `falsified`". `null` means
  undiagnosed, `false` means diagnosed as a bug, anything else means
  diagnosed as a discovery.
- **The implementation does**: an object,
  `{as, at, form, [by], [note], [stale]}`, across five acceptance kinds
  (`evidence`, `risk`, `discovery`, `historical`, `superseded`), plus
  `acceptance_history` and `superseded_by`. It accepts against `holds`
  and `unknown` too, not only `falsified`, and binds each acceptance to
  the `form` hash so it goes stale when the code moves.
- **If the implementation moves**: high. The five-kind model is load
  bearing; collapsing it to a scalar loses the staleness binding, which
  is the part CI actually depends on.
- **If the spec moves**: v0.2 has to describe the object, the five kinds,
  and form-binding. That is a substantial addition, and it makes the
  simple case (a falsified claim someone signed off) more verbose.

Note the two are not merely different shapes: v0.1.0's `accepted` answers
"was this falsification a bug or a discovery", while the implementation's
answers "what did a human decide about this claim, and is that decision
still current". The second subsumes the first.

**Ruling:** [ ] implementation moves  [ ] spec moves  [ ] other

---

## D. Sections the specification does not name

v0.1.0 provides exactly one extension point: `meta`, "a single, optional,
namespaced object that any tool or layer can put arbitrary content under,
at the function level, the claim level, or both". Its two worked examples
are `meta: {concepts: [...]}` and `meta: {math: {...}}`.

The implementation promoted both of those to top-level keys and added
eight more. Unknown *top-level* keys are not sanctioned the way unknown
`meta` keys are, so each group below is either a v0.2 field or belongs
under `meta`.

### D1. Knowledge and analysis sections

`concepts` (list of tags), `references` (role-nested citations), `math`
(the lifted symbolic form), `dependencies` (one-deep callees with their
own form hashes and a freshness state), `raises` (the exception surface).

- **For promoting**: these are not tool-private decoration. `concepts`
  are knowledge-graph node ids, `references` is how a function carries
  its own model-risk documentation, and `dependencies` is what makes
  staleness composable across a call graph, which v0.1.0 has no concept
  of at all.
- **For leaving under `meta`**: the specification stays small, and
  nothing here is needed to read a claim or its verdict.

**Ruling:** [ ] promote all  [ ] promote some (list)  [ ] keep under `meta`

### D2. Acceptance and history sections

`discoveries`, `historical`, `superseded` (each a list of whole claim
rows moved out of `claims`), `concepts_accepted`, `intent_accepted`.

These exist because the implementation treats claim membership as
append-only: a claim never simply disappears, it moves to a section that
says why. That is a direct answer to v0.1.0's own rule that a falsified
claim "is never deleted", which v0.1.0 states but gives no place to
record.

**Ruling:** [ ] promote all  [ ] promote some (list)  [ ] keep under `meta`

### D3. Extra `identity` keys

`tier` (an analysis depth, already noted in the implementation's own test
as an undocumented extension), `claims_fingerprint` (a hash over the
canonical claim text, see E1), `integrity` (a checksum over
`(name, verdict)` pairs plus `form`, currently advisory).

`claims_fingerprint` is the one with cross-implementation consequences:
it is meant to be portable, so if it is in the schema at all, v0.2 has to
say exactly what is hashed.

**Ruling:** [ ] document all three  [ ] document `claims_fingerprint` only  [ ] keep under `meta`

---

## E. Canonical form

### E1. Does a conditional claim's `statement` carry its premise?

- **The implementation now renders it in.** A claim written
  `assuming x + y == 2, f(x, y) <= 1` records a `statement` that includes
  the `assuming` clause. This was a soundness fix: without the premise,
  a statement round-tripped through the declared layer re-parsed as an
  *unconditional* claim, and the example above came back falsified on a
  point the premise excludes.
- **Why it matters beyond one tool**: `claims_fingerprint` hashes the
  statement and is explicitly the portable, cross-language identity of a
  claim. Two conformant tools that render a conditional claim differently
  compute different fingerprints for the same claim. v0.2 has to state
  which rendering is canonical.
- **The implementation's position**: the premise is part of the claim; a
  statement that omits it does not denote the claim; a rendered claim
  must round-trip through the grammar that produced it.

A companion rule the implementation also adopted: where a grammar accepts
several spellings of one thing, they **must** canonicalise, because
anything that reaches the fingerprint has to. (`is_defined(f)` and
`f is defined` were producing two fingerprints for one claim.)

**Ruling:** [ ] premise is part of `statement`  [ ] premise is a separate field  [ ] other

---

## F. Versioning

### F1. A record-shape version, separate from the spec version

There is no version on the record *shape*. `spec_version` (or
`CDD_spec_version`, see A3) names which specification a record claims to
follow, but the shape can move within a spec version, and already has in
the reference implementation (`lineage.commit` and `identity.integrity`
are both newer than the field list they sit in).

A consumer reading a store written over a year has no way to ask "which
shape is this row" without inferring it from which keys happen to be
present.

**Ruling:** [ ] add `schema_version`  [ ] spec version is enough  [ ] other

Sub-questions if added: what is it called, where does it sit (top level
or inside `lineage`), and is it an integer that increments or a semantic
version.

---

## S. Structure

### S1. Six documents, or one

v0.1.0 is six documents totalling 965 lines. The split is: the loop
(`cdd.md`), the claim tuple (`claim-anatomy.md`), the two shapes
(`declared-schema.md`, `verified-schema.md`), what is true of both
(`record-schema.md`), and the verdict vocabulary
(`evidence-ladder.md`).

This draft keeps the split, so the diff is reviewable per concern. A
consolidation to one document has been raised before and never ruled. It
is a separate decision from everything above and can be taken after the
content is settled.

**Ruling:** [ ] keep six  [ ] consolidate to one  [ ] decide after v0.2 content is agreed
