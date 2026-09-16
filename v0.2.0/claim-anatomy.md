# Claim anatomy

Status: v0.2.0.

A claim about a function `f : A → B` is a four-part structure, optionally
carrying a premise, and an adjudication lifecycle status:

- **domain (D)**: which inputs the statement is asserted over, and how,
  e.g. 'over all inputs', 'inputs in a declared range', 'a sampled family'
  (see `declared-schema.md`'s `domain` field). The quantifier, whether the
  claim means "for all," "or only within this range", is nested
  inside the domain rather than a separate element alongside it: a
  domain's exact structure is defined by the grammar that reads it, not
  by this spec.
- **statement (φ)**: the equation or inequality itself, written in some
  expression grammar; a claim states which one (see `declared-schema.md`'s
  `grammar` field), since no single grammar is assumed.
- **tolerance (ε)**: how close counts as equal, for floating-point
  comparison; there's no spec-level default (see `declared-schema.md`'s
  `tolerance` field).
- **evidence route**: how the claim gets checked. Not a fixed set: `probe`
  (seeded random testing against the live function) and `derive` (proof
  against a lifted symbolic form) are the two well-known routes as of
  v0.1, and a tool using a different verification technique is expected to
  name its own rather than force-fit one of these (see `record-schema.md`,
  "Open for extension").
- **premise (optional)**: what the claim takes for granted. A claim that
  is only true under a side condition states that condition rather than
  being weakened or dropped; see "Premises" below.

A claim's lifecycle status is what checking it produces:

```mermaid
flowchart LR
    C[conjectured] --> V{checked}
    V -->|true on every trial| H["holds (n=...)"]
    V -->|derived proof| P["proven"]
    V -->|false on some trial| F["falsified<br/>counterexample kept"]
    V -->|settled nothing| U[unknown]
    V -->|blocked, with a reason| S[skipped]
    H -->|later check no longer supports it| I["invalidated<br/>previous verdict kept"]
    P -->|later check no longer supports it| I
    F -->|diagnosis: implementation was wrong| R["re-implement<br/>cdd.md loop, step 2"] --> V
    F -->|diagnosis: claim was wrong| K["retained as new knowledge<br/>cdd.md loop, step 5"]
```

`skipped` covers a claim adjudication was attempted on and **blocked**,
for a reason the tool can name: the code has effects that make algebraic
probing meaningless, the statement doesn't parse, or its route isn't one
this tool implements. `unknown` is the different case where nothing was
wrong and nothing was settled, including the state every claim starts in
before it has been checked at all. A falsified claim keeps its
counterexample permanently; it is never deleted, and it is never quietly
reverted back to `conjectured` once a diagnosis has been made (see
`cdd.md`'s loop, step 4, "Diagnose," for how that decision gets made).
See `evidence-ladder.md` for all eight verdicts and their relative
strength.

## Premises

Some claims are true only under a condition that isn't part of the
domain: a quadratic's root is real only when its discriminant is
nonnegative, a ratio is well defined only when its denominator isn't
zero, an identity holds only where the function is defined at all.

A premise states that condition as part of the claim. It is not the same
as narrowing the domain, and it is not the same as weakening the claim:

- A **domain** says which inputs the claim is about. It is a region.
- A **premise** says what is assumed true of those inputs. It may be a
  relation between parameters, which no box-shaped domain can express,
  or it may name another claim that must itself hold first.

A premise that names another claim makes the dependency explicit, and
caps the evidence: a claim resting on a premise that only `holds`
empirically cannot itself be `proven`, whatever route decided it. The
weakest link bounds the result, and a tool should record that it did.

**A premise is part of the claim.** It changes what is being asserted, so
a rendering that omits it does not denote the same claim. Whether a
grammar carries it inside the statement text or in a field beside it is
the grammar's decision (see `record-schema.md`, "Claim identity belongs
to the grammar"); what no tool may do is drop it and record a statement
that reads as unconditional.

## Domain: declared vs enforced

A domain condition in a claim (`alpha in (0, 1]`) is a statement about
mathematics: how far the statement is asserted to hold. Whether the code
actually rejects input outside that domain is a statement about
engineering, a different question, checked as its own ordinary claim, not
folded into the first one.

Both live in the ordinary claim shape from `declared-schema.md`, nothing
special-cased:

- The mathematical claim carries a `domain` field, and a checking tool
  samples inside it, so the claim is verified where it's actually claimed
  (in-domain adjudication), not against arbitrary values it was never
  intended for.
- A tool that supports domain-scoped claims may extract a second, ordinary
  claim from the same declaration, asserting that out-of-domain input is
  rejected (`raises(f(alpha))` or similar, see `declared-schema.md`, "Domain
  is a claim field"). Its verdict is `holds` if the code actually rejects
  bad input; what a silent acceptance counts as depends on the checking
  tool's own posture, see `evidence-ladder.md`, "Strict vs lenient
  checking."

### The region a verdict actually covered

A claim's `domain` says where it was asserted. What a checker actually
established can be narrower: a proof may close on a sub-region, and a
premise constrains the region further still. A verified record may carry
that narrower region separately (`condition`, see `verified-schema.md`),
so the difference between "what was claimed" and "what was shown" stays
visible rather than being collapsed into one field.

### Domain types

A domain is over some set, and which set matters to a checker: sampling
the integers is not sampling the reals, and neither is sampling the
complex plane. A grammar that types its domains should be able to name at
least the reals, the integers, the naturals, and the complex numbers, and
should be able to say whether a missing or absent value is part of the
input space, since that is a real and frequently overlooked case. The
exact spelling belongs to the grammar, not to this document.

## Claim families

Suggested claim families to check: symmetry/intertwining, order, compositional, aggregate/
conservation, asymptotic, structural, domain. Actual implementation is left to the conformant tool. It is expected that these will evolve and future work is expected to create libraries of appropriate reference claims for a given implementation.

### Suggested Claims

**Symmetry / intertwining**: `f(T x) = S(f x)`. The common instances are
even (`f(-x) = f(x)`), odd (`f(-x) = -f(x)`), permutation-invariant
(shuffling a sequence input leaves the result unchanged), scale-equivariant
(`c·f(x) = f(c·x)`), translation-equivariant (`f(x) + c = f(x + c)`). A
conformant tool is expected to check the common instances directly and
accept any other instance of this family (periodic, rotation-invariant,
general `T`/`S`) as a free-form statement over `f` and its parameters.

**Order**: monotone (single-scalar-argument functions) and bounded (for a
sequence argument: `min(x) <= f(x, ...) <= max(x)`). Arbitrary bounds and
contractions are free-form statements in the same shape.

**Compositional**: idempotent (`f(f(x)) = f(x)`, single-argument functions).
Involution and homomorphism statements are expressible as free-form ones.

**Aggregate / conservation** (totals, energy, norms preserved or dominated).
Expressible as a free-form statement where the statement itself computes
the aggregate (for example `sum(x) == sum(f(x))`).

**Asymptotic** (convergence, decay rate, error order, quantified over a
family of inputs rather than a point).

**Structural** (purity, determinism, complexity). Purity and determinism are
cheap to read directly off the code or observe by repeated calls.

**Domain** (domain validity, other parameter constraints)

**Partiality** (raising is behaviour too). That a function raises on a
given input is a checkable statement about it, and a claim quantified
over a region where the function raises is not satisfied by that raise:
a claim about a returned value is falsified when no value is returned.
Stating the raise as its own claim is how a partial function gets an
honest record rather than a quietly narrowed one.
