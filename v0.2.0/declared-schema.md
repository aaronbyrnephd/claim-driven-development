# Declared shape

Status: v0.2.0. Part of the record schema; see `record-schema.md` for how
this relates to the verified shape.

One or more claims about one function, keyed by dotted name
(`module.qualname`). This is what a human writes by hand, or what a model
proposes for human approval, before anything runs.

### This YAML is the interchange shape, not the only way to author a claim

What follows is the shape two different tools conform to, not necessarily
what a human actually types. Writing the YAML by hand is certainly one route to authorship,
but not the only legitimate one: a decorator on the function itself, a
structured section in its docstring, a refinement embedded in its type
signature, a declaration alongside its prototype in a header file, could
all serve the same purpose in whatever language and toolchain fits. A tool
is free to let claims be authored in whatever way is native to it, as long as it
can produce and consume this shape for interop with anything else. What a
human writes and what gets exchanged between tools don't have to be the
same artifact.

```yaml
geo.gc_distance:
  intent: Length of the great-circle arc between two points on the unit sphere
  signature: "(real, real, real, real) -> real"
  grammar: implementation-defined
  claims:
    - name: symmetric
      statement: "d(p, q) == d(q, p)"
      route: derive
    - name: nonnegative
      statement: "d(p, q) >= 0"
      route: probe
      domain:
        p: [-1.5707963, 1.5707963]
      meta:
        concepts: [metric-space]
```

Fields per claim:

| field | required | meaning |
|---|---|---|
| `name` | yes | short identifier, unique within the function's claim set |
| `statement` | yes | the equation or inequality, over `f` and the function's parameters, written in whatever expression grammar `grammar` names |
| `route` | no | how the claim gets checked; unstated leaves the choice to the tool, strongest-first in the reference behaviour, see "Route is advice" below and `record-schema.md`, "Open for extension" |
| `domain` | no, default `(-inf, inf)` | per-parameter bounds this claim is asserted over, including the quantifier it's asserted under; see "Domain is a claim field" below |
| `tolerance` | no, recommended when `statement` does a near-equality float comparison | how close counts as equal; no spec-level default, see "Tolerance has no default, and is left to the implementation" below |
| `premise` | no | what the claim assumes, when it is true only under a side condition; see "Premises" below and `claim-anatomy.md` |
| `dependencies` | no | what this claim requires to hold first; see "Claim dependencies" below |
| `authored` | no | what the author knows about the claim's origin (`by`, `at`, `ref`), merged forward by the checking tool; see below |
| `meta` | no | a namespaced bucket for anything not defined by this spec; see `record-schema.md`, "The `meta` extension point" |

`statement` is the one field name shared between the declared and verified
shapes: it's the same content before and after checking, unlike `intent`,
which means something different on each side (see `record-schema.md`,
"Where `intent` comes from"). A conformant tool carries `statement`,
`route`, and `tolerance` forward into the verified record unchanged, so
that record is self-contained, readable, and checkable against, without
having to go back to whatever declared it. See `verified-schema.md`.

A declared claim **may** carry an `authored` object holding what its
author knows and a checker cannot observe: who wrote it (`by`), when
(`at`), and a reference if the authoring format has one. The checking
tool adds what it does observe and carries the rest forward, and a
stated author is never overwritten. The verified record is where this
resolves; see `verified-schema.md`, "`authored`: tracing a claim back to
where it came from."

### Premises

A claim that is only true under a side condition states the condition
rather than being narrowed or dropped. The condition may be a relation
between parameters, which a per-parameter `domain` cannot express, or it
may name another claim that has to hold first.

A grammar may carry the premise inside `statement` (`assuming k != 0,
f(x, k) == x/k`) or in a separate `premise` field; what it may not do is
drop it, or record a statement that reads as unconditional when the claim
is not.

Which of those two a grammar chooses, and how it renders a claim
canonically, is the grammar's decision to publish, not this document's
to make: see `record-schema.md`, "Claim identity belongs to the
grammar." What matters here is only that the premise is not lost, since
a claim recorded without it asserts something the author did not.

### Claim dependencies

A claim may require something else to be true before it means anything.
The record of that is one typed entry, so a new kind of prerequisite
slots in without changing the shape:

```yaml
    - name: recurrence_closed_form
      statement: "f(n) == f(n-1) + f(n-2)"
      dependencies:
        - kind: claim
          ref: base_case
          requires: proven
        - kind: function-form
          ref: geo.haversine
```

- **`kind`** names what sort of thing is depended on. `claim` and
  `function-form` are the two well-known kinds; a tool with others (a
  named test passing, a data-availability check) names its own, the same
  way `route` and `verdict` are open.
- **`ref`** identifies it, in whatever way that kind is identified.
- **`requires`** is the state the dependency has to reach, when the kind
  has states. Omitted means "exists and is current."

A dependency caps the evidence. A claim whose prerequisite only `holds`
cannot itself be `proven`, whatever route decided it: the weakest link
bounds the result, and a tool should record that the cap applied rather
than reporting the stronger verdict. A prerequisite that cannot be
resolved at all leaves the dependent claim `unknown`, not `skipped`,
since nothing was wrong with the claim itself. A cycle among claim
dependencies is an error, not a verdict.

Function-level dependencies, what a function calls and reads, are a
different thing living on the verified record; see `verified-schema.md`,
"Composing freshness."

### Domain is a claim field, not a separate mechanism

A declared domain (`alpha in (0, 1]`) is what `claim-anatomy.md`'s
four-tuple calls the domain quantifier scope: which inputs the statement is
actually asserted over. Rather than inventing a parallel system for it,
it's just another field on the same claim, `domain`, a mapping from
parameter name to its bounds. `domain` is optional, not required: a claim
with none stated is asserted without restriction, `(-inf, inf)` for every
parameter, regardless of the parameter's actual type, and a claim with one
is asserted only inside it. A checking tool samples accordingly, so the claim is verified where it's actually claimed.

Leaving `domain` unstated isn't a smaller claim than stating one, it's a
larger one: it asserts the statement holds everywhere, and a checking tool
is expected to actually try inputs outside whatever narrower range the
author may have had in mind but never wrote down. A claim that turns out
to only hold in some narrower range surfaces that gap as a `falsified`
verdict rather than hiding it, the same as any other unclaimed behavior
(see `cdd.md`, "Claim coverage, in addition to code coverage").

Whether the code *enforces* that boundary, rejecting input outside it, is
a different question from whether the statement holds inside it, but it
doesn't need a different kind of answer. A tool that supports
domain-scoped claims may, when it checks one, extract a second, perfectly
ordinary claim from the same domain declaration: same shape, same
`name`/`statement`/`verdict`/`n`/`counterexample` as anything else in the
record, just produced by the tool rather than hand-authored, reporting
whether out-of-domain input is actually rejected. There is no separate
"enforcement verdict" concept anywhere in this spec; it's a claim like any
other.

Stating that a claim is about error behavior rather than a value is a
matter for the grammar, not this spec, the same as any other statement,
but a grammar needs some way to say it, or "rejects out-of-domain input"
has nothing to be expressed as. A common shape for this, illustrative, not
mandated:

```
statement: "raises(f(alpha))"              # asserts the call raises some error
statement: "raises(f(alpha), ValueError)"  # asserts it raises specifically this one
```

`raises(...)` here is just a predicate like any other in the grammar that
defines it; nothing about this spec treats error behavior differently from
a value comparison. A tool extracting the enforcement claim above would
produce a `statement` in exactly this shape, in whatever `grammar` the
rest of the record already uses.

### Tolerance has no default, and is left to the implementation

Unlike `route`, there's no agreed-upon standard to default to here, across
languages or across floating-point representations. C++'s gtest compares
floating-point equality by ULP count, not an epsilon at all; Catch2
defaults to a relative tolerance scaled off float's own epsilon; Rust's
approx crate defaults to f64::EPSILON itself; Python's math.isclose
and numpy.isclose disagree with each other (`1e-9` relative vs `1e-5`
relative plus `1e-8` absolute); Haskell has no dominant convention at all;
COBOL's native decimal types mostly don't have this problem in the first
place.

So this spec sets none. `tolerance` is recommended whenever a claim's
`statement` does a near-equality comparison sensitive to floating-point
rounding, and absent otherwise; how close counts as equal genuinely
depends on the problem's own required accuracy, not on anything this spec
can decide in advance. A conformant tool is expected to set its own
sensible default for claims that don't state one explicitly, some default
is friendlier than forcing every claim to state a value it doesn't
necessarily need, but that default is the tool's own convention to
document, not this spec's, and `tolerance` on an individual claim should
always override it.

### Route is advice; an unstated route leaves the choice to the tool

Unlike `grammar`, which has no default because guessing the wrong dialect
breaks parsing outright, a claim that states no `route` leaves the
choice of evidence to the tool. The reference behaviour is to try the
strongest evidence available and fall back: derive where the function
lifts to a symbolic form, probe otherwise. `probe`, some form of live
sampling against the real function, is the floor: close to a universal
capability, nearly any tool can do it, so a tool with nothing stronger
lands there, the same default a property-based testing tool makes when
no strategy is given. Either way the recorded `route` names what
actually decided, never the instruction (see `record-schema.md`,
"Subtypes: the colon convention").

`claim-anatomy.md`'s claim families (symmetry, order, and so on) are a way
of thinking about what to check, not a field in this shape: this schema
doesn't ask a claim to be classified into a taxonomy to be valid. Anyone
who does want to tag a claim's category can do so under `meta`, the same
way as any other extension, without this spec prescribing the vocabulary
for it.

### Grammar
`grammar` can be specified as a default by the tool implementing it, at the function level for every claim sepcified for that function, (alongside `intent`/`signature`), or an individual claim can set its own (allowing for mixing of implementations):

| field | required | meaning |
|---|---|---|
| `grammar` | yes (but can inherit directly from tool)| names the expression language `statement` strings are written in. May also be set per-claim to override the defaults. |

**`statement` has no single grammar, and none is assumed by default.** A
bare string like `"f(-x) == -f(x)"` is only checkable by a tool that knows
the dialect it's written in, and different tools will reasonably use
different ones (a subset of their own language, SMT-LIB, whatever fits).
`grammar` names that dialect so a record doesn't ask a reader to guess.
Defining, documenting, and parsing any given grammar is entirely up to the
implementation that uses it; this document does not specify or enumerate
any. A record with no `grammar` field in the declared shape inherits it from
the conformant tool itself and is recorded in the verified shape output record.

The same claim, "f is odd," stated three genuinely different ways, none of
them privileged over the others:

```yaml
# a tool built around Python expressions over f and its parameters
claims:
  - name: odd
    statement: "f(-x) == -f(x)"
    grammar: python-expression
    route: probe

# a tool built around Hypothesis-style property testing
claims:
  - name: odd
    statement: "assert f(-x) == -f(x)"
    grammar: python-hypothesis-assertion
    route: probe

# a tool built around an SMT solver
claims:
  - name: odd
    statement: "(assert (forall ((x Real)) (= (f (- x)) (- (f x)))))"
    grammar: smt-lib2
    route: derive
```

The first two look similar because both happen to be Python; nothing about
the shape requires that. The SMT-LIB one is quantified explicitly and typed
because that grammar requires it, and its route is `derive`, not `probe`,
because an SMT solver is proving the claim, not sampling it. A record
mixing claims from more than one of these under the same function is legal
as long as each claim states its own `grammar`.

Below is a wider spread of illustrative grammars for a claim needing a
`tolerance`, using the Pythagorean identity: $\sin^2(x) + \cos^2(x) = 1$.

The `tolerance` field stays a
structured input in every case, instead of a number typed twice; a grammar
that needs it inside the statement itself references it with a `{tolerance}`
placeholder. The verification tool should  substitute the claim's own `tolerance` value in before parsing, so
that the field and the statement can never disagree with each other.

Unlike the `f()` calling examples above, the below are deliberately self-contained math for illustrative reasons. Usually
`statement` would call the actual function the record's key names, `f(x)` or
`gc_distance(p, q)`, the way the other examples in this document do (but it is grammer specific):

```yaml
# a tool built around Rust, using the `approx` crate
claims:
  - name: pythagorean_identity
    statement: "approx::relative_eq!(x.sin().powi(2) + x.cos().powi(2), 1.0, max_relative = {tolerance})"
    grammar: rust-approx
    route: probe
    tolerance: 1e-9

# a tool built around C++, using Catch2's Approx
claims:
  - name: pythagorean_identity
    statement: "std::sin(x)*std::sin(x) + std::cos(x)*std::cos(x) == Approx(1.0).epsilon({tolerance})"
    grammar: cpp-catch2
    route: probe
    tolerance: 1e-9

# a tool built around an SMT solver
claims:
  - name: pythagorean_identity
    statement: "(assert (forall ((x Real)) (<= (abs (- (+ (* (sin x) (sin x)) (* (cos x) (cos x))) 1)) {tolerance})))"
    grammar: smt-lib2
    route: derive
    tolerance: 1e-9

# a tool built around Clojure
claims:
  - name: pythagorean_identity
    statement: "(< (Math/abs (- (+ (Math/pow (Math/sin x) 2) (Math/pow (Math/cos x) 2)) 1)) {tolerance})"
    grammar: clojure-expression
    route: probe
    tolerance: 1e-9

# a tool built around Scala, using Scalactic's tolerance operator
claims:
  - name: pythagorean_identity
    statement: "math.pow(math.sin(x), 2) + math.pow(math.cos(x), 2) === 1.0 +- {tolerance}"
    grammar: scala-scalactic
    route: probe
    tolerance: 1e-9

# a tool built around R, using base R's all.equal
claims:
  - name: pythagorean_identity
    statement: "isTRUE(all.equal(sin(x)^2 + cos(x)^2, 1, tolerance = {tolerance}))"
    grammar: r-base
    route: probe
    tolerance: 1e-9

# a tool built around Julia, using its built-in isapprox
claims:
  - name: pythagorean_identity
    statement: "isapprox(sin(x)^2 + cos(x)^2, 1; atol={tolerance})"
    grammar: julia-isapprox
    route: probe
    tolerance: 1e-9
```

None of these grammars has a conformant tool as of today. For the `probe` route,
the gap is small: each ecosystem's own testing library, `approx`, Catch2,
Scalactic, `testthat`, base R, `isapprox`, already checks near-equality;
a conformant tool mostly just wires that up to this record shape.

The `derive` route is a different, *much harder* problem.

The declared shape's `signature`, defined above, is a free-text description a
human or model writes for what they intend the signature to be before any code is written let alone checked; it is optional and not machine-verified. This is in contrast with the verified shape's `signature` (see [`verified-schema.md`](v0.1.0/verified-schema.md)),
which comes from the checking tool's own introspection.
