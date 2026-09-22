# What Is an Agent?

!!! example "Hands-on"
    Full runnable recipe: [`what-is-an-agent/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/what-is-an-agent) in the companion cookbook — the same customer-support question run through a fixed pipeline and through a decision-point agent, with a real captured trace.

??? abstract "TL;DR — quick revision"
    - **The defining property isn't "uses an LLM."** Anthropic's working definition: a **workflow** is a system where "LLMs and tools are orchestrated through predefined code paths"; an **agent** is a system where "LLMs dynamically direct their own processes and tool usage." The line is *who decides the next step* — fixed code, or the model.
    - **A retry-on-failure loop written in application code is still a workflow, not an agent**, under this definition — Anthropic's own evaluator-optimizer pattern (generate, grade, loop until a criterion is met) is listed as a *workflow*, because the LLM isn't the one deciding whether or how to retry.
    - **Definitions genuinely disagree at the edges.** Simon Willison: "an LLM agent runs tools in a loop to achieve a goal." Chip Huyen: something that perceives and acts on an environment. OpenAI: "systems that independently accomplish tasks on your behalf." None of these are wrong; they draw the line in slightly different places, and a strong interview answer names the disagreement rather than picking one as *the* definition.
    - **Most systems marketed as agents aren't, by any of these definitions.** Menlo Ventures' December 2025 enterprise survey found only 16% of enterprise and 27% of startup deployments calling themselves "agents" actually qualify as one — most are "if-then logic around a model call."
    - **A worked trace beats a definition in an interview.** This page's own demo shows the same misclassification happening in both a workflow and an agent — the difference isn't that the agent classifies better, it's that the agent has a step where it can act on its own doubt, and the workflow structurally doesn't.

## The line, worked through an example

A plain LLM call is input in, output out — one shot, no loop, no branching on what comes back. The moment you add a loop, one question decides whether you've built a workflow or an agent: **who decides what happens next?**

If the answer is "the code I wrote" — a fixed sequence of steps, even one with retries or a quality check baked in — that's a workflow. Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) names five workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer) and is explicit that all five stay workflows even though several of them loop and even though some of them use an LLM to grade another LLM's output — because the *decision to keep going* is still governed by code the developer wrote (a criterion, a step count, a fixed router), not by the model choosing its own next action from live options.

If the answer is "the model, based on what it just observed" — that's an agent. The system's next action depends on the outcome of its own previous action, decided by the model itself at runtime, not read off a lookup table.

This is genuinely useful to apply, not just recite. Take a common interview scenario: a junior engineer builds a "customer support agent" that's really `classify intent -> retrieve FAQ -> generate response`. Every step can involve an LLM call. It is still a workflow — there's no point where the system can decide "this retrieval didn't actually answer the question, let me try something else." The moment you add that check, and let the *model* pick what happens next (broaden the search, ask a clarifying question, escalate), you've crossed into agent territory. That's exactly what this page's [hands-on recipe](https://github.com/DhruvMakwana/agents-cookbook/tree/main/what-is-an-agent) builds, side by side, on the same task.

## The two pipelines, in code

Same 5-entry FAQ knowledge base for both:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/what-is-an-agent/workflow_vs_agent_docs.py:faq_kb"
```

**The workflow** — `classify -> retrieve -> generate`, nothing can change what the next step does:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/what-is-an-agent/workflow_vs_agent_docs.py:workflow_pipeline"
```

**The agent** adds exactly two things the workflow doesn't have: a judge step, and a real decision point where the *model* — not an `if/else` in this code — picks what happens next.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/what-is-an-agent/workflow_vs_agent_docs.py:agent_judge_step"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/what-is-an-agent/workflow_vs_agent_docs.py:agent_decision_point"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/what-is-an-agent/workflow_vs_agent_docs.py:agent_loop"
```

!!! success "A real, unforced run — both pipelines given the same misclassification"
    Test query: *"My storage suddenly shows as full even though I deleted a bunch of files last week, and now I can't upload anything — is there some sync delay, or did I lose data?"* One real run against Claude Haiku 4.5 — see the recipe README for the full setup.

    Both pipelines' `classify_intent` step picked `storage_limits` — a reasonable-sounding guess given the word "storage," and the wrong bucket for what's actually being asked (this isn't a tier-size question).

    **Workflow**, stuck with that FAQ entry, replied:
    > "I don't have enough information in the provided FAQ entry to answer your question. The FAQ only states storage limits for different tiers, but doesn't address sync delays, data loss, or why your storage might still show as full after deletion. [...] I'd recommend contacting support directly so they can investigate your account specifically."

    A genuinely honest finding worth stating plainly: Claude *declined on its own* here rather than confidently inventing an answer, and did so more explicitly than that — it named exactly what the FAQ doesn't cover and pointed the customer elsewhere. So the standard "naive pipeline confidently makes something up" framing doesn't automatically hold, and this held for both a small local model and a frontier one on this same query. What the workflow structurally still lacks, regardless of how well the wording comes out, is a **mechanism to act** on that doubt: whatever the generation step says, that's the final message back to the customer. There's no next step for it to take.

    **Agent**, with the identical misclassification, ran its judge step, got `sufficient: false`, and the model itself chose `CLARIFY` — the one real decision point in this recipe's design. (The clarifying question it triggers is a fixed reply in this minimal version, not model-generated text; what's real here is the *action choice* itself, not its wording.)

    Same initial mistake, same model. The difference isn't a smarter classifier — it's that the agent's architecture gave the model somewhere to go with its own uncertainty, and the workflow's didn't.

## Definitions genuinely disagree — and that's fine to say out loud

There is no single canonical definition, and naming that is a stronger interview answer than confidently picking one:

- **Anthropic**: workflows are "orchestrated through predefined code paths"; agents are where "LLMs dynamically direct their own processes and tool usage." Umbrella term: "agentic systems."
- **OpenAI**: "Agents are systems that independently accomplish tasks on your behalf." Their [Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) frames it around three components: a model, tools, and instructions.
- **Simon Willison**: ["an LLM agent runs tools in a loop to achieve a goal."](https://simonwillison.net/2025/Sep/18/agents/) Deliberately mechanism-first, not autonomy-first.
- **Chip Huyen**: an agent is anything that perceives its environment and acts on that environment through tools — closer to the classical AI definition, decoupling *planning* from *execution*.
- **LangChain**: a spectrum rather than a binary, from a single tool call up to a fully autonomous loop.

Where they agree: an agent's next action has to depend on live feedback from its own previous action. Where they disagree: how much autonomy, and over how many steps, something needs before the label applies.

## LLM agents vs. classical RL agents

Interviewers sometimes ask this to check you're not just pattern-matching "agent = has tools." A classical RL agent and an LLM agent share the loop structure (perceive, decide, act, observe, repeat) but differ in almost everything about how that loop gets *good*:

| | Classical RL agent | LLM agent |
|---|---|---|
| Policy | Learned via a reward signal over many episodes | Mostly prompted, not trained, per task |
| Action space | Fixed, often small and discrete | Open-ended — any tool call, any text |
| Improvement mechanism | Gradient updates from reward | In-context learning, reflection, occasionally fine-tuning on trajectories (GRPO/DPO — see the Training section of this site) |
| Sample efficiency | Needs many episodes | Zero/few-shot, generalizes from pretraining |
| Typical failure mode | Reward hacking, poor generalization outside training distribution | Hallucinated actions, brittle to prompt drift, errors compounding over long horizons |

RL-style training is increasingly applied *on top of* the LLM-agent loop rather than instead of prompting — that bridge is where an interviewer probing GRPO or DPO experience will usually head next.

## A short history worth knowing

The "wave" of 2023 agent projects — AutoGPT, MetaGPT, ChatDev, AgentVerse, HuggingGPT — mostly wired an LLM into a fully autonomous loop with minimal guardrails and let it run. Several of those repositories are effectively stale today: interesting proofs of concept, not architectures anyone builds on directly now. What replaced that era wasn't a bigger loop; it was the workflow-vs-agent distinction itself (Anthropic's framing, December 2024) — the realization that most real tasks are better served by a *bounded* system that only reaches for full agentic autonomy where the task genuinely needs it, plus (from mid-2025 on) a second shift toward **context engineering**: treating what goes into the model's context on each step as the thing you actually design, not an afterthought once the loop is built. Both threads run through the rest of this site.

## When you would *not* build an agent

This is a favorite trade-off question, and the strong answer leads with cost, not capability: agentic loops add **latency** (multiple model round-trips instead of one), **cost** (tokens scale with steps, and a stuck loop can burn a lot of them), **non-determinism** (harder to test, harder to guarantee a specific output shape), and a genuinely harder **evaluation and debugging** problem (see this site's Evaluating Agents page). If a task is well-scoped, doesn't need to branch on intermediate results, and a fixed pipeline already gets it right, a workflow is more reliable, cheaper, and easier to monitor — and reaching for an agent anyway is exactly the reflex interviewers are testing candidates for.

## Use it for / skip it for

**Reach for an agent when:** the right sequence of steps genuinely can't be known in advance — the task is open-ended, multi-hop, or the correct next action depends on what a previous step actually returned (not just whether it succeeded).

**Skip it when:** the task is well-scoped and repetitive, the steps are known ahead of time, or a fixed pipeline with a quality check already handles the cases you see in practice — a workflow with an evaluator step is still simpler to reason about and monitor than an agent, even one that loops.

## Interview angle

**Weak answer**: "An agent uses an LLM and a pipeline doesn't." Wrong — plenty of pipelines call an LLM at every stage and are still fixed pipelines.

**Strong answer**: name the decision-point distinction, give a concrete example of a pipeline that looks agentic but isn't (classify → retrieve → generate, no branching), and know at least two other framings exist (Willison's tool-loop, Huyen's perceive-and-act) without treating any one as universally "correct." Bonus: cite that most production systems marketed as agents don't actually meet this bar (Menlo's 16%/27% figure) — it signals you've looked past the marketing.

**Follow-up to expect**: "When would you *not* want to build something agentic, even though it's technically possible?" — see the section above. Interviewers like candidates who don't reflexively reach for an agent.

## Build it yourself — 30 minutes

1. Pick a small, narrow knowledge base (5 or so entries is enough) and a `classify -> retrieve -> generate` pipeline over it, using any provider — see the [recipe](https://github.com/DhruvMakwana/agents-cookbook/tree/main/what-is-an-agent) for a working `llm.py` (Claude by default; a free local Ollama model also works).
2. Find a query that breaks it — something with vocabulary mismatch, so classification picks the wrong bucket.
3. Add exactly one thing: a judge step that asks the model "does this actually answer the question?" and, if not, a step where the **model** — not your code — picks the next action from a short menu (try again with a different category, ask a clarifying question, escalate).
4. Run the same broken query through both versions and compare. You now have a working, honest answer to "show me the difference between a workflow and an agent" instead of a definition.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: What Is an Agent?">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A team built a support bot: classify the customer's message into one of 8 categories, retrieve the matching help-center article, and have an LLM turn it into a reply. They call it 'our support agent.'",
      "question": "Under Anthropic's workflow-vs-agent distinction, is this an agent?",
      "options": [
        "Yes — it uses an LLM at every step, so it's agentic",
        "No — it's a workflow, because the sequence of steps is fixed regardless of what any step returns",
        "It depends on how many categories there are",
        "Yes, because retrieval makes any pipeline agentic"
      ],
      "correct": 1,
      "explanations": [
        "Using an LLM at each step doesn't make a system agentic — the classic mistake this page opens with. The line is who decides the next step, not how many LLM calls happen.",
        "Correct. There's no point where the outcome of one step changes which step runs next — that's exactly Anthropic's definition of a workflow, even with LLM calls throughout.",
        "Category count is irrelevant to the workflow-vs-agent distinction — it's about whether the control flow can branch on live feedback, not scale.",
        "Retrieval is orthogonal to this distinction; RAG pipelines are workflows unless something in the loop can decide to retrieve differently based on what came back."
      ]
    },
    {
      "scenario": "An engineer adds a retry: if the generated answer's confidence score (computed by fixed code) is below a threshold, regenerate up to 3 times before returning the best-scoring attempt.",
      "question": "Does adding this retry loop make the system an agent?",
      "options": [
        "Yes — any loop with more than one attempt is agentic",
        "No — this is Anthropic's evaluator-optimizer workflow pattern; the retry logic is still fixed code, not a model decision",
        "Yes, because it now has multiple steps",
        "No — because it uses a threshold instead of an LLM judge"
      ],
      "correct": 1,
      "explanations": [
        "A loop alone doesn't cross the line — Anthropic explicitly lists a generate-grade-retry loop as a workflow pattern (evaluator-optimizer), not an agent.",
        "Correct. The decision to retry and when to stop is governed by fixed code (a threshold, a step count) — the model never chooses its own next action from live options.",
        "Step count isn't the criterion; a five-step fixed pipeline is still a workflow, and a one-step agent that picks its own tool is still an agent.",
        "The mechanism used to score the output (a threshold vs. an LLM judge) doesn't matter here — what matters is who decides whether/how to retry, and that's still the code."
      ]
    },
    {
      "scenario": "You're asked in an interview: 'Given unlimited engineering time, would you make every feature in your product agentic?'",
      "question": "What's the strongest answer?",
      "options": [
        "Yes — agents are strictly more capable, so there's no reason not to",
        "No — reflexively reaching for agentic architecture adds latency, cost, and non-determinism that a well-scoped fixed pipeline doesn't need, and evaluation gets harder",
        "It depends only on whether the team has experience with agent frameworks",
        "No — agents are never appropriate for production systems"
      ],
      "correct": 1,
      "explanations": [
        "More capable at open-ended tasks, yes — but that capability comes with real costs (latency, cost, non-determinism, harder eval) that a fixed pipeline doesn't pay when the task doesn't need branching.",
        "Correct. This is the trade-off question this page's 'When you would not build an agent' section is built around — interviewers are testing judgment, not enthusiasm.",
        "Framework familiarity is an implementation detail, not the reason to choose an architecture; the decision should be driven by whether the task genuinely needs branching on live feedback.",
        "Too absolute in the other direction — plenty of production systems (this whole site's Systems and Production sections) are agentic where the task genuinely calls for it."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents) (2024-12-19) — the workflow-vs-agent definition and the five workflow patterns this page builds on.
- OpenAI, ["A Practical Guide to Building Agents"](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) (2025) — the "independently accomplish tasks" definition and the model/tools/instructions framing.
- Simon Willison, ["I think 'agent' may finally have a widely enough agreed upon definition"](https://simonwillison.net/2025/Sep/18/agents/) (2025-09-18).
- Chip Huyen, ["Agents"](https://huyenchip.com/2025/01/07/agents.html) (2025-01-07) — the perceive-and-act framing and the planner/executor split.
- Menlo Ventures, ["2025: The State of Generative AI in the Enterprise"](https://menlovc.com/perspective/2025-the-state-of-generative-ai-in-the-enterprise/) (2025-12-09) — the 16%/27% "true agent" finding.
- Lilian Weng, ["LLM Powered Autonomous Agents"](https://lilianweng.github.io/posts/2023-06-23-agent/) (2023-06-23) — the earlier planning/memory/tool-use framing, useful background even though this site organizes differently.
