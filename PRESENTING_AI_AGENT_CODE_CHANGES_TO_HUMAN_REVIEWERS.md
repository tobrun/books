# Presenting AI-Agent Code Changes to Human Reviewers

## Intent, Scope, Risk, and Evidence in Practice

A coding agent finishes a change.
The tests are green.
The pull request contains a confident explanation and an attractive architecture diagram.
The reviewer still has to discover whether the implementation solves the requested problem, whether the explanation matches the code, and whether the tests establish the behavior that matters.

Presentation should make those questions easier to answer.
It should make unsupported claims easier to challenge.
It should preserve access to every changed line.

The practical pattern developed in this book is:

> Present a short account of intent and scope, attach evidence to an immutable change snapshot, guide attention through concerns in risk order, and add a diagram only when a structural question needs one.

That pattern applies before implementation, when a human reviews a plan, and after implementation, when a human reviews the resulting diff.
The two decisions remain separate.
An approved plan establishes a direction.
An approved implementation establishes that particular changes are acceptable.

The running example is a retrying event handler that queues duplicate notification jobs.
The agent proposes a receipt table and an atomic transaction.
The review package must distinguish local job deduplication from the much stronger claim of exactly-once external delivery.
That distinction is small enough to fit in a sentence and consequential enough to change the review decision.

This is a practical book for engineers using coding agents, maintainers reviewing their work, and teams building review tools.
It covers research interpretation, plan review, evidence collection, concern grouping, conditional diagrams, test review, a complete local example, review-surface design, experiments, and rollout.

**Research scope.**
Public primary papers and official documentation were checked on November 24, 2026.
Several studies are recent preprints.
Observational associations, controlled task results, prototype preferences, and vendor documentation are identified separately.
The supplied research guided the questions; the source appendix records qualifications and sources.
This book does not claim that the complete proposed format has been experimentally validated.

**Example scope.**
The Python example uses the standard library and disposable SQLite databases.
It demonstrates a local transaction and a reproducible regression.
It does not implement webhook authentication, a production database migration, an external delivery worker, or a deployable service.
Sample review records are templates unless explicitly identified as results from executing the example.

**Relationship to the other books.**
[AI Engineering Evals in Practice](AI_ENGINEERING_EVALS_IN_PRACTICE.md) develops evaluation methods used in the experiments.
[How to Start a Greenfield Project in the AI Age](HOW_TO_START_A_GREENFIELD_PROJECT_IN_THE_AI_AGE.md) explains repository contracts and enforceable verification.
[Remote Coding Agents in Practice](REMOTE_CODING_AGENTS_IN_PRACTICE.md) covers the execution environments from which review evidence may arrive.
This book concentrates on the handoff to a human who must make a decision.

---

## Contents

1. [Define the decision a reviewer must make](#1-define-the-decision-a-reviewer-must-make)
2. [Read the evidence without overstating it](#2-read-the-evidence-without-overstating-it)
3. [Bind the presentation to a change snapshot](#3-bind-the-presentation-to-a-change-snapshot)
4. [Declare intent and scope before implementation](#4-declare-intent-and-scope-before-implementation)
5. [Present a plan that exposes decisions](#5-present-a-plan-that-exposes-decisions)
6. [Assign risk and ownership explicitly](#6-assign-risk-and-ownership-explicitly)
7. [Build the layered review package](#7-build-the-layered-review-package)
8. [Make verification evidence reproducible](#8-make-verification-evidence-reproducible)
9. [Review tests as behavioral claims](#9-review-tests-as-behavioral-claims)
10. [Group the diff by concern and order attention](#10-group-the-diff-by-concern-and-order-attention)
11. [Replace prefixed diagrams with triggered deltas](#11-replace-prefixed-diagrams-with-triggered-deltas)
12. [Build a diagram pipeline that preserves provenance](#12-build-a-diagram-pipeline-that-preserves-provenance)
13. [Execute the complete local example](#13-execute-the-complete-local-example)
14. [Present the example for human review](#14-present-the-example-for-human-review)
15. [Adapt the format to different changes](#15-adapt-the-format-to-different-changes)
16. [Check narrative and diff consistency](#16-check-narrative-and-diff-consistency)
17. [Design a review surface around human control](#17-design-a-review-surface-around-human-control)
18. [Run the human review and revision loop](#18-run-the-human-review-and-revision-loop)
19. [Evaluate the presentation with controlled tasks](#19-evaluate-the-presentation-with-controlled-tasks)
20. [Roll out the practice and operate it](#20-roll-out-the-practice-and-operate-it)
21. [Appendix: templates, qualifications, and sources](#21-appendix-templates-qualifications-and-sources)

---

## 1. Define the decision a reviewer must make

The reviewer receives a proposal to change a system.
The proposal may contain code, tests, configuration, documentation, dependencies, and instructions that alter future agent behavior.
The human needs to decide whether that proposal can be accepted under the project's requirements and operating constraints.

That decision contains several smaller judgments.

| Judgment | Question | Useful evidence |
|---|---|---|
| Intent | Is this the problem we meant to solve? | Task, acceptance criteria, approved decisions |
| Scope | Did the implementation stay within the requested work? | Declared paths, actual diff, deviations |
| Behavior | Does the change produce the required result? | Executed tests, traces, manual observations |
| Compatibility | What existing behavior might break? | Callers, public contracts, upgrade and rollback checks |
| Risk | Where would a mistake matter most? | Trust boundaries, persistent state, operational effects |
| Ownership | Who is qualified to accept the remaining uncertainty? | Named reviewer and explicit unresolved questions |

A file summary can help locate work.
It usually cannot answer all of these questions.
“Updated event handling and added tests” says little about what event identity means, whether tenants share an identity namespace, or whether retries can lose work.

### 1.1 Start with the observable problem

Consider a notification service.
An upstream sender retries an event when it does not receive an acknowledgment.
The current handler inserts a notification job on every request.
Two identical retries produce two jobs.

A useful intent statement is:

> Prevent a retry of the same tenant's event from creating another local notification job.

This gives the reviewer a behavior to inspect.
It identifies the identity boundary.
It also limits the promise to local queue insertion.

Compare:

> Improve reliability using robust idempotency and comprehensive validation.

The second statement names desirable qualities.
It gives the reviewer no precise outcome.
Words such as “robust” and “comprehensive” carry no useful evidence unless the proposal defines and demonstrates them.

### 1.2 Make the approval object explicit

Review approval should identify a snapshot.
For ordinary Git work, that includes the base commit, the head commit, and the comparison rule.
For a deployed preview, it also includes the deployed build and relevant configuration.

A comment saying “approved” without a snapshot is ambiguous once the agent pushes another commit.
The review tool may handle dismissal or stale approvals.
The presentation should still make its own identity visible.

The reviewer should be able to answer:

> Which code did I inspect, and which evidence belongs to it?

This becomes especially important when an agent continues editing after a review conversation has begun.

### 1.3 Preserve disagreement

A review package should expose the parts that could reasonably be rejected.

For the notification change:

- Should duplicate event identity be scoped to a tenant?
- Should reuse of an event ID with a different order be rejected?
- How long must receipts remain available?
- What happens when the transaction cannot acquire a write lock?
- Does an existing deployment need a migration and historical backfill?

The agent may have sensible answers.
The answers should remain visible decisions with supporting reasons.
A smooth narrative that hides these choices makes the reviewer reconstruct them from implementation details.

### 1.4 Optimize the handoff, not the transcript

The agent may have read hundreds of files and tried several designs.
The reviewer needs the parts of that work that affect acceptance.

Useful decision history includes:

- The requirement that selected the chosen approach.
- A rejected alternative with a relevant limitation.
- An assumption that remains unverified.
- A deviation that enlarged the work.

A full session transcript is a secondary investigation artifact.
It can explain a surprising action.
It should not be required reading for every ordinary change.

The handoff should be understandable to a reviewer who did not watch the agent work.

---

## 2. Read the evidence without overstating it

Research provides reasons to improve presentation.
It does not establish that every proposed presentation feature improves defect detection.
Separating those claims keeps the design honest.

### 2.1 Use evidence categories

| Category | What it can support | What it cannot establish alone |
|---|---|---|
| Controlled experiment | An effect in the studied task and population | The same effect in every repository |
| Repository mining | An association in recorded development activity | A causal effect of the presentation |
| Participatory design or survey | Problems, preferences, and perceived usefulness | Observed defect detection or time saved |
| Vendor documentation | Documented product behavior and configuration | Independent effectiveness |
| Practitioner guidance | A useful hypothesis or operating practice | A measured general effect |
| Proposed design | A coherent workflow to implement and test | A proven outcome |

Use these categories near the claim, especially for numbers.
Readers should not need to inspect the bibliography to learn that an apparent result is a preference survey.

### 2.2 Understanding gives presentation its purpose

Bacchelli and Bird's study of modern code review at Microsoft identified understanding the code and change as a central challenge.
The study supports treating review as an understanding task with several outcomes, rather than reducing it to finding defects.
It does not establish a specific optimal PR template. [S1]

Google's case study combined approximately nine million reviewed changes with interviews and a survey.
It described a review practice dominated by small changes, including a median change size of 24 lines.
Those observations give useful context for keeping routine review packages short.
They are descriptions of Google's practice, not a universal size limit. [S2]

The practical design implication is a proposed one:
make the purpose and relevant boundaries visible before asking someone to reason through implementation.
Keep the amount of presentation proportional to the change.

### 2.3 Order changes attention

Fregnan and colleagues studied 219,476 PRs and then ran a controlled experiment with 106 participants.
Earlier files received more comments in the mining study.
In the experiment, one seeded defect type was unaffected by position; the other had 64% lower odds of detection when its file appeared last rather than first.
That is a relative odds result for one defect type.
It is not a 64 percentage point reduction in detection for all defects. [S3]

More recent work adds useful limits.
An industrial file-ordering study examined alternatives to alphabetical order.
A large developer survey reported dissatisfaction with alphabetical order and interest in logical grouping.
An outcome-level mining study found modest associations between file position and later bug-fixing involvement.
Attention, preferences, and escaped defects are different outcomes. [S4, S5, S6]

Risk-first ordering is therefore a reasonable design hypothesis.
It deserves local evaluation rather than being described as a proven universal optimum.

### 2.4 Grouping has a narrower benefit than “faster review”

The supplied research describes work by Barnett and colleagues on decomposing changes and a controlled study by Di Biase and colleagues on untangled review.
The publisher-deposited abstract of the latter reports fewer wrongly reported issues without an improvement in rationale understanding or the number of found defects. [S22]
The operating lesson is to separate unrelated concerns so reviewers make more precise judgments.
Do not promise a speedup from grouping alone.
The appendix distinguishes the checked abstract from the supplementary research and full-text access limits.

A concern can be internally related while spanning several files.
A single file can also contain unrelated concerns.
Directory boundaries are useful clues, not a complete grouping rule.

### 2.5 Specific feedback requests help give review a target

Pirouzkhah, Wurzel Gonçalves, and Bacchelli analyzed 80,000 PRs across 156 projects.
Their study associated description elements with review outcomes.
Stating the desired type of feedback was uncommon, appearing in 16.20% of descriptions, and was a strong predictor of engagement and acceptance.
This is observational evidence; authors who request useful feedback may differ in other ways. [S7]

The proposed practice is inexpensive:

> Please inspect whether receipt identity and job insertion remain atomic when two connections accept the same event.

That request gives an appropriate reviewer a concrete task.
“Please review” provides much less direction.

### 2.6 Description consistency matters

Gong and colleagues analyzed 23,247 agent-authored PRs across five agents.
They identified high message–code inconsistency in 1.7%.
Acceptance was 28.3% for those PRs and 80.0% for the comparison group; reported merge time was also higher.
The most common inconsistency type claimed changes that were not implemented.
The study establishes an association, not that editing a description would by itself cause acceptance. [S8]

This supports checking claims against the diff.
It also argues against treating every generated description as unreliable.
The important problem is a consequential mismatch that may be relatively uncommon.

An earlier study of 18,256 PRs using generated descriptions reported improved review and merge outcomes in quasi-experimental analyses.
Developers supplemented generated text with missing intent, testing information, and other context.
Generated descriptions can be useful; their usefulness does not remove the need to verify their contents. [S9]

### 2.7 Trust and attention are separate from correctness

A participatory design study with JetBrains involved 17 practitioners in discovery and 43 in validation.
It proposed overview, file-analysis, and snippet-review levels and reported positive perceived usefulness.
The often-cited expectation of reduced effort was a respondent expectation, not an observed reduction during real reviews. [S10]

An eye-tracking study of LLM-labeled code examined attention and scan behavior.
Additional fixation time should not be reported as improved defect detection, because that outcome was not measured. [S11]

Perry and colleagues found that participants using an AI assistant produced less secure solutions on studied tasks while expressing greater confidence in their security.
This provides a reason to avoid persuasive reassurance as a substitute for evidence.
It does not quantify the security risk of every current coding agent. [S12]

### 2.8 Read the diagram evidence carefully

The VisPR preprint examined approximately 2.47 million human-authored PRs in repositories where visual artifacts were an established practice.
13.3% contained visuals.
Diagrams were a small share of sampled visual artifacts, rather than 2% of all PRs.
Matched comparisons found no significant difference in overall time to merge or churn-normalized review efficiency, while visual PRs attracted more scrutiny of user-visible behavior.
The study was observational and did not isolate diagrams for agent changes. [S13]

That is evidence against promising a general review-speed benefit from adding pictures.
It is not evidence that a correct sequence diagram cannot help answer a particular ordering question.

Studies of LLM-generated UML and sequence diagrams report correctness and completeness problems.
Much of that work starts from requirements text.
The “Code Faithfulness” dimension in the UML taxonomy concerns generated PlantUML's agreement with the model's reasoning, not direct agreement with repository source code.
It should not be presented as a measured error rate for diff-grounded diagrams. [S14, S15]

### 2.9 State the resulting hypothesis

The rest of this book develops the following proposed format:

1. Identify intent, scope, and the change snapshot.
2. Expose consequential assumptions and deviations.
3. Attach reproducible evidence and its limits.
4. Group related changes and put high-risk concerns early.
5. Add a small structural delta when it answers a relevant question.
6. Preserve the complete diff and human decision authority.

The literature supports several ingredients.
The combined format still needs direct evaluation on agent changes.
That evaluation is part of operating the practice.

---

## 3. Bind the presentation to a change snapshot

A review package is a description of particular bytes and observations.
Without a stable identity, the reviewer cannot tell whether the explanation, diagram, and green checks describe the same change.

### 3.1 Record three Git identities

Use:

- **Target tip:** the target branch commit observed when preparing the package.
- **Comparison base:** the commit from which the presented change is calculated.
- **Head:** the proposed implementation commit.

The comparison base may be the merge base between the target tip and head.
It may instead be an explicitly selected parent for a stacked change.
State the rule.

For a feature branch:

```bash
git fetch origin
target_tip="$(git rev-parse origin/main)"
head_sha="$(git rev-parse HEAD)"
comparison_base="$(git merge-base "$target_tip" "$head_sha")"

git diff --name-status "$comparison_base" "$head_sha"
git diff --stat "$comparison_base" "$head_sha"
git diff --check "$comparison_base" "$head_sha"
```

These commands inspect committed history.
They do not prove that a working directory is clean.
Check that separately before collecting head-bound evidence.

Use task-specific variable names in scripts.
Do not overwrite shell variables such as `HOME`.

### 3.2 Explain the baseline

A regression test may fail on the comparison base.
A deployment integration check may need to run against the current target tip instead.
A stack may depend on a lower change that is not in the target branch yet.

These are different baselines.
Record the tested revision for each run.

For example:

```yaml
comparison:
  rule: merge-base
  target_tip: "<full-commit>"
  base: "<full-commit>"
  head: "<full-commit>"
evidence:
  regression_base: "<full-commit>"
  regression_head: "<full-commit>"
  integration_target: null
```

This is a proposed record format.
`null` means the integration check has not been recorded.
It must not render as a green check.

### 3.3 Treat new commits as new presentation work

If the agent edits a file after evidence collection:

1. Identify the new head.
2. Regenerate the changed-file inventory.
3. Refresh affected claims and code anchors.
4. Rerun the checks whose conclusions may have changed.
5. Mark prior evidence as belonging to the previous head.

A timestamp later than the last edit is useful context.
An immutable revision and a clean execution checkout are stronger evidence of identity.
A timestamp alone cannot establish which bytes were tested.

A documentation-only edit may not invalidate an application test.
Preserve that distinction explicitly if reusing evidence.
State which source inputs are unchanged and which check result is carried forward.

### 3.4 Handle rebases and target movement

A rebase changes commit identity.
Even when the patch appears similar, conflict resolution or a newer target can alter behavior.
The presentation should refresh the comparison and any evidence affected by the integration.

Do not silently reuse an old before/after diagram as if it were derived from the new head.
Do not label a test run “current” because the same branch name still exists.

A review surface can show:

> This package describes head H1. The live PR is at H2. Review the updated package before approving H2.

That is useful state.
It keeps a reviewer from approving through stale context.

### 3.5 Use stable anchors with readable labels

An anchor should include:

- The revision.
- The file path.
- The relevant range or symbol.
- Which side of the comparison it describes.

A symbol label helps a person navigate.
The revision and range identify the actual evidence.
Symbol names alone can become ambiguous after a rename or when several modules export the same name.

For removed behavior, link to the base side.
For added behavior, link to the head side.
For a claim of no change, link to the relevant unchanged context and say that it is context, rather than a changed hunk.

### 3.6 Bind artifacts beyond Git

Some observations depend on more than source code.

| Artifact | Additional identity to record |
|---|---|
| UI screenshot | Build, route, viewport, fixture, feature flags |
| Runtime trace | Build, configuration, input, instrumentation version |
| Database migration rehearsal | Schema versions, database engine, fixture |
| Benchmark | Dataset, hardware, dependency versions, run conditions |
| Static graph | Extractor version, entrypoints, resolution limits |
| AI review finding | Reviewed head, reviewer tool version where available |

A screenshot from another build is still an image.
It is weak evidence about the proposed head.

---

## 4. Declare intent and scope before implementation

The agent's task should establish a boundary that the final diff can be compared against.
This boundary is especially useful when the agent finds adjacent cleanup opportunities.

### 4.1 Write acceptance criteria as observations

For the running example:

```text
Intent:
Prevent retries from creating duplicate local notification jobs.

Acceptance criteria:
- The first event for a tenant queues one job.
- Retrying that event with the same order creates no additional job.
- The same event ID used by a different tenant remains independent.
- Reusing an event ID for another order is rejected without another job.
- Receipt creation and job insertion succeed or roll back together.
```

Each line can become a question for implementation and evidence.
The criteria avoid implementation vocabulary until the transaction requirement, which is deliberately part of the design contract.

An acceptance criterion should not merely repeat a function name:

> Add `deduplicate_event()`.

That line asks for an implementation artifact.
It does not define the resulting behavior.

### 4.2 Separate requested scope from discovered scope

A plan may propose:

```yaml
declared_scope:
  - path: receiver.py
    purpose: local event acceptance and transaction
  - path: test_receiver.py
    purpose: regression and boundary checks
excluded_work:
  - external notification delivery
  - existing database upgrade
  - receipt retention
```

The exclusions clarify the proposal.
They do not automatically make missing production requirements acceptable.
If the actual task is to deploy this behavior into an existing service, a database upgrade cannot simply be excluded to keep the diff small.
The author must include it or explain why deployment is not ready.

### 4.3 Give scope changes a consequence

Suppose the agent discovers that job insertion is hidden behind a queue adapter.
The initial file list was incomplete.

A useful deviation is:

> Add `queue_store.py` to scope because acceptance currently cannot share a transaction with job insertion. The adapter's transaction contract needs review.

A weak deviation is:

> Touched an additional helper as needed.

The useful version connects the new path to the intended behavior and exposes the additional review obligation.

Scope declarations are living contracts.
They are not a prohibition against necessary discovery.
They make that discovery reviewable.

### 4.4 Use a complete path comparison

Classify every changed path:

| Path status | Meaning | Reviewer action |
|---|---|---|
| Declared and changed | Work occurred where expected | Inspect under its concern |
| Declared and unchanged | Planned work was unnecessary or missing | Explain which |
| Undeclared and changed | Scope expanded | Inspect the reason and risk |
| Renamed or deleted | Identity or behavior moved or disappeared | Examine both sides |
| Generated or vendored | Source of change needs attribution | Inspect input and regeneration method |

Do not only list unexpected files.
A planned migration that never appeared can be more important than an unexpected documentation update.

For directories, distinguish an exact path from a prefix rule.
A blanket declaration such as `src/**` provides little protection against scope drift.
Prefer concrete modules with a reason.

### 4.5 Record assumptions with verification status

Useful assumption:

> Tenant identity is supplied by the authenticated caller. Verification: inspect the real endpoint adapter. Status: outside the local example; required before production integration.

Weak assumption:

> Authentication works normally.

The useful version identifies the trusted input and the missing check.
It gives a human owner something to settle.

An assumption record can contain:

```yaml
id: A1
statement: Event identity is unique only within a tenant.
source: upstream contract
verification: review the upstream event specification
status: unresolved
owner: event-platform-maintainer
```

Use “observed,” “documented,” “inferred,” or “unresolved.”
Avoid presenting all assumptions as facts because they appear in a formal record.

---

## 5. Present a plan that exposes decisions

A plan should let a human change the proposed approach while change is still inexpensive.
It needs enough concrete detail to expose consequences.
It should remain short enough to inspect in one sitting.

### 5.1 Lead with the decision requested

For the event change:

> Confirm that `(tenant_id, event_id)` is the receipt identity and that reuse for a different order should be rejected.

This is the choice that determines schema and error behavior.
Put it early.

If no human decision is required, say what the plan is for:

> This plan records implementation scope and verification obligations for an already specified behavior.

Not every plan needs to manufacture uncertainty.

### 5.2 Compare real alternatives

| Approach | Why consider it | Consequential limitation |
|---|---|---|
| In-memory set of event IDs | Simple local prototype | State disappears on restart and is not shared between workers |
| Unique job key only | Small database change | Needs a policy for conflicting payloads and any other acceptance state |
| Receipt and job in one transaction | Explicit identity and conflict behavior | Adds durable state and a retention obligation |

The alternatives should explain why the chosen design fits the task.
There is no need to enumerate every possible architecture.

The receipt design is not automatically superior.
If the job table already has the right identity constraint and stores the accepted payload, another table may be unnecessary.
A plan should justify new abstractions through requirements.

### 5.3 Name files and responsibilities

“Add idempotency across the service” is too abstract.
“Change `receiver.py` to accept an authenticated tenant and insert a receipt plus job within one transaction” is reviewable.

For a larger service, list responsibilities:

- Endpoint: authenticate, parse, validate, map results to responses.
- Store: enforce event identity and transaction semantics.
- Worker: consume committed jobs.
- Migration: add schema and define historical behavior.
- Tests: reproduce retry and concurrency cases.

This makes omitted work visible.
It also helps reviewers assess whether the proposed split is excessive.

### 5.4 Expose failure paths in the plan

Successful flow is rarely the only design decision.

For the example:

| Condition | Proposed behavior |
|---|---|
| New identity | Commit receipt and job, return `queued` |
| Same identity and order | Return `duplicate`, create no job |
| Same identity and different order | Raise a conflict, preserve existing job |
| Job insertion fails | Roll back receipt and job changes |
| Database remains locked | Surface the database error; caller decides retry handling |

The final line deliberately leaves an integration decision visible.
The local function cannot decide the upstream sender's retry protocol.

### 5.5 Include a verification plan before editing

A useful plan identifies:

- Which regression should fail on the old behavior.
- Which existing behavior must remain.
- Which boundary cases need direct checks.
- Which integration cannot be exercised locally.
- Which owner accepts that remaining limitation.

For the running example:

> Run a duplicate retry test against both implementations using the same test bytes. Add separate head checks for tenant independence, conflict behavior, rollback, concurrent connections, and restart persistence.

This helps prevent the agent from choosing tests after implementation merely to match what it wrote.

### 5.6 Label plan diagrams as proposals

A proposed sequence diagram can help explain transaction ordering.
At plan time it is a design sketch.
It has not been derived from an implementation that does not exist yet.

Label it:

> Proposed transaction flow; implementation verification pending.

After implementation, replace it with a code-supported delta or remove it.
Leaving the plan sketch in the final PR without updating its status blurs design intent and observed behavior.

### 5.7 Keep plan approval distinct from implementation approval

The plan reviewer may accept event identity and the transaction approach.
The implementation may still:

- Use a global event key.
- Insert the job after committing the receipt.
- Treat every database error as a duplicate.
- Swallow a conflict.
- Skip a migration needed by the real service.

Approval of the design does not establish that the code implements it.
The post-change presentation should show where the implementation follows and departs from the approved plan.

---

## 6. Assign risk and ownership explicitly

Risk labels help allocate attention when they explain a possible failure.
A bare “low risk” badge is another unsupported claim.

### 6.1 Separate consequence and uncertainty

Two questions determine review depth:

1. What could happen if this change is wrong?
2. How much relevant uncertainty remains?

A familiar one-line permission change can have a severe consequence.
A large documentation reorganization can have modest operational consequences but a substantial comprehension burden.
Lines changed are useful workload information.
They are an incomplete risk measure.

### 6.2 Use a small, local tier policy

| Tier | Example conditions | Suggested review obligation |
|---|---|---|
| Low | Local behavior, limited callers, easy recovery | Inspect intent, scope, evidence, and the complete small diff |
| Medium | Shared behavior, concurrency, dependency or persistent-state change | Inspect concerns, failure cases, compatibility, and relevant tests |
| High | Authorization, credentials, destructive data operations, public contract or difficult rollback | Named domain owner, explicit unresolved questions, integration and recovery evidence |

This is a starting policy.
Teams should adapt it to their systems.
An internal label must not override branch protection or established ownership requirements.

The running example is medium for a local teaching exercise.
The same logic inside a production billing pipeline could be high.
Context determines consequence.

### 6.3 Explain blast radius concretely

“Touches the backend” is not useful.

Useful statement:

> Changes acceptance of retried order events for every tenant. Adds persistent receipt state. Does not change the external worker in this proposal. Existing deployments need a separate migration decision before release.

That identifies affected behavior, state, and an unresolved deployment condition.

Blast radius can include:

- Public callers.
- Shared libraries.
- Persisted records.
- Tenants or customers.
- Rollout regions.
- Credentials and permissions.
- Future CI or agent behavior.

### 6.4 Review control surfaces early

Changes to CI, permission policy, test selection, dependency resolution, and repository instructions affect how the rest of the proposal is validated or executed.
They deserve early inspection when present.

A source diff accompanied by “skip this failing integration test” changes both the application and its verification conditions.
The reviewer should see that relationship immediately.

Do not automatically label configuration as low risk.
A small workflow change can disable the checks on which approval relies.

### 6.5 Preserve the origin of risk judgments

Keep separate:

- Author or agent risk assessment.
- Deterministic policy flags.
- Independent AI reviewer findings.
- Human assessment.

For example:

```text
Agent assessment: medium, because receipt and job share a transaction.
Policy flag: persistent-state change requires storage-owner review.
AI reviewer finding: inspect tenant identity at the endpoint boundary.
Human disposition: endpoint integration remains outside this local example.
```

These are different signals.
Combining them into one confidence percentage obscures their provenance.

### 6.6 Make downgrades reviewable

An agent may classify a change as low risk because tests pass.
Passing tests reduces some uncertainty.
It does not erase consequence.

If a high-risk rule is downgraded, record:

- Which rule triggered.
- Why the rule does not apply in this instance.
- Who accepted the exception.
- Which evidence supports it.

An agent-generated explanation alone should not silently remove required domain review.

### 6.7 Assign ownership to unresolved questions

An owner is someone qualified and authorized to decide.
It is not merely the person nearest the PR.

For an event pipeline:

- The upstream owner defines event identity and retries.
- The storage owner reviews transaction and migration behavior.
- The delivery owner reviews external side effects.
- The security owner reviews authentication and tenant binding.

One person may fill several roles.
The review package should still identify the questions requiring each kind of expertise.

---

## 7. Build the layered review package

Layers allow a reviewer to orient quickly and then inspect details.
They do not form a mandatory linear reading sequence.
The complete diff should be available from the start.

### 7.1 Layer 0: the decision header

The header should answer, in roughly half a minute:

- What problem does this solve?
- Which snapshot is under review?
- Why does it have this risk tier?
- Did scope change?
- What decision or inspection is requested?

Example:

```text
Intent: Stop duplicate local jobs when a tenant's order event is retried.
Snapshot: base B, head H; full identities in the evidence record.
Risk: Medium. Changes durable state and concurrent acceptance.
Scope: receiver.py and test_receiver.py; no unexpected paths.
Deviation: Added an explicit conflict result for reused event IDs.
Requested review: Inspect tenant-scoped identity and transaction rollback.
```

The base and head placeholders are unfilled template values.
A real package must replace them.

The header should not list every changed function.
That information belongs in the tour and diff.

### 7.2 Layer 1: evidence and limits

Show the important behavior before and after:

> Before: two identical calls create two local jobs. After: the second call returns `duplicate`; one job remains.

Then identify the evidence:

- The regression command.
- Base and head outcomes.
- Additional boundary checks.
- Test changes.
- Integration or manual checks.
- What was not verified.

Use exact language for status:

| Status | Meaning |
|---|---|
| Passed | The named command succeeded under the recorded conditions |
| Expected regression failure | The old behavior failed the intended assertion |
| Failed | A required check did not succeed |
| Not run | No execution evidence exists |
| Inconclusive | Execution occurred but did not resolve the question |
| Stale | The evidence describes another snapshot or configuration |

“Verified” without a named property and method is too broad.

### 7.3 Layer 2: the review tour

The tour groups work into concerns.
Each concern includes:

- A short purpose.
- Code anchors.
- The possible failure.
- Evidence links.
- A question for the reviewer when needed.

For the running example:

1. Transaction and receipt identity.
2. Duplicate and conflict behavior.
3. Caller validation and integration assumptions.
4. Regression and boundary tests.

The highest-risk concern appears first.
Within it, show the schema contract before the code that depends on that contract.

This reconciles risk order with dependency order.
It does not require choosing one global ordering rule for every file.

### 7.4 Layer 3: a triggered structural delta

Include a visual only if it helps inspect a changed relationship or sequence.

For this change, the useful question is:

> Do receipt creation and job insertion happen within the same transaction?

A sequence delta can answer that question.
A diagram of the entire notification platform would spend attention on unchanged components.

The diagram needs provenance, code anchors, and a status.
“Human-authored schematic checked against the example” is a truthful status for a teaching illustration.
“Extracted from production traces” would require actual trace evidence.

### 7.5 Layer 4: the complete diff

The diff is the authoritative record of changed code.
Render it in tour order where possible.
Preserve an ordinary file view and raw patch.

Tests can begin collapsed until the reviewer has formed expectations about the implementation.
The collapse must remain visible:

> Test changes: two files; one assertion removed. Review required.

A hidden test section with no count or alert makes test weakening easy to miss.
For a tests-only change, tests are the primary implementation and should appear first.

### 7.6 Use proportionate defaults

| Change | Presentation |
|---|---|
| Small local correction | Header, relevant evidence, direct diff |
| Multi-file behavioral change | Header, evidence, concern tour, complete diff |
| Flow or boundary change | Add a supported structural delta |
| High-consequence change | Add domain ownership, unresolved questions, rollout and recovery evidence |
| Investigation without implementation | Findings, evidence, recommended next decision |

Do not require empty diagram sections.
Do not make a five-layer UI a prerequisite for reviewing a typo.

### 7.7 Keep reassurance out of the evidence layer

Replace:

> All tests pass and the implementation is safe.

With:

> The duplicate regression and six boundary tests passed on the recorded head. Production migration and external delivery were not exercised.

The second statement gives a reviewer a basis for accepting or rejecting the remaining uncertainty.
It is shorter than a long confident explanation and more useful.

---

## 8. Make verification evidence reproducible

An evidence record should let someone understand what was exercised and reproduce the check.
It should also make gaps easy to see.

### 8.1 Record a claim, method, result, and boundary

For each important claim:

| Field | Example |
|---|---|
| Claim | Identical retries create one local job |
| Method | Execute the same regression test against base and head |
| Result | Base fails the count assertion; head passes |
| Identity | Implementation revisions and test-file digest |
| Conditions | Python version, SQLite version, disposable database |
| Boundary | Does not test the production endpoint or external delivery |

The boundary is part of the evidence.
It prevents a narrow result from becoming a broad assurance.

### 8.2 Use the same test for the counterfactual

A strong regression demonstration applies the same new test to both versions.
It changes the implementation under the test while preserving the test's bytes and relevant conditions.

Otherwise, two successful runs of different tests may establish little about the change.
The head test might use an easier fixture.
The base test might expect the old behavior.

Record:

- The test content digest.
- How the test was applied to the base.
- The base implementation identity.
- The head implementation identity.
- The exact failing assertion.

If the new test requires a public interface that does not exist on base, a direct run may be impossible.
Use a minimal compatibility adapter that preserves the intended observation, explain it, and review it.
Alternatively, demonstrate the old behavior through its existing interface.
Do not silently redesign the test until it fails for an unrelated reason.

### 8.3 Distinguish the expected failure from broken execution

For the event regression:

```text
Expected old behavior:
Assertion fails because local job count is 2 rather than 1.
```

These are different:

```text
ImportError: no module named receiver
DatabaseError: cannot open fixture
TimeoutError: dependencies did not install
```

An environment failure does not demonstrate the bug.
Show the old behavior's intended failure explicitly.
If the regression never reaches the assertion, the counterfactual is inconclusive.

### 8.4 Use disposable execution environments

Run base and head in separate temporary directories or worktrees.
Avoid switching the author's working directory repeatedly during active editing.
Separate databases and output directories prevent one version from contaminating another.

A generic workflow is:

```text
Resolve immutable base and head.
Create separate disposable checkouts.
Install the recorded dependencies.
Apply the same regression test to each.
Run the base check and retain its expected failure.
Run the head check and retain its result.
Collect the complete head suite and relevant manual observations.
Publish identities and bounded logs.
```

Use reviewed execution procedures for untrusted code.
Tests can execute arbitrary repository behavior.
A presentation system should not fetch code and run it with broad credentials merely because the task is called verification.

### 8.5 Preserve full output and publish a useful excerpt

The summary should show:

- Command.
- Exit code.
- Key assertion or result.
- Artifact location.
- Snapshot.

The full log should remain accessible under the project's retention policy.
An excerpt is useful for orientation.
It should not remove a failing prerequisite or warning that changes interpretation.

For sensitive systems, retain raw logs privately and publish a redacted excerpt.
Mark that redaction.
Do not expose credentials, customer payloads, or private tokens in a public PR.

### 8.6 Use manual evidence deliberately

Manual checks can establish behavior that automated tests do not currently cover.
They need a procedure and observation.

Weak:

> Tested manually.

Useful:

> Opened the order page in the recorded preview build, submitted the same synthetic event twice, and observed one queued job. The browser result and database query are attached. Multi-worker delivery was not exercised.

For a UI change, record route, state, fixture, and viewport.
For an operational change, record the relevant build and environment.
Use screenshots to show an observable state.
Use traces or logs to show ordering and runtime events.

### 8.7 Record negative evidence honestly

“No errors appeared in the console” is an observation during a particular interaction.
It is not proof that there are no errors.
“The static analyzer found no callers” is an observation under a resolver's scope.
It is not proof that dynamic callers do not exist.

Attach the method's limits to absence claims.
For example:

> No statically resolved references outside this package; plugin loading and reflection were not analyzed.

### 8.8 Reuse evidence only when its inputs are unchanged

A spelling correction in the PR body does not require rerunning application tests.
A transaction edit does.
A dependency update may change execution even if application source is identical.

A carried-forward record should identify:

- The earlier run.
- The unchanged source and dependency inputs.
- The change that occurred afterward.
- Why that change does not affect the observed property.

This is a reasoned reuse decision.
It should not appear as a newly executed run.

---

## 9. Review tests as behavioral claims

Tests are another part of the proposal.
Agent-written tests can be useful, mistaken, incomplete, or shaped around the implementation's assumptions.
Their presence does not establish correctness.

### 9.1 Form an independent expectation first

Before reading the test names, ask what behavior should distinguish the fix from the bug.

For duplicate events:

> Two calls with the same tenant, event, and order should leave one job.

Then ask what mistakes a plausible implementation might contain:

- Global identity suppresses another tenant's event.
- Receipt commits before job insertion.
- Conflict handling treats a different order as a harmless retry.
- Concurrent callers both pass a check before inserting.
- State exists only in memory.

Those expectations give the reviewer a basis for judging the tests.
They reduce dependence on the agent's chosen framing.

### 9.2 Inspect observable assertions

A behavioral test should exercise a public or executable interface and inspect a meaningful outcome.

For the example:

```python
accept_event(path, "tenant_a", "event_1", "order_1")
accept_event(path, "tenant_a", "event_1", "order_1")
self.assertEqual(job_count(path), 1)
```

This checks persisted behavior.
It can fail if the implementation creates duplicate work.

A test that opens `receiver.py` and searches for `BEGIN IMMEDIATE` checks a string.
The string may be in a comment or an unused branch.
It does not establish atomic job insertion.

Source inspection remains useful during review.
It should not masquerade as an execution test.

### 9.3 Show changed and removed assertions

A test-change alert should identify:

- Deleted tests.
- Skipped tests.
- Removed or weakened assertions.
- Replaced fixtures.
- Reduced parameter sets.
- Changes to test discovery.
- Coverage exclusions.
- CI conditions that omit execution.

A deterministic scanner can identify suspicious syntax or configuration changes.
A reviewer must decide whether the change actually weakens the intended contract.
An assertion may be obsolete because behavior deliberately changed.
That still deserves an explanation.

### 9.4 Review correlated assumptions

The agent may write implementation and tests using the same mistaken assumption.
For example, both may treat event ID as globally unique.
All tests can pass while another tenant's event is discarded.

Countermeasures include:

- Derive tests from acceptance criteria established before implementation.
- Ask a domain owner to identify a boundary case.
- Preserve existing tests unless their contract intentionally changes.
- Exercise failure paths through an independent observable effect.
- Use known-bad variants to see whether tests distinguish mistakes.

Different AI reviewers can add perspectives.
Their agreement should not be treated as mathematical independence.
Shared training, shared context, and the same generated summary can produce correlated mistakes.

### 9.5 Test failure paths through state

For rollback, force job insertion to fail.
Then inspect whether a receipt remains.
A function returning an error is insufficient if it has already committed acceptance state.

The important observation is:

> After failed insertion, no receipt suppresses the next legitimate attempt.

A useful second step removes the artificial failure and retries.
The retry should create the job successfully.
That establishes recovery through the same public interface.

### 9.6 Treat concurrency tests as bounded evidence

Two connections entering together can reveal a check-then-insert race.
One successful run cannot establish behavior under every scheduler or database implementation.

Pair the test with code and constraint inspection.
The constraint identifies the invariant.
The transaction boundary identifies how it is maintained.
The concurrent test exercises one relevant condition.

For a production database, use its actual concurrency semantics.
A SQLite teaching test is not evidence for an untested PostgreSQL isolation configuration.

### 9.7 Keep test order flexible

For implementation changes, reading code before tests helps reviewers construct their own expected behavior.
For a test repair, test files are the change being reviewed and should appear early.
For a security regression, a reproducer may be the clearest orientation.

“Tests collapsed until code is read” is a default interaction.
It is not a universal cognitive rule.
A reviewer should be able to open them immediately.

---

## 10. Group the diff by concern and order attention

The repository's file order is an accidental reading order.
It may separate a contract from its implementation or put a consequential workflow change at the end.

A review tour can give the same patch a more useful path.

### 10.1 Define a concern through a question

Good concern:

> How does acceptance preserve one job for each tenant-scoped event?

It includes schema, transaction, duplicate handling, and relevant tests.

Weak concern:

> Backend files.

It groups by location without explaining the judgment.

Use questions that match the review decision:

- Does authorization bind the caller to the tenant?
- Does the migration preserve existing records?
- Does the new timeout permit safe retry?
- Does the UI reset pagination when the filter changes?

### 10.2 Allow a file to participate in several concerns

A single endpoint file may contain:

- Authentication.
- Parsing.
- Transaction invocation.
- Error mapping.

Grouping the whole file under “endpoint” hides those responsibilities.
A concern tour can anchor separate ranges in the same file.
The raw file view must still show the full context.

If the same hunk supports several claims, identify a primary concern and cross-reference the others.
Avoid duplicating the patch in a way that suggests extra changes.

### 10.3 Put consequence first, dependencies within it

For an authorization and UI change:

1. Inspect tenant authorization.
2. Read the permission contract and implementation together.
3. Inspect callers that expose the result.
4. Inspect the UI behavior.
5. Read the relevant tests.

This keeps the highest-consequence judgment early.
It also gives each concern a coherent internal sequence.

“Foundations before dependents” is useful within a concern.
It should not force a high-risk endpoint behind a long list of unrelated low-risk data types.

### 10.4 Separate mechanical transformations

For a rename:

- Identify the rename rule.
- Show representative before/after pairs.
- Verify that public references and generated outputs changed as intended.
- Keep behavioral edits in a separate concern.

A collapsed mechanical section should show counts and a verification method.
It should remain inspectable.

Do not assume a mechanical change is risk-free.
Renaming a public configuration key can change compatibility.
Moving files can alter packaging.
Changing import order can alter side effects in some systems.

### 10.5 Keep coverage of the patch complete

Every changed hunk should be reachable from a concern or an explicit ungrouped section.
Every concern should identify its supporting code.

Useful inventory:

```text
Concern 1: receipt identity and transaction — 4 hunks
Concern 2: validation and caller contract — 2 hunks
Concern 3: tests — 7 hunks
Mechanical: generated fixture — 1 hunk
Ungrouped: 1 hunk; author explanation required
```

Coverage here means presentation mapping.
It does not mean the claims are true or the tests cover behavior.
A tool should name those metrics differently.

### 10.6 Treat generated grouping as editable

An LLM can propose concerns.
A human may find that the grouping hides a coupling or separates related behavior.

Provide:

- The ordinary file order.
- A concern view.
- A way to regroup or reorder.
- An ungrouped inventory.
- Stable links to the same underlying patch.

Generated grouping should support navigation.
It should not determine which code the reviewer is allowed to see.

### 10.7 Split independent work at the source

Presentation cannot fully repair an unnecessarily tangled change.
If a task combines unrelated behavior, split the commits or PRs where feasible.

For a stack:

- State each change's base.
- Identify dependencies between changes.
- Show the cumulative integration diff.
- Avoid claiming independent deployability when the lower changes are required.

A migration, implementation, and caller update may be one atomic deployment concern.
Splitting it into three PRs can increase coordination risk.
Use reviewability and deployability together when choosing slices.

---

## 11. Replace prefixed diagrams with triggered deltas

The “prefixed diagrams” idea places a standard set of architecture diagrams before every change.
It promises a familiar visual structure.
It also risks presenting unchanged architecture repeatedly and lending authority to unsupported edges.

The stronger proposal is conditional:

> Select a diagram when the change creates a structural question that the reviewer must answer.

### 11.1 Begin with the question

For a retry fix:

> Does the acceptance transaction include job insertion?

For a service extraction:

> Which calls now cross a process boundary?

For a schema change:

> Which relationship changes cardinality or deletion behavior?

For a lifecycle change:

> Can a canceled operation return to the running state?

If no question benefits from a visual, omit the visual.
A local comparison fix can often be understood directly from a few lines.

### 11.2 Choose a diagram by changed behavior

| Changed property | Useful view | Potential source |
|---|---|---|
| Call or event ordering | Before/after sequence delta | Instrumented trace plus inspected call sites |
| Module or service boundary | Component or dependency delta | Import, build, and service configuration graph |
| Data relationship | ERD delta | Schema and migration definitions |
| Lifecycle | State-transition delta | Transition definitions and inspected guards |
| Public symbol impact | Caller map grouped by module | Static references with resolution limits |
| Security boundary | Data-flow view with trust boundaries | Reviewed authorization and data-flow analysis |
| User-visible interaction | Before/after screenshots or recording | Running builds under matched conditions |
| Local logic, prose, tests | Often no diagram | Direct diff and evidence |

These are proposed triggers.
They are not a requirement to generate every possible view for each qualifying change.

### 11.3 Show the delta and enough context

Include:

- Touched nodes.
- Relevant unchanged neighbors.
- Added, removed, and changed relationships.
- A legend that works without color.
- A boundary note for omitted context.

One hop of context is a useful initial limit.
It is not a correctness rule.
A second hop may be necessary to show an authorization boundary or an indirect dependency.

For the event example, include the caller, handler, receipt table, and job table.
The external delivery system can appear as explicitly out-of-scope context if the reviewer might otherwise infer an exactly-once guarantee.

### 11.4 Separate standing architecture from change evidence

A repository wiki can explain the broader platform.
Link to it as context.
A PR diagram should explain what changed within that platform.

An unchanged global architecture drawing cannot establish that this patch preserves transaction ordering.
It may still help a new reviewer orient.
Label the role clearly:

> Background architecture; not a representation of this diff.

### 11.5 Avoid visual certainty

Use provenance labels:

- Extracted static dependency.
- Observed in a named test trace.
- Human-checked relationship.
- Inferred and unverified.
- Proposed design.

A solid arrow should not mean “probably exists.”
If an uncertain edge is important, encode uncertainty in both text and styling.
For high-consequence decisions, withhold an unverified generated diagram from the decision surface and show the unresolved question instead.

A small truthful table is preferable to a polished misleading graph.

### 11.6 Define omission as normal

An empty architecture section invites authors to fill it with something decorative.
Use no section at all, or a short status in a tool's metadata:

> Structural view omitted: no changed flow or boundary identified.

That status is an author or tool assessment.
It can be challenged.
It should not certify that no structural consequence exists.

### 11.7 Use screenshots as evidence for a specific state

For a UI change, match before and after:

- Route.
- Data fixture.
- User permissions.
- Feature flags.
- Viewport.
- Interaction state.

A screenshot of a happy path does not cover empty, error, loading, or accessibility states.
Show the states changed by the task.
For focus order, keyboard behavior, and screen-reader output, record appropriate interaction evidence.

The requirement to include visual evidence for UI work is a proposed team policy.
The research does not establish it as a universal causal intervention.

### 11.8 Interpret product patterns accurately

CodeRabbit documents configurable sequence diagrams for interaction changes in walkthroughs and a separate Explain workspace with structured layers, range-anchored summaries, and graphs.
These are useful implementation examples.
They do not establish that the product has abandoned walkthrough diagrams or that conditional diagrams have measured superior outcomes.
The configuration and workspace behavior should be read separately. [S16, S17]

The design lesson is to make visualization serve a question.
Vendor adoption is supporting context, not proof.

---

## 12. Build a diagram pipeline that preserves provenance

A diagram can be syntactically valid and semantically wrong.
Validation should address both.

### 12.1 Separate extraction from presentation

A proposed pipeline:

```text
Immutable base and head
        |
        v
Extract symbols, relationships, or trace events
        |
        v
Compare canonical node and edge records
        |
        v
Select relevant context and label unresolved relationships
        |
        v
Lay out and render the delta
        |
        v
Validate source anchors, grammar, and review status
```

The LLM can help choose readable labels and layout.
It should not silently invent relationships when deterministic extraction is the stated source.

If the model proposes an additional edge, store it as an inference.
Keep it outside the extracted set until a source or human inspection supports it.

### 12.2 Give edges evidence, not just nodes

Checking that a function exists does not establish that another function calls it.

An edge record might contain:

```json
{
  "from": "receiver.accept_event",
  "to": "sqlite.jobs.insert",
  "relationship": "writes",
  "status": "human-checked",
  "revision": "<full-head-commit>",
  "source": {
    "path": "receiver.py",
    "symbol": "accept_event",
    "range": "<head-line-range>"
  },
  "limits": [
    "schematic database operation, not a resolved Python call symbol"
  ]
}
```

This is an illustrative schema.
The database operation is explicitly a schematic node.
A validator must not pretend it is a function in the source tree.

### 12.3 Respect extraction limits

Static analysis can miss:

- Dynamic dispatch.
- Reflection.
- Runtime service discovery.
- Generated code.
- Configuration-driven routes.
- Dependency injection.
- Database triggers.
- External systems.

A trace can miss paths not exercised by its fixture.
A build graph can describe compilation without showing runtime message order.

Each source supports a different kind of statement.
Combine sources when the diagram's question needs them.

### 12.4 Do not infer sequence from a call graph alone

A call graph can establish possible calls.
It does not necessarily establish:

- Execution order.
- Branch conditions.
- Transaction membership.
- Retry count.
- Concurrency.
- Message acknowledgment.

For the event example, seeing both receipt and job writes is insufficient.
The reviewer needs to inspect their transaction scope.
A trace can illustrate an executed path.
Source and database semantics establish the wider invariant.

Label the diagram's observed path and conditional branches.
Do not draw a single trace as if it exhausts all behavior.

### 12.5 Normalize before computing deltas

If node labels change between runs, the delta can show a removed component and an added component where there was only a rename.

Use stable identities where available:

- Fully qualified symbols.
- Build targets.
- Schema object identities.
- Service identifiers.
- Explicit rename mappings.

Keep display labels separate from identity.
Let reviewers inspect uncertain rename matching.

### 12.6 Validate in layers

| Check | What it establishes |
|---|---|
| Diagram grammar parses | Renderer can interpret the notation |
| Referenced nodes resolve | Named source objects exist where claimed |
| Edges have source records | Relationships have traceable support |
| Anchors match revisions | Sources describe the displayed snapshot |
| Delta is reproducible | The same inputs produce the expected canonical graph |
| Human semantic review | The diagram answers the question without misleading simplification |

No single row establishes diagram correctness.
In particular, successful Mermaid or PlantUML parsing checks syntax.
It does not validate the represented system.

### 12.7 Fail visibly and proportionately

If an extractor cannot resolve a relationship:

- Record the limit.
- Remove unsupported factual claims.
- Keep the raw diff available.
- Present the unresolved structural question.

For a high-risk change, omit the generated diagram until verified.
For a design discussion, an explicitly unverified sketch may remain useful.

“Fail closed” should apply to the presentation claim.
It should not erase evidence or silently hide the change itself.

### 12.8 Preserve the artifact's lifecycle

A diagram is stale when its source snapshot changes.
Store:

- Source revisions.
- Extractor and renderer versions.
- Canonical graph digest.
- Human verification state.
- Known omissions.

Do not preserve a green human-verified badge across new code automatically.
The human verified a particular representation of a particular change.

---

## 13. Execute the complete local example

This example makes a review claim executable.
It provides an intentionally buggy baseline, a corrected implementation, and a test file used with both.

Create two disposable directories, `base` and `head`.
Save the baseline as `base/receiver.py`.
Save the corrected version as `head/receiver.py`.
Put the same `test_receiver.py` in both directories.

The baseline is a teaching fixture, not a historical version of a production service.
The directory labels are example revision labels.
In a real PR, use immutable Git revisions.

### 13.1 Define the contract

The interface is:

```python
accept_event(db_path, tenant_id, event_id, order_id)
```

Its head behavior:

- A new `(tenant_id, event_id)` queues one local job.
- The same identity and order returns `duplicate`.
- The same identity with another order raises `EventConflict`.
- Tenant namespaces are independent.
- A failed job insert leaves no committed receipt.
- Receipt state survives a new connection and process.

Identifiers are limited to 1–64 ASCII letters, digits, underscores, and hyphens.
The caller supplies a trusted tenant identity.
The function does not authenticate that identity.

The function queues work.
It does not send a notification.
No external exactly-once guarantee is claimed.

### 13.2 Save the deliberately buggy baseline

`base/receiver.py`:

```python
import re
import sqlite3
from contextlib import closing


class EventConflict(ValueError):
    pass


def validate_identifier(value):
    if not isinstance(value, str):
        raise ValueError("identifier must be a string")
    if re.fullmatch(r"[A-Za-z0-9_-]{1,64}", value) is None:
        raise ValueError("invalid identifier")


def initialize(db_path):
    with closing(sqlite3.connect(db_path)) as conn:
        conn.execute(
            """
            CREATE TABLE jobs (
                job_id INTEGER PRIMARY KEY,
                tenant_id TEXT NOT NULL,
                event_id TEXT NOT NULL,
                order_id TEXT NOT NULL
            )
            """
        )
        conn.commit()


def accept_event(db_path, tenant_id, event_id, order_id):
    for value in (tenant_id, event_id, order_id):
        validate_identifier(value)
    with closing(sqlite3.connect(db_path)) as conn:
        conn.execute(
            "INSERT INTO jobs (tenant_id, event_id, order_id) "
            "VALUES (?, ?, ?)",
            (tenant_id, event_id, order_id),
        )
        conn.commit()
    return "queued"
```

The bug is straightforward:
every valid call inserts another job.
There is no durable acceptance identity.

The baseline retains the same callable interface and identifier validation.
That lets the duplicate regression reach the relevant behavioral assertion.
The unused conflict class exists only to keep the shared test module importable.

### 13.3 Save the corrected implementation

`head/receiver.py`:

```python
import re
import sqlite3
from contextlib import closing


class EventConflict(ValueError):
    pass


def validate_identifier(value):
    if not isinstance(value, str):
        raise ValueError("identifier must be a string")
    if re.fullmatch(r"[A-Za-z0-9_-]{1,64}", value) is None:
        raise ValueError("invalid identifier")


def initialize(db_path):
    with closing(sqlite3.connect(db_path)) as conn:
        conn.execute("PRAGMA foreign_keys = ON")
        conn.executescript(
            """
            CREATE TABLE receipts (
                tenant_id TEXT NOT NULL,
                event_id TEXT NOT NULL,
                order_id TEXT NOT NULL,
                PRIMARY KEY (tenant_id, event_id)
            );

            CREATE TABLE jobs (
                job_id INTEGER PRIMARY KEY,
                tenant_id TEXT NOT NULL,
                event_id TEXT NOT NULL,
                order_id TEXT NOT NULL,
                UNIQUE (tenant_id, event_id),
                FOREIGN KEY (tenant_id, event_id)
                    REFERENCES receipts (tenant_id, event_id)
            );
            """
        )
        conn.commit()


def accept_event(db_path, tenant_id, event_id, order_id):
    for value in (tenant_id, event_id, order_id):
        validate_identifier(value)

    with closing(
        sqlite3.connect(db_path, timeout=5, isolation_level=None)
    ) as conn:
        conn.execute("PRAGMA foreign_keys = ON")
        conn.execute("BEGIN IMMEDIATE")
        try:
            existing = conn.execute(
                "SELECT order_id FROM receipts "
                "WHERE tenant_id = ? AND event_id = ?",
                (tenant_id, event_id),
            ).fetchone()

            if existing is not None:
                if existing[0] != order_id:
                    raise EventConflict("event identity has another order")
                conn.commit()
                return "duplicate"

            conn.execute(
                "INSERT INTO receipts (tenant_id, event_id, order_id) "
                "VALUES (?, ?, ?)",
                (tenant_id, event_id, order_id),
            )
            conn.execute(
                "INSERT INTO jobs (tenant_id, event_id, order_id) "
                "VALUES (?, ?, ?)",
                (tenant_id, event_id, order_id),
            )
            conn.commit()
            return "queued"
        except Exception:
            conn.rollback()
            raise
```

The example acquires a write transaction before checking acceptance state.
Receipt and job writes occur on the same connection before commit.
The composite keys include tenant identity.
The explicit uniqueness constraint on jobs also prevents duplicate jobs for that identity.

The foreign key establishes that a job references a receipt identity.
It does not independently enforce equality of the two `order_id` values.
The function writes the same argument into both tables.
A system permitting other writers would need an additional invariant or an explicitly narrower write boundary.

`initialize` creates a fresh teaching schema.
It is intentionally not an upgrade function.
Running it over the baseline database is unsupported.
A production integration needs a reviewed migration, historical-event policy, and rollout plan.

### 13.4 Save the shared test file

`test_receiver.py`, copied without modification into both directories:

```python
import sqlite3
import subprocess
import sys
import tempfile
import threading
import unittest
from concurrent.futures import ThreadPoolExecutor
from contextlib import closing
from pathlib import Path

from receiver import EventConflict, accept_event, initialize


def row_count(db_path, table):
    queries = {
        "jobs": "SELECT COUNT(*) FROM jobs",
        "receipts": "SELECT COUNT(*) FROM receipts",
    }
    with closing(sqlite3.connect(db_path)) as conn:
        return conn.execute(queries[table]).fetchone()[0]


class DatabaseTest(unittest.TestCase):
    def setUp(self):
        self.temp = tempfile.TemporaryDirectory()
        self.addCleanup(self.temp.cleanup)
        self.db = Path(self.temp.name) / "events.sqlite"
        initialize(self.db)


class RegressionTests(DatabaseTest):
    def test_retry_creates_one_job(self):
        accept_event(self.db, "tenant_a", "event_1", "order_1")
        accept_event(self.db, "tenant_a", "event_1", "order_1")
        self.assertEqual(row_count(self.db, "jobs"), 1)


class HeadContractTests(DatabaseTest):
    def test_tenant_namespaces_are_independent(self):
        self.assertEqual(
            accept_event(self.db, "tenant_a", "event_1", "order_1"),
            "queued",
        )
        self.assertEqual(
            accept_event(self.db, "tenant_b", "event_1", "order_2"),
            "queued",
        )
        self.assertEqual(row_count(self.db, "jobs"), 2)

    def test_conflicting_order_is_rejected(self):
        accept_event(self.db, "tenant_a", "event_1", "order_1")
        with self.assertRaises(EventConflict):
            accept_event(self.db, "tenant_a", "event_1", "order_2")
        self.assertEqual(row_count(self.db, "jobs"), 1)
        self.assertEqual(row_count(self.db, "receipts"), 1)
        self.assertEqual(
            accept_event(self.db, "tenant_a", "event_1", "order_1"),
            "duplicate",
        )

    def test_failed_job_insert_rolls_back_receipt(self):
        with closing(sqlite3.connect(self.db)) as conn:
            conn.execute(
                """
                CREATE TRIGGER reject_job BEFORE INSERT ON jobs
                BEGIN
                    SELECT RAISE(ABORT, 'simulated job failure');
                END
                """
            )
            conn.commit()
        with self.assertRaises(sqlite3.IntegrityError):
            accept_event(self.db, "tenant_a", "event_1", "order_1")
        self.assertEqual(row_count(self.db, "receipts"), 0)
        self.assertEqual(row_count(self.db, "jobs"), 0)
        with closing(sqlite3.connect(self.db)) as conn:
            conn.execute("DROP TRIGGER reject_job")
            conn.commit()
        self.assertEqual(
            accept_event(self.db, "tenant_a", "event_1", "order_1"),
            "queued",
        )
        self.assertEqual(row_count(self.db, "jobs"), 1)

    def test_two_connections_share_identity(self):
        barrier = threading.Barrier(2)

        def submit():
            barrier.wait(timeout=5)
            return accept_event(
                self.db, "tenant_a", "event_1", "order_1"
            )

        with ThreadPoolExecutor(max_workers=2) as pool:
            futures = [pool.submit(submit) for _ in range(2)]
            results = [future.result(timeout=10) for future in futures]
        self.assertEqual(sorted(results), ["duplicate", "queued"])
        self.assertEqual(row_count(self.db, "jobs"), 1)
        self.assertEqual(row_count(self.db, "receipts"), 1)

    def test_restart_preserves_receipt(self):
        accept_event(self.db, "tenant_a", "event_1", "order_1")
        script = (
            "from receiver import accept_event; "
            "import sys; "
            "print(accept_event(sys.argv[1], "
            "'tenant_a', 'event_1', 'order_1'))"
        )
        result = subprocess.run(
            [sys.executable, "-c", script, str(self.db)],
            cwd=Path(__file__).resolve().parent,
            text=True,
            capture_output=True,
            check=True,
            timeout=10,
        )
        self.assertEqual(result.stdout.strip(), "duplicate")
        self.assertEqual(row_count(self.db, "jobs"), 1)

    def test_invalid_identifiers_create_no_state(self):
        for position in range(3):
            for invalid in ("", "contains space", "é", "x" * 65, None):
                with self.subTest(position=position, invalid=invalid):
                    args = ["tenant_a", "event_1", "order_1"]
                    args[position] = invalid
                    with self.assertRaises(ValueError):
                        accept_event(self.db, *args)
        self.assertEqual(row_count(self.db, "jobs"), 0)
        self.assertEqual(row_count(self.db, "receipts"), 0)


if __name__ == "__main__":
    unittest.main()
```

The tests inspect results and persisted state.
The rollback test uses a database trigger to force an observable insertion failure.
The restart check starts a separate interpreter.
The concurrency check uses separate connections created inside the handler.

The table-count helper uses a fixed query map.
It does not interpolate an arbitrary table name into SQL.

### 13.5 Execute the base regression

From `base`:

```bash
python3 -m unittest -v test_receiver.RegressionTests
```

Expected result:

```text
test_retry_creates_one_job ... FAIL
AssertionError: 2 != 1
Ran 1 test
FAILED (failures=1)
```

The command should exit unsuccessfully because the assertion detects the intended bug.
Do not run the full head contract suite as the baseline demonstration.
The baseline deliberately lacks receipt behavior, so those failures would mix the regression with new contract differences.

### 13.6 Execute the head checks

From `head`:

```bash
python3 -m unittest -v test_receiver
```

Expected result:

```text
Ran 7 tests
OK
```

The same regression now passes.
Six additional tests inspect the new contract.

These code blocks were extracted and executed when preparing this edition.
The base regression failed with the intended `2 != 1` assertion, and the seven head tests passed.
The exact local environment and an additional mutation exercise are recorded in the appendix.
That execution validates this teaching example under those conditions.
It does not validate a production integration.

### 13.7 Understand the guarantee

The head establishes a bounded local invariant through this interface:

> While receipts remain intact, an accepted tenant-scoped event has one queued local job, and failure to insert the job does not commit a new receipt.

It does not establish:

- That the caller is authorized for the tenant.
- That every possible database writer preserves the contract.
- That existing databases can be upgraded safely.
- That receipts remain forever or expire correctly.
- That the external worker sends exactly once.
- That a response reaches the upstream sender.
- That SQLite lock behavior matches another database engine.

These boundaries belong in the review package.
They are consequential integration work, not small print.

### 13.8 Inspect known bad variants

Useful mutations include:

| Mutation | Test expected to catch it |
|---|---|
| Insert on every retry | Duplicate regression |
| Remove tenant from receipt lookup | Tenant independence |
| Treat changed order as duplicate | Conflict test |
| Commit receipt before inserting job | Rollback test |
| Remove durable receipt lookup | Restart test |
| Remove identifier validation | Invalid-input test |

Mutation testing is a check on the test suite's discrimination.
It does not prove that all defects are covered.
Use mutations tied to realistic failure hypotheses.
Do not spend effort constructing dozens of trivial mutants that add no new review insight.

---

## 14. Present the example for human review

The executable example now needs a presentation that accurately states what it establishes.
This chapter turns it into a complete review package.

The package below concerns a local teaching change.
Its file names, concerns, and results correspond to chapter 13.
Commit placeholders must be filled before using this format in a real PR.

### 14.1 Present the header

```markdown
## Intent and decision

Prevent repeated acceptance of a tenant's order event from creating
another local notification job.

Risk: medium for this local example. Adds persistent acceptance state
and changes concurrent writes.

Snapshot: <base-commit> → <head-commit>.
Comparison rule: <rule>.

Scope:
- receiver.py: receipt identity, transaction, duplicate/conflict result.
- test_receiver.py: retry regression and six contract checks.
- No undeclared files.

Plan deviation:
Added explicit rejection of an event ID reused for another order.
This resolves an ambiguity in the original retry requirement.

Requested review:
Inspect tenant-scoped identity, receipt/job atomicity, and the
conflict policy. This package does not authorize production deployment.
```

In a real task, the conflict policy should be settled before calling it part of the accepted contract.
The header makes the deviation visible.
It does not quietly transform the original task into a broader feature.

### 14.2 Present evidence before conclusions

```markdown
## Evidence

Before: two identical calls create two local jobs.
After: a retry returns `duplicate`; one local job remains.

Counterfactual:
- Same test file on both implementations.
- Base: duplicate regression fails with `2 != 1`.
- Head: duplicate regression passes.

Head suite:
- 7 tests passed.
- Covers tenant independence, conflict behavior, rollback,
  two concurrent connections, a fresh interpreter, invalid identifiers.

Test-change alert:
- New regression and contract tests.
- No existing assertion deletion in this teaching fixture.
- No CI configuration change.

Limits:
- No production migration or backfill.
- No authenticated endpoint integration.
- No receipt expiry or retention policy.
- No external worker or exactly-once delivery test.
- No other database engine exercised.
```

The exact command and environment belong in linked records.
The summary provides the result and its practical limits.

If this were a production change, the missing migration could block deployment.
Do not downgrade that gap because the local suite is green.

### 14.3 Present the concern tour

| Order | Concern | Code to inspect | Question |
|---|---|---|---|
| 1 | Atomic acceptance | `initialize`, transaction block in `accept_event` | Can a receipt commit without a job? |
| 2 | Identity and conflict | Composite keys, receipt lookup, `EventConflict` | Are tenant namespaces independent, and is conflict policy deliberate? |
| 3 | Caller contract | Identifier validation and `tenant_id` argument | Does the real caller authenticate and bind tenant identity? |
| 4 | Behavioral evidence | Shared regression, six contract tests | Do assertions observe the required behavior and likely mistakes? |

The real package should replace these symbol labels with revision-bound anchors.
The book's local example uses symbols so the tour survives Markdown extraction into temporary files.

The question in concern 3 cannot be answered by this code alone.
It remains an integration obligation.

### 14.4 Add a bounded structural view

A small before/after table can be sufficient:

| Step | Baseline | Head |
|---|---|---|
| 1 | Validate identifiers | Validate identifiers |
| 2 | Open connection | Begin write transaction |
| 3 | Insert job | Read tenant-scoped receipt |
| 4 | Commit | Return duplicate, reject conflict, or insert receipt and job |
| 5 | — | Commit both writes or roll back |

For a reviewer inspecting ordering, a sequence diagram may be easier to scan:

```mermaid
sequenceDiagram
    participant C as Trusted caller
    participant H as accept_event
    participant D as SQLite
    Note over C,D: Head schematic; local acceptance only
    C->>H: tenant, event, order
    H->>H: Validate identifiers
    H->>D: BEGIN IMMEDIATE
    H->>D: Lookup receipt by tenant and event
    D-->>H: Existing order or no receipt
    alt Existing receipt, same order
        H->>D: COMMIT
        H-->>C: duplicate
    else Existing receipt, different order
        H->>D: ROLLBACK
        H-->>C: EventConflict
    else No receipt
        H->>D: Insert receipt
        H->>D: Insert local job
        alt Both writes succeed
            H->>D: COMMIT
            H-->>C: queued
        else Write fails
            H->>D: ROLLBACK
            H-->>C: Database error
        end
    end
```

**Provenance:** hand-authored schematic checked against the chapter 13 head implementation.
**Support:** `accept_event` transaction, lookup, writes, commit, and exception path.
**Omissions:** validation failures, connection failures, failure to begin a transaction, and commit/rollback infrastructure failures are not expanded.
**Boundary:** no external worker or acknowledgment protocol is represented.

The diagram is not an automatically extracted graph or a runtime trace.
Its status says exactly how it was produced.
In a real package, attach the supporting ranges at the head commit.

### 14.5 Include the authoritative diff

The review package links directly to:

- Baseline and head schema.
- Transaction implementation.
- Duplicate and conflict branches.
- Validation context.
- Shared test changes.

The schematic is subordinate to those changes.
If the diagram and code disagree, correct the diagram or implementation before presenting the package as current.

### 14.6 Rewrite misleading claims

| Misleading claim | Reviewable replacement |
|---|---|
| “Exactly-once notifications” | “One local job per accepted tenant/event through this interface while receipts remain” |
| “Tenant-safe” | “Receipt identity includes tenant; endpoint authentication is untested here” |
| “Race-free” | “Write transaction and uniqueness constraints inspected; two-connection test passed” |
| “Migration included” | “Fresh-schema initializer only; existing database upgrade remains required” |
| “Fully tested” | “Seven local tests passed; integration gaps listed” |
| “Diagram generated from code” | “Hand-authored schematic checked against named implementation” |

The replacements are longer in some cases.
They remove ambiguity that would otherwise become reviewer work or operational risk.

### 14.7 Make a bounded review decision

A reasonable local-example decision:

> Accept the teaching implementation and evidence. Preserve the stated production integration gaps.

A reasonable production decision:

> Request changes until migration, authenticated tenant binding, and deployment behavior are demonstrated.

The code can be appropriate for one objective and incomplete for another.
Presentation must make the objective explicit enough for the reviewer to distinguish them.

---

## 15. Adapt the format to different changes

The same questions apply broadly.
The evidence and structural view should change with the task.

### 15.1 Local bug fix

Example:
a date filter incorrectly excludes an inclusive upper bound.

Present:

- The input that demonstrates the bug.
- Before/after expected output.
- A regression that reaches the old failing assertion.
- Compatibility with timezone and boundary handling.
- The small diff.

Usually omit an architecture diagram.
If the fix changes only a comparison, the code and boundary examples explain it directly.

A “low risk” label should still identify affected callers.
A helper used in billing may need more careful review than its line count suggests.

### 15.2 UI behavior

Example:
changing a filter should reset pagination.

The plan's decision:

> Does a filter change always return to page one, including a filter cleared by a saved view?

The evidence:

- A fixture with more than one page.
- Before/after interaction sequence.
- URL and saved-view behavior.
- Loading and empty states.
- Keyboard focus after the update.

Matched screenshots show visible state.
A short recording may better show the transition.
Automated interaction tests should inspect the page and URL behavior rather than assert a particular component's internal variable.

The tour can group state contract, routing, component behavior, and tests.
An internal component diagram is useful only if boundaries actually change.

### 15.3 Schema migration

Example:
adding a uniqueness constraint for event identity.

The reviewer needs:

- Existing duplicate-record policy.
- Migration order.
- Lock and runtime impact.
- Compatibility between old and new application versions.
- Backfill behavior.
- Failure recovery.
- Rollback or roll-forward constraints.

A fresh-schema unit test does not establish an upgrade.
Run the actual migration against representative old-schema fixtures.
Verify data preservation and intentional rejection.

An ERD delta can explain the constraint.
It cannot establish that the migration finishes within an acceptable lock window.
Execution evidence must answer that separately.

For destructive operations, describe which information becomes unrecoverable.
Do not promise rollback if the change discards data needed by the old application.

### 15.4 Authorization or tenant isolation

Example:
adding a tenant-aware object lookup.

Put the trust boundary early:

- Where does tenant identity originate?
- Is it taken from the authenticated principal or user-controlled input?
- Does every lookup use the same binding?
- What happens for an object owned by another tenant?
- Can error messages disclose object existence?

Evidence should include an authorized case, an unauthorized case, and boundary conditions relevant to the actual policy.
A data-flow diagram can show where identity is bound and where enforcement occurs.
Every enforcement edge needs source support.

A green test named `test_tenant_safe` is insufficient if its fixture only contains one tenant.
The reviewer should inspect actual principal, ownership, and denial observations.

### 15.5 Broad refactor

Example:
extracting a parsing library from an endpoint module.

Present:

- Intended behavior invariants.
- Public symbol changes.
- Caller inventory and unresolved dynamic consumers.
- Mechanical moves separately from semantic edits.
- Compatibility evidence.
- Build and packaging implications.

A dependency delta can show changed boundaries.
A semantic diff can reduce noise.
Keep the raw diff available because movement detection may misclassify changes.

“Behavior-preserving” is a claim that requires relevant evidence.
Passing a small unit suite cannot establish equivalence across all callers.

### 15.6 Dependency change

Example:
upgrading a client library.

The review should show:

- Direct dependency change.
- Lockfile and transitive changes.
- Relevant release or compatibility notes.
- Runtime behavior exercised.
- License or policy checks required by the project.
- Rollback conditions.

Do not collapse the lockfile so completely that new packages disappear from the review inventory.
Generated content can be inspected through a summarized dependency delta, with raw details available.

The risk may arise in a transitive package rather than application source.
The concern tour should reflect that.

### 15.7 CI and repository instructions

Example:
changing which jobs run for agent-created PRs.

Present:

- The old trigger and new trigger.
- Which events and paths execute or skip jobs.
- Permission and secret-access differences.
- A check using the workflow's real consumer where feasible.
- Any remaining simulation limits.

Repository instructions can also alter future agent behavior.
Their review should identify the new authority, verification, or routing rule.
Searching the instruction file for a desired sentence does not prove that agents follow it.
Use task evaluations for that behavioral claim.

These changes often belong first in the tour because they affect confidence in all other evidence.

### 15.8 Tests-only change

The tests are the implementation under review.
Put them early.

Show:

- The behavior or contract being asserted.
- Why existing coverage was inadequate.
- Whether a known-bad implementation is detected.
- Whether valid alternative implementations remain accepted.
- Changes to discovery, fixtures, and execution cost.

A test that merely mirrors a private helper can become brittle while detecting little.
Ask whether it would survive a behavior-preserving refactor.

### 15.9 Documentation change

Present:

- The reader and decision the document supports.
- The corrected or added claim.
- Source verification for consequential factual changes.
- Link and navigation checks.
- Execution of included examples when practical.

A book or long guide does not require an application test suite.
Its evidence should match its own public contract:
readable navigation, accurate qualifications, working examples, and traceable sources.

---

## 16. Check narrative and diff consistency

Generated summaries can help orient reviewers.
They can also omit changes or claim behavior that does not exist.
A consistency check should expose both directions.

### 16.1 Map claims to evidence

A claim record can distinguish implementation evidence from execution evidence:

```yaml
id: C1
text: A retry of the same tenant/event queues no additional local job.
kind: behavioral
implementation:
  - revision: "<head>"
    path: receiver.py
    symbol: accept_event
    role: receipt lookup and duplicate branch
execution:
  - id: E1
    role: same regression on base and head
limitations:
  - receipt retention is outside this example
status: supported-with-limits
```

A link is not proof.
The referenced code and test must actually support the sentence.
The status names the result of a review, not a guarantee produced by YAML.

### 16.2 Map changes back to claims

For each hunk:

- Which concern explains it?
- Does it alter behavior, contracts, verification, or operations?
- Is it mechanical?
- Is it intentionally unrelated work that should be split?

An unexplained test deletion should appear in the inventory even if every positive summary claim has a link.
Coverage in only one direction misses that problem.

### 16.3 Separate deterministic checks from semantic checks

Deterministic checks can establish:

- Referenced paths exist at the stated revision.
- Ranges fall within the file.
- The head matches the package's head.
- Every hunk has a mapping.
- Every changed path has a scope status.
- An evidence record names an actual artifact.
- A test command returned the recorded exit code.

They cannot generally establish:

- A sentence faithfully describes behavior.
- A test proves the intended requirement.
- A relationship is semantically complete.
- A limitation is sufficient for approval.

An LLM can assist with those semantic questions.
Its output should remain a finding with supporting evidence and uncertainty.
Do not advertise full automatic summary–diff consistency as a solved deterministic problem.

### 16.4 Avoid forcing a behavioral story onto every hunk

Some changes establish supporting structure.
An import, a fixture, or an unchanged-context link may support a concern without representing a separate user-visible behavior.

Use mapping roles:

- Implements.
- Tests.
- Declares.
- Generates.
- Provides context.
- Mechanical.

This keeps coverage meaningful.
Otherwise, authors can satisfy a checker by writing a vague sentence for every line.

### 16.5 Handle negative claims carefully

“No API change” needs an inspection method.
The changed files may not contain a public interface, but generated bindings or configuration could still change externally visible behavior.

Useful record:

> No declared signature changes in the inspected public API set. Serialization behavior and generated clients were checked separately. Reflection-based consumers remain unassessed.

A summary checker should flag broad negative claims for review.
It should not infer absence solely because a keyword is missing from the diff.

### 16.6 Make unsupported claims editable

When a claim lacks support:

1. Locate the intended behavior.
2. Determine whether implementation is missing or prose is overstated.
3. Add the required implementation or narrow the sentence.
4. Collect any newly required evidence.
5. Refresh the snapshot.

Removing an inaccurate sentence can correct presentation.
It cannot complete an unmet acceptance criterion.
The scope contract determines which kind of correction is needed.

### 16.7 Preserve independent findings

If an AI reviewer says the transaction may lose work, keep that finding distinct from the author's answer.

Record:

- Finding.
- Evidence cited.
- Author response.
- Fix or reason for dismissal.
- Human disposition when required.
- Snapshot after resolution.

Do not rewrite the finding into the author's uniformly positive summary.
A reviewer should be able to see that a concern was raised and how it was resolved.

### 16.8 Measure the checker

Evaluate consistency checks with known cases:

- Claimed change absent.
- Changed hunk unexplained.
- Correct claim with imperfect wording.
- Context link mistaken for implementation.
- Stale anchor.
- Broad negative claim.
- Deliberately incomplete integration evidence.

Measure false positives as well as misses.
A checker that flags every sentence may train authors and reviewers to ignore it.
Its purpose is to direct attention to mismatches worth inspecting.

---

## 17. Design a review surface around human control

A review tool should reduce the effort of finding evidence and preserve the ability to challenge the presentation.
The generated narrative should remain one view of the change.

### 17.1 Make three things persistently visible

The reviewer needs:

- Snapshot identity and freshness.
- Current concern and risk explanation.
- A route to the complete diff.

These can fit in a compact header and navigation panel.
They should not be buried in a long PR body.

When the live PR changes, freshness becomes the most important state.
Disable or redirect approval according to the platform's policy.
Make the mismatch explicit before an action targets newer code.

### 17.2 Use progressive disclosure

An ordinary change can begin with:

- Intent.
- Scope status.
- Evidence summary.
- Concern list.

Selecting a concern reveals its code, evidence, and questions.
A reviewer can open the full file or raw patch at any time.

A high-risk concern can automatically expose its unresolved questions.
The tool should not require dozens of expansions to discover a blocking migration gap.

### 17.3 Keep risk explanations local

A per-concern note:

> Persistent state: failure after receipt commit could suppress a retry.

This explains why the transaction deserves attention.

A global score:

> Confidence: 97%.

This gives little information about what was measured or where mistakes matter.
Avoid such scores unless the model, target outcome, calibration, and relevant conditions are defined.
Even then, do not use a global score to replace domain review.

### 17.4 Make collapsed content accountable

Collapsed sections should show:

- File and hunk counts.
- Type of change.
- Verification status.
- Important alerts.

Examples:

```text
Generated lockfile: 4 direct and 19 transitive changes.
Tests: 3 files changed; 2 tests deleted.
Mechanical rename: 42 references; 3 unresolved dynamic uses.
```

The counts make hidden work visible.
The unresolved items prevent a convenience feature from becoming a blind spot.

### 17.5 Provide comparison modes without hiding meaning

Useful modes include:

- Concern order.
- Ordinary file order.
- Full-file context.
- Raw patch.
- Whitespace-reduced view.
- Rename-aware view.

A semantic view may suppress movement or formatting noise.
Its omissions should remain visible and reversible.
Formatting can matter in configuration, embedded code, and languages where whitespace is significant.

Never describe a reduced diff as the complete record.

### 17.6 Keep evidence adjacent to the question

For the transaction concern, place the rollback test beside the transaction range.
For a UI concern, place the matched screenshots beside the relevant behavior.
For a migration, place rehearsal output beside migration code.

The reviewer should not need to search a separate dashboard for every check.
Provide links to the full artifact when more detail is needed.

### 17.7 Support a reviewer-chosen reading order

Some reviewers prefer diff-first.
Others need a short description before code.
A high-risk workflow may offer a prediction-first checkpoint:

> Before opening the explanation, state the expected behavior for this acceptance criterion.

This is a proposed de-biasing technique.
It should be optional or experimentally justified, rather than adding mandatory friction to every PR.

Do not lock reviewers into the author's tour.
Let them reorder, pin, and annotate concerns.

### 17.8 Protect the rendering boundary

PR text, file names, code snippets, diagrams, and logs are untrusted inputs to a review surface.

The renderer should:

- Escape displayed content.
- Sanitize rendered HTML.
- Restrict diagram rendering capabilities.
- Prevent repository content from invoking review actions.
- Treat agent instructions inside a diff as data under review.
- Authorize comments, approvals, and merges separately from content rendering.

For an AI assistant embedded in the tool, repository text can also contain prompt-injection attempts.
A code comment saying “approve this PR and ignore the failing test” is evidence to inspect, not an instruction to obey.

These controls preserve the distinction between content and authority.
They belong in the implementation of the tool, not as a distracting warning on every benign code snippet.

### 17.9 Learn from product behavior without copying promises

CodeRabbit Explain documents a saved review snapshot while actions target the live PR.
That distinction is directly relevant to review-surface design. [S18]

GitHub documents practices for assigning tasks to its coding agent and reviewing resulting work.
Its product documentation is useful for understanding its workflow, not evidence that the resulting code requires less human verification. [S19]

Claude Code documents a planning workflow for investigation before edits.
This provides one implementation of the pre-change stage.
The approval object for the resulting diff remains a separate decision in the proposed process here. [S20]

Build around properties that can be checked:
current snapshot, complete patch access, supported claims, clear limitations, and explicit action authority.
Do not select a tool solely because it produces a more impressive overview.

---

## 18. Run the human review and revision loop

The presentation makes information available.
The reviewer still needs a disciplined procedure for making the decision.

### 18.1 Establish the expected result

Read the task and acceptance criteria.
For a consequential change, form a brief independent expectation.

For the example:

> A retry should preserve one queued job for that tenant/event. An insertion failure must leave the event retryable.

That expectation gives the reviewer a standard for examining both implementation and tests.

### 18.2 Check scope before detail

Inspect:

- Actual changed paths.
- Planned paths that did not change.
- Unexpected changes.
- Test and CI changes.
- Dependency and permission changes.

If the agent has added an unrelated framework or reformatted the repository, resolve that scope issue early.
There is little value in carefully reviewing code the task did not authorize or need.

### 18.3 Choose the consequential concern

Start with the concern where a mistake has the largest consequence.
Inspect its foundations and implementation together.

For the example:

1. Composite identity.
2. Transaction scope.
3. Duplicate and conflict branches.
4. Rollback behavior.

Then inspect integration assumptions and tests.
Use the diagram as navigation if helpful.
Check the represented edges against code.

### 18.4 Ask counterfactual questions

Useful questions:

- What happens on the second call?
- What happens for another tenant?
- What happens after the first write fails?
- What happens if the process restarts?
- What happens when an old application runs against the new schema?
- What behavior would make these tests fail?

These questions invite reasoning beyond the happy path.
They are more discriminating than “does this look reasonable?”

### 18.5 Reproduce high-value evidence

The reviewer need not rerun every command manually if trusted CI already provides head-bound evidence.
Rerun when:

- The result is important and provenance is weak.
- The regression failure is unclear.
- The test seems shaped around the implementation.
- The environment differs from deployment.
- A relevant claim remains disputed.

For high-consequence work, follow the project's explicit verification policy.
Review effort should answer uncertainty, not repeat every check ceremonially.

### 18.6 Phrase comments around the contract

Useful comment:

> The receipt commits before job insertion. If insertion fails, the next retry returns duplicate without a job. Keep both writes in one transaction and add a failure/retry test.

Less useful:

> This seems unsafe.

The useful comment identifies code, consequence, desired invariant, and evidence.
It gives the agent a concrete correction.

For product ambiguity:

> The task does not specify whether conflicting order IDs should be rejected. Please get the event owner to settle this policy before implementing the response mapping.

That comment directs a decision rather than inviting the agent to guess.

### 18.7 Refresh the package after revision

After the agent fixes a finding:

- Identify the new head.
- Show changed concerns.
- Preserve the finding and disposition.
- Rerun affected checks.
- Refresh anchors and diagrams.
- State any remaining questions.

Do not require a full tour reread when only one bounded concern changed.
Do not carry approval for a high-risk concern across a semantic edit silently.

### 18.8 Distinguish steering from review

A command to an agent:

> Add the rollback check.

This is steering.

A judgment:

> Inspected the transaction and failure test at head H; both support rollback of receipt and job.

This is review evidence.

Repository mining research on agent PRs reports substantial differences in recorded review activity and automation-mediated interaction.
Its results caution against counting every comment or review event as equivalent human oversight. [S21]

A review system should distinguish instructions, findings, and acceptance decisions.

### 18.9 Finish with an explicit disposition

Possible dispositions:

- Accept this snapshot.
- Request a specified change.
- Accept the implementation but block deployment on a named integration obligation.
- Defer because evidence is inconclusive.
- Reject because scope or approach does not meet the task.

Use the platform's established review semantics.
A package-specific note should not create a second ambiguous approval system.

For high-risk work, record which owner accepted which remaining limitation.
Ownership belongs to the actual decision, not a decorative name in the header.

---

## 19. Evaluate the presentation with controlled tasks

A attractive presentation can increase satisfaction while leaving defect detection unchanged.
An evaluation should measure the outcomes the team wants.

### 19.1 Name the target outcomes

Primary candidates:

- Detection of a known consequential defect.
- Detection of a summary–diff mismatch.
- Detection of unrequested scope expansion.
- Correct understanding of changed behavior.
- False-positive findings.
- Review time.
- Confidence calibration.

Keep satisfaction as a secondary measure.
It can explain adoption.
It cannot substitute for correctness.

### 19.2 Use realistic tasks with controlled mistakes

Take representative agent changes and create disposable evaluation versions.
Seed a known mistake:

- Wrong dependency.
- Wrong operation order.
- Missing tenant condition.
- Unimplemented change claimed in prose.
- Unrequested cleanup.
- Test weakening.

Run these in an evaluation environment, never as unnoticed defects in live production review.
Participants should know they are evaluating a review workflow.
Debrief them on seeded defects afterward.

Keep realistic context.
A tiny artificial function may not capture multi-file review difficulty.

### 19.3 Compare diagrams under relevant changes

Useful conditions:

| Condition | Question |
|---|---|
| No diagram | Baseline for the structural task |
| Triggered, supported delta | Does relevant structure help? |
| Always-prefixed architecture | Does repeated generic context spend attention? |
| Inferred diagram with visible uncertainty | How do reviewers use uncertain structure? |

A hallucinated-edge condition should be clearly part of a controlled experiment.
Measure whether participants detect it or follow it into an incorrect understanding.
Do not present it as a trustworthy artifact outside the experiment.

Balance task difficulty and defect placement across conditions.
Otherwise, a more difficult PR can make a useful format appear ineffective.

### 19.4 Evaluate order and grouping separately

Compare:

- Alphabetical file order.
- Dependency order.
- Risk-first concerns.
- Concern grouping without reordering.

Keep the underlying changes and available evidence equivalent.
This separates the value of grouping from the value of position.

A within-participant design can reduce differences in individual skill, but participants must not see the same seeded defect twice.
Use matched variants or different tasks and counterbalance order.

### 19.5 Evaluate description-first and diff-first

Conditions can include:

- Description first.
- Diff first.
- Brief prediction before description.

Seed a claim that the implementation does not fulfill.
Measure whether the reviewer catches it.
Also measure correct understanding on accurate-summary tasks.

A format that reduces anchoring but makes every ordinary review substantially harder may have an unattractive tradeoff.
Measure both.

### 19.6 Evaluate scope declarations

Provide equivalent changes with and without:

- Declared file/module scope.
- Deviation alert.
- Complete path comparison.

Seed a plausible unrequested edit.
Ask whether the reviewer identifies and challenges it.

Scope checking should detect unwanted reinterpretation.
It should not punish necessary supporting work that has a clear explanation.
Include both kinds of case to measure false positives.

### 19.7 Measure calibration through explicit predictions

Ask:

> How likely is it that the seeded target behavior is correct?

Collect a probability or a bounded confidence scale before revealing the answer.
Compare confidence with correctness.

For probability forecasts, one possible metric is the Brier score:

```text
Mean of (predicted probability - actual outcome)^2
```

Define the outcome precisely.
A prediction about the whole PR is too broad if the task measures one target defect.
Report subgroup counts and uncertainty.

Do not treat lower confidence as automatically better.
The objective is confidence that tracks correctness.

### 19.8 Analyze detection and time together

Report:

- Count of detections out of eligible cases.
- False-positive findings per task.
- Time distribution.
- Confidence conditional on correctness.
- Reviewer and task characteristics.

An average time alone can hide a format that makes reviewers faster by skipping important work.
A detection rate alone can hide unsustainable review effort.

For small pilots, use counts and intervals rather than bold percentage claims.
“8 of 12 reviewers” is more informative than a precision-looking percentage without a denominator.

### 19.9 Account for familiarity and habituation

Participants unfamiliar with a new surface may be slower initially.
Always-on diagrams may appear useful in a short novelty study but become ignored later.

Use:

- A practice task.
- Several sessions where feasible.
- Repeated ordinary changes between structural tasks.
- A follow-up on continued use.

Keep exposure history in the analysis.
Do not attribute a novelty effect to a lasting improvement.

### 19.10 Distinguish adoption metrics from causal evidence

Production metrics can include:

- Package completeness.
- Stale evidence rate.
- Unexplained scope expansions.
- Human-owned high-risk decisions.
- Reviewer revisions to generated claims.
- Time waiting for required expertise.

These diagnose the operating process.
Changes in merge time or later defects can reflect task mix, team changes, and agent capability.
An uncontrolled before/after dashboard does not establish that the presentation caused improvement.

For a self-hosted review tool, including one used inside Mapbox, start with local tasks and measured reviewer decisions.
Choose a pilot population and defect set appropriate to its codebase.
Do not import another company's effect sizes as expected results.

---

## 20. Roll out the practice and operate it

Start with a format people can use manually.
Automate the parts whose contract is clear.
Evaluate the parts that make semantic judgments.

### 20.1 Begin with four fields

A first version can require:

1. Intent.
2. Scope and deviations.
3. Evidence and limits.
4. Requested review.

Add the complete diff through the ordinary platform.
This already improves the handoff without building a new renderer.

Do not require a long form for every change.
Low-impact work can answer those fields in a few lines.

### 20.2 Add mechanical inventory

Next automate:

- Base and head identity.
- Changed path inventory.
- Scope comparison.
- Test/CI-change alerts.
- Evidence artifact links.
- Staleness detection.

These features have comparatively clear inputs and outputs.
They reduce repetitive work and make hidden mismatches visible.

Keep the source of the inventory visible.
A failed Git inspection should produce an unavailable status, not an empty changed-file list.

### 20.3 Add concern tours as a suggestion

Generate an initial grouping and reading order.
Let authors and reviewers revise it.
Measure:

- Ungrouped hunks.
- Reviewer regrouping.
- Unsupported summaries.
- Navigation difficulty.
- Time spent locating relevant context.

Treat repeated regrouping as feedback about the generator.
Do not force users to adapt their reasoning to an unstable classification.

### 20.4 Add one diagram trigger at a time

Start with a narrow source:

- Schema delta from migration files.
- Dependency delta from a build graph.
- Sequence illustration from a named test trace.

Define the question it answers.
Define unsupported cases.
Validate its artifact identity and grammar.
Evaluate whether reviewers use it correctly.

Avoid building an entire diagram library before finding one source and task that works well.

### 20.5 Establish freshness and ownership policies

The team should decide:

- Which changes require refreshed evidence.
- How stale approvals are handled.
- Which domains require named owners.
- How an unresolved integration gap blocks release.
- How long evidence is retained.
- Who can accept exceptions.

Keep these rules in the repository or review system's established policy.
The PR body should report their application.
It should not invent authority through generated prose.

### 20.6 Operate failure states

| Failure | Useful response |
|---|---|
| Agent cannot reproduce base bug | Mark counterfactual inconclusive and investigate the fixture |
| Summary disagrees with diff | Correct prose or implementation, then refresh evidence |
| Diagram extractor fails | Omit unsupported visual and expose the structural question |
| Head changes during review | Show stale package state and rebuild affected records |
| CI omits required jobs | Treat missing evidence as missing, not passing |
| Scope expands unexpectedly | Explain or split work before approval |
| Evidence contains sensitive data | Redact public output and retain appropriate private evidence |
| Domain owner unavailable | Preserve pending status and identify the actual decision blocked |

Failure reporting should be specific.
“Could not verify” without naming the property leaves the next person to restart the investigation.

### 20.7 Maintain an evidence budget

Review attention and verification execution both have costs.
Spend them on unresolved questions.

For a local fix, a regression and a small compatibility check may be enough.
For a schema migration, execution against old data and rollout compatibility matter more than a decorative sequence diagram.
For an authorization change, boundary cases and a qualified reviewer matter more than a broad passing suite unrelated to policy.

Do not collect every possible artifact.
Collect the artifacts needed to assess this proposal.

### 20.8 Audit quality through samples

Periodically sample accepted packages and ask:

- Did the summary match the diff?
- Was scope expansion explained?
- Did evidence belong to the accepted head?
- Were material limits visible?
- Did the right owner inspect high-consequence concerns?
- Did collapsed content hide a relevant change?
- Did diagrams contain unsupported edges?

Use findings to improve the template and tooling.
Do not turn every minor omission into a new mandatory section.
A form that grows indefinitely eventually becomes boilerplate.

### 20.9 Assign maintenance

Useful responsibilities:

| Role | Responsibility |
|---|---|
| Author or agent operator | Accurate package, complete scope, current evidence |
| Reviewer | Independent judgment and clear disposition |
| Domain owner | Consequential contract and remaining uncertainty |
| Tool maintainer | Rendering, identity, provenance, extraction limits |
| Evaluation owner | Measures effectiveness and false positives |

Small teams can combine roles.
They still need a clear answer to who repairs stale automation and who decides whether a high-risk gap is acceptable.

### 20.10 Keep the operating principle simple

The practice should make three actions easy:

- Find the changed behavior.
- Challenge the claims with evidence.
- Accept or reject a particular snapshot.

A short package can do that well.
A sophisticated surface can do it badly if it hides code, merges uncertain judgments, or lets attractive structure imply correctness.

The tool's success should be judged through reviewer decisions and operating outcomes.
The presentation's job is to support those decisions with accurate, inspectable information.

---

## 21. Appendix: templates, qualifications, and sources

These templates are proposed team contracts.
They are not schemas required by GitHub, CodeRabbit, or any coding agent.
Delete fields that do not help assess a small change.
Keep consequential unknowns visible.

### 21.1 Plan-review template

```markdown
# Plan: <observable problem>

## Intent

<Problem, affected user or caller, and task/spec link.>

## Decision requested

<Specific design or product question, or state that requirements
are already settled and this is a scope/verification review.>

## Approach

<Chosen design and why it satisfies the contract.>

Alternatives:
- <Option>: <relevant tradeoff>.
- <Option>: <relevant tradeoff>.

## Scope

| Path/module | Responsibility | Reason |
|---|---|---|
| <path> | <change> | <requirement> |

Excluded work:
<Boundaries; identify any exclusions that prevent deployment.>

## Assumptions and unknowns

| ID | Statement | Basis | Verification | Owner | Status |
|---|---|---|---|---|---|
| A1 | <statement> | <source/inference> | <method> | <owner> | unresolved |

## Risk

<Tier and concrete failure consequence.>
<Affected callers, data, permissions, and operational boundaries.>

## Verification

- Regression to reproduce before the fix: <behavior and command>.
- Preserved behavior: <checks>.
- Failure/boundary checks: <checks>.
- Manual/visual checks: <procedure>.
- Integration or rollout gaps: <owner and required evidence>.

## Structural proposal

<Include only when useful. Label as a proposed design.
State the question the sketch answers.>

## Slicing and delivery

<One change or stack, dependencies, deployment order,
and recovery constraints.>
```

### 21.2 Post-change template

```markdown
# <One-line intent>

Task/plan: <link>
Snapshot: <comparison-base> → <head>
Target tip: <commit>
Comparison rule: <rule>
Risk: <tier> because <consequence>

## Scope and decisions

- Declared versus actual paths: <inventory>.
- Undeclared changes: <reason, or none>.
- Planned but absent work: <reason, or none>.
- Plan deviations: <decision and owner>.
- New assumptions: <status and verification>.
- Requested review: <specific question>.

## Evidence

Before: <observable behavior>.
After: <observable behavior>.

| Claim | Method/artifact | Tested revision | Result | Limits |
|---|---|---|---|---|
| <claim> | <command/link> | <revision> | <status> | <boundary> |

Counterfactual:
<Same test identity, base failure, head result, conditions.>

Test/CI alert:
<Deleted/skipped tests, weakened assertions, discovery,
coverage, and workflow changes.>

Not verified:
<Specific gaps and consequences for approval or release.>

## Review tour

1. <Highest-consequence concern>: <anchors, risk, evidence, question>.
2. <Next concern>: <anchors, risk, evidence, question>.
3. <Tests>: <anchors and behavioral claims>.

Mechanical/generated:
<Counts, provenance, verification, raw-view link.>

Ungrouped:
<Remaining hunks, or none.>

## Structural delta

<Omit unless useful. Include source revision, extraction or
authorship method, supporting anchors, limits, verification status.>

## Complete diff

<Concern-order view and ordinary raw/file view.>

## Delivery and recovery

<Required when the change affects rollout, state, or compatibility.>
```

### 21.3 Evidence-record template

```json
{
  "schema": "example-review-evidence/v1",
  "id": "E1",
  "claim": "<property being checked>",
  "implementation_revision": "<full-commit>",
  "test_identity": {
    "revision": "<test-source-commit-or-fixture-label>",
    "sha256": "<test-bytes-digest>"
  },
  "command": ["<executable>", "<argument>"],
  "working_directory_role": "disposable-head-checkout",
  "started_at": null,
  "finished_at": null,
  "environment": {
    "runtime": "<version>",
    "database": "<engine-and-version>",
    "dependencies": "<lock-or-environment-identity>"
  },
  "exit_code": null,
  "result": "not-run",
  "artifact": null,
  "limitations": ["<unexercised behavior>"]
}
```

An argument array avoids ambiguous shell command reconstruction.
If a check genuinely needs a shell, record the shell and exact script.
Do not interpolate untrusted file names into shell text.

`null` and `not-run` are unfilled states.
A renderer must not interpret them as success.

### 21.4 Structural-view record

```yaml
schema: example-structural-view/v1
question: Which writes belong to the acceptance transaction?
kind: sequence-delta
base: "<full-commit>"
head: "<full-commit>"
source:
  method: human-inspected-source
  extractor: null
  trace: null
verification:
  grammar: not-run
  anchors: pending
  semantic_review: pending
nodes: []
edges: []
omissions:
  - external worker
  - authentication adapter
status: draft
```

A populated diagram needs supported node and edge records.
The empty arrays in this template are not a representation of a real change.

### 21.5 Reviewer checklist

Use this as a decision aid:

- I know the requested behavior and the reviewed snapshot.
- The actual scope is explained, including planned work that is absent.
- I inspected changes to verification and execution controls.
- I examined the consequential concern and its failure paths.
- The important claims have relevant implementation and execution support.
- Test changes assert observable behavior and expose weakening.
- Structural views have provenance and do not replace code inspection.
- Material unknowns are visible and assigned to appropriate owners.
- My disposition applies to the current snapshot.

This is not evidence that review occurred merely because every box is checked.
The useful record is the judgment and its supporting observations.

### 21.6 Local execution record for the example

The chapter 13 code blocks were extracted verbatim into separate temporary directories.
The same test file was copied to both.
Execution and the six mutation checks completed during preparation on November 24, 2026; the recorded completion timestamp was `2026-11-24T10:40:27Z`.

| Item | Observed value |
|---|---|
| Python | 3.14.7 |
| SQLite | 3.53.4 |
| Baseline command | `python3 -m unittest -v test_receiver.RegressionTests` |
| Baseline result | Exit 1; one intended assertion failure, `2 != 1` |
| Head command | `python3 -m unittest -v test_receiver` |
| Head result | Exit 0; seven tests passed |
| Mutation exercise | Six deliberately incorrect variants; each targeted check failed |

Content identities, including the terminating newline:

```text
base/receiver.py SHA-256
79e057ed8e39b61a7e2d0f594add52ec433a06fad2500640b1ac20ab2f3c92a8

head/receiver.py SHA-256
40df8447f3856536c298c467607be3195bcee2730a79aebcad417708e4be7c3e

test_receiver.py SHA-256
cb26f3248eb6570d06e6ae582c6126c3cefbb707f96d7f5f69815f62a8d599d3
```

These are standalone fixture identities.
They are not repository commit hashes.
The temporary test directories and logs are not required repository dependencies.
Readers can reproduce the checks from the complete code blocks.

The mutation exercise changed one behavior at a time:

| Variant | Targeted test | Observed detection |
|---|---|---|
| Baseline insertion on every call | Duplicate regression | Assertion failure |
| Receipt lookup without tenant | Tenant independence | Unexpected conflict error |
| Conflict branch disabled | Conflicting order | Expected exception absent |
| Receipt committed before job | Rollback | Receipt count remained one |
| Durable lookup result ignored | Fresh interpreter | Subprocess failed on duplicate job constraint |
| Identifier validation skipped | Invalid identifiers | Expected rejection absent, including unintended writes |

Some mutations produce errors rather than the intended assertion failure.
For this exercise that still demonstrates rejection of a known-bad implementation.
For the base regression, the exact `2 != 1` assertion is required because it establishes the specific historical behavior being demonstrated.

The exercise does not cover all transaction failures, all schedulers, or another database engine.
It is a bounded check of the example's behavioral tests.

### 21.7 Qualifications to the supplied research

The research supplied with this project is a useful synthesis.
Several statements need a narrower interpretation in a book intended for reuse.

| Supplied framing | Qualification used here |
|---|---|
| Understanding is the bottleneck rather than defect spotting | Foundational work identifies understanding as central; review still has several objectives and task-dependent bottlenecks |
| Last-file defects have 64% lower detection | The experiment reports 64% lower odds for one of two studied defect types |
| Inconsistent descriptions cause rejection | Mining shows an association; inconsistency can correlate with implementation quality and other factors |
| Diagrams are about 2% of PR visuals | The figure concerns sampled images; media types and all-PR prevalence use different denominators |
| Visuals do not change outcomes | The study found no significant overall merge-time or efficiency difference in its matched comparison and observed other review associations |
| Code Faithfulness measures disagreement with repository code | In the cited UML taxonomy, it means disagreement between reasoning and generated PlantUML |
| A call graph can generate a verified sequence | Calls alone do not establish execution order, branch conditions, or transaction scope |
| Vendors moved away from always-on diagrams | Checked CodeRabbit documentation describes interaction-triggered walkthrough diagrams and Explain layers/graphs; it does not establish a universal replacement policy |
| A generated quality judgment establishes a good tour | Generated assessments need their own evaluation; presentation quality and behavioral correctness remain separate |
| The full layered format is effective | It is a proposed combination of evidence-informed ingredients, requiring direct evaluation |

The supplied report also contains vendor usage figures, productivity estimates, and practitioner experiments.
This edition does not repeat their magnitudes as established facts.
It does not independently assess current capabilities of every tool named in that report.
Unassessed means unassessed.
It should not be read as absent or ineffective.

Supplementary claims about whole-system architectural comprehension, ReviewVis preferences, and particular practitioner recommendations remain useful context.
They are not treated here as direct proof that the proposed agent-review format improves defect detection.

### 21.8 Sources and reading notes

Sources are identified by `[S#]` in the text.
Links point to primary papers, primary publication pages, or official product documentation.
Search result pages were used for discovery, not as the final reference.

For the foundational Microsoft paper, publication identity was verified through DOI metadata; the primary full text could not be retrieved in this research session.
The broad understanding finding follows the supplied research and the established publication.
The Google PDF and the arXiv full texts for S3, S7–S11, S13–S15, and S21 were inspected.
For S4–S6 and S12, the primary abstract pages were checked; their detailed results are deliberately summarized at a high level.
For S22, the publisher-deposited abstract and publication metadata were checked through Crossref; the primary full text was inaccessible in this session.
Product documentation describes features, not independent effectiveness.

**[S1] Alberto Bacchelli and Christian Bird.**
*Expectations, Outcomes, and Challenges of Modern Code Review.*
ICSE, 2013.
[Publication DOI](https://doi.org/10.1109/ICSE.2013.6606617).
Foundational understanding and review-outcome context.

**[S2] Caitlin Sadowski, Emma Söderberg, Luke Church, Michal Sipko, and Alberto Bacchelli.**
*Modern Code Review: A Case Study at Google.*
ICSE-SEIP, 2018.
[Primary publication page](https://research.google/pubs/modern-code-review-a-case-study-at-google/).
[Paper](https://storage.googleapis.com/gweb-research2023-media/pubtools/4476.pdf).
Large industrial case study; descriptive practice and review context.

**[S3] Enrico Fregnan, Larissa Braz, Marco D'Ambros, Gül Çalikli, and Alberto Bacchelli.**
*First Come First Served: The Impact of File Position on Code Review.*
ESEC/FSE, 2022.
[Paper](https://arxiv.org/abs/2208.04259).
Mining plus a controlled experiment; attention and task-specific defect detection.

**[S4] Farid Bagirov, Pouria Derakhshanfar, Alexey Kalina, Elena Kartysheva, and Vladimir Kovalenko.**
*Assessing the Impact of File Ordering Strategies on Code Review Process.*
2023.
[Primary abstract and paper](https://arxiv.org/abs/2306.06956).
Industrial ordering study; complements the attention result.

**[S5] Md Shamimur Rahman, Zadia Codabux, and Chanchal K. Roy.**
*Breaking the Alphabet: Rethinking File Ordering in Code Review.*
ICSE, 2026.
[Primary abstract and paper](https://arxiv.org/abs/2609.04207).
Large survey; preferences and perceived difficulty are not measured detection.

**[S6] Md Shamimur Rahman, Zadia Codabux, and Chanchal K. Roy.**
*Does Order Matter? An Empirical Investigation into the Impact of File Ordering on Code Review Effectiveness.*
2026 preprint.
[Primary abstract and paper](https://arxiv.org/abs/2609.22610).
Outcome-level mining with modest associations; avoid causal claims.

**[S7] Shirin Pirouzkhah, Pavlína Wurzel Gonçalves, and Alberto Bacchelli.**
*The Value of Effective Pull Request Description.*
MSR, 2026.
[Paper](https://arxiv.org/abs/2602.14611).
Description taxonomy and associations with engagement and acceptance.

**[S8] Jingzhi Gong, Giovanni Pinna, Yixin Bian, and Jie M. Zhang.**
*Analyzing Message-Code Inconsistency in AI Coding Agent-Authored Pull Requests.*
MSR, 2026.
[Paper](https://arxiv.org/abs/2601.04886).
Agent PR message–code inconsistency; observational outcomes.

**[S9] Tao Xiao, Hideaki Hata, Christoph Treude, and Kenichi Matsumoto.**
*Generative AI for Pull Request Descriptions: Adoption, Impact, and Developer Interventions.*
FSE, 2024.
[Paper](https://arxiv.org/abs/2402.08967).
Generated descriptions and developer supplementation; quasi-experimental analysis.

**[S10] Lo Gullstrand Heander, Agnia Sergeyuk, Ilya Zakharov, Emma Söderberg, and Nikita Mukhortov.**
*Trust-Calibrated Code Review: A Participatory Design Study of Review Workflows for LLM-Generated Multi-File Changes.*
2026 preprint.
[Paper](https://arxiv.org/abs/2606.01969).
Workflow design and perceived usefulness; no measured end-to-end detection benefit.

**[S11] Ranim Khojah, Francisco Gomes de Oliveira Neto, Mazen Mohamad, Julian Frattini, and Philipp Leitner.**
*Same Scrutiny, More Time: Eye Tracking Insights into Reviewing LLM-Labelled Code.*
2026 preprint.
[Paper](https://arxiv.org/abs/2606.26505).
Attention and scan behavior; does not measure defect detection.

**[S12] Neil Perry, Megha Srivastava, Deepak Kumar, and Dan Boneh.**
*Do Users Write More Insecure Code with AI Assistants?*
CCS, 2023.
[Primary abstract and paper](https://arxiv.org/abs/2211.03622).
Security-task performance and confidence; scoped to the studied assistant and tasks.

**[S13] Tianzi Hao and colleagues.**
*An Empirical Study of Visual Artifacts in Pull Request Descriptions.*
2026 preprint, VisPR.
[Paper](https://arxiv.org/abs/2611.23289).
Human-authored PR visuals; observational study, not a controlled agent-diagram experiment.

**[S14] Mojdeh Rahmanian, Foutse Khomh, Yanchao Yu, and Ashkan Sami.**
*Tracing Hallucinations in LLM Reasoning for UML Class Diagram Generation: An Empirical Taxonomy and Mitigation Study.*
2026 preprint.
[Paper](https://arxiv.org/abs/2610.30946).
Requirements-to-reasoning-to-PlantUML failures; not a repository-diff benchmark.

**[S15] Alessio Ferrari, Sallam Abualhaija, and Chetan Arora.**
*Model Generation with LLMs: From Requirements to UML Sequence Diagrams.*
2024 preprint; the linked HTML version also uses the title *Model Generation from Requirements with LLMs: an Exploratory Study*.
[Paper](https://arxiv.org/abs/2404.06371).
Requirements-based generation and qualitative correctness/completeness limits.

**[S16] CodeRabbit.**
*PR Walkthroughs* and *Configuration V2 Reference.*
[Walkthrough documentation](https://docs.coderabbit.ai/pr-reviews/walkthroughs).
[Configuration reference](https://docs.coderabbit.ai/reference/configuration).
Official documentation for summaries and configurable sequence diagrams.

**[S17] CodeRabbit.**
*CodeRabbit Explain.*
[Official documentation](https://docs.coderabbit.ai/change-stack/index).
Layered review workspace, summaries, and graphs; vendor feature description.

**[S18] CodeRabbit.**
*Snapshots and Freshness.*
[Official documentation](https://docs.coderabbit.ai/change-stack/snapshots).
Saved review context versus actions on a live PR.

**[S19] GitHub.**
*Best Practices for Using GitHub Copilot to Work on Tasks.*
[Official documentation](https://docs.github.com/en/copilot/using-github-copilot/coding-agent/best-practices-for-using-copilot-to-work-on-tasks).
Agent-task and review workflow; the product and URL may redirect as naming evolves.

**[S20] Anthropic.**
*Claude Code Common Workflows.*
[Official documentation](https://code.claude.com/docs/en/common-workflows).
Planning and investigation workflow; feature documentation, not a review-effectiveness study.

**[S21] Kacper Duma, Patryk Wróblewski, Jagoda Bobińska, Julia Winiarska, and Piotr Przymus.**
*These Aren't the Reviews You're Looking For: How Humans Review AI-Generated Pull Requests.*
EASE, 2026.
[Paper](https://arxiv.org/abs/2605.02273).
Recorded human and automation-mediated review activity; useful caution about metric interpretation.

**[S22] Marco di Biase, Magiel Bruntink, Arie van Deursen, and Alberto Bacchelli.**
*The Effects of Change Decomposition on Code Review—a Controlled Experiment.*
PeerJ Computer Science, 2019.
[Publication DOI](https://doi.org/10.7717/peerj-cs.193).
Controlled experiment with 28 developers; abstract supports the narrower false-positive finding.

### 21.9 A compact starting contract

For a team adopting this practice, the first repository rule can be short:

```markdown
Present each agent change with:

- The observable intent and reviewed base/head.
- Declared versus actual scope, deviations, and unresolved assumptions.
- Evidence for the important behavior, including the intended base failure
  for regressions where feasible, and explicit unverified limits.
- A concern tour for multi-file work, with consequential changes early.
- A supported structural delta only when it answers a review question.
- Access to the complete diff and changed tests.

An approved plan does not approve the implementation.
Generated summaries, diagrams, and AI findings remain review inputs.
Required human ownership and release policy still apply.
```

Evaluate that contract on real review tasks.
Keep the parts that improve understanding and decisions.
Revise the parts that become boilerplate or create false certainty.
