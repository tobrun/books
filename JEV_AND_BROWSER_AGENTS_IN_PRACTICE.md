# Jev and Browser Agents in Practice

## A Practical Book on Typed Decisions, Browser-Use Agents, and Live-Web Retrieval

Most teams that add an AI agent to a browser, a Slack channel, or a search pipeline reach for the same tool first: a large language model that reads context and writes text.
That works, but it is the wrong shape for most of the decisions inside an agent loop.
Deciding which DOM element to click, whether a cached answer is still fresh, or whether a page is safe to read into context are not writing tasks.
They are typed, bounded, repeatable decisions, and they happen dozens of times per task.

This book is about building agent systems that use the right tool for each layer of the loop.
It uses Jev, the flagship model from TypeSafe AI, as the running example of a fast, cheap, calibrated decision primitive, and it uses the open-source browser-use and computer-use ecosystem as the running example of an execution layer that needs exactly that kind of primitive.
The two halves of the book fit together: Chapters 1 to 3 explain what Jev is and why "typed decisions instead of text" matters.
Chapters 4 to 6 survey the browser-use and computer-use landscape and show where Jev's typed primitives slot into an agent's control loop.
Chapters 7 to 10 build three integration patterns end to end: general browser automation, a Slack agent with live search, and runtime resolution for volatile facts.
Chapters 11 to 14 cover the parts that decide whether any of this survives contact with production: security, benchmark literacy, adoption evidence, and a rollout plan.

A note on timing. Jev launched on September 15, 2026, one week before this book was drafted.
Treat every performance and cost figure in this book as vendor-reported and directional, not independently verified.
The engineering patterns are durable even where the specific numbers are not; re-run every benchmark that matters to a decision on your own tasks before you trust it.

---

## Contents

1. [The shape of the problem](#1-the-shape-of-the-problem)
2. [What Jev actually is](#2-what-jev-actually-is)
3. [The three typed primitives](#3-the-three-typed-primitives)
4. [The open-source browser-use landscape](#4-the-open-source-browser-use-landscape)
5. [Computer-use agents and managed browser infrastructure](#5-computer-use-agents-and-managed-browser-infrastructure)
6. [Where Jev fits inside a browser loop](#6-where-jev-fits-inside-a-browser-loop)
7. [Pattern A: general browser automation](#7-pattern-a-general-browser-automation)
8. [Pattern B: a Slack agent with live search](#8-pattern-b-a-slack-agent-with-live-search)
9. [Pattern C: runtime and live-web resolution](#9-pattern-c-runtime-and-live-web-resolution)
10. [The MCP server as the integration boundary](#10-the-mcp-server-as-the-integration-boundary)
11. [Security: prompt injection and the confused deputy](#11-security-prompt-injection-and-the-confused-deputy)
12. [Reading benchmarks without fooling yourself](#12-reading-benchmarks-without-fooling-yourself)
13. [Evidence from production deployments](#13-evidence-from-production-deployments)
14. [A fourteen-week rollout plan](#14-a-fourteen-week-rollout-plan)
15. [Appendix: working templates](#15-appendix-working-templates)

---

## 1. The shape of the problem

### 1.1 Agents make far more decisions than they write sentences

A browser-use agent completing a ten-step task on a web form makes roughly ten to thirty micro-decisions: which element to click, whether to scroll, whether the page has finished loading, whether the last action succeeded, whether to retry or give up.
Almost none of these decisions require prose.
They require picking one option from a short, well-defined list, or scoring how confident the agent should be that a condition holds.

A general-purpose LLM can make all of these decisions.
It does so by generating a token sequence, parsing it back into structure, and hoping the model did not paraphrase its way out of the schema.
That approach carries three costs that compound across a long agent run: latency, because autoregressive generation is sequential; price, because you are paying for a frontier model's full reasoning stack to answer a multiple-choice question; and reliability, because free-text output can drift out of the expected shape under load, especially with unusual page content in context.

### 1.2 A decision layer distinct from a reasoning layer

The pattern this book develops is a two-layer agent architecture.

```text
Reasoning layer (frontier LLM)
  - understands the user's goal
  - plans the overall approach
  - writes free text when free text is the actual deliverable
        ↓
Decision layer (typed, calibrated, fast)
  - picks the next action from a bounded set
  - scores confidence, freshness, and risk
  - verifies claims against evidence
  - reranks candidates
        ↓
Execution layer (deterministic code + tools)
  - clicks, types, navigates, fetches, writes
  - owns retries, budgets, and stop conditions
```

The reasoning layer sets the goal once, or occasionally re-plans when the decision layer reports it is stuck.
The decision layer runs many times per task, cheaply and quickly, and only escalates to the reasoning layer when its own confidence is low.
The execution layer never trusts model output directly; it validates every action against the actual, observed state of the world before running it.

This is the same "cascade" idea used in classical machine learning systems: a cheap, calibrated classifier handles the easy majority of cases, and an expensive model handles the hard minority.
Jev is built specifically to be the cheap stage of that cascade.
Browser-use and computer-use agents are the execution layer that most urgently needs one.

### 1.3 What this book will and will not claim

This book will not claim that Jev or any browser-use framework is "solved."
The best published score on the hardest write-heavy browser benchmark available at the time of writing is 64.4%.
Authentication walls, not reasoning quality, are the most common reason browser agents fail in practice.
Every pattern in this book is built around those limits rather than around the marketing copy that surrounds them.

### Lab: name your own decision points

Before writing any code, take one workflow you want to automate and list every point where the agent must choose between a small number of options, or judge whether something is true, fresh, or safe.
Write it as a table with three columns: decision, options or scale, and how wrong an answer would hurt.

**Done when:** you have at least five rows, and at least one row where a wrong decision would be expensive enough to justify human review rather than full automation.

---

## 2. What Jev actually is

### 2.1 Company and positioning

TypeSafe AI is a San Francisco lab that came out of stealth on September 15, 2026 with a $40 million seed round led by DCVC.
It was founded in 2024 by CEO Diogo Almeida, a former OpenAI researcher and a co-inventor of RLHF, InstructGPT, and ChatGPT, together with Erik Gafni and Sasha Sheng.
Jev, its first public model, was trained exclusively on synthetic data during roughly two years in stealth.

TypeSafe's founding argument is worth stating plainly, because it explains every architectural choice that follows: language models were shaped by RLHF and RLVR to produce text that humans prefer to read.
That is the right optimization target for a chat assistant.
It is the wrong optimization target for software, which needs decisions that are typed, calibrated, and cheap to make thousands of times a second.
TypeSafe's pitch is that if AI is going to change how software gets built, code needs to be able to consume intelligence directly, not only through a human-facing chat interface.

### 2.2 A "System One Model"

TypeSafe calls Jev a System One Model, a reference to Daniel Kahneman's description of fast, intuitive, non-deliberative human cognition.
The name is a claim about scope, not a claim about intelligence: Jev is meant for the fast, pattern-matching judgment calls that make up most of an agent's decision volume, not for open-ended reasoning or long-horizon planning.

Jev is trained with what TypeSafe calls Reinforcement Learning for Calibrated Decisions (RLCD).
The three architectural properties that follow from this training approach are the ones that matter for integration work:

- **Non-autoregressive, parallel sampling.** Jev produces its output in a single pass instead of token by token. This is the direct source of its latency advantage; there is no sequential decoding loop to wait on.
- **Typed output only.** Jev returns a choice, a score, or a probability, matched against a schema you provide, never free text. Because it cannot generate strings, it cannot hallucinate a value outside the schema and it cannot produce a malformed response that breaks a downstream parser.
- **Calibrated confidence on every answer.** Each response carries a probability distribution over the possible answers, not just a single winner. This is what makes the cascade pattern from Chapter 1 practical: code can act immediately when confidence is high, and escalate to a slower, more capable model when it is not.

### 2.3 Performance and pricing, as reported by TypeSafe

All of the following figures are vendor-reported at launch and have not been independently reproduced at the time of writing.

| Property | Reported figure |
| --- | --- |
| Latency | 70 to 500 milliseconds end to end |
| Input price | $0.042 per million tokens |
| Output price | Free ("too cheap to meter") |
| Speed comparison | Up to 193.6x faster than reference frontier models on TypeSafe's workflow evals |
| Cost comparison | Up to 444.6x cheaper on the same evals |
| Intelligence comparison | Benchmarked against the average of GPT-6 Astra and Fable 5.1 on "System One-shaped" tasks |

Treat the multiplier claims (100x, 200x, 400x, depending on which write-up you read) as marketing framing for the same underlying fact: a non-autoregressive, single-pass, typed-output model is going to be dramatically faster and cheaper than an autoregressive frontier model for a bounded classification or ranking task, because it is doing meaningfully less work per call.
The interesting question for an integration decision is not the multiplier.
It is whether Jev's calibrated confidence actually matches a frontier model's judgment on your specific decisions, which Chapter 12 covers.

### 2.4 Access

Jev is in waitlisted early access at typesafe.ai, with API keys issued through console.typesafe.ai.
It is also available through an OpenRouter alpha integration with pinned model versions (for example, `jev-latest` resolving to `typesafe/jev-1.13`), through Cloudflare Workers AI as `typesafe/jev`, and through a LangChain integration (`langchain-typesafe`, exposing a `TypeSafeClassifier`).
At launch, TypeSafe serves from a single region on the U.S. West Coast.
If your traffic originates elsewhere, measure real round-trip latency from your own infrastructure before you build a latency-sensitive path around Jev; a 70 millisecond model call can easily become a 250 millisecond round trip once cross-region network time is added.

The name "Jev" is a reference to the economist William Stanley Jevons and the Jevons paradox, the observation that making a resource more efficient tends to increase, not decrease, total consumption of it.
That is a claim about what happens to demand for typed decisions once they become nearly free, and it is worth remembering as you design a system: cheap decisions invite you to make more of them, not fewer.

### Lab: get a Jev key and make one call

Sign up for early access, and once you have a key, make a single classification call against a toy example: five short product reviews, and a Choice question asking whether each is positive, negative, or mixed.
Record the latency and the returned confidence for each.

**Done when:** you have a real latency number from your own network path, not the vendor's number, and you have looked at what a low-confidence response looks like, not just a high-confidence one.

---

## 3. The three typed primitives

Jev exposes exactly three question types.
Every decision you route through it must be expressed as one of these three, which is itself a useful design discipline: if you cannot phrase a decision as a Choice, a Score, or a Noul, it probably belongs in the reasoning layer, not the decision layer.

### 3.1 Choice

A Choice question asks Jev to pick one option out of up to 255 candidates, given the current state.
The response includes the selected choice, a probability for every candidate, and an overall confidence score.

```yaml
question_type: choice
state: |
  Page: checkout, step 2 of 3.
  Element table:
    12: button "Continue to payment"
    13: link "Edit shipping address"
    14: checkbox "Save this address" (unchecked)
    15: link "Return to cart"
question: "Which element should be interacted with to proceed to payment?"
options: [12, 13, 14, 15]
```

This is the exact shape used by `jev-ultrafast` to pick the next browser action: the agent serializes the observable DOM into a numbered element table, and Jev returns which element index to act on along with the confidence that this is the right move.

### 3.2 Score

A Score question asks Jev to place the state on a fixed rubric, typically 2 to 10, along with a confidence value.
This is the right primitive for graded judgments: how relevant is this search result, how close is this draft to done, how risky is this action.

```yaml
question_type: score
state: |
  Query: "refund policy for annual plans"
  Candidate: "Cancellation and Refunds - Monthly Plans" (support article, last updated 14 months ago)
question: "How relevant is this candidate to the query, on a 2-10 scale?"
```

### 3.3 Noul

A Noul question asks Jev whether a statement is true of the current state, returning a probability between 0 and 1 rather than a hard boolean.
This is the primitive for verification and freshness checks: is this claim supported by this evidence, is this cached answer still fresh enough, is this page safe to read into context.

```yaml
question_type: noul
state: |
  Cached answer (fetched 6 hours ago): "Flight AA118 departs at 14:20 from gate C22."
  Freshness policy for flight status: stale after 15 minutes.
question: "Is this cached answer still fresh enough to answer the user's question?"
```

### 3.4 Speculative fan-out

Because Jev evaluates every question against the same state in a single parallel pass, you are not penalized for asking several related questions in one request.
`jev-ultrafast` uses this directly: instead of first deciding the action type and then, in a second round trip, deciding the target, it asks "what would the CLICK target be," "what would the TYPE_TEXT target be," and "what would the SELECT target be" all at once, then keeps only the answer that matches the action type it also asked for in the same call.
This collapses what would be a two-step decision into one round trip, at the cost of computing a few answers you discard.
Given that output tokens are free and latency is the scarce resource, that trade is usually worth making.

### 3.5 Composing primitives into a decision

A single agent step rarely needs just one primitive.
The reference pattern used by `jev-browser` is three questions per step, asked together:

1. **Choice** - which action to take next.
2. **Noul** - has the overall goal been reached.
3. **Noul** - is the agent stuck (repeating the same failed action, or blocked by an unexpected state).

Code then applies simple thresholds: stop and report success when the goal-Noul exceeds roughly 0.85, stop and escalate when the stuck-Noul exceeds roughly 0.85, otherwise execute the chosen action and loop.
None of this logic lives inside the model.
It lives in your code, which is exactly where budgets, retries, and stop conditions belong.

### Lab: rewrite your decision table as typed questions

Take the decision table from the Chapter 1 lab and rewrite each row as a Choice, a Score, or a Noul question, including the state you would need to supply and the options or scale.

**Done when:** every row maps cleanly onto one primitive. If a row does not fit any of the three, write one sentence explaining why it belongs in the reasoning layer instead.

---

## 4. The open-source browser-use landscape

Jev needs an execution layer to be useful for browser work, and that layer already has a mature, if bounded, open-source ecosystem.
The projects in this space split into four architectural shapes, distinguished by one question: who owns the control loop, the framework or your code?

| Project | License | Who owns the loop | Primary observation | Best published score |
| --- | --- | --- | --- | --- |
| Browser Use | MIT | The framework | Serialized interactive elements, vision optional | 63.3% on its own hard suite (the open-source library, not the managed cloud) |
| Skyvern | AGPL-3.0 core | The workflow engine | Vision-first with grounding | 64.4% on WebBench, 85.8% on WebVoyager |
| Stagehand | MIT | You, with AI-backed steps | DOM-oriented, resolved actions cached to selectors | Roughly 65% on Online-Mind2Web with a strong underlying model |
| Playwright MCP | Apache-2.0 | Your existing agent | Accessibility tree | Not applicable; this is tooling, not an agent |

### 4.1 Browser Use: agent-owns-the-loop, maximum ecosystem

Browser Use is the category leader by adoption, with roughly 108,000 GitHub stars and 11,900 forks at the time of writing, and it is the project TypeSafe partnered with to ship `jev-ultrafast`.
You hand it a goal in natural language, and the framework's own loop decides, step by step, what to do next, using a serialized table of interactive DOM elements as its primary observation.
Vision is optional and adds cost; most tasks do not need it.

The honest number to remember is 63.3%, the score of the open-source, pip-installable library on Browser Use's own hard benchmark suite.
The frequently repeated 97% figure belongs to Browser Use Cloud, the managed product, not the library you self-host.
Conflating the two is the single most common mistake teams make when evaluating this framework, and it is worth restating in Chapter 12.

### 4.2 Skyvern: vision-first, strongest on write-heavy portals

Skyvern takes a different observation strategy: it grounds actions visually rather than relying primarily on DOM structure, which makes it comparatively resilient to hostile or obfuscated markup.
Its core is licensed AGPL-3.0, a strong copyleft license; if you expose a modified version of Skyvern as a network service, that license carries real obligations, and you should get legal sign-off before adopting it in a commercial product rather than after.

Skyvern reports 64.4% on WebBench, a 5,750-task benchmark spanning 452 sites that Skyvern itself co-built with Halluminate, and 85.8% on WebVoyager.
It has the strongest native support in this survey for two of the hardest practical problems in browser automation: authentication (native TOTP handling for two-factor logins) and CAPTCHA resolution.
That makes it the natural choice for login-and-form-heavy portal work, which is also where most browser-use projects fail in practice, as Chapter 11 discusses.

### 4.3 Stagehand: you keep the loop, AI only on brittle steps

Stagehand is built directly on Playwright and is meant for teams that already have working automation and want AI applied only where the DOM is brittle or changes without notice.
You write the control flow; Stagehand resolves individual steps ("click the button that lets the user add a new payment method") using a model, and then caches the resolved action to a concrete selector.
On repeated runs of a stable workflow, the AI-resolution cost decays toward zero and the automation degrades gracefully toward plain Playwright.
This is the safest default for teams whose main goal is reducing selector maintenance, not full autonomous task completion, and it reports roughly 65% on Online-Mind2Web when paired with a strong underlying model.

### 4.4 Playwright MCP: tooling, not an agent

Playwright MCP is not a browser agent at all; it is an MCP server that exposes Playwright's browser primitives, grounded in the page's accessibility tree, as tools your existing agent orchestrator can call.
It has more GitHub stars than either agent framework above, roughly 35,900, which is a signal about how many teams already have an orchestrator and want browser capability added to it rather than a new autonomous framework bolted on.
If that describes your situation, this is the right starting point, and Chapter 10 covers exactly how to wire it in.

### 4.5 Choosing among the four

The decision is mostly about who should own the control loop, not about raw capability:

- Choose **Stagehand** if you already run Playwright and want AI applied surgically to brittle steps.
- Choose **Browser Use** for goal-level autonomy and the largest surrounding ecosystem, and accept that the honest ceiling is around two-thirds task completion without human fallback.
- Choose **Skyvern** for portal work dominated by logins, 2FA, and forms, after clearing the AGPL-3.0 licensing question with legal.
- Choose **Playwright MCP** if you already have an orchestrator and just need browser tools, not a new agent.

### Lab: pick a shape and read its quickstart

Pick the project that matches your control-loop preference, install it locally, and run its quickstart example against a site you own or are permitted to automate.
Note every place where the framework's default behavior surprised you.

**Done when:** you have completed one real task end to end and written down at least one failure mode you observed, not just the success path.

---

## 5. Computer-use agents and managed browser infrastructure

### 5.1 Screen-grounding computer-use agents

Browser-use frameworks act on structured page state.
Computer-use agents are a broader category: they act on pixels, which lets them drive any graphical interface, not only a browser.

- **OpenAI CUA** (`computer-use-preview`, via the Responses API) ships a sample app with both a Playwright-driven browser agent and a PyAutoGUI-driven desktop agent.
- **Anthropic Claude Computer Use** runs in a Docker container at a recommended resolution of 1024x768, and is explicitly labeled a beta capability.
- **ByteDance UI-TARS** and its desktop counterpart are fully open, weights included, and post the strongest open-source results on OS-level desktop benchmarks, but real-world accuracy on dense, unfamiliar UIs has been measured as low as 38% in independent comparisons, with high variance.
- **Simular Agent S3** self-reports 72.6% on OSWorld; **Cua** runs the agent inside a virtual machine rather than on your own device, which is a meaningful safety property; **Open Interpreter**'s `--os` mode can run fully local via Ollama; **Fazm** operates through the macOS accessibility tree rather than pixels, trading some generality for a real privacy advantage, since it never has to screenshot the screen.

The general pattern across independently measured benchmarks is a real gap between open and closed intelligence: on OSWorld-Verified, the strongest tracked open-weight model sits around 66.7%, while frontier closed models are reported in the mid-70s to mid-80s, an 18.7 point gap.
Most "open-source" computer-use agents are open on the agent scaffolding, not on the underlying model; check which one you are actually adopting before you count on self-hosting for either cost or data-residency reasons.

### 5.2 Managed browser infrastructure is not an agent

A separate category of vendor runs the browser for you without deciding what it does: Browserbase, Steel, Browserless, Kernel, and Firecrawl each solve infrastructure problems (fingerprint management, CAPTCHA solving, session persistence, proxying) that sit underneath whichever agent framework you choose.

Browserbase is the most widely adopted of these for agent workloads, reporting 800,000 weekly SDK downloads, more than 35 million monthly sessions, and roughly 10,000 companies using the product, figures that are self-reported and, per independent coverage, corroborated in magnitude by reporting from mid-2026.
Steel is the strongest open-source, self-hostable option if data residency or vendor lock-in rules out a managed service.
Kernel specializes in keeping authenticated, headful sessions alive across runs, which matters directly for the authentication problem in Chapter 11.
Firecrawl is worth choosing specifically when your deliverable is clean extracted web data rather than an interactive task; its Browser Sandbox product sits between a scraping API and a full agent framework.

### 5.3 Where cost actually comes from

Managed infrastructure billing looks small per unit and adds up quickly at agent scale.
Browserbase's free and Developer tiers step up to a Startup tier around $99 per month, with overage typically in the $0.10 to $0.12 per browser-hour range and proxy bandwidth around $10 to $12 per gigabyte; free CAPTCHA solving is included on every tier.
Browser Use Cloud separately lists $0.02 per browser-hour.
None of these numbers include the model cost of the decisions made during each session, which is exactly the cost Jev is positioned to reduce, because a cheap per-step decision does not change how many browser-hours you consume, but it does change how many expensive model calls you make per hour of browsing.

### Lab: estimate your own cost stack

For one representative task, estimate browser-hours, average steps per task, and the model calls per step under two designs: one where every step calls a frontier model directly, and one where a cheap decision primitive like Jev handles the routine steps and escalates only the ambiguous ones.

**Done when:** you have two total cost estimates side by side, and you can say in one sentence what fraction of steps you expect to need escalation.

---

## 6. Where Jev fits inside a browser loop

### 6.1 `jev-ultrafast`: the reference integration

`jev-ultrafast` is Browser Use's own MIT-licensed integration with Jev, and it is the clearest existing example of the two-layer architecture from Chapter 1.
Each observation is serialized into a numbered table of interactive elements, without screenshots in the default loop.
One Jev request picks both the operation (CLICK, TYPE_TEXT, SELECT, SCROLL_UP, SCROLL_DOWN, WAIT, DONE, or BLOCKED) and the target element, using the speculative fan-out technique from Section 3.4.
A small text model is invoked only when the chosen operation is TYPE_TEXT, since that is the one action that genuinely requires free-text generation.

The reported result on a representative task, a Google Flights search, is a drop from 9.45 seconds to 7.09 seconds end to end, roughly 25% faster, and a drop in underlying browser-protocol calls from 1,092 to 101, a 91% reduction.
The protocol-call reduction is the more interesting number: it suggests the gain is not primarily about model speed, but about the agent needing far fewer exploratory round trips once its per-step decisions are both fast and confident enough to commit to without re-checking.

At the time of writing, `jev-ultrafast` explicitly does not handle shadow DOM, iframes, file uploads, or multiple tabs.
Treat it as a fast reference implementation for a bounded class of tasks, not a general replacement for Browser Use's default loop.

### 6.2 `jev-browser`: code owns the loop

`jev-browser` (npm package `@jkudish/jev-browser`) takes the Stagehand-style position: your code owns the loop, budgets, and stop conditions, and Jev supplies the per-step judgment.
Each step issues one Jev call bundling the three questions from Section 3.5: which action to take, whether the goal has been reached, and whether the agent is stuck.
The library returns the final page in several formats (plain text, markdown, HTML, or the accessibility tree), a screenshot, and a full step trace with confidences, console errors, and running cost, which is exactly the audit trail Chapter 11 requires for any agent with access to sensitive systems.

A representative run reported by the project, navigating from the Wikipedia Main Page to the Ristretto article, completed in 5.9 seconds for $0.0021.
A companion project, `jev-mcp`, exposes Jev purely as judgment tools rather than as a navigation agent: verify a claim against evidence, screen page content before it enters an LLM's context, and rank or rerank a list of candidates. Chapters 8 and 9 use exactly these three tools.

### 6.3 Other community integrations

A handful of smaller projects extend the same pattern to other surfaces: `jev-ego`, a lighter-weight browser agent; `jev-for-chrome`, a Manifest V3 Chrome extension; and `typesafe-computer-use`, a macOS agent that pairs OCR with Jev classification and reports a cost around $0.0002 per step. TypeSafe also publishes a "Browser Automation for AI Agents" skill under `typesafe-ai/skills`.
None of these are as battle-tested as `jev-ultrafast` or `jev-browser`; treat them as starting points to fork, not dependencies to pin in production without your own review.

### 6.4 What Jev does not solve

Jev removes the cost and latency of the per-step decision.
It does not remove the limitations of the execution layer underneath it: authentication walls, shadow DOM, CAPTCHA, and detection all remain exactly as hard as they are for the browser framework you paired it with.
Do not let a fast, cheap decision layer create false confidence about the execution layer's actual ceiling; the two are independent variables, and Chapter 12 exists specifically to keep them separate in your evaluation.

### Lab: trace one step by hand

Using `jev-browser` or the equivalent shape, run one task and pull the full step trace for a single step: the state supplied, the three questions asked, the returned choices and confidences, and the action actually executed.

**Done when:** you can point to the exact confidence value that would have triggered escalation under an 0.85 threshold, and say whether it should have.

---

## 7. Pattern A: general browser automation

### 7.1 When browser automation is the right tool

The correct ordering of tools, from cheapest and most reliable to most brittle and expensive, is: a native API or connector first, browser automation second, and pixel-level computer use last.
Reach for browser automation only when you have confirmed there is no usable API, not as a default because it feels more general.

### 7.2 Architecture

```text
User or system goal
        ↓
Reasoning layer sets the goal and stop criteria
        ↓
Browser-agent MCP server (jev-browser, Skyvern, or Playwright MCP)
        ↓
Per-step decision (Jev Choice: action + target)
        ↓
Execution against a sandboxed, isolated browser session
        ↓
Per-step verification (Jev Noul: goal reached? stuck?)
        ↓
Structured result + full step trace back to the caller
```

### 7.3 Choosing runtime location

Run the browser locally when the target is a trusted internal tool and you want the lowest possible cost; run it in managed infrastructure (Browserbase or Steel) when you need session isolation, fingerprint management, CAPTCHA handling, or session replay for debugging.
Never run an agent-controlled browser on an operator's own workstation with their real, authenticated sessions; that collapses the isolation boundary Chapter 11 depends on.

### 7.4 Observation model

Prefer DOM or accessibility-tree observation (Browser Use, Stagehand, `jev-ultrafast`) over screenshots by default: it is cheaper in tokens, faster, and safer, because a screenshot risks exposing far more of the page, and therefore far more of any injected content, than a targeted element table does.
Fall back to vision (Skyvern, computer-use agents) specifically for hostile or malformed DOMs where structural grounding fails.

### 7.5 Determinism through caching

For any workflow you expect to run more than a handful of times, adopt Stagehand's caching strategy even outside Stagehand itself: once a model has resolved an action to a concrete selector, store that resolution and only re-invoke the model when the cached selector fails.
A high-frequency, stable workflow should decay toward the cost and reliability of plain Playwright over time, using the model layer only to repair drift when the underlying page changes.

### 7.6 Budgets and stop conditions belong in code

Never let the model decide when to stop trying.
Set an explicit step budget, a wall-clock budget, and a cost budget per task in code, and treat exceeding any of them as a hard stop that returns the partial trace, not a silent retry loop.
A stuck-Noul confidence above threshold is a signal to your code, not a decision the model makes on your behalf.

### Lab: build one end-to-end automation

Pick one real internal workflow with no usable API (for example, checking order status on a portal you own), and build it using the architecture above: a browser-agent MCP server, Jev for per-step decisions, a sandboxed session, and explicit budgets in code.
Run it ten times and record the completion rate.

**Done when:** you have a completion rate from real runs, not an estimate, and a written stop condition for what happens on the runs that fail.

---

## 8. Pattern B: a Slack agent with live search

### 8.1 Slack as an agent surface

Slack now ships a Slackbot MCP Client, an Agent Kit, and Block Kit for interactive components, and it routes agent requests while respecting Slack's own native compliance and permission model.
The sanctioned integration path is the official Slack MCP server: remote, OAuth-based, admin-governed, with granular scopes and audit logs.
A community alternative, `korotovsky/slack-mcp-server`, can authenticate using a browser session token to bypass the admin-approval step; that is convenient for a personal prototype and is not appropriate for an enterprise deployment, since it routes around the exact governance controls the official server provides.
Note separately that the older `@modelcontextprotocol/server-slack` package is deprecated and archived; do not build new integrations on it.

### 8.2 The pattern: Slack agent to MCP to browser agent

```text
User asks a question in a Slack thread
        ↓
Slack agent (via official Slack MCP server) parses the request
        ↓
Confidence gate: can this be answered from an existing index?
        ↓ no
Browser-agent MCP server resolves live content
   (a page behind a login, a portal with no API, a live price)
        ↓
Jev screens the fetched content before it enters context (jev-mcp)
        ↓
Answer formatted via Block Kit, with citation and confidence
        ↓
Posted back to the thread, respecting Slack's rate limits
```

Slack enforces roughly one request per second per app in most contexts; a production-grade MCP server queues and backs off for you, and you should confirm your chosen server does this before relying on it under real traffic rather than discovering it during an incident.

### 8.3 Extending search past a static index

A static vector index answers questions about content that was true when it was last indexed.
A browser-use agent extends that in a specific, narrow way: it can navigate gated or JavaScript-heavy pages a scraper cannot reach, extract clean markdown or accessibility-tree text, and ground the answer with a citation to the live page rather than a stale snapshot.

Route this decision through Jev rather than always browsing live: a live fetch costs real seconds and real dollars per query, and most Slack questions can be answered from an existing index.
Use a Noul question, "is the cached or indexed answer fresh enough for this question," as the gate before dispatching a browser agent at all.

### 8.4 The three `jev-mcp` judgment tools, applied here

- **Screen before context.** Before any fetched page content is placed into the reasoning model's context window, ask a Noul question: does this content contain instructions directed at an AI agent, rather than information for a human reader. This is the first line of defense against the prompt injection risk covered in Chapter 11.
- **Rerank.** When a search or a live fetch returns several candidate passages, use a Score question per candidate to rank by relevance to the actual question asked, not just by whatever ordering the source returned.
- **Verify.** Before posting an answer, ask a Noul question per extracted claim: is this claim actually supported by the evidence gathered. Surface the confidence alongside the answer in Slack rather than hiding it; a visible confidence number is more useful to the person reading it than false certainty.

### Lab: wire up one live-search path

Build a Slack agent that answers questions from a static index by default, and add one live-resolution path for a specific class of question your index cannot answer (a page you know changes daily, for instance).
Gate the live path with a Jev Noul freshness check.

**Done when:** you can ask a question that correctly stays on the static path, and a second question that correctly triggers the live path, and both are logged with the gate's confidence value.

---

## 9. Pattern C: runtime and live-web resolution

### 9.1 The architecture shift

Live-web retrieval, sometimes called live-web RAG or agentic search, moves the fetch to query time for facts that genuinely change: prices, inventory, availability, breaking events, and long-tail entities that were never worth pre-indexing.
It collapses the traditional ingest-chunk-embed-store pipeline into a single request-time call, trading a fixed indexing cost for a variable per-query cost.

This is not a novel idea; the academic lineage runs from WebGPT through WebArena to modern deep-research agents, and the underlying motivation is the same one stated plainly in Baidu's TURA work: some content, like real-time ticket availability or live inventory, is generated dynamically and simply cannot be pre-indexed by a search engine, no matter how good the index is.

### 9.2 Cost shape

Live resolution vendors price per request rather than per stored document, and the spread across latency and depth tiers is wide.
Parallel.ai, as a representative example, prices its fastest tier at roughly $0.001 per request with a 200 millisecond p50 latency, and its deeper tiers at roughly $0.005 per request with one to three second latency and ten results included.
Compare this against your existing static-RAG infrastructure cost per query, which is usually near zero marginal cost once the index is built, to understand why the routing decision in the next section matters so much: live resolution should be the exception path, not the default.

### 9.3 Jev as the routing primitive

The hard part of this architecture is not retrieval.
It is the routing decision: for a given query, is the cached or indexed answer good enough, or does this need a live browse.
That decision has exactly the shape Jev is built for: bounded, repeatable, and needed on every single query.

```text
Query arrives
        ↓
Jev Choice/Noul: static index, or live resolve? (~100ms, sub-cent)
        ↓ static                                    ↓ live
Vector or keyword store                    Browser-agent MCP server
        ↓                                            ↓
                                            Jev screens and reranks the result
        ↓                                            ↓
             Grounded answer with citation and confidence
                        ↓
                Cache with a per-topic freshness TTL
```

### 9.4 Setting freshness TTLs

A single global freshness TTL is almost always wrong, because different facts decay at different rates.
Set TTLs per topic, not per system: a flight status might be stale after 15 minutes, a product price after a day, a company policy document after a quarter.
Store the TTL as metadata alongside the cached answer, and let the Jev freshness Noul from Section 3.3 read it directly rather than hardcoding a single threshold in application code.

### 9.5 A note on "poi-factory"

If your organization uses an internal tool or workflow referred to as "poi-factory" for point-of-interest search, be aware that no public browser-use or live-search tool by that name could be found during the research for this book.
The public matches for the string are unrelated: a consumer GPS points-of-interest file-sharing community, the Apache POI Java library for Office file formats, an unrelated Excel export utility, and a fan-made game browser.
If "poi-factory" is meaningful in your context, it is an internal name; document what it actually refers to in your own system rather than assuming it maps to any of these open-source or commercial products.

### Lab: build the routing gate

For a query type you care about, implement the Jev routing decision from Section 9.3 against a stub static index and a stub live-fetch path.
Set a per-topic freshness TTL and confirm the gate correctly routes both a fresh-cache case and a stale-cache case.

**Done when:** you can show one query taking the cheap path and a near-identical query, differing only in cache age, taking the live path.

---

## 10. The MCP server as the integration boundary

### 10.1 Why MCP is the consistent pattern

Every major project in this survey, `jev-browser`, Skyvern with its 35-tool server, Playwright MCP, and Browser Use, ships as or alongside an MCP server.
That convergence is not an accident: MCP is exactly the boundary that lets a Slack agent, a coding agent, or an internal orchestrator attach browser capability without embedding a specific framework's control loop into every caller.

Treat the MCP server as the seam where you enforce scope, not as a transparent pipe.
Every tool you expose through it should do one narrow thing well, with an explicit, auditable contract, rather than exposing a general "run this JavaScript" or "execute this command" escape hatch.

### 10.2 A narrow tool contract, not a remote shell

```yaml
tool: browser_fetch_page
description: >
  Navigate to a URL and return its content as markdown.
  Read-only. Does not submit forms, click, or authenticate.
inputs:
  url: string
outputs:
  markdown: string
  fetched_at: timestamp
  screened: boolean   # true once jev-mcp has run the injection-screen check
scope: read-only, no credentials attached
```

```yaml
tool: browser_complete_task
description: >
  Complete a bounded, named workflow (e.g. "check_order_status")
  using a pre-authorized, scoped session. Does not accept free-text
  goals; only a fixed set of named workflows.
inputs:
  workflow: enum [check_order_status, submit_support_ticket]
  parameters: object   # validated against the workflow's own schema
outputs:
  result: object
  step_trace: array
scope: write, single named workflow, session pre-authenticated and time-boxed
```

The second tool is deliberately more restrictive than "give the agent a browser and a goal in natural language."
That restriction is the point: a named, schema-validated workflow is auditable and reviewable in a way that an open-ended natural-language goal is not, and it is the difference between an automation you can put in front of a compliance review and one you cannot.

### 10.3 Where budgets and audit trails live

Put step budgets, cost budgets, and timeouts in the MCP server's own code, not in the prompt.
Log every step's proposed action, the confidence that produced it, and the action actually executed, even when they match, because the moments they diverge are exactly what you need during an incident review.

### Lab: write one narrow tool contract

Write the YAML contract for one MCP tool you actually plan to expose, following the shape above.
Have a colleague read only the contract, without any other context, and ask them what they think the tool could be tricked into doing.

**Done when:** you have tightened the contract at least once based on that question.

---

## 11. Security: prompt injection and the confused deputy

### 11.1 The dominant threat is not the browser sandbox

A browser-use agent reads untrusted page content directly into its context window.
Every page it visits is therefore an input channel to the model, not just a source of data, and a malicious page can attempt to hijack an agent that is carrying an authenticated session, a pattern security researchers call the confused deputy: an agent with privileged access, exposed to untrusted input, that also has the ability to take real actions.
Google's Threat Intelligence Group reported a 32% relative increase in malicious indirect-injection content across Common-Crawl-scanned pages between November 2025 and February 2026, which suggests this is an actively growing attack surface, not a theoretical one.

Process-level sandboxing, running the browser in a container or VM, does not by itself stop this attack.
Sandboxing protects the host machine if the browser process is compromised; it does nothing to stop the agent's own reasoning from being manipulated by text it reads on a page, because that manipulation happens inside the model, not inside the browser process.

### 11.2 Consensus mitigations

- **Least-privilege tool scopes.** Constrain what the agent can do at the tool layer, using the narrow contracts from Chapter 10. Do not rely on the agent choosing to refuse an instruction it reads on a page; assume it will not always refuse, and make the refusal unnecessary by removing the capability.
- **Isolation.** Run every session in an isolated container, VM, or microVM, separate from any host that holds credentials the agent does not explicitly need for the task at hand.
- **Plan-then-execute separation.** Keep the user's trusted intent and your system's policy separate from untrusted page content in how you construct prompts, so that text read from a page is clearly marked as data, never as an instruction the model should follow. Reject under-specified requests rather than guessing, and grant no permissions by default.
- **Human in the loop for authentication.** Use pre-seeded session state, native TOTP support where available (Skyvern), or a remote-assist handoff where the agent pauses and a human clears a 2FA or CAPTCHA challenge before the agent resumes. Do not store or type a password into a form field via the agent; treat credential entry as a boundary the agent does not cross.
- **Observability and audit.** Keep the full step trace from Chapter 10 for every run: proposed action versus executed action, confidence at each step, console and network errors, and any anomaly such as a "summarize this page" task that suddenly attempts to open an email or admin interface.
- **Output handling.** Never let raw model output become a selector, a coordinate, a shell command, or injected JavaScript directly. Validate every target against the actual, currently observed DOM node before acting on it, and re-check page freshness and visual occlusion immediately before executing, which is what `jev-ultrafast` does by design.

### 11.3 Jev-specific defaults worth keeping

The TypeSafe SDK blocks browser use by default when a client-side API key is exposed, because a browser-exposed key combined with browser-control capability is exactly the confused-deputy shape described above; run both the model call and the browser server-side, never from a client the end user's browser can inspect.
`jev-browser` never offers password or file-upload inputs as valid targets for its own action selection, which removes an entire class of credential-handling mistakes from the decision layer by construction rather than by policy.

### 11.4 Authentication is a staffing decision, not just an engineering one

If a workflow hits a real login wall on every single run, with no way to pre-seed or persist a session, you do not have an automation problem; you have a staffing plan wearing an automation costume.
Re-scope the workflow so that authentication happens once, infrequently, and with a human present, and let the agent operate inside that already-authenticated session for the bulk of its work.
If the target system is not one you own or are contractually permitted to automate, stop; that is a compliance question, not an engineering one, and no amount of sandboxing changes the answer.

### Lab: red-team your own tool contract

Take the MCP tool contract from the Chapter 10 lab, and write one page of untrusted content designed to make an agent using that tool do something it should not, for example instructing it to navigate somewhere out of scope. Run it against your actual implementation in an isolated environment.

**Done when:** either the attack fails because of a scope you already enforced, or you have found a real gap and fixed it before moving on.

---

## 12. Reading benchmarks without fooling yourself

### 12.1 The cloud-versus-library trap

The single most common evaluation mistake in this space is comparing a managed cloud product's number against a self-hosted library's number as though they measured the same thing.
Browser Use's 97% figure belongs to Browser Use Cloud; the pip-installable library scores 63.3% on the same class of task.
Before you cite any number from a vendor's page, confirm which deployment it was measured on, and assume the gap is real until you have reproduced it yourself.

### 12.2 Benchmark provenance matters as much as the score

Skyvern's 64.4% figure comes from WebBench, a benchmark Skyvern itself co-authored.
That does not make the number false, but it does mean you should treat it the way you would treat any vendor grading its own homework: informative about relative capability, not a substitute for your own held-out evaluation.
Different browser-agent benchmarks use different suites, different judges, and different harnesses, so a score from one benchmark is not directly comparable to a score from another, even when both are reported as a single percentage.

### 12.3 The 200-run eval

Before committing to any framework, build your own evaluation: roughly 20 representative internal tasks, each run 10 times, scored on completion.
That is about 200 runs, roughly a day of engineering time, and it is the only number in this book you should actually trust for your own decision, because it is measured on your own sites, your own auth flows, and your own failure modes.
Set a concrete threshold before you run it, not after: proceed if you clear roughly 70% unattended completion, or if you have a clean human-in-the-loop fallback for the remainder; do not retroactively lower the bar because the number came in lower than the vendor's marketing page implied.

### 12.4 Evaluating Jev specifically

Before routing a real decision through Jev in place of an existing classifier or a frontier-model call, build a held-out set of that decision, labeled by your own frontier model or by a human reviewer, and confirm Jev's calibrated confidence actually tracks correctness on that set: high-confidence answers should be right far more often than low-confidence ones, not just right on average.
Only promote Jev to replace an existing decision path once it matches your reference on that held-out set; do not adopt it on latency and cost numbers alone, since a fast, cheap, wrong decision is worse than a slow, expensive, correct one for anything with real consequences.

### Lab: run your own 200-run eval

Pick the framework you selected in the Chapter 4 lab, define 20 representative tasks, and run each 10 times.
Score strictly on completion, and separately note every failure's cause: login wall, missing element, timeout, or wrong action.

**Done when:** you have a real completion percentage, a breakdown of failure causes, and a written decision, proceed or not, made against the threshold you set before you started.

---

## 13. Evidence from production deployments

The deployments below are drawn from vendor and customer marketing pages, chiefly Browserbase and Browser Use, and from the metrics those companies report about their own customers.
None of the figures here have been independently audited; read them as evidence that this pattern works somewhere in production, not as a guarantee it will work at your scale.

| Company | Domain | Reported pattern | Reported outcome |
| --- | --- | --- | --- |
| Ramp | Fintech | Procurement and receipt-fetching agents | Customers report automating 5M+ receipts per month; Browserbase reports saving customers 4,200+ hours per month |
| Commure | Healthcare | Insurance claims reconciliation across payer portals | 20x more claims processed daily; 8,000+ hours automated in three months; HIPAA-compliant with a signed BAA |
| Magicare | Post-acute healthcare | SNF admissions across 13 portals and 60 Epic instances | Decisions that took 60 minutes now take under 60 seconds; 200,000+ referrals processed |
| Amplitude | Analytics | Auto-built sales-demo environments via simulated user journeys | 40+ demo environments, 5,600+ simulated journeys, $10M+ attribution-based influenced pipeline |
| Numeral | Sales-tax compliance | Logs into dozens of state tax portals to file returns | Raised a $35M Series B at a $350M valuation in September 2025, led by Mayfield |
| Chronicle | Legal | Social Security disability case monitoring | 100,000 sessions per month; migrated off a self-hosted Selenium Grid |
| Vercel | Internal tooling | "Prism" monitors customer sites for adoption and churn signals | Feeds a daily internal Slack channel |

Two of these are worth reading carefully rather than at face value.
Amplitude's "$10M+ influenced pipeline" figure is attribution-based, the softest metric in this table, and should not be treated as directly measured revenue.
Numeral's fundraising figure is a real, third-party-reported number (via TechCrunch) and is useful as a signal that investors believe the underlying automation works at commercial scale, even though it says nothing directly about task-completion rate.

Skyvern, by contrast, publishes only aggregate figures (30,000-plus users, 10 million-plus workflows, 500-plus enterprise teams across healthcare, insurance, fintech, and HR) without naming individual enterprise customers, which makes it harder to sanity-check any single claim against a named deployment.

### Lab: find your own comparable

Look for one deployment in the table above that resembles your own use case in domain or workflow shape, and read the primary source, not just the summary here.
Write down which parts of their reported outcome you believe transfer to your context, and which parts (industry, data sensitivity, portal count) do not.

**Done when:** you have a one-paragraph note distinguishing what you are borrowing as evidence from what you are borrowing as aspiration.

---

## 14. A fourteen-week rollout plan

This plan sequences the patterns in this book so that each stage produces the evidence the next stage needs, rather than committing to an architecture before you have data.

### Stage 0: Framing (before week 1)

Adopt the connector-first, browser-second, screen-last ordering from Chapter 7 as written policy, not just convention.
Treat Jev and browser-use agents as complementary: Jev is the decision layer, browser agents are the execution layer of last resort.

### Stage 1: Pilot the MCP-wrapped browser agent (weeks 1-6)

Pick a framework using the guidance in Section 4.5.
Run every session in managed isolation or a sandboxed container, never on an operator's own machine.
Build the 200-run eval from Chapter 12 before committing to any framework, and hold the line at your pre-set threshold.

### Stage 2: Add Jev as the router and gate (weeks 4-10)

Insert Jev as the request-time classifier deciding static-index versus live-resolve (Chapter 9), as the content screen before untrusted page text enters context (Chapter 11), and as the reranker and verifier for extracted results (Chapter 8).
Before it replaces any existing classifier, confirm it matches your frontier-model reference on a held-out set, per Section 12.4.
Account for the single-region latency caveat from Section 2.4 in your design.

### Stage 3: Slack and search integration (weeks 8-14)

Build on the official Slack MCP server, not the community token-bypass server, so that OAuth scoping and audit logs stay intact.
Expose the browser-agent MCP server behind the confidence gates built in Stage 2, and respect Slack's rate limits with a queuing server.

### Stage 4: Runtime resolution (weeks 12 onward)

Implement live-web resolution with per-topic freshness TTLs, with Jev owning the fetch-versus-cache decision from Section 9.3.
Expand only once live-resolution latency and per-query cost sit inside your product's UX budget; a reasonable starting target is sub-3-second latency and sub-$0.01 per query for the browse tier, adjusted to your own product's tolerance.

### Security gates that override the schedule

Apply these throughout, not just at the end:

- If an agent needs standing access to sensitive systems, email, banking, admin panels, or PII, require sandboxing, least-privilege scopes, and a human approval checkpoint, or do not ship it.
- If a workflow hits a login wall on every run with no way to persist a session, re-scope it as described in Section 11.4 rather than automating around the wall.
- If the target system is not one you own or are contractually permitted to automate, stop.

### Lab: draft your own fourteen-week plan

Using this plan as a template, write your own version with real dates, named owners for each stage, and the specific threshold numbers from your own 200-run eval and your own held-out Jev evaluation, not the placeholder numbers in this chapter.

**Done when:** every stage has an owner, a date, and a number that would tell you to stop or slow down, not just a number that would tell you to proceed.

---

## 15. Appendix: working templates

### 15.1 A Jev decision call, all three primitives together

```yaml
request:
  state: |
    <serialized page state, or task state, relevant to this step only>
  questions:
    - type: choice
      id: next_action
      options: [CLICK_12, CLICK_13, TYPE_TEXT_14, SCROLL_DOWN, DONE, BLOCKED]
    - type: noul
      id: goal_reached
      statement: "The stated task goal has been fully achieved."
    - type: noul
      id: agent_stuck
      statement: "The last three actions repeated without changing observable state."
response:
  next_action: { choice: CLICK_12, confidence: 0.94, probabilities: {...} }
  goal_reached: { probability: 0.03 }
  agent_stuck: { probability: 0.02 }
```

### 15.2 Stop-condition pseudocode

```python
def step(state, budget):
    if budget.steps_used >= budget.max_steps:
        return Result.STOPPED_BUDGET
    if budget.cost_used >= budget.max_cost:
        return Result.STOPPED_BUDGET
    decision = jev.ask(state, NEXT_STEP_QUESTIONS)
    log_step(state, decision)
    if decision.goal_reached.probability > 0.85:
        return Result.DONE
    if decision.agent_stuck.probability > 0.85:
        return Result.ESCALATE_TO_REASONING_LAYER
    execute(decision.next_action.choice)
    return Result.CONTINUE
```

### 15.3 MCP tool contract checklist

Before shipping an MCP tool, confirm each of the following:

- The tool does one named thing, not an open-ended action.
- Every input is validated against a schema before it reaches the browser or any credentialed system.
- The tool's scope (read-only versus write, which systems, which credentials) is stated in its description, not just implied by its name.
- Every call is logged with its full step trace, not just its final result.
- A step budget, a cost budget, and a timeout are enforced in code, not left to the model to self-limit.
- Untrusted page content is screened before it reaches any reasoning model's context.
- Authentication is handled through a pre-seeded or human-assisted session, never by the agent typing a password.

### 15.4 The 200-run eval scorecard

```yaml
framework: <name>
tasks: 20
runs_per_task: 10
total_runs: 200
completion_rate: <percent>
failure_causes:
  login_wall: <count>
  missing_element: <count>
  timeout: <count>
  wrong_action: <count>
  other: <count>
threshold_set_before_run: 70%
decision: proceed | do_not_proceed | proceed_with_human_fallback
```

### 15.5 Freshness TTL table (starting point, adjust per domain)

| Fact type | Example | Suggested TTL |
| --- | --- | --- |
| Flight or transit status | Gate, departure time | 15 minutes |
| Retail price or stock | Product price, inventory count | 1 to 24 hours |
| News or breaking events | Incident status, public announcements | 15 to 60 minutes |
| Company policy documents | Travel policy, expense policy | 1 quarter |
| Org structure or ownership | Team ownership of a system | 1 quarter |
| Reference documentation | API reference, internal runbooks | Until the source changes, verified on read |

---

Jev is one week old at the time this book was written, and the specific numbers in it will age quickly.
The architecture will not.
A reasoning layer that sets goals, a fast and calibrated decision layer that makes the bulk of an agent's micro-decisions, and an execution layer that never trusts model output without validating it against the world it is actually acting on, that shape will keep being the right one long after Jev's launch-week benchmarks have been superseded.
Build your own evidence before you trust anyone else's, including this book's.
