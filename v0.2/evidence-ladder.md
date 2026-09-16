# Evidence ladder

Status: v0.2.0.

Every claim's adjudicated verdict is bound to a specific kind of evidence, and the
verdict states which kind, so a reader of the knowledge base never has to guess how hard a
statement was checked. This is the well-known, documented set; `verdict`
itself is an open field, not a closed enum, see `record-schema.md`, "Open
for extension."

| verdict | meaning | strength | common today? |
|---|---|---|---|
| `proven` | established by a formal proof, algebraic derivation on a lifted symbolic form, an SMT solver, a proof assistant, often with a domain condition | strongest | no; building this kind of proof machinery into a tool is a substantial project in itself |
| `holds (n=...)` | held under seeded probing on n synthesized inputs | evidence, not proof | yes |
| `documented` | the function's own docstring or doc comment states it, taken at face value | unverified, not checked | yes, mainly for records where no source is available to probe at all e.g. 3rd party libraries |
| `declared` | the declared claim states it, no actual object documentation found | unverified, not checked | uncommon |
| `unknown` | adjudication ran and settled nothing either way, or the claim has not been adjudicated yet | no evidence, and none claimed | yes |
| `falsified (counterexample)` | definitively false; the counterexample is kept | definitive | yes |
| `invalidated` | was supported by a previous adjudication of this claim, and is not supported now | definitive, and a regression | yes, once records have history |
| `skipped` | adjudication was attempted and blocked, with an identifiable reason (effects present, unparseable statement, an unavailable route) | a strict check should treat it as a falsification (see below) | yes |

`documented` doesn't require any special markup in the docstring. A tool
producing this verdict is just reading the function's existing
documentation and citing whatever it plainly says as the basis for the
claim, without checking it independently.

Both `declared` from the declared schema and `documented` are weak
evidence classes: the docstring might be
wrong, out of date, or aspirational, but it's still the author's stated
intent and useful to include in the knowedge base for review. 
Sometimes this might be the only evidence available for code with no
accessible source, a compiled or third-party function, or an API say.

`documented` and `declared` are evidence classes for **intent-level
statements**, what a function is for, rather than for full claims with a
statement a checker could adjudicate. A tool that reads a docstring and
finds an adjudicable claim in it should check that claim and record the
verdict checking produced; `documented` is for what it takes at face
value because there is nothing to check it against.

## `unknown` and `skipped` are different answers

v0.1.0 had only `skipped`, and it carried two meanings that pull apart in
practice.

- **`skipped`** now means adjudication was attempted and **blocked**, for
  a reason the tool can name: the statement does not parse, the route is
  one this tool does not implement, the code has effects that make
  probing meaningless, a dependency is missing. Something is wrong with
  the claim, the code, or the tool's coverage, and the reason says which.
- **`unknown`** means nothing is wrong and nothing was settled. Either
  the claim has not been adjudicated yet, which is the state every
  declared claim starts in, or adjudication ran to completion and decided
  neither way. A prover that terminated without a proof or a
  counterexample produces `unknown`, not `skipped`: it was not blocked,
  it simply did not settle the question.

The distinction earns its place because the two have different remedies.
A `skipped` claim names something to fix. An `unknown` claim names the
limit of what was tried, and the honest response is to try harder, to
accept the gap deliberately, or to leave it.

A claim that has never been adjudicated carries **no `route`**: `route`
names the mechanism that actually decided, and nothing decided.

## `invalidated`: records have memory

A verified record is not a snapshot that gets overwritten. When a claim
that was previously `proven` or `holds` is re-adjudicated and no longer
is, that transition is itself a fact about the system, and a record that
simply overwrote the old verdict would make "was proven, now isn't"
indistinguishable from "was never supported".

`invalidated` is that transition made explicit. It is written when a
claim's previous verdict was supporting and its current adjudication is
not, and it persists until the claim is supported again. It is a
regression and an error state, never a silent downgrade to `unknown` or
a bare `falsified`.

A tool writing `invalidated` should carry the previous verdict alongside
it, so a reader can see what was lost. Where it carries it is
implementation-defined; `meta` is the obvious home.

## Rolling verdicts up

Every consumer that counts verdicts, a coverage report, a CI gate, a
dashboard, has to group them, and tools that each invent their own
grouping do not produce comparable numbers. The fold below is the
standard one:

| stance | verdicts |
|---|---|
| `supported` | `proven`, `holds` |
| `refuted` | `falsified`, `invalidated` |
| `blocked` | `skipped` |
| `undecided` | `unknown`, `documented`, `declared`, and any value outside this table |

`documented` and `declared` fold to `undecided` rather than `supported`
deliberately: they are the author's word, and nothing has checked them.

A verdict outside the documented set folds to `undecided` rather than
being malformed, the same treatment `record-schema.md`, "Open for
extension," gives it: unranked, not rejected.

Two things worth emphasising:

- **`holds` is not `proven`.** A held claim survived some number of
  synthesized trials. It is honest, reproducible (a seeded sampler gives
  the same trials every run) evidence, and it is the strongest evidence
  most tools can produce on their own today. `proven` sits above it in
  the design precisely because probing can miss a counterexample that a
  proof cannot. A rollup that buckets them together loses the one
  distinction the ladder exists to make, so a `supported` count is
  reported **beside** the individual counts, never instead of them.
- **`falsified` is not a bug report by itself.** A falsified claim means
  either the implementation is wrong (regenerate against the
  counterexample) or the claim was wrong (a discovery, retained as
  knowledge). See `cdd.md`'s loop, step 4, "Diagnose," and step 5,
  "Retain as new knowledge," for how that decision gets made and what
  happens next either way. Claim coverage counts refutation as
  adjudicated, not as failure. 
  
  The **CDD** loop should continue until all falsifications have been resolved either by manual acceptance that the falsification is acceptable (and becomes a known behaviour of the system with the counter-example as a new claim)  or all claims have been verified as *true* (at the given level of evidence).

## Strict vs lenient checking

Some verdicts depend on the nature of the problem and the stance the checking tool itself takes, not on
anything about the claim: whether a `skipped` claim counts as merely
unverifiable or as an outright falsification. This shows up wherever "the code
silently accepted something it probably shouldn't have". For example,
domain-enforcement claims (see `claim-anatomy.md`, "Domain: declared vs
enforced") offers a clear example: the function receives input outside of it's declared domain, `holds` means out-of-domain input was actually rejected as desired, but a silent acceptance is `skipped`,
in lenient mode, and `falsified` in strict mode.

Which mode is active is a setting the caller of a checking tool controls,
not a property of the claim or the record; the same claim set can be
checked leniently during development and strictly in CI.

Lenient treats skipped as a verified claim, strict treats it as falsified.

`unknown` follows the same rule as `skipped`: lenient proceeds, strict
refuses. `invalidated` does **not**: it is a regression from a verdict
this claim previously had, so it fails in both modes. A tool that let a
regression through leniently would be hiding exactly the transition the
verdict exists to surface.

For critical code, the expected behaviour should be strict checking for the following reason: if a claim cannot be verified, choosing to accept it anyway becomes an explicit decision point. The person making that decision may have valid reasons based on information or reasoning outside the scope of the code, but the decision to proceed despite an unverified (i.e. falsified) claim should be deliberate rather than implicit.

That decision point is what an acceptance records, see
`verified-schema.md`, "Acceptance." An accepted `unknown` claim
reclassifies to a verdict naming both facts, that it was never settled
and that a human took it on anyway, rather than quietly counting as
verified.
