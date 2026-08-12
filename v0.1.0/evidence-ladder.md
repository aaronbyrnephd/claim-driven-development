# Evidence ladder

Status: v0.1.0.

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
| `falsified (counterexample)` | definitively false; the counterexample is kept | definitive | yes |
| `skipped` | not evaluable (effects present, unparseable statement, or an unavailable route) | a strict check should treat it as a falsification (see below) | yes |

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

Two things worth emphasising:

- **`holds` is not `proven`.** A held claim survived some number of
  synthesized trials. It is honest, reproducible (a seeded sampler gives
  the same trials every run) evidence, and it is the strongest evidence
  most tools can produce on their own today. `proven` sits above it in
  the design precisely because probing can miss a counterexample that a
  proof cannot.
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

For critical code, the expected behaviour should be strict checking for the following reason: if a claim cannot be verified, choosing to accept it anyway becomes an explicit decision point. The person making that decision may have valid reasons based on information or reasoning outside the scope of the code, but the decision to proceed despite an unverified (i.e. falsified) claim should be deliberate rather than implicit.