# Building Reliable AI Applications

## A Practical Book on Evaluation Systems

AI applications do not start working reliable because you deploy it with the most capable model or tht a couple example use-cases succeed. They become reliable when you have a system in place that ensures quality.

This book is a field guide to building that capability. It applies to question-answering systems, retrieval-augmented generation, extraction pipelines, chat bots, workflow automations, and tool-using agents. Its recommendations are intentionally practical: begin with small, inspectable work, automate only after you understand it and keep yourself close to the evidence.

---

## Contents

1. [The job of evaluation](#1-the-job-of-evaluation)
2. [Start with a decision, not a metric](#2-start-with-a-decision-not-a-metric)
3. [The smallest useful evaluation system](#3-the-smallest-useful-evaluation-system)
4. [Capture traces that can explain behavior](#4-capture-traces-that-can-explain-behavior)
5. [Turn observations into a failure taxonomy](#5-turn-observations-into-a-failure-taxonomy)
6. [Build a dataset that represents the product](#6-build-a-dataset-that-represents-the-product)
7. [Choose the right kind of evaluator](#7-choose-the-right-kind-of-evaluator)
8. [Use model-based judges without fooling yourself](#8-use-model-based-judges-without-fooling-yourself)
9. [Evaluate retrieval, structured output, and safety](#9-evaluate-retrieval-structured-output-and-safety)
10. [Evaluate tool-using agents](#10-evaluate-tool-using-agents)
11. [Run experiments and regression gates](#11-run-experiments-and-regression-gates)
12. [Make production evaluation a learning loop](#12-make-production-evaluation-a-learning-loop)
13. [Build the team and operating model](#13-build-the-team-and-operating-model)
14. [Avoid the common traps](#14-avoid-the-common-traps)
15. [A ninety-day rollout plan](#15-a-ninety-day-rollout-plan)
16. [Appendix: templates](#16-appendix-templates)

---

## 1. The job of evaluation

An evaluation is a repeatable method for deciding whether an AI system behaved correctly for a particular case. An evaluation system is putting that into practice by wrapping a larger loop around the method:

```text
Real or designed cases
        ↓
Run the application
        ↓
Inspect outputs and traces
        ↓
Label outcomes and diagnose failures
        ↓
Change model, prompt, tools, data, or product flow
        ↓
Re-run the same cases and monitor new behavior
```

The goal is to make engineering and product decisions with less guesswork.

Evalations are the only real signal we have. Unlike traditional software where you can assert correctness code and tests, LLM behavior is probalistic and depends on prompts, model version and context. Without evals you ar basicallky guessing whether a change made things better or worse.

### AI quality is multidimensional

Traditional software engineering is mostly determenistic: a function returns the expected value and a test ensures it doesn't regress. Building AI applications however isn't as clear cut. A response can be factually correct but toxic, concise but incomplete, safe but not completing the task, or fluent while halucinating.

That probablity behavior should also influence you how to judge the output of AI integrating. Therefore, avoid asking, “What is our quality score?” Ask a more precise questions:

> Is this system good enough on the dimensions that matter for this decision and this risk?

Typical dimensions include:

- Task completion
- Factual accuracy
- Groundedness in approved evidence
- Relevance and usefulness
- Correct format or schema
- Appropriate tool use
- Safety and policy compliance
- Latency and cost
- User trust and satisfaction

No universal metric captures all of these. A good system makes trade-offs explicit.

### Evaluation is part of the development, not an after thought

### Reliability comes from feedback, not clear requirements

Prompts matter, but feedback matters more. An application improves when it gives the model and the team useful signals:

- deterministic checks for things that must be exact
- test fixtures for known regressions
- tools that return informative errors
- a clear record of what happened in each run
- targeted human review for ambiguous or high-impact decisions
- release gates that prevent known problems from returning

The result is a system that can learn from its own failures instead of repeating them.

---

## 2. Start with a decision, not a metric

The first question is not “Which evaluation framework should we use?” It is:

> What decision will this evaluation help us make?

An evaluation that cannot change a decision is usually reporting noise.

### Examples of useful decisions

| Decision                                  | Evidence needed                                                         |
| ----------------------------------------- | ----------------------------------------------------------------------- |
| Should we ship a new prompt?              | Regression results on high-risk and representative cases                |
| Should we switch models?                  | Quality, latency, and cost comparison on the same task distribution     |
| Should this workflow be autonomous?       | Completion rate, unsafe-action rate, escalation rate, and reversibility |
| Should we add a retrieval source?         | Change in answer quality, citation support, and harmful retrievals      |
| Where should engineering invest next?     | Frequency and impact of observed failure categories                     |
| Is a new tool worth exposing to an agent? | Tool-selection accuracy, argument correctness, and net task success     |

### Write an evaluation contract

Before collecting data, write a short contract. It should fit on one page and be understandable by product and engineering.

```yaml
feature: account-support assistant
user_job: resolve a billing question without contacting a human
decision: approve or reject a new retrieval and prompt configuration

success_definition:
  - Gives the correct policy answer when policy evidence is available
  - Does not invent account actions or transaction details
  - Escalates when the request requires identity verification or an unsupported action

non_negotiable_failures:
  - Claims that an account was changed when it was not
  - Exposes information from another account
  - Gives advice that violates policy

important_tradeoffs:
  - Accuracy is more important than brevity
  - A safe escalation is preferable to an unsupported answer
  - Median response time must remain below the product budget
```

This contract prevents a common mistake: improving a convenient proxy while degrading what users actually need.

### Define a baseline

Every experiment needs a baseline. It may be:

- the production configuration;
- a simpler non-AI workflow;
- a previous prompt or model;
- a human process;
- a rules-based classifier;
- a deliberately minimal version of the application.

The non-AI baseline is especially important. If a form, search page, template, or conventional workflow solves the user problem more reliably, an AI layer is not automatically an improvement.

### Separate ship blockers from optimization goals

Not all criteria deserve equal treatment.

**Ship blockers** are failures that should stop a release, such as unsafe actions, sandbox leaks, schema invalidity, or halucinations. They should have strict thresholds and clear ownership.

**Optimization goals** are areas where improvement is valuable but a small regression may be acceptable, such as verbosity, stylistic consistency, or a small latency increase in exchange for a major accuracy gain.

Don't mix these into one overall score, it hides the trade-off, keep them separate.

---

## 3. The smallest useful evaluation system

Do not begin by buying a vendor tool or designing a large benchmark. Begin with a small table and a review process.

### The minimum viable benchmark

Create a spreadsheet, notebook, or simple db table with these columns:

| Field               | Purpose                                                       |
| ------------------- | ------------------------------------------------------------- |
| `case_id`           | Stable identifier for a scenario                              |
| `input`             | User request or task definition                               |
| `context`           | Documents, history, state, or tool setup used by the app      |
| `output`            | Final response or action result                               |
| `configuration`     | Model, prompt version, retrieval version, tools, and settings |
| `expected_behavior` | What must happen, not necessarily an exact answer             |
| `label`             | Pass, fail, or needs-review                                   |
| `failure_tags`      | One or more categories when it fails                          |
| `review_notes`      | Short explanation from a reviewer                             |
| `risk`              | Low, medium, high, or critical                                |

Run twenty to fifty cases through the application. Review the results manually. Write plain-language notes. This exercise will often reveal that the team has not yet agreed on the task, the desired behavior, or the relevant edge cases.

At this stage, resist the urge to calculate a grand score. The useful output is insight:

- Which failures recur?
- Which failures hurt most?
- Which requirements were never written down?
- Which parts of the system are easy to check mechanically?
- Which judgments require a domain expert?

### The review rhythm

Use a small recurring practice:

1. Review a sample after meaningful changes.
2. Review a fresh production sample at a regular interval.
3. Review after incidents, complaint spikes, model changes, or new tool access.
4. Record the first meaningful failure in each trace.
5. Turn recurring, important failures into durable test cases.

New products benefit from frequent review because their task distribution and failure modes are still moving. Mature products can review less often, but should immediately increase scrutiny after a material change.

### One accountable quality owner

Crowdsourced quality definitions often produce inconsistent labels and long debates. For a narrow product capability, appoint a principal domain owner who makes the final call on ambiguous examples.

This does not mean one person must label everything forever. It means someone is accountable for the definition of acceptable behavior, for resolving disagreements, and for updating the rubric when the product changes.

---

## 4. Capture traces that can explain behavior

A final answer alone is rarely enough to diagnose an AI application. You need a trace: the record of a request from initial input through intermediate decisions to final outcome.

### What to record

For each run, capture enough information to reproduce and explain it:

```yaml
trace_id: tr_01842
timestamp: 2026-09-08T10:15:00Z
request:
  user_input: "Can I change the delivery address?"
  user_context:
    locale: en-GB
    authenticated: true

configuration:
  application_version: 2026.09.08.3
  prompt_version: support-v17
  model: model-family/version
  retrieval_index: policy-2026-09
  tool_policy: support-tools-v4

spans:
  - name: classify_request
    input: ...
    output: address_change
    latency_ms: 180
  - name: retrieve_policy
    query: ...
    documents: [...]
  - name: generate_response
    prompt_hash: ...
    output: ...
  - name: invoke_tool
    tool: request_address_change
    arguments: ...
    result: ...

outcome:
  final_response: ...
  final_action: submitted
  latency_ms: 2100
  cost_estimate: ...
```

Do not log sensitive content indiscriminately. Apply data minimization, access control, retention limits, and redaction. A trace must be useful without becoming a privacy liability.

### Make configurations reproducible

Every result should be attributable to an immutable configuration. At minimum, record:

- application version;
- prompt and policy versions;
- model identifier and generation settings;
- retrieval index or corpus version;
- tool definitions and permission policy;
- code revision for deterministic logic.

Without this, a team cannot answer the most basic regression question: “What changed?”

### Observe the context, not only the output

For agents, the effective input is the entire context window: instructions, conversation history, retrieved data, tool results, summaries, and injected policies. When an agent behaves strangely, the explanation often lies there:

- a tool returned malformed data;
- an earlier turn established a false assumption;
- the context accumulated conflicting instructions;
- relevant information was omitted;
- a summary discarded the crucial constraint;
- a long context diluted the important evidence.

Treat context as a first-class observable artifact. Debugging an agent without seeing its context is like debugging a program without logs or stack traces.

### Trace quality affects evaluation quality

Bad instrumentation creates false diagnoses. For example:

- If retrieved documents are not captured, a reviewer may blame generation for a retrieval failure.
- If tool arguments are hidden, a task failure may be misclassified as a wrong tool choice.
- If the configured model is not logged, a quality drop may be mistaken for a prompt regression.
- If user edits or retries are not attached, the product may miss dissatisfaction signals.

Instrumentation is not an administrative task. It determines whether the organization can learn from production behavior.

---

## 5. Turn observations into a failure taxonomy

The most valuable early evaluation activity is error analysis: systematically inspecting examples to discover how the application fails in the real world.

### The error-analysis loop

1. **Sample traces.** Mix random traffic, user complaints, long sessions, retries, low-confidence runs, high-cost runs, and recently changed flows.
2. **Write open notes.** A domain reviewer describes what went wrong without forcing it into a preexisting label.
3. **Group similar notes.** Cluster observations into a failure taxonomy.
4. **Count frequency and assess impact.** A rare critical safety failure can outrank a common minor formatting issue.
5. **Choose an intervention.** Fix a bug, improve the product flow, adjust retrieval, add a tool, update a policy, write a deterministic check, or build an evaluator.
6. **Add representative cases.** Important failures become regression tests.
7. **Repeat.** The taxonomy should evolve as the product and users evolve.

### Start with the first failure

Multi-step systems often produce cascades. A bad retrieval may cause a wrong plan, then a failed tool call, then an invented explanation. Labeling every downstream symptom can obscure the root cause.

Start by recording the first consequential failure. Add secondary tags only when they are independent and useful. This leads to interventions earlier in the causal chain.

### An example taxonomy

For a knowledge assistant, a first draft might be:

| Category             | Description                                                | Typical intervention                              |
| -------------------- | ---------------------------------------------------------- | ------------------------------------------------- |
| Missing evidence     | Relevant source was not retrieved                          | Improve indexing, query formation, or corpus      |
| Unsupported claim    | Answer goes beyond available evidence                      | Grounding check, response policy, citation UX     |
| Wrong interpretation | System misunderstood user intent                           | Improve routing, clarify UX, add examples         |
| Incomplete answer    | Correct but misses a necessary step                        | Prompt, workflow, or rubric update                |
| Policy violation     | Gives a prohibited instruction                             | Deterministic rule, policy lane, human escalation |
| Bad escalation       | Refuses when it could act, or acts when it should escalate | Revise decision boundary                          |
| Presentation defect  | Correct content in an unusable format                      | Schema enforcement, rendering tests               |

For a tool-using agent, add:

| Category               | Description                                                   |
| ---------------------- | ------------------------------------------------------------- |
| Wrong tool             | Selected an unsuitable capability                             |
| Invalid arguments      | Chose the right tool but passed malformed or incorrect values |
| Bad plan               | Worked toward the wrong goal or violated a constraint         |
| Tool-result misuse     | Received useful output but interpreted it incorrectly         |
| Premature success      | Claimed completion without verifying the intended outcome     |
| Inefficient trajectory | Reached the result with unacceptable time, cost, or steps     |

### Saturation, not perfection

Continue reviewing until new traces rarely reveal new, consequential categories. The goal is not to enumerate every imaginable failure. It is to find the patterns that should guide the next investment.

The taxonomy is a living product artifact. It should be versioned, discussed in planning, and updated when the system’s behavior changes.

---

## 6. Build a dataset that represents the product

An evaluation dataset is not a static exam. It is a curated collection of cases used to compare configurations and prevent known failures from returning.

### Use several case sources

A healthy dataset blends:

- **Production examples:** the strongest evidence of actual user behavior.
- **Incident cases:** high-value regressions from past failures.
- **Expert-authored cases:** known policies, rare but serious edge cases, and important business rules.
- **Synthetic cases:** systematic coverage of combinations not yet common in traffic.
- **Adversarial cases:** attempts to break instructions, boundaries, or tools.
- **Canary cases:** a small, stable set that runs on every change.

No one source is sufficient. Production data is realistic but may omit rare risk. Synthetic data improves coverage but can become artificial. Expert cases encode important knowledge but may overrepresent what experts expect rather than how users speak.

### Preserve the distribution—and challenge it

The dataset should resemble the product’s task distribution, but it should not merely mirror the most common requests. Partition it:

```text
Evaluation set
├── Common tasks
├── High-value tasks
├── Long-tail tasks
├── Safety and policy boundaries
├── Known historical failures
├── New feature cases
└── Adversarial and abuse cases
```

Report results by slice. An overall pass rate can conceal a regression in a high-value category.

### Capture behavior, not only golden strings

For open-ended language tasks, exact reference answers are often too rigid. Write expected behavior instead:

```yaml
case_id: billing_042
input: "I was charged twice for the same order. What should I do?"

must:
  - Explain the appropriate resolution path
  - Avoid claiming that a refund has already been issued
  - Ask only for information needed to continue

must_not:
  - Request full payment-card information
  - Promise a timeline that policy does not support

preferred:
  - Acknowledge the concern in the opening sentence
  - Give the next action in a scannable format
```

This gives reviewers and automated judges a clear target without rewarding superficial phrase matching.

### Create synthetic cases systematically

Avoid the vague prompt “Generate test questions.” It yields repetitive, obvious examples. Instead, define the dimensions of the task:

```yaml
dimensions:
  customer_status: [new, returning, verified, restricted]
  request_type: [refund, delivery, subscription, account_access]
  complexity: [single_step, multi_step, ambiguous, contradictory]
  evidence_state: [complete, incomplete, conflicting, unavailable]
  risk_level: [low, medium, high]
```

Then:

1. Manually write a small set of meaningful combinations.
2. Generate additional structured combinations.
3. Filter impossible or irrelevant combinations.
4. Convert combinations into natural language in a separate step.
5. Run them through the actual application and inspect the traces.

Generating structure before phrasing produces better coverage and less stylistic repetition.

### Balance successes and failures

An evaluator cannot learn a useful decision boundary from a dataset full of easy passes. Include meaningful failures. If real failures are scarce, use weaker configurations, older prompts, reduced context, missing retrieval, or deliberately constrained tools to generate plausible, product-shaped errors.

Do not invent bizarre defects solely to make a dataset look difficult. The failure distribution should support decisions the team actually expects to make.

### Keep a holdout set

Separate the dataset into:

- **Development set:** used during iteration;
- **Regression set:** known cases that must continue to work;
- **Holdout set:** protected cases consulted less often to detect overfitting;
- **Production sample:** fresh data that prevents the suite from becoming detached from reality.

If a team repeatedly tunes until a benchmark passes, it is optimizing for the benchmark. Holdouts and fresh production review are the antidote.

---

## 7. Choose the right kind of evaluator

Use the cheapest evaluator that faithfully measures the requirement. The best evaluation systems are layered.

### Layer 1: deterministic checks

Use deterministic checks whenever the behavior can be specified precisely:

- JSON or schema validity;
- required fields;
- type and range constraints;
- forbidden content patterns;
- exact calculations;
- presence of approved identifiers;
- tool argument validation;
- authorization checks;
- latency and budget limits;
- expected state transitions.

These checks are fast, cheap, explainable, and ideal for release gates.

```python
def evaluate_order_lookup(result):
    assert result.schema_valid
    assert result.account_id == result.requested_account_id
    assert result.latency_ms < 3000
    assert "full_card_number" not in result.output.lower()
```

Do not use a language model to judge what ordinary code can verify.

### Layer 2: reference-based checks

Use a reference answer or reference facts when there is a stable correct target:

- extraction from a document;
- classification labels;
- arithmetic;
- code compilation and tests;
- retrieval of a known document;
- answer to a closed-domain question.

Use flexible matching when several answers are equivalent. A structured list of required facts is often more robust than string equality.

### Layer 3: rubric-based human review

Humans are essential when the judgment requires domain expertise, product understanding, or nuanced trade-offs. Use a rubric with concrete definitions and examples:

| Criterion     | Pass                                                  | Fail                                           |
| ------------- | ----------------------------------------------------- | ---------------------------------------------- |
| Groundedness  | Every material claim is supported by allowed evidence | Includes a material claim without support      |
| Actionability | User can complete the next step from the answer       | User is left without a viable next step        |
| Escalation    | Escalates when policy or capability requires it       | Acts beyond authority or refuses unnecessarily |

Binary pass/fail labels are often more useful than a five-point scale. They are quicker to apply, easier to calibrate, and map directly to release decisions. Add a short critique to preserve nuance.

### Layer 4: model-based judges

Model-based judges are useful for high-volume fuzzy checks after calibration against human labels. They can assess a narrow criterion such as relevance, groundedness, policy compliance, or comparison between two outputs.

They should supplement—not replace—human review.

### Layer 5: online outcomes

Offline evaluation asks whether a system _should_ perform well. Online evaluation asks whether it _does_ create value in use. Examples:

- completion rate;
- successful task outcome;
- time to resolution;
- rate of user correction or retry;
- escalation rate;
- explicit feedback;
- abandonment;
- downstream business result.

Online metrics are powerful but delayed and confounded. A user may accept a bad answer, or a successful task may depend on factors outside the AI. Use them alongside trace review and offline evaluations.

---

## 8. Use model-based judges without fooling yourself

An AI judge is another fallible model. Treat it as a measurement instrument that must be designed, calibrated, and monitored.

### Give each judge one job

Avoid a single all-powerful prompt that scores accuracy, relevance, tone, safety, completeness, and helpfulness at once. It hides disagreement and makes debugging impossible.

Prefer small evaluators:

```text
groundedness judge       → is every material claim supported?
policy judge             → does the response violate a rule?
task-completion judge    → did the user receive a usable next step?
style judge              → does the response meet presentation requirements?
tool-argument evaluator  → are arguments valid for the selected tool?
```

Combine results with explicit policy. For example, a safety failure blocks release; a style failure is recorded but does not.

### Build a judge from labeled examples

The workflow is straightforward:

1. Select a narrow criterion.
2. Gather examples that humans have labeled pass or fail.
3. Include the inputs, relevant context, output, and rubric.
4. Ask the judge for a binary decision and concise rationale.
5. Compare the judge’s decision to human labels.
6. Inspect disagreements.
7. Improve the rubric, examples, context, or judge design.
8. Repeat until accuracy is adequate for the decision’s risk.

Measure more than agreement. Check false positives and false negatives by failure category. A judge that overlooks a critical safety problem may be unacceptable even if its aggregate accuracy looks high.

### Example judge contract

```yaml
name: groundedness_v3
question: "Does the answer make any material claim that is not supported by the supplied evidence?"

inputs:
  - user_request
  - approved_evidence
  - assistant_answer

output:
  verdict: pass | fail
  unsupported_claims: [string]
  rationale: string

rules:
  - Ignore harmless paraphrases of supported evidence
  - Fail when the answer invents a policy, status, action, number, or deadline
  - Do not reward confident wording
  - If evidence is insufficient, the acceptable response should state uncertainty or escalate
```

### Calibrate the rubric before blaming the product

When a judge disagrees with a human, several things may be wrong:

- the system output is bad;
- the human label is wrong;
- the rubric is ambiguous;
- the judge lacks needed context;
- the reference answer is overly literal;
- the evaluator is applying the wrong standard.

Do not automatically “fix the model” when an evaluation fails. First determine whether the measurement itself is valid. Evaluation systems can have bugs too.

### Guard against judge bias

Model-based judges can be biased toward:

- longer answers;
- polished prose;
- familiar wording;
- their own model family;
- the first answer in a pairwise comparison;
- an answer that mirrors the rubric.

Mitigations include:

- randomizing answer order in comparisons;
- requesting concise justification;
- evaluating the same cases with more than one prompt or model when stakes justify it;
- using human spot checks;
- avoiding criteria that reward verbosity;
- designing judges around evidence and behavior rather than style.

### Use pairwise comparison when absolute scoring is vague

For some design choices, “Which answer better satisfies the rubric?” is more reliable than “Give this answer a 1–5 score.” Pairwise evaluation works well for comparing prompt variants, response formats, or models. Randomize order and allow ties.

---

## 9. Evaluate retrieval, structured output, and safety

Different architectures fail differently. Make the evaluation reflect the system, not just its final text.

### Retrieval-augmented generation

Evaluate retrieval and generation separately, then together.

**Retrieval questions:**

- Was relevant evidence retrieved?
- Was irrelevant or harmful evidence retrieved?
- Was the evidence current, permitted, and authoritative?
- Did rank order put the best evidence where the generator could use it?

**Generation questions:**

- Did the answer use the retrieved evidence correctly?
- Did it make unsupported claims?
- Did it reconcile conflicting evidence appropriately?
- Did it acknowledge when evidence was missing?

Useful retrieval metrics include recall at a practical cutoff, rank quality, corpus coverage, and stale-document rate. But inspect examples: a retrieval score cannot reveal whether a document was technically relevant yet misleading in context.

### Structured output

For extraction, routing, or APIs, establish a contract at three levels:

1. **Syntax:** Is the output valid JSON, XML, or another required format?
2. **Schema:** Are fields, types, enums, and constraints valid?
3. **Semantics:** Are the values correct for the input?

The first two should be deterministic. The third may require references, a domain reviewer, or a narrow judge.

```yaml
expected:
  intent: cancel_subscription
  requires_human: false
  entities:
    subscription_id: string

assertions:
  - output is valid JSON
  - intent is a permitted enum value
  - subscription_id is copied from the supplied state, not invented
  - requires_human is true when identity verification is missing
```

### Safety and policy

Safety evaluation should not be a generic “toxicity score.” It must model the actual harmful actions and boundaries of the application.

Build a policy matrix:

| Scenario                            | Expected behavior                                         | Evaluation                                  |
| ----------------------------------- | --------------------------------------------------------- | ------------------------------------------- |
| User requests prohibited action     | Refuse and provide safe alternative                       | Deterministic policy route + reviewed cases |
| Request needs authorization         | Verify, escalate, or limit action                         | State and permission checks                 |
| Tool result includes sensitive data | Redact or avoid exposing it                               | Data-flow and output checks                 |
| Prompt injection in retrieved text  | Ignore hostile instructions and follow application policy | Adversarial retrieval cases                 |
| Agent is uncertain                  | State limitation or hand off                              | Rubric-based evaluation                     |

Critical boundaries should be enforced outside the model wherever possible. A model can recommend an action; deterministic authorization and policy logic should decide whether the action is allowed.

---

## 10. Evaluate tool-using agents

An agent is not just a text generator. It is a loop that receives context, chooses actions, observes results, and updates its plan. Evaluation must inspect that loop.

### Separate the failure modes

At minimum, distinguish:

1. **Task interpretation:** Did the agent understand the user’s goal and constraints?
2. **Planning:** Did it choose a sensible sequence of actions?
3. **Tool selection:** Did it select the correct tool?
4. **Argument construction:** Did it provide valid and correct arguments?
5. **Tool-result interpretation:** Did it use the returned data correctly?
6. **State management:** Did it preserve necessary context and avoid conflicting actions?
7. **Verification:** Did it check that the task was actually complete?
8. **Efficiency:** Did it stay within reasonable time, cost, and step limits?

“Agent failure” is not a diagnosis. A specific failure category points to a specific intervention.

### Evaluate both trajectory and outcome

A final result can look correct even when the path was unsafe, expensive, or irreproducible. Conversely, an agent may take an awkward path and still complete a low-risk task acceptably.

Track both:

```yaml
outcome_metrics:
  - task_success
  - final_state_correct
  - policy_compliance

trajectory_metrics:
  - correct_tool_selection
  - valid_tool_arguments
  - redundant_tool_calls
  - number_of_steps
  - time_to_completion
  - estimated_cost
  - verification_performed
```

The right balance depends on product risk. A harmless internal research assistant may tolerate meandering. A financial workflow should not.

### Build tool-level evaluators

For each important tool, define:

```yaml
tool: schedule_transfer
preconditions:
  - user is authenticated
  - destination is approved
  - amount is within limit

valid_selection_when:
  - user explicitly requests a transfer
  - all required details are present

required_arguments:
  - source_account
  - destination_account
  - amount
  - currency

forbidden_arguments:
  - freeform_instructions

verification:
  - confirm returned transaction identifier
  - explain only what the tool result confirms
```

This lets the team locate whether a problem is routing, capability design, parameter extraction, or tool reliability.

### Make the environment testable

Agent evaluation is easier in a controlled environment:

- deterministic fixtures or sandbox accounts;
- stable tool responses for test cases;
- simulated failures and permission denials;
- resettable state;
- explicit success conditions;
- no irreversible effects during offline runs.

Tool errors are useful feedback. Return actionable messages that help an agent recover, rather than vague failures that cause random retries.

### Design for verifiability

Long action chains multiply uncertainty. Break workflows into right-sized units with observable checkpoints. Each unit should have:

- a clear goal;
- the minimal context needed;
- an allowed action set;
- a legible completion condition;
- a safe recovery or escalation path.

Where possible, use a separate verification step or verifier role. An agent pursuing a goal is often poorly positioned to judge its own work without independent evidence.

### Test adversarially

For every action-capable agent, test:

- missing or conflicting user details;
- malicious instructions in tool output or retrieved documents;
- partial tool failure;
- stale state;
- unavailable permissions;
- duplicate requests;
- ambiguous objectives;
- requests that tempt the agent to exceed its authority;
- success-like messages that do not prove success.

The last case is especially important. Agents should not infer completion from a plausible-looking response; they should verify the state that matters.

---

## 11. Run experiments and regression gates

An evaluation suite earns its value when it makes changes safe to compare.

### Treat each change as an experiment

For every material change, record:

```yaml
hypothesis:
  "Adding a policy-aware retrieval step will reduce unsupported answers
  for eligibility questions without increasing unnecessary escalations."

change:
  retrieval_query_template: v12
  response_policy: v8

comparison:
  baseline: production-2026-09-01
  candidate: branch-policy-retrieval
  dataset_slices:
    - eligibility_common
    - eligibility_ambiguous
    - policy_boundaries

success_criteria:
  - groundedness does not regress
  - unnecessary escalation decreases
  - critical policy failures remain zero
  - median latency stays within budget
```

Run baseline and candidate on the same cases. Compare results by slice, not only in aggregate.

### Use confidence and uncertainty honestly

Small datasets create noisy results. A two-point improvement across twenty cases may be chance. Report counts alongside percentages:

```text
Groundedness: 46/50 → 48/50
High-risk slice: 18/20 → 20/20
Median latency: 1.8s → 2.0s
```

For consequential decisions, use larger samples, repeated runs for nondeterministic systems, and confidence intervals or statistical tests appropriate to the metric. Do not pretend precision that the sample cannot support.

### Release gates

Good gates are specific, visible, and risk-based:

```yaml
release_gate:
  required:
    - all schema tests pass
    - no critical policy case fails
    - no known high-severity regression returns
    - high-risk task completion is not below baseline
  review_required_when:
    - any evaluator disagrees with the domain owner on a critical case
    - cost rises more than 20 percent
    - median latency exceeds the service budget
```

Avoid an undifferentiated gate such as “overall score must exceed 0.85.” It encourages gaming and can permit an unacceptable regression in a critical slice.

### Regression tests should have a story

Each permanent case should answer:

- What happened?
- Why did it matter?
- What behavior now prevents recurrence?
- Who owns its continued relevance?

Cases without context become mysterious artifacts that teams either ignore or delete. A short note keeps the suite meaningful.

### Keep a change log

Store experiment metadata next to results:

| Date       | Change                | Hypothesis               | Result                                        | Decision |
| ---------- | --------------------- | ------------------------ | --------------------------------------------- | -------- |
| 2026-09-08 | New retrieval ranking | Better evidence coverage | Improved high-risk recall; small latency cost | Ship     |
| 2026-09-15 | Shorter system prompt | Lower cost               | No material quality change                    | Ship     |
| 2026-09-22 | New model             | Better tool reliability  | More invalid arguments                        | Reject   |

This builds institutional memory and prevents teams from re-running old experiments without realizing it.

---

## 12. Make production evaluation a learning loop

Offline evaluations protect against known behavior. Production teaches you what you did not know to test.

### Sample production deliberately

Do not review only user complaints. Sample across:

- random traffic;
- high-value users or workflows;
- long or costly traces;
- multiple retries;
- sessions with escalations;
- outputs flagged by automated checks;
- new features;
- changed models, prompts, or retrieval sources;
- low-confidence routing decisions.

Random sampling protects against blind spots. Targeted sampling finds expensive or risky problems efficiently. Use both.

### Monitor leading indicators

Production metrics should help locate examples worth reviewing, not replace review. Useful indicators include:

- tool-call error rate;
- retry count;
- step count;
- context length;
- retrieval-empty rate;
- structured-output repair rate;
- policy refusal rate;
- escalation rate;
- user abandonment after response;
- cost per completed task;
- latency by workflow;
- judge score drift;
- disagreement between human and automated evaluation.

An alert should point toward a set of traces, not merely announce that a chart moved.

### Watch for distribution shift

Quality can change even when code does not:

- users begin asking for a new kind of task;
- a policy document changes;
- a downstream tool changes behavior;
- a provider updates a model;
- a new locale or customer segment arrives;
- the product UI changes the way people phrase requests.

Compare current traffic to the evaluation dataset. When the distribution changes, update the dataset and revisit the rubric.

### Turn incidents into assets

After an incident:

1. Preserve the trace and relevant state.
2. Identify the first consequential failure.
3. Add or refine a failure category.
4. Add a regression case.
5. Decide the best control: product change, deterministic guardrail, evaluator, tool redesign, or human handoff.
6. Verify the fix against neighboring cases, not only the incident.

An incident that produces no durable learning is likely to recur.

### Build a data flywheel carefully

The production loop should look like:

```text
Production traces
    → privacy-safe sampling
    → human review and taxonomy
    → curated cases and labels
    → improved evaluators and tests
    → safer changes
    → better production behavior
```

Do not indiscriminately train or tune on all production traffic. Curate examples, protect sensitive data, and preserve holdouts.

---

## 13. Build the team and operating model

Evaluation is cross-functional. Engineering can build the harness, but it cannot independently define what is useful, safe, or acceptable for a domain.

### Roles

| Role                           | Core responsibility                                          |
| ------------------------------ | ------------------------------------------------------------ |
| Product owner                  | Defines user job, business outcomes, and trade-offs          |
| Domain owner                   | Makes quality judgments and resolves ambiguous labels        |
| AI engineer                    | Implements application changes and evaluation harnesses      |
| Data or platform engineer      | Builds trace capture, sampling, storage, and access controls |
| Security or compliance partner | Defines high-risk boundaries and review requirements         |
| Operations or support partner  | Brings real failure reports and workflow knowledge           |

One person can hold several roles on a small team. The responsibilities still need clear ownership.

### A practical cadence

**Every change**

- Run deterministic checks and the targeted regression slice.
- Compare candidate and baseline on high-risk cases.
- Review meaningful disagreements.

**Weekly**

- Review a fresh set of traces.
- Add representative failures to the dataset.
- Check operational drift and alert examples.

**Monthly**

- Revisit the failure taxonomy.
- Retire obsolete cases and label stale data.
- Audit judge agreement with humans.
- Review cost, latency, and completion trade-offs.

**After incidents**

- Run the incident learning loop immediately.

### Make quality visible

Quality reports should tell a story, not dump a dashboard:

- the top observed failure modes;
- their impact and frequency;
- the traces that illustrate them;
- what changed since the last review;
- what experiment is next;
- what decision or risk requires attention.

This communicates progress more effectively than a generic “quality score.”

### Version everything that can change behavior

At minimum, version:

- prompts and policies;
- model and inference settings;
- tool definitions;
- retrieval corpus and index;
- evaluator prompts and code;
- dataset cases and labels;
- rubrics;
- application code.

Evaluation without versioning produces anecdotes. Versioning turns it into engineering.

---

## 14. Avoid the common traps

### Trap: choosing a model from public leaderboard numbers

Benchmarks measure a particular task distribution, harness, scoring rule, and set of assumptions. They can reveal useful capability signals, but they cannot certify fit for your users, tools, policies, or workflow.

Use public benchmarks to form hypotheses. Use your own evaluation set to decide.

### Trap: building infrastructure before understanding failures

Complex dashboards and orchestration systems can make a team feel productive while delaying the essential work: looking at outputs. Begin with a small review surface. Automate only when the failure mode is understood and recurs often enough to justify the effort.

### Trap: one giant quality score

An aggregate score can hide a severe safety regression behind improvements in low-risk style cases. Report dimensions and slices separately. Use an explicit decision policy to combine them.

### Trap: a judge that evaluates everything

Large judging prompts are difficult to calibrate and hard to debug. Build narrow judges that correspond to a decision or a known failure mode.

### Trap: trusting synthetic data too much

Synthetic cases improve coverage, but models tend to generate familiar, tidy examples. Keep production examples, user language, and messy edge cases in the suite.

### Trap: labeling with a vague scale

“Give quality a score from one to five” sounds flexible but creates inconsistency. Define binary pass/fail criteria for release decisions. Use notes and tags to capture nuance.

### Trap: measuring the answer but not the system

For retrieval and agents, final text hides the real source of error. Capture retrieval, routing, tool selection, arguments, tool results, state changes, and verification.

### Trap: allowing the model to enforce critical boundaries alone

Prompts are not an access-control system. Put authorization, data boundaries, rate limits, approvals, and irreversible-action controls in deterministic infrastructure.

### Trap: testing only the happy path

The interesting cases are missing data, ambiguous requests, conflicting information, partial failures, stale state, adversarial input, and changes at integration boundaries.

### Trap: optimizing the benchmark until it no longer measures reality

Protect holdouts, sample production, rotate challenge sets, and periodically ask whether the dataset still represents the product.

### Trap: ignoring cost and latency

An agent that succeeds but takes too long, consumes an unlimited budget, or makes excessive tool calls may not be viable. Measure efficiency alongside outcome quality.

### Trap: treating a passing test as proof of user value

Tests demonstrate that a specified behavior occurred. They do not prove that the specification was right, the user experience is clear, or the system will handle new conditions. Keep human review and real outcome metrics in the loop.

---

## 15. A ninety-day rollout plan

This plan assumes an existing AI feature with some traffic. Adapt the pace to risk and team size.

### Days 1–15: make behavior visible

1. Write an evaluation contract for one high-value workflow.
2. Instrument traces with configuration versions, retrieval, tool calls, and final outcomes.
3. Create a spreadsheet or notebook with twenty to fifty representative cases.
4. Run the current system and manually review every result.
5. Name the first failure taxonomy.
6. Identify one critical ship blocker and one high-frequency user pain point.

**Deliverable:** a small reviewable dataset, a trace view, and a prioritized failure list.

### Days 16–30: build the first durable checks

1. Turn critical deterministic requirements into tests.
2. Add incident and high-risk cases to a regression suite.
3. Define a binary rubric with examples for the most important fuzzy criterion.
4. Establish baseline quality, latency, and cost by slice.
5. Set a lightweight review cadence.

**Deliverable:** a repeatable baseline run and a release checklist for the workflow.

### Days 31–45: add a calibrated judge

1. Collect a balanced set of human-labeled passes and failures.
2. Build one narrow model-based judge.
3. Compare its decisions to the domain owner’s labels.
4. Inspect and resolve disagreement patterns.
5. Use the judge as a screening signal, not the sole gate.

**Deliverable:** one calibrated evaluator with documented limits.

### Days 46–60: evaluate the whole architecture

1. Add retrieval-level, tool-level, or state-transition checks as appropriate.
2. Create adversarial and partial-failure cases.
3. Add a holdout slice.
4. Record experiments and decisions in a change log.
5. Alert on operational indicators that point to reviewable traces.

**Deliverable:** layered evaluation across the important components, not only final prose.

### Days 61–90: operationalize learning

1. Formalize weekly trace review and monthly taxonomy review.
2. Turn new incidents into regression cases.
3. Review dataset freshness and production distribution shift.
4. Clarify owners for quality, instrumentation, and policy.
5. Expand to the next workflow only after the first loop is useful and stable.

**Deliverable:** a working learning system that informs product and engineering decisions.

---

## 16. Appendix: templates

### A. Case record

```yaml
case_id: support_refund_017
title: Duplicate charge with incomplete account state
source: production | incident | expert | synthetic | adversarial
slice:
  - billing
  - incomplete_context
  - medium_risk

input:
  user_message: "I think I was charged twice. Can you fix it?"
  conversation_history: []
  application_state:
    authenticated: true
    visible_orders: 1

expected_behavior:
  must:
    - Explain the appropriate next step for confirming a duplicate charge
    - Avoid claiming that a refund was issued
    - Avoid requesting sensitive payment information
  must_not:
    - Invent a second transaction
    - Invoke a refund tool without sufficient evidence

labels:
  task_completion: pass
  groundedness: pass
  policy: pass

review:
  reviewer: domain_owner
  notes: "A verification path is acceptable; an immediate refund is not."

metadata:
  created_at: 2026-09-08
  last_reviewed_at: 2026-09-08
  risk: medium
```

### B. Failure report

```markdown
## Failure: Agent claimed a change completed without checking state

Severity: high
First observed: 2026-09-08
Owner: workflow team

### What happened

The agent submitted an update request, received a timeout, and told the user the change was complete.

### First consequential failure

The agent treated a timeout as confirmation instead of verifying final state.

### Taxonomy

- Tool-result misuse
- Premature success

### Fix

- Require an explicit state-read after update attempts.
- Block completion language unless the state-read succeeds.
- Add deterministic handling for timeout responses.

### Regression cases

- `agent_update_011`
- `agent_update_012`
```

### C. Human labeling rubric

```markdown
Criterion: Grounded response

Pass:

- Every material factual statement is supported by the supplied evidence.
- When evidence is incomplete, the response states the limitation or takes the
  approved escalation path.

Fail:

- The response invents a status, action, policy, number, date, or capability.
- The response presents an unsupported assumption as fact.

Instructions:

- Judge only this criterion.
- Ignore tone, verbosity, and minor wording issues.
- Mark fail if any unsupported material claim could change the user’s decision.
- Add a one-sentence critique quoting or describing the unsupported claim.
```

### D. Experiment report

```markdown
# Experiment: Retrieval re-ranking for policy questions

## Decision

Whether to ship candidate configuration `policy-rerank-v2`.

## Hypothesis

The candidate increases evidence coverage for policy questions without causing
more unsupported answers or unacceptable latency.

## Dataset

- 120 representative policy cases
- 40 ambiguous boundary cases
- 25 historical regressions
- 20 protected holdout cases

## Results

| Metric                   | Baseline | Candidate | Decision |
| ------------------------ | -------: | --------: | -------- |
| Groundedness             |  142/165 |   153/165 | Improve  |
| Critical policy failures |        0 |         0 | Pass     |
| Unnecessary escalations  |   19/165 |    12/165 | Improve  |
| Median latency           |     1.7s |      2.0s | Accept   |

## Review notes

The candidate improved ambiguous-policy handling. Two new failures involved
stale documents and are addressed by corpus freshness checks.

## Outcome

Ship with document-age monitoring and add the two failures to regression.
```

### E. Evaluation-system checklist

```markdown
## Foundation

- [ ] A product decision is attached to each important evaluation.
- [ ] Success criteria and ship blockers are written down.
- [ ] A simpler baseline has been considered.

## Data

- [ ] The dataset contains production, incident, expert, and challenge cases.
- [ ] Results are reported by meaningful slices.
- [ ] High-risk failures are represented.
- [ ] Holdout cases are protected.

## Measurement

- [ ] Deterministic properties use deterministic checks.
- [ ] Human rubrics are binary and concrete.
- [ ] Each model-based judge has one narrow job.
- [ ] Judges are calibrated against human labels.

## Architecture

- [ ] Traces include configurations, retrieval, tools, and relevant state.
- [ ] Agent tool selection and arguments are evaluated separately.
- [ ] Critical authorization and policy controls are deterministic.

## Operations

- [ ] Significant changes run against a baseline.
- [ ] Production traces are sampled and reviewed.
- [ ] Incidents become regression cases.
- [ ] Dataset, prompts, models, tools, and evaluators are versioned.
```

---

## Closing principle

The most durable advantage in AI application development is not a particular prompt, framework, or model. It is the ability to observe behavior, make quality judgments, run controlled changes, and retain what has been learned.

Build that loop early. Keep it close to real users. Make every important failure teach the system something permanent.
