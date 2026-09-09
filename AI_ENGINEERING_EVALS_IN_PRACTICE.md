# AI Engineering Evals in Practice

## A Hands-On Guide to Building, Debugging, and Shipping Reliable AI Systems

AI evaluation is the quality gate equivalent for AI engineers as testing is for Software Engineers.

It should tell you whether to merge a prompt change, where a retrieval pipeline is failing, why an agent chose the wrong tool, which production traces deserve review, and whether a new model is worth its added cost. If an eval system cannot help answer those questions, you are operating blindly and you have no insights if your system is improving or regressing.

This book builds that capability from the ground up.

The goal for this book is practicallity. We will define test cases, implement an eval runner, add deterministic checks, calibrate model-based judges, compare experiments, create release gates, inspect retrieval and tool trajectories, and close the loop with production data. The examples use Python and plain files, but the architecture applies in any language or evaluation platform.

The companion book, *Building Reliable AI Applications*, explains the ideas and operating model behind evaluation. This book starts one step later:

> You understand why evals matter. Now you need to build them.

---

## Contents

1. [The system we will build](#1-the-system-we-will-build)
2. [Turn a product requirement into an eval contract](#2-turn-a-product-requirement-into-an-eval-contract)
3. [Create the first dataset](#3-create-the-first-dataset)
4. [Build a reproducible eval harness](#4-build-a-reproducible-eval-harness)
5. [Add deterministic evaluators first](#5-add-deterministic-evaluators-first)
6. [Review failures and design a taxonomy](#6-review-failures-and-design-a-taxonomy)
7. [Build and calibrate an AI judge](#7-build-and-calibrate-an-ai-judge)
8. [Evaluate retrieval systems](#8-evaluate-retrieval-systems)
9. [Evaluate tool-using agents](#9-evaluate-tool-using-agents)
10. [Compare candidates without lying to yourself](#10-compare-candidates-without-lying-to-yourself)
11. [Turn evals into CI release gates](#11-turn-evals-into-ci-release-gates)
12. [Connect offline evals to production](#12-connect-offline-evals-to-production)
13. [Debug common eval failures](#13-debug-common-eval-failures)
14. [A four-week implementation plan](#14-a-four-week-implementation-plan)
15. [Appendix: reference implementation](#15-appendix-reference-implementation)
16. [Appendix: working templates](#16-appendix-working-templates)

---

## 1. The system we will build

We will use a running example throughout the book: a support assistant for an online service.

The assistant can:

- answer questions from an approved policy knowledge base;
- inspect account state through read-only tools;
- request a limited set of account changes;
- escalate unsupported or high-risk requests;
- explain what it did without claiming actions that did not happen.

This example is intentionally ordinary. It contains the same engineering problems found in internal chat bots, document assistants, workflow automations, coding agents, and research tools:

- open-ended language;
- changing prompts and models;
- retrieved context;
- structured outputs;
- tool calls;
- policy boundaries;
- probabilistic behavior;
- cost and latency constraints.

### 1.1 What “done” looks like

By the end of the book, the project will have this shape:

```text
support-assistant/
├── app/
│   ├── assistant.py
│   ├── retrieval.py
│   └── tools.py
├── evals/
│   ├── cases/
│   │   ├── common.jsonl
│   │   ├── regression.jsonl
│   │   ├── high_risk.jsonl
│   │   └── holdout.jsonl
│   ├── rubrics/
│   │   ├── groundedness.md
│   │   └── task_completion.md
│   ├── evaluators/
│   │   ├── deterministic.py
│   │   ├── judge.py
│   │   └── retrieval.py
│   ├── run.py
│   ├── compare.py
│   └── gate.py
├── artifacts/
│   └── .gitkeep
└── pyproject.toml
```

The data flow will be:

```text
Cases + configuration
        ↓
Application adapter
        ↓
Output + trace
        ↓
Deterministic checks + judges + outcome checks
        ↓
Per-case records
        ↓
Slice summary + candidate comparison
        ↓
Release decision
```

The harness will deliberately remain separate from the application. It calls the application through a small interface and records everything needed to reproduce a result.

### 1.2 Start with a thin vertical slice

Do not begin with a platform. Begin with one decision:

> Should we replace the current support prompt with a candidate prompt?

To answer it, we need only:

1. a small set of cases;
2. a way to run both configurations;
3. a few trustworthy checks;
4. a result format;
5. an explicit shipping rule.

That is enough to produce value. Storage services, dashboards, distributed workers, annotation queues, and hosted evaluation products can come later.

### 1.3 The core engineering rules

The implementation in this book follows six rules:

1. **Save raw evidence.** Summaries are not enough for debugging.
2. **Version behavior-changing inputs.** A score without a configuration is an anecdote.
3. **Prefer narrow evaluators.** One evaluator should answer one question.
4. **Keep critical checks deterministic.** Authorization and schema validity are not matters of opinion.
5. **Report by slice.** Aggregate scores hide important regressions.
6. **Make every failure actionable.** A failed eval should point toward a component, owner, or next inspection.

### Lab: create the workspace

Create the directories shown above. Add `artifacts/` to `.gitignore`, but keep datasets, rubrics, evaluator code, and experiment definitions under version control.

Your first commit should contain no sophisticated evaluation logic. Its purpose is to establish a place where cases, results, and decisions can live together.

**Done when:** another engineer can identify where to add a case, where to add an evaluator, and where a run will write its results.

---

## 2. Turn a product requirement into an eval contract

Engineers often receive requirements such as:

- “The assistant should be accurate.”
- “It must not hallucinate.”
- “Make the agent more reliable.”
- “The answer should be helpful.”

These statements express intent, but they cannot yet be tested.

An eval contract turns product intent into observable behavior. It defines the workflow, the decision, the required behaviors, the forbidden behaviors, and the acceptable trade-offs.

### 2.1 Write the contract before the cases

For the support assistant:

```yaml
feature: billing-support-assistant
workflow: answer duplicate-charge questions
decision: approve or reject prompt candidate billing-v8

inputs:
  - user message
  - authentication state
  - visible transactions
  - approved billing policy documents

required_behavior:
  - distinguish a duplicate charge from a pending authorization
  - use only visible account state and approved policy
  - give the next valid action
  - state uncertainty when evidence is incomplete

forbidden_behavior:
  - invent a transaction, refund, status, or deadline
  - request full payment-card details
  - invoke a refund without the required evidence
  - claim completion when a tool call failed or timed out

quality_goals:
  - grounded answers
  - correct escalation
  - concise and actionable instructions

operational_limits:
  p95_latency_ms: 5000
  max_model_calls: 4
  max_estimated_cost_usd: 0.08
```

The contract does not need to use YAML. It needs to be reviewable. A product owner should understand it, a domain expert should be able to challenge it, and an engineer should be able to translate it into checks.

### 2.2 Convert requirements into observables

Every requirement needs evidence.

| Requirement | Observable evidence | Evaluator |
| --- | --- | --- |
| Output follows API contract | Parsed object and schema errors | Deterministic |
| No invented transaction ID | IDs in output compared with supplied state | Deterministic |
| Claims are supported | Answer, claims, and approved evidence | Model judge or human |
| Correct next action | User state, policy, and answer | Rubric judge or human |
| No action without authorization | Tool trace and auth state | Deterministic |
| Does not claim failed action succeeded | Tool result and final answer | Deterministic plus judge |
| Response is fast enough | End-to-end duration | Deterministic |

If you cannot name the evidence, instrumentation is missing or the requirement is too vague.

### 2.3 Separate invariants from preferences

An invariant must always hold:

- no cross-account data exposure;
- no unapproved write action;
- valid structured output;
- no success claim after a failed tool call.

A preference can be traded against another goal:

- shorter answers;
- warmer tone;
- fewer clarifying questions;
- lower cost;
- faster response.

This distinction controls how the eval will be used. Invariants become hard gates. Preferences become comparison metrics.

### 2.4 Define a decision policy

Do not wait until after the run to decide what the results mean.

```yaml
ship_candidate_when:
  - zero critical invariant failures
  - no regression on historical high-severity cases
  - task completion is not lower than baseline on any high-risk slice
  - groundedness improves or remains within an agreed tolerance
  - p95 latency remains under 5000 ms

manual_review_when:
  - candidate and baseline trade different failure types
  - judge confidence is low on more than 5 percent of cases
  - fewer than 30 cases exist in a decision-critical slice
```

Precommitting to a policy makes it harder to rationalize a favorite candidate after seeing the results.

### Lab: write one eval contract

Choose one narrow workflow in your application. Write:

- the decision this eval supports;
- three required behaviors;
- three forbidden behaviors;
- two operational limits;
- the evidence needed to evaluate each item;
- the shipping rule.

**Done when:** each statement can be tied to a trace field or a human-reviewable artifact.

---

## 3. Create the first dataset

An eval case is a controlled description of a situation. It contains enough input and state to run the application, plus enough expected behavior to judge the result.

The first dataset should be small and inspectable. Twenty carefully reviewed cases are more useful than two thousand generated cases nobody understands.

### 3.1 Use behavior specifications, not golden prose

For open-ended answers, avoid requiring one exact string:

```json
{
  "case_id": "billing_pending_001",
  "title": "Pending authorization mistaken for duplicate charge",
  "slice": ["billing", "pending_authorization", "common"],
  "risk": "medium",
  "input": {
    "message": "Why did you charge me twice?",
    "authenticated": true,
    "transactions": [
      {"id": "tx_104", "kind": "capture", "amount": 42.00, "status": "settled"},
      {"id": "auth_992", "kind": "authorization", "amount": 42.00, "status": "pending"}
    ]
  },
  "expected": {
    "must": [
      "Explain that one item is a pending authorization",
      "Give the policy-approved next step"
    ],
    "must_not": [
      "Claim that two settled charges exist",
      "Claim that a refund was issued",
      "Request full card details"
    ],
    "allowed_tools": ["get_transaction_details"],
    "forbidden_tools": ["issue_refund"]
  }
}
```

The case specifies behavior while allowing many acceptable phrasings.

### 3.2 Keep cases self-contained

A case should not silently depend on changing production state. If the application uses documents or tools, provide fixtures or stable identifiers:

```json
{
  "case_id": "refund_policy_004",
  "slice": ["billing", "policy_boundary", "high_risk"],
  "risk": "high",
  "input": {
    "message": "Refund this six-month-old purchase.",
    "authenticated": true
  },
  "fixtures": {
    "documents": [
      {
        "doc_id": "refund-policy-v3",
        "text": "Self-service refunds are available for 30 days after settlement."
      }
    ],
    "tool_responses": {
      "get_purchase": {
        "purchase_id": "purchase_44",
        "age_days": 183,
        "status": "settled"
      }
    }
  },
  "expected": {
    "must": [
      "Explain that the purchase is outside the self-service refund window",
      "Offer the approved escalation route"
    ],
    "must_not": [
      "Invoke issue_refund",
      "Promise that an exception will be granted"
    ]
  }
}
```

Self-contained fixtures improve reproducibility and make failures easier to understand.

### 3.3 Give every case a reason to exist

Store provenance:

```json
{
  "source": {
    "type": "incident",
    "reference": "INC-1842",
    "note": "Assistant treated a timeout as proof that a refund completed."
  }
}
```

Useful source types include:

- `production`;
- `incident`;
- `domain_expert`;
- `synthetic`;
- `adversarial`;
- `new_feature`;
- `historical_regression`.

Provenance helps reviewers decide whether a case is still relevant. It also prevents synthetic cases from silently dominating the suite.

### 3.4 Design slices before calculating scores

Slices are meaningful groups of cases:

- workflow: billing, account access, delivery;
- risk: low, medium, high, critical;
- context state: complete, missing, conflicting;
- behavior: answer, clarify, refuse, escalate, act;
- language or locale;
- source: production, incident, synthetic;
- architecture: retrieval-only, read tool, write tool.

A case can belong to several slices. Keep the labels stable enough for comparisons.

An overall result of 92% can mean:

```text
Common questions:       98/100
High-risk boundaries:    4/10
Historical incidents:   10/10
```

The aggregate looks good. The product is not ready.

### 3.5 Add synthetic data only for a named gap

Synthetic generation is useful when you can say what coverage is missing:

```text
Need: conflicting account state for refund requests

Dimensions:
- authenticated: true / false
- purchase state: pending / settled / reversed
- refund window: inside / outside / unknown
- user wording: direct / ambiguous / emotionally charged
- tool response: success / timeout / malformed / permission denied
```

Generate combinations, review them, and keep only plausible cases. Never treat generated expected behavior as ground truth without domain review.

### 3.6 Split the dataset by purpose

Use at least three groups:

```text
development   Frequently inspected while building a change
regression    Known behaviors that should never return
holdout       Protected cases used to detect overfitting
```

Add a fresh production sample when the application is live.

Do not repeatedly tune against the holdout. If engineers know every case by memory, it is no longer functioning as a holdout.

### Lab: build the first 20 cases

Create:

- eight common cases;
- four missing or ambiguous context cases;
- four high-risk boundary cases;
- two historical failures;
- two integration-failure cases.

Review every case with the product or domain owner.

**Done when:** every case has a stable ID, one or more slices, a risk level, fixtures, expected behavior, and provenance.

---

## 4. Build a reproducible eval harness

The eval harness has one job: run a configuration on a set of cases and preserve the evidence.

It should not contain product behavior. It should not silently rewrite inputs. It should not hide failed runs. It should make comparisons fair.

### 4.1 Define the application boundary

Use a small protocol:

```python
from dataclasses import dataclass, field
from typing import Any, Protocol


@dataclass
class ToolCall:
    name: str
    arguments: dict[str, Any]
    result: dict[str, Any] | None = None
    error: str | None = None
    duration_ms: int | None = None


@dataclass
class AppResult:
    final_text: str
    structured_output: dict[str, Any] | None = None
    retrieved_documents: list[dict[str, Any]] = field(default_factory=list)
    tool_calls: list[ToolCall] = field(default_factory=list)
    model_calls: list[dict[str, Any]] = field(default_factory=list)
    final_state: dict[str, Any] = field(default_factory=dict)
    latency_ms: int = 0
    estimated_cost_usd: float = 0.0


class Application(Protocol):
    def run(self, case: dict[str, Any], config: dict[str, Any]) -> AppResult:
        ...
```

The application adapter can call a local function, an HTTP service, or a deployed endpoint. The harness should not care.

### 4.2 Make configuration explicit

```json
{
  "config_id": "billing-v8-model-b",
  "application_revision": "git:7ea3f21",
  "prompt_version": "billing-v8",
  "model": "provider/model-version",
  "temperature": 0,
  "retrieval_index": "support-policy-2026-09-01",
  "tool_policy": "support-tools-v4",
  "max_steps": 6
}
```

Avoid mutable names such as `latest`, `production`, or `current`. Resolve them to immutable versions before the run starts.

### 4.3 Define the result record

Each case should produce a record even when the application crashes:

```python
from dataclasses import asdict
from datetime import datetime, timezone
import traceback


def execute_case(application, case, config):
    started = datetime.now(timezone.utc)

    try:
        app_result = application.run(case, config)
        return {
            "case_id": case["case_id"],
            "config_id": config["config_id"],
            "started_at": started.isoformat(),
            "status": "completed",
            "result": asdict(app_result),
            "run_error": None,
        }
    except Exception as exc:
        return {
            "case_id": case["case_id"],
            "config_id": config["config_id"],
            "started_at": started.isoformat(),
            "status": "error",
            "result": None,
            "run_error": {
                "type": type(exc).__name__,
                "message": str(exc),
                "traceback": traceback.format_exc(),
            },
        }
```

Infrastructure failures are results, not missing data. If they disappear from the report, the system can look healthier precisely when it is least reliable.

### 4.4 Store immutable run artifacts

A run directory should contain:

```text
artifacts/2026-09-08T141200Z_billing-v8-model-b/
├── manifest.json
├── cases.jsonl
├── results.jsonl
├── evaluations.jsonl
├── summary.json
└── report.md
```

The manifest records:

```json
{
  "run_id": "2026-09-08T141200Z_billing-v8-model-b",
  "dataset_hash": "sha256:...",
  "config": {},
  "evaluator_versions": {},
  "host": {},
  "started_at": "2026-09-08T14:12:00Z",
  "completed_at": null
}
```

Hash the exact case content, not only the filename. Otherwise two runs can claim to use `regression.jsonl` while using different cases.

### 4.5 Control nondeterminism

For comparisons:

- run baseline and candidate on the same case order;
- use the same fixtures;
- pin model versions where possible;
- record all generation settings;
- cache stable judge calls when appropriate;
- repeat cases when outcome variance matters;
- distinguish app variance from evaluator variance.

Temperature zero does not guarantee deterministic behavior. Providers, routing layers, retrieval order, tool services, and concurrency can all introduce variation.

For unstable tasks, run each case several times:

```text
case success@1   Did one run succeed?
case success@3   Did all three runs succeed?
case pass rate   What fraction of repeated runs passed?
```

For high-risk invariants, “passed once” is usually the wrong standard.

### 4.6 Add a command-line interface

A useful first interface is:

```bash
python -m evals.run \
  --dataset evals/cases/regression.jsonl \
  --config configs/billing-v8.json \
  --output artifacts/
```

Later:

```bash
python -m evals.compare \
  --baseline artifacts/run-baseline \
  --candidate artifacts/run-candidate
```

The CLI makes local use and CI use follow the same path.

### Lab: run and save one case

Implement enough of the harness to:

1. load one JSONL case;
2. load one configuration;
3. call an application adapter;
4. save the raw result;
5. preserve exceptions as records.

**Done when:** deleting the application logs does not prevent you from reconstructing what the eval saw.

---

## 5. Add deterministic evaluators first

Deterministic evaluators are the foundation of an eval suite. They are cheap, fast, explainable, and stable. Use them for every property that ordinary code can establish.

### 5.1 Standardize evaluator output

Every evaluator should return the same shape:

```python
from dataclasses import dataclass, field
from typing import Literal


Verdict = Literal["pass", "fail", "error", "not_applicable"]


@dataclass
class EvalResult:
    evaluator: str
    version: str
    verdict: Verdict
    score: float | None = None
    reason: str = ""
    evidence: dict = field(default_factory=dict)
```

Keep `error` separate from `fail`.

- `fail` means the application violated a criterion.
- `error` means the evaluator could not make the measurement.

Treating evaluator errors as passes is dangerous. Treating all of them as product failures creates noisy gates. Report them separately and decide policy explicitly.

### 5.2 Validate structured output

Suppose the assistant returns:

```json
{
  "response": "The second item is a pending authorization.",
  "action": "explain",
  "escalate": false,
  "cited_document_ids": ["billing-auth-v2"]
}
```

Check syntax and shape in code:

```python
ALLOWED_ACTIONS = {"explain", "clarify", "escalate", "act"}


def evaluate_output_contract(record) -> EvalResult:
    output = record["result"].get("structured_output")

    if not isinstance(output, dict):
        return EvalResult(
            evaluator="output_contract",
            version="1",
            verdict="fail",
            reason="structured_output is not an object",
        )

    required = {"response", "action", "escalate", "cited_document_ids"}
    missing = sorted(required - output.keys())
    if missing:
        return EvalResult(
            evaluator="output_contract",
            version="1",
            verdict="fail",
            reason=f"missing required fields: {missing}",
            evidence={"missing": missing},
        )

    if output["action"] not in ALLOWED_ACTIONS:
        return EvalResult(
            evaluator="output_contract",
            version="1",
            verdict="fail",
            reason="action is outside the allowed enum",
            evidence={"action": output["action"]},
        )

    if not isinstance(output["escalate"], bool):
        return EvalResult(
            evaluator="output_contract",
            version="1",
            verdict="fail",
            reason="escalate must be boolean",
        )

    return EvalResult("output_contract", "1", "pass")
```

In a production codebase, use the same schema library used by the application. The principle is more important than the library.

### 5.3 Check tool permissions

```python
def evaluate_allowed_tools(case, record) -> EvalResult:
    expected = case.get("expected", {})
    allowed = set(expected.get("allowed_tools", []))
    forbidden = set(expected.get("forbidden_tools", []))
    called = [call["name"] for call in record["result"]["tool_calls"]]

    forbidden_calls = sorted(forbidden.intersection(called))
    unexpected_calls = sorted(set(called) - allowed) if allowed else []

    if forbidden_calls:
        return EvalResult(
            "allowed_tools",
            "1",
            "fail",
            reason=f"called forbidden tools: {forbidden_calls}",
            evidence={"called": called},
        )

    if unexpected_calls:
        return EvalResult(
            "allowed_tools",
            "1",
            "fail",
            reason=f"called tools not allowed by the case: {unexpected_calls}",
            evidence={"called": called},
        )

    return EvalResult("allowed_tools", "1", "pass", evidence={"called": called})
```

In real systems, permission enforcement belongs in the application infrastructure. The eval verifies that the boundary works; it is not the boundary itself.

### 5.4 Detect invented identifiers

If account IDs, transaction IDs, document IDs, or ticket IDs appear in an answer, compare them with the supplied state and tool results.

```python
import re


TRANSACTION_ID = re.compile(r"\b(?:tx|auth|refund)_[a-zA-Z0-9]+\b")


def evaluate_known_transaction_ids(case, record) -> EvalResult:
    text = record["result"]["final_text"]
    mentioned = set(TRANSACTION_ID.findall(text))

    known = {
        item["id"]
        for item in case["input"].get("transactions", [])
        if "id" in item
    }

    for call in record["result"]["tool_calls"]:
        if call.get("result"):
            known.update(collect_ids(call["result"]))

    invented = sorted(mentioned - known)
    if invented:
        return EvalResult(
            "known_transaction_ids",
            "1",
            "fail",
            reason="answer contains transaction IDs absent from supplied evidence",
            evidence={"invented": invented, "known": sorted(known)},
        )

    return EvalResult("known_transaction_ids", "1", "pass")
```

This check does not solve groundedness in general. It catches a concrete, costly class of hallucination with high precision.

### 5.5 Verify actions against final state

A successful tool response is not always the same as a successful user outcome. Verify the state that matters:

```python
def evaluate_requested_state(case, record) -> EvalResult:
    expected_state = case.get("expected", {}).get("final_state")
    if expected_state is None:
        return EvalResult(
            "requested_state", "1", "not_applicable"
        )

    actual_state = record["result"]["final_state"]
    mismatches = {
        key: {"expected": value, "actual": actual_state.get(key)}
        for key, value in expected_state.items()
        if actual_state.get(key) != value
    }

    if mismatches:
        return EvalResult(
            "requested_state",
            "1",
            "fail",
            reason="final state does not match expected state",
            evidence={"mismatches": mismatches},
        )

    return EvalResult("requested_state", "1", "pass")
```

### 5.6 Measure budgets

```python
def evaluate_budgets(case, record, config) -> list[EvalResult]:
    result = record["result"]
    checks = []

    checks.append(EvalResult(
        evaluator="latency_budget",
        version="1",
        verdict="pass" if result["latency_ms"] <= config["max_latency_ms"] else "fail",
        score=result["latency_ms"],
        reason=f'{result["latency_ms"]} ms',
    ))

    checks.append(EvalResult(
        evaluator="model_call_budget",
        version="1",
        verdict="pass" if len(result["model_calls"]) <= config["max_model_calls"] else "fail",
        score=len(result["model_calls"]),
        reason=f'{len(result["model_calls"])} model calls',
    ))

    checks.append(EvalResult(
        evaluator="cost_budget",
        version="1",
        verdict="pass"
        if result["estimated_cost_usd"] <= config["max_cost_usd"]
        else "fail",
        score=result["estimated_cost_usd"],
        reason=f'${result["estimated_cost_usd"]:.4f}',
    ))

    return checks
```

Budget checks are often better as summary percentiles than per-case hard limits. Use a hard limit only when one expensive trajectory is itself unacceptable.

### 5.7 Test the evaluators

Evaluator code is production code. Write unit tests using known pass and fail fixtures:

```python
def test_forbidden_tool_is_detected():
    case = {
        "expected": {
            "allowed_tools": ["get_purchase"],
            "forbidden_tools": ["issue_refund"],
        }
    }
    record = {
        "result": {
            "tool_calls": [
                {"name": "issue_refund", "arguments": {"purchase_id": "p_1"}}
            ]
        }
    }

    result = evaluate_allowed_tools(case, record)

    assert result.verdict == "fail"
    assert "issue_refund" in result.reason
```

A buggy evaluator can block good releases or approve bad ones. Both are engineering incidents.

### Lab: implement five exact checks

Choose five from:

- schema validity;
- required output fields;
- forbidden tool use;
- tool argument constraints;
- known identifier validation;
- expected final state;
- latency;
- cost;
- maximum steps;
- presence of a required citation.

**Done when:** each evaluator has a pass fixture, a fail fixture, and a stable version identifier.

---

## 6. Review failures and design a taxonomy

Automation should follow understanding. Before building a fuzzy evaluator, inspect failures manually.

### 6.1 Create a review queue

Send cases to review when:

- a deterministic check fails;
- an evaluator errors;
- baseline and candidate disagree;
- a case is high-risk;
- a run is unusually slow or expensive;
- a fresh production sample has no label;
- the system reports uncertainty;
- the case was recently added.

A review item should show the case and the full trace side by side:

```text
CASE
- user request
- state and fixtures
- required and forbidden behavior

APPLICATION
- final answer
- retrieved documents
- tool calls and results
- final state
- model and prompt version

EVALUATION
- check results
- judge rationale
- baseline output, if comparing

REVIEW
- pass / fail / needs discussion
- first consequential failure
- severity
- failure tags
- note
```

Do not ask reviewers to judge from the final answer alone when the system used retrieval or tools.

### 6.2 Label the first consequential failure

Consider this trajectory:

1. Retrieval misses the refund policy.
2. The model assumes refunds are allowed for 90 days.
3. The agent invokes the refund tool.
4. The tool denies the request.
5. The assistant tells the user there is a temporary system problem.

Possible labels include unsupported claim, wrong tool use, tool failure, and misleading answer. The earliest useful diagnosis is missing evidence or retrieval failure. Fixing that may prevent the whole cascade.

Record secondary failures only if they support a separate intervention.

### 6.3 Build the taxonomy from evidence

Start with open notes. After reviewing 30–50 traces, group recurring patterns:

```yaml
retrieval:
  missing_relevant_document:
    owner: search
    severity_default: medium
  stale_document:
    owner: knowledge-platform
    severity_default: high
  unauthorized_document:
    owner: security
    severity_default: critical

reasoning:
  unsupported_assumption:
    owner: assistant
    severity_default: medium
  ignored_user_constraint:
    owner: assistant
    severity_default: high

tools:
  wrong_tool:
    owner: workflow
    severity_default: high
  invalid_arguments:
    owner: workflow
    severity_default: high
  result_misinterpreted:
    owner: workflow
    severity_default: high
  completion_not_verified:
    owner: workflow
    severity_default: high

presentation:
  missing_next_step:
    owner: experience
    severity_default: medium
  excessive_detail:
    owner: experience
    severity_default: low
```

A useful category has:

- a clear definition;
- positive and negative examples;
- a likely owner;
- an intervention it can trigger;
- limited overlap with neighboring categories.

### 6.4 Track severity separately from category

The same category can have different impact:

- a missing citation in an internal summary may be low severity;
- an unsupported dosage claim in a clinical workflow may be critical.

Use an impact model:

```text
Severity = potential harm × likelihood of user reliance × reversibility
```

You do not need false numerical precision. A clear rubric is enough:

| Severity | Meaning |
| --- | --- |
| Critical | Could cause serious harm, unauthorized action, or sensitive-data exposure |
| High | Materially wrong outcome or action; difficult to reverse |
| Medium | User cannot complete the task or is materially misled |
| Low | Friction, style, or minor completeness issue |

### 6.5 Measure reviewer agreement

For a new rubric, double-label a sample. Record:

- raw agreement;
- disagreements by category;
- false-pass and false-fail patterns;
- cases that reveal missing rubric language.

Low agreement is not automatically a reviewer problem. It often means the product requirement is underspecified.

### Lab: review 30 traces

Review a mixed sample without inventing evaluator prompts yet.

Produce:

- a first-failure label for each failed trace;
- severity;
- a taxonomy with definitions;
- the top three failure categories by impact;
- one proposed intervention for each.

**Done when:** the team can use the taxonomy to choose engineering work, not merely describe defects.

---

## 7. Build and calibrate an AI judge

Use a model-based judge when:

- the criterion is semantic;
- deterministic checks are insufficient;
- the rubric can be stated clearly;
- enough human-labeled examples exist to test the judge;
- the decision does not rely on blind trust in the judge.

Groundedness is a good first example.

### 7.1 Define one narrow task

Bad judge request:

> Score this response for accuracy, relevance, style, completeness, safety, and usefulness.

Better judge request:

> Does the response contain a material factual claim that is unsupported by the supplied evidence?

The second task has a clearer decision boundary.

### 7.2 Require structured output

```json
{
  "verdict": "pass",
  "unsupported_claims": [],
  "reason": "All material claims are supported by the transaction state and policy text.",
  "confidence": 0.93
}
```

Validate this output deterministically. Do not parse free-form prose with fragile string rules.

### 7.3 Write the judge rubric

```markdown
# Groundedness judge, version 3

## Question

Does the assistant response contain any material factual claim that is not
supported by the supplied evidence?

## Pass

- Every material claim is directly supported by the evidence.
- The response clearly marks uncertainty when evidence is insufficient.
- The response may paraphrase evidence without copying its exact wording.

## Fail

- It invents a policy, status, action, number, date, deadline, identity, or
  capability.
- It presents an inference as confirmed fact when the evidence does not support
  that confidence.
- It claims that an action completed without a successful tool result or final
  state verification.

## Ignore

- Tone and writing style.
- Whether the response is concise.
- Harmless conversational language.

## Output

Return JSON with verdict, unsupported_claims, reason, and confidence.
```

The `Ignore` section prevents the judge from drifting into unrelated quality dimensions.

### 7.4 Build a labeled calibration set

Use human-reviewed examples:

```text
40 clear passes
40 clear failures
20 difficult boundary cases
```

Include realistic failure diversity:

- invented status;
- invented policy;
- unsupported deadline;
- overconfident inference;
- false completion claim;
- correct uncertainty;
- supported paraphrase;
- partial support where one claim goes too far.

A calibration set containing only obvious hallucinations will produce an impressive but misleading judge.

### 7.5 Measure the judge as a classifier

Create a confusion matrix:

| | Human pass | Human fail |
| --- | ---: | ---: |
| Judge pass | true pass | false pass |
| Judge fail | false fail | true fail |

Then calculate:

```text
recall on failures = true fail / (true fail + false pass)
precision on failures = true fail / (true fail + false fail)
```

For a safety-oriented gate, missing a real failure may be worse than flagging an acceptable response. For a high-volume review queue, excessive false alarms may make the system unusable. Choose the operating point based on the decision.

Always inspect disagreements. Aggregate accuracy does not tell you which failures the judge misses.

### 7.6 Implement the judge behind an interface

```python
import json


class GroundednessJudge:
    name = "groundedness"
    version = "3"

    def __init__(self, client, rubric: str):
        self.client = client
        self.rubric = rubric

    def evaluate(self, case, record) -> EvalResult:
        evidence = {
            "input": case["input"],
            "documents": record["result"]["retrieved_documents"],
            "tool_calls": record["result"]["tool_calls"],
            "final_state": record["result"]["final_state"],
        }

        payload = {
            "rubric": self.rubric,
            "evidence": evidence,
            "assistant_response": record["result"]["final_text"],
        }

        raw = self.client.generate_json(payload)
        parsed = validate_judge_output(raw)

        return EvalResult(
            evaluator=self.name,
            version=self.version,
            verdict=parsed["verdict"],
            score=parsed["confidence"],
            reason=parsed["reason"],
            evidence={
                "unsupported_claims": parsed["unsupported_claims"]
            },
        )
```

Record the judge model, prompt hash, generation settings, and rubric version alongside every result.

### 7.7 Defend against judge leakage and bias

Do not tell the judge which answer is the baseline or candidate.

For pairwise comparison:

- randomize A/B order;
- run both orderings on a calibration sample;
- allow ties;
- keep the rubric separate from candidate prompts;
- do not expose irrelevant configuration names;
- inspect whether longer answers win disproportionately.

If the application output can contain hostile instructions, place it in clearly delimited data and instruct the judge to treat it as content, not instructions. Still validate the judge empirically; prompt wording is not a security guarantee.

### 7.8 Use confidence carefully

Model-reported confidence is not automatically calibrated. It may still be useful for ranking review items, but do not treat `0.91` as a measured 91% probability without calibration.

Safer uses:

- route low-confidence judgments to humans;
- compare confidence distributions over time;
- combine with disagreement across repeated judge calls;
- identify cases near the decision boundary.

### Lab: calibrate one judge

Build one judge for groundedness, task completion, or escalation correctness.

Report:

- number of labeled examples;
- confusion matrix;
- failure recall and precision;
- disagreement categories;
- known limitations;
- approved use: screening, metric, soft gate, or hard gate.

**Done when:** the judge has a documented job and evidence that it can perform that job.

---

## 8. Evaluate retrieval systems

A retrieval-augmented system has at least two major components:

1. finding evidence;
2. using evidence.

Evaluating only the final answer makes these failures difficult to separate.

### 8.1 Add retrieval truth to cases

For cases where the relevant source is known:

```json
{
  "case_id": "policy_refund_012",
  "expected": {
    "relevant_document_ids": ["refund-policy-v3"],
    "must": ["State the 30-day self-service limit"],
    "must_not": ["Claim the limit is 90 days"]
  }
}
```

When several documents are acceptable:

```json
{
  "relevant_document_groups": [
    ["refund-policy-v3"],
    ["billing-faq-v7", "exceptions-policy-v2"]
  ]
}
```

This means either the first document or the combination in the second group can supply the required evidence.

### 8.2 Calculate retrieval metrics

Recall at `k` asks whether relevant evidence appeared in the first `k` results:

```python
def recall_at_k(retrieved_ids, relevant_ids, k):
    relevant = set(relevant_ids)
    found = relevant.intersection(retrieved_ids[:k])
    return len(found) / len(relevant) if relevant else None
```

Reciprocal rank rewards placing the first relevant item earlier:

```python
def reciprocal_rank(retrieved_ids, relevant_ids):
    relevant = set(relevant_ids)
    for rank, doc_id in enumerate(retrieved_ids, start=1):
        if doc_id in relevant:
            return 1.0 / rank
    return 0.0
```

Also track:

- empty retrieval rate;
- unauthorized document rate;
- stale document rate;
- duplicate result rate;
- context bytes or tokens;
- relevant evidence truncated before generation;
- retrieval latency.

### 8.3 Evaluate the query and filters

Capture:

```json
{
  "retrieval": {
    "original_request": "Can I refund this old purchase?",
    "generated_query": "refund eligibility purchase age policy",
    "filters": {
      "locale": "en-GB",
      "product": "consumer",
      "effective_at": "2026-09-08"
    },
    "results": []
  }
}
```

Common failures include:

- the query drops a critical constraint;
- an incorrect locale filter removes the relevant policy;
- no effective-date filter allows stale policy;
- tenancy filters are missing;
- reranking promotes a superficially similar but wrong document.

Retrieval evaluation should inspect these intermediate decisions.

### 8.4 Build a component matrix

For each case, classify:

| Retrieval | Answer | Diagnosis |
| --- | --- | --- |
| Pass | Pass | System success |
| Fail | Fail | Retrieval likely caused or contributed |
| Pass | Fail | Generation, reasoning, or instruction failure |
| Fail | Pass | Lucky answer; hidden risk |

The last row deserves attention. A correct answer without the required evidence may indicate memorization or unsupported inference. It should not be celebrated as a clean pass.

### 8.5 Test corpus changes separately

When adding or reindexing documents:

1. freeze the generator;
2. compare retrieval results;
3. inspect changed rankings;
4. run end-to-end generation;
5. evaluate both answer quality and harmful retrievals.

A corpus change can improve recall while increasing conflicting or unauthorized context.

### 8.6 Test adversarial documents

Include fixtures containing:

- instructions addressed to the model;
- obsolete policies;
- content from another tenant;
- unsupported claims in user-generated text;
- duplicated sections with conflicting dates;
- very long documents where the relevant clause appears late.

Expected behavior is not merely “ignore prompt injection.” The retrieval layer should apply access controls and provenance, while the application should distinguish trusted instructions from untrusted content.

### Lab: diagnose ten RAG failures

For ten failed answers, record:

- whether relevant evidence was retrieved;
- rank of the best evidence;
- whether it reached the generator;
- whether the answer used it;
- the earliest failed component.

**Done when:** each failure points to corpus, query, filtering, ranking, context construction, or generation.

---

## 9. Evaluate tool-using agents

Tool-using agents need two kinds of evaluation:

- **outcome evaluation:** did the requested state become true?
- **trajectory evaluation:** did the agent act safely and sensibly?

An agent can reach the right outcome through an unacceptable path. It can also take a reasonable path and fail because a dependency was unavailable. The eval should distinguish these cases.

### 9.1 Model tools as contracts

```yaml
tool: request_email_change
mode: write

preconditions:
  - user is authenticated
  - new email is syntactically valid
  - new email differs from current email
  - user has confirmed the exact address

arguments:
  account_id: string
  new_email: string
  idempotency_key: string

success_evidence:
  - tool returns request_id
  - verification read shows status pending_verification

forbidden:
  - infer account_id from another user's data
  - retry with a new idempotency key after timeout
  - claim the email changed before verification completes
```

This contract becomes:

- tool implementation validation;
- trace validation;
- eval case design;
- agent prompt content;
- review guidance.

### 9.2 Evaluate selection, arguments, and timing separately

```python
def evaluate_tool_sequence(case, record) -> list[EvalResult]:
    calls = record["result"]["tool_calls"]
    results = []

    results.append(check_tool_names(case, calls))
    results.append(check_tool_arguments(case, calls))
    results.append(check_preconditions(case, calls))
    results.append(check_duplicate_writes(calls))
    results.append(check_verification_after_write(calls))

    return results
```

“Tool use failed” is not specific enough. The fix for wrong selection differs from the fix for invalid arguments or missing verification.

### 9.3 Represent expected trajectories with constraints

Do not require one exact sequence unless the order is itself important.

```json
{
  "expected": {
    "trajectory": {
      "must_call": ["get_account", "request_email_change"],
      "must_not_call": ["set_email_directly"],
      "required_order": [
        {"before": "get_account", "after": "request_email_change"},
        {"before": "request_email_change", "after": "get_change_status"}
      ],
      "max_tool_calls": 4,
      "write_calls_require_confirmation": true
    }
  }
}
```

This allows harmless implementation variation while preserving the sequence that makes the action safe. The key is to encode required constraints, not incidental details.

### 9.4 Simulate failures

An agent eval environment should support:

- timeout before a write reaches the server;
- timeout after a write succeeds;
- permission denial;
- malformed tool response;
- stale read;
- partial batch success;
- rate limiting;
- duplicate request;
- tool not found;
- conflicting state between two reads.

The timeout-after-success case is especially important. A naive retry can duplicate an action. The agent should use an idempotency key or verify state before retrying.

### 9.5 Use a resettable sandbox

Each case should:

1. create known state;
2. run the agent;
3. inspect final state;
4. save a trace;
5. reset the environment.

Never point an offline agent eval at production write tools.

For a local simulator:

```python
class FakeAccountService:
    def __init__(self, initial_state, failures=None):
        self.state = initial_state.copy()
        self.failures = failures or {}
        self.calls = []

    def request_email_change(self, account_id, new_email, idempotency_key):
        self.calls.append({
            "tool": "request_email_change",
            "account_id": account_id,
            "new_email": new_email,
            "idempotency_key": idempotency_key,
        })

        if self.failures.get("request_email_change") == "timeout_after_write":
            self.state["pending_email"] = new_email
            raise TimeoutError("response timed out after commit")

        self.state["pending_email"] = new_email
        return {"request_id": "req_101", "status": "pending_verification"}
```

Now you can test whether the agent verifies state instead of blindly retrying.

### 9.6 Check final claims against tool evidence

Create a deterministic completion-language check for high-risk actions:

```text
If final answer contains a completion claim:
    require successful write result
    and require final state verification
Else:
    allow uncertainty, retry guidance, or escalation
```

The exact language classifier may need a narrow judge, but the evidence rule should remain explicit.

### 9.7 Evaluate efficiency without rewarding recklessness

Track:

- total steps;
- repeated reads;
- repeated model calls;
- failed tool calls;
- wall-clock time;
- cost;
- unnecessary writes.

Do not optimize step count before safety and completion. A three-step unsafe trajectory is not better than a five-step verified one.

### Lab: build an agent fault suite

Create cases for:

- wrong tool temptation;
- missing required argument;
- permission denial;
- timeout before write;
- timeout after successful write;
- stale read after write;
- duplicate user request;
- hostile instruction in tool output.

**Done when:** the report distinguishes outcome, tool selection, arguments, preconditions, recovery, and verification.

---

## 10. Compare candidates without lying to yourself

Most eval runs support a comparison:

- old prompt versus new prompt;
- model A versus model B;
- retrieval configuration A versus B;
- agent policy A versus B;
- workflow with and without a tool.

The comparison is more important than an isolated score.

### 10.1 Use paired results

Run baseline and candidate on the same cases. Then compare each pair:

```python
def paired_outcome(baseline_pass: bool, candidate_pass: bool) -> str:
    if baseline_pass and candidate_pass:
        return "both_pass"
    if not baseline_pass and not candidate_pass:
        return "both_fail"
    if not baseline_pass and candidate_pass:
        return "candidate_win"
    return "candidate_regression"
```

Count wins and regressions by slice and severity:

```text
All cases:
  candidate wins:        14
  candidate regressions:  6

High-risk:
  candidate wins:         1
  candidate regressions:  2
```

An average score may prefer the candidate. The decision policy may correctly reject it.

### 10.2 Preserve metric direction

Some metrics should increase:

- task completion;
- groundedness;
- retrieval recall.

Some should decrease:

- latency;
- cost;
- unsafe action rate;
- unnecessary escalation.

Make direction explicit in metric definitions:

```python
METRICS = {
    "task_completion": {"better": "higher"},
    "groundedness": {"better": "higher"},
    "latency_ms": {"better": "lower"},
    "estimated_cost_usd": {"better": "lower"},
}
```

This prevents report code from accidentally presenting a cost increase as an improvement.

### 10.3 Report counts, rates, and denominators

Prefer:

```text
Groundedness: 87/100 → 91/100
```

over:

```text
Groundedness improved by 4.6%.
```

The first reveals sample size and absolute failures.

For slices:

```text
High-risk groundedness: 9/10 → 10/10
French groundedness:    2/3  → 3/3  (sample too small for confidence)
```

### 10.4 Account for repeated runs

Suppose each case runs five times:

```text
Case A baseline: 5/5 pass
Case A candidate: 3/5 pass
```

Do not flatten all repetitions into unrelated rows. Keep case identity and report:

- mean pass rate;
- worst-case pass rate;
- number of perfectly stable cases;
- number of flaky cases;
- critical cases with any failure.

For critical behavior, a useful gate is:

```text
No critical case may fail in any repeated run.
```

### 10.5 Use statistical tests as support, not ceremony

For paired binary outcomes, a paired method such as McNemar’s test can help assess whether the difference is likely to be noise. For continuous paired measures such as latency, bootstrap confidence intervals over case-level differences are often understandable and robust.

Statistics do not rescue a bad dataset or evaluator. Before asking whether a result is statistically significant, ask:

- Does the dataset represent the decision?
- Are labels trustworthy?
- Are cases independent enough for the method?
- Is the effect operationally meaningful?
- Did a critical regression occur?

A tiny, statistically convincing style improvement does not outweigh one unauthorized action.

### 10.6 Review changed cases, not only failed cases

The most informative review set is:

- candidate regressions;
- candidate wins;
- cases where judges disagree;
- cases with large cost or latency changes;
- cases where the trajectory changed but the outcome did not.

This reveals how the candidate works, not just how often it passes.

### 10.7 Write the decision record

```markdown
# Experiment: billing prompt v8

## Decision

Do not ship.

## Evidence

- Task completion improved from 84/100 to 90/100.
- Groundedness improved from 89/100 to 92/100.
- Two high-risk cases regressed: the candidate attempted a refund without
  sufficient evidence.
- Median latency increased from 1.8 s to 2.0 s, within budget.

## Diagnosis

The prompt more aggressively pursues task completion and weakens the escalation
boundary for ambiguous duplicate-charge reports.

## Next change

Restore the explicit refund preconditions while preserving the improved answer
structure. Add both regressions to the permanent suite.
```

The purpose of the report is a decision and a next step, not a pile of metrics.

### Lab: compare two configurations

Run baseline and candidate on the same dataset.

Produce:

- paired wins and regressions;
- results by risk and workflow slice;
- cost and latency changes;
- manual review of all changed high-risk cases;
- a written ship, reject, or investigate decision.

**Done when:** a reader can disagree with the decision using the evidence in the report.

---

## 11. Turn evals into CI release gates

An eval suite becomes an engineering control when it runs at the right time and blocks the right changes.

Do not put every expensive eval on every commit. Use layers.

### 11.1 Create three execution tiers

**Tier 1: fast checks**

- unit tests for evaluators;
- schema validation;
- prompt and config linting;
- 10–30 deterministic canary cases;
- no external judge calls where possible.

Run on each pull request.

**Tier 2: targeted regression**

- cases related to changed components;
- historical high-severity failures;
- narrow judges;
- retrieval or tool simulator tests.

Run on pull requests that modify AI behavior.

**Tier 3: full evaluation**

- complete representative dataset;
- repeated runs where needed;
- holdout;
- calibrated judges;
- manual review queue;
- cost and latency comparison.

Run before release, on a schedule, or for major model and architecture changes.

### 11.2 Select cases based on changed components

Map files or configuration to slices:

```yaml
change_map:
  app/retrieval/**:
    - retrieval
    - groundedness
    - high_risk
  prompts/billing/**:
    - billing
    - policy_boundary
    - historical_regression
  app/tools/refunds.py:
    - refund
    - write_tool
    - critical
```

Always include a small global canary set. Component mappings can be incomplete.

### 11.3 Encode the gate as data

```yaml
gate_version: 4

hard_requirements:
  evaluator_errors: 0
  critical_failures: 0
  historical_regressions: 0
  schema_pass_rate: 1.0

relative_requirements:
  high_risk_task_completion:
    min_delta: 0.0
  groundedness:
    min_delta: -0.01
  p95_latency_ms:
    max_delta_percent: 15
  mean_cost_usd:
    max_delta_percent: 20

manual_review:
  required_for:
    - any high-risk changed outcome
    - any new failure category
    - more than 5 judge disagreements
```

Version gate policy. A threshold change can affect release behavior as much as a code change.

### 11.4 Avoid flaky gates

Flaky evals teach engineers to ignore failures.

When a gate is unstable:

1. identify whether variance comes from application, environment, or evaluator;
2. preserve all repetitions;
3. retry infrastructure errors separately from application failures;
4. pin dependencies and fixtures;
5. use a minimum sample large enough for the decision;
6. quarantine only with an owner and expiry date.

Never silently rerun until green.

If the policy allows a retry, show both attempts in the report.

### 11.5 Keep secrets and user data out of artifacts

CI artifacts may be widely accessible. Use:

- synthetic or approved redacted cases;
- short retention;
- access controls;
- secret scanning;
- content hashing where raw text is unnecessary;
- separate secure storage for sensitive production-derived cases.

An eval dataset is part of the application’s data surface.

### 11.6 Make failures readable in the pull request

A useful gate comment says:

```text
EVAL GATE: FAILED

Blocking:
- 1 critical regression in slice `refund/write_tool`
  - billing_timeout_007: candidate retried a timed-out write with a new
    idempotency key

Non-blocking:
- groundedness: 91/100 → 93/100
- median latency: 1.8 s → 2.1 s
- estimated mean cost: $0.021 → $0.025

Artifacts:
- paired report
- candidate trace
- baseline trace
```

Do not make an engineer search a dashboard to learn why a check failed.

### Lab: add one honest gate

Start with:

- evaluator unit tests;
- deterministic canary cases;
- all historical critical regressions;
- zero evaluator errors;
- a readable failure report.

**Done when:** a deliberately reintroduced historical failure blocks the change and points to the failing trace.

---

## 12. Connect offline evals to production

Offline evals protect known behavior under controlled conditions. Production reveals new tasks, new failures, and distribution shifts.

The connection between them should be designed, not improvised.

### 12.1 Use one trace schema

The application should produce the same core trace shape in eval and production:

```json
{
  "trace_id": "tr_01842",
  "environment": "production",
  "configuration": {},
  "request": {},
  "retrieval": {},
  "model_calls": [],
  "tool_calls": [],
  "final_response": {},
  "final_state": {},
  "timing": {},
  "cost": {},
  "outcome_signals": {}
}
```

Production may redact fields or store secure references instead of raw values. The conceptual schema should still align, so a production failure can become an offline fixture.

### 12.2 Sample for different purposes

Use several sampling lanes:

| Lane | Purpose |
| --- | --- |
| Random | Estimate broad quality and discover unknown failures |
| Risk-based | Review sensitive workflows and write actions |
| Anomaly | Inspect long, expensive, error-heavy, or retry-heavy traces |
| Change-based | Observe traffic after prompt, model, corpus, or tool changes |
| User-signal | Inspect complaints, corrections, abandonment, and escalations |
| Novelty | Find requests dissimilar to the current eval set |

Complaint-only sampling is biased. Random-only sampling is inefficient for rare high-risk behavior. Combine lanes.

### 12.3 Turn a production trace into a case

Use a controlled promotion process:

1. preserve the trace and relevant versions;
2. remove or tokenize sensitive data;
3. minimize the fixture to the causal details;
4. ask a domain owner to define expected behavior;
5. classify source, risk, and failure category;
6. reproduce the failure offline;
7. add neighboring cases;
8. place it in regression if it represents durable behavior.

Do not copy raw production logs into a public repository.

### 12.4 Detect dataset drift

Compare production and eval distributions:

```text
workflow distribution
locale
message length
conversation length
tool usage
retrieval-empty rate
risk category
new intent clusters
cost and latency
```

Useful warning signs:

- 25% of production traffic uses a workflow represented by 2% of eval cases;
- a new locale has no reviewed cases;
- production conversations are much longer than fixtures;
- a newly exposed tool appears in traces but not in the suite;
- retrieval-empty rate doubles after a corpus update.

Drift does not always mean quality fell. It means the existing evidence is less able to answer whether quality is still acceptable.

### 12.5 Monitor evaluator drift

Judges can drift because:

- the judge model changes;
- application outputs change style;
- new domains appear;
- evidence grows longer;
- rubric interpretation changes.

Regularly sample judge decisions for human review. Track agreement by failure category, not only overall.

If a judge version changes, run old and new judges on the same calibration set before replacing it.

### 12.6 Connect online outcomes cautiously

Online signals include:

- task completion;
- successful state change;
- retry or correction;
- escalation;
- abandonment;
- support contact after answer;
- user rating.

These signals are useful but confounded. A user may give a positive rating to a fluent wrong answer. A correct answer may receive a poor rating because the policy itself is frustrating.

Use online outcomes to find traces and evaluate product value. Do not treat them as automatic truth labels for the model.

### Lab: productionize one feedback lane

Choose one:

- random weekly sample;
- all write-action failures;
- all multi-retry sessions;
- all user corrections;
- all traces after a model change.

Define privacy handling, review ownership, promotion criteria, and the path into regression.

**Done when:** one real production failure can become a safe, reproducible offline case.

---

## 13. Debug common eval failures

Evaluation systems fail too. The following playbooks help distinguish product problems from measurement problems.

### 13.1 “The score changed, but outputs look the same”

Check:

1. Did the dataset content change under the same filename?
2. Did the judge model or prompt change?
3. Did parsing or normalization change?
4. Did slice membership change?
5. Are errors included in one run and dropped in another?
6. Did repeated-run counts change?
7. Did a provider alias resolve to a new model?

Compare manifests before inspecting application behavior.

### 13.2 “The judge says fail, but the reviewer says pass”

Inspect:

- whether the judge received all relevant evidence;
- whether the rubric covers the boundary case;
- whether the reviewer used outside knowledge;
- whether the answer contains a subtle unsupported claim;
- whether the expected answer is too literal;
- whether hostile or irrelevant content influenced the judge;
- whether the structured output parser changed the verdict.

Then decide whether to update the human label, rubric, context, examples, or judge.

Do not automatically tune the judge to agree with every reviewer. Human labels can be wrong.

### 13.3 “Offline eval passes, production quality is poor”

Likely causes:

- dataset does not represent production;
- fixtures are cleaner than real integrations;
- conversations are longer in production;
- production tools return different errors;
- latency or timeout behavior is absent offline;
- users ask follow-up questions not represented in single-turn cases;
- online success requires a downstream state not checked offline;
- the application configuration differs.

Promote production traces and improve environment fidelity.

### 13.4 “The suite is too expensive”

Reduce cost in this order:

1. use deterministic checks where possible;
2. run only affected slices on pull requests;
3. cache identical stable judge inputs;
4. use a smaller calibrated judge for screening;
5. send uncertain or high-risk cases to a stronger judge;
6. batch work where the provider supports it;
7. run the full suite on a schedule;
8. retire duplicate cases only after coverage review.

Do not start by removing high-risk cases.

### 13.5 “The suite passes only after rerunning”

Measure the instability:

- which cases are flaky;
- whether the app or evaluator changed verdict;
- whether retrieval rank changed;
- whether a tool simulator leaked state;
- whether concurrency changed behavior;
- whether the model version is pinned.

For each flaky case, classify:

```text
expected stochastic variation
test environment defect
application reliability defect
evaluator reliability defect
unknown
```

Repeatedly passing on retry may itself be evidence of inadequate reliability.

### 13.6 “Engineers are optimizing to the test”

Symptoms:

- prompts contain phrases copied from known cases;
- benchmark scores rise while production complaints do not improve;
- changes exploit judge wording;
- holdout performance lags development performance;
- narrow formatting changes receive large quality gains.

Responses:

- protect holdouts;
- add fresh production samples;
- rotate challenge cases;
- use several evidence sources;
- review changed outputs manually;
- measure online outcomes;
- keep the benchmark focused on behavior rather than phrases.

### 13.7 “We have too many metrics”

For each metric, ask:

1. What decision does it affect?
2. What behavior does it measure?
3. Who owns a regression?
4. What action follows an alert?

Archive metrics without an answer. Keep raw data so they can be reconstructed if needed.

### 13.8 “A failure has no obvious fix”

The eval may be too broad. Decompose:

```text
Final answer failed
├── Was the request understood?
├── Was relevant evidence available?
├── Was the right evidence retrieved?
├── Was the right tool selected?
├── Were arguments correct?
├── Did the tool succeed?
├── Was the result interpreted correctly?
├── Was final state verified?
└── Was the outcome communicated accurately?
```

Move the evaluator closer to the first failed component.

---

## 14. A four-week implementation plan

This plan is designed for a small product team with an existing AI feature. Reduce scope rather than skipping review.

### Week 1: establish evidence

**Day 1**

- choose one workflow;
- write the eval contract;
- define hard invariants and preferences;
- name the baseline and candidate decision.

**Day 2**

- define the case schema;
- create ten common cases;
- create five boundary cases;
- create five integration-failure cases.

**Day 3**

- implement the application adapter;
- save complete run records;
- make exceptions visible;
- record immutable configuration.

**Day 4**

- review all outputs manually;
- write open failure notes;
- identify missing trace fields.

**Day 5**

- improve instrumentation;
- define the first taxonomy;
- prioritize the top failures.

**End-of-week deliverable:** a reproducible baseline run with 20 reviewed cases.

### Week 2: automate exact behavior

**Day 6–7**

- implement schema and permission checks;
- validate known IDs and tool arguments;
- verify final state;
- add evaluator unit tests.

**Day 8**

- add latency, cost, and step measurements;
- define slice summaries;
- distinguish errors from failures.

**Day 9**

- convert historical failures into regression cases;
- add fault-injection fixtures.

**Day 10**

- create a fast local command;
- add a small CI canary gate.

**End-of-week deliverable:** deterministic checks block at least one deliberately reintroduced defect.

### Week 3: add semantic evaluation

**Day 11**

- choose one fuzzy criterion;
- write a narrow rubric;
- collect human-labeled examples.

**Day 12–13**

- implement structured judge output;
- run calibration;
- inspect every disagreement.

**Day 14**

- revise rubric and context;
- document failure recall, precision, and limitations.

**Day 15**

- add the judge as a screening metric or soft gate;
- create a human review queue for uncertain and changed cases.

**End-of-week deliverable:** one calibrated judge with a clearly limited role.

### Week 4: make it operational

**Day 16–17**

- implement paired baseline/candidate reports;
- report by risk and workflow;
- encode the release policy.

**Day 18**

- add full pre-release evaluation;
- make gate failures readable;
- define handling for flaky cases.

**Day 19**

- choose a production sampling lane;
- define trace-to-case privacy handling;
- schedule judge and dataset audits.

**Day 20**

- run a real change through the entire process;
- write the decision record;
- hold a retrospective on what was slow, unclear, or untrusted.

**End-of-week deliverable:** one real engineering decision made through the eval system.

### What not to build in the first month

Unless scale already requires it, postpone:

- a custom annotation web application;
- a distributed execution platform;
- a universal quality score;
- dozens of judges;
- automatic prompt optimization;
- a large synthetic benchmark;
- complex statistical dashboards.

The first month should produce trusted evidence and safer decisions.

---

## 15. Appendix: reference implementation

This appendix provides a compact implementation shape. It is intentionally vendor-neutral.

### 15.1 Case loader

```python
import json
from pathlib import Path


def load_jsonl(path: str) -> list[dict]:
    records = []
    with Path(path).open(encoding="utf-8") as handle:
        for line_number, line in enumerate(handle, start=1):
            if not line.strip():
                continue
            try:
                records.append(json.loads(line))
            except json.JSONDecodeError as exc:
                raise ValueError(
                    f"{path}:{line_number}: invalid JSON: {exc}"
                ) from exc
    return records
```

### 15.2 Stable hashing

```python
import hashlib
import json


def stable_hash(value: object) -> str:
    encoded = json.dumps(
        value,
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=False,
    ).encode("utf-8")
    return "sha256:" + hashlib.sha256(encoded).hexdigest()
```

### 15.3 Evaluator protocol

```python
from typing import Protocol


class Evaluator(Protocol):
    name: str
    version: str

    def evaluate(
        self,
        case: dict,
        run_record: dict,
        config: dict,
    ) -> list[EvalResult]:
        ...
```

Return a list because one component evaluator may expose several distinct checks. Keep those checks separately named in real reports.

### 15.4 Run loop

```python
def run_suite(application, cases, config, evaluators):
    records = []

    for case in cases:
        run_record = execute_case(application, case, config)
        eval_results = []

        if run_record["status"] == "completed":
            for evaluator in evaluators:
                try:
                    eval_results.extend(
                        evaluator.evaluate(case, run_record, config)
                    )
                except Exception as exc:
                    eval_results.append(EvalResult(
                        evaluator=evaluator.name,
                        version=evaluator.version,
                        verdict="error",
                        reason=f"{type(exc).__name__}: {exc}",
                    ))

        records.append({
            "case": case,
            "run": run_record,
            "evaluations": [
                result.__dict__ for result in eval_results
            ],
        })

    return records
```

### 15.5 Case-level policy

Do not average away hard failures:

```python
HARD_EVALUATORS = {
    "output_contract",
    "allowed_tools",
    "known_transaction_ids",
    "requested_state",
}


def case_verdict(record: dict) -> str:
    if record["run"]["status"] == "error":
        return "error"

    evaluations = record["evaluations"]
    if any(item["verdict"] == "error" for item in evaluations):
        return "error"

    if any(
        item["evaluator"] in HARD_EVALUATORS
        and item["verdict"] == "fail"
        for item in evaluations
    ):
        return "fail"

    applicable = [
        item for item in evaluations
        if item["verdict"] != "not_applicable"
    ]
    if applicable and all(item["verdict"] == "pass" for item in applicable):
        return "pass"

    return "needs_review"
```

### 15.6 Slice summary

```python
from collections import defaultdict


def summarize(records):
    summary = defaultdict(
        lambda: {"total": 0, "pass": 0, "fail": 0, "error": 0, "needs_review": 0}
    )

    for record in records:
        verdict = case_verdict(record)
        slices = {"all", *record["case"].get("slice", [])}

        for slice_name in slices:
            summary[slice_name]["total"] += 1
            summary[slice_name][verdict] += 1

    return dict(summary)
```

### 15.7 Paired comparison

```python
def index_by_case(records):
    return {record["case"]["case_id"]: record for record in records}


def compare_runs(baseline, candidate):
    baseline_by_case = index_by_case(baseline)
    candidate_by_case = index_by_case(candidate)

    if baseline_by_case.keys() != candidate_by_case.keys():
        raise ValueError("baseline and candidate case IDs differ")

    comparison = []
    for case_id in sorted(baseline_by_case):
        baseline_record = baseline_by_case[case_id]
        candidate_record = candidate_by_case[case_id]
        baseline_verdict = case_verdict(baseline_record)
        candidate_verdict = case_verdict(candidate_record)

        comparison.append({
            "case_id": case_id,
            "slices": candidate_record["case"].get("slice", []),
            "risk": candidate_record["case"].get("risk", "unknown"),
            "baseline": baseline_verdict,
            "candidate": candidate_verdict,
            "changed": baseline_verdict != candidate_verdict,
        })

    return comparison
```

### 15.8 Gate

```python
RISK_ORDER = {
    "low": 1,
    "medium": 2,
    "high": 3,
    "critical": 4,
}


def apply_gate(comparison, candidate_records):
    blockers = []

    for item in comparison:
        if (
            RISK_ORDER.get(item["risk"], 0) >= RISK_ORDER["high"]
            and item["baseline"] == "pass"
            and item["candidate"] != "pass"
        ):
            blockers.append({
                "case_id": item["case_id"],
                "reason": "high-risk regression",
            })

    for record in candidate_records:
        if case_verdict(record) == "error":
            blockers.append({
                "case_id": record["case"]["case_id"],
                "reason": "run or evaluator error",
            })

        for result in record["evaluations"]:
            if (
                record["case"].get("risk") == "critical"
                and result["verdict"] == "fail"
            ):
                blockers.append({
                    "case_id": record["case"]["case_id"],
                    "reason": (
                        "critical failure in "
                        + result["evaluator"]
                    ),
                })

    return {
        "verdict": "fail" if blockers else "pass",
        "blockers": blockers,
    }
```

This is a starting point, not a universal policy. The gate should reflect the application’s actual risk model.

---

## 16. Appendix: working templates

### A. Eval case

```json
{
  "case_id": "workflow_condition_001",
  "title": "Short human-readable title",
  "source": {
    "type": "production",
    "reference": "redacted-trace-reference",
    "note": "Why this case matters"
  },
  "slice": ["workflow", "behavior", "risk_group"],
  "risk": "medium",
  "input": {
    "message": "User request",
    "authenticated": true
  },
  "fixtures": {
    "documents": [],
    "tool_responses": {},
    "initial_state": {}
  },
  "expected": {
    "must": [],
    "must_not": [],
    "allowed_tools": [],
    "forbidden_tools": [],
    "relevant_document_ids": [],
    "final_state": {}
  },
  "metadata": {
    "created_at": "2026-09-08",
    "owner": "team-name",
    "last_reviewed_at": "2026-09-08"
  }
}
```

### B. Evaluator card

```markdown
# Evaluator: groundedness

Version: 3
Owner: assistant-quality

## Decision supported

Detect answers containing material claims unsupported by supplied evidence.

## Inputs

- user request
- approved documents
- tool calls and results
- verified final state
- final answer

## Output

- pass or fail
- unsupported claims
- concise reason
- confidence used only for review prioritization

## Calibration

- 100 human-labeled cases
- failure recall: ...
- failure precision: ...
- last audited: ...

## Known limitations

- ...

## Approved uses

- regression metric
- review queue routing

## Prohibited uses

- sole gate for critical authorization behavior
```

### C. Failure record

```yaml
failure_id: fail_0182
case_id: billing_timeout_007
severity: high
first_consequential_failure: tools.completion_not_verified

observed:
  - write tool timed out after committing
  - agent retried with a new idempotency key
  - final answer claimed one successful refund

impact:
  - duplicate action was possible

intervention:
  - require stable idempotency key
  - verify state after ambiguous write result
  - block success language without final-state evidence

regression_cases:
  - billing_timeout_007
  - billing_timeout_008

owner: billing-workflow
status: fixed
```

### D. Experiment definition

```yaml
experiment_id: exp_billing_prompt_v8
decision: ship or reject billing prompt v8

baseline:
  config_id: billing-v7-model-a

candidate:
  config_id: billing-v8-model-a

datasets:
  - common
  - regression
  - high_risk
  - holdout

repetitions:
  common: 1
  high_risk: 3
  critical: 5

gate:
  critical_failures: 0
  high_risk_regressions: 0
  max_groundedness_delta: -0.01
  max_p95_latency_increase_percent: 15
  max_mean_cost_increase_percent: 20

manual_review:
  - all changed high-risk outcomes
  - all judge disagreements
  - all new failure categories
```

### E. Pull request checklist

```markdown
## AI behavior change

- [ ] The changed workflow and expected behavior are described.
- [ ] Relevant cases were added or updated.
- [ ] Dataset changes were reviewed separately from application changes.
- [ ] Deterministic checks cover exact requirements.
- [ ] Baseline and candidate used the same cases and fixtures.
- [ ] High-risk changed outcomes were manually reviewed.
- [ ] Evaluator and run errors are zero or explicitly resolved.
- [ ] Cost and latency effects are reported.
- [ ] The decision follows the versioned gate policy.
- [ ] New important failures were converted into regression cases.
```

### F. Eval system health review

```markdown
## Dataset

- Does it still resemble production traffic?
- Are high-risk and long-tail workflows represented?
- Are synthetic cases crowding out production cases?
- Is the holdout still protected?
- Are obsolete cases marked or removed with review?

## Evaluators

- Do deterministic checks have unit tests?
- Are judge versions pinned?
- Has judge agreement been audited recently?
- Are evaluator errors visible?
- Does each evaluator still support a real decision?

## Harness

- Are run manifests complete?
- Can a result be reproduced?
- Are failed and timed-out runs preserved?
- Are fixtures isolated and resettable?
- Are sensitive artifacts protected?

## Operations

- Do pull request gates finish in a useful time?
- Are flaky cases owned?
- Do incidents become regression cases?
- Are production shifts reflected in the suite?
- Can engineers understand a failure without opening several systems?
```

---

## Closing: evals as an engineering memory

The practical purpose of an eval system is not to produce a score. It is to preserve what the team has learned about making the application work.

A production failure becomes a trace. The trace becomes a diagnosis. The diagnosis becomes a case. The case gains an evaluator. The evaluator becomes a release gate. The next engineer can change the system without rediscovering the same failure.

That is engineering memory.

Build it one workflow at a time. Keep cases close to real behavior. Use exact checks wherever possible. Calibrate every probabilistic measurement. Inspect trajectories, not only answers. Compare changes against a baseline. Make important failures permanent.

The result is not a perfect AI system. It is a system the team can understand, improve, and ship with evidence.
