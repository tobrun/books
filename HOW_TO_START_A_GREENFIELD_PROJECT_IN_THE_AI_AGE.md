# How to Start a Greenfield Project in the AI Age

## A Practical Book on Locked Foundations, Generated Guardrails, and Autonomous Build-Out

An empty repository gives you freedom.
It also gives every generated file the opportunity to become a precedent.

The first shortcut becomes the next agent's example.
The first ambiguous requirement becomes an implementation assumption.
The first test that passes without exercising anything becomes evidence that the project is healthy.
By the time these mistakes are visible, they may have been copied across the codebase.

AI makes it easier to create the system you described.
That puts more responsibility on describing the right system, designing its boundaries, and knowing whether it works.

> Build a foundation that makes important mistakes difficult, makes failures visible, and makes successful work easy to verify.

This book develops that foundation.
It begins with a product contract, turns architectural decisions into enforceable rules, protects the verification system, and establishes a repeatable loop for building features.
Autonomy grows as that loop demonstrates reliable outcomes.

The approach applies whether one developer uses an assistant in a terminal or a team dispatches many background tasks.
The product itself does not need to contain AI.
The subject is building software with AI assistance.

The examples use a fictional shared request tracker.
People submit requests inside a workspace, members assign them, and authorized members resolve or reopen them.
The product is deliberately ordinary.
Its authorization, persistence, retries, and user journeys expose the engineering problems that attractive demos often hide.

The examples describe contracts and proposed interfaces.
They are not configuration for a particular product, language, framework, or agent.
Choose implementations appropriate to your environment.
Keep the outcomes and verification obligations.

---

## Contents

1. [Understand what changes when code becomes cheap](#1-understand-what-changes-when-code-becomes-cheap)
2. [Define the product and its risks](#2-define-the-product-and-its-risks)
3. [Separate foundations from autonomous build-out](#3-separate-foundations-from-autonomous-build-out)
4. [Choose a small stack and enforceable principles](#4-choose-a-small-stack-and-enforceable-principles)
5. [Design boundaries that agents can follow](#5-design-boundaries-that-agents-can-follow)
6. [Make the architecture executable](#6-make-the-architecture-executable)
7. [Turn decisions into the right checks](#7-turn-decisions-into-the-right-checks)
8. [Test the rules before trusting them](#8-test-the-rules-before-trusting-them)
9. [Protect the foundation and its verifier](#9-protect-the-foundation-and-its-verifier)
10. [Make the application runnable and observable](#10-make-the-application-runnable-and-observable)
11. [Build a layered verification system](#11-build-a-layered-verification-system)
12. [Keep tests independent and challenge their strength](#12-keep-tests-independent-and-challenge-their-strength)
13. [Create a feature map and an evidence contract](#13-create-a-feature-map-and-an-evidence-contract)
14. [Give agents a map of the repository](#14-give-agents-a-map-of-the-repository)
15. [Run a reliable coding session](#15-run-a-reliable-coding-session)
16. [Decompose work before increasing parallelism](#16-decompose-work-before-increasing-parallelism)
17. [Control entropy as the project grows](#17-control-entropy-as-the-project-grows)
18. [Expand autonomy through measured outcomes](#18-expand-autonomy-through-measured-outcomes)
19. [Separate deployment, release, and recovery](#19-separate-deployment-release-and-recovery)
20. [Build the foundation in the first weeks](#20-build-the-foundation-in-the-first-weeks)
21. [Recognize the traps before they become habits](#21-recognize-the-traps-before-they-become-habits)
22. [Appendix: working contracts and checklists](#22-appendix-working-contracts-and-checklists)

---

## 1. Understand what changes when code becomes cheap

The price of producing a plausible implementation has fallen.
The work required to decide whether that implementation belongs in the product has not disappeared.

A generated patch can be internally consistent and still solve the wrong problem.
It can pass local tests while violating a workspace boundary.
It can follow the repository's existing patterns while reproducing a mistake that nobody intended to standardize.

The scarce resources become clearer:

- Attention to requirements.
- Architectural judgment.
- Independent evidence.
- Review capacity.
- Operational ownership.

Treat those resources as design constraints.
A workflow that generates more code than you can understand or verify is accumulating unfinished work.

### 1.1 The repository teaches the agent

An agent sees more than your instructions.
It sees names, imports, tests, comments, examples, and the shape of nearby code.
Those signals suggest how a change should be made.

Suppose the first screen in the request tracker writes directly to storage.
A later agent is asked to add a second screen.
The first implementation offers an obvious pattern to copy.
Your architecture document may say that writes belong in an application service, but the working code demonstrates a different convention.

Resolve that contradiction at the source.
Move the write behind the intended interface.
Make a direct storage import fail.
Leave one clear example of the correct path.

This is why the earliest implementation deserves disproportionate attention.
It is both product behavior and teaching material.

### 1.2 Prefer the strongest useful control

There are several ways to prevent a recurring mistake.
Their strength depends on what they can actually establish.

| Control | What it contributes | Where it falls short |
|---|---|---|
| API and module design | Makes invalid operations unavailable or difficult to express | Cannot represent every business constraint |
| Types and build boundaries | Rejects incompatible shapes and forbidden dependencies | Cannot prove all runtime behavior |
| Static analysis | Detects prohibited code patterns before execution | Can miss semantic behavior or produce false positives |
| Runtime tests and observations | Exercises behavior under specified conditions | Covers only the cases and environments examined |
| Procedures and instruction files | Explains intent and how to work | Depends on the agent applying the guidance |
| Human judgment | Resolves ambiguity and evaluates consequences | Has limited throughput and attention |

Use the strongest control that fits the property.
Do not add a lint rule when removing an unnecessary public method eliminates the mistake.
Do not use an import rule to claim that authorization is correct.
Do not ask a prompt to enforce a filesystem permission.

The controls complement one another.
A good interface reduces the number of tests you need to write.
A good test exposes an interface that allows too many invalid states.
A human review discovers a missing invariant that can become a mechanical check.

### 1.3 Optimize for accepted outcomes

Code volume, task count, and proposed changes are activity measures.
They are useful for understanding workload.
They are weak measures of product progress.

A better unit is a behavior that has been implemented, verified, accepted, and preserved.

For the request tracker, “reopening a resolved request works” means more than a new button exists.
The action must respect permissions, persist the state transition, survive a reload, and produce the expected audit record.
It must also leave neighboring behavior intact.

Measure the whole outcome.
If one fast implementation creates several days of verification and repair, the generation step has not made the project faster.

### 1.4 Preparation becomes more valuable

When execution is cheap, weak preparation can generate a large amount of wrong work quickly.
Clear preparation prevents that amplification.

Before requesting implementation, determine:

1. Which user outcome matters.
2. Which constraints must remain true.
3. What evidence would establish success.
4. What the agent may change.
5. What happens if the attempt fails.

This is a compact engineering brief.
It should be proportionate to the task.
A wording change may need one sentence and a visual check.
A new authorization path needs explicit actors, denied cases, and tests.

The aim is enough precision to remove consequential guessing.

---

## 2. Define the product and its risks

A greenfield project should begin with an account of the user's job.
Starting with a stack or a directory tree reverses the dependency.
Those choices should serve an outcome you can name.

For the running example:

> A workspace member can record a request, give it a clear owner, and see whether it has been resolved.

That statement is still too broad for implementation.
Turn it into a small contract.

### 2.1 Write a product contract

```yaml
product: shared-request-tracker

user_job:
  keep requests visible and owned until resolution

first_release:
  - a member creates a request in their workspace
  - an authorized member assigns an owner
  - an authorized member resolves or reopens a request
  - members see the current state after reloading

excluded:
  - external notification delivery
  - custom workflow designers
  - cross-workspace reporting
  - automated resolution

critical_invariants:
  - a member cannot read or change another workspace's requests
  - an accepted state change is durably recorded
  - an unauthorized change has no effect
  - a rejected change leaves the prior state intact

ownership:
  product_decisions: product-owner
  architecture: engineering-owner
  release_and_recovery: service-owner
```

The role labels are placeholders.
A small project may assign every role to one person.
The important property is that responsibility is visible.

The exclusions matter as much as the feature list.
They stop an agent from interpreting a request tracker as permission to build a notification platform or a workflow engine.

### 2.2 Specify ambiguous behavior

“Members can resolve requests” conceals several decisions.
Can any member resolve any request?
Must a request have an owner first?
Can a request be resolved twice?
What happens if two people change it at the same time?

Write down the decisions before the implementation chooses them by accident.

For the example, use these rules:

- Requests have `open`, `assigned`, and `resolved` states.
- Assignment moves an open request to assigned.
- Resolution is allowed from open or assigned.
- Reopening moves a resolved request to open and clears its owner.
- Only members with the relevant permission may change state.
- A stale update is rejected rather than silently overwriting a newer change.

These rules are illustrative.
A different product may retain the owner on reopening or permit a different transition.
What matters is that the choice is explicit and testable.

### 2.3 Separate product risk from implementation difficulty

A change can be easy to write and dangerous to get wrong.
Removing a workspace filter may take one line.
Its consequence can be exposure of other users' data.

A visually complex screen may have much lower consequences if it operates on public information and can be reverted immediately.

Assess risk through questions such as:

| Question | Why it matters |
|---|---|
| Whose data or authority is involved? | Determines isolation and authorization obligations |
| Can the action be undone? | Determines approval and recovery requirements |
| Does failure become visible immediately? | Determines monitoring and evidence needs |
| Can retries repeat an external effect? | Determines idempotency and delivery design |
| Does the change alter durable data? | Determines migration and rollback planning |
| What is the impact of an incorrect result? | Determines verification depth |

Use the answers to choose controls.
Do not assign the same verification burden to every file because that is easier to configure.

### 2.4 Express operational requirements as observations

Words such as fast, reliable, and secure do not establish acceptance.
Describe how the system will be observed.

For example:

> Under the agreed test load and representative dataset, the request list meets the approved response-time budget.

This requires the project to define the load, dataset, budget, and measurement method.
It does not invent a universal latency target.

For durability:

> After the service acknowledges a state change, a fresh read from the authoritative store shows that change.

For authorization:

> A member from another workspace cannot obtain request content through the user interface, direct API calls, or guessed identifiers.

For recovery:

> The operator can restore the service using the documented procedure and demonstrate that the agreed recovery objectives were met.

These statements tell you what evidence the scaffold must make available.

### 2.5 Keep an uncertainty register

Some decisions cannot be settled before work begins.
Preserve them as explicit questions.

```yaml
question: must request history be retained after account deletion?
owner: product-owner
needed_before: deletion-feature
current_constraint: do not implement destructive deletion
evidence_needed: approved retention policy
```

An unresolved question should constrain dependent work.
It should not prevent independent work from continuing.

Agents may research options and prepare alternatives.
They should not silently turn uncertainty into policy.

**Exercise.**
Write the first-release contract in one page.
Then list three failures that would make a successful demo unacceptable.
Those failures are candidates for the earliest acceptance tests.

---

## 3. Separate foundations from autonomous build-out

Use two main operating phases, preceded by intake.

During intake, discover the product, constraints, owners, and unresolved decisions.
During foundation work, establish the environment and controls.
During build-out, let agents implement bounded changes inside that approved foundation.

The distinction concerns authority.
It does not require humans to handwrite the foundation.
An agent can generate most of its files and tests.
A responsible person remains accountable for what becomes policy.

### 3.1 Use gates with concrete outputs

A gate should answer a decision and produce evidence.
“Architecture reviewed” is too vague.
“The dependency policy is approved, its generated configuration is reproducible, and a forbidden dependency fails the installed check” is useful.

| Gate | Decision | Required output |
|---|---|---|
| Product | What are we building and excluding? | Product contract, risks, acceptance outline |
| Principles and stack | Which defaults and constraints apply? | Toolchain declaration, dependency policy, enforceable principles |
| Architecture | Where do responsibilities and authority belong? | Module map, public interfaces, dependency policy |
| Rules and verification | Which properties block acceptance? | Tested checks, runtime procedures, evidence format |
| Foundation lock | Who can change the controls? | Protected surfaces, trusted verification route, recovery procedure |

Each gate should be small enough to review.
Avoid presenting hundreds of generated files as one approval request.

### 3.2 The lock is a change policy

Locked foundations can evolve.
The lock establishes how they evolve.

Ordinary feature work may use the approved interfaces and add implementation details.
A change to the dependency graph, required acceptance checks, or protected-path policy follows a foundation change procedure.

That procedure should show:

1. Why the old decision no longer fits.
2. Which behaviors and guarantees change.
3. How the enforcement and documentation change together.
4. What verifies the new foundation.
5. Who accepts the consequences.

This prevents a failing test from becoming permission to lower the quality bar.
It also prevents a mistaken rule from becoming permanent merely because it was approved early.

### 3.3 Prove rejection before granting autonomy

Every important guardrail needs a deliberate failure demonstration.

Attempt a prohibited import.
Submit malformed data.
Run a forbidden state transition.
Remove a required test from a disposable verification copy.
Try to modify a protected rule through the ordinary implementation path.

The expected result is a visible rejection at the intended boundary.
Record what rejected the attempt and why.

Use a separate temporary copy for these challenges.
Do not damage the baseline that future sessions depend on.
Clean up the planted violations and run the valid baseline afterward.

A guardrail that has only produced green output has not yet demonstrated that it guards anything.

### 3.4 Allow experiments without confusing them with foundations

An exploratory prototype can answer whether an interaction is useful or an integration is feasible.
Label it accordingly.

Give the experiment a question, a time limit, and a disposal or promotion decision.
Do not let its directory structure become the architecture merely because it already exists.

When promoting an experiment:

- Reassess its dependencies.
- Remove temporary credentials and shortcuts.
- Put behavior behind the intended interfaces.
- Add the checks appropriate to the product risk.
- Reproduce the result from a clean environment.

The experiment establishes feasibility.
The promotion process establishes suitability.

### 3.5 Stop foundation work at a usable boundary

Preparation can become avoidance.
You do not need a complete internal platform before the first feature.

The foundation is sufficient for initial supervised work when:

- The product's first useful outcome is clear.
- The application can start reproducibly.
- Important dependency boundaries are enforced.
- Critical acceptance criteria have a trusted verification route.
- The agent's authority is explicit.
- A failed attempt can be inspected and recovered.

Then implement the first vertical slice.
Use what it teaches you to improve the foundation.

**Exercise.**
For each gate, write one artifact and one rejection demonstration.
If a gate produces only prose, identify whether it needs an executable check or whether it is a human judgment gate.

---

## 4. Choose a small stack and enforceable principles

The stack should make the product legible, testable, and maintainable.
Familiarity to the team is valuable because the team must judge generated work and operate the result.

Choose fewer overlapping tools.
Use one formatter, one primary dependency-boundary mechanism for each relevant build unit, and one standard route for verification.
Add another tool when it covers a property the existing route cannot establish.

### 4.1 Evaluate tools by the state they own

The question is broader than whether a tool can generate a starter project.
Determine what it will own after the starter project exists.

| Concern | Decision to make |
|---|---|
| Runtime | Supported version, upgrade route, operational owner |
| Dependencies | Approved sources, lock behavior, update policy |
| Storage | Durability, migrations, backup and restore |
| Application framework | Validation, error handling, lifecycle, testability |
| Verification | Deterministic entry point, discovery checks, evidence |
| Deployment | Artifact identity, configuration, rollback |
| Agent execution | Workspace isolation, tools, credentials, limits |

Avoid distributing one concern across several declarations without a clear owner.
Two configurations that both claim to define architectural boundaries will eventually disagree unless one is generated from the other.

### 4.2 Write principles that constrain decisions

A principle should explain an outcome and its failure mode.
It should help someone choose between implementations.

Weak:

> Use clean architecture.

Useful:

> Domain rules run without network or storage access so they can be tested independently and reused across entry points.

Weak:

> Make errors consistent.

Useful:

> Expected business rejections use a typed result that adapters translate into user-facing responses.
> Unexpected failures retain diagnostic context without exposing confidential data.

Weak:

> Do not write bad comments.

Useful:

> Comments explain constraints and non-obvious decisions.
> They do not grant exceptions to architecture or verification policy.

The useful versions suggest concrete controls.
They also preserve room for judgment.

### 4.3 Classify principles by enforcement

Do not pretend that every belief can become a lint rule.

| Principle | Suitable control |
|---|---|
| Domain code has no infrastructure dependency | Build graph or import checker |
| External input is validated before becoming trusted domain data | Boundary API design, type checks, parser tests |
| Expected failures have a defined representation | Types, public contract tests |
| Confidential fields do not enter ordinary logs | Restricted logging API, dataflow checks where practical, integration tests |
| Changes remain easy to review | Task decomposition and review judgment |
| Interfaces feel clear to users | User journeys, visual review, usability evaluation |

Some decisions combine several controls.
Record that combination.
Do not force an awkward static rule onto a semantic property just to fill an “enforced by” field.

### 4.4 Constrain hazardous capabilities at their entry points

Low-level operations often require care:

- Starting processes.
- Scheduling retries.
- Writing files.
- Issuing network requests.
- Opening transactions.
- Emitting logs.

Give product code a narrow interface that captures the required policy.
For example, a request client can supply deadlines, validated responses, bounded retries, and classified errors.

Keep raw access inside the adapter that owns those concerns.
The rest of the application depends on the safer contract.

A blanket ban is rarely the complete design.
If the low-level capability is necessary, someone must implement and verify its approved use.

### 4.5 Keep dependency decisions proportionate

Every dependency introduces maintenance and compatibility responsibilities.
Every replacement you write introduces responsibilities too.

A small transparent helper may be easier to own than a large dependency.
A mature library for a difficult protocol may be much safer than a generated reimplementation.

Evaluate:

- The complexity of the problem.
- The quality and visibility of the dependency's behavior.
- Security and update obligations.
- How easily failures can be reproduced.
- Whether the team can maintain an alternative.

Do not choose custom code merely because an agent can produce it quickly.
Generation cost does not establish lifetime maintenance cost.

### 4.6 Pin the inputs you depend on

Declare supported toolchain versions.
Preserve dependency resolution.
Document required system capabilities.
Make initialization check those declarations.

Reproducibility has limits.
External services, clocks, and operating systems can still vary.
The project should identify those variations and make them visible.

An environment that fails early with a precise explanation is more useful than one that silently installs something different.

**Exercise.**
Choose five core principles.
For each, write the failure it prevents, the mechanism that enforces it, and the person responsible when the mechanism cannot decide.

---

## 5. Design boundaries that agents can follow

An architecture should make ownership visible in code.
Its value is the reduction of accidental coupling and the clarity of permitted change.

Choose a structure appropriate to the product.
A small application may use a few modules in one process.
A larger system may need separate deployable components.
Distribution should follow a real operational or ownership requirement.

### 5.1 Begin with responsibilities

The request tracker has several distinct responsibilities:

- Domain rules describe valid requests and transitions.
- Application services coordinate a user action.
- Storage adapters persist and retrieve data.
- Interface adapters translate external requests and responses.
- A composition root constructs the running application.

The composition root is the assembly point.
It connects implementations to the contracts the application expects.
Business behavior should remain outside it.

One possible dependency direction is:

```text
Interface adapters
        |
        v
Application services -----> Domain rules
        |
        v
Port contracts
        ^
        |
Storage and external adapters

Composition root assembles the implementations.
```

Arrows mean “may depend on.”
Runtime calls can travel through interfaces in ways that differ from source-code dependencies.
State both meanings when documenting the architecture.

### 5.2 Put invariants at the boundary that owns them

A user interface can disable a resolve button.
That improves the interaction.
It cannot establish that unauthorized callers are unable to resolve requests.

The server-side application boundary must enforce the permission.
The storage operation must preserve the intended workspace scope and concurrency rule.
The interface should faithfully display the outcome.

Similarly, input validation belongs where untrusted data becomes an accepted application value.
Do not scatter slightly different validation logic across screens, handlers, and storage code.

Prefer a flow such as:

```text
Untrusted input
    |
    v
Shape and syntax validation
    |
    v
Authenticated actor and workspace context
    |
    v
Authorization and business rules
    |
    v
Durable state change
    |
    v
Documented response
```

The order may vary for performance or information-disclosure reasons.
The responsibilities must remain explicit.

### 5.3 Separate syntactic validity from authority

A well-formed request identifier does not establish access.
A valid state name does not establish permission to enter that state.
A decoded message does not establish that its claims are trusted.

Use interfaces that make these distinctions visible.
For example:

```text
parse_request_id(raw) -> RequestId or ParseError

authorize_change(actor, workspace, request)
    -> AuthorizedChangeContext or AccessDenied

resolve_request(context, expected_version)
    -> ResolvedRequest or BusinessRejection
```

This is language-neutral pseudocode.
The types illustrate responsibilities rather than guaranteeing a particular implementation.

Even a type called `AuthorizedChangeContext` needs a trustworthy constructor and appropriate runtime checks.
Naming alone does not create security.

### 5.4 Keep public interfaces narrow

If another module can reach every implementation detail, the module boundary is largely cosmetic.

Declare public entry points.
Hide internal storage models, helper functions, and adapter details.
Reject imports that reach through the public entry point.

For example, the interface adapter may call:

```text
requests.resolve(actor_context, request_id, expected_version)
```

It should not reach into:

```text
requests.storage.internal.update_row_without_checks(...)
```

Restricting the second path reduces both accidental misuse and the temptation to work around a failing public contract.

### 5.5 Treat cross-cutting concerns as designed interfaces

Authentication, configuration, logging, tracing, and clocks cross module boundaries.
They need an explicit entry route.

A small shared package can hold narrow contracts.
It should not become a place where unrelated business logic accumulates.

Set rules such as:

- Configuration is validated at startup.
- Application code receives the values it needs.
- Domain logic receives a clock when time affects behavior.
- Logging goes through an approved event interface.
- Request identity travels through a defined context.

These choices make tests easier and failures more interpretable.

### 5.6 Avoid creating a universal layer diagram

A fixed layer order can help one project and distort another.
An event-processing service, a desktop application, and a data pipeline may need different boundaries.

The durable questions are:

1. Which module owns this behavior?
2. Which dependencies may it use?
3. Which data and authority cross the boundary?
4. Which contract protects that crossing?
5. Which check detects a violation?

Answer those questions before multiplying layers.

**Exercise.**
Draw the dependency graph for the first feature.
Then identify one import that must be forbidden and one runtime property that the graph cannot prove.
Give each its own check.

---

## 6. Make the architecture executable

An architecture document explains the system to people.
A machine-readable declaration gives tooling a stable input.
Use both, with a clear relationship between them.

The declaration should describe the boundaries you intend to enforce.
It should not attempt to model every implementation detail.

### 6.1 Establish one canonical policy

Here is a proposed architecture record for the request tracker:

```yaml
schema: architecture/v1
default_dependency_policy: deny

modules:
  request_domain:
    paths: [source/requests/domain]
    public_entry: source/requests/domain/public
    may_depend_on: []

  request_ports:
    paths: [source/requests/ports]
    public_entry: source/requests/ports/public
    may_depend_on: [request_domain]

  request_application:
    paths: [source/requests/application]
    public_entry: source/requests/application/public
    may_depend_on: [request_domain, request_ports]

  interface:
    paths: [source/interface]
    public_entry: source/interface/public
    may_depend_on: [request_application, request_domain]

  storage:
    paths: [source/storage]
    public_entry: source/storage/public
    may_depend_on: [request_ports, request_domain]

  assembly:
    paths: [source/assembly]
    public_entry: source/assembly/main
    may_depend_on:
      - request_domain
      - request_ports
      - request_application
      - interface
      - storage

cross_module_access:
  public_entries_only: true

unclassified_source:
  action: fail
```

This is an illustrative schema.
Its fields need a generator and checker that understand them.
Writing the file alone does not establish a boundary.

Default-deny means the declaration permits only the dependency edges it names.
Internal dependencies within a module need a separate, explicit interpretation.
Decide how tests, generated source, shared types, and external packages are classified.

### 6.2 Generate the configurations you actually run

Use the canonical declaration to produce the build visibility, import rules, or structural tests that your stack supports.
Choose one primary mechanism per relevant boundary.

The generation process should be deterministic:

```text
Approved architecture declaration
            |
            v
Validate schema and module coverage
            |
            v
Generate enforcement configuration
            |
            v
Check generated output matches the repository
            |
            v
Run the installed boundary checker
```

Fail when regeneration would change committed output.
This makes stale generated files visible.
Do not hand-edit those outputs.

The generator also belongs to the trusted foundation.
An agent that cannot change the policy should not be able to alter the generator so that the policy produces weaker checks.

### 6.3 Validate the declaration itself

Check for:

- Unknown module references.
- Missing source directories.
- Overlapping classifications.
- Source files assigned to no module.
- Public entry points that do not exist.
- Forbidden cycles.
- Exceptions without an owner or expiration policy.

An empty module may be intentional before its feature exists.
Represent that state explicitly.
Do not let a misspelled directory look like a successfully checked empty module.

Coverage of the source graph matters.
A perfect rule over the wrong files is still ineffective.

### 6.4 Understand the checker's observation boundary

Different mechanisms see different dependencies.
An import scanner may understand ordinary imports while missing dynamically constructed loads.
A build graph may enforce compiled dependencies while ignoring runtime plugin discovery.
A package boundary may allow access to exported symbols that the architecture intended to keep internal.

State the limitations.
Then constrain the uncovered paths or add another suitable check.

For example, if arbitrary dynamic loading is unnecessary, keep it out of product code.
If plugins are a real requirement, give them a declared registry and validate it.

A tool's green result means that the properties it checked held within its observed scope.
It is not a universal architecture certificate.

### 6.5 Keep meaning linked to enforcement

The human-facing architecture document should explain:

- Why the modules exist.
- Which responsibilities each owns.
- Which dependencies are allowed.
- Where authority and untrusted data enter.
- Which decisions established the boundaries.
- Which checks enforce them.

Generated summaries can help keep the module map accurate.
Human-written rationale still has value because a graph cannot explain the tradeoffs.

Avoid maintaining two independent lists of allowed imports.
Keep the prose about intent and the declaration about policy.
Cross-link them.

### 6.6 Use baselines only where they serve a transition

A new project should begin with zero known boundary violations.
If an experiment is being promoted or a later architectural change creates a migration, a temporary baseline may be necessary.

Freeze exact accepted violations.
Prevent new ones.
Assign each old violation an owner and a removal plan.

Do not automatically regenerate the baseline during ordinary verification.
That would turn new violations into accepted state.

Broad directory exclusions are especially dangerous.
They can hide future mistakes that were never reviewed.

**Exercise.**
Create a source file outside all declared modules.
The architecture check should fail because classification is incomplete.
Then plant a forbidden edge inside a correctly classified module and verify a different, precise failure.

---

## 7. Turn decisions into the right checks

A decision record preserves why the project made a choice.
It should also identify how the choice remains true.

The conversion from decision to check is an engineering task.
It requires classification, examples, and judgment.
An agent can assist with all three.

### 7.1 Record the decision and its obligation

A compact decision record might say:

```text
Decision: workspace scope is explicit on request storage operations

Context:
Requests belong to a workspace.
Looking up a request by identifier alone can bypass isolation.

Decision:
Storage operations used by the application require workspace scope.
An unscoped administrative operation uses a separate interface.

Consequences:
Ordinary callers cannot omit scope accidentally.
Administrative paths require separate authority and verification.

Enforcement:
The public storage contract requires workspace scope.
Contract tests attempt cross-workspace access.
Acceptance tests call the external interface with a foreign identifier.

Owner:
Engineering owner.
```

This decision has several enforcement routes because the property spans several boundaries.

The contract prevents an easy omission.
The tests verify behavior.
Neither should be presented as sufficient in isolation.

### 7.2 Classify the property before generating a rule

| Property | Start with |
|---|---|
| A forbidden dependency edge | Build or import boundary enforcement |
| A prohibited code shape | Syntax-aware static analysis |
| A value that must have a valid shape | Parsing, types, and parser tests |
| Confidential data must not reach a sink | Restricted APIs, appropriate dataflow analysis, runtime checks |
| A state transition must preserve an invariant | Domain and property tests |
| Concurrent operations must satisfy a protocol | Concurrency tests or a model of the protocol |
| A user can complete a journey | End-to-end verification |
| An interaction is understandable | Human observation and usability evaluation |
| A style preference | Formatter, guidance, or review |

Try API design first when it can remove the hazardous option.
A smaller interface often eliminates a class of lint exceptions.

Do not force every decision into one category.
Use the table to identify the primary enforcement route, then add complementary routes where the risk requires them.

### 7.3 Distinguish invariants from preferences

“The interface adapter cannot write directly to storage” is an invariant.
“Helpers should usually appear below exported functions” is a preference.

An invariant has a meaningful failure consequence.
A preference may improve readability without deserving a blocking gate.

Blocking rules consume attention.
They also influence generated code.
A weak rule can make agents contort good implementations into worse ones.

Promote a preference to a blocking rule only when its benefit justifies those effects.
Record the rationale so the rule can be reconsidered later.

### 7.4 Turn a recurring correction into a repair sequence

Suppose review repeatedly finds raw exception objects written to ordinary logs.
Do more than add “do not log exceptions” to the instructions.

Work through the problem:

1. Identify what sensitive values the exceptions may contain.
2. Provide a logging interface that accepts approved fields.
3. Create examples of allowed and prohibited use.
4. Add a suitable static or runtime check.
5. Repair existing violations in small changes.
6. Document the approved route.

The correction now changes the environment the next agent works in.

The right rule may be narrower than the original complaint.
Diagnostic logging might be valid in a restricted environment with a different retention policy.
Model that distinction explicitly.

### 7.5 Make failures teach the permitted alternative

A diagnostic should explain:

- What violated the policy.
- Why the policy exists.
- Which approved path to use.
- Where to read the decision.

Weak:

```text
Invalid dependency.
```

Useful:

```text
BOUNDARY-004: interface code imports a storage implementation.
Request writes must pass through the application service so
authorization and concurrency checks run consistently.
Use the request application public entry point.
Decision: docs/decisions/request-write-boundary.md
```

This helps both humans and agents repair the implementation.
A message that merely says “forbidden” encourages guessing.

Keep the message accurate.
An instruction to use a nonexistent alternative creates another failure loop.

### 7.6 Make exceptions reviewable

Sometimes the sanctioned path does not fit a real requirement.
The agent should preserve the failure and propose a foundation change.

A temporary exception needs:

- The exact rule and location.
- The reason the usual route cannot be used.
- The risk created.
- Compensating verification.
- An owner.
- An expiration or review condition.

Avoid blanket suppression.
Scope the exception to the smallest relevant case.

An exception is a recorded engineering decision.
A suppression comment without ownership is merely hidden policy.

**Exercise.**
Take one review comment you expect to repeat.
Design an interface improvement, a check, and a diagnostic.
If none is appropriate, explain why human review remains the right control.

---

## 8. Test the rules before trusting them

A generated checker is software.
It can misunderstand the requirement, miss files, report the wrong location, or run no rules at all.

Validate the checker with the same care you apply to the feature it protects.
This is especially important when a green result grants an agent more authority.

### 8.1 Write fixtures before the implementation

For a rule that forbids interface-to-storage imports, begin with examples.

Invalid fixtures:

- A direct import of the storage public module.
- An import of an internal storage file.
- An aliased path that resolves to storage.
- A re-export that exposes storage through another module.

Valid near-misses:

- An import of the application public entry point.
- A local type with a similar name.
- A storage adapter used by the composition root.
- A string literal containing a path without importing it.

These fixtures force you to define the scope.
They also expose whether the proposed checker needs dependency resolution or only syntax matching.

Do not accept a syntax-only rule for a requirement that depends on resolved module identity unless another mechanism covers that gap.

### 8.2 Verify positive and negative behavior

A trustworthy rule must do both:

1. Reject known prohibited cases.
2. Accept known permitted cases.

False negatives leave a hole.
False positives train people and agents to distrust the check.

Include the near-misses most likely to occur in your codebase.
You do not need a large artificial corpus for every small rule.
You do need examples that challenge its actual interpretation.

When the rule depends on language syntax, use a parser-aware mechanism.
Text matching can be suitable for simple repository metadata, but it is often too brittle for source semantics.

### 8.3 Assert discovery and execution

A successful process exit does not establish that the expected rules ran.
The wrapper around the checker should verify its effective work.

For example:

```json
{
  "required_rule_ids": [
    "BOUNDARY-004",
    "LOGGING-002"
  ],
  "discovered_rule_ids": [
    "BOUNDARY-004",
    "LOGGING-002"
  ],
  "fixtures_executed": 12,
  "unknown_fixture_ids": [],
  "missing_fixture_ids": [],
  "status": "passed"
}
```

This is a proposed report format.
The wrapper must derive the values from actual execution.
Printing the expected numbers would prove nothing.

Validate rule identity, fixture identity, and required scope.
A rule count above zero is useful but insufficient.
One loaded rule can still hide another missing rule.

### 8.4 Challenge the installed path

Unit fixtures test the rule logic.
They do not establish that the real verification command loads it.

Create a prohibited example at a path matched by the production configuration.
Run the normal project gate in a disposable copy.
Assert the expected rule identifier appears and the gate fails.

Then remove the example and run again.
Assert the valid baseline passes.

This catches:

- Incorrect file globs.
- Unused configuration.
- Missing rule registration.
- Different local and remote entry points.
- Wrappers that ignore failure status.

Test the guardrail where it actually grants or denies acceptance.

### 8.5 Test failure handling in the harness

Intentionally remove a required parser or checker dependency from a disposable environment.
Supply malformed configuration.
Interrupt a test runner.
Make a required evidence directory unavailable.

The harness should report an incomplete or failed verification.
It must not convert inability to check into success.

Keep the vocabulary precise:

| Status | Meaning |
|---|---|
| Passed | Required work executed and satisfied its assertions |
| Failed | Required work executed and found a violation |
| Blocked | A prerequisite prevented execution |
| Not applicable | A declared policy determined the check was unnecessary |
| Skipped | The check did not run |

Required skipped or blocked checks cannot support acceptance.
A warning-colored “success” banner is a poor substitute for those distinctions.

### 8.6 Review changes to the checks

Rule changes need examples and a scope explanation.
Adding a stricter rule can break legitimate code just as weakening a rule can allow bad code.

Review:

- The problem the rule addresses.
- Invalid and valid fixtures.
- The diagnostic.
- Whole-repository results.
- Suppression behavior.
- Discovery and planted-violation results.

Pin the relevant tool versions and retest the installed path after upgrades.
An upgrade can change parsing, file discovery, or rule interpretation even when configuration is unchanged.

**Exercise.**
Disable one rule without deleting its fixtures.
Your meta-check should fail because an expected rule did not execute.
Then bypass the local wrapper and demonstrate that the authoritative gate still detects the planted violation.

---

## 9. Protect the foundation and its verifier

The implementation path and the acceptance path have different responsibilities.
An agent may need broad freedom to edit application code.
That does not require freedom to redefine success for its own task.

Start by stating the threat you intend to contain.
Accidental edits, shortcuts under deadline pressure, and deliberate bypass attempts require different controls.
A local advisory hook is useful feedback.
It is not a security boundary against an agent that can disable the hook or invoke another tool.

### 9.1 Identify the protected surfaces

Protect the controls that determine acceptance:

- Architecture policy and its generator.
- Boundary and custom-rule configuration.
- Rule fixtures and meta-checks.
- Required acceptance specifications.
- Verification commands and orchestration.
- Dependency and credential policy.
- Build and release definitions.
- Path ownership and review policy.
- The configuration that activates those protections.

Protection can mean read-only mounting, restricted credentials, required review, or execution from a trusted policy revision.
Choose the mechanism appropriate to the surface.

Do not protect every file indiscriminately.
The agent still needs to add tests and improve ordinary implementation code.

### 9.2 Distinguish test roles

Test files have different authority.

| Test role | Ordinary implementation authority |
|---|---|
| New unit tests for the change | Agent can usually create and edit |
| Existing implementation tests | Changes are allowed within policy and reviewed for weakened assertions |
| Required acceptance tests | Agent proposes changes through the protected route |
| Hidden evaluation cases | Agent does not receive them |
| Verification infrastructure | Foundation review required |

The precise policy depends on the project.
The principle is that removing evidence must not be an ordinary way to pass.

A test can be wrong.
Provide a route to challenge it with the requirement, reproduction, and proposed correction.
Keep that correction distinct from an implementation claim that the original test passed.

### 9.3 Use local controls for fast feedback

Local checks can reject edits to protected paths, detect common bypass commands, and run fast verification after changes.
They help the agent stay within its assignment.

Self-test those controls with representative input.
If a required dependency is missing, report that enforcement is unavailable.
For controls intended to deny unauthorized actions, fail closed.

Be careful about shell-command inspection.
A small list of banned strings is easy to route around and can reject harmless commands.
Restrict actual capabilities and credentials where a meaningful security boundary is needed.

The local layer should explain mistakes quickly.
The authoritative layer must decide whether the result can be accepted.

### 9.4 Put the authoritative decision outside ordinary edits

A remote gate is only authoritative if the proposed change cannot freely redefine the gate.
Running a branch's editable verification script and trusting its exit code leaves the same problem in another location.

Use an appropriate trust arrangement:

- Run protected checks from an approved policy revision.
- Require independent review before accepting changes to verification definitions.
- Keep acceptance fixtures in a separately controlled location.
- Restrict which identity can approve or publish a result.
- Verify the exact candidate revision with the approved policy.

The setup may be simple for one developer.
The developer can explicitly review foundation changes before running trusted acceptance.
Larger systems can encode the same separation in access controls and repository policy.

The important question is:

> Can the implementation agent cause its own change to be accepted by altering the thing that judges it?

If the answer is yes, the lock is incomplete.

### 9.5 Checksums need an independent reference

A manifest can detect unexpected changes to protected files.
It does not help if the agent can change the files, update the manifest, and alter the checker together.

Compare against an approved reference outside the ordinary edit path, or require review of both policy changes and their manifest.
Bind acceptance to the reviewed policy version.

Checksums establish identity and integrity relative to a reference.
They do not establish that the policy itself is correct.

### 9.6 Protect runtime authority too

Repository protections do not limit what the running agent can do elsewhere.

Scope:

- Filesystem access.
- Network destinations.
- External service credentials.
- Shared databases.
- Publication and deployment rights.
- Destructive operations.

Give a coding task the access it needs for that task.
Keep production mutation and irreversible external actions behind an explicit policy.

Isolation should also contain ordinary mistakes.
A test that deletes a database should reach a disposable test database, never a shared production resource.

### 9.7 Provide a foundation change route

When a protected decision needs to change, the agent prepares:

1. A statement of the blocking requirement.
2. Evidence that the approved route is insufficient.
3. A proposed policy and implementation change.
4. Updated fixtures and negative challenges.
5. The resulting scope of authority.
6. A rollback or recovery plan where relevant.

The accountable owner reviews that package.
Record the accepted policy version.
Re-run the valid baseline and the rejection demonstrations.

This route keeps the project adaptable without letting convenience redefine acceptance.

**Exercise.**
Attempt to weaken a required acceptance check and update its local manifest in a disposable branch.
Demonstrate which independent mechanism prevents acceptance.
If only the local manifest catches it, strengthen the trust arrangement.

---

## 10. Make the application runnable and observable

An agent that cannot reliably start the application will invent substitutes.
It may test a helper while the actual route is broken.
It may inspect markup without exercising the running interface.
It may declare success based on a build that never started the service.

Give the project a supported route from a clean checkout to a working application.
Make that route useful to humans too.

### 10.1 Define a stable operational interface

A small set of project commands can describe the lifecycle:

```text
project initialize
project doctor
project start --environment <id>
project verify --environment <id> --feature <id>
project evidence --environment <id>
project stop --environment <id>
```

These names describe a proposed interface.
They are not an installed command.
Implement the equivalent using your project's normal tooling.

Keep the interface stable even if the internals change.
Bundled, versioned scripts reduce the measurement drift caused by agents writing a new launch or verification script for every task.

### 10.2 Separate initialization from readiness

Initialization prepares dependencies and disposable local state.
Readiness checks whether the application can do useful work.

A doctor command should distinguish:

- Required toolchain missing.
- Dependency resolution incomplete.
- Configuration invalid.
- Storage unavailable.
- Schema incompatible.
- Service started but not ready.
- Ready for the required verification.

Use bounded waiting.
A fixed delay followed by an optimistic request is fragile.
Poll an appropriate readiness condition until it passes or a deadline expires.

Readiness should test the capabilities the journey requires.
A process listening on a port may still be unable to access storage.

### 10.3 Isolate each environment

Two coding sessions should not share mutable test state accidentally.

Assign each environment:

- A unique identity.
- A private checkout or equivalent workspace.
- Allocated ports.
- A private database or storage namespace.
- Private temporary files.
- Separate logs and evidence.
- Explicit credentials for disposable services.

Use an environment record:

```yaml
environment_id: task-reopen-01
candidate_revision: "<revision>"
policy_revision: "<approved-policy-revision>"
ports:
  application: "<allocated-port>"
storage_namespace: task-reopen-01
artifact_directory: evidence/task-reopen-01
resource_owner: task-reopen-01
expires_when: task-results-preserved
```

The record is illustrative.
The environment manager must allocate and enforce the actual resources.

### 10.4 Own the resources you create

Cleanup should act on resources owned by the environment.
Avoid broad process-name termination or deletion of a shared temporary directory.

Record identifiers for started processes, containers, databases, or leases.
Stop those resources specifically.
Preserve evidence before reclaiming them.

When startup fails halfway, cleanup still needs to know what was created.
Write ownership state as resources are allocated.
Make cleanup safe to repeat.

This turns interrupted sessions from mystery failures into recoverable lifecycle events.

### 10.5 Make behavior inspectable

At minimum, the running application should expose:

- Structured errors.
- Correlated application logs.
- Health and readiness status.
- Enough state inspection to verify the critical journey.

Add traces and metrics when they help diagnose a real problem.
A small application may not need a full observability platform per environment.
It still needs useful evidence.

For a request transition, useful fields include:

```text
event: request.transition
environment_id
correlation_id
request_id
workspace_id
actor_id
from_state
to_state
result
duration
```

Identifiers may require redaction or restricted access.
Do not record request bodies or credentials merely because they make debugging convenient.
Define logging policy alongside the feature.

### 10.6 Let verification inspect the actual system

The request tracker journey should use the running interface and its real disposable storage.
A denied transition should leave durable state unchanged.
A successful transition should remain visible after a fresh read.

Use diagnostics to explain failure:

```text
User action
    |
    v
Interface request with correlation identity
    |
    v
Application decision
    |
    v
Storage result
    |
    v
Observed response and fresh state
```

Evidence from each step can reveal where behavior diverged.
A screenshot of the final screen is helpful but cannot establish every property in that chain.

### 10.7 Test the lifecycle itself

Before autonomous feature work, demonstrate:

1. A clean environment initializes.
2. A second isolated environment starts concurrently.
3. A critical journey passes in each.
4. Stopping one does not disrupt the other.
5. Interrupted startup can be cleaned up.
6. Evidence remains available after cleanup.

These checks verify the development harness.
They prevent feature agents from debugging the harness as part of every task.

**Exercise.**
Start two environments with different request data.
Resolve a request in one.
Demonstrate that the other environment's data and process remain unchanged.

---

## 11. Build a layered verification system

Verification answers different questions at different layers.
Choose a set of checks that can contradict the implementation's claims.

The first question is:

> What plausible failure would this check detect?

If the answer is unclear, the check may be adding ceremony rather than confidence.

### 11.1 Establish a small blocking baseline

For an ordinary application, a useful initial baseline includes:

- Formatting and basic static checks.
- Strict type checks where the language supports them.
- Dependency and architecture checks.
- Tests of important domain behavior.
- Contract checks at relevant boundaries.
- A critical end-to-end journey.
- Checks that required tests and rules actually executed.
- Secret and dependency policy checks appropriate to the project.

Keep the required route deterministic.
Failures should explain the violated property and preserve diagnostic output.

Coverage can identify unexercised code.
It cannot establish that assertions are meaningful.
Treat it as one signal within the verification plan.

### 11.2 Use unit tests for local behavior

The request state machine is a good unit-test target.
It can run without storage, the user interface, or network access.

Test:

- Open to assigned.
- Open or assigned to resolved.
- Resolved to open, clearing the owner.
- Rejected transitions.
- Behavior at relevant version or validation boundaries.

Keep the expected behavior derived from the product contract.
Do not write assertions that merely preserve whatever the first generated function happened to return.

### 11.3 Use property tests for families of cases

A property describes behavior across many inputs.
For example:

> A rejected transition preserves the request's business state.

Or:

> Serializing and parsing a valid canonical request value preserves that value.

The qualifiers matter.
A lossy serializer cannot satisfy a universal round-trip property.
An audit record may legitimately change even when the business operation is rejected.
Specify what state is being compared.

Good property targets include:

- Parsers.
- Serializers.
- State transitions.
- Ordering and deduplication.
- Idempotent operations.
- Input normalization.

Generators should include boundary cases and preserve realistic constraints.
A property exercised only over trivial inputs can look rigorous while missing the failure you care about.

### 11.4 Use contract tests at seams

Contract tests verify the agreement between modules or services.

For the storage port, test that:

- Scope is respected.
- Missing records have the documented result.
- A stale version cannot overwrite a newer version.
- A successful write can be read.
- Failure does not leave a partial business update.

Run the same relevant contract against the real adapter.
A fake storage implementation passing the contract does not establish that the production adapter does.

External integrations need their own strategy.
Use recorded or simulated cases for speed, then verify critical assumptions against a representative real environment.
Document where simulation ends.

### 11.5 Exercise the user journey

An end-to-end test establishes that the layers work together for a specific journey.

For the request tracker:

1. Sign in as an authorized workspace member.
2. Create a request.
3. Assign it.
4. Resolve it.
5. Reload and confirm the state.
6. Reopen it.
7. Confirm the owner is cleared.
8. Attempt the same operation as an unauthorized actor.
9. Confirm denial and unchanged durable state.

The journey should use realistic interfaces.
If the product has a graphical interface, verify the visible interaction as well as the backing behavior.

Visual evidence can establish layout and state.
Behavioral assertions establish the conditions that screenshots alone cannot reveal.

### 11.6 Add deeper techniques where risk earns them

Fuzzing is useful for inputs with a large or adversarial search space.
Load tests address capacity assumptions.
Concurrency tests address interleavings.
Security testing challenges authority and data boundaries.

Formal modeling can help with a protocol whose correctness depends on states and interleavings that are difficult to enumerate.
It is valuable only when the model's assumptions are explicit and its relationship to the implementation is maintained.

A proven model does not automatically prove the deployed application.
The implementation, environment, and model can diverge.
Record what was proved, under which assumptions, and how changes trigger reassessment.

Choose these techniques by consequence and uncertainty.
Avoid making every greenfield project pay for machinery that has no meaningful property to examine.

### 11.7 Combine targeted checks with integration coverage

Fast local checks help an agent iterate.
Checks scoped to changed behavior can make deeper testing affordable.
The scope calculation is itself part of the harness.

If a shared parser changes, all its relevant callers may be affected.
If dependency selection is incomplete, targeted verification can omit the regression.

Use:

- Fast checks during editing.
- Relevant domain and contract checks before proposing completion.
- Required integration checks for acceptance.
- Broader suites at a cadence and scope justified by the system.

Do not replace all integration evidence with small tests simply because small tests are fast.
Do not run every expensive suite after every keystroke.
Place the right evidence at the decision it supports.

### 11.8 Preserve uncertainty honestly

A missing credential, unavailable external service, or unresolved flaky test leaves a gap.
Report the gap.
Do not infer success from neighboring checks.

An acceptance report should make required incomplete checks easy to see.
The reviewer can then decide whether to fix the environment, narrow the claim, or postpone acceptance.

**Exercise.**
Choose one critical invariant.
List the unit, contract, journey, and operational evidence that bears on it.
Explain what each layer still cannot establish.

---

## 12. Keep tests independent and challenge their strength

When an agent writes code and then derives tests from that code, both can share the same misunderstanding.
A polished test suite may confirm an incorrect implementation very consistently.

Create some verification from the requirement before inspecting the implementation.
Use implementation-aware tests for other purposes.
The two perspectives are complementary.

### 12.1 Begin with a specification-derived oracle

An oracle is the basis for deciding whether an observed result is correct.
For reopening a request, the product contract supplies the oracle:

- The prior state is resolved.
- The actor has permission.
- The new state is open.
- The owner is cleared.
- The change persists.
- Unauthorized attempts have no effect.

Write acceptance cases from those statements.
Then build the implementation.

If the implementation returns a different state, investigate the disagreement.
Do not update the oracle automatically to match the output.

### 12.2 Use implementation-aware tests for failure paths

Once implementation exists, inspect details that requirements alone may not expose.

For example:

- What happens when storage fails after validation?
- Can an exception escape without diagnostic context?
- Is a retry policy applied to a non-repeatable action?
- Can a concurrent update invalidate an earlier authorization or version assumption?

These tests challenge the chosen design.
They do not replace the original acceptance cases.

Specification-derived tests ask whether the feature does the right thing.
Implementation-aware tests ask where this implementation might fail.

### 12.3 Separate verifier perspective from verifier authority

A second agent can provide a fresh perspective.
A different model may produce different hypotheses.
Neither automatically creates independent enforcement.

The verifier needs:

- A clear specification.
- Access to the relevant evidence.
- A task that encourages contradiction.
- Limits on editing the acceptance criteria.
- A trusted route to run checks.

Useful adversarial questions include:

> Which requirement would still be violated if every reported test passed?

> What is the smallest counterexample to the proposed fix?

> Can this success be produced by stale state or a mocked dependency?

The review generates hypotheses.
Reproduce those hypotheses with executable evidence or careful human analysis.
Agreement between two models is not a proof of correctness.

### 12.4 Use mutation testing to examine assertions

Mutation testing deliberately changes behavior in a small way and asks whether the tests notice.

For the request tracker, useful mutations include:

- Removing the workspace constraint.
- Allowing a stale version.
- Keeping the owner when reopening.
- Returning success without persisting.
- Allowing an unauthorized actor.

If a relevant test fails, it detected that mutation.
If tests pass, investigate whether the mutation exposes a missing assertion.

Some mutations are behaviorally equivalent.
Some cannot execute because they do not compile.
Some lie outside the acceptance claim.
Classify them rather than interpreting every survivor as a defect.

### 12.5 Keep mutation scope meaningful

Changed code is a useful starting point.
Critical invariants may involve code outside the diff.

Select the scope from the affected behavior and its risk.
For an authorization change, include the surrounding enforcement path even if only one helper changed.

Use a reviewed set of critical mutations when the general tool cannot express the property cleanly.
A named challenge that removes workspace scoping can be more informative than a broad percentage.

A mutation score measures detection of the mutations attempted.
It does not certify all possible behavior.
Thresholds should reflect an understood corpus and risk.

### 12.6 Watch for evidence weakening

Review test changes for:

- Removed denied cases.
- Broadened expected results.
- Assertions replaced with snapshots without review.
- Required suites marked optional.
- Error paths no longer exercised.
- Test discovery reduced.
- Fakes that now agree with a broken implementation.

Simple detectors can flag test-count or assertion changes.
Those signals need interpretation.
Fewer assertions can result from a legitimate refactor, and more assertions can still test nothing useful.

Require explicit review when acceptance evidence changes.
Do not use assertion count as the final authority.

### 12.7 Make wrong tests correctable

A protected test may contain an error.
Freezing it forever would force the implementation to satisfy the wrong requirement.

The correction package should show:

1. The approved requirement.
2. The conflicting test expectation.
3. A minimal reproduction.
4. The proposed correction.
5. The remaining acceptance coverage.

After review, update the trusted test version and rerun the candidate.
Preserve the record of why the criterion changed.

This is different from editing the test until an implementation becomes green.
The authority comes from the requirement and accountable review.

**Exercise.**
Introduce a change that returns success without persisting a request.
Find which test catches it.
If none does, add a fresh-read assertion through the appropriate boundary.

---

## 13. Create a feature map and an evidence contract

A feature map connects user-visible behavior to the procedure that verifies it.
It gives future sessions a durable account of how the product works.

Keep it separate from a list of implementation tasks.
“Add a service class” describes construction.
“An authorized member reopens a resolved request” describes behavior.

### 13.1 Give each feature a stable identity

```yaml
feature_id: request-reopen
user_job: return a resolved request to active work

preconditions:
  - a resolved request exists in the member's workspace
  - the member has permission to reopen

journey:
  - open the request detail
  - choose reopen
  - confirm the result
  - reload the detail

expected:
  - state is open
  - owner is cleared
  - state survives the reload

negative_cases:
  - an unauthorized member cannot reopen
  - a member from another workspace cannot access the request
  - a stale update does not overwrite a newer change

verification:
  procedure: verify-request-reopen
  required_evidence:
    - journey-results
    - fresh-state-check
    - authorization-results
    - relevant-application-log
```

The procedure name is a proposed project-local entry point.
It should resolve to versioned code or documented steps.

### 13.2 Separate the specification from observed status

Avoid storing a single permanent `passes: true` flag as the feature's truth.
A feature can pass on one revision and regress on the next.

Keep:

- The feature specification.
- The latest verification result.
- The candidate revision.
- The policy and environment versions.
- The evidence location.

For example:

```json
{
  "feature_id": "request-reopen",
  "candidate_revision": "<revision>",
  "verification_policy_revision": "<policy-revision>",
  "environment_id": "task-reopen-01",
  "status": "passed",
  "required_checks_executed": [
    "transition",
    "persistence",
    "authorization",
    "stale-update"
  ],
  "evidence": "evidence/task-reopen-01/report.json"
}
```

Prefer machine-generated observed status.
Agents may report what they did, but the trusted runner should establish required results.

Use structured files for validation and identity.
The file format does not protect the contents by itself.

### 13.3 Standardize the verification procedure

A useful procedure has five parts:

1. **Launch:** create the intended application environment.
2. **Readiness:** establish that prerequisites and services work.
3. **Drive:** perform the defined user actions and negative cases.
4. **Evidence:** record assertions, observations, and diagnostics.
5. **Cleanup:** reclaim only the resources the procedure owns.

Allow the procedure to reuse an existing ready environment when policy permits.
Record that choice.
State reset must be explicit so a prior run cannot manufacture the next run's success.

### 13.4 Tie evidence to the actual candidate

Evidence should identify:

- Candidate revision.
- Uncommitted changes, if verification allows them.
- Build artifact identity where relevant.
- Verification policy version.
- Environment configuration.
- Fixture or dataset version.
- Time of execution.
- Required checks and their results.
- Missing or skipped work.

If code changes after verification, the evidence belongs to the earlier state.
Rerun the affected checks for the new candidate.

Do not attach screenshots from a previous session merely because they look correct.
Freshness and identity are part of the evidence contract.

### 13.5 Make evidence useful to a reviewer

A compact review packet should answer:

> What changed?
> Which behavior does it establish?
> How was it tested?
> What remains uncertain?

For a graphical interaction, include relevant screenshots or a short recording.
For a storage guarantee, include the assertion and fresh read.
For a performance claim, include the workload and measured result.
For a rule, include the prohibited fixture and reported rejection.

Avoid dumping an entire log directory into the review.
Preserve raw artifacts for investigation.
Present the evidence that supports the claim.

### 13.6 Keep evidence within data policy

Verification artifacts can contain sensitive information.
Use synthetic fixtures where possible.
Redact secrets and personal data.
Set retention and access rules.

A screenshot can expose a token as easily as a log line.
An evidence directory should not become a permanent uncontrolled copy of production data.

### 13.7 Maintain the map as the product changes

When an interaction changes, update its journey and expected observations.
When a feature is retired, mark it retired with a reason.
Do not silently delete history to make the map smaller.

Check that:

- Every active feature has a verification route.
- Every named route exists.
- Required fixtures remain available.
- The route still reaches the intended behavior.

The feature map is operational documentation.
Its quality is demonstrated by execution.

**Exercise.**
Give a fresh session only the repository map and one feature record.
It should be able to locate the requirement, run the supported verification, and explain the resulting evidence without reconstructing the workflow from conversation history.

---

## 14. Give agents a map of the repository

The repository should explain how to work without requiring a long conversation to be remembered.
Keep the entry instructions short.
Use them to route the reader to the relevant knowledge.

The entry document is a navigation surface.
It should not duplicate every decision, feature, and operating procedure.

### 14.1 Use progressive disclosure

Organize context in layers:

| Layer | Contents |
|---|---|
| Entry map | Purpose, key paths, standard commands, authority limits |
| Domain documentation | Product rules, architecture, decisions, feature map |
| Task context | Current objective, affected interfaces, acceptance criteria |
| Procedure details | Scripts, fixtures, diagnostics, recovery instructions |

The agent loads what the task needs.
Critical constraints still belong in the entry map if missing them would cause harmful action.

Do not bury “this environment has access to shared production data” in an optional reference.
Prefer removing that access from ordinary coding tasks altogether.

### 14.2 Build a small knowledge structure

A possible repository layout is:

```text
repository-map.md
architecture.yaml

docs/
  product/
    first-release.md
    open-questions.md
  design/
    principles.md
    architecture.md
  decisions/
  features/
  operations/
    initialize.md
    verify.md
    release.md
    recover.md
  plans/
    active/
    completed/
  quality/
    known-debt.md
    domain-status.md

source/
checks/
verification/
```

These paths are illustrative.
Use the conventions your repository already supports.
The responsibilities matter more than the filenames.

### 14.3 Preserve decisions outside chat

Conversation is good for exploring choices.
Accepted decisions need a durable home.

Record:

- The decision.
- Its rationale.
- The scope.
- The enforcement or review route.
- The owner.
- Conditions for reconsideration.

A future agent should not need to infer which of several conversational alternatives was accepted.
Remove abandoned proposals from active guidance or label them clearly.

### 14.4 Keep procedures close to their executable parts

A verification procedure should point to the runner it uses.
A rule description should point to its fixtures.
A recovery procedure should identify the environment and authority required.

Bundle deterministic scripts with the workflow that depends on them.
Avoid making agents recreate commands from prose each time.

Where instructions become executable scripts, keep the prose focused on intent, prerequisites, and interpretation.
Duplicated command sequences drift.

### 14.5 Check what machines can check

Automate validation of:

- Broken internal links.
- Missing referenced files.
- Unknown rule or feature identifiers.
- Required ownership fields.
- Verification routes that do not exist.
- Active plans without a current status.

Use human review for meaning.
A link checker cannot tell whether a design explanation is still true.

Documentation maintenance should compare claimed behavior with code and verification results.
It should not rewrite a guarantee to match a regression.

### 14.6 Keep progress records compact

A session handoff should contain:

- What changed.
- The exact revision or patch location.
- Checks run and their results.
- Remaining questions.
- The next bounded action.

Do not preserve a transcript of every thought.
That creates noise and encourages future sessions to repeat abandoned reasoning.

Durable progress is a record of state and evidence.
It is not an imitation of memory.

### 14.7 Treat retrieved content as information

Issue text, logs, web pages, and external documents may contain instructions.
Those instructions do not automatically receive authority over the task.

The agent should use external content as evidence within the approved workflow.
Repository policy, credential scope, and action authority come from trusted configuration and accountable decisions.

This matters during debugging.
A malicious string in an error log should not become permission to run a command.
The harness must preserve that boundary through tool access and review policy.

**Exercise.**
Reduce the entry instructions to what a fresh session needs first.
Move detailed procedures into linked documents.
Then verify that every critical constraint is still either visible at entry or enforced by the environment.

---

## 15. Run a reliable coding session

A coding session should leave the project in a state another session can understand.
Design the workflow around that handoff.

The agent begins with the current repository and evidence.
It should not assume that its previous conversation still describes reality.

### 15.1 Establish the starting state

At session start:

1. Read the repository map.
2. Read the assigned task and relevant decisions.
3. Inspect the current revision and working changes.
4. Read the latest compact progress record.
5. Initialize or identify the isolated environment.
6. Run the relevant baseline smoke test.

The baseline answers whether the task starts from a working state.
If it fails, preserve that failure before making changes.

Do not hide a baseline defect inside a feature patch.
Decide whether fixing it is necessary to the task, a separate bounded task, or a blocker.
The record should distinguish those cases.

### 15.2 Use a task contract

```yaml
task_id: implement-request-reopen
base_revision: "<reviewed-revision>"

objective:
  allow an authorized member to reopen a resolved request

acceptance:
  - state becomes open
  - owner is cleared
  - result survives a fresh read
  - unauthorized and cross-workspace attempts have no effect
  - stale updates are rejected

scope:
  may_change:
    - request application implementation
    - interface behavior
    - new implementation tests
  protected:
    - architecture policy
    - required acceptance specification
    - verification infrastructure

evidence:
  - domain and contract results
  - reopen journey result
  - relevant negative cases

completion:
  preserve the candidate and evidence for review
```

This example does not authorize publication or production deployment.
Those actions require their own authority.

The scope should be specific enough to detect drift.
It should also leave room for ordinary implementation decisions.

### 15.3 Implement one coherent behavior

Work on a vertical slice where possible.
For reopening, the slice includes the relevant domain behavior, application coordination, interface action, and verification.

Avoid filling the repository with many partially connected features.
That increases the number of unknowns when the application fails.

A bounded slice makes contradictions easier to find.
If the interface reports open while storage remains resolved, the failure belongs to a small path you can inspect.

### 15.4 Run the inner loop deliberately

Use this sequence:

```text
Understand the requirement
          |
          v
Create or confirm the acceptance cases
          |
          v
Implement the smallest coherent change
          |
          v
Run relevant fast checks
          |
          v
Inspect the actual behavior
          |
          v
Challenge the success claim
          |
          v
Preserve candidate and evidence
```

When a check fails, repair the implementation or identify a genuine policy disagreement.
Do not silently remove the check.

If the agent repeatedly reaches for a prohibited path, examine the approved interface.
The task may reveal that it is incomplete.
Prepare a foundation change rather than accumulating workarounds.

### 15.5 Define completion as a set of states

| State | What is known |
|---|---|
| Started | A task and environment exist |
| Implemented | A candidate change exists |
| Verified | Required checks ran against the candidate |
| Preserved | Candidate and evidence are durable |
| Accepted | The accountable review route approved the change |
| Released | The accepted behavior is available to the intended users |

An agent can stop after preserving a failed candidate.
That is a useful handoff, but it is not verified completion.

An accepted patch can remain unreleased.
A deployed artifact can remain unavailable to users.
Use the state vocabulary consistently.

### 15.6 Make stopping safe

Agents encounter time limits, unavailable services, and unresolved requirements.
The workflow should support an honest checkpoint.

Before stopping:

- Preserve the patch or revision.
- Record the environment identity.
- Preserve failed and successful evidence.
- State the remaining blocker or next action.
- Clean up owned resources according to policy.

A completion gate should prevent false claims of success.
It should not force an endless session when verification cannot run.

The agent may report blocked or incomplete.
The system should retain enough state to resume without repeating investigation.

### 15.7 Review the resulting diff

Checks establish particular properties.
The final diff still deserves inspection for scope and intent.

Look for:

- Unrelated files.
- Foundation changes.
- Weakened assertions.
- New dependencies.
- Debugging residue.
- Unnecessary abstractions.
- Duplicate implementations.
- Accidental secrets or private data.

A narrow diff helps the reviewer connect the requirement, implementation, and evidence.
If the task expanded, rewrite its description around the actual change and seek the relevant review.

**Exercise.**
Interrupt a session after implementation but before verification.
The next session should recover the candidate, establish its baseline, and complete the required checks without relying on unstated conversation history.

---

## 16. Decompose work before increasing parallelism

Parallel work helps when tasks are independent enough to be implemented and verified separately.
It creates trouble when several agents must negotiate an unstated interface while changing it.

The first scaling technique is decomposition.
Additional agents come afterward.

### 16.1 Separate decisions from implementation

Consider adding external notifications to the request tracker.
Possible tasks include:

- Decide which state changes produce notifications.
- Define the notification contract.
- Choose delivery and retry semantics.
- Implement the durable event record.
- Implement the delivery adapter.
- Add the user-facing preference.
- Verify retries and duplicates.

The implementation tasks depend on earlier decisions.
Dispatching all of them at once encourages incompatible assumptions.

Approve the contract first.
Then parallelize work behind that contract where ownership permits.

### 16.2 Use a dependency graph for tasks

```text
Approved notification behavior
              |
              v
Event and delivery contract
       /              \
      v                v
Event recording    Delivery adapter
      \                /
       v              v
       Integrated delivery verification
              |
              v
       Controlled release
```

The graph describes an illustrative extension.
The first release still excludes notifications.
Add this machinery only when that feature becomes a real requirement.

An integration task owns the seam.
Without an owner, each component can be locally correct while the feature remains broken.

### 16.3 Bound each assignment

A good assignment has:

- A stable starting revision.
- One coherent objective.
- Owned paths or interfaces.
- Explicit dependencies.
- Acceptance evidence.
- A resource and time budget.
- A durable result location.

Avoid assignments such as “improve the backend.”
They create overlapping interpretations of success.

Prefer:

> Implement the storage adapter for the approved event-record contract and run the adapter contract tests.

That task has a visible boundary.

### 16.4 Isolate work and shared state

Give parallel tasks separate working directories and test environments.
Shared read-only caches can reduce setup time.
Shared mutable databases or working trees create contamination.

Isolation must include:

- Temporary storage.
- Ports.
- Test identities.
- Fixture data.
- Logs.
- Build output where collisions are possible.

If a resource must be shared, state how access is coordinated.
Do not rely on the agents choosing different names independently.

### 16.5 Assign ownership of shared surfaces

Files such as public contracts, dependency declarations, and feature maps attract many changes.
Either establish them before parallel work or designate one task as their owner.

Other tasks can propose changes to the owner.
They should not independently redefine the contract to fit their implementation.

A file conflict is often evidence of a responsibility conflict.
Resolving the text conflict is only part of the repair.
Reconcile the assumptions too.

### 16.6 Reverify the integrated candidate

Task A passes against its starting revision.
Task B passes against the same starting revision.
The combination can still fail.

After integration, run the checks relevant to the combined behavior.
Bind the evidence to the integrated candidate.

Inspect:

- Contract compatibility.
- Initialization order.
- Shared configuration.
- Error semantics.
- Resource ownership.
- Duplicate or omitted effects.

Two green task reports do not establish one green product.

### 16.7 Keep review capacity in the concurrency budget

Parallelism creates finished work in bursts.
If the team cannot review the burst, candidates become stale.
Dependencies move, evidence ages, and integration becomes harder.

Control admission using more than machine capacity.
Include:

- Verification capacity.
- Review capacity.
- Shared integration ownership.
- Time to resolve failures.

Start with the smallest concurrency that demonstrates useful throughput.
Increase it when acceptance and recovery remain manageable.

**Exercise.**
Split one feature into three tasks.
Name the contract they share and the person or task that owns integration.
If no stable contract exists, keep that portion sequential.

---

## 17. Control entropy as the project grows

Generated code can accumulate faster than its patterns are reconsidered.
Make maintenance part of the operating model from the beginning.

The aim is a repository whose examples remain worth copying.

### 17.1 Maintain a small set of golden examples

Provide representative implementations of:

- Input validation.
- Authorization.
- Application orchestration.
- Storage access.
- Expected error handling.
- Logging.
- Verification evidence.

They can be real production features.
Avoid a separate example tree that quietly diverges from the application.

When the approved approach changes, update the representative implementation.
Remove obsolete examples or label them as migration work.

### 17.2 Detect drift in small batches

Recurring checks can look for:

- Repeated implementation patterns that should be consolidated.
- Forbidden dependencies.
- Expired exceptions.
- Unused public interfaces.
- Stale feature journeys.
- Decision records without a working enforcement route.
- Quality gaps concentrated in one domain.

Use these checks to open bounded maintenance work.
A broad “clean up the repository” assignment is difficult to verify and easy to expand.

### 17.3 Stop new violations before repairing old ones

If a bad pattern has spread, first establish the intended rule and migration boundary.
Then repair existing instances.

For example:

1. Introduce the approved logging interface.
2. Reject new raw exception logging.
3. Freeze the exact known violations.
4. Repair one domain at a time.
5. Remove the baseline when complete.

This prevents cleanup from chasing a moving target.
The baseline must stay protected and reviewed.

### 17.4 Keep exceptions alive only while justified

An exception needs a review condition.
That can be a date, a dependency upgrade, or completion of a named migration.

When the condition arrives, reassess it.
Do not extend every exception automatically.

Track:

- Exact affected location.
- Original rationale.
- Current owner.
- Compensating checks.
- Removal plan.
- Last review.

A baseline with no owner is deferred accountability.

### 17.5 Review the rules as well as the code

Rules can become stale or counterproductive.
Observe how they behave.

| Signal | Possible interpretation |
|---|---|
| Frequent failures on ordinary tasks | The interface or rule may be poorly designed |
| Frequent overrides | Policy may conflict with legitimate work |
| Repeated false positives | Rule needs narrower scope or better resolution |
| No failures | Rule may be effective, irrelevant, or disconnected |
| Many repairs using the same awkward pattern | Approved alternative may be inadequate |

A rule that never fires is not automatically useless.
It may protect a low-frequency but severe failure.
Challenge its installation and reconsider its risk before removing it.

Use observation to improve the foundation.
Do not turn rule telemetry into a quota.

### 17.6 Maintain documentation and verification together

When the reopen action moves to a new screen, the feature map must change.
When the interaction changes meaning, the acceptance specification may need review too.

A documentation maintenance task should identify:

- What became stale.
- The current source of truth.
- Whether behavior changed or only navigation changed.
- Which verification proves the new description.

If documentation claims unauthorized users are denied but tests reveal access, repair the behavior.
Do not merely update the documentation to describe the defect.

### 17.7 Use quality status to locate work

Track domain quality with explicit evidence:

```yaml
domain: requests

architecture:
  status: passing
  evidence: "<report>"

critical_journeys:
  status: gap
  reason: reopen concurrency case has no current evidence

exceptions:
  count: 1
  owner: engineering-owner
  next_review: "<review-condition>"

recovery:
  status: unverified
  reason: restore drill not yet run
```

This is more useful than an unexplained score.
If you use a score, preserve its components and supporting evidence.

### 17.8 Apply ordinary acceptance to maintenance

A cleanup task is still a change.
It needs scope, verification, and a durable result.

Automatically accepting maintenance is appropriate only when the action class, controls, and recovery behavior justify that authority.
A job called “cleanup” may still alter permissions or delete durable state.
Judge the action, not the label.

**Exercise.**
Choose one drift check and one maintenance task it can produce.
Define the task's maximum scope and the evidence required to accept it.

---

## 18. Expand autonomy through measured outcomes

Autonomy is a collection of permissions.
Specify those permissions separately.

An agent may be excellent at implementing a patch while having no authority to release it.
Another workflow may safely update a generated document but require review for a dependency change.

### 18.1 Create an authority ladder

| Level | Agent authority | Acceptance responsibility |
|---|---|---|
| Observe | Read, diagnose, and propose | Human decides what work to start |
| Implement | Edit in an isolated workspace and verify | Human reviews the candidate |
| Propose integration | Prepare a reviewable change and evidence | Protected acceptance route approves |
| Integrate bounded changes | Accept approved classes with required gates | Policy owner reviews the class and exceptions |
| Deploy bounded artifacts | Deploy verified artifacts to approved environments | Release policy controls user exposure |
| Operate bounded repairs | Respond to specific signals with recoverable actions | Operational owner defines limits and escalation |

This ladder is illustrative.
Some projects should stop at implementation.
Others can safely automate specific integration or release classes.

Keep permissions narrow enough to explain.
“Full autonomy” is not an actionable policy.

### 18.2 Define action classes

A documentation link repair, a new domain function, and a database migration have different consequences.
Classify work accordingly.

For each class, record:

- Allowed surfaces.
- Required evidence.
- Review requirements.
- External authority.
- Recovery behavior.
- Escalation conditions.

Automatic acceptance of one class should not imply authority over another.

### 18.3 Measure reliability and useful throughput

Useful measures include:

- Accepted behaviors per unit of review time.
- Time from task start to accepted result.
- Rejected candidates and their causes.
- Defects found after acceptance.
- User-visible regressions.
- Rollbacks and forward repairs.
- Manual intervention.
- Cost of generating, verifying, reviewing, and repairing.

Counts need denominators.
Ten defects across ten thousand changes differ from ten defects across twenty changes.
The severity and exposure of those defects matter too.

A low rollback rate can mean stability.
It can also mean that rollback is difficult or that defects are being tolerated.
Use several signals.

### 18.4 Watch the unmeasured boundary

Automation often improves what the gate measures.
The product can still degrade elsewhere.

Examples:

- Faster acceptance with worse usability.
- More passing tests with duplicated logic.
- Fewer detected failures because tests stopped discovering files.
- Lower model cost with more human repair time.

Keep occasional broader review of architecture, user experience, and operations.
Use incidents and reviewer corrections to identify missing measurements.

The goal is to improve the system, not to optimize one dashboard value.

### 18.5 Expand one permission at a time

After supervised tasks are consistently accepted, consider automating integration for a narrow class.
Observe that class through enough representative work to see ordinary failures.

Before expanding again, establish:

- Required gates work.
- Failed candidates remain contained.
- Recovery is practical.
- Reviewers are not hiding a growing repair burden.
- Escalations reach an accountable owner.

Choose the observation period by workload and consequence.
There is no universal number of green tasks that proves readiness.

### 18.6 Keep the outer loop narrower than the inner loop

An outer loop turns a signal into a task:

```text
Observed signal
      |
      v
Classify and deduplicate
      |
      v
Create a bounded task
      |
      v
Implement and verify
      |
      v
Apply the permitted acceptance route
```

An alert should not grant unrestricted production authority.
It should select from approved responses or create a reviewable proposal.

Add:

- Attempt limits.
- Rate limits.
- Deduplication.
- Resource budgets.
- Escalation for repeated failure.
- A way to disable the automation.

Prevent a failed repair from creating an endless stream of larger repair attempts.

### 18.7 Make revocation part of the policy

State what causes autonomy to contract:

- A protected boundary was bypassed.
- Evidence was stale or fabricated.
- A serious escaped defect occurred.
- Verification infrastructure became unreliable.
- The task class changed.

Revoke the relevant permission.
Preserve evidence.
Repair the control and demonstrate its rejection behavior before restoring authority.

This is normal operation.
Trust should respond to current evidence.

**Exercise.**
Choose one action class you might eventually automate.
Write the authority, evidence, escalation, and revocation policy.
Keep every external action explicit.

---

## 19. Separate deployment, release, and recovery

Deployment installs an artifact.
Release makes behavior available to users.
Recovery restores an acceptable state after failure.

Design those transitions separately.
An agent finishing implementation does not establish any of them.

### 19.1 Identify the artifact you are deploying

Build from an accepted candidate using the approved process.
Record:

- Source revision.
- Build inputs and relevant versions.
- Artifact identity.
- Configuration requirements.
- Migration requirements.
- Verification evidence.

Run deployment checks against that artifact.
Avoid rebuilding something different after acceptance and assuming the earlier evidence applies.

Configuration is part of the behavior.
A correct artifact with the wrong workspace, secret, or storage configuration can fail catastrophically.

### 19.2 Verify the deployed environment

Pre-acceptance tests do not establish that deployment succeeded.
Run post-deployment checks appropriate to the environment:

- Readiness.
- Configuration compatibility.
- Storage connectivity.
- Schema compatibility.
- A safe critical transaction or synthetic journey.
- Observability.

Keep those checks bounded and non-destructive.
If they mutate state, use an owned synthetic identity or another explicit isolation mechanism.

### 19.3 Control exposure where it helps

Feature switches and limited rollouts can reduce exposure.
Use them when the feature and risk justify the operational cost.

A switch can hide a new interaction.
It cannot undo a destructive migration or recall an external message.
Server-side permissions must still apply when a user bypasses the hidden interface.

Record:

- Who controls exposure.
- Which users or environments receive it.
- What observations trigger expansion.
- What stops the rollout.
- How the switch is retired.

Permanent forgotten switches create another form of architectural drift.

### 19.4 Observe released behavior

Connect operational signals to product guarantees.

For the request tracker:

- Unexpected failure rate.
- Latency under actual load.
- Failed or conflicting state changes.
- Evidence of cross-workspace access attempts.
- Durability and audit consistency.
- User difficulty completing the critical journey.

Separate harmless business rejections from service failures.
A stale update rejected correctly should not look identical to a storage outage.

Use restricted diagnostic data when investigating security-sensitive behavior.
Do not create an information leak in the monitoring system.

### 19.5 Plan recovery before irreversible changes

Recovery can require:

- Reverting an artifact.
- Disabling exposure.
- Applying a forward fix.
- Restoring data.
- Replaying a durable event.
- Compensating for an external effect.

These actions have different risks.

For schema changes, consider a compatible sequence:

1. Add the new representation without removing the old one.
2. Deploy code that can operate during the transition.
3. Migrate and verify data.
4. Switch the primary path.
5. Remove the old representation after the recovery window.

This pattern may be unnecessary for a disposable prototype.
For durable shared data, it can preserve options that a one-step migration would remove.

### 19.6 Treat external effects as durable obligations

When notifications become a real feature, a state change and a delivery can fail independently.
Define what happens between them.

A durable event record may allow a delivery worker to retry after the business update commits.
The design still needs a policy for duplicates, ordering, and exhausted retries.

“Exactly once” should not be promised casually.
Define the observable guarantee and the limits of the external service.
An idempotency key can reduce duplication where the receiver supports it.
It does not solve every failure mode.

Test interruption and retry at the relevant boundaries.
The happy path is only one delivery case.

### 19.7 Drill recovery and feed it back into the foundation

Run a controlled recovery exercise.
Record:

- What failed.
- Which signal detected it.
- Which authority was required.
- What data or artifact was restored.
- How long recovery took.
- What verification established the recovered state.

If the procedure depends on an undocumented command or one person's memory, repair it.
If the agent proposes a production repair, require evidence appropriate to that action class.

Recovery evidence is part of autonomy readiness.
A workflow that can make changes but cannot recover them has a narrow safe operating range.

**Exercise.**
Deploy a candidate into a disposable environment with one intentionally invalid configuration value.
Confirm that readiness fails clearly.
Then exercise the recovery procedure without deleting or recreating unrelated resources.

---

## 20. Build the foundation in the first weeks

Use the first weeks to establish a working loop.
The schedule below describes a sequence of outcomes.
Its duration depends on product risk, environment complexity, and available ownership.

Do not move to the next stage merely because the calendar says a week has passed.
Use the exit evidence.

### 20.1 Stage one: define and start

Establish:

- The first useful user outcome.
- The first-release exclusions.
- Critical failure modes.
- Named decision and operational owners.
- The smallest appropriate stack.
- A clean initialization and readiness route.

For the request tracker, start with creating and reading one request inside an authorized workspace.
Use disposable data.
Treat this slice as supervised foundation work.
It establishes the harness before autonomous build-out.
Do not add assignment, reopening, notifications, and reporting in the same first task.

Write acceptance cases for the important boundaries before implementation.
A member from another workspace must not obtain the request.
Invalid input must not create a partial record.
A successful create must survive a fresh read.

**Exit evidence:**
A fresh environment starts, the baseline journey runs, and the product contract explains what it establishes.

### 20.2 Stage two: enforce the first boundaries

Establish:

- The module map.
- Public entry points.
- The canonical dependency policy.
- Generated enforcement.
- Tested custom rules for genuinely important conventions.
- The trusted route for changing those controls.

Plant a forbidden dependency.
Place a file outside the module map.
Attempt to bypass the approved storage interface.
Demonstrate rejection through the normal verification path.

Keep the first rule set small.
It should cover the foundation's meaningful invariants.
Do not spend the stage inventing rules for mistakes nobody can explain.

**Exit evidence:**
The valid baseline passes, planted violations fail, and the enforcement's scope and limitations are documented.

### 20.3 Stage three: prove one complete feature loop

Choose the next coherent behavior, such as assignment.

Run the whole session protocol:

1. Establish the starting state.
2. Confirm specification-derived cases.
3. Implement the slice.
4. Run domain, contract, and journey checks.
5. Challenge at least one critical claim.
6. Preserve the candidate and evidence.
7. Review and accept through the defined route.

Measure where the effort went.
Was initialization slow?
Did an unclear requirement cause rework?
Did the checker reject legitimate code?
Did the reviewer need to reconstruct missing evidence?

Repair the loop before adding more task volume.

**Exit evidence:**
A reviewer can understand and accept the feature from the preserved packet without rerunning the entire conversation.

### 20.4 Stage four: prove continuity and isolation

Start a new session for reopening.
Use the durable repository context.
Interrupt it before completion and resume from its handoff.

Then run two bounded, independent tasks in separate environments.
Choose tasks with stable interfaces and limited overlap.

Verify:

- Environment state does not leak.
- Owned resources clean up correctly.
- The integrated candidate passes its relevant checks.
- Evidence remains tied to the correct revisions.
- Review capacity remains manageable.

**Exit evidence:**
The project can survive a fresh context, an interruption, and limited parallel work.

### 20.5 Stage five: rehearse release and recovery

Deploy the accepted artifact to a representative non-production environment.
Check configuration, storage, readiness, and the critical journey.

Introduce a controlled failure.
Practice the documented recovery.
Verify the recovered state.

If the product is not yet intended for deployment, establish the equivalent distribution and recovery route.
A local tool still needs installation, data handling, and a way to recover from a bad update.

**Exit evidence:**
The owner can explain how the accepted result becomes available and how an unsuccessful release is contained.

### 20.6 Establish the ongoing cadence

Choose maintenance frequency from actual change volume and risk.

| Cadence | Typical work |
|---|---|
| Every coding session | Baseline, bounded implementation, required verification, handoff |
| Every accepted change | Candidate identity, relevant integration evidence, review |
| Every release | Artifact and configuration checks, exposure decision, monitoring |
| Regular maintenance | Drift, stale documentation, exceptions, dependency updates |
| After incidents or major changes | New acceptance cases, guardrail reassessment, recovery drill |

Some projects need daily maintenance.
Others need a check after each substantial change.
The schedule should make failures visible before they accumulate.

### 20.7 Resist premature platform work

The first month can disappear into building a task controller, a policy engine, and a dashboard.
Those tools may eventually be valuable.
They should answer a demonstrated operating problem.

Begin with explicit records and reliable commands.
Automate repetitive coordination after you can describe its states and failure modes.

A useful test is:

> Can we complete and accept one representative task without the proposed platform?

If the answer is no because verification is missing, build verification.
If the answer is yes but coordination has become repetitive, automate the coordination.

**Exercise.**
Write the next stage's exit evidence.
Remove activities that do not contribute to that evidence.
Keep unresolved questions visible and assign owners.

---

## 21. Recognize the traps before they become habits

The most damaging problems often begin as reasonable shortcuts.
Recognize their trigger and repair the mechanism that allows them to spread.

### 21.1 The comprehensive instruction file

**Trigger:**
Every review correction becomes another paragraph in the agent's entry instructions.

**Failure:**
The file grows while the codebase still permits the mistake.
Old guidance conflicts with new guidance.
The agent spends context on rules unrelated to its task.

**Repair:**
Keep a short map.
Move durable decisions to their own records.
Convert recurring, enforceable corrections into APIs or checks.
Remove duplicated instructions.

**Evidence:**
A fresh session finds the right procedure, and a planted violation fails without relying on the instruction being remembered.

### 21.2 The green empty run

**Trigger:**
A verification command exits successfully.

**Failure:**
No tests or rules ran because a path, identifier, or discovery setting was wrong.

**Repair:**
Assert required identities and execution.
Challenge the installed path with a known violation.
Treat missing work as incomplete verification.

**Evidence:**
Disabling a required rule produces a discovery failure.
Planting its prohibited case produces the intended diagnostic.

### 21.3 The self-approved patch

**Trigger:**
The agent owns implementation, acceptance tests, runner, and manifest.

**Failure:**
The agent can make the patch pass by changing its judge.
The same risk exists in a remote gate if that gate runs only editable branch logic.

**Repair:**
Separate ordinary edits from authoritative policy.
Require independent acceptance for changes to that policy.
Verify the candidate with the approved policy version.

**Evidence:**
Changing both a protected file and its local manifest cannot produce acceptance through the ordinary implementation route.

### 21.4 The architecture diagram without a boundary

**Trigger:**
A module layout is approved, but source imports remain unrestricted.

**Failure:**
Folders look organized while callers reach into internals.
Later agents copy those paths.

**Repair:**
Declare public entry points.
Enforce dependency directions and classification.
Move existing code onto the approved route.

**Evidence:**
A forbidden import fails where ordinary acceptance runs.

### 21.5 The tests that agree with the mistake

**Trigger:**
An agent writes tests only after inspecting its implementation.

**Failure:**
The tests preserve the same wrong assumption.
A required behavior is absent from both code and tests.

**Repair:**
Derive some cases from requirements first.
Use adversarial hypotheses and critical mutations.
Keep acceptance edits on a reviewed path.

**Evidence:**
Removing workspace scope or persistence causes a relevant test to fail.

### 21.6 The screenshot that proves too much

**Trigger:**
The interface looks correct after one successful interaction.

**Failure:**
State was never persisted, an authorization case was not tested, or the screenshot belongs to an earlier build.

**Repair:**
Bind artifacts to candidate identity.
Verify fresh state and denied cases.
Use visual evidence for the properties it can show.

**Evidence:**
The reviewed candidate reproduces the journey, survives a reload, and rejects the relevant unauthorized action.

### 21.7 The sophisticated foundation with no first feature

**Trigger:**
Every possible future risk becomes a required platform component.

**Failure:**
The team maintains scaffolding whose value has not been demonstrated.
The architecture solves an imagined scale rather than the current product.

**Repair:**
Choose the smallest useful blocking baseline.
Build a supervised vertical slice.
Add deeper verification or orchestration when a concrete property needs it.

**Evidence:**
Each foundation component supports a named invariant, workflow, or observed failure.

### 21.8 The cleanup that rewrites the system

**Trigger:**
A maintenance agent receives an open-ended quality goal.

**Failure:**
It changes many domains, alters contracts, and produces a diff too broad to verify.

**Repair:**
Turn drift reports into bounded tasks.
Keep one responsibility or pattern per change.
Apply ordinary acceptance to maintenance.

**Evidence:**
The reviewer can explain the cleanup's scope and verify its preserved behavior in one focused review.

### 21.9 The parallel tasks with a missing seam

**Trigger:**
Several agents build components before agreeing on a contract.

**Failure:**
Each local result passes, but integration requires redesign.
Nobody owns the mismatch.

**Repair:**
Sequence the contract decision.
Assign integration ownership.
Reverify the combined candidate.

**Evidence:**
The integrated journey passes against the actual combined revision.

### 21.10 The autonomous repair loop that cannot stop

**Trigger:**
Every alert launches another agent attempt without limits.

**Failure:**
Repeated failures consume resources, duplicate work, or broaden production changes.

**Repair:**
Deduplicate signals.
Set attempt and authority limits.
Escalate repeated failure.
Keep a revocation path.

**Evidence:**
A repeated controlled failure stops at the defined limit and preserves a useful handoff.

### 21.11 The quality rule that became an obstacle

**Trigger:**
A once-useful rule fires constantly or forces awkward alternatives.

**Failure:**
People suppress it and agents work around it.
The foundation teaches worse code.

**Repair:**
Review the rule's scope, rationale, and approved interface.
Test legitimate near-misses.
Change policy through the foundation route.

**Evidence:**
The revised rule still rejects the harmful pattern and permits the legitimate use without contortion.

### 21.12 The outcome hidden behind activity

**Trigger:**
Progress is reported as generated files, completed tasks, or proposed changes.

**Failure:**
Review, integration, or repair work remains invisible.
The product's reliability may decline while the activity dashboard improves.

**Repair:**
Track accepted behavior, escaped defects, review burden, and recovery.
Keep the states from implementation through release distinct.

**Evidence:**
The team can name which user outcomes are available, which are awaiting acceptance, and which still carry an unresolved risk.

---

## 22. Appendix: working contracts and checklists

Use these templates as starting points.
Delete fields that do not serve a decision.
Add fields when a real failure mode requires them.

The schemas are illustrative and project-local.
They need validation and enforcement before they can carry authority.

### A. Greenfield intake

```yaml
project: "<name>"

user_job: "<outcome>"
primary_users: ["<actor>"]

first_release:
  included: ["<behavior>"]
  excluded: ["<behavior>"]

critical_invariants:
  - id: "<invariant-id>"
    statement: "<must remain true>"
    consequence_of_failure: "<impact>"
    verification_route: "<proposed-check>"

operating_constraints:
  deployment: "<target>"
  data_sensitivity: "<classification>"
  expected_load: "<workload>"
  availability: "<agreed-objective>"
  recovery: "<agreed-objectives>"

ownership:
  product: "<owner>"
  architecture: "<owner>"
  operations: "<owner>"

open_questions:
  - question: "<unresolved-decision>"
    owner: "<owner>"
    needed_before: "<dependent-work>"
    temporary_constraint: "<safe-current-boundary>"
```

Do not fill unknown operational targets with arbitrary numbers.
Keep them unknown, assign an owner, and constrain the work that depends on them.

### B. Foundation gate record

```yaml
gate_id: "<gate>"
candidate_revision: "<revision>"
decision_owner: "<owner>"

decision:
  summary: "<approved-choice>"
  scope: "<where-it-applies>"
  exclusions: ["<outside-scope>"]

artifacts:
  - "<policy>"
  - "<generated-configuration>"
  - "<human-facing-rationale>"

verification:
  valid_baseline: "<report>"
  planted_violations: ["<challenge-report>"]
  known_limitations: ["<observation-boundary>"]

approval:
  status: "<proposed-or-accepted>"
  approved_policy_revision: "<revision>"
  record: "<durable-approval-reference>"

change_route: "<foundation-change-procedure>"
```

Approval should refer to concrete artifacts.
An approval of an earlier draft does not automatically cover later policy changes.

### C. Decision record

```markdown
# Decision: <title>

Status: <proposed, accepted, superseded, retired>

Owner: <accountable owner>

## Context

<Problem and constraints.>

## Decision

<Chosen policy and scope.>

## Alternatives

<Relevant alternatives and why they were not selected.>

## Consequences

<Benefits, obligations, and known limitations.>

## Enforcement and review

<Types, rules, tests, runtime evidence, or human review.>

## Reconsider when

<Conditions that could invalidate the decision.>
```

For a judgment-only decision, say so.
Do not invent a mechanical check that cannot establish the property.

### D. Rule specification

```yaml
rule_id: "<stable-id>"
decision: "<decision-record>"

property:
  prohibited: "<bad-pattern>"
  permitted: "<approved-alternative>"
  scope: ["<paths-or-modules>"]

mechanism: "<build-boundary, syntax, dataflow, or-other>"
observation_limits: ["<what-it-cannot-see>"]

fixtures:
  invalid: ["<fixture-id>"]
  valid_near_misses: ["<fixture-id>"]

diagnostic:
  explanation: "<what-and-why>"
  remediation: "<sanctioned-route>"

meta_checks:
  - required rule identity discovered
  - required fixture identities executed
  - planted violation fails at a production-matched path
  - valid baseline passes

exceptions:
  route: "<review-procedure>"
  required_fields: [reason, owner, scope, review_condition]
```

The rule implementation, fixtures, and activation configuration belong in the same review.
An unactivated correct rule provides no protection.

### E. Foundation change request

```yaml
change_id: "<id>"
requester: "<task-or-person>"

current_policy: "<approved-policy-revision>"
reason: "<requirement-the-current-policy-cannot-support>"
evidence: ["<reproduction-or-analysis>"]

proposal:
  policy_change: "<exact-change>"
  implementation_change: "<required-change>"
  authority_change: "<permissions-affected>"

effects:
  guarantees_retained: ["<invariant>"]
  guarantees_changed: ["<invariant>"]
  affected_features: ["<feature-id>"]

verification:
  updated_fixtures: ["<fixture>"]
  negative_challenges: ["<challenge>"]
  integrated_checks: ["<check>"]

recovery:
  procedure: "<revert-or-forward-repair>"
  limitations: ["<irreversible-effect>"]

approval_owner: "<owner>"
```

Treat a new stricter rule as a policy change too.
It can change which legitimate implementations remain possible.

### F. Feature record

```yaml
feature_id: "<stable-id>"
user_job: "<outcome>"
specification_owner: "<owner>"

actors:
  permitted: ["<actor>"]
  denied: ["<actor>"]

preconditions: ["<required-state>"]
journey: ["<action>"]
expected_results: ["<observable-result>"]
negative_cases: ["<failure-or-denied-case>"]

verification:
  procedure: "<versioned-route>"
  fixture: "<versioned-fixture>"
  required_checks: ["<check-id>"]
  required_artifacts: ["<artifact-type>"]

specification_change_route: "<protected-review-route>"
```

Keep observed pass or fail status in a result record tied to a candidate.
The specification describes the requirement.
The result describes one execution.

### G. Coding task

```yaml
task_id: "<id>"
base_revision: "<revision>"
policy_revision: "<approved-policy>"

objective: "<one-coherent-outcome>"
feature_ids: ["<feature-id>"]

scope:
  owned_surfaces: ["<path-or-interface>"]
  permitted_changes: ["<change-class>"]
  protected_surfaces: ["<path-or-policy>"]
  dependencies: ["<approved-contract-or-task>"]

authority:
  external_reads: ["<approved-service>"]
  external_writes: []
  publication: false
  production_mutation: false

acceptance:
  required_checks: ["<check-id>"]
  evidence: ["<artifact>"]

limits:
  elapsed: "<budget>"
  resources: "<class>"
  attempts: "<bounded-retry-policy>"

handoff:
  result_location: "<durable-location>"
  blocked_state: "<required-record>"
```

A budget is an operating limit.
Reaching it does not establish completion.
Preserve the result and its actual verification state.

### H. Verification result

```yaml
run_id: "<id>"
task_id: "<id>"
feature_ids: ["<feature-id>"]

identity:
  candidate_revision: "<revision>"
  working_tree_changes: "<none-or-record>"
  artifact: "<identity-if-relevant>"
  policy_revision: "<revision>"
  environment_id: "<id>"
  fixture_revision: "<revision>"

checks:
  - id: "<check-id>"
    required: true
    status: "<passed, failed, blocked, skipped, not-applicable>"
    executed_cases: ["<case-id>"]
    result: "<report>"

observations:
  critical_journey: "<result>"
  negative_cases: "<result>"
  diagnostics: ["<artifact>"]
  visual_evidence: ["<artifact-if-relevant>"]

limits:
  unverified_claims: ["<claim>"]
  missing_prerequisites: ["<prerequisite>"]

overall_status: "<derived-from-required-checks>"
```

The overall status should be derived from required results.
Required failed, blocked, or skipped checks prevent verified completion.
“Not applicable” needs a policy basis.

### I. Session handoff

```markdown
# Handoff: <task>

Candidate: <revision or durable patch>

Policy: <approved revision>

Environment: <identity and current lifecycle state>

## Changed

<Behavior and affected surfaces.>

## Verification

<Required checks run, results, and evidence locations.>

## Remaining

<Blocker, uncertainty, or unfinished acceptance.>

## Next action

<One bounded step with its prerequisites.>

## Resource ownership

<Resources preserved, stopped, or awaiting cleanup.>
```

Keep the handoff factual.
Do not claim accepted or released status unless the relevant transition happened.

### J. Autonomy policy

```yaml
action_class: "<class>"
policy_owner: "<owner>"

allowed:
  surfaces: ["<surface>"]
  operations: ["<operation>"]
  environments: ["<environment>"]

required:
  evidence: ["<check-or-artifact>"]
  acceptance_route: "<trusted-route>"
  recovery: "<procedure>"

limits:
  scope: "<maximum-change>"
  attempts: "<limit>"
  rate: "<limit>"

escalate_when:
  - "<uncertainty-or-repeated-failure>"

revoke_when:
  - "<boundary-or-reliability-failure>"

restore_after:
  - "<control-repaired-and-demonstrated>"
```

Authority is specific to the action class.
Do not infer production or publication rights from permission to implement.

### K. Release and recovery record

```yaml
release_id: "<id>"
owner: "<owner>"

artifact:
  identity: "<verified-artifact>"
  source_revision: "<accepted-revision>"
  verification: "<evidence>"

deployment:
  target: "<environment>"
  configuration_revision: "<revision>"
  migration: "<procedure-or-none>"
  readiness_result: "<report>"

exposure:
  scope: "<intended-users>"
  control: "<switch-or-release-mechanism>"
  expansion_condition: "<observed-evidence>"
  stop_condition: "<signal>"

recovery:
  procedure: "<rollback, forward-fix, restore, or-compensation>"
  authority: "<required-role>"
  data_compatibility: "<constraints>"
  last_drill: "<evidence>"

post_release:
  observations: ["<signal>"]
  outcome: "<state>"
```

Record the limits of rollback.
An old artifact may be incompatible with changed data, and external effects may require compensation.

### L. Foundation readiness checklist

Use this checklist before ordinary autonomous implementation.

- [ ] The first user outcome and exclusions are explicit.
- [ ] Critical failures have owners and verification routes.
- [ ] The stack has supported versions and an update route.
- [ ] A clean environment initializes and reaches readiness.
- [ ] Modules, public interfaces, and dependency directions are clear.
- [ ] Architecture policy has one canonical source.
- [ ] Generated enforcement matches that source.
- [ ] Required rules have invalid fixtures and valid near-misses.
- [ ] Required rule and test identities are discovered and executed.
- [ ] Planted violations fail through the installed acceptance route.
- [ ] The valid baseline passes afterward.
- [ ] Required acceptance criteria cannot be weakened through ordinary task edits.
- [ ] The authoritative verifier uses an approved policy.
- [ ] Protected policy changes have an accountable review route.
- [ ] Feature journeys have stable identities and executable procedures.
- [ ] Evidence identifies the candidate, policy, environment, and fixture.
- [ ] Required incomplete checks cannot appear as verified completion.
- [ ] Credentials and mutable resources are scoped to the task.
- [ ] Interrupted work can be preserved and resumed.
- [ ] Cleanup targets only owned resources.
- [ ] Parallel work has isolated state and integration ownership.
- [ ] Release authority is separate from implementation authority.
- [ ] Recovery is documented and tested to the required depth.
- [ ] Maintenance and exception review have owners.
- [ ] Autonomy has escalation and revocation conditions.

Some items will be inapplicable to a small local prototype.
Record why.
Do not mark a missing control inapplicable merely because it is inconvenient.

### M. Review questions for the first accepted feature

Ask:

1. Does the behavior solve the stated user job?
2. Which important assumption did the implementation make?
3. Which permitted path does the code demonstrate to future agents?
4. Which prohibited path is mechanically rejected?
5. Which test was derived from the requirement?
6. Which counterexample challenged the implementation?
7. Does evidence belong to this exact candidate?
8. Can the ordinary agent weaken the acceptance route?
9. Can a fresh session reproduce the result?
10. Can the owner recover from an unsuccessful release?

The first feature is the first complete demonstration of the operating model.
Keep it small enough that these answers remain concrete.
Its structure, verification, and handoff will become examples for the work that follows.
