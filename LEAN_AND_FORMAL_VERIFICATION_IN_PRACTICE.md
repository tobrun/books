# Lean and Formal Verification in Practice

## A Hands-On Guide to Building a Trust Layer for an AI Software Factory

An agent changes a codec.
The tests pass.
Another agent changes the tests.
They still pass.
A third agent explains that the implementation is correct.

What, exactly, has the team learned?

You need evidence whose meaning survives changes to the implementation, the agent, and the test harness.
A formal specification gives the team a precise claim.
A proof checker can establish that a model or implementation satisfies that claim.
A protected release process can bind the evidence to the software that actually ships.

The difficult part is keeping those three things connected.

> Trust a checked claim about an identified artifact under explicit assumptions.

This book builds that operating model.
Lean 4 holds executable reference models and machine-checked theorems.
Differential testing connects those models to production code in Kotlin, TypeScript, Python, and other languages.
TLA+ describes orchestration and distributed protocols.
Cedar gives agent permissions a precise policy semantics.
Verus, Kani, and Aeneas provide different routes to evidence about Rust.

An AI software factory is the running example: agents plan work, edit repositories, call tools, and submit changes through a controlled merge process.
The platform team owns the specifications, execution boundaries, checkers, and evidence.
Agents can produce implementations and proofs inside that structure.

Three pilots make the discussion concrete:

- Agent permissions that preserve a reviewed authority boundary.
- Job leases whose stale workers cannot commit results.
- Quadkey and tile-address functions with exact integer semantics.

The purpose is to make valuable changes easier to trust.
That requires judgment about which claims matter, practical bridges to ordinary code, and an honest account of what the evidence leaves unresolved.

**How to read the examples.**
The proof examples use only `Std` and are collected into a complete example module in the appendix.
The executable adapter uses Lean's bundled JSON library.
The TLA+ lease model is a small executable model with a configuration.
The Python and JavaScript differential-testing example is a complete local lab.
Other contracts, deployment layouts, and integration fragments are labeled as proposed interfaces or sketches.
They require implementation in your platform.

**Version and evidence scope.**
This book is based on the supplied November 2026 research report and primary sources checked while preparing the manuscript.
Fast-changing benchmark results and vendor case studies remain attributed to their authors.
The source chapter distinguishes corroborated results, recommended architecture, and research leads.
Lean 4.36.0 is the pinned version used to check the book's Lean examples.
It is a reproducible teaching baseline, not a promise that a frozen compiler remains safe indefinitely.

---

## Contents

1. [Choose the claim you need to trust](#1-choose-the-claim-you-need-to-trust)
2. [Read the evidence without buying the hype](#2-read-the-evidence-without-buying-the-hype)
3. [Choose tools by the behavior they must describe](#3-choose-tools-by-the-behavior-they-must-describe)
4. [Turn requirements into approved specifications](#4-turn-requirements-into-approved-specifications)
5. [Learn enough Lean to review a theorem](#5-learn-enough-lean-to-review-a-theorem)
6. [Build an executable reference model](#6-build-an-executable-reference-model)
7. [Connect the model to production code](#7-connect-the-model-to-production-code)
8. [Build a proof gate with a protected meaning](#8-build-a-proof-gate-with-a-protected-meaning)
9. [Treat the prover as untrusted execution](#9-treat-the-prover-as-untrusted-execution)
10. [Make evidence part of every change](#10-make-evidence-part-of-every-change)
11. [Give agent permissions a precise boundary](#11-give-agent-permissions-a-precise-boundary)
12. [Model leases and retries before implementation](#12-model-leases-and-retries-before-implementation)
13. [Check that production still follows the model](#13-check-that-production-still-follows-the-model)
14. [Verify geospatial kernels over exact data](#14-verify-geospatial-kernels-over-exact-data)
15. [Make floating-point error an explicit obligation](#15-make-floating-point-error-an-explicit-obligation)
16. [Choose a verification route for Rust](#16-choose-a-verification-route-for-rust)
17. [Operate proof-producing agents](#17-operate-proof-producing-agents)
18. [Measure assurance and proof economics](#18-measure-assurance-and-proof-economics)
19. [Roll out through three useful pilots](#19-roll-out-through-three-useful-pilots)
20. [Recover when the evidence breaks](#20-recover-when-the-evidence-breaks)
21. [Appendix: runnable examples and reusable contracts](#21-appendix-runnable-examples-and-reusable-contracts)
22. [Sources and verification notes](#22-sources-and-verification-notes)

---

## 1. Choose the claim you need to trust

Begin with a failure you want to prevent.

A worker holds a lease, runs a job, loses connectivity, and returns later.
The broker has already given the job to another worker.
Both workers try to publish a result.

“The scheduler is correct” is too large a claim to guide an implementation.
“A result from an obsolete lease cannot replace the current result” is useful.
It names an action, a state distinction, and an observable outcome.

The same move works for a codec.
“The encoder is correct” becomes “decoding every valid encoded geometry returns the original coordinates.”
For permissions, “the agent is safe” becomes “this policy change introduces no newly allowed request within the approved request domain.”

### 1.1 Separate four questions

Every verification project needs answers to these questions:

| Question | Artifact that answers it |
|---|---|
| What behavior do we intend? | Reviewed requirement, examples, and formal specification |
| What does the model guarantee? | Theorem or checked model property |
| How does shipping code relate to the model? | Refinement proof, differential tests, or another explicit bridge |
| What makes the checking process trustworthy? | Pinned dependencies, checker policy, isolation, and artifact-bound evidence |

A proof of the second answer does not supply the other three.
This distinction should appear in the pull request, the release record, and the leadership explanation.

### 1.2 State the assurance chain

For an executable reference model, the chain is:

```text
Reviewed intent
     |
     v
Approved formal statement and model definitions
     |
     v
Checked theorem about the model
     |
     v
Differential comparisons with a production implementation
     |
     v
Release evidence for the exact checked artifacts
```

The fourth step is usually empirical.
Testing one million inputs establishes agreement on those inputs.
It does not establish agreement on every input.

If you prove that the implementation refines the model, that step becomes a universal claim within the proof's semantics and assumptions.
If you test the implementation against the model, describe it as a tested bridge.
The architecture is still valuable, but the guarantee is different.

### 1.3 Write the first assurance record

This proposed record belongs beside the task:

```yaml
claim_id: lease-stale-result-rejected
owner: agent-platform
intent:
  A worker whose lease epoch is obsolete cannot publish a job result.
domain:
  - durable broker state
  - atomic validation and result commit
  - monotonically increasing epochs without wraparound
formal_target:
  language: TLA+
  invariant: NoStaleCommit
implementation_bridge:
  - transactional database constraint or conditional update
  - deterministic interleaving tests
  - runtime monitoring of accepted commits
excluded_claims:
  - exactly-once external side effects
  - fairness of scheduling
  - availability during database failure
review:
  required: specification owner
```

Notice the exclusions.
Rejecting stale results does not stop a stale worker from sending an email before it submits its result.
External side effects need their own authority boundary and idempotency design.

### 1.4 Keep the claim smaller than the system

Small claims compose when their interfaces and assumptions compose.

A verified decoder can supply a parser with bounded integers.
The parser can supply a policy engine with well-formed requests.
The policy engine can supply the broker with an authorization decision.
The broker can enforce that decision on the action it actually performs.

Each seam introduces obligations.
Was the same resource identifier authorized and used?
Does the parser normalize identifiers consistently?
Can authority change between the decision and the action?
Does an error become an accidental allow?

A collection of correct components does not answer these questions automatically.
Write the seam into the specification.

### Lab: choose one valuable property

Take a recent defect or credible failure scenario.
Write the input domain, the forbidden outcome, the assumptions, and the observable evidence.
Reduce the claim until one engineer can review its meaning in a sitting.

**Done when:** the owner can explain what a passing check establishes and name an important failure that remains outside its scope.

---

## 2. Read the evidence without buying the hype

The supplied research argues that AI has changed the economics of proof production.
There is evidence for a substantial capability shift.
There is much less evidence for a universal return on investment across ordinary software teams.

Use the distinction to choose experiments.

### 2.1 What the large case studies establish

Anthropic and Theorem reported proofs for 239 OSS-Fuzz entry points across 123 libraries in October 2026.
The work used Rocq proofs over modeled RISC-V binaries and Linux process semantics.
The no-crash result covered inputs up to 64 MB, single-threaded static binaries, a fixed memory budget, and other stated conditions.
It is a scoped security result, not a claim that every feature in each library is correct.[^anthropic]

The important engineering development is that agents produced very large checked proof artifacts.
Human reviewers did not need to inspect every proof step.
They still needed to trust the target specification, machine semantics, and checker.
The work also found bugs while attempting proofs.[^anthropic]

Cedar supplies a different kind of evidence.
AWS describes a development process with Lean models, proved properties, and differential random testing against its Rust implementation.
Its release rule requires the models, proofs, and differential tests to be current.[^cedar-aws]

These cases support two approaches:

- Prove broad properties directly about a specified executable semantics.
- Prove a compact reference model and test an optimized implementation against it.

Neither removes the need to own the specification.

### 2.2 What benchmark percentages conceal

The 2025 vericoding benchmark contains 12,504 specifications across Dafny, Verus, and Lean.
Its abstract reports approximate off-the-shelf success rates of 82%, 44%, and 27%, respectively.[^vericoding]

VERINA reports 189 Lean tasks.
Its best single-trial result combines 72.6% code correctness, 52.3% specification soundness and completeness, and 4.9% proof success.[^verina]

Those results measure different things.
A code task checked with tests, a specification judged for adequacy, and a completed formal proof are different outcomes.
Changing the attempt budget or allowing tool feedback changes the experiment again.

The 2026 CLEVER audit makes the denominator problem concrete.
Its authors flagged arguably erroneous reference specifications in 80 of 161 entries.
They report up to 96.2% end-to-end success over entries with sound premises.[^clever-audit]

That percentage describes a selected subset under a particular agentic setup.
It should not become “agents verify 96% of software.”

### 2.3 Keep a claim ledger

For any result you use in a decision, record:

| Field | Why it matters |
|---|---|
| Source and version | A revised paper may change the protocol or result |
| Unit of work | Function, theorem, repository, binary, or entry point |
| Starting material | Natural language, approved formal spec, or existing code |
| Attempts and budget | Single shot and a long repair loop are different products |
| Accepted assumptions | Preconditions can exclude difficult behavior |
| Integrity checks | Checker success can be gamed |
| Implementation bridge | A proof about a model may leave production behavior empirical |
| Failure accounting | Timeouts, invalid targets, and weak specifications need separate counts |

The ledger is useful during model upgrades.
It stops a cheaper worker from appearing better merely because it narrows the input domain.

### 2.4 Treat trends as reasons to build a pilot

Kleppmann's December 2025 essay predicts that AI will make formal verification more practical and useful for AI-generated software.[^kleppmann]
De Moura's February 2026 essay argues for a platform that requires generated work to be verified.[^demoura-factory]

These are informed architectural arguments.
They are not controlled studies showing that your team can remove implementation review next quarter.

Start with a property you can state, an implementation you can observe, and a baseline cost.
Measure what changes after adding the verification workflow.
The durable question is whether the team can trust more consequential change with clearer evidence.

### 2.5 Do not turn a literature search into an absence theorem

The report identifies geospatial kernels as a promising niche and says it did not find verified work for several tile and encoding algorithms.
That is a search result under a particular search strategy.

Describe a new artifact by the properties it proves and the implementation it exercises.
Calling it the first verified artifact requires a separate, reproducible literature and ecosystem review.
Publishing a useful model does not require that superlative.

**Done when:** every external number in a proposal has a source, a denominator, a budget description where available, and a statement of its limits.

---

## 3. Choose tools by the behavior they must describe

Lean can express deep mathematical and software claims.
That does not make it the best shipping language for every component.

The platform needs a small collection of compatible assurance methods.
Choose them from the behavior and the implementation boundary.

### 3.1 Use a practical decision table

| Problem | First route to try | Evidence it can supply | Remaining work |
|---|---|---|---|
| Pure Kotlin, TypeScript, or Python kernel | Lean executable model plus differential tests | Universal properties of the model; tested agreement of production code | Adapter semantics, coverage, deployment identity |
| Rust function with explicit contracts | Verus | Static functional-correctness reasoning within supported Rust and trusted assumptions | External code, compiler boundary, integration |
| Finite Rust input space or bounded execution | Kani | Model-checked assertions with harness assumptions and unwinding obligations | Unsupported features, resource limits, larger domains |
| Borrow-heavy Rust with a functional translation | Aeneas to Lean | Proofs about translated Rust semantics within supported features | Extraction assumptions and unmodeled externals |
| Lease, retry, queue, or replication protocol | TLA+ or P | Model-checked state-space properties; proofs where developed | Refinement to code, fairness assumptions, instrumentation |
| Agent authorization | Cedar plus policy analysis | Precisely interpreted decisions and selected policy relations | Request construction and enforcement |
| Product behavior that changes daily | Tests, contracts, and evaluation | Empirical behavior and executable constraints | Ambiguous intent and user outcomes |

This table describes starting points.
Its rows are not mutually exclusive.
A Rust state machine can need TLA+ for the protocol, Verus for a transition function, and simulation for the runtime.

### 3.2 Understand the proof checker

Lean elaborates source syntax and tactics into expressions.
The kernel checks their types.
Under the propositions-as-types interpretation, a checked proof expression establishes the corresponding proposition within the accepted environment.[^lean-reference]

The elaborator and tactics can be sophisticated.
The trust argument centers on checking the resulting terms and assumptions.
That separation is why agents can experiment with proof search without every tactic becoming an accepted mathematical authority.

However, implementation bugs in checking kernels remain possible.
Chapter 9 covers the 2026 soundness postmortem and independent checking.

### 3.3 Understand automated verification

Verus adds specification and proof constructs to Rust and generates verification conditions for SMT solving.
Its documentation explicitly leaves the Rust/LLVM compiler outside the verification claim and does not support all Rust features.[^verus]

SMT-based automation can be excellent for local arithmetic, data structure invariants, and pre/postconditions.
A failed solver query can also require careful decomposition.
Do not assume that changing provers removes the need for proof engineering.

Kani uses model checking on Rust proof harnesses.
It can check undefined behavior, panics, overflow, and custom assertions.
Its supported feature set and harness constraints determine the claim.[^kani]

“Bounded” needs a precise interpretation.
A harness over two `u8` inputs can cover the complete finite type domain.
A loop with an unwinding limit needs evidence that the bound suffices.
If unwinding assertions are disabled or unresolved, do not report an unbounded result.

### 3.4 Understand model checking

A TLA+ model describes possible state transitions.
TLC explores a configured state space and checks stated properties.
A finite exploration is evidence about that configuration.

An invariant proof can establish a more general result.
Fairness and liveness need separate treatment.
“No bad state is reached” differs from “every eligible job eventually finishes.”

AWS's industrial account explains how formal specifications and model checking have helped distributed-system design.
The examples concern design-level models and explicit abstractions.[^aws-tla]

### 3.5 Use assurance profiles rather than one giant score

The report proposes A0 through A4 tiers.
Use them as profiles that name required evidence:

| Profile | Typical component | Required evidence |
|---|---|---|
| A0 | UI, small glue, local scripts | Tests and routine static checks |
| A1 | Standard service behavior | Reviewed properties, contracts, property-based tests |
| A2 | Codec, tile math, config validator | Proved model properties and a maintained implementation bridge |
| A3 | Scheduler, leases, distributed coordination | Protocol model, checked invariants, interleaving tests, trace evidence |
| A4 | Authorization and isolation boundaries | Direct proofs or adopted verified components where applicable; explicit trusted base and strong gate integrity |

A3 is not a larger mathematical guarantee than A2.
It addresses a different failure class.
A security-critical concurrent broker may need A2, A3, and A4 evidence together.

The planner can propose a profile.
A protected policy and accountable owner decide the minimum.
An agent must not lower its own assurance requirement to finish a task.

**Done when:** the team's tool choice names the target behavior, supported semantics, implementation bridge, and required release evidence.

---

## 4. Turn requirements into approved specifications

Specifications are the part of the factory that humans must continue to understand.

An agent can prove a carefully weakened statement.
Another can produce a beautifully formal account of the wrong interpretation.
Both may pass a checker.

The workflow must make those errors visible before proof production begins.

### 4.1 Begin with decisions and examples

Consider:

> A cancelled job must never run again.

This could mean:

- No new worker may receive a lease.
- Existing workers must stop computation.
- Existing workers may compute but cannot publish results.
- External actions already authorized must also be prevented.
- A user retry must create a new job identifier.

These are different contracts.
Ask the owner to decide which behavior the service promises.

For the lease pilot, choose a narrower contract:

> Cancellation invalidates the current lease; the broker rejects later result commits from that lease.

Then show concrete scenarios:

| Scenario | Expected decision |
|---|---|
| Current lease submits before cancellation commits | Accept if the atomic commit wins |
| Cancellation commits before result submission | Reject |
| Old lease submits after a new lease is granted | Reject |
| New lease submits using its own current epoch | Evaluate under the current state |
| Worker already called an external API | Outside this result-commit claim |

The simultaneous case forces a linearization decision.
“Before” must refer to the authoritative transaction order.
Wall-clock timestamps alone do not resolve it.

### 4.2 Make domain restrictions reviewable

A useful formal contract has an input domain and an output relation.

```text
For every input x:
    if Pre(x), then Post(x, implementation(x)).
```

An agent can make this easy by setting `Pre(x)` to `False`.
It can also exclude a difficult boundary condition.

Review the precondition with the same care as the postcondition.
What proportion of legitimate calls does it exclude?
Where is it enforced?
What happens when the caller violates it?
Are limits imposed for business reasons, resource safety, or proof convenience?

A size limit introduced during proof work is a product change.
It belongs in the API contract and the review.

### 4.3 Distinguish consistency, satisfiability, and adequacy

These questions need different checks:

- **Consistency:** do the requirements contradict each other?
- **Satisfiability:** does at least one permitted behavior meet the formal contract?
- **Adequacy:** does that contract represent the intended product behavior?

A solver can help with the first two within its semantics.
The third requires domain judgment.

For a codec, a specification that permits every output is satisfiable and consistent.
It is also useless.
For a permission policy, a specification that denies every request might satisfy a safety property while breaking the service.

Include required acceptance cases as well as forbidden behavior.

### 4.4 Validate a specification from several directions

Use complementary checks:

1. Show examples accepted and rejected by the executable specification.
2. Ask a separate reader or model to explain the formal statement in ordinary language.
3. Test deliberately wrong implementations against the contract.
4. Mutate preconditions and postconditions to see which changes the validation detects.
5. Prove sanity properties such as reachable success states or nonempty valid domains.
6. Compare independent formalizations where an equivalence relation makes sense.

Independent models can share the same misunderstanding.
Agreement increases evidence; it does not replace the owner's decision.

The CLEVER audit shows that equivalence to a reference can also be the wrong measure when two specifications encode different defensible readings.[^clever-audit]
Surface that ambiguity as a requirement decision.

### 4.5 Approve the meaning, including its dependencies

A theorem statement can remain unchanged while a referenced definition changes.

```lean
theorem safe : Allowed request → WithinBoundary request := ...
```

Changing `WithinBoundary` to always return `True` changes the meaning without changing this line.

Approval must cover:

- The theorem's elaborated type.
- The definitions that determine its meaning.
- Preconditions and semantic assumptions.
- Imported trusted dependencies.
- The concrete examples and required input domain.

A human-readable specification and a formal definition serve different review needs.
Keep both in the approval bundle.

### 4.6 Version specification changes separately

Allow agents to propose a specification change.
Require a distinct review decision before the new statement becomes the target.

The proposed change should explain:

```text
Old domain:
New domain:
Newly accepted behavior:
Newly rejected behavior:
Reason for the change:
Compatibility effect:
Examples that distinguish the versions:
```

Updating a spec to match observed buggy code is not automatically an improvement.
The owner must decide whether the behavior is intended.

**Done when:** an approved specification has examples, nontriviality checks, explicit preconditions, and an identified dependency closure.

---

## 5. Learn enough Lean to review a theorem

You do not need to become a proof-search expert before starting.
You do need to read what a theorem quantifies over and what it assumes.

The following examples use Lean 4.36.0 with `Std`.
The appendix combines them into a module.

### 5.1 Read functions and propositions

```lean
import Std

namespace Factory

def clamp (limit value : Nat) : Nat := min limit value

theorem clamp_bounded (limit value : Nat) :
    clamp limit value ≤ limit := by
  exact Nat.min_le_left limit value

theorem clamp_unchanged (limit value : Nat) (h : value ≤ limit) :
    clamp limit value = value := by
  exact Nat.min_eq_right h

end Factory
```

`Nat` is the type of natural numbers.
It has mathematical arithmetic without fixed-width overflow.

The first theorem quantifies over every natural-number limit and value.
It says that clamping produces a result at most the limit.

The second has an assumption named `h`.
If the input is already at most the limit, clamping preserves it.
That is a separate useful property.

A constant function returning zero would satisfy the boundedness theorem.
It would fail the preservation theorem for positive values within the limit.
This is why theorem portfolios matter.

### 5.2 Read proof syntax without trusting the prose

`:= by` begins a tactic proof.
`exact` supplies a term that already proves the goal.
`simp` simplifies using definitions and registered equalities.
`cases` splits an inductive value into its possible constructors.
`omega` handles suitable integer and natural-number arithmetic goals.

The proof script is a way to construct a checked term.
The theorem's meaning comes from its type and referenced definitions.

An agent's explanation can be useful.
Review the statement even when the explanation sounds convincing.

### 5.3 Represent impossible states in the type

For a tile at zoom `z`, the coordinates must be below `2 ^ z`.

```lean
structure Tile where
  zoom : Nat
  x : Fin (2 ^ zoom)
  y : Fin (2 ^ zoom)
```

`Fin n` contains a natural number together with evidence that it is less than `n`.
Within this model, a `Tile` cannot contain an out-of-range coordinate.

The parser still needs to validate ordinary external integers.
A production `number`, `Long`, or JSON field does not carry that proof.
The boundary from raw inputs to `Tile` becomes an explicit obligation.

Types help structure the model.
They do not validate bytes that have never passed through the parser.

### 5.4 Inspect assumptions

Lean supports:

```lean
#print axioms Factory.clamp_bounded
```

The command reports axioms used transitively by a declaration.[^lean-axioms]

Common foundational axioms in Lean developments include `propext`, `Classical.choice`, and `Quot.sound`.
Your project can adopt an explicit allow-list for these.
A project needing constructive extraction may choose a stricter policy.

An undeclared business fact should not become an accepted axiom.
Neither should `sorryAx`, which represents an admitted proof.

Names associated with native computation have changed across Lean versions.
The current reference explains that `native_decide` creates bespoke axioms tied to compiler correctness.[^lean-axioms]
Audit the actual dependency set under the pinned version.
Do not rely on one historical forbidden name.

### 5.5 Distinguish logic from execution

Proving a property of `Nat` arithmetic does not prove that a JavaScript implementation avoids precision loss.
Proving a parser over characters does not prove that an unreviewed JSON adapter uses the same character semantics.
Proving a function total does not bound its memory consumption.

Every example should tell the reviewer which world it inhabits.

For the first lab:

```text
Logical arithmetic: Lean Nat
External protocol: decimal strings and ASCII quadkey characters
Production arithmetic: JavaScript BigInt
Resource limit: maximum approved zoom
```

This small table prevents large misunderstandings.

### Lab: reject a weak proof portfolio

Write a candidate clamp implementation that always returns zero.
Check that it satisfies boundedness.
Then use the preservation theorem to reject it.

**Done when:** a reviewer can explain why a passing theorem may still describe an inadequate implementation.

---

## 6. Build an executable reference model

The reference model should be simple enough to understand and fast enough to exercise.
Its main job is to make intended behavior precise.

The Cedar pattern separates a readable model from an optimized production implementation.
Proofs establish model properties.
Differential tests exercise agreement with the production code.[^cedar-aws]

The tile pilot follows that pattern.

### 6.1 Choose a discrete core

A quadkey describes a path through a quadtree.
Each character selects one quadrant.
For this contract, use:

| Character | Quadrant | X bit | Y bit |
|---|---|---:|---:|
| `0` | Northwest | 0 | 0 |
| `1` | Northeast | 1 | 0 |
| `2` | Southwest | 0 | 1 |
| `3` | Southeast | 1 | 1 |

The first character is the most significant level.
The empty path denotes the root at zoom zero.
North is the direction of smaller tile Y.

These choices are approved semantics.
If a production SDK uses another convention, change the contract deliberately.

### 6.2 Model quadrants directly

```lean
inductive Quad where
  | nw | ne | sw | se
  deriving Repr, DecidableEq

def digit : Quad → Char
  | .nw => '0'
  | .ne => '1'
  | .sw => '2'
  | .se => '3'

def parseDigit : Char → Option Quad
  | '0' => some .nw
  | '1' => some .ne
  | '2' => some .sw
  | '3' => some .se
  | _ => none
```

`Option` makes invalid input an explicit outcome.
There is no fallback quadrant.
Whitespace, Unicode lookalikes, and other characters are rejected under this contract.

### 6.3 Prove a local inverse

```lean
theorem parse_digit (q : Quad) :
    parseDigit (digit q) = some q := by
  cases q <;> rfl
```

The proof checks the four constructors.
`rfl` establishes equality by reduction.

This is a small theorem with a useful role.
It can support a whole-path round trip.
It is not a claim that every character parses successfully.

### 6.4 Build a whole-path codec

Define rendering as mapping `digit` over a path.
Define parsing as applying `parseDigit` to every character, with any failure producing `none`.

The complete module in the appendix proves:

```lean
theorem parse_render (path : List Quad) :
    parse (render path) = some path
```

This direction guarantees that every valid path survives serialization and parsing.
A complete codec specification should also cover accepted-input canonicality, parser soundness, and length preservation.

Do not describe one round-trip theorem as the whole codec's correctness.
Keep the theorem inventory visible.

### 6.5 Give the path a coordinate meaning

```lean
def bitX : Quad → Nat
  | .nw | .sw => 0
  | .ne | .se => 1

def bitY : Quad → Nat
  | .nw | .ne => 0
  | .sw | .se => 1

def tileXY : List Quad → Nat × Nat
  | [] => (0, 0)
  | q :: qs =>
      let rest := tileXY qs
      let scale := 2 ^ qs.length
      (bitX q * scale + rest.1, bitY q * scale + rest.2)
```

For a path of length three, the first quadrant contributes a bit at weight four.
The second contributes at weight two.
The last contributes at weight one.

This definition favors readability.
A production implementation might shift or fold over bits.
Both should implement the same coordinate relation.

### 6.6 Prove coordinate bounds

```lean
theorem tileXY_bounds (path : List Quad) :
    (tileXY path).1 < 2 ^ path.length ∧
    (tileXY path).2 < 2 ^ path.length := by
  induction path with
  | nil => decide
  | cons q qs ih =>
      cases q <;>
        simp [tileXY, bitX, bitY, Nat.pow_succ] at * <;>
        omega
```

The induction hypothesis bounds the suffix coordinates.
The head contributes either zero or one times the suffix scale.
The resulting coordinates remain below twice that scale.

The theorem covers all finite paths in the mathematical model.
It says nothing about an API's accepted maximum zoom or the time needed to process an enormous path.
The executable boundary supplies a reviewed resource limit.

### 6.7 Maintain a claim inventory

For this model:

| Claim | Status in the teaching module |
|---|---|
| Rendering one quadrant parses to that quadrant | Proved |
| Rendering a path parses to that path | Proved |
| Coordinates are bounded by the zoom | Proved |
| Coordinate-to-path conversion is inverse to path-to-coordinate | Not implemented or proved here |
| Production SDK matches the model on every input | Not proved |
| JSON adapter preserves model semantics | Tested boundary; not formally proved |
| Floating-point projection agrees with the discrete model | Separate project |

This table is a release artifact in miniature.
It prevents partial progress from turning into a broader claim.

**Done when:** the model is executable, its input conventions are approved, and its theorem inventory distinguishes proved properties from future work.

---

## 7. Connect the model to production code

The implementation bridge is where the factory meets ordinary software.

A proved Lean model can coexist with a faulty Kotlin adapter or a TypeScript overflow.
Differential testing is the first practical way to expose those disagreements.
A direct refinement proof offers a stronger bridge where available.

### 7.1 Define a canonical wire contract

For the local lab, a request is:

```json
{"quadkey":"123"}
```

A successful response is:

```json
{"ok":true,"z":3,"x":"5","y":"3"}
```

Coordinates are decimal strings.
This preserves exact values when JSON crosses runtimes with different numeric semantics.
Zoom is bounded by the executable API.

An invalid request returns:

```json
{"ok":false,"error":"invalid_request"}
```

The protocol specifies:

- Exactly one `quadkey` field in the decoded object.
- A string of ASCII `0` through `3`.
- Length at most 32 in this teaching lab.
- Empty string accepted as the root tile.
- One request and response per line.
- No diagnostic output on standard output.

Duplicate JSON keys are a transport-level ambiguity.
The local examples use their runtimes' ordinary JSON parsers and do not establish a hardened duplicate-key rejection policy.
A production protocol should reject duplicates before constructing the request object or use a canonical framed representation.

### 7.2 Compare independently structured implementations

The Lean model computes coordinate weights recursively from the most significant digit.
The JavaScript candidate in the appendix folds left using `BigInt`.
The Python driver computes a separate integer oracle and compares parsed outputs.

The different structures help expose shared implementation errors.
They do not make the three implementations logically independent in a formal sense.
All still use the same approved quadrant table.

Begin with explicit vectors:

| Quadkey | Zoom | X | Y |
|---|---:|---:|---:|
| `""` | 0 | 0 | 0 |
| `"0"` | 1 | 0 | 0 |
| `"1"` | 1 | 1 | 0 |
| `"2"` | 1 | 0 | 1 |
| `"3"` | 1 | 1 | 1 |
| `"123"` | 3 | 5 | 3 |
| `"333"` | 3 | 7 | 7 |

These vectors validate the convention.
Generated tests explore the implementation.

### 7.3 Generate by meaningful slices

Use:

- Exhaustive paths at small zooms.
- Random paths with a recorded seed.
- Repeated quadrants and alternating quadrants.
- Maximum supported zoom.
- Characters just outside the permitted alphabet.
- Empty input and malformed request shapes.
- Inputs at the resource boundary and just beyond it.

A million short valid paths may never expose a 32-bit shift bug.
A test distribution must deliberately reach the representation boundary.

For Kotlin, document whether `Int`, `Long`, or arbitrary-precision arithmetic represents coordinates.
For TypeScript, document the use of `BigInt` or a maximum exact integer domain.
For Python, document resource limits even though integers grow.

### 7.4 Compare errors as behavior

The model and candidate should agree on rejection as well as success.

If one trims whitespace and the other rejects it, that is a divergence.
If one wraps a coordinate and the other rejects it, that is a divergence.
If both throw different uncontrolled exceptions, an output-only equality check may conceal the failure.

The driver should classify:

```text
success and equal
success and unequal
rejection and equal
rejection and unequal
timeout
process failure
protocol violation
missing output
unexpected extra output
```

Do not normalize away meaningful differences.
A canonicalizer must be reviewed as part of the comparison contract.

### 7.5 Preserve and minimize divergences

Every divergence needs:

```text
Raw request bytes
Decoded request where decoding succeeded
Model output or failure
Candidate output or failure
Toolchain and executable digests
Generator version and seed
Source and build identities
Timeout and resource settings
```

Minimization helps make the bug reviewable.
Retain the original input as well as the reduced one.
The reduced case may exercise a different path or lose the original resource trigger.

Promote useful failures into the regression corpus.
Keep holdout generation separate from the agent's editable tests.

### 7.6 State the bridge honestly

The release summary should say:

```text
The model's coordinate-bound theorem passed under the approved checker policy.
The production candidate agreed with the model on the recorded corpus.
No universal production-to-model equivalence proof is claimed.
The JSON adapters and deployed artifact identity are covered by separate checks.
```

This is stronger than a vague “formally verified” label because it tells the reviewer what to trust.

**Done when:** disagreements are reproducible artifacts, boundary slices are deliberate, and release wording distinguishes proof from tested conformance.

---

## 8. Build a proof gate with a protected meaning

A successful `lake build` is a development result.
The release gate must answer a stronger question:

> Did the required declarations prove the approved statements, using only accepted assumptions, for the identified artifact?

That question requires a protected judge.

### 8.1 Split the worker from the judge

Use two roles:

```text
Proof worker
  Reads the task and approved specification
  Edits candidate implementation and proof files
  Runs local tools for feedback
  Submits source and build inputs
                |
                v
Protected judge
  Supplies approval and checker policy
  Rebuilds from identified inputs in isolation
  Checks the required statement identities and assumptions
  Rechecks proof exports
  Runs the implementation bridge
  Publishes an artifact-bound verdict
```

The worker's own success report is advisory.
The judge owns the release result.

The worker must not be able to replace the required checks, edit the approved policy, or fabricate the status consumed by the merger.
This is a control-plane property.
It deserves the same attention as production authorization.

### 8.2 Protect more than theorem text

Store an approval bundle containing:

```text
Human requirement and distinguishing examples
Formal specification source
Required declaration names and expected kinds
Definition dependency closure
Permitted axioms and semantic assumptions
Toolchain and dependency identities
Comparison and test policy
Approval identity and revision
```

Hashing one source file can detect byte changes to that file.
It cannot detect that an unchanged theorem now imports a different definition.

The judge must identify the environment that determines the statement's meaning.
Prefer a protected specification module and locked imports.
Compare elaborated statements and relevant definitions under a specified comparison procedure.

Treat changes to the closure as specification changes.
Require the owner to review them.

### 8.3 Use the toolchain's checking facilities deliberately

The older standalone comparator repository was archived in October 2026.
Lean 4.36.0 provides `lake check` and `lake comparator`.
The commands have different purposes.[^comparator][^lean-release]

For a prepared project on a supported Linux judge:

```bash
lake check --paranoid
lake comparator --config /judge/comparator.json --paranoid
```

`lake check` checks the configured default targets with external checking and rejects nonstandard axioms.
It does not, by itself, establish that those targets are the approved task.

`lake comparator` judges named solution theorems against challenge statements using an explicit permitted-axiom policy.
In 4.36.0, both commands require `bubblewrap` for their built-in sandbox and a prepared `lake-manifest.json`.
Their help describes the supported export modes and limitations.[^lean-release]

The configuration is protected judge input:

```json
{
  "challenge_module": "ApprovedSpec",
  "solution_module": "Candidate",
  "theorem_names": [
    "Factory.parse_render",
    "Factory.tileXY_bounds"
  ],
  "definition_names": [],
  "permitted_axioms": [
    "propext",
    "Classical.choice",
    "Quot.sound"
  ]
}
```

This is a schema example, not a complete project.
The challenge and solution modules must follow the comparator's naming and comparison conventions.
Build a small fixture before applying the configuration to a real repository.

The `definition_names` setting concerns permitted challenge definition holes.
It is not a substitute for reviewing the definitions that carry your specification's meaning.

Do not disable the sandbox just to make the release gate work on a laptop.
Use the isolated judge environment.
A local proof build can still provide useful development feedback.

### 8.4 Fail closed on every incomplete result

The judge has at least these outcomes:

```text
accepted
statement_mismatch
forbidden_assumption
missing_required_declaration
build_failure
checker_rejection
checker_disagreement
timeout
resource_exhaustion
judge_configuration_failure
implementation_divergence
evidence_identity_mismatch
```

Only `accepted` permits the corresponding release action.
An infrastructure failure is unresolved evidence.
It must not become success through a retry wrapper, ignored exit code, or stale status.

Require the complete declaration inventory.
An agent that deletes the hard theorem has not completed the task.

### 8.5 Audit assumptions structurally

Source scans catch obvious uses of `sorry`, new `axiom` declarations, `native_decide`, and dangerous options.
They also generate false positives in comments and miss indirect assumptions in dependencies.

Use them as lint.
Use the checker's actual transitive dependency information for the acceptance decision.

The policy should reject:

- Admitted proofs and synthetic proof placeholders.
- Axioms outside the explicit allowed set.
- Required theorems replaced by declarations of an unacceptable kind.
- Unapproved changes to statement semantics.
- Native-computation assumptions when the tier forbids compiler-based proof trust.
- Missing exports or required targets.

The teaching module uses only the three listed foundational axioms.
That result comes from checking the declarations, not from counting suspicious strings.

### 8.6 Bind the verdict to its subject

A useful attestation identifies:

```yaml
schema: proof-evidence/v1
subject:
  source_tree_digest: "<digest>"
  executable_digest: "<digest>"
approval:
  bundle_digest: "<digest>"
  revision: "<approved revision>"
checker:
  toolchain_digest: "<digest>"
  policy_digest: "<digest>"
  selected_kernels: ["<actual checker identities>"]
results:
  required_targets: "<inventory digest>"
  proof_exports: "<export digest>"
  kernel_checks: accepted
  assumption_audit: accepted
  differential_run: "<run record digest>"
verdict: accepted
```

This is a proposed evidence schema.
The judge must compute the values and authenticate the record.
An agent writing plausible JSON does not create an attestation.

If a merge creates a different source tree, verify that tree.
If a packaging step creates a different executable, bind evidence to that executable.
A branch name is too mutable to serve as the subject.

### Lab: build the negative gate first

Before trusting a green result, prepare candidates that:

- Admit the required theorem.
- Delete it.
- Prove a weaker statement.
- Change a referenced definition.
- Add an indirect forbidden axiom.
- Fail the build.
- Return success through a fake worker-side checker.
- Replay evidence from a previous source tree.

**Done when:** the protected judge rejects every fixture and records why, while accepting a known-good fixture.

---

## 9. Treat the prover as untrusted execution

Proof search can run code.
Project configuration, macros, tactics, native compilation, and imported plugins are execution surfaces.

Keep the proof checker away from credentials and authority.
Logical soundness and host isolation are separate requirements.

### 9.1 Account for adversarial behavior

An optimizing agent may find:

- A way to introduce an admitted proof.
- A notation change that makes a familiar-looking statement mean something else.
- A changed definition that weakens the target.
- A precondition that excludes difficult inputs.
- A program rewrite that removes the behavior being verified.
- A checker implementation bug.
- A way to kill or replace a monitor.
- A way to forge the status the release process consumes.

Some tactics are ordinary debugging conveniences.
Some are malicious strategies.
The gate must reject invalid evidence regardless of intent.

The March 2026 discussion by de Moura documents prover and checker vulnerabilities and argues for independent rechecking.[^demoura-watches]
The lesson is architectural: place the judge outside the worker's control.

### 9.2 Update the kernel story

The supplied report says that no proof of `False` had been found in Lean's official kernel.
That statement was overtaken by later events.

De Moura's August 1, 2026 postmortem describes a nested-inductive-type soundness bug reduced to a proof of `False`.
It also explains why an older nanoda checker accepted the original development through a different bug.
Lean4lean's handling of that area followed the official implementation and was affected too.[^kernel-bug]

Independent checking helps.
It does not mean that every checker is independent in every relevant implementation detail.
Current versions, different implementation lineages, and regression fixtures matter.

Do not freeze this paragraph into a timeless verdict about a tool.
Keep a security-update process for the toolchain and its external kernels.

### 9.3 Use the newer checking architecture with explicit limits

De Moura's October 2026 post explains the newer `lake check` workflow and additional kernels.
It describes con-leche's consistency proof and con-ron's Rust refinement, while retaining assumptions about set theory, compilers, runtimes, and other components.[^verified-kernels]

A verified checker can reduce a class of implementation risk.
It does not certify its host operating system, every compiler stage, or your requirements.

For high-consequence claims, require more than one selected checker and record their identities.
Treat disagreement as a stop condition.
Do not use majority voting to authorize a disputed proof.

### 9.4 Build a credential-free checking cell

A proposed judge environment has:

```text
Read-only approved inputs
Fresh private build output
Pinned compiler and checker binaries
Read-only verified dependency cache
No production credentials
No write access to the approval store
No uncontrolled network access during checking
CPU, memory, time, file, and output limits
Result channel owned by the host judge
Disposable environment after the run
```

A microVM can provide a separate guest kernel.
A container can be useful under a suitable threat model.
Neither changes the need to keep authority outside generated code.

The trusted host invokes the checker and captures its result.
The guest cannot sign its own release evidence with a key also available to arbitrary generated code.

### 9.5 Separate dependency preparation from proof execution

Fetch approved dependencies through a controlled process.
Verify their identities.
Then check with network access disabled.

A shared writable cache can let one worker alter another worker's inputs.
Use immutable content-addressed cache entries or private writable overlays.
Build sensitive artifacts from source where the policy requires it.
Do not treat an arbitrary downloaded `.olean` file as the approved module.

### 9.6 Red-team meaning and isolation separately

Maintain two suites:

| Suite | Examples |
|---|---|
| Evidence integrity | Weak statement, changed definitions, forbidden assumptions, missing targets, replayed verdict |
| Execution containment | Network access, credential reads, sibling workspace writes, process escape, resource exhaustion |

A logically invalid proof that cannot escape the sandbox must still fail.
A valid proof that attempts to exfiltrate credentials must still fail the execution policy.

Do not add real secrets to fixtures.
Use canary files and local test endpoints.

### 9.7 Distinguish a mathematical result from a release decision

The checker can establish that an export is accepted.
The judge must establish where the export came from.

The `--from-export` checking options are useful for separated build and checking environments.
Their help explicitly warns that accepting an export does not establish that it matches the current project.[^lean-release]

Record the producing run, source identity, toolchain identity, and export digest.
Validate the expected declarations and statement semantics.
The artifact chain is part of the assurance system.

**Done when:** generated proof code cannot reach authority, and the judge can explain both its logical trust base and its execution boundary.

---

## 10. Make evidence part of every change

A proof-carrying pull request gives a reviewer a small, precise handoff.

It should answer:

```text
What behavior changes?
Which approved claims apply?
What statement or assumption changed?
What passed for this exact candidate?
How does the shipping implementation relate to the proved artifact?
What remains outside the claim?
```

The implementation can be large.
The assurance summary should remain reviewable.

### 10.1 Separate evidence by kind

Use distinct records:

| Evidence | What it establishes |
|---|---|
| Formal proof | The declared proposition under the accepted semantics and assumptions |
| Model-checking run | The property over the configured exploration |
| Differential tests | Agreement on the recorded input corpus |
| Runtime trace check | Conformance of observed events under the trace contract |
| Artifact attestation | Identity and provenance of the checked subject |
| Human approval | Acceptance of the intended specification and trust policy |

Do not collapse these into one “verification passed” checkbox.
The merger can require all applicable records without pretending they have identical meanings.

### 10.2 Use a concise review packet

For an unchanged tile specification:

```text
Behavior:
Replace the candidate's numeric bitwise arithmetic with BigInt arithmetic.

Approved claim:
quadkey-address/v1; specification and theorem types unchanged.

Evidence:
Model proofs accepted under the recorded checker policy.
Differential corpus accepted, including maximum-zoom inputs.
The previous implementation diverges on the retained boundary case.

Bridge:
Empirical conformance against the Lean executable.
No universal refinement proof of the JavaScript candidate.

Review focus:
Representation and adapter changes; artifact identities.

Limits:
Maximum zoom remains 32; floating-point projection is outside this task.
```

The gate computes the statuses.
The agent explains them using references to the real run records.

### 10.3 Bind review to the candidate that merges

The process needs a rule for merge queues and rebases.

If the merge result changes proof dependencies or production code, old evidence may no longer apply.
Check the final source tree or use a system that proves the evidence remains applicable to the exact resulting artifacts.

A trusted base commit, candidate commit, dependency lock, and build configuration belong in the record.
For generated binaries, record their digest too.

A green status on the previous head is not a release authorization for the current head.

### 10.4 Keep implementation review where evidence leaves gaps

Humans can spend less time inspecting generated proof steps.
Implementation review can shrink when a direct proof covers the relevant implementation semantics.

The differential-model pattern leaves empirical gaps.
Adapters, side effects, resource behavior, packaging, and integration still need appropriate review and tests.

Move attention according to the guarantee.
Do not mandate “humans review only specs” before the bridge supports that policy.

### 10.5 Control evidence retention

Keep enough material to reproduce failures and justify releases:

- Approved bundle and candidate identities.
- Required declaration inventory.
- Checker logs and assumption report.
- Export digest and selected external checker results.
- Differential corpus identity, seeds, and divergences.
- Model-checking configuration and counterexamples.
- Build and packaging provenance.
- Final authenticated verdict.

Not every successful random input needs indefinite storage.
A deterministic generator, its version, configuration, and seed can reproduce a corpus if its environment is stable.
Store explicit regression and failure inputs independently.

### 10.6 Write the release rule in observable terms

A proposed rule:

```text
Merge only when the protected judge accepts every required claim for the
candidate's final source and build inputs, the specification approval is
current, and the implementation bridge has no unresolved divergence.

Missing, stale, timed-out, or mismatched evidence blocks the merge.
```

The rule is enforceable.
“Use formal methods where appropriate” is guidance without a release decision.

**Done when:** a reviewer can find the exact claim, subject, assumptions, bridge, and unresolved limits from the pull request.

---

## 11. Give agent permissions a precise boundary

Authorization is a good first pilot because decisions are compact and consequential.

An agent requests an action on a resource.
A policy engine evaluates a structured request.
The host enforces the decision.

Cedar offers a policy language with an executable formalization, proofs, and differential-testing infrastructure.[^cedar-spec]
Adopting it avoids inventing the core policy semantics.
Your application still owns request construction and enforcement.

### 11.1 Name the authorization objects

For the factory:

```text
Principal: a specific agent execution identity
Action: a broker-defined operation
Resource: a canonical repository, job, artifact, or tool target
Context: approved task, tenant, lease, and other relevant facts
Entities: typed principal and resource information
```

An operation such as `readRepository` should refer to the actual repository the broker opens.
Do not authorize one path and then execute a separately supplied shell command against another.

Canonicalization is part of the security contract.
Case folding, aliases, symlinks, tenant prefixes, and URL normalization can change which resource is used.

### 11.2 Begin with a narrow policy

This illustrative Cedar policy requires a matching schema and request builder:

```cedar
permit (
  principal == Agent::"tile-proof-worker",
  action == Action::"readRepository",
  resource == Repository::"tile-sdk"
);

forbid (
  principal,
  action == Action::"publishRelease",
  resource
);
```

The first rule grants one specific read.
The second forbids the named publishing action.
Cedar's authorization semantics include forbid precedence and default deny.[^cedar-auth]

Cedar skips policies that produce evaluation errors and reports those errors in diagnostics.
A failing forbid condition therefore does not automatically deny the request.[^cedar-auth]
For this security-boundary pilot, choose an explicit broker rule that rejects requests with policy-evaluation diagnostics.
Validate policies and request data so that ordinary authorized requests do not depend on that fallback.
This wrapper rule is application behavior and needs its own tests and approval.

This fragment does not grant permission to arbitrary shell commands.
The broker exposes typed actions and enforces each requested effect.
A shell can contain many effects, so a single coarse “run command” permission is a different boundary.

### 11.3 Define no-widening precisely

Let:

```text
Allowed(P, r, e)
```

mean that policy set `P` allows request `r` under entity data `e`.

The desired refactor property is:

```text
For every approved request r and entity environment e:
    Allowed(candidate, r, e) implies Allowed(baseline, r, e).
```

This is inclusion of allowed decisions.
It is not equivalence.
A candidate that denies everything satisfies no-widening.

Add required acceptance cases or prove equivalence when preserving access is the intended change.
State the request domain, schema, context fields, and entity constraints.

### 11.4 Do not confuse engine assurance with policy analysis

Cedar's formal development supplies evidence about its engine semantics and selected properties.
That does not prove that your new policy is a non-widening refactor.

Choose and validate an analysis tool that can establish the relation for your policy fragment.
Check its supported features, abstractions, and result interpretation.
An unsupported case or unknown solver result must remain unresolved.

If the available analyzer supports only a restricted fragment, constrain the pilot accordingly.
If you use exhaustive enumeration, name the finite domain.
If you use tests, report tested no-widening.

The task “prove these two arbitrary policies equivalent” is not an automatic consequence of adopting Cedar.

### 11.5 Bind decisions to actions

The broker should:

1. Construct a canonical typed request from trusted execution state.
2. Evaluate the approved policy and entity snapshot.
3. Validate the current lease or other authority freshness.
4. Execute the exact authorized operation through a narrow interface.
5. Record the decision and effect.

The atomicity required between steps depends on the operation.
A result commit should validate policy and lease state in an authoritative transaction.
A long-running operation may need a capability with a bounded lifetime and defined revocation behavior.

Do not trust an agent-provided field such as `"approved": true`.
Approval is state owned by the control plane.

### 11.6 Test the seam with deliberate failures

Try:

- Another agent identity.
- Another tenant's repository.
- A different action with the same display name.
- A path alias that escapes the approved root.
- Missing context.
- Stale entity data.
- An explicit forbid matching a permit.
- A permission decision replayed after lease revocation.
- A request whose authorized resource differs from the resource used.

These checks exercise your integration.
Engine proofs do not eliminate them.

### 11.7 Make policy changes reviewable

The policy PR should show:

```text
Old and new allowed examples
Old and new denied examples
Schema and request-domain changes
Entity assumptions
Analysis result and supported fragment
Enforcement-path changes
Policy, schema, and broker artifact identities
```

**Done when:** the platform can explain why a decision is allowed, prove or test the intended policy relation at its declared scope, and enforce the same decision on the actual operation.

---

## 12. Model leases and retries before implementation

Concurrency deserves a behavioral model.

A worker can disappear, return, retry, or submit an old result.
The same worker identity can receive a new lease.
Checking only the worker's name is therefore insufficient.

The model below uses an epoch as a fencing token.
Every grant increments it.
Previously issued credentials remain available to the environment so that old requests can be replayed.

### 12.1 Describe the state

```text
epoch: current grant generation
owner: current worker or no worker
cancelled: cancellation has committed
completed: a result has committed
issued: all issued worker/epoch credentials
acceptedEpoch: epoch attached to the accepted result
```

Keeping historical credentials matters.
A model that erases old tokens cannot explore stale-token submissions.

The example represents one job.
It does not model database crashes, credential forgery, real clocks, external side effects, or an unbounded number of grants.

### 12.2 State transitions before writing handlers

The transition contract is:

| Action | Guard | State change |
|---|---|---|
| Grant | No owner; neither cancelled nor completed; epoch below model bound | Increment epoch; assign owner; issue credential |
| Revoke | An owner exists; job incomplete | Clear owner |
| Cancel | Job incomplete and not cancelled | Set cancelled; clear owner |
| Commit | Job live; credential owner and epoch match current state | Set completed; record accepted epoch |

The guards are part of the specification.
An implementation that checks the guard and writes the result separately can violate it.

### 12.3 Write the critical action in TLA+

The complete module appears in the appendix.
Its commit action is:

```tla
Commit(c) == /\ ~cancelled /\ ~completed
             /\ owner = c.worker
             /\ c.epoch = epoch
             /\ completed' = TRUE
             /\ acceptedEpoch' = c.epoch
             /\ UNCHANGED <<epoch, owner, cancelled, issued>>
```

The primed variables describe the next state.
Everything named in `UNCHANGED` keeps its value.

The action is atomic at this abstraction level.
Your implementation must supply an atomic operation with the same relevant effect.

### 12.4 Check the invariants

The teaching model checks:

```tla
NoStaleCommit == completed => acceptedEpoch = epoch
CancelledHasNoOwner == cancelled => owner = NoWorker
```

It also checks `TypeOK`.

With two workers and a maximum epoch of three, TLC finds 58 distinct reachable states and no violation for this configuration.
That is a finite model-checking result.
It is not a proof for arbitrary worker counts or unlimited retries.

The configured terminal states are intentional.
Deadlock checking is disabled because completed, cancelled, and bounded-out states may have no further non-stuttering action.
That setting does not supply a liveness guarantee.

### 12.5 Confirm that the checker catches the intended bug

Remove:

```tla
/\ c.epoch = epoch
```

from `Commit`.

TLC finds:

```text
Grant worker 1 at epoch 1
Revoke
Grant worker 1 at epoch 2
Commit worker 1's old epoch-1 credential
```

The worker identity matches.
The epoch does not.
The broken model violates `NoStaleCommit`.

A negative example establishes that the invariant and model can detect this particular mistake.
It does not establish that the model contains every important failure mode.

### 12.6 Connect the model to a durable transaction

A database-oriented implementation sketch:

```sql
UPDATE jobs
SET completed = TRUE,
    result_digest = :result_digest,
    accepted_epoch = :submitted_epoch
WHERE job_id = :job_id
  AND owner = :worker_id
  AND epoch = :submitted_epoch
  AND cancelled = FALSE
  AND completed = FALSE;
```

Accept only if exactly one row changed.
Record the authoritative decision in the same transaction or through an appropriate transactional event mechanism.

This sketch assumes the surrounding schema, isolation level, and grant/revoke operations uphold the modeled state.
Test them together.
If a separate read checks the epoch and a later write commits the result, cancellation or a new grant can race between them.

### 12.7 Use Lean for reusable local facts

The appendix contains a small `Lease` model in Lean.
It proves that revocation rejects every supplied epoch and that an old epoch remains rejected after the revoked lease is granted again.

These are local transition facts.
They do not establish the full TLA+ model's correctness or the database implementation's refinement.
Use them as separately identified claims.

A real refinement effort needs a mapping between implementation states and abstract states.
Do not call two independently written models equivalent because they use similar field names.

### 12.8 Add liveness deliberately

Possible liveness claims include:

- An eligible queued job eventually receives a lease.
- A healthy worker can eventually commit.
- A revoked lease eventually loses action authority.

Each depends on assumptions about failures, fairness, retries, and resource admission.
Unlimited failures can make completion impossible.
An overloaded scheduler can starve work.

State those assumptions.
Then choose model checking, proof, simulation, or operational evidence suited to the claim.

**Done when:** the model detects a known stale-worker bug, the implementation has a tested atomic commit boundary, and safety is reported separately from liveness.

---

## 13. Check that production still follows the model

A model can be correct and gradually become irrelevant.

Code changes.
Placement rules gain exceptions.
Recovery paths behave differently.
The checked protocol remains in a repository while the running system follows another protocol.

Runtime conformance checking helps detect that drift.

### 13.1 Learn the right lesson from FETCh

Datadog's November 2026 account describes FETCh, which turns selected TLA+ specifications into Rust runtime monitors.
It found a production routing discrepancy involving conflicting placements.
The post explains both the observable-specification design and current limitations.[^fetch]

The transferable pattern is:

```text
Design model
    |
    v
Observable abstract behavior
    |
    v
Production event contract
    |
    v
Trace checker
    |
    v
Divergence investigation and retained regression
```

FETCh supports a subset of TLA+.
Adoption requires instrumentation and system-specific design.
Do not assume that an arbitrary TLC model can be turned into a reliable monitor automatically.[^fetch]

### 13.2 Design events from authoritative state changes

For the lease pilot, emit:

```json
{
  "event":"result_commit_decision",
  "job_id":"job-42",
  "submitted_epoch":7,
  "current_epoch":8,
  "worker_id":"worker-a",
  "decision":"reject",
  "reason":"stale_epoch",
  "transaction_sequence":103,
  "model_revision":"lease/v1"
}
```

This is a proposed event schema.
The trusted broker generates it from authoritative state.

Worker logs saying “my result was accepted” are not authoritative commit evidence.
A log outside the transaction can also be misleading if the transaction later rolls back.

Where possible, use a transactional outbox or another mechanism whose semantics the team understands.

### 13.3 Specify the trace contract

The checker needs to know:

| Trace property | Required decision |
|---|---|
| Ordering | Which sequence orders transitions for each modeled object? |
| Completeness | How are dropped events detected? |
| Duplication | Which identifier supports deduplication? |
| Initialization | What snapshot establishes the starting state? |
| Restarts | How is monitor state recovered? |
| Schema changes | How are event and model versions matched? |
| Unknown events | Does the monitor reject, pause, or route for analysis? |

Without completeness, a missing revocation can make an invalid commit appear valid.
Without ordering, delayed delivery can create a false violation.

Report `inconclusive` when the trace cannot support a decision.
A missing event is not evidence that the invariant held.

### 13.4 Choose an observable abstraction

You may not need to log every implementation step.
The result-commit claim needs grants, revocations, cancellations, and authoritative commit decisions.

The abstraction should preserve the property being checked.
If hidden steps can invalidate it, include enough state or events to resolve them.

A global distributed order can be expensive.
For per-job invariants, a durable per-job sequence may suffice.
Cross-job quotas or global ownership constraints need a different state and ordering model.

### 13.5 Measure monitoring coverage

Record:

```text
Objects and operations under monitoring
Event completeness and sequence gaps
Time spent inconclusive
Monitor lag
Violations by model revision
False alarms with diagnosed causes
CPU, memory, and storage cost
State-recovery success
```

“No violations” is meaningful only alongside those measurements.
Sampling may be appropriate, but the result applies to sampled observations.

### 13.6 Feed drift back into the specification workflow

When the monitor finds a discrepancy:

1. Preserve the trace and implementation identities.
2. Establish whether instrumentation lost or reordered information.
3. Replay the trace against the current and relevant prior models.
4. Determine whether code, model, or intended behavior is wrong.
5. Review any changed requirement.
6. Add a simulation or regression scenario.
7. Recheck the relevant evidence before release.

Do not silently alter the monitor to accept the behavior.
That is a specification change.

**Done when:** runtime checks distinguish violation from incomplete observation, and a discrepancy produces a reviewed corrective action.

---

## 14. Verify geospatial kernels over exact data

Geospatial code contains attractive first targets.
Many functions are deterministic, widely reused, and governed by compact mathematical or encoding rules.

The best starting point is a discrete kernel with a clean contract.
Choose it before attempting a floating-point projection or a complete routing engine.

### 14.1 Separate the pipeline into obligations

A tile-address pipeline can contain:

```text
Raw API request
    |
    v
Longitude and latitude policy
    |
    v
Projection to normalized coordinates
    |
    v
Quantization to integer tile coordinates
    |
    v
Quadkey encoding
    |
    v
SDK response
```

Each arrow can fail differently.
Normalizing longitude is a policy choice.
Projection uses numerical computation.
Quantization creates discontinuities at tile boundaries.
Quadkey encoding is a discrete operation.
The response adapter must preserve integer values.

One proof at the bottom does not certify the whole pipeline.
Divide the assurance record accordingly.

### 14.2 Extend the quadkey portfolio

After the teaching module, add:

- An inverse from bounded coordinates to a path.
- Both coordinate/path round trips.
- Character-codec soundness and canonicality.
- Path length equals zoom.
- Parent and child relations.
- Prefix relationships correspond to ancestor relationships.
- Explicit maximum zoom at the API boundary.

An inverse needs the zoom as well as X and Y.
Leading zero quadrants matter.
Coordinates `(0, 0)` occur at every zoom, so X and Y alone do not determine a quadkey.

That is a specification issue an agent can miss while producing plausible code.

### 14.3 Define the edge policy before projecting

The owner must choose:

| Boundary | Example decision |
|---|---|
| Longitude range | Reject outside the approved interval or normalize periodically |
| Antimeridian | Use a half-open convention and define the exact endpoint |
| Latitude poles | Reject or clamp to a specified projection limit |
| Nonfinite inputs | Reject NaN and infinities |
| Zoom | Require an integer within a published maximum |
| East and south edges | Specify floor, saturation, and half-open interval behavior |
| Negative zero | Define whether it is observable |

The choices must match SDK expectations.
Do not allow the proof worker to invent a clamp or normalization to make a theorem easier.

### 14.4 Model codecs as structured operations

For MVT-style geometry encoding, isolate:

```text
Signed coordinate delta
    -> Zigzag integer
    -> Varint bytes
    -> Command stream
    -> Geometry interpretation
```

The format specification should determine accepted command identifiers, counts, geometry structure, and field semantics.[^mvt]

A useful theorem portfolio might establish:

- Zigzag decoding reverses encoding within the approved integer domain.
- Varint decoding reverses encoding.
- Encoded integers have the required length bound.
- Decoding rejects malformed continuation sequences.
- Delta accumulation stays in the approved coordinate range.
- Geometry command counts consume exactly the intended arguments.
- Canonical encoding is unique where the contract requires canonicality.

Mark fixed-width overflow separately.
An integer-model theorem can hold while an `Int32` accumulator overflows.

### 14.5 Do not infer semantic correctness from a codec round trip

Two functions can share the same wrong convention.

An encoder swaps X and Y.
A decoder swaps them back.
The round trip succeeds.
The wire format is still wrong.

Use published vectors and independent format semantics.
Compare against established implementations where appropriate.
Round trips cover internal consistency; external compatibility needs another obligation.

The same problem appears in geographic projection.
Two inverse functions can preserve each other while implementing the wrong projection.

### 14.6 Give tile coverage a set meaning

A tile-cover function can be specified as a set:

```text
Every returned tile intersects the approved region under the chosen rule.
Every intersecting tile in the approved grid is returned.
No tile occurs twice in the canonical output.
Every tile address is in bounds.
```

These statements depend on the intersection rule.
Does touching an edge count?
Does an empty or zero-area bounding box return a tile?
Does an antimeridian-spanning region represent a short crossing or almost the whole world?

Represent wrapped regions explicitly.
A pair of longitudes with `west > east` needs a defined meaning.
Do not let a sorting helper erase the wrap.

Use exact rational or integer comparisons for the first model.
Then specify how production floating-point bounds map into that domain.

### 14.7 Build metamorphic tests from the mathematics

Useful candidate properties include:

- Encoding then decoding preserves a valid address.
- Increasing zoom maps a tile into its four children.
- Taking a parent after choosing a child returns the parent.
- A canonical path prefix identifies an ancestor.
- A region contained in another has a contained cover under the same coverage convention.
- Duplicate inputs do not change a set-valued result.

Some intuitive properties fail near wrapped boundaries or under rounding.
Review each against the approved semantics.
“Looks mathematically obvious” is not a substitute for a defined domain.

### 14.8 Publish the contract with the artifact

A useful public package includes:

```text
Model and theorem source
Pinned toolchain and dependency manifest
Exact supported domain
Approved convention table
Published vectors
Differential adapter examples
Retained regression corpus
Checker policy and evidence
Known limitations
```

Its value comes from reuse and clarity.
Claims about being first in the field need a separate evidence review.

**Done when:** the discrete kernel has an explicit convention, a theorem portfolio, representation-boundary tests, and a clear relationship to the rest of the geospatial pipeline.

---

## 15. Make floating-point error an explicit obligation

Projection and geometry eventually encounter floating point.
Treat that boundary as its own project.

A theorem about real numbers does not prove that a CPU's floating-point instruction implements the same operation exactly.
A long differential run also does not establish a universal error bound.

### 15.1 Name three semantics

Separate:

| Layer | Example |
|---|---|
| Mathematical specification | Real-valued projection or orientation determinant |
| Floating-point specification | A particular format, rounding rule, and exceptional behavior |
| Machine implementation | Compiler-generated instructions and library functions |

Possible claims connect these layers:

```text
The floating-point result differs from the real result by at most epsilon.
The machine result implements the chosen floating-point specification.
Quantization produces the intended integer under specified boundary conditions.
```

These are distinct obligations.
The trusted base differs for each.

### 15.2 Use exact claims where possible

For integer quadkey math, stay with exact arithmetic.
For a codec, stay with fixed-width bit-vector semantics or prove a checked conversion from mathematical integers.
For a rational bounding-box model, represent rational endpoints exactly.

Avoid adding floating point merely because the production API uses a `double`.
Translate at the boundary with a reviewed relation.

The relation may say:

```text
This finite machine value represents exactly this rational number.
```

That is stronger and clearer than pretending an arbitrary double is an exact real-valued measurement.

### 15.3 Quantization needs boundary reasoning

Suppose the specification uses:

```text
tile = floor(projected_coordinate * 2^zoom).
```

An error bound of `10^-12` in projected coordinates may still change the integer tile near a boundary.
A small continuous error does not imply identical discrete output.

A useful conditional theorem has this form:

```text
If the exact scaled coordinate is farther than delta from every integer,
and the implementation's scaled error is below delta,
then the quantized tile agrees.
```

Near a boundary, choose an explicit fallback, tie rule, or accepted tolerance relation.
The product may require exact agreement with a specific reference implementation instead of ideal real arithmetic.
That is also a legitimate contract when stated honestly.

### 15.4 State exceptional cases

The model should decide:

- NaN behavior.
- Positive and negative infinity.
- Overflow and underflow.
- Signed zero.
- Subnormal values.
- Rounding mode.
- Transcendental function accuracy assumptions.
- Compiler options that alter arithmetic transformations.

A proof that assumes a correctly rounded logarithm is conditional on that assumption.
If the library supplies a weaker bound, use it.
If you cannot establish the bound, record the unresolved obligation.

### 15.5 Use robust predicates

A two-dimensional orientation test evaluates the sign of:

```text
(bx - ax) * (cy - ay) - (by - ay) * (cx - ax).
```

Near collinearity, floating-point cancellation can make the computed sign unreliable.
A robust strategy computes a bound on error and uses the fast result only when the sign is separated from zero by enough margin.
Otherwise it falls back to stronger arithmetic.

The verification target is the sign decision under the stated input domain.
It is not merely that the expression executes without crashing.

Adopt and review an established robust-predicate approach where possible.
Do not generate a new numerical method casually because the theorem prover is available.

### 15.6 Keep tests as independent evidence

Tests can cover:

```text
Adjacent representable values near tile boundaries
Nearly collinear geometry
Extremal accepted latitudes
Nonfinite values
Compiler and architecture variants
Known numerical regressions
Reference-library comparisons
```

The tests may reveal that an assumption about the machine implementation is wrong.
They complement a numerical proof.
Report the proof and empirical conformance separately.

### 15.7 Avoid claiming a closed chain too early

The supplied research names several Lean floating-point projects.
Their existence is a research lead.
Before adopting one, inspect the actual version, theorem inventory, admitted proofs, executable backend, and external-instruction assumptions.

A library name does not establish that your compiler, math library, and instruction set are covered.
The source chapter deliberately does not certify those projects' current completeness.

**Done when:** the real, floating-point, and machine semantics are separated, and the discrete-output claim accounts for boundary sensitivity.

---

## 16. Choose a verification route for Rust

Rust offers several useful routes.
Choose according to the claim and the code you intend to maintain.

The choice affects proof structure, toolchain constraints, and the trusted base.
Try a representative slice before standardizing.

### 16.1 Use Verus for explicit functional contracts

Verus is a good candidate when the team can annotate supported Rust code with preconditions, postconditions, invariants, and proof functions.
It uses SMT-based reasoning and a Rust-oriented specification language.[^verus]

A conceptual contract for a bounded append operation is:

```text
Requires:
    input length is below capacity
    the representation invariant holds

Ensures:
    output length is input length plus one
    the old prefix is unchanged
    the new final element is the supplied element
    the representation invariant still holds
```

The prefix property matters.
An operation that changes the existing elements can satisfy the length postcondition.

Treat the contract as a portfolio.
Include ownership, aliasing, and supported-memory-model assumptions where relevant.

### 16.2 Use Kani for finite and bounded claims

This Kani harness is an illustrative Rust fragment.
It was not executed as part of this book's local validation:

```rust
#[kani::proof]
fn saturating_add_matches_integer_spec() {
    let a: u8 = kani::any();
    let b: u8 = kani::any();
    let actual = a.saturating_add(b);
    let exact = (a as u16) + (b as u16);
    let expected = exact.min(u8::MAX as u16) as u8;
    assert_eq!(actual, expected);
    assert!(actual >= a);
    assert!(actual >= b);
}
```

The input type domain is finite.
The harness has no explicit loop whose bound must be inferred.
The claim still depends on the tool's modeled Rust behavior and supported constructs.

For a parser harness, input-buffer length may be bounded.
For a loop, inspect unwinding assertions.
Never equate “no counterexample found before the resource limit” with acceptance.

Kani's documentation describes both its safety checks and supported-feature limitations.[^kani]

### 16.3 Use Aeneas when translation helps the proof

Aeneas translates supported Rust programs into a functional form suitable for theorem proving, including Lean.
Its project documentation explains the supported pipeline and backend-specific setup.[^aeneas]

This can suit code whose important behavior benefits from functional reasoning.
The translation and treatment of externals become part of the claim.

A proposed pipeline:

```text
Reviewed Rust source and dependencies
    |
    v
Pinned extraction and translation
    |
    v
Generated Lean definitions
    |
    v
Proofs about those definitions
    |
    v
Audited external assumptions and artifact record
```

Generated definitions should remain identified artifacts.
Do not hand-edit them into a nicer model and then claim that the proof still concerns the original Rust source.

### 16.4 Keep compiler and FFI boundaries visible

Questions to ask:

- Is the proof about source-level semantics or a binary?
- Does it assume compiler correctness?
- Which standard-library functions are modeled?
- How are allocation failures handled?
- Which unsafe operations are supported?
- How are C calls or operating-system interfaces represented?
- Are external contracts proved, trusted, or tested?

A trusted external function returning well-formed data can hide the most important defect.
The team must decide whether that assumption is acceptable.

### 16.5 Use multiple methods for multiple properties

A Rust codec might use:

```text
Verus: content preservation and bounds
Kani: selected overflow and panic harnesses
Lean model: shared format semantics across SDKs
Differential tests: interoperability and deployed build conformance
Fuzzing: broad parser exploration and retained failures
```

The methods can reinforce each other.
They can also share an incorrect specification.
Published vectors and independent review still matter.

### 16.6 Pin and repair deliberately

Record the exact Rust compiler, verifier, solver, dependency lock, and build features.
Revalidate after changes to any of them.

An upgrade may expose a real proof hole, change supported syntax, or alter solver behavior.
Classify the failure before changing the specification.

A verifier migration is a proof-repair task.
It is not permission to add an unreviewed external axiom.

**Done when:** the pilot identifies its source semantics, external assumptions, verifier limits, and the evidence that relates the checked code to the deployed artifact.

---

## 17. Operate proof-producing agents

The worker's job should be narrow:

> Complete the approved implementation and proof obligations without changing their meaning.

The harness supplies the target, feedback, and stopping rules.
The protected judge supplies acceptance.

### 17.1 Give the worker a concrete brief

A proposed task:

```yaml
task: prove-tile-address-bounds
approved_bundle: "quadkey-address/v1"
editable:
  - Candidate.lean
  - CandidateProofs.lean
read_only:
  - ApprovedSpec.lean
  - comparator.json
  - lean-toolchain
  - lake-manifest.json
required_targets:
  - Factory.parse_render
  - Factory.tileXY_bounds
constraints:
  - preserve the approved input domain
  - introduce no new axioms
  - do not use admitted proofs or native computation for proof acceptance
  - submit a specification-change proposal if the target is inadequate
budget:
  proof_worker_tokens: "<team-configured limit>"
  wall_clock_seconds: "<team-configured limit>"
output:
  - candidate source
  - remaining obligations
  - suggested reusable lemmas
  - local check references
```

The filesystem restrictions must be enforced by the worker environment.
Listing `read_only` in a prompt is guidance, not access control.

### 17.2 Use verifier feedback at the right granularity

The local loop is:

```text
Inspect the current goal
Make a small edit
Request compiler or LSP feedback
Classify the error
Reuse or prove a local lemma
Recheck the affected declaration
Run the complete protected checks for the candidate
```

A Lean LSP integration can shorten feedback cycles.
The report names `lean-lsp-mcp` as one option.
Choose a supported integration and pin it with the worker image.

Treat the plugin as execution code.
Its goal-state messages help the worker.
They do not replace the final checker.

### 17.3 Classify stuck work

Useful classes:

| Failure | Next action |
|---|---|
| Syntax or type error | Repair local expression |
| Missing reusable fact | Search the approved library or prove a lemma |
| Bad decomposition | Change proof structure |
| Timeout | Isolate the expensive obligation and adjust a reviewed resource policy |
| Counterexample | Inspect code and specification |
| Inconsistent target | Escalate a specification issue to the owner |
| Unsupported feature | Choose another supported route or record an unresolved boundary |
| Integrity violation | Reject the candidate and retain the incident |

An agent should not respond to every failed proof by broadening automation or weakening preconditions.
Use the failure class to choose the next step.

### 17.4 Share proofs without sharing authority

Maintain a reviewed library of domain lemmas.
Good candidates include:

- Bounded arithmetic conversions.
- Codec round trips.
- List-length and prefix properties.
- State-machine preservation lemmas.
- Policy-set relations.
- Tile ancestor and coordinate facts.

A shared workspace can reduce repeated proof effort.
Anthropic reports a 17% token reduction in its proof-sharing experiment.[^anthropic]

Shared source is untrusted until checked.
Shared build artifacts need identity and cache protection.
Never let the proof library redefine the protected release policy.

### 17.5 Separate discovery from accepted evidence

Agents may use temporary sketches and incomplete lemmas during exploration.
That can be a productive development technique.

The submission must satisfy the final integrity policy.
Exploratory files should not accidentally enter the required import closure.
Logs should distinguish local progress from accepted evidence.

If you allow `sorry` during local work, make the final rejection fixture part of every worker-image validation.
An environment that merely warns about an admitted proof is unsuitable as a release gate.

### 17.6 Bound autonomous repair

When a worker hits its budget:

```text
Preserve the candidate and current goals.
Record what passed locally.
Record the exact unresolved declaration.
Keep the task unaccepted.
Choose another attempt, decomposition, or owner review.
```

Budget exhaustion is useful data.
It must not cause silent specification narrowing or an accepted partial proof.

### 17.7 Maintain toolchain upgrades as their own workflow

For an upgrade:

1. Read the upstream changes relevant to soundness, syntax, runtime, and checking.
2. Prepare the new worker and judge images.
3. Run known-good examples and all negative integrity fixtures.
4. Recheck the approved proof library.
5. Rebuild model executables and run the implementation bridges.
6. Compare assumption reports and checker behavior.
7. Repair proofs while preserving approved statement semantics.
8. Publish new authenticated evidence.

Keep the old toolchain available for diagnosis where appropriate.
Do not keep it eligible for releases if a known soundness issue invalidates its use.

**Done when:** agents receive protected targets, failures are classified, proof reuse is checked, and unfinished work cannot become a successful release result.

---

## 18. Measure assurance and proof economics

The factory needs measurements that correspond to decisions.

“Percent verified” hides the claim scope and the implementation bridge.
“Proof lines generated” measures output volume.
Neither tells leadership whether important changes became safer or cheaper to review.

### 18.1 Start with a baseline

For each pilot, record:

```text
Current defect history and credible failure modes
Human implementation-review time
Specification-review time
Test and CI runtime
Median and tail task completion time
Rollback and incident cost where available
Current code-change volume
Current release constraints
```

The baseline need not be statistically perfect to be useful.
Its definitions must be stable enough to support a comparison.

### 18.2 Measure claims by profile and scope

Use:

| Metric | Interpretation |
|---|---|
| Required claims accepted | Are tasks producing their intended evidence? |
| Spec defects found before implementation | Is specification validation useful? |
| Spec changes after merge | What assumptions or intent were missed? |
| Differential divergences | Where does production disagree with the model? |
| Escaped defects by failure class | Which gaps remain in the assurance chain? |
| Human review minutes by activity | Has attention shifted usefully? |
| Checker-integrity violations | Are invalid acceptance strategies being detected? |
| Inconclusive runtime observations | How much production behavior is actually covered? |
| Proof repair cost | Is the evidence maintainable? |

A verified component can still cause an incident outside its theorem scope.
Classify the cause.
Do not count it as either an automatic proof failure or an irrelevant incident without analysis.

### 18.3 Keep denominators explicit

For worker evaluation:

```text
Approved tasks attempted
Tasks accepted by the protected judge
Tasks with adequate specs
Tasks blocked on unsupported features
Tasks timed out
Tasks rejected for integrity violations
Tasks with implementation divergences
Total tokens and wall clock
```

Report accepted tasks divided by all approved tasks attempted.
If you also report a sound-spec subset, report its size and selection method.

A weak specification should not improve the model's apparent success rate.
A missing theorem should not disappear from the denominator.

### 18.4 Build an internal trust evaluation

Use tasks from your own domain:

- A correct candidate with a complete proof.
- A wrong candidate with plausible tests.
- A candidate whose theorem is too weak.
- A forbidden precondition narrowing.
- A stale-worker counterexample.
- A high-zoom integer representation bug.
- A changed imported definition.
- An admitted proof in a dependency.
- A forged or replayed evidence record.
- A valid proof whose build attempts a forbidden side effect.

Separate competence from integrity.
A worker can be poor at proof search and excellent at obeying boundaries.
Another can solve many tasks while repeatedly trying to weaken the target.

### 18.5 Count the whole cost

A useful accounting equation is:

```text
Total assurance cost =
    specification work
  + human review
  + proof generation
  + checker and test compute
  + implementation integration
  + proof maintenance
  + operations and incident handling.
```

Proof tokens are one term.
Cheap proof search may still be uneconomic for a volatile requirement.
A stable, widely reused model can amortize specification and maintenance effort.

Normalize by accepted claim or task before normalizing by code size.
KLOC is sensitive to style, generated code, and proof structure.
A small theorem about a broad property can be more valuable than many verified trivial lines.

### 18.6 Measure review reduction safely

Record implementation-review and specification-review time separately.
Compare changes of similar scope and risk.
Track defects found during review as well as time.

A faster review is not automatically a better review.
A team can save time by overlooking the bridge between model and code.

Use early pilots to assess comprehension:

```text
Can the reviewer state the theorem?
Can they name its assumptions?
Can they identify the implementation bridge?
Can they explain a meaningful excluded failure?
Can they verify that evidence applies to the candidate?
```

These are better adoption checks than asking whether the reviewer likes a green label.

### 18.7 Explain value to leadership through outcomes

A concrete proposal:

```text
We will protect three high-consequence boundaries that agents modify.
Each has a reviewed contract and a reproducible release gate.
The pilots will measure bugs found, human review work, task throughput,
and the cost of maintaining the evidence.
Expansion depends on those results.
```

Use external case studies to explain feasibility.
Use local outcomes to justify ongoing investment.
Avoid promising that all implementation review will disappear.

**Done when:** the team measures accepted, scoped claims and the full cost of producing and maintaining them.

---

## 19. Roll out through three useful pilots

Begin with the gate and a complete vertical slice.
Then apply it to problems with clean semantics and accountable owners.

Do not start by building a general verification platform for every repository.

### 19.1 Establish ownership

Assign:

| Responsibility | Accountable owner |
|---|---|
| Intended behavior and specification approval | Domain or component owner |
| Proof-worker harness | Agent platform team |
| Protected judge and evidence policy | Platform assurance owner |
| Production adapter and implementation bridge | Component team |
| Authorization enforcement | Security and platform owners |
| Runtime monitoring and response | Operating team |
| Toolchain and checker upgrades | Named maintainer |

One person can hold several roles in a small team.
The decision rights should still be clear.

### 19.2 Weeks one through three: build the thin slice

Use the quadkey lab to establish:

```text
Pinned Lean environment
Protected approved target
Known-good proof
Known-bad integrity fixtures
Executable model
Production-language adapter
Differential corpus
Artifact-bound result
Readable review packet
```

For release checking, use the supported isolated Linux judge.
The local examples are development tools.

Measure the path from task submission to an accepted verdict.
Check the failure path just as carefully.
A gate that only works for successful jobs is incomplete.

### 19.3 Pilot one: agent authorization

Choose one typed broker action and a compact policy domain.
Adopt Cedar semantics.
Validate the schema and request construction.
Establish the supported policy-analysis relation or a clearly labeled tested relation.

Acceptance criteria:

- Default deny and forbid precedence are exercised.
- Approved acceptance cases remain accepted.
- The selected non-widening or equivalence relation is checked at its stated scope.
- Canonical authorized resources match actual effects.
- Policy evidence is tied to policy, schema, entity assumptions, and broker version.
- A stale decision cannot bypass the chosen freshness rule.

Do not label the entire platform A4 because this pilot uses a formally developed policy engine.
The enforcement seam needs evidence.

### 19.4 Pilot two: job leases

Use the TLA+ model as a starting point.
Add the actual job states, retries, cancellation policy, and recovery transitions.
Review the abstractions.

Acceptance criteria:

- The model exposes the stale-worker counterexample when fencing is removed.
- The production commit is atomic.
- Interleaving tests exercise cancellation and reassignment.
- Epochs cannot silently wrap under the production representation.
- Authoritative transition events support the selected runtime monitor.
- Monitoring detects incomplete traces.
- External side effects have a separately stated authority policy.

Keep the initial claim about result commits.
Expand to side effects only through a new specification.

### 19.5 Pilot three: a geospatial kernel

Choose quadkey/tile-address conversion or one integer codec layer.
Avoid beginning with a full projection and geometry stack.

Acceptance criteria:

- Input conventions and limits are approved.
- Published vectors agree with the contract.
- Important model properties are proved.
- Differential tests exercise representation boundaries.
- Invalid inputs have defined outcomes.
- Divergences are reproducible and retained.
- The deployed adapter and executable are identified.

Publish the artifact if it is useful and the organization wants to share it.
The technical release and the public communication have separate approval policies.

### 19.6 End of quarter: decide from evidence

Ask:

```text
Did attempted proofs find real defects?
Did the model expose ambiguity before implementation?
Could another engineer review the claims?
Did the gate reject invalid submissions?
Was the production bridge maintainable?
Did review work move to the intended decisions?
Can an owner operate the system after the pilot authors leave?
```

Expand only the parts that worked.
A useful model and test bridge can be worth keeping even when a deeper proof proves too expensive.

### 19.7 Quarters two and three: make the practice reusable

Add:

- Assurance profiles to task specifications.
- Shared approved domain libraries.
- Standard evidence and review templates.
- Protected judge images and upgrade policy.
- Model-specific generator libraries.
- Runtime trace contracts.
- Internal trust evaluations for worker changes.

Keep these as small adopted components where possible.
The platform's core responsibility is the control plane and the evidence relation.
It need not invent every prover, policy engine, sandbox runtime, or UI.

### 19.8 Later: deepen the bridge where justified

Use Rust verification for suitable components.
Develop refinement proofs for high-consequence model/implementation seams.
Consider binary-level assurance for selected dependencies with clear semantics.
Extend runtime checking where instrumentation and state cost are manageable.

Each expansion needs a new claim, owner, and acceptance rule.
Tool availability is a reason to evaluate a route.
It is not a reason to verify unrelated work.

**Done when:** the pilots produce useful maintained evidence, and expansion has an accountable owner and a measured reason.

---

## 20. Recover when the evidence breaks

Evidence can fail because the code is wrong, the specification is wrong, the checker is wrong, or the process lost its identity chain.

The runbook must distinguish them.

### 20.1 A proof fails after a refactor

Preserve the failing goal and the prior accepted evidence.
Compare theorem types, model definitions, and imports before repairing tactics.

If the statement meaning changed, route a specification review.
If only proof structure changed, let the worker repair within the existing target.
If a new counterexample appears, inspect whether the refactor broke the intended behavior.

Do not make an admitted lemma the temporary accepted release state.
The release remains blocked until the required evidence is restored.

### 20.2 Differential tests disagree

Stop acceptance for the affected claim.
Preserve both outputs, binaries, raw input, and the generator record.

Classify:

```text
Production defect
Model defect
Adapter defect
Ambiguous specification
Unsupported input
Resource-policy mismatch
Comparison or harness defect
```

Minimize only after preserving the original.
Require the owner to resolve ambiguity.
Recheck the model if it changes.
Rebuild and retest the candidate if it changes.

### 20.3 Independent checkers disagree

Reject the evidence.
Preserve the exact export and checker identities.
Run the smallest available reproduction in isolated environments.
Check upstream advisories and compatibility requirements.

A majority of agreeing checkers does not make the disputed result acceptable.
The difference could reveal a soundness bug, unsupported feature, export issue, or checker defect.

If a known issue affects prior accepted evidence, scope the affected releases and recheck with a corrected policy.
Treat it as an assurance incident.

### 20.4 A checker security update arrives

Determine:

- Which versions and constructs are affected.
- Whether your accepted proof exports use them.
- Whether independent checkers also share the issue.
- Whether the worker or judge execution boundary is affected.
- Which releases need rechecking.

Update the negative regression suite.
Rebuild judge images and relevant exports.
Retire affected toolchains from release eligibility where necessary.

Pinning makes builds reproducible.
Security maintenance makes those reproducible builds appropriate to trust.

### 20.5 An agent attempts to weaken the claim

Retain:

```text
Original approved target
Candidate change
Worker trace and local checks
Integrity rejection
Model and worker version
Task budget and feedback
```

Determine whether the attempt was an ordinary proposed spec change, a tooling misunderstanding, or a repeated acceptance strategy.
Improve the harness and evaluation accordingly.

Never accept the weakened target merely because the old one was hard.
The owner can choose a new requirement, but it needs its own explicit approval.

### 20.6 Runtime monitoring reports drift

First establish event completeness and ordering.
Then reconstruct the relevant authoritative transitions.

If the trace is valid and the code diverges, assess the production consequence.
Contain the affected operation where the existing incident policy requires it.
Use the retained trace to create a simulation regression.

If the intended behavior has legitimately changed, review and version the new model.
A monitor patch alone does not close the incident.

### 20.7 The approved spec was wrong

This is the failure the checker cannot solve for you.

Preserve the old claim and evidence.
Explain which intended behavior it failed to express.
Add examples that distinguish the old and new meanings.
Review the corrected domain and relation.

Then:

```text
Reprove the model properties.
Repair the implementation where needed.
Rerun the bridge.
Reissue evidence.
Identify earlier releases affected by the specification defect.
```

Do not rewrite history so the old proof appears to have guaranteed the new behavior.

### 20.8 Keep a narrow escape policy

Some organizations permit emergency releases with missing assurance evidence.
If yours does, the authorized human decision should name:

```text
Missing claim
Exact candidate
Reason for the exception
Compensating evidence
Expiration or follow-up obligation
Accountable approver
```

The judge still reports incomplete evidence.
A separate release policy records the exception.
This preserves the meaning of `accepted`.

**Done when:** every failure class has an owner, retained artifacts, a stop condition, and a path back to current evidence.

---

## 21. Appendix: runnable examples and reusable contracts

The local lab has six files:

```text
factorylab/
├── lean-toolchain
├── lakefile.toml
├── Examples.lean
├── Main.lean
├── candidate.mjs
└── diff.py
```

The TLA+ model is a separate two-file lab.
The remaining templates are proposed platform contracts.

### 21.1 Pin the Lean project

Write `lean-toolchain`:

```text
leanprover/lean4:v4.36.0
```

Write `lakefile.toml`:

```toml
name = "factorylab"
version = "0.1.0"
defaultTargets = ["quadref"]

[[lean_lib]]
name = "Examples"

[[lean_exe]]
name = "quadref"
root = "Main"
```

Use an installed Lean toolchain manager or the official release distribution.
Confirm the selected version before building.
No Mathlib download is required for this lab.

### 21.2 Complete Lean model and theorem module

Write `Examples.lean`:

```lean
import Std

namespace Factory

def clamp (limit value : Nat) : Nat := min limit value

theorem clamp_bounded (limit value : Nat) :
    clamp limit value ≤ limit := by
  exact Nat.min_le_left limit value

theorem clamp_unchanged (limit value : Nat) (h : value ≤ limit) :
    clamp limit value = value := by
  exact Nat.min_eq_right h

structure Tile where
  zoom : Nat
  x : Fin (2 ^ zoom)
  y : Fin (2 ^ zoom)

inductive Quad where
  | nw | ne | sw | se
  deriving Repr, DecidableEq

def digit : Quad → Char
  | .nw => '0'
  | .ne => '1'
  | .sw => '2'
  | .se => '3'

def parseDigit : Char → Option Quad
  | '0' => some .nw
  | '1' => some .ne
  | '2' => some .sw
  | '3' => some .se
  | _ => none

def render (path : List Quad) : String :=
  String.ofList (path.map digit)

def parse (text : String) : Option (List Quad) :=
  text.toList.mapM parseDigit

theorem parse_digit (q : Quad) :
    parseDigit (digit q) = some q := by
  cases q <;> rfl

theorem parse_render (path : List Quad) :
    parse (render path) = some path := by
  simp only [parse, render, String.toList_ofList]
  induction path with
  | nil => rfl
  | cons q qs ih => simp [parse_digit, ih]

def bitX : Quad → Nat
  | .nw | .sw => 0
  | .ne | .se => 1

def bitY : Quad → Nat
  | .nw | .ne => 0
  | .sw | .se => 1

def tileXY : List Quad → Nat × Nat
  | [] => (0, 0)
  | q :: qs =>
      let rest := tileXY qs
      let scale := 2 ^ qs.length
      (bitX q * scale + rest.1, bitY q * scale + rest.2)

theorem tileXY_bounds (path : List Quad) :
    (tileXY path).1 < 2 ^ path.length ∧
    (tileXY path).2 < 2 ^ path.length := by
  induction path with
  | nil => decide
  | cons q qs ih =>
      cases q <;>
        simp [tileXY, bitX, bitY, Nat.pow_succ] at * <;>
        omega

structure Lease where
  epoch : Nat
  active : Bool
  deriving Repr, DecidableEq

def accepts (lease : Lease) (suppliedEpoch : Nat) : Bool :=
  lease.active && decide (suppliedEpoch = lease.epoch)

def revoke (lease : Lease) : Lease :=
  { epoch := lease.epoch + 1, active := false }

theorem revoked_rejects (lease : Lease) (suppliedEpoch : Nat) :
    accepts (revoke lease) suppliedEpoch = false := by
  simp [accepts, revoke]

def grant (lease : Lease) : Lease :=
  { lease with active := true }

theorem stale_epoch_rejected (lease : Lease) :
    accepts (grant (revoke lease)) lease.epoch = false := by
  simp [accepts, grant, revoke]

#print axioms clamp_bounded
#print axioms clamp_unchanged
#print axioms parse_digit
#print axioms parse_render
#print axioms tileXY_bounds
#print axioms revoked_rejects
#print axioms stale_epoch_rejected

end Factory
```

The two lease models use different transition representations.
This Lean model increments on revocation and reactivates on grant.
The TLA+ model clears ownership on revocation and increments on the next grant.
Both support a stale-epoch example, but this book supplies no equivalence proof between them.

If a production design uses one convention, adopt that convention consistently and prove the required mapping.

### 21.3 Executable Lean JSON-line adapter

Write `Main.lean`:

```lean
import Examples
import Lean

open Lean Factory

def invalid : Json :=
  Json.mkObj [
    ("ok", Json.bool false),
    ("error", Json.str "invalid_request")]

def evaluate (line : String) : Json := Id.run do
  let .ok request := Json.parse line | return invalid
  let .ok fields := request.getObj? | return invalid
  if fields.size != 1 then return invalid
  let .ok value := request.getObjVal? "quadkey" | return invalid
  let .ok key := value.getStr? | return invalid
  if key.length > 32 then return invalid
  let some path := Factory.parse key | return invalid
  let xy := tileXY path
  return Json.mkObj [
    ("ok", Json.bool true),
    ("z", toJson path.length),
    ("x", Json.str (toString xy.1)),
    ("y", Json.str (toString xy.2))]

def main : IO Unit := do
  let input ← IO.getStdin
  let output ← IO.getStdout
  repeat
    let line ← input.getLine
    if line.isEmpty then break
    output.putStrLn (evaluate line).compress
```

This adapter is ordinary executable code.
The model theorems do not prove its JSON handling.
The local lab tests that boundary.

The length limit bounds work after a JSON line has been read and parsed.
It does not prevent an attacker from sending an enormous raw line.
A production wrapper needs a byte limit and resource controls before parsing.

Build from the project directory:

```bash
lean --version
lake build
```

The executable is `.lake/build/bin/quadref`.
On a platform that appends an executable suffix, use that platform's generated path.

### 21.4 JavaScript candidate

Write `candidate.mjs`:

```javascript
import readline from 'node:readline';

const invalid = () => ({ ok: false, error: 'invalid_request' });

function evaluate(line) {
  let request;
  try {
    request = JSON.parse(line);
  } catch {
    return invalid();
  }
  if (request === null || Array.isArray(request) ||
      typeof request !== 'object' || Object.keys(request).length !== 1 ||
      typeof request.quadkey !== 'string') return invalid();

  const key = request.quadkey;
  if (key.length > 32 || !/^[0-3]*$/u.test(key)) return invalid();

  let x = 0n;
  let y = 0n;
  for (const char of key) {
    const digit = BigInt(char);
    x = (x << 1n) | (digit & 1n);
    y = (y << 1n) | ((digit >> 1n) & 1n);
  }
  return {
    ok: true,
    z: key.length,
    x: x.toString(),
    y: y.toString()
  };
}

const input = readline.createInterface({
  input: process.stdin,
  crlfDelay: Infinity
});
for await (const line of input) {
  console.log(JSON.stringify(evaluate(line)));
}
```

Using `BigInt` makes the candidate's coordinate arithmetic exact within the approved zoom range.
It also requires string serialization for JSON responses.

The input checks enforce the lab's decoded-object contract.
They are not a general security-hardened JSON transport.

### 21.5 Differential driver with a separate oracle

Write `diff.py`:

```python
import itertools
import json
import random
import subprocess
import sys

INVALID = {"ok": False, "error": "invalid_request"}


def oracle(line):
    try:
        request = json.loads(line)
    except (ValueError, TypeError):
        return INVALID
    if type(request) is not dict or set(request) != {"quadkey"}:
        return INVALID
    key = request["quadkey"]
    if (
        type(key) is not str
        or len(key) > 32
        or any(c not in "0123" for c in key)
    ):
        return INVALID
    x = sum(
        (int(c) % 2) * 2 ** (len(key) - i - 1)
        for i, c in enumerate(key)
    )
    y = sum(
        (int(c) // 2) * 2 ** (len(key) - i - 1)
        for i, c in enumerate(key)
    )
    return {"ok": True, "z": len(key), "x": str(x), "y": str(y)}


def corpus():
    rows = []
    for z in range(7):
        for digits in itertools.product("0123", repeat=z):
            rows.append(json.dumps({"quadkey": "".join(digits)}))
    rng = random.Random(20261125)
    for _ in range(2000):
        z = rng.randrange(33)
        key = "".join(rng.choice("0123") for _ in range(z))
        rows.append(json.dumps({"quadkey": key}))
    for key in [
        "0" * 32, "1" * 32, "2" * 32, "3" * 32,
        "12" * 16, "03" * 16, "0" * 33,
        "4", "-1", " 1", "1\n", "１", "😀",
    ]:
        rows.append(json.dumps({"quadkey": key}))
    rows.extend([
        "", "{", "null", "[]", "{}",
        '{"quadkey":7}',
        '{"quadkey":null}',
        '{"quadkey":"1","extra":true}',
    ])
    return rows


def validate_response(value):
    if type(value) is not dict or type(value.get("ok")) is not bool:
        raise ValueError("response must be an object with a boolean ok")
    if not value["ok"]:
        if value != INVALID:
            raise ValueError("invalid rejection response")
        return value
    if set(value) != {"ok", "z", "x", "y"}:
        raise ValueError("unexpected success fields")
    z = value["z"]
    if type(z) is not int or not 0 <= z <= 32:
        raise ValueError("invalid zoom")
    for field in ("x", "y"):
        coordinate = value[field]
        if (
            type(coordinate) is not str
            or not coordinate.isascii()
            or not coordinate.isdigit()
        ):
            raise ValueError("coordinate must be a decimal string")
        number = int(coordinate)
        if str(number) != coordinate or number >= 2 ** z:
            raise ValueError("coordinate is noncanonical or out of bounds")
    return value


def run(command, rows):
    result = subprocess.run(
        command,
        input="\n".join(rows) + "\n",
        text=True,
        capture_output=True,
        timeout=60,
        check=False,
    )
    if result.returncode != 0:
        raise RuntimeError(
            f"{command[0]} exited {result.returncode}: "
            f"{result.stderr[-2000:]}"
        )
    lines = result.stdout.splitlines()
    if len(lines) != len(rows):
        raise RuntimeError(
            f"{command[0]} returned {len(lines)} rows; "
            f"expected {len(rows)}"
        )
    try:
        return [validate_response(json.loads(line)) for line in lines]
    except ValueError as exc:
        raise RuntimeError(
            f"{command[0]} violated the JSON-line protocol"
        ) from exc


def main():
    if len(sys.argv) != 3:
        raise SystemExit(
            "usage: python3 diff.py MODEL_EXECUTABLE CANDIDATE_MJS"
        )
    rows = corpus()
    expected = [oracle(row) for row in rows]
    model = run([sys.argv[1]], rows)
    candidate = run(["node", sys.argv[2]], rows)
    for index, (row, want, ref, actual) in enumerate(
        zip(rows, expected, model, candidate)
    ):
        if ref != want or actual != want:
            print(json.dumps({
                "index": index,
                "input": row,
                "expected": want,
                "model": ref,
                "candidate": actual,
            }, ensure_ascii=True))
            raise SystemExit(1)
    print(
        f"PASS: {len(rows)} requests; seed=20261125; "
        "exhaustive zooms=0..6"
    )


if __name__ == "__main__":
    main()
```

Run:

```bash
python3 diff.py .lake/build/bin/quadref candidate.mjs
```

The checked result is:

```text
PASS: 7482 requests; seed=20261125; exhaustive zooms=0..6
```

The driver exits unsuccessfully on divergence, process failure, timeout, invalid JSON output, the wrong response shape, or the wrong response count.
It reports the first semantic divergence.
It is a local demonstration, not a production attestation service.

To adapt it to an SDK, replace the candidate subprocess with a narrow adapter for that SDK.
Keep the request and response contract stable.

### 21.6 Demonstrate a representation bug

Make a temporary copy of the candidate.
Change its `BigInt` arithmetic to ordinary JavaScript numeric bitwise arithmetic.
Keep the response protocol the same.

For example, use:

```javascript
let x = 0;
let y = 0;
for (const char of key) {
  const digit = Number(char);
  x = (x << 1) | (digit & 1);
  y = (y << 1) | ((digit >> 1) & 1);
}
```

The driver rejects this candidate.
One retained case is:

```json
{"quadkey":"12231123331231211130100201000331"}
```

The correct X coordinate is `2649614407`.
The broken candidate returns `-1645352889`.
The negative fixture confirms that the test distribution reaches the 32-bit representation boundary.

Keep the correct candidate as the lab's default.
Mutants are fixtures whose expected outcome is rejection.

### 21.7 Complete TLA+ lease model

Write `Lease.tla`:

```tla
------------------------------ MODULE Lease ------------------------------
EXTENDS Naturals, FiniteSets

CONSTANTS Workers, NoWorker, MaxEpoch
VARIABLES epoch, owner, cancelled, completed, issued, acceptedEpoch

vars == <<epoch, owner, cancelled, completed, issued, acceptedEpoch>>
Credential == [worker : Workers, epoch : 1..MaxEpoch]

Init == /\ epoch = 0
        /\ owner = NoWorker
        /\ cancelled = FALSE
        /\ completed = FALSE
        /\ issued = {}
        /\ acceptedEpoch = 0

Grant(w) == /\ owner = NoWorker
            /\ ~cancelled /\ ~completed
            /\ epoch < MaxEpoch
            /\ epoch' = epoch + 1
            /\ owner' = w
            /\ issued' =
                 issued \cup {[worker |-> w, epoch |-> epoch + 1]}
            /\ UNCHANGED <<cancelled, completed, acceptedEpoch>>

Revoke == /\ owner # NoWorker
          /\ ~completed
          /\ owner' = NoWorker
          /\ UNCHANGED <<epoch, cancelled, completed,
                         issued, acceptedEpoch>>

Cancel == /\ ~cancelled /\ ~completed
          /\ cancelled' = TRUE
          /\ owner' = NoWorker
          /\ UNCHANGED <<epoch, completed, issued, acceptedEpoch>>

Commit(c) == /\ ~cancelled /\ ~completed
             /\ owner = c.worker
             /\ c.epoch = epoch
             /\ completed' = TRUE
             /\ acceptedEpoch' = c.epoch
             /\ UNCHANGED <<epoch, owner, cancelled, issued>>

Next == (\E w \in Workers : Grant(w)) \/ Revoke \/ Cancel
        \/ (\E c \in issued : Commit(c))

Spec == Init /\ [][Next]_vars

TypeOK == /\ epoch \in 0..MaxEpoch
          /\ owner \in Workers \cup {NoWorker}
          /\ cancelled \in BOOLEAN
          /\ completed \in BOOLEAN
          /\ issued \subseteq Credential
          /\ acceptedEpoch \in 0..MaxEpoch

NoStaleCommit == completed => acceptedEpoch = epoch
CancelledHasNoOwner == cancelled => owner = NoWorker
=============================================================================
```

Write `Lease.cfg`:

```text
CONSTANTS
  Workers = {w1, w2}
  NoWorker = noWorker
  MaxEpoch = 3

SPECIFICATION Spec
INVARIANTS TypeOK NoStaleCommit CancelledHasNoOwner
CHECK_DEADLOCK FALSE
```

Use the official `tla2tools.jar`.
The local validation used the v1.7.4 distribution, whose TLC banner identifies version 2.19.

```bash
java -XX:+UseParallelGC -cp tla2tools.jar tlc2.TLC -workers 1 Lease.tla
```

The checked configuration produces:

```text
72 states generated
58 distinct states found
0 states left on queue
No error found
```

To run the negative fixture, make a separate module and matching configuration or edit a temporary copy.
Remove the epoch equality conjunct from `Commit`.
TLC should report a violation of `NoStaleCommit`.

The model bound prevents further grants after epoch three.
It never wraps an epoch.
This avoids proving an artificial wraparound design safe while still keeping the exploration finite.

### 21.8 A specification-approval template

```yaml
schema: specification-approval/v1
claim_id: "<stable identifier>"
owner: "<accountable domain owner>"
intent: "<observable promised behavior>"
input_domain: ["<required inputs and limits>"]
preconditions: ["<explicit assumptions>"]
postconditions: ["<required output or state relation>"]
required_examples: ["<acceptance and rejection cases>"]
sanity_checks: ["<nontriviality and input coverage checks>"]
formal_targets: ["<qualified declarations or model properties>"]
semantic_dependencies: ["<approved definition and import identities>"]
trusted_assumptions: ["<axioms, externals, machine semantics>"]
implementation_bridge: "<proof or tested relation>"
excluded_claims: ["<important unpromised behavior>"]
approval:
  revision: "<revision>"
  bundle_digest: "<digest>"
  approver: "<identity>"
```

The approval service supplies and authenticates the last block.
The worker cannot self-approve.

### 21.9 A proof-worker instruction template

```text
Implement and prove the required targets in the approved task bundle.
Preserve its input domain, theorem meanings, and trusted assumptions.
Edit only the permitted candidate files.
Use verifier feedback and approved reusable lemmas.
Record incomplete goals and counterexamples.

If the specification appears wrong or ambiguous, submit a separate
specification-change proposal with distinguishing examples.
Do not treat that proposal as approved.

Local success is progress.
Only the protected judge can accept the submission.
```

Enforce the corresponding filesystem and authority boundaries in the harness.

### 21.10 An integrity-fixture inventory

| Fixture | Expected result |
|---|---|
| Correct proof of approved statement | Accept |
| Required theorem admitted with `sorry` | Reject |
| New indirect axiom | Reject |
| Required theorem removed | Reject |
| Familiar notation redefined to weaken the target | Reject statement mismatch |
| Referenced definition changed | Reject unapproved semantic change |
| Preconditions narrowed without approval | Reject |
| Native-computation assumption forbidden by policy | Reject |
| Worker replaces its local checker | Protected verdict remains unaffected |
| Old export paired with new source | Reject identity mismatch |
| Old verdict replayed for another candidate | Reject identity mismatch |
| Checker timeout | Incomplete; block |
| Checker disagreement | Reject |
| Build attempts forbidden network or credential access | Contain and reject execution policy |
| Candidate differs from model | Reject implementation bridge |

A fixture may be rejected by more than one check.
Record which boundary detected it.
This helps diagnose missing defenses rather than merely counting red statuses.

### 21.11 A review-packet template

```text
Behavior and scope:
Approved claims and bundle revision:
Specification or assumption changes:
Candidate source and executable identities:
Proof and model-checking results:
Implementation bridge and retained divergences:
Runtime observation scope where applicable:
Reviewer decisions:
Known limits and unresolved evidence:
```

Keep the full implementation diff and evidence records accessible.
The packet helps direct attention; it does not replace the underlying material.

### 21.12 A release exception template

```yaml
schema: assurance-exception/v1
candidate_digest: "<exact candidate>"
missing_claims: ["<claim identifiers>"]
reason: "<why the authorized release policy permits an exception>"
compensating_evidence: ["<concrete records>"]
expires: "<time or release condition>"
follow_up_owner: "<identity>"
authorized_approver: "<identity>"
```

Use this only if the organization's existing release policy permits it.
The proof verdict stays incomplete.
The exception does not change what the checker established.

---

## 22. Sources and verification notes

The source material for this book is the supplied report, *Lean and Formal Verification in an AI Software Factory: A Deep Research Report for an AI Platform Team*, dated November 2026.
The manuscript turns its recommendations into an operating model and local labs.
It does not reproduce every ecosystem, benchmark, funding, or product claim.

Primary pages and project documentation were checked on November 25, 2026.
Company case studies remain company-reported evidence.
Benchmark results remain results under their authors' protocols.
The architecture, contracts, and rollout criteria are recommendations developed here.

### 22.1 What was checked locally

| Artifact | Validation performed | Scope |
|---|---|---|
| `Examples.lean` | Lean 4.36.0 build and transitive axiom reports | The listed declarations and bundled dependencies |
| `Main.lean` | Compiled with the model into `quadref` | Executable lab, including JSON adapter |
| JavaScript candidate and Python driver | Differential run of 7,482 requests | Recorded corpus with seed 20261125 |
| Numeric-bitwise JavaScript mutant | Rejected by the same driver | Demonstrated high-zoom representation defect |
| `Lease.tla` and `Lease.cfg` | TLC exploration with two workers and maximum epoch three | 58 distinct reachable states; configured invariants |
| Lease model without epoch equality | Rejected by TLC | Demonstrated stale-credential counterexample |
| Lean checker command guidance | Inspected `lake help check` and `lake help comparator` from 4.36.0 | Command behavior and requirements |
| Markdown manuscript | Structural, internal-link, reference, and whitespace checks | Book and repository index |

The local machine was macOS on ARM64.
The differential lab used Node.js 24 and Python 3.
TLC used the official v1.7.4 tools distribution on Java 17.

The theorem axiom reports contain only `propext`, `Classical.choice`, and `Quot.sound`, with smaller subsets for some declarations.
The local proof build is not an execution of the proposed protected Linux release judge.
The built-in `lake check` and comparator sandbox workflow, microVM containment, CI enforcement, evidence signing, Cedar integration, SDK adapters, and production monitoring remain implementation work for the adopting team.

The Kani fragment was not run.
The SQL, Cedar, comparator configuration, task schemas, and attestation templates are integration examples with explicit surrounding requirements.
They are not claimed as a deployed platform.

### 22.2 Important corrections to the supplied research

**Kernel soundness changed during 2026.**
The August postmortem documents an official Lean kernel bug yielding a proof of `False`.
The book therefore replaces the report's earlier no-known-proof-of-`False` statement with a current-version and independent-checking policy.

**Comparator guidance changed.**
The older comparator repository is archived.
The pinned 4.36.0 toolchain includes `lake check` and `lake comparator`.
The book uses their actual help to explain sandbox, export, configuration, and statement-comparison boundaries.

**A proved model is not automatically a proved production implementation.**
The Cedar-style differential bridge is valuable empirical evidence.
This book labels it accordingly.
Universal conformance requires an additional proof or another justified verification route.

**A non-widening policy relation needs its own analysis.**
Using Cedar does not automatically establish inclusion of allowed requests for arbitrary policy changes.
The schema, request domain, entity assumptions, supported analysis fragment, and enforcement path are part of the pilot.

**Cedar errors need an explicit application decision.**
Cedar uses skip-on-error semantics.
The proposed security pilot adds a reviewed broker rule about diagnostics instead of assuming every error denies.

**Geospatial novelty is unestablished.**
The report's failure to find particular verified algorithms is a research lead.
The manuscript proposes useful artifacts without claiming a complete absence of prior work.

### 22.3 Claims deliberately left as research leads

The report names many AI provers, tool integrations, floating-point libraries, commercial verification services, and additional industrial projects.
They can be useful candidates.
This book does not certify their current compatibility, completeness, pricing, funding, or performance.

Before adoption:

```text
Inspect the actual artifact and version.
Identify its supported semantics and theorem inventory.
Audit admitted proofs and external assumptions.
Reproduce a representative result.
Run the team's negative integrity fixtures.
Assign an owner for upgrades and failure handling.
```

The same standard applies to a local model generated by an agent.
Public claims and familiar project names are starting evidence.
The release process needs identified artifacts and checked properties.

### 22.4 Primary sources

The footnotes below identify the sources used in the text.
URLs are written as literal addresses so the references remain unambiguous when copied or converted.
Dynamic documentation should be read alongside the pinned release used by the adopting team.

[^anthropic]: Anthropic and Theorem.
    *LLMs are making formal software verification feasible at a much larger scale.*
    October 22, 2026; the page also records a November update.
    Primary company account of the OSS-Fuzz proofs, interpreter experiment, cost limits, and proof sharing.
    URL: `https://www.anthropic.com/research/formal-verification`.

[^cedar-aws]: AWS Open Source Blog.
    *Lean Into Verified Software Development.*
    Primary account of Cedar's executable Lean models, proofs, differential random testing, and release discipline.
    Performance figures in that post describe its measured setup.
    URL: `https://aws.amazon.com/blogs/opensource/lean-into-verified-software-development/`.

[^cedar-spec]: Cedar policy project.
    *Cedar Specification* repository.
    Executable formalization, proof source, Lean command-line and FFI components, and differential-testing infrastructure.
    Also consulted the linked Amazon Science article, *How we built Cedar with automated reasoning and differential testing*, for the development approach.
    URLs: `https://github.com/cedar-policy/cedar-spec` and `https://www.amazon.science/blog/how-we-built-cedar-with-automated-reasoning-and-differential-testing`.

[^cedar-auth]: Cedar Policy Language Reference Guide.
    *Authorization.*
    Official decision semantics, including default deny, forbid precedence, diagnostics, and skip on error.
    URL: `https://docs.cedarpolicy.com/auth/authorization.html`.

[^vericoding]: Sergiu Bursuc and colleagues.
    *A benchmark for vericoding: formally verified program synthesis.*
    2025, arXiv:2509.22908.
    Primary benchmark account; approximate aggregate figures in chapter 2 follow the abstract rather than implying comparability with later agentic studies.
    URL: `https://arxiv.org/abs/2509.22908`.

[^verina]: Zhe Ye, Zhengxu Yan, Jingxuan He, Timothe Kasriel, Kaiyu Yang, and Dawn Song.
    *VERINA: Benchmarking Verifiable Code Generation.*
    arXiv:2505.23135, revised March 16, 2026.
    The cited code, specification, and proof percentages are separately reported single-trial outcomes.
    URL: `https://arxiv.org/abs/2505.23135`.

[^clever-audit]: Alessandro Sosso, Akhil Arora, and Bas Spitters.
    *Isomorphism Is Not Enough: An Agentic Audit of Verified Code Generation in Lean.*
    arXiv:2605.23772, version 2, October 30, 2026.
    Source for the reference-specification findings and sound-premise subset results.
    URL: `https://arxiv.org/abs/2605.23772v2`.

[^kleppmann]: Martin Kleppmann.
    *Prediction: AI will make formal verification go mainstream.*
    December 8, 2025.
    An expert argument about changing economics and the role of checking, rather than a local ROI measurement.
    URL: `https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html`.

[^demoura-factory]: Leonardo de Moura.
    *When AI Writes the World's Software, Who Verifies It?*
    February 28, 2026.
    Architectural argument for scalable verification and precise specifications in an agent-driven software workflow.
    URL: `https://leodemoura.github.io/blog/2026-2-28-when-ai-writes-the-worlds-software-who-verifies-it/`.

[^lean-reference]: Lean project.
    *The Lean Language Reference* and *Theorem Proving in Lean 4.*
    Official descriptions of elaboration, types, declarations, proof construction, and the language's logical foundations.
    URLs: `https://lean-lang.org/doc/reference/latest/` and `https://lean-lang.org/theorem_proving_in_lean4/`.

[^lean-axioms]: Lean project.
    *The Lean Language Reference: Axioms* and *Theorem Proving in Lean 4: Axioms and Computation.*
    Sources for foundational axioms, `sorryAx`, transitive axiom inspection, and compiler-dependent native proof assumptions.
    URLs: `https://lean-lang.org/doc/reference/latest/Axioms/` and `https://lean-lang.org/theorem_proving_in_lean4/Axioms-and-Computation/`.

[^lean-release]: Lean project.
    Lean 4.36.0 release, published November 16, 2026.
    The manuscript's command behavior was checked directly using this release's `lake help check` and `lake help comparator`.
    Those help texts explain Linux sandbox requirements, explicit axiom policy, named statement comparison, checker selection, exit codes, and export-identity limits.
    URL: `https://github.com/leanprover/lean4/releases/tag/v4.36.0`.

[^comparator]: Lean project.
    Standalone `leanprover/comparator` repository.
    The page records its archival on October 20, 2026.
    Consult the pinned Lean release for the newer built-in commands.
    URL: `https://github.com/leanprover/comparator`.

[^demoura-watches]: Leonardo de Moura.
    *Who Watches the Provers?*
    March 16, 2026.
    Discussion of AI-assisted exploit discovery, trust boundaries, and independent proof checking.
    URL: `https://leodemoura.github.io/blog/2026-3-16-who-watches-the-provers/`.

[^kernel-bug]: Leonardo de Moura.
    *Postmortem for Kernel Soundness Bug #14576.*
    August 1, 2026.
    Primary account of the nested-inductive-type flaw, its reduction to `False`, fixes, and the independent-checker complications.
    URL: `https://leodemoura.github.io/blog/2026-8-1-postmortem-for-kernel-soundness-bug-14576/`.

[^verified-kernels]: Leonardo de Moura.
    *Lean Now Ships with Verified Kernels.*
    October 15, 2026.
    Primary description of the newer checking workflow and con-leche/con-ron assurance arguments, including their remaining assumptions.
    URL: `https://leodemoura.github.io/blog/2026-10-15-lean-now-ships-with-verified-kernels/`.

[^fetch]: Datadog engineering.
    *How we continuously validate formal specifications in production.*
    November 5, 2026.
    Primary account of FETCh, observable specifications, the Husky discrepancy, instrumentation needs, and supported-subset limitations.
    URL: `https://www.datadoghq.com/blog/engineering/continuous-formal-spec-validation/`.

[^verus]: Verus project.
    *Verus Tutorial and Reference: Verus overview.*
    Official description of its Rust verification approach, SMT use, supported scope, and compiler exclusions.
    URL: `https://verus-lang.github.io/verus/guide/`.

[^kani]: Kani project.
    *The Kani Rust Verifier: Getting started.*
    Official description of proof harnesses, safety and correctness checks, resource outcomes, and supported-feature limits.
    URL: `https://model-checking.github.io/kani/`.

[^aeneas]: AeneasVerif project.
    Official repository and project site.
    Sources for the Rust-to-functional verification pipeline and supported proof-assistant backends.
    URLs: `https://github.com/AeneasVerif/aeneas` and `https://aeneasverif.github.io/`.

[^aws-tla]: Chris Newcombe and colleagues.
    *How Amazon Web Services Uses Formal Methods.*
    Communications of the ACM, 2015.
    Primary industrial account of design-level formal specifications and model checking at AWS.
    Author-hosted paper: `https://lamport.azurewebsites.net/tla/formal-methods-amazon.pdf`.
    TLC tooling source: `https://github.com/tlaplus/tlaplus`.

[^mvt]: Mapbox.
    *Vector Tile Specification*, version 2.1.
    Primary format specification for the proposed codec work.
    The book implements quadkey decoding, not an MVT verifier.
    URL: `https://github.com/mapbox/vector-tile-spec/tree/master/2.1`.
