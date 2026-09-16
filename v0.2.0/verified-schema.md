# Verified shape

Status: v0.2.0. Part of the record schema; see `record-schema.md` for how
this relates to the declared shape.

What a conformant tool writes after checking a function's claims. One entry
per function, keyed the same way. Where a tool persists this on disk is not
part of this shape; see `record-schema.md`, "Where records get stored."

Only the normative core is required (`record-schema.md`, "What this
document requires, and what it leaves alone"). Everything else below is a
**recommended spelling for a fact a tool may want to record**: if you
record it, this is the name and meaning other tools will expect. The
example is illustrative of a rich record, not a minimum.

```yaml
geo.gc_distance:
  name: gc_distance
  signature: "(phi1: float, lam1: float, phi2: float, lam2: float) -> float"
  intent: Length of the great-circle arc between two points on the unit sphere
  grammar: python-expression
  identity:
    form: 9f3ac2e1b4a0
    sig: 7b1e4c9a0d2f
    source_available: true
    pure: true
  claims:
    - name: symmetric
      statement: "d(p, q) == d(q, p)"
      route: derive
      verdict: proven
      counterexample: null
      condition: "for all p, q on the unit sphere"
      authored:
        surface: claims-file
        ref: gc_distance_cdd.yaml
    - name: nonnegative
      statement: "d(p, q) >= 0"
      route: probe
      verdict: holds
      n: 72
      counterexample: null
      domain:
        p: [-1.5707963, 1.5707963]
      meta:
        concepts: [metric-space]
        comments: "......"
      authored:
        surface: decorator
        ref: "geo.gc_distance:decorator:L142"
        by: janesmith@corp.org
        at: "2026-08-10 14:20:05"
    - name: never_exceeds_pi
      statement: "d(p, q) <= 3.14159265"
      route: probe
      tolerance: 1e-9
      verdict: falsified
      n: 64
      counterexample: [(1.5707963, 0.0),(-1.5707963, 0.0)]
      accepted: janesmith@corp.org 
      authored:
        surface: claims-file
        ref: geo/symspec.yaml
        by: janesmith@corp.org
        commit: 4f2a9c1e8b7d3a6f0e5c2b9d8a1f4e7c3b6d0a9e
        reviewed:
          - by: alexlee@corp.org
            at: "2026-08-12 09:02:11"
    - name: monotone_in_latitude
      statement: "d(p, q) >= d(p, r)"
      route: probe
      verdict: invalidated
      n: 128
      counterexample: [(0.4, 0.0),(0.9, 0.0),(0.6, 0.0)]
      authored:
        surface: docstring
      meta:
        previous_verdict: holds
  reasoning:
    - step: intent
      claim: Length of the great-circle arc between two points on the unit sphere
      basis: "documented; read from the function's own docstring"
    - step: evidence
      claim: "d(p, q) == d(q, p)"
      basis: "proven, derived from symbolic logic"
  lineage:
    generated_by: example-tool 0.1.0
    CDD_spec_version: 0.2.0
    timestamp: "2026-08-11 16:04:21"
    commit: 4f2a9c1e8b7d3a6f0e5c2b9d8a1f4e7c3b6d0a9e
```

The `n: 64` above is not a canonical number; it's however many trials the
tool that produced this record actually ran, and different tools, or the
same tool with a different trial budget, will show a different one. It is
the specific number this run used, not a constant the shape prescribes.

**`reasoning` isn't separately authored.** A conformant tool generates it
as a byproduct of the same work that produces the rest of the record, one
entry per step the tool actually took, each citing what that step is based
on: an `intent` step cites wherever intent came from (see `record-schema.md`,
"Where `intent` comes from"); an `evidence` step is added per claim that got
a real verdict, citing that verdict's own basis. There's no fixed list of
`step` values because a tool that does more, a `structure` step reading the
AST, a `formalization` step for a lifted symbolic form, adds them the same
way. The rule is simply: state what you did, and what you're basing it on.

**The verified shape deliberately carries forward `statement`, `route`,
`tolerance` (when present), `grammar`, `domain`, and `meta`** from the
declared shape that proposed each claim, so nothing about the verified
record depends on a reader also having the declared one in hand:
`statement` and `grammar` together because a statement is unreadable
without knowing which dialect it's written in, `route` and `tolerance`
because they're part of how the verdict was actually reached, not just
how it was proposed, `domain` because it's what scoped the sampling that
produced the verdict, and `meta` because a tool that doesn't understand a
given extension should still pass it through rather than lose it on the
way to a verified record.

`authored` works differently: the checking tool stamps what it observes,
and merges in whatever the declared layer knew that it cannot. See
"`authored`: tracing a claim back to where it came from" below.

Notes:

- **`identity.form` and `identity.sig` answer different questions, which
  is why both exist.** `form` (a rename/format-invariant structural hash
  of the function body; one common technique is to alpha-normalize the AST,
  then hash it) answers "is this the same implementation." `sig` (a hash of
  the parameter signature) answers "is this the same interface." A single
  hash can't answer both questions at once, and a reader needs both
  answers, so there are four cases, not one:
  - **Both unchanged**: nothing meaningful moved (a rename doesn't count,
    `form` is already invariant to that).
  - **`sig` changes, `form` doesn't**: the declared contract moved
    (widened, narrowed, a parameter added) while the actual computation
    didn't. Worth knowing, low risk: callers relying on the old interface
    may need to adapt, but the behavior they were already depending on is
    unchanged.
  - **`form` changes, `sig` doesn't**: the dangerous case. A silent
    behavioral change behind a stable-looking interface, exactly the kind
    of drift a caller has no other way to notice, since nothing about the
    interface it depends on appeared to move.
  - **Both change**: an open, visible rewrite. The least dangerous kind of
    change precisely because nothing about it is hidden.

  A real behavioral change invalidates `form`, and exactly the claims that
  depended on the part that changed; a rename invalidates neither.

  This also makes `form` a cheap re-verification filter: a tool can compare
  the stored `form` against the current one before doing any real checking,
  and skip straight to re-reporting the existing verdicts for every function
  whose `form` hasn't moved, since nothing it previously verified could have
  changed. Re-verification only needs to touch the functions whose `form`
  actually did.

  `form` alone is not a complete freshness test, because a function's
  behaviour also depends on what it calls and what it reads. See
  "Composing freshness" below.

- **`identity.source_available`** is whether the tool had the function's
  source to analyze at all, as opposed to working from a docstring or
  other external documentation only. Worth recording, because "nothing
  was proved" and "there was nothing to read" are different facts and a
  reader cannot tell them apart from the claims alone. Optional: a tool
  that never works from documentation alone has nothing to say here.

- **`identity.pure`** is `true` only when the tool has a real
  basis for saying that the function has limited side effects (from source code analysis).
  When it can't be analysed, `pure` is `null` instead of `false`. This is
  the general rule in `cdd.md`, "Say what you know, and no more,"
  applied to one field: `false` asserts impurity, and a tool that could
  not look has not established that either.

- **`claims[].verdict`** is `proven`, `holds`, `documented`, `declared`,
  `unknown`, `falsified`, `invalidated`, or `skipped` in a v0.2
  record (see `evidence-ladder.md` for the level of trust to put in each),
  open to other values from tools using other verification techniques, per
  `record-schema.md`, "Open for extension."

- **`claims[].condition`** is the region the evidence actually covered,
  which may be narrower than the declared `domain`: a proof may hold on
  a sub-region, and a premise may constrain the region further. `domain`
  states what was claimed, `condition` states what was established.
  Optional, and display-oriented: a reader should not treat it as the
  claim's own scope.

- **`claims[].accepted`** records a human decision about a claim. In its
  simplest form it is `null`, `false`, `true` or an accepting identity, and
  in that form it is only meaningful when `verdict` is `falsified`. `null` means the
  falsification hasn't been diagnosed yet (`cdd.md`'s loop, step 4).
  `false` means it was diagnosed as an implementation bug: expected to be
  fixed and re-verified back at step 2, with the counterexample kept as the
  regression case that fix has to satisfy. Anything else means it was
  diagnosed as a genuine discovery and is retained as knowledge (step 5):
  a bare `true` when a tool has no identity to attach, or, the same
  flexibility `authored` allows, a person's name, or
  whatever else records who actually made that call. See "Acceptance"
  below for the fuller form.

- **`lineage.CDD_spec_version`** states which version of this document the
  record conforms to. A reader should check it before assuming a field's
  meaning, since the schema may grow.

- **`lineage.commit`** is the revision of the code the record was written
  against, when the tool has one. Optional, and worth having: it is what
  lets a reader go and look at the exact implementation a verdict was
  about, rather than the one in front of them now.

### `authored`: tracing a claim back to where it came from

Once claims from more than one file, or a mix of declared-layer and
tool-extracted claims, end up merged into one list, a falsified claim
needs to be traceable back to whoever wrote it, the same way a failing
test points at the file and line that defined it, not just a bare
assertion with no origin.

`authored` is an object, and **it is required**: every claim in a
machine-produced record traces to something, and a record should not be
able to present a claim with no answer to "where did this come from."

How much it holds varies enormously by setting, and deliberately so. A
small project may know only that a claim came from a docstring. A
regulated one needs the commit, the git identity, and who reviewed it.
Both are conformant.

```yaml
      # the least a tool can say
      authored:
        surface: docstring

      # a fuller record of the same claim
      authored:
        surface: claims-file
        ref: "geo/symspec.yaml#L14"
        by: janesmith@corp.org
        at: "2026-08-11 16:04:21"
        commit: 4f2a9c1e8b7d3a6f0e5c2b9d8a1f4e7c3b6d0a9e
        reviewed:
          - by: alexlee@corp.org
            at: "2026-08-12 09:02:11"
```

**`surface` is the only required key**, and it is the minimum for
interop. It is the one fact a checking tool cannot fail to have: it read
the claim from somewhere, and it knows where. A file path may not exist
(a claim the tool generated itself has none), an author may be unknown
(a claim inherited from a repository nobody remembers), a commit may not
apply (a store outside version control), but the surface is always
known. Requiring anything richer would make the field unfillable for
some honest tool, and requiring less would let a record present a claim
from nowhere.

Well-known values, open for extension the same way `route` and `verdict`
are: `docstring`, `decorator`, `annotation`, `claims-file`, `inline`,
`generated`.

The rest are recommended spellings. Record them when you have them:

| key | holds |
|---|---|
| `ref` | where exactly: a path, a `path#Lnn`, a URL, whatever locates it |
| `by` | who: a person, a team, an agent, or the tool's own name for a claim it generated |
| `at` | when the claim was authored |
| `commit` | the revision it was authored in |
| `reviewed` | a list of `{by, at}` events, for a claim someone checked before it was trusted |

`by` is worth recording even when it is a machine. `cdd.md`'s loop
explicitly allows a claim to be *proposed* by an agent, and explicitly
does not exempt such a claim from being checked; a reader who can see
that a claim was model-proposed can weigh it accordingly, and one who
cannot, cannot.

`reviewed` is a list rather than a single entry because review is an
event that can happen more than once, and because a tool that starts
with one reviewer and later needs an approval chain should not have to
change the shape to get it.

**Anything else goes in the same object.** A tool that tracks a ticket
number, an approval workflow state, or a model's prompt hash puts it
here under its own name, and a reader that does not recognise it passes
it through (`record-schema.md`, "The `meta` extension point"). The
object is open in exactly the way the rest of the specification is.

#### The declared layer may fill part of it

v0.1.0 said the declared shape had no `authored` field, on the reasoning
that a claim's origin is stamped by the checker rather than asserted by
the thing being checked. That holds for what a checker can observe, and
not for what only the author knows.

So: **a declared claim may carry an `authored` object with what its
author knows**, typically `by`, `at`, and a `ref` if the authoring
format has one. A checking tool fills in what it observes (`surface`
always, `commit` and a more precise `ref` when it can) and carries the
rest forward unchanged.

**A checking tool does not overwrite an authorship fact the declared
layer asserted.** It cannot know better than the author who wrote a
claim. Where both have a value for the same key, the declared one wins
for `by` and `at`, and the tool's wins for anything it observed
directly. A tool that finds itself discarding a stated author is doing
something wrong.

Either way, **the verified record is where this resolves**. A reader of
the record gets the merged result and never needs the declared file,
which is the same rule as everything else in this shape.

#### Reading a v0.1.0 record

In v0.1.0 `authored` was a bare string. A v0.2 reader encountering one
treats it as `{ref: <string>}`: v0.1.0's own examples were file paths
and references, so that is what the value meant. A v0.2 writer always
writes the object.

## Records have memory

Re-checking a function does not produce a fresh record that replaces the
old one; it produces the next state of the same record. Two consequences
are part of the shape.

**A claim that was supported and no longer is gets `invalidated`**, not a
silent downgrade. A tool writing it should carry the previous verdict
alongside, so a reader can see what was lost. See `evidence-ladder.md`,
"`invalidated`: records have memory."

**Claim membership is append-only.** A claim that has been adjudicated
does not vanish because it was deleted from an authoring surface; it is
repopulated from the record and keeps being adjudicated. The three exits
are `superseded`, a discovery, and `historical`, and each retains the row
rather than dropping it:

```yaml
  superseded:
    - name: never_exceeds_pi
      statement: "d(p, q) <= 3.2"
      verdict: holds
      superseded_by: never_exceeds_pi
      accepted:
        as: superseded
        at: "2026-08-14 09:11:02"
        by: janesmith@corp.org
        commit: 4f2a9c1e8b7d
  discoveries:
    - name: monotone_in_latitude
      statement: "d(p, q) >= d(p, r)"
      verdict: invalidated
      superseded_by: monotone_in_latitude_corrected
  historical:
    - name: uses_legacy_radius
      statement: "d(p, q, radius) == radius * d(p, q)"
      verdict: skipped
```

A discovery additionally declares the corrected claim in the live
`claims` list, carrying the counterexample that killed the old one as the
new one's supporting witness, and links the two so the record says what
replaced what.

The sections above are one arrangement, shown because it reads clearly.
**What a tool does here is its own business; that it retains the row
rather than dropping it is not.** A tool that keeps retired claims in the
same list with a status field, or in a sidecar, satisfies the rule
equally.

## Acceptance

`accepted` in its scalar form answers one question: was this
falsification a bug or a discovery. A tool that supports a wider set of
human decisions needs more room, because the decision that matters in CI
is usually not about a falsification at all but about a gap someone is
knowingly carrying.

The fuller form is an object:

```yaml
      accepted:
        as: risk
        at: "2026-08-14 09:11:02"
        by: janesmith@corp.org
        note: "loop shape is out of proof scope; monitored"
        form: 9f3ac2e1b4a0
```

- **`as`** names the decision: a falsification diagnosed as a discovery,
  empirical evidence judged sufficient, an unsettled claim whose risk
  someone is owning, and the two membership exits above.
- **`form`** binds the decision to the code it was made about. This is
  the part that earns the object: an acceptance is about a specific
  implementation, so when `form` moves, the acceptance goes stale and
  the decision is asked for again rather than silently carried onto code
  nobody agreed to.

An accepted gap stays visible. A tool that reclassifies an accepted
`unknown` claim should do so to a value that still says it was never
settled, so a lenient gate can proceed while a strict one still refuses
and neither can mistake it for a verified claim.

There is deliberately **no accepting a bug**. If the code is wrong, the
code changes; the recorded counterexample replays on every later
adjudication until the claim stops falsifying.

Both forms above are conformant. **What this specification requires is
the principle, not the shape**: an acceptance records who decided and
when, it is bound to the version of the code the decision was made
about, and it lapses when that version moves. A tool with one kind of
acceptance and a bare identity string satisfies that; so does a tool
with five kinds and an object. What neither may do is carry a human
decision forward onto code nobody agreed to.

## Composing freshness

`identity.form` answers "did this function's own body change." It does
not answer "did this function's behaviour change," because a function's
behaviour also depends on what it calls and what it reads.

A record may therefore carry the things it depends on, each with its own
identity, so a reader can tell a stale verdict from a current one without
re-running anything:

```yaml
  dependencies:
    - name: haversine
      kind: function
      key: geo.haversine
      form: 2b8e1d0c7a4f
      freshness: current
    - name: EARTH_RADIUS_KM
      kind: constant
      value: 6371.0
      freshness: current
```

A callee carries its `form`, so a change there invalidates the caller's
claims the same way a change to the caller's own body would. A module
constant carries its **value**, because a constant's change is invisible
to any form hash and would otherwise be a silent behavioural change
nothing detected.

The re-adjudication triggers that follow are: the function's own `form`
moved, its claim set moved, a dependency's `form` moved, or a dependency
constant's value moved.

Whether a tool records this, and under what name, is
implementation-defined. **The rule it exists to serve is not: a tool
that reports a claim as current must have a basis for saying so.** A
tool that only checks the function's own `form` should not report
freshness it has not established, which is the same rule as everywhere
else (`cdd.md`, "Say what you know, and no more").

A claim-level form, where one claim depends on another claim holding, is
described in `declared-schema.md`.
