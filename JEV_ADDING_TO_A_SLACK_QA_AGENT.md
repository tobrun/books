# Adding Jev to a Slack Q&A Agent

## An Additive Integration Guide

A Slack Q&A agent already has a working division of labor: one model decides whether a message deserves a response, another researches and writes that response, and a third turns useful feedback into knowledge-base improvements. Adding a new model should not casually disturb that arrangement.

This book explains how to add TypeSafe AI's Jev (`jev-1.13.0`) to that system without replacing any of its existing model jobs. The starting assumption is deliberately conservative: you have a Jev API key, an operating Slack agent, and an existing LLM relevance gate, answering agent, and feedback writer that you want to preserve.

Jev begins as an observer. It runs beside the LLM relevance gate, records an independently structured judgment, and changes no user-visible outcome. Only after you have labels, calibration data, and evidence that a narrow confident band is safe should it be allowed to short-circuit a small share of traffic. Many valuable Jev jobs—feedback triage, answer-quality judging, duplicate detection, and tagging—never need to touch the critical answering path at all.

This is an integration guide, not a claim that any decision model is a drop-in replacement for evaluation. Jev's typed output can make malformed output impossible. It cannot make a wrong answer impossible. The system still needs good questions, explicit escape hatches, timeouts, observability, human labels, and a kill switch.

> **Timing note.** Product capabilities, pricing, limits, availability, and benchmarks in this book are time-sensitive. The specific Jev facts here describe `jev-1.13.0` as documented and reported when this guide was prepared. Verify them in the vendor documentation and contract before you ship.

---

## Contents

1. [The conservative thesis](#1-the-conservative-thesis)
2. [The reference architecture](#2-the-reference-architecture)
3. [Jev facts that shape the design](#3-jev-facts-that-shape-the-design)
4. [The failure modes to design around](#4-the-failure-modes-to-design-around)
5. [Integration options](#5-integration-options)
6. [Implement the Jev gate](#6-implement-the-jev-gate)
7. [Wire Jev alongside the existing gate](#7-wire-jev-alongside-the-existing-gate)
8. [Express per-tenant scope carefully](#8-express-per-tenant-scope-carefully)
9. [Add useful jobs beyond relevance](#9-add-useful-jobs-beyond-relevance)
10. [Build the evaluation system first](#10-build-the-evaluation-system-first)
11. [Operate the integration](#11-operate-the-integration)
12. [Roll out in phases](#12-roll-out-in-phases)
13. [Appendix: implementation checklist](#13-appendix-implementation-checklist)
14. [Sources to verify](#14-sources-to-verify)

---

## 1. The conservative thesis

Jev is a hosted, non-generative decision model. Instead of prompting a model to write an answer and then parsing that answer, you send state plus named, typed questions. Jev returns probabilities over the answers you made legal.

The question types are:

- **Noul:** a typed yes/no decision.
- **Choice:** one selection from a declared set of choices.
- **Score:** a bounded score.

This shape is useful in a Slack agent because much of the work around an answer is not prose generation:

- Is this a question the channel should answer?
- Is this message an announcement rather than a request?
- Does a prior answer resolve the new question?
- Is this feedback a knowledge-base gap or just noise?
- Did the agent's answer address the question and remain grounded?

None of those decisions require a paragraph. They need a bounded answer, a probability, and code that knows what to do with uncertainty.

That does **not** mean Jev should replace the LLM everywhere. A Slack Q&A system still needs an LLM to reason over a knowledge base, decide which read-only tools to use, synthesize an answer, and draft an edit from human guidance. The safe design is additive:

```text
Existing LLM relevance gate ────────────────┐
                                             ├── current answer/no-answer decision
New Jev relevance observer ── metrics/logs ─┘

Existing answering agent ── tools ── answer
Existing feedback writer ── human guidance ── proposed KB edit

New Jev background jobs ── triage / judge / dedup / tagging
```

The integration rule for the rest of this book is simple:

> Steps that add Jev keep the current model calls and their code paths. Every Jev error, timeout, malformed integration response, or disabled setting degrades to exactly the behavior the bot had before Jev existed.

This rule gives the integration an honest default. Jev has to earn influence through measured performance rather than through a large architectural bet.

### The real blocker is labels

The hardest dependency is not an SDK, a secret, or a new environment variable. It is a labeled evaluation set.

A model should not make a user-visible relevance decision until you know how it behaves on your channels: messages aimed at a named person, deploy notices written as questions, shorthand, multilingual messages, actual requests with poor grammar, prompt injection, social chatter, and tenant-specific scope boundaries.

For a paired comparison with the existing gate, roughly 800–1,000 adjudicated top-level messages is a practical target. At an expected 10% disagreement rate, that yields about 85 disagreements—enough to make a paired statistical comparison useful. Until then, Jev is an observer or a background worker.

---

## 2. The reference architecture

Slack Q&A bots differ in details, but most production implementations follow this shape.

### 2.1 Intake and routing

Slack events arrive through Socket Mode or the Events API. A deterministic router rejects events that should never reach a model:

- messages from the bot itself;
- unsupported message subtypes;
- thread replies, if the bot only handles top-level questions;
- channels or workspaces the bot is not authorized to serve;
- empty or trivially invalid content.

This layer should remain deterministic. It is a cheap guardrail and produces clear reasons for a skip.

### 2.2 The LLM relevance gate

For eligible top-level messages, a small LLM call answers the operational question:

> Should the bot answer this message?

The current gate commonly returns `YES` or `NO` through a JSON schema. Direct messages and messages that explicitly mention the bot usually skip the gate because user intent is already clear. If the gate errors, the system often fails open and lets the answering agent attempt a response.

That behavior is part of the current product contract. Adding Jev must not silently change it.

### 2.3 The answering agent

When a message passes the gate, a larger LLM runs an agent loop. Its tools are normally read-only: a knowledge base, source control, ticket tracker, wiki, and perhaps Slack search. The agent either provides an answer with appropriate evidence or reports that it cannot find one.

This remains an LLM job. An answer is open-ended language grounded in retrieved evidence, not a bounded decision.

### 2.4 The feedback writer

Human feedback—an upvote, downvote, correction, or explicit instruction—can be transformed into a proposed knowledge-base edit. A feedback-writing LLM does that drafting. A human or deterministic publishing process should still control whether the edit becomes authoritative.

Jev can improve the *input* to this job through triage. It does not replace the writer.

### 2.5 Observability and evaluation

The baseline system needs metrics for answered, skipped, and errored messages, broken down by tenant and model. It also needs an evaluation script that runs both the current gate and prospective alternatives over a fixed golden set.

Without this baseline, there is no way to distinguish an attractive demo from a safe integration.

---

## 3. Jev facts that shape the design

The following properties are the integration-relevant facts to verify against current Jev documentation and your agreement.

| Property | Reported / documented behavior for Jev 1.13 | Design consequence |
| --- | --- | --- |
| Interface | `POST /v1/systemone` with state and named questions; Python SDK package `typesafe-sdk` | Isolate it behind a separate client; it does not require a change to the LLM provider. |
| Output | A probability distribution over the options you define | Scores can support thresholds and cascades, after calibration. |
| Customization | Runtime instructions and criteria, rather than fine-tuning | Tenant scope must be conveyed through runtime state or criteria. |
| Limits | Roughly 64k tokens per request, up to 255 choices, and high rate limits | Slack-message workloads are comfortably below the service limits. |
| Language | English is reportedly strongest | Treat non-English channels as a separate evaluation slice. |
| Data | Not trained on customer data; zero data retention reportedly available through an enterprise contract; hosted in the US | Complete a data-handling review before sending real Slack messages. |
| Reliability | Provided without an SLA; initial availability reported around 99.8% | Use short timeouts, no request-path retries, fallbacks, and fail-open behavior. |
| Versioning | Moving aliases can resolve to new versions | Pin `jev-1.13.0` and record the returned model identifier. |
| Price | Roughly $0.042 per million input tokens, output free | Shadow traffic is inexpensive, but measure actual usage before forecasting. |
| Latency | Vendor quote around 70–500 ms | Measure from the deployment region; never make the hot path depend on the quote. |

Two implications deserve emphasis.

First, schema-valid output is a real operational benefit. A model that only returns the choices you declared cannot invent an unparseable category. It reduces parser and repair code. It does not guarantee that the selected category is semantically correct.

Second, probabilities are not permissions. A score of `0.99` is only useful if a `0.99` score on your own labeled data corresponds to the reliability you need. Thresholding without calibration turns a number with a decimal point into false certainty.

---

## 4. The failure modes to design around

The relevance gate is a short, noisy, in-scope-versus-out-of-scope classification problem. That is exactly where reported independent Jev tests identified important weaknesses. Start by designing around them.

### 4.1 Forced answers need an escape hatch

If a Choice question has no valid “none” option, an out-of-scope input must still be placed in one of the supplied categories. Reported tests found that such inputs could be forced into a category at approximately `0.99` confidence. Adding an explicit escape option caught most of them.

Every broad classification in this guide therefore includes an escape hatch:

```python
"none_of_these": "Anything else, gibberish, unrelated content, or an unclear case."
```

The wording matters. The option should be a real, acceptable answer rather than a token afterthought.

### 4.2 Broad questions hide multiple decisions

A single question such as “Should the bot answer this?” packages several judgments together:

- Is there a request?
- Is it related to this channel?
- Is it directed to the group rather than one person?
- Is it an announcement or social message?
- Is it in the tenant's declared scope?

Reported benchmarks found a large difference between one broad question and several atomic questions combined on labeled data. Do not treat that exact benchmark result as a guarantee. Treat the pattern as a design instruction: ask separable questions separately, then learn or validate how to combine them.

### 4.3 Literal criteria are part of the program

Jev answers the question that was written, not the question its author intended. Bad criteria can make a classifier worse than an uninformative baseline.

Write criteria as if they were production code:

- Define positive and negative cases.
- State whether access requests, approvals, or personal opinions count.
- Include an explicit escape condition.
- Test counterexamples, not just clean examples.
- Version and review criteria with the same care as a prompt or routing rule.

### 4.4 State is untrusted and noisy state degrades decisions

The Slack message is untrusted data. A message might contain an instruction aimed at a model, irrelevant logs, quoted text, or content that is adversarially chosen. Scope text can be noisy too. Reported tests indicate that irrelevant state and injected text can move Jev's decision.

Keep state minimal and clearly named. Never let message content redefine the question, criteria, permitted actions, tenant policy, or thresholds. Keep operator-controlled scope separate from user-controlled message text.

### 4.5 Confidence is imperfectly calibrated

Choice and Score responses have been reported as tending toward overconfidence, while Noul responses have tended toward underconfidence. Whether that holds for your traffic is an empirical question. Calibrate every question separately and inspect reliability curves by tenant and language.

### 4.6 No SLA means no dependency of record

An unavailable observer should not make the bot unavailable. Jev must never raise from its integration boundary. In the request path, use a hard timeout and zero retries; the existing LLM gate remains the fallback. In shadow and audit modes, run the call asynchronously so it adds no user-visible latency at all.

---

## 5. Integration options

Every option below is additive. The existing gate remains available as the primary decider and fallback.

| Option | What changes | User-visible? | Verdict |
| --- | --- | --- | --- |
| Shadow gate | Jev runs beside the LLM gate; both verdicts are logged; only the LLM decision counts. | No | Start here. |
| Confidence pre-filter | Jev short-circuits only an evaluated, highly confident band; everything else goes to the unchanged LLM gate. | Yes, for a small share of traffic | Consider only after shadow evidence. |
| Tie-breaker / audit | Jev re-checks LLM `NO` decisions and flags likely misses for reviewers. | Reviewers only | A useful safety net. |
| New background jobs | Jev handles triage, judging, deduplication, or tagging. | Varies | Valuable without changing the gate. |
| Replace the LLM gate | Jev becomes the sole relevance decider. | Yes | Out of scope for this guide. |

### 5.1 Start with a shadow gate

The shadow gate is intentionally boring:

1. Run the existing LLM gate exactly as it runs today.
2. Start a Jev classification in parallel.
3. Record both verdicts and the Jev probability distribution.
4. Return the LLM result.

The only new production risk is the cost of an additional asynchronous call and the privacy consequence of transmitting message text. The answering behavior is unchanged.

### 5.2 Audit LLM `NO` decisions

False negatives are often more expensive than false positives in a help channel: a missed question gets no help, while an unnecessary answer can at least be ignored. An audit mode can re-check LLM `NO` decisions, then flag a small, high-confidence subset for human review.

This is especially helpful before pre-filtering because it gives you a stream of candidate misses without allowing Jev to answer users directly.

### 5.3 A narrow, asymmetric pre-filter

The cascade pattern is familiar from FrugalGPT-style and RouteLLM-style model routing: a cheap model handles easy cases and sends uncertain ones to a more capable model. It works only when the first-stage confidence is measured on the actual task.

For a Slack relevance gate, thresholds should be asymmetric:

- A very high **calibrated YES** may skip the LLM gate and go to the existing answering agent.
- A **NO** should only skip the LLM gate for clearly harmless categories such as an announcement or social message, and only after a dedicated evaluation demonstrates that this does not create misses.
- Every uncertain result, timeout, error, missing score, new tenant, new language slice, or changed question goes to the existing LLM gate.

The asymmetry reflects the product cost: missing a genuine question usually harms more than attempting an unnecessary answer.

---

## 6. Implement the Jev gate

The implementation boundary should return one shared result type and should never raise. The rest of the application should not need to know about SDK exceptions, HTTP status codes, or response-shape details.

### 6.1 A fail-closed client boundary

```python
# gate_jev.py
import os
import time
from dataclasses import dataclass

JEV_MODEL = os.getenv("GATE_JEV_MODEL", "jev-1.13.0")  # pinned
JEV_TIMEOUT = float(os.getenv("GATE_JEV_TIMEOUT_S", "1.5"))


@dataclass
class JevVerdict:
    # None means the integration did not obtain a usable decision.
    p_answer: float | None
    kind: str | None
    probs: dict | None
    model: str
    latency_ms: int | None


def _questions(tenant_scope: str | None):
    from typesafe_sdk import Choice, Noul

    questions = {
        "is_request": Noul(
            instructions=(
                "Does `message` ask a question or request help, information, "
                "or a fix?"
            )
        ),
        "is_in_domain": Noul(
            instructions=(
                "Is `message` about the technical topics this channel supports?"
            )
        ),
        "addressed_to_person": Noul(
            instructions=(
                "Is `message` directed at a specific named person rather than "
                "the channel?"
            )
        ),
        "kind": Choice(
            instructions="Which best describes `message`?",
            criteria={
                "question": "A question the channel could answer.",
                "request_for_person": (
                    "Something only a specific person can handle, such as "
                    "an approval, access grant, or personal opinion."
                ),
                "announcement": "An announcement, status update, deploy notice, or FYI.",
                "social": "A greeting, thanks, joke, reaction, or casual conversation.",
                "none_of_these": (
                    "Anything else: gibberish, unrelated content, or an unclear case."
                ),
            },
        ),
    }
    if tenant_scope:
        questions["in_tenant_scope"] = Noul(
            instructions=(
                "Does `message` fall inside the operator-provided scope in `scope`?"
            )
        )
    return questions


def classify_jev_sync(text: str, tenant_scope: str | None) -> JevVerdict:
    """Return a usable verdict or a neutral failure result; never raise."""
    if not text.strip():
        return JevVerdict(None, None, None, JEV_MODEL, None)

    started = time.monotonic()
    try:
        client = get_jev_client(timeout=JEV_TIMEOUT, max_retries=0)
        state = {"message": text[:2000]}
        if tenant_scope:
            state["scope"] = tenant_scope

        response = client.system_one(
            model=JEV_MODEL,
            state=state,
            questions=_questions(tenant_scope),
        )
        probs = flatten(response.answers)
        return JevVerdict(
            p_answer=combine(probs),
            kind=top_kind(probs),
            probs=probs,
            model=response.model,
            latency_ms=int((time.monotonic() - started) * 1000),
        )
    except Exception:
        # Log and count the exception internally, without leaking message text.
        return JevVerdict(None, None, None, JEV_MODEL, None)
```

`get_jev_client`, `flatten`, and `top_kind` are deliberately application-specific:

- `get_jev_client` should create or retrieve a singleton client configured from the secret-backed environment.
- `flatten` should turn the SDK's response object into a stable internal dictionary. Preserve the full probability distribution, not only the winner.
- `top_kind` should return the highest-probability category from the `kind` Choice answer.

### 6.2 Start with an understandable combination rule

`combine` turns atomic probabilities into an initial `p_answer`. Begin with a visible rule that can be inspected and tested. For example:

```python
def combine(probs: dict) -> float | None:
    request = probs.get("is_request", {}).get("yes")
    domain = probs.get("is_in_domain", {}).get("yes")
    addressed = probs.get("addressed_to_person", {}).get("yes")
    kind = probs.get("kind", {})
    if None in (request, domain, addressed):
        return None

    # This is a starting heuristic, not a calibrated probability.
    negative_kind = max(
        kind.get("announcement", 0.0),
        kind.get("social", 0.0),
        kind.get("request_for_person", 0.0),
        kind.get("none_of_these", 0.0),
    )
    return request * domain * (1 - addressed) * (1 - negative_kind)
```

That formula is not “the Jev relevance probability.” It is a transparent temporary feature combination. Once you have labels, replace the hand-written heuristic with a logistic regression or another simple calibrated model trained on the atomic probabilities. Keep the model versioned alongside the questions and criteria that produced its features.

### 6.3 Preserve a narrow interface

Do not spread raw Jev response objects through the Slack application. One stable `JevVerdict` gives you three important properties:

- application code has one neutral failure representation: `p_answer is None`;
- tests can mock one function instead of an external SDK;
- a future API or SDK change stays local to `gate_jev.py`.

### 6.4 Test the failure contract

Mock `system_one` the same way the LLM client is mocked. At minimum, cover:

- an empty message makes no external call;
- a timeout maps to `p_answer=None`;
- HTTP 429 and other SDK failures map to `p_answer=None`;
- a partial or unexpected response maps to `p_answer=None`;
- shadow mode never changes the existing gate decision;
- model, role, and latency fields are recorded without recording sensitive raw message text in metrics.

Pin the SDK dependency and its compatible version in the normal dependency manifest. The Jev-specific evaluation job needs only the Jev key as a repository secret; it should not need cloud-model credentials unless it is intentionally running the existing LLM baseline too.

---

## 7. Wire Jev alongside the existing gate

The initial gate wiring should make the hierarchy unmistakable: the LLM decides; Jev observes.

```python
async def gate(text, tenant):
    llm = await existing_llm_gate(text, tenant)  # unchanged: True / False / None

    mode = os.getenv("GATE_JEV_MODE", "off")  # off | shadow | prefilter | audit
    if mode == "off":
        return llm

    async def observe_jev():
        try:
            jev = await asyncio.wait_for(
                asyncio.to_thread(classify_jev_sync, text, tenant.scope),
                timeout=2,
            )
        except (TimeoutError, asyncio.TimeoutError):
            jev = JevVerdict(None, None, None, JEV_MODEL, None)

        record_pair(llm=llm, jev=jev, tenant=tenant)
        if mode == "audit" and llm is False and (jev.p_answer or 0) >= AUDIT_T:
            flag_possible_miss(text, tenant)  # reviewer-only notice

    if mode in {"shadow", "audit"}:
        asyncio.create_task(observe_jev())
        return llm

    # Prefilter is intentionally omitted here until its own gate is approved.
    return llm
```

The exact task-supervision pattern depends on your runtime. The important behavior is:

1. In `off`, there is no Jev call.
2. In `shadow` and `audit`, the response path does not wait for Jev.
3. Every Jev failure records a neutral observation and returns the original LLM outcome.
4. Audit notices are reviewer-only. They never cause an answer by themselves.

### 7.1 A safe pre-filter path

Pre-filtering changes the order. It should be a distinct, explicitly enabled branch with a clear fallback:

```text
Jev classification
  │
  ├─ usable, calibrated YES above YES threshold
  │      └─ send to unchanged answering agent
  │
  ├─ usable, calibrated NO in a proven-safe benign category
  │      └─ skip with a recorded Jev reason
  │
  └─ uncertain / new slice / error / timeout / everything else
         └─ run unchanged LLM relevance gate
```

Use a configuration value such as `GATE_JEV_MODE=prefilter`, but do not let its existence imply readiness. The branch is only appropriate after the frozen evaluation set, shadow results, and calibration analysis support its thresholds.

### 7.2 Keep the kill switch absolute

`GATE_JEV_MODE=off` must remove Jev from the decision path and restore pre-integration behavior. Test this as a release criterion. A kill switch that still depends on an SDK import, remote configuration fetch, or Jev client construction is not a kill switch.

---

## 8. Express per-tenant scope carefully

Many Slack agents allow teams to maintain scope text, perhaps in a `relevance.md` file. Jev can use that material, but scope is not free context: irrelevant or ambiguous text can lower decision quality.

Choose the least complicated representation that is accurate enough for the tenant.

### 8.1 Pass short scope text as state

The most self-service option is to pass a concise `scope` string and ask a dedicated `in_tenant_scope` Noul question.

```python
state = {
    "message": text[:2000],
    "scope": tenant.scope,
}
```

This preserves a tenant's ability to describe its own channel. Keep the scope short and operator-controlled. Do not include an entire wiki page, previous Slack threads, or unreviewed user-generated instructions.

### 8.2 Render structured scope into criteria

A more precise alternative is a structured scope model:

```yaml
in_scope:
  - deployment and incident questions
  - service ownership and runbooks
  - access troubleshooting with documented procedures
out_of_scope:
  - approval requests
  - requests for a specific manager's opinion
  - non-engineering HR matters
```

Render that into explicit criteria for an atomic question. This can be easier to review than prose, but it creates a stronger correctness requirement: wrong criteria are worse than vague criteria because they encode a confidently wrong policy.

### 8.3 Add labeled examples only after you have them

Examples from a tenant's own adjudicated mistakes can clarify difficult boundaries. Do not add examples opportunistically from a handful of anecdotes. Each example becomes part of the classifier's policy and must be maintained when scope changes.

### 8.4 Fit a tenant-specific head only when supported by data

Once a tenant has about 300 reliable labels, a per-tenant calibrated head over the same Jev probabilities may be justified. Until then, a tenant-specific model will usually overfit sparse examples and obscure behavior.

### 8.5 Opt out when scope is not expressible

Some channels have social rules or authority boundaries that cannot be described reliably as short criteria. Leave these tenants on the existing LLM gate. An opt-out is a valid design outcome, not a failure of the rollout.

---

## 9. Add useful jobs beyond relevance

The highest-value Jev work may live entirely outside the initial gate. These jobs preserve the existing LLM roles while giving them cleaner inputs or better feedback.

| Job | Fit | How to use it safely |
| --- | --- | --- |
| Answer-quality judge | High | Score groundedness, whether the answer addressed the question, and tone asynchronously. Compare scores with 👍/👎 rather than treating them as truth. |
| Feedback triage | High | Use a Choice such as `kb_gap`, `wrong_answer`, `irrelevant`, `tone`, or `noise`. Send only useful categories to the existing feedback writer. |
| Duplicate detection | Medium–high | Retrieve a shortlist with search or embeddings, then ask one Noul per candidate: “Does `prior_answer` resolve `message`?” Offer an earlier answer as a link. |
| Message tagging | Medium | Add questions for urgency, topic, or component in the same call. Use tags in dashboards and routing, not as undisputed facts. |
| Tool hints | Low–medium | Predict likely tool families as hints for the answering agent. Never prune the agent's tool set solely from the prediction. |
| Thread follow-up detection | Medium | Include thread context and compare against the existing deterministic rule in shadow mode before changing routing. |
| Abstention prediction | Low | A message alone does not reveal what the knowledge base covers. Do not expect Jev to infer it reliably without retrieval evidence. |

### 9.1 Feedback triage: an especially clean first job

Feedback triage is attractive because it is off the hot path and bounded. The existing feedback writer stays responsible for converting useful guidance into proposed edits; Jev simply reduces the writer's exposure to reactions, vague complaints, and irrelevant messages.

An example Choice:

```text
Which best describes this feedback?

- kb_gap: the answer could not use information that should exist in the KB
- wrong_answer: the answer made a factual or procedural error
- irrelevant: the answer did not address the user's request
- tone: the answer's style or phrasing was the problem
- noise: no actionable product or KB feedback is present
- none_of_these: unclear or outside these categories
```

Use the result to prioritize review, not to auto-publish content changes.

### 9.2 Answer judging must be validated against humans

An asynchronous judge can provide a useful leading indicator for answer quality. It is not a replacement for human labels. Measure correlation with explicit positive and negative feedback, then inspect the cases where the judge and humans disagree. Strong agreement in one tenant does not automatically generalize to another.

### 9.3 Duplicate detection needs retrieval first

Jev should not compare a new question against an unbounded archive. First retrieve a short candidate list using lexical search, embeddings, or both. Then use a separate typed question per candidate. This keeps state compact and ensures the decision is answering the right question: whether the prior answer resolves the new request, not whether two strings look similar.

---

## 10. Build the evaluation system first

Evaluation is the feature that lets every later feature become safe.

### 10.1 Collect labels from production signals

Build a retention-conscious decision log for every eligible top-level message. Store enough to reproduce evaluation under appropriate access controls:

- stable message and tenant identifiers;
- deterministic router result;
- existing LLM gate verdict;
- Jev model identifier, questions version, probabilities, and derived verdict;
- final user-visible outcome;
- latency and error class;
- human label, when one becomes available.

Add product feedback that creates useful labels:

- a “should have answered” control on skips or audit notices records a likely false negative;
- a “didn't need an answer” reason on downvotes records a likely false positive;
- reviewer decisions on audit flags record a high-value adjudicated slice.

Mine three to six months of allowed channel history. A strong LLM can pre-label it offline to reduce reviewer effort, but humans should adjudicate cases where candidate models disagree. Do not turn the pre-label into ground truth.

### 10.2 Add the cases production will hide from you

Build a deliberately hard slice alongside organic samples:

- questions directed at a named person;
- announcements phrased as questions;
- non-English messages;
- prompt-injection-like text;
- empty, malformed, and gibberish messages;
- messages at the tenant-scope boundary;
- requests that need an answer but look socially phrased.

The point is not to create a contest set. It is to find the shape of the model's unsafe confidence.

### 10.3 Use the right metrics

The primary metric is recall on true `YES`: the system must not quietly miss real requests for help. Precision should not fall below the current LLM gate's baseline.

Track at least:

- recall and precision, overall and per tenant;
- false-negative rate on genuine questions;
- cost-weighted error, using your actual business costs;
- coverage: the share of traffic eligible for a confident Jev action;
- expected calibration error and reliability curves per atomic question;
- p50 and p95 latency;
- timeout and error rates;
- slices by language, scope complexity, channel, and new versus established tenant.

Keep all model, question, criteria, and threshold versions in the evaluation output. A score that cannot be tied to the exact classifier configuration is not reproducible.

### 10.4 Compare paired predictions

The existing LLM gate and Jev see the same messages, so use a paired comparison such as McNemar's test instead of treating their results as independent samples. The meaningful unit is a disagreement: one gate is right where the other is wrong.

As a planning estimate, about 85 disagreement cases make the comparison informative. If the systems disagree on around 10% of inputs, that implies roughly 850 labeled messages. This is why ten polished examples are not enough.

### 10.5 Freeze, re-run, and recalibrate

Run shadow mode for at least two weeks or about 1,000 top-level messages, whichever gives the better coverage of normal traffic. Freeze a representative evaluation set and re-run it whenever you change:

- the pinned Jev model;
- question wording or criteria;
- the tenant scope representation;
- the probability-combination model;
- thresholds;
- the existing LLM gate.

Changing any of these changes the classifier. A previously approved threshold no longer has the same meaning.

---

## 11. Operate the integration

### 11.1 Cost

At an estimated 1,300 input tokens per message and a reported input price of roughly $0.042 per million tokens, Jev costs approximately:

```text
1,300 / 1,000,000 × $0.042 ≈ $0.000055 per call
100,000 messages ≈ $5.46
```

Those are directional arithmetic, not a bill forecast. Use measured request tokens, actual vendor pricing, retries if any, and the number of calls per message. Shadow traffic is normally cheap enough that cost should not be the reason to skip evaluation.

### 11.2 Latency

Independent reports cited ranges around 0.2–0.65 seconds, while the vendor quote is wider. Both are less useful than a histogram from the region where the Slack agent runs.

Use a hard 1.5-second request-path timeout with no retries. A retry can turn a transient problem into a user-visible delay while preserving little value: the LLM gate is already available as the fallback. Shadow and audit calls should not be awaited by the user path.

### 11.3 Metrics

Record the exact resolved model, not only a moving alias. Add a role dimension such as:

- `primary` for a user-visible pre-filter decision;
- `shadow` for a non-decision observation;
- `audit` for a reviewer-only check.

Useful counters and distributions include:

- `JevAgree` and `JevDisagree` against the LLM gate;
- `JevTimeout`;
- `JevDeferred` for calls not made or results that were not eligible to act;
- confidence histograms by question and action;
- classification latency;
- top error classes;
- audit flags and their human-confirmed precision.

Dimension growth can make a metrics system expensive or unusable. Bound tenant cardinality where necessary, and put high-cardinality identifiers in traces or protected logs rather than in every metric label.

### 11.4 Security and privacy

Treat Slack message text as untrusted data placed in Jev state. It is data, not policy. Never interpolate it into instructions or criteria. Keep scope text operator-controlled and review who can change it.

Before production traffic:

- complete a vendor and data-handling review;
- determine whether message content includes regulated, confidential, or residency-constrained data;
- verify the region, retention terms, and zero-retention option in the contract;
- store the API key in the existing secret manager;
- inject it at runtime as an environment variable;
- allow only the necessary outbound HTTPS destination;
- define retention, access control, and deletion behavior for paired decision logs.

If data terms change, switch Jev off until the review is complete.

---

## 12. Roll out in phases

Each phase has a narrow purpose and an observable exit criterion.

| Phase | Actions | Exit criteria |
| --- | --- | --- |
| 0. Prepare | Complete data review, put the key in the secret manager, pin the model, add the kill switch. | Approval complete; key deployed; `off` tested. |
| 1. Labels | Add decision logging and feedback controls; mine history; improve the evaluation script. | At least 800 adjudicated rows and a measured LLM-gate baseline. |
| 2. Offline evaluation | Run the LLM gate and Jev atomic questions, with and without scope, on the frozen set; calibrate each question. | Jev's confident band matches the baseline recall at meaningful coverage, for example 40% or more. |
| 3. Shadow | Set `GATE_JEV_MODE=shadow` for one or two tenants, then expand; review disagreements daily. | At least 1,000 paired decisions and no systematic miss class. |
| 4. Audit | Set `GATE_JEV_MODE=audit` for LLM `NO` cases; send likely misses to reviewers. | Flag precision is useful enough to justify reviewer attention. |
| 5. Pre-filter (optional) | Short-circuit only calibrated, high-confidence `YES`; keep the LLM gate for every other result. | Miss rate and p95 latency are no worse than baseline. |
| 6. New jobs | Add triage, answer judging, duplicate detection, or tags with their own evaluation. | Each new job has a task-specific success metric and fallback. |

### When to stop or roll back

Use `GATE_JEV_MODE=off` if any of the following occurs:

- availability falls below the service's error budget;
- the provider's data terms change;
- the pinned model version is deprecated or unexpectedly resolves differently;
- shadow-mode recall on true `YES` is materially worse than the LLM gate;
- a systematic high-confidence miss class appears;
- a question, scope, or threshold change has not been re-evaluated.

An additive integration is successful when turning it off is uneventful.

---

## 13. Appendix: implementation checklist

### Before the first request

- [ ] Complete data handling, privacy, and residency review.
- [ ] Store the Jev API key in the existing secret manager.
- [ ] Pin `GATE_JEV_MODEL=jev-1.13.0`.
- [ ] Set `GATE_JEV_MODE=off` by default.
- [ ] Configure `GATE_JEV_TIMEOUT_S=1.5` and no request-path retries.
- [ ] Add the pinned `typesafe-sdk` dependency.
- [ ] Define a singleton Jev client boundary that never raises.
- [ ] Confirm outbound HTTPS access to the service endpoint.

### Before shadow mode

- [ ] Add an explicit Choice escape hatch, such as `none_of_these`.
- [ ] Split the broad relevance question into atomic questions.
- [ ] Version questions, criteria, scope representation, and combination logic.
- [ ] Keep message text and operator-controlled scope separate in state.
- [ ] Implement neutral handling for empty text, timeouts, rate limits, and SDK errors.
- [ ] Record model ID, role, probability distributions, latency, and error class.
- [ ] Verify shadow mode returns the LLM gate result in every test.
- [ ] Add retention and access controls for paired decision logs.

### Before any pre-filter

- [ ] Assemble at least 800 adjudicated, representative messages.
- [ ] Measure the current LLM gate on the same frozen set.
- [ ] Inspect recall, precision, calibration, coverage, and tenant/language slices.
- [ ] Compare paired predictions and inspect every important disagreement class.
- [ ] Calibrate each atomic question and the final combined probability.
- [ ] Define asymmetric YES and NO thresholds with an explicit fallback path.
- [ ] Confirm all unknown, error, timeout, and uncertain results call the existing LLM gate.
- [ ] Run shadow mode for at least two weeks or roughly 1,000 top-level messages.
- [ ] Test that `GATE_JEV_MODE=off` restores the original bot behavior.

---

## 14. Sources to verify

This guide is intentionally specific about claims that should be checked against primary sources and your own measurements.

- TypeSafe AI documentation for models, `jev-1.13`, the API and SDK, model behavior, pricing, data handling, and legal terms.
- TypeSafe AI's availability, retention, and enterprise data-processing commitments.
- Independent Jev write-ups and tests, including Flavio Copes' deep dive; Beri.net's investigation of broad versus atomic questions, phishing, and calibration; and the `PriorBench/jev` and `Running-Dolphins/jev-bench` repositories.
- Integration examples from Langfuse, DevelopersIO, AWS Builder Center community posts, and other implementation reports. Treat them as patterns to reproduce locally, not as evidence that your deployment will behave the same way.
- Chen et al., *FrugalGPT* (arXiv:2305.05176), for the cascade framing.
- Ong et al., *RouteLLM* (arXiv:2406.18665), for router evaluation and routing trade-offs.

The practical standard is higher than a source list: benchmark the pinned version, questions, criteria, tenant scope, latency, and cost on your own Slack messages. Jev can become a valuable typed-decision layer, but only when the system around it makes uncertainty visible and keeps the current agent safe when that layer is wrong or unavailable.
