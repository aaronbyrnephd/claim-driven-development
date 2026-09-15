# v0.2 decisions

Status: v0.2.0-draft. **Not part of the specification.** This file
records what was decided while drafting v0.2 and why, and is deleted
before v0.2.0 is published.

An earlier revision listed eleven open items. All are now ruled. Two
calls made while applying the last of them are flagged at the bottom for
a sanity check.

---

## Scope: the spec is the method, plus a small normative core

The spec describes claim-driven development. It is deliberately broader
than any implementation of it, and it is not a schema to validate
against field by field.

It pins a short list of things two tools must agree on and explicitly
leaves everything else to the implementation. See `record-schema.md`,
"What this document requires, and what it leaves alone."

This settled six items outright. Whether a tool emits
`identity.source_available`, what it calls its lineage keys beyond the
spec version, whether concepts and references and dependency records are
top-level sections or live under `meta`, whether it carries a
record-shape version: **all implementation-defined**. Recording those
facts is encouraged and the spec names spellings for them, but a tool
that does it differently is conformant.

## Claim identity belongs to the grammar

Whether two tools agree that they checked "the same claim", how a claim
is canonically rendered, whether alternative spellings collapse, and
whether there is a fingerprint at all: **not this document's business.**
A claim's text is written in a grammar, and the grammar is what knows
how to read it, the same way a domain's internal structure already was.

A grammar shared between tools has to answer those questions in its own
documentation. The spec says only that it must.

This settled the item with the most cross-implementation consequence by
moving it somewhere it can actually be answered. "Does a conditional
claim's statement carry its premise" is now decided once by each grammar
for every tool using it. What the spec still requires is that the
premise is not silently **lost**, which is a statement about honesty,
not about rendering.

## The epistemic rule is normative; the fields illustrating it are not

`cdd.md`, "Say what you know, and no more" states the general rule.
`identity.pure` is a worked example of it rather than a required field.

## An acceptance is defined by its principle, not its shape

Who decided, when, bound to the version of code it was about, lapsing
when that version moves. Both a scalar and a richer object satisfy that.

## `authored` is an object, sized to the setting

Ruled: nested, extensible, with as much detail as a given deployment
needs, resolving in the verified layer, with parts allowed to originate
in the declared layer and propagate.

**The minimum for interop is `authored.surface`,** and nothing else.
The reasoning: it is the one fact a checking tool cannot fail to have,
because it read the claim from somewhere and therefore knows where. Every
richer fact fails for some honest tool. A file path does not exist for a
claim the tool generated itself. An author is unknown for a claim
inherited from a repository nobody remembers. A commit does not apply to
a store outside version control. Requiring any of those would make the
field unfillable for a conformant tool; requiring less would let a record
present a claim from nowhere, which is what the field exists to prevent.

`ref`, `by`, `at`, `commit` and a `reviewed` event list are named with
defined meanings and are all optional, and a tool may add its own keys
alongside them. So a lightweight implementation writes
`{surface: docstring}` and a regulated one writes the commit, the git
identity and the review chain, without either being a different shape.

## `lineage.CDD_spec_version`

Ruled: the spec adopts the spelled-out key rather than a bare
`spec_version`, because a record routinely carries version fields from
several layers at once and which specification it follows should not be
the ambiguous one.

## Six documents, not one

Ruled: keep the split. One document would be too much at once.

---

## Two calls worth a sanity check

Neither is a question I was asked; both are small decisions made while
applying the `authored` ruling, flagged so they are seen rather than
inherited silently.

**The name stayed `authored`.** Renaming it to `lineage` was raised, and
there is a real argument for it now the block carries dates and review
events. I kept `authored` because a claim-level `lineage` beside the
record-level `lineage` is a genuine footgun in a specification trying to
stay small: two blocks, one name, different required keys, different
scope. `provenance` would be the better name on a blank sheet, but
`authored` is a published core name and the spec's own discipline is not
to rename released things. Easy to overrule.

**`reviewed` is a list, not a single entry.** Review is an event that
can happen more than once, and a tool that starts with one reviewer and
later needs an approval chain should not have to change shape to get
there. A single review is a one-element list.
