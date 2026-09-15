# Record schema

Status: v0.2.0-draft. The shape splits into two files,
[`declared-schema.md`](declared-schema.md) (how a claim set is proposed)
and [`verified-schema.md`](verified-schema.md) (what a conformant tool
writes after checking one). This document covers what's true of both, or
true of the relationship between them.

## What this document requires, and what it leaves alone

This specification describes a method. It is deliberately broader than
any one implementation, and it is not a schema a tool validates against
field by field.

What it pins is a **small normative core**: the handful of things two
tools must agree on to read each other's records at all. Everything
outside that core is implementation-defined, and a tool that carries more
than the core, differently named or differently arranged, is conformant
so long as the core is present and correct.

**The normative core.** A verified record MUST carry, under these names:

| | |
|---|---|
| `claims` | the list of claims, each with `name`, `statement`, `verdict`, `route` (`null` until an adjudication actually decided, since `route` names the mechanism that decided), and `authored` (an object, of which only `authored.surface` is required) |
| `grammar` | which expression dialect the statements are written in, at function level or per claim |
| `identity` | with at least `form` and `sig` |
| `lineage` | with at least the spec version the record conforms to |

That is the whole of it. A conformance suite tests that, and nothing
beyond it.

**Everything else is implementation-defined**, including: every other
field named anywhere in this specification, how a tool arranges what it
knows, what it puts under `meta` or beside the core, what it stores about
dependencies, concepts, references, symbolic forms, or its own analysis,
and where and how it persists any of it.

Fields named in the rest of these documents are **recommended spellings
for facts a tool may well want to record**, not a checklist. Where one is
described below, take it as "if you record this fact, here is the name
and meaning other tools will expect", not as "you must record it".

## Declared vs verified

There are two shapes, and they are not peers. The **declared** shape is
how a claim set is proposed, before verification: intermediary, disposable,
not yet evidence of anything. The **verified** shape is what a conformant
tool writes after actually checking a declared claim set, the output of
running it through a CI gate or a development cycle, and it's this shape
that matters and persists. A declared claim set that never gets checked is
just an idea; a verified record is the knowledge base artifact this whole spec exists to
produce. Nothing downstream, a reader, a CI gate, an AI agent, another tool, should
trust the declared shape for anything beyond what claims someone intended
to check.

**A verified record stands on its own.** Everything needed to read and
re-check a claim is carried forward into the verified shape rather than
left behind in the declared one: at minimum the statement, the grammar it
is written in, the route and tolerance that shaped how it was checked,
and the domain that scoped it. A reader, or another tool, should never
need to go find whatever declared file originally proposed a claim just
to understand the verified record in front of them; the declared shape is
genuinely intermediary, disposable once it's been checked, not a second
source of truth the verified record quietly depends on.

This is a principle, not a field list. A tool that carries the same
information under different names still satisfies it; a tool whose
records only make sense with the declared file open does not.

The examples in `declared-schema.md` and `verified-schema.md` are
illustrative of the shape, not a literal dump from any one tool.

## Field alignment

For the fields this specification names, which shape(s) they appear in,
and what changes crossing from declared to verified. Only the rows marked
**core** are required; the rest are recommended spellings.

| field | declared | verified | notes |
|---|---|---|---|
| `name` | claim name, required | carried forward unchanged | **core** |
| `statement` | required | required, carried forward unchanged | **core**; same field name in both shapes, not renamed in transit |
| `route` | optional, default `probe` | required once adjudicated, `null` before (a never-adjudicated claim has no mechanism to name) | **core**; see "Open for extension" below |
| `verdict` | doesn't exist | required | **core**; a declared claim hasn't been checked yet |
| `authored` | optional, what the author knows (`by`, `at`, `ref`) | required object; the tool stamps what it observes and merges the declared part forward | **core**, but only `authored.surface` within it; see `verified-schema.md` |
| `grammar` | recommended, function-level or per-claim | carried forward | **core**; needed to read `statement` at all |
| `identity` | doesn't exist | required | **core**, with at least `form` and `sig`; nothing to hash before there's code |
| `lineage` | doesn't exist | required | **core**, with at least `CDD_spec_version`. Record-level; not to be confused with a claim's own `authored` |
| `domain` | optional, default `(-inf, inf)` | carried forward as declared | scoped the sampling that produced the verdict |
| `condition` | doesn't exist | optional | the region the evidence actually covered, which may be narrower than `domain` |
| `tolerance` | required when relevant, no default | carried forward when present | so a verified record is self-contained |
| `meta` | optional | carried forward unchanged | opaque extension point, see below |
| `intent` | proposed, pre-check | documented, from the docstring when one exists | see "Where `intent` comes from" below |
| `signature` | free-text, human-written, not verified | from the tool's own introspection | |
| `accepted` | doesn't exist | records a human decision about a claim | see `verified-schema.md`, "Acceptance" |
| `reasoning` | doesn't exist | generated by the tool | audit trail for how the record's own content was produced |

## Claim identity belongs to the grammar

Two tools reading the same claim need to agree that it is the same claim.
**This specification does not define how**, for the same reason it does
not define what a domain's internal structure looks like: a claim's text
is written in a grammar, and the grammar is what knows how to read it.

A grammar that is used across more than one tool is therefore responsible
for saying, in its own documentation:

- **which spellings it accepts, and which of them is canonical**, where
  it accepts more than one for the same thing;
- **how a claim in it is identified**, if the grammar offers an identity
  or fingerprint at all;
- **whether a rendering round-trips**, so that reading back what the
  grammar emitted gives the same claim.

Those are real obligations, and a grammar that leaves them unanswered
cannot be shared between implementations. They are just not this
document's obligations to discharge. A tool states which grammar it is
using (`grammar`), and everything about how that dialect's claims are
written, compared and identified follows from there.

The practical consequence is that questions like "does a conditional
claim's statement carry its premise" are settled by the grammar, once,
for every tool that uses it, rather than by this specification for
grammars it has never seen.

## The `meta` extension point

Rather than this spec naming specific fields for capabilities that don't
exist yet, `meta` is a single, optional, namespaced object that any tool or
layer can put arbitrary content under, at the function level, the claim
level, or both. Something nobody has designed yet writes whatever it
needs under its own key. None of that requires this document to change: the
core shape only needs to say that `meta` exists and that its contents are
opaque to a reader that does not recognise them.

A tool that doesn't understand a given `meta` key should pass
it through unchanged rather than silently drop it; a tool that does
understand it may read, enrich, or replace it. Nothing under `meta` is
required: a record with several extensions' worth of content under `meta`
is exactly as valid as one where `meta` is missing entirely.

Two rules on top of v0.1.0's:

- **Adjudication carries a claim's declared `meta` forward.** A tool that
  checks a claim copies that claim's declared `meta` onto the recorded
  claim, its own namespaced keys winning on a collision. Otherwise the
  pass-through rule holds only until the first checker runs, which is the
  moment it matters.
- **Consumers ignore what they do not recognise.** This applies to
  unrecognised keys wherever they appear, not only under `meta`. A reader
  should preserve them on rewrite and place no other meaning on them. A
  record carrying sections this document has never heard of is a normal
  record, not a malformed one.

## Open for extension: `route` and `verdict`

Both are strings, not a fixed enum a reader should reject unknown values
for. `claim-anatomy.md` and `evidence-ladder.md` document a well-known set
for each (`probe`/`derive` for `route`; `proven`/`holds`/`documented`/`declared`/
`unknown`/`falsified`/`invalidated`/`skipped` for `verdict`), and that set
is what gives the evidence ladder a meaningful strength ordering. A tool built on a different
verification technique, symbolic execution or an SMT solver, say, is
expected to use whatever route and verdict actually fit what it did rather
than force its evidence into the closest existing value. A value outside
the documented set above is unranked in this version, not malformed, and
folds to `undecided` in a rollup (see `evidence-ladder.md`, "Rolling
verdicts up").

### Subtypes: the colon convention

A tool often knows more about how a verdict was reached than the base
value carries. Rather than inventing a parallel field, subtype the value
with a colon: `probe:semi_analytical`, `derive:extensive`,
`skipped:unparseable`.

**Classification splits at the first colon.** `skipped:unparseable` is a
skip to every consumer that does not recognise the subtype, and anything
prefixed `falsified:` counts as falsified in every gate. That is the
whole contract: a reader that knows the subtype may use it, a reader that
does not falls back to the base value and is never wrong about which
bucket the claim is in.

Two consequences worth stating:

- **A subtype never changes the bucket.** A value whose base is
  `skipped` is treated as a skip, so a tool cannot use `skipped:` to
  smuggle a passing verdict past a strict gate.
- **`route` names the mechanism that actually decided**, so cascade or
  preference values (`auto`, `best`, "try the strongest first") are
  **input-side instructions only** and never appear in a record. A
  record's `route` is a statement about what happened.

### Open vocabularies are additive only

Every open vocabulary here, verdicts, routes, grammar names, and whatever
a tool names in its own namespace, follows one discipline: **a value is
public once released. It is never renamed, and never repointed at a
different meaning. A new meaning gets a new value.**

This is what lets a record written a year ago still be read correctly. A
renamed value silently breaks every stored record that used it, and a
repointed one is worse, because nothing breaks and the meaning quietly
changes underneath.

## Where `intent` comes from

`intent` appears in both shapes, and it means something different in each,
because the two sit on opposite sides of implementation.

In the **declared shape**, `intent` is a human's or a model's *proposed*
statement of purpose, written before anything is checked, and typically
before the function is even implemented. It functions as an input
requirement for whatever development cycle produces the code, human or
agentic: what you want realized. It has the same epistemic status as an
unchecked claim, someone's belief about what the function should be for,
not yet evidence of anything.

In the **verified shape**, `intent` should come from the function's own
docstring or doc comment whenever one exists, read directly off the real
function object the tool has in hand at check time (Python terms: roughly
`inspect.getdoc(fn)`, or whatever the equivalent introspection is in
another language), not copied from a separate YAML file. This is a
statement about what actually got realized, extracted from the finished
artifact rather than requested before it existed. That's the `documented`
evidence class from `evidence-ladder.md`: evidence read from the implementation itself,
not asserted about it.

**Documentation from the object wins whenever both exist.** If a declared `intent` from the
declared shape and the function's actual docstring disagree, the `documented` intent is taken. The rationale is as follows the declared intent describes what was requested, the
docstring describes what was actually built, and when they diverge, the
record is about the artifact as it stands, so the description of what was
actually realized outranks the description of what was originally wanted.
The declared `intent` is only used as a fallback, tagged `declared` rather
than `documented`, when the function has no docstring at all to read.

## Versioning

`lineage` states which version of this specification a record claims to
follow, and a reader should check it before assuming a field's meaning.
That is a core requirement; the recommended spelling is
`lineage.CDD_spec_version`. The key is spelled out rather than a bare
`spec_version` because a record routinely carries version fields from
several layers at once, and which specification a record follows should
not be the ambiguous one.

Whether a record additionally carries a version of the *shape* it is
written in, separate from the version of this specification, is
implementation-defined. A tool whose own record shape moves faster than
this document will want one.

## Where records get stored

The shape above is what travels between tools; where a tool persists it
is not, and nothing about the shape requires any particular file
layout. A record only needs to be findable by its dotted-name key; a store
could be one file with many top-level keys, one file per function, or
something else entirely, a database, whatever fits the tool.
File-per-function is a convenience some tools will reasonably choose
(clean per-function diffs, no two writers touching the same file), not a
requirement, and a file's name is not load-bearing either way: the record
already states its own key inside the YAML.


**Claims merge, they don't override.** The key is already the function's
own fully-qualified path, so two declared-layer files declaring the same
key are necessarily about the same function, not two different
interpretations of an ambiguous name; there's no identity ambiguity to
resolve. A tool merges every claim from every declared-layer file that
declares the key into one combined claims list, rather than picking a
winning file, the same way a shared project-wide file and a project-local
file both contributing tests to the same suite just means the suite has
more tests, not a conflict. A claim `name` repeated across files with the
same `statement` is redundant and can be silently deduplicated; for the same function key, the same
`name` with a *different* `statement` is a real conflict, since one named
claim can't honestly mean two things at once, and that's the case a tool
should surface as an error rather than resolve quietly.

That last rule is about a claim's *meaning*, not about which file it came
from, so it holds wherever two versions of one named claim meet: across
declared files, between a declared claim and a recorded one, and between
an authoring surface and the verified layer. A tool that resolves such a
conflict by preferring whichever version it happened to read last is
choosing silently on the author's behalf, which is the outcome this rule
exists to prevent.

- Singular string fields, like `intent` don't merge the way a claim list does, but are concatentated to allow different files to document different intents about the function if this pattern is so desired.
- `grammar` applied at function level applies to all claims define below it until another `grammar` key is detected, this allows for mixing of different claim grammars within the same declared function yaml. 

## Claim membership is append-only

A verified record's claim list does not shrink by forgetting. Once a
claim has been adjudicated and recorded, removing it from every authoring
surface does not delete what was learned: the next sweep repopulates it
from the record and keeps adjudicating it.

This is the storage-level consequence of `cdd.md`'s rule that a falsified
claim is retained as knowledge and "not deleted". A tool that rebuilds
its claim list purely from the current declared sources, without merging
forward from the previous record, silently drops exactly the retained
falsifications this specification says must survive.

There are three ways a claim leaves the live list, and each records why:
it was **superseded** by a re-authored version of the same named claim, it
became a **discovery** (`cdd.md`'s loop, step 5), or it is **historical**,
the code having moved past it entirely. In all three the row is retained,
not removed. How a tool arranges that retention is its own business; that
it retains it is not.
