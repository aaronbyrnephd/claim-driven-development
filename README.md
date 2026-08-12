# Claim-Driven Development

A specification for a development workflow where the unit of work is the
**claim**: a checkable promise about what a function does. Not the test, and not solely
the source code, but the reasoning behind the intent of the code and its region of validity.
The level of trust to be put on the correctness of code is proportional to the weight of evidence behind it.

## The problem this is for

Writing code used to be the hard part, and reviewing it was comparatively
easy: a human read a diff, recognized the pattern, and trusted it because
(a) they understood it fully or (b) moslty understood it but knew and trusted the author.
This has changed substantially over the past few years with the emergence of AI.

A model can now readily write a
correct-looking package (with a suite of test files and documentation), in
less time than it takes to write a single correct function manually.
Generating plausible and highly tested code is no longer
the hard part. In fact the ease of generation may have already become a new problem,
because how do you end up trusting what is generated?

Reviewing the output of code development to a high standard has become the bottleneck and a likely point of failure.
A human now has to decide whether to trust code
they are ultimately less familiar with and where they have limited ability to gauge the expertise of the code generating model.

Increasingly code will not be read line by line, and "passing
tests" is weak evidence when the tests themselves were written by the same
process that wrote the code, with the same biases and blind spots.

The real thing that
used to make code trustworthy: a person with sufficient expertise understanding and verifying it (usually by writing unit tests to express their intent),
doesn't scale to agentic code development practices.

With the rise of generative code development workflows, we're faced with an age old problem, namely that language is imprecise for defining intent and instead CDD reaches for something more mathematical to solve this.

CDD is aimed to reduce these exact gaps: it separates out
*stating what must be true about a function* from the actual *writing of the function itself*, so
the thing a human reviews is a relatively short list of properties (the knowledge base), not solely an
implementation. The part that gets checked mechanically is whether the
implementation has actually implemented those properties correctly, via running it in a CI pipeline and iterative development loop, not by unwarranted trust.

## The loop

1. **State claims first.** Written by a human, or proposed by an agent for
   a human to look over. A claim starts as a conjecture either way.
2. **Implement, or generate.** The claim set plus the intent describes what
   the function must satisfy, informing an implementation the way a test
   suite already does in TDD, whether a human writes the function by hand
   or a model generates it. CDD doesn't assume AI wrote the code, and it
   doesn't replace an existing test suite: ordinary tests still pin
   concrete regressions, claims add implementation-independent laws on
   top.
3. **Verify.** Each claim is checked against the real implementation:
   `holds (n=...)` under seeded testing, or `falsified` with a
   counterexample kept permanently.
4. **Diagnose.** A falsified claim has exactly two possible causes: either
   the implementation is wrong, and the counterexample drives another pass
   through step 2, or the claim was wrong, resulting in a discovery about the problem
   rather than a defect in the code.
5. **Retain as new knowledge.** A claim diagnosed as a genuine discovery is
   kept as new recorded knowledge.
6. **Accept.** Once every claim has been adjudicated, (proved or falsified),
   any implementation that re-verifies the same claim set is trusted the
   same as the one before it; a later regeneration re-enters the loop at
   step 2, and the counterexamples already on record make it also a regression
   check.

Full definition, including who typically does each step and a diagram of
the cycle: [`v0.1.0/cdd.md`](v0.1.0/cdd.md).

## Why not an existing verification approach

Every individual mechanism CDD makes use of has precedent; the combination and the
role it plays are what's different.

- **Design by Contract / refinement types (`Eiffel`, `deal`, `icontract`,
  `Liquid Haskell`, `Dafny`)**: contracts check the inputs that happen to occur
  at runtime, or require the code to be written in a way a prover accepts.
  
  > **CD**D claims are verified across the declared domain before acceptance, not
  wherever the code happens to be called from, and don't require rewriting
  the implementation to satisfy a proof assistant.

- **Property-based testing (`QuickCheck`, `Hypothesis`)**: 
  > **CDD** builds directly upon property based testing, but treats the property as the
  primary artifact of development instead of a testing technique applied
  after the fact.
  
  The claim set is normally stated *before* the code (and
  informs the generation prompt if AI is used), not a check on code that already exists,
  though claims can just as well be retrofitted onto code that predates
  them; **CDD** doesn't require greenfield. A claim's `statement` (see
  `v0.1.0/declared-schema.md`) can be written in whatever expression grammar
  a tool defines, including one that's a superset of an existing
  property-testing library's own property language, so adopting **CDD**
  doesn't require throwing away properties already written for QuickCheck
  or Hypothesis.

- **Invariant inference (`Daikon`, `QuickSpec`)**: these discover likely
  invariants from execution traces.
  
  > **CDD's** claims are stated with intent,
  human- or model-proposed, then adjudicated; the
  verifier never trusts its own inference, and a proposer never adjudicates
  its own claims.

- **BDD (`Cucumber/Gherkin`)**: scenarios are still worked examples, concrete
  inputs and outputs. 
  >Claims are statements, quantified over a domain, not example cases.

- **Data contracts (`dbt tests`, `Great Expectations`)**: these make claims about a
  dataset in its current state.
  
  >*CDD's* claim anatomy generalizes this
  (domain, statement, tolerance, evidence route) to arbitrary functions,
  not only tabular data.

- **Content-addressed code (`Unison`)**: identity by hash instead of by name
  or file location, within one language.
  
  >**CDD** borrows this identity move
  (rename-invariant `form` hash) but binds it to *evidence*, not only to
  storage: two implementations with the same verified claim set can be treated
  as the same function even if their hashes/implementations differ *(assuming sufficient claim coverage)*.

What's actually new is the composition: claims can be stated before generation/implementation, as
the compilation contract for what's **intended**, bound to a rename-invariant identity, explicitly
designed for a world where the implementation will change rapidly and easily, but where oversight becomes increasingly difficult. Nobody needed that combination until code files stopped
having a single, known, trusted author.

## What's in this repo

- [`v0.1.0/cdd.md`](v0.1.0/cdd.md): the workflow, the evidence ladder.
- [`v0.1.0/claim-anatomy.md`](v0.1.0/claim-anatomy.md): the claim
  structure (domain, statement, tolerance, evidence route).
- [`v0.1.0/record-schema.md`](v0.1.0/record-schema.md): how the declared
  and verified shapes relate. This is the actual interoperability
  contract: any tool that reads and writes these shapes can work with any
  other tool that does, without depending on its code.
- [`v0.1.0/declared-schema.md`](v0.1.0/declared-schema.md): the YAML shape
  for proposing a claim set, before anything is checked.
- [`v0.1.0/verified-schema.md`](v0.1.0/verified-schema.md): the YAML shape
  a conformant tool writes after checking one.
- [`v0.1.0/evidence-ladder.md`](v0.1.0/evidence-ladder.md): the verdict
  vocabulary and what each verdict means.
- [`CHANGELOG.md`](CHANGELOG.md): spec versions, on their own cadence,
  independent of any implementation's release schedule.

## Implementations

Intentionally none are bundled with this spec: there is no code in this repo,
and no implementation is privileged by these documents. Any tool, in any
language, that reads and writes the record shape defined here conforms.

The first tool (in development) against this spec is **mathema**, a downstream python
project. Nothing in this spec depends on it or assumes
its choices.

## License

The documents in this repo are CC-BY-4.0: reuse and adapt freely, with
attribution. That's deliberately a different kind of license from whatever
an implementation might use for its code (a software license like MIT or
Apache-2.0); the spec for CDD is meant to be able to
move independently from any implementations.
