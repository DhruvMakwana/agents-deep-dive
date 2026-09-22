# Multi-Agent Systems

!!! example "Hands-on"
    Full runnable recipe: [`multi-agent-systems/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/multi-agent-systems) in the companion cookbook — Anthropic's and Cognition's opposing claims about multi-agent systems, tested directly on real toy tasks with real measured numbers.

??? abstract "TL;DR — quick revision"
    - **A multi-agent *system* is not the same thing as orchestrator-workers.** [Workflow Patterns](workflow-patterns.md)' orchestrator-workers is a fixed topology — Anthropic classifies it as a *workflow*. A multi-agent system is what Anthropic calls an *agent* — a lead agent that delegates to subagents that are themselves autonomous, operating in parallel with their own judgment, their own tool calls, and their own context windows.
    - **A real measured run put a real number on Anthropic's token-cost claim**: single agent answering three independent questions, 1 call, 1,048 tokens; a lead agent dispatching three subagents plus a synthesis call, 4 calls, 3,369 tokens — a real **3.21x** multiplier, the same direction as (if smaller than) Anthropic's own reported 4x/15x figures.
    - **A real test of Cognition's inter-agent consistency risk did not reproduce the failure** — twice. Two subagents, each blind to the other's output, independently inventing a shared fact (a trial length) converged on the same value both times, even after the prompt was redesigned specifically to remove an obvious reason they'd default to the same answer. Reported as an honest negative result, with the real, disclosed reason why the repro's own design likely couldn't have shown the failure Cognition describes.
    - **MAST's real taxonomy**: 14 distinct failure modes across 3 categories — system design issues, inter-agent misalignment, task verification — built from 1,600+ annotated traces across 7 frameworks, and the paper's own stated finding is that multi-agent systems' "performance gains on popular benchmarks are often minimal."
    - **"When to use one agent" has a real, non-hand-wavy answer**: genuinely independent, breadth-first sub-tasks are where the token cost buys something real (Anthropic's own 90.2% improvement, largely attributable to spending more tokens); tightly-coupled tasks needing shared context between steps are where a single agent avoids a coordination problem it would otherwise have to solve by hand.

## Multi-agent systems vs. orchestrator-workers

This is the distinction most likely to get muddled in an interview, and Anthropic's own writing draws it precisely: in the orchestrator-workers *workflow* pattern, "a central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results" — but the workers are single calls executing a fixed role the developer wired in ahead of time, the same way every pattern on the [Workflow Patterns](workflow-patterns.md) page is a fixed topology with an LLM filling in judgment at specific, pre-built decision points.

A multi-agent *system*, in the sense Anthropic uses for their own Research product, is different in kind: "a multi-agent architecture with an orchestrator-worker pattern, where a lead agent coordinates the process while delegating to specialized subagents that operate in parallel" — but here, each subagent is itself a full agent, with its own tool access, its own reasoning loop, its own context window, deciding its own actions rather than executing one pre-scripted call. The topology looks similar on a diagram — a lead node, worker nodes — but the thing at each worker node is qualitatively different: a fixed function call versus an autonomous decision-maker. That difference is exactly why multi-agent systems carry the specific risks this page tests (token cost, coordination failure) that a orchestrator-workers workflow, by design, doesn't — a workflow's workers can't disagree with each other in a way the code didn't already anticipate, because they aren't making open-ended decisions to begin with.

## The debate: Anthropic vs. Cognition

Two influential, opposite-leaning posts, published within weeks of each other in mid-2025. Anthropic's engineering team, describing their own production Research system: multi-agent with Opus 4 as lead and Sonnet 4 subagents "outperformed single-agent Claude Opus 4 by 90.2%" on an internal breadth-first research eval (identifying board members across S&P 500 IT companies) — but with a cost, stated just as plainly: "agents typically use about 4x more tokens than chat interactions, and multi-agent systems use about 15x more tokens than chats," and on a separate benchmark (BrowseComp), "token usage by itself explains 80% of the variance" in performance — meaning much of multi-agent's advantage there is bought with tokens, not architecture.

Cognition's post argues close to the opposite, from direct experience building coding agents: "running multiple agents in collaboration only results in fragile systems. The decision-making ends up being too dispersed and context isn't able to be shared thoroughly enough between the agents." Their concrete example: two subagents build a Flappy Bird clone, one given the sub-task of the background, the other the bird — the background subagent produces something that "looks like Super Mario Bros," while the bird "doesn't look like a game asset and it moves nothing like the one in Flappy Bird" — a coherent single vision fractured into two locally-reasonable, globally-inconsistent pieces, left for a final agent to somehow reconcile. Their stated principle: "actions carry implicit decisions, and conflicting decisions carry bad results" — because each subagent only sees the sliver of context it was handed, not the full trace of decisions the other subagent made along the way.

Both are right about their own evidence, and the difference is task shape, not which company is correct — which is exactly what the two real experiments below test.

## Real experiment 1: token economics

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/multi-agent-systems/multi_agent_systems_docs.py:token_economics"
```

!!! success "A real run — three genuinely independent questions, two architectures, real token counts"
    **Input**, identical three questions for both conditions: explain (1) how a Bloom filter achieves space-efficient set-membership testing, (2) how consistent hashing reduces cache invalidation, (3) how a skip list achieves O(log n) search without tree-style rebalancing — genuinely independent of each other, the shape Anthropic's own post argues favors multi-agent.

    **Single-agent** (Claude Sonnet 5, one call given all three questions): **1 call, 1,048 total tokens.**

    **Multi-agent** (a lead agent dispatching one subagent per question — each subagent blind to the other two — then a synthesis call combining all three): **4 calls, 3,369 total tokens.**

    **Ratio: 3.21x.** Smaller than Anthropic's own reported 4x (agent vs. chat) or 15x (multi-agent vs. chat) — those are measured on real production workloads against a plain-chat baseline, not against a single-agent-doing-everything baseline on one small toy task — but the same direction, on a real number this page computed itself rather than borrowed. The mechanism is visible in the design itself: each subagent call repeats system framing overhead ("You are a research subagent...") that a single combined call only pays once, and the synthesis call has to re-read all three subagent outputs in full before producing the final answer — tokens single-agent's one pass never had to spend at all.

## Real experiment 2: consistency risk

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/multi-agent-systems/multi_agent_systems_docs.py:consistency_risk"
```

!!! success "A real run — an honest negative result, with the real reason disclosed"
    **Input** — two FAQ sections for a fictional app, Nimbus: a Pricing section stating a free-trial length in days, and a Refunds section that must reference that same trial length when explaining no refund is needed during it. Deliberately underspecified — "there is no official policy yet, pick a specific, slightly unusual number of days, not a common default like 7, 14, or 30" — designed to force each writer to genuinely invent a value rather than recall one everyone already agrees on.

    **Single-agent** (one call, both sections written in the same pass): picked **19 days** in Pricing, referenced **19 days** in Refunds. Consistent — trivially, since it's one generation with the value fixed the moment it was first written.

    **Multi-agent** (two subagents, each given only its own section's instruction, each blind to the other's output — exactly Cognition's "conflicting decisions" scenario): the Pricing subagent picked **11 days**; the Refunds subagent, asked separately and with no visibility into the Pricing subagent's choice, also landed on **11 days**. Consistent.

    This didn't reproduce Cognition's failure — on this run, or on an earlier attempt before the "avoid common defaults" instruction was added (which converged on 14 days both times, before the fix). The honest, disclosed reason is a real limitation of this specific repro, not evidence the underlying risk is overstated: Cognition's own example is about **open creative interpretation** — what a background or a game character *looks like*, with genuinely high variance and no shared convention to fall back on. This repro tested convergence on a **single scalar fact**, and two independent runs suggest the same model, even instructed to be unusual, tends to agree with itself on a number more than the stylistic, high-variance choices Cognition's example turns on. A harder, more faithful repro of the consistency risk would test creative or structural divergence, not a number — real future work this page doesn't claim to have already done.

## MAST: how multi-agent systems actually fail

Beyond one company's coding-agent anecdote or another's research-system win, MAST (Cemri et al.) is the closest thing to a systematic answer: a taxonomy built from rigorous analysis of 150 real multi-agent traces (kappa = 0.88 inter-annotator agreement), then validated against 1,600+ annotated traces across 7 popular frameworks. Fourteen distinct failure modes, clustered into three categories:

1. **System design issues** — problems baked in before the system ever runs: unclear role or task specification, agents stepping on each other's responsibilities, missing conversation history that a later step actually needed.
2. **Inter-agent misalignment** — the category Cognition's Flappy Bird example lives in: agents talking past each other, making incompatible assumptions, or failing to actually incorporate what another agent already established (real echoes of "actions carry implicit decisions, and conflicting decisions carry bad results" from a completely independent research effort).
3. **Task verification** — nobody checks the final output is actually right; a subtly wrong result from one stage propagates all the way to the end because no step's job was to catch it.

The paper's headline finding is worth stating plainly, because it's easy to only remember the wins: "despite enthusiasm for Multi-Agent LLM Systems (MAS), their performance gains on popular benchmarks are often minimal." That's not a contradiction of Anthropic's 90.2% — it's a reminder that the win is real *for the task shape it was measured on* (breadth-first, parallelizable, worth the token spend), and MAST's broader survey across many frameworks and many task types finds that gain doesn't generalize automatically to every multi-agent deployment.

## When to use one agent

Both companies' own evidence, plus MAST's broader survey, point to the same practical answer, stated as precisely as the evidence allows: reach for multi-agent when a task genuinely decomposes into independent pieces that don't need to see each other's intermediate reasoning to succeed — Anthropic's board-member research task is the clean case, and this page's own token-economics experiment shows that architecture costs real tokens (3.21x, on our own small task) in exchange for that parallelism. Stay with a single agent when steps are coupled — when a later step's correctness depends on the specific reasoning path an earlier step took, not just its final output — because that's exactly the condition under which Cognition's "conflicting implicit decisions" risk applies, and this page's own consistency-risk experiment, even though it didn't reproduce a failure, tested exactly the shape of task (shared, underspecified fact needing agreement across pieces) where that risk structurally exists. MAST's task-verification category adds a third, orthogonal axis that applies regardless of the choice: whichever architecture is used, something has to actually check the final output is right, because neither a fixed workflow nor an autonomous multi-agent system does that for free.

## Interview angle

**Weak answer** to "would you use a multi-agent system for this task": *"Multi-agent is more powerful, so yes, for anything complex enough to benefit from parallelism."* This treats multi-agent as a strictly-better upgrade rather than a real trade-off, and it can't explain either half of this page's own real evidence — the 3.21x token cost that's real regardless of task, or the specific shape of task (tightly-coupled, shared-context-dependent) where Cognition's risk structurally applies even when this page's own repro happened not to trigger it.

**Strong answer**: the question to ask is whether the task's sub-parts are genuinely independent — can each one be solved correctly using only its own slice of context, with no need to see what another piece decided? If yes, multi-agent's parallelism and specialization are worth the measured token premium, the way Anthropic's board-member research task was. If no — if a later piece's correctness depends on a decision an earlier piece made that isn't fully captured in its final output — a single agent with the whole task in one context window avoids a coordination problem that a multi-agent system would otherwise have to solve explicitly (shared state, full-trace sharing, or an explicit verification step), which is real, non-trivial engineering Cognition's post argues most teams underestimate.

**Follow-up to expect**: "your own consistency-risk experiment didn't find a failure — doesn't that undercut the argument for being cautious?" No — it's a scoped, honest result, not a general one. The real finding is that a single scalar fact, chosen by the same model twice independently, tends to converge — which says something specific and narrow, not that inter-agent inconsistency isn't a real risk. MAST's 1,600+-trace survey and Cognition's own creative-task example are the broader evidence that the risk is real; this page's own repro tested one narrow instance of it and, honestly, wasn't positioned to reproduce the failure — a fact worth stating precisely rather than either overclaiming a disproof or hiding an inconvenient result.

## Build it yourself — 30 minutes

1. Pick a task with three or more genuinely independent sub-parts — something where each part's correct answer doesn't depend on knowing the others. Run it as one agent, one call, and record the real token count from your API response's usage field.
2. Run the same task as a lead agent dispatching one subagent call per sub-part, plus a synthesis call. Record the real token count the same way. Compare the real ratio — don't estimate it.
3. Design a second task where two pieces *must* agree on one shared, underspecified fact. Run it as one agent (both pieces in one pass) and as two subagents, each blind to the other's output. Extract the fact from each output programmatically and check agreement — don't just skim and assume it's fine.
4. If your consistency check doesn't find a mismatch, don't discard it — ask why, honestly, the way this page did. Was the shared fact a scalar with a strong convention, or something genuinely high-variance like Cognition's creative example? The answer changes what the result actually tells you.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Multi-Agent Systems">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A team built an orchestrator-workers pipeline (per Anthropic's workflow-pattern definition: a central LLM call decides which of several pre-built worker functions to invoke, each worker executes one fixed role). A teammate says: 'This is a multi-agent system, since it has a lead agent and workers operating under it.'",
      "question": "What's the most accurate correction?",
      "options": [
        "Any lead-plus-workers topology qualifies as multi-agent, regardless of what the workers are",
        "It's closer to Cognition's fragile-multi-agent case, since workers can't see each other's context either way",
        "It's a workflow -- the workers execute pre-built roles, not open-ended decisions as autonomous agents",
        "It only counts as multi-agent if the workers run concurrently rather than sequentially"
      ],
      "correct": 2,
      "explanations": [
        "Collapses a distinction Anthropic's own writing draws explicitly -- a fixed topology with workers executing pre-scripted roles is not the same thing as autonomous agents making open-ended decisions, even when the diagram looks similar.",
        "Misapplies Cognition's critique -- their argument is specifically about autonomous subagents making independent, potentially conflicting judgment calls, not about any lead-plus-workers shape; a fixed-role worker structurally can't make the kind of conflicting decision Cognition describes.",
        "Correct. Anthropic's own writing classifies orchestrator-workers as a workflow pattern precisely because the workers execute a fixed, developer-defined role -- the multi-agent Research system is different specifically because its subagents are autonomous, with their own tool access and judgment, not because of the lead-plus-workers shape itself.",
        "Introduces a fabricated distinguishing criterion -- concurrency versus sequential execution isn't what separates a workflow's workers from a multi-agent system's subagents in either Anthropic's or Cognition's framing."
      ]
    },
    {
      "scenario": "A real experiment measured a single agent answering three independent questions in 1,048 tokens, versus a lead-plus-3-subagents-plus-synthesis design using 3,369 tokens on the identical questions -- a real 3.21x multiplier. A team cites Anthropic's reported 4x/15x figures and concludes: 'Our number is basically wrong, or our multi-agent implementation is broken, since it doesn't match Anthropic's reported numbers.'",
      "question": "What's the strongest problem with that conclusion?",
      "options": [
        "The conclusion is right -- any real multiplier that doesn't match a cited figure indicates a bug",
        "Anthropic's figures use a different baseline (real chat) and workload -- not a contradiction",
        "3.21x actually exceeds 15x once counted correctly, so the team undercounted tokens",
        "Token multipliers are never comparable across two different measurements, under any circumstances"
      ],
      "correct": 1,
      "explanations": [
        "Treats a citation as a universal constant rather than a measurement under specific conditions -- reasonable real numbers vary by task, baseline, and scale without indicating an error.",
        "Correct. Anthropic's 4x and 15x figures compare agents and multi-agent systems against a plain-chat baseline on real production workloads -- not a single-agent-does-everything baseline on one small three-question toy task. A smaller real multiplier, in the same direction, from a genuinely different measurement setup is expected, not a red flag.",
        "A fabricated, arithmetically false claim -- 3.21 does not exceed 15 under any accounting; nothing in the real experiment supports this.",
        "Overcorrects into an unreasonable, absolutist position -- comparing real measurements while being explicit about differing conditions is exactly the right way to use a cited figure as context, not a reason to avoid comparison."
      ]
    },
    {
      "scenario": "A real consistency-risk repro (two subagents, each blind to the other, independently inventing a shared numeric fact) found no disagreement across two separate attempts -- even after the prompt was changed specifically to discourage a common default answer. Someone argues: 'This proves Cognition's inter-agent consistency concern is overstated.'",
      "question": "What's the most accurate pushback?",
      "options": [
        "The repro tested a scalar number, narrower than Cognition's own creative-interpretation example",
        "Two consistent real runs are sufficient to generalize that the concern doesn't apply broadly",
        "The result should be dismissed outright, since a repro using a fictional app name is inherently invalid",
        "The repro actually did find a disagreement -- the outcome described is being misread"
      ],
      "correct": 0,
      "explanations": [
        "Correct. Cognition's own example (a background matching one visual style, a bird matching a different one) is about creative/stylistic interpretation -- genuinely high-variance with no shared fallback. This repro tested a scalar fact, which two real runs suggest the same model tends to agree with itself on even when blind and instructed to be unusual -- a real, disclosed, narrower test, not a disproof of the broader risk.",
        "Overgeneralizes two real but narrow results into a broad claim the repro's own design isn't positioned to support -- consistency on a scalar number doesn't settle the question of consistency on open-ended, high-variance creative choices.",
        "An overly strong, unsupported rule -- fictional scenarios are the established pattern across this entire cookbook precisely because they let mechanisms be tested cheaply and safely; that doesn't invalidate a result.",
        "Misstates the real outcome -- both real runs found the two subagents' answers matched (19/19 and 11/11), not a mismatch."
      ]
    },
    {
      "scenario": "MAST's own reported finding is that multi-agent systems' 'performance gains on popular benchmarks are often minimal,' based on a 1,600+-trace survey across 7 frameworks. A candidate cites this in an interview as: 'This means Anthropic's reported 90.2% multi-agent improvement must be exaggerated or unrepresentative.'",
      "question": "What's the most accurate response to that claim?",
      "options": [
        "Correct -- a survey finding minimal average gains directly contradicts any single large reported improvement",
        "MAST's finding only applies to open-source frameworks, so it says nothing about a proprietary system",
        "90.2% isn't comparable to any benchmark result, since it was self-reported rather than independently measured",
        "MAST reports a broad average across many frameworks; Anthropic's number is scoped to one favorable task shape"
      ],
      "correct": 3,
      "explanations": [
        "Treats a broad average and a scoped, task-specific result as if they must agree exactly -- a general survey finding modest average gains is fully consistent with a real, larger gain on a specific task shape well-suited to the architecture.",
        "Fabricates a restriction not stated in the paper -- MAST's abstract doesn't scope its finding to open-source frameworks only, and nothing about the taxonomy's failure modes is specific to open-source implementations.",
        "An overreaching dismissal -- self-reported doesn't mean incomparable; the real point is that the two numbers describe different scopes, not that either measurement type is invalid.",
        "Correct. MAST's finding describes gains across popular benchmarks broadly; Anthropic's 90.2% is explicitly scoped to a breadth-first research task where independent sub-tasks are exactly the shape multi-agent is well-suited to. A big win on a favorable, specific task and a modest average across many different tasks and frameworks can both be true at once."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["How we built our multi-agent research system"](https://www.anthropic.com/engineering/multi-agent-research-system) (2025-06-13) — the 90.2% improvement, the 4x/15x token multipliers, the 80% variance figure, orchestrator-worker architecture.
- Cognition, ["Don't Build Multi-Agents"](https://cognition.com/blog/dont-build-multi-agents) (2025-06-12) — the Flappy Bird example, "share full agent traces, not just individual messages," "actions carry implicit decisions."
- Cemri et al., ["Why Do Multi-Agent LLM Systems Fail?"](https://arxiv.org/abs/2503.13657) (arXiv 2503.13657, 2025-03) — MAST: 14 failure modes, 3 categories, 1,600+ traces across 7 frameworks, kappa = 0.88.
- Anthropic, ["Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents) (2024-12-19) — orchestrator-workers as a named workflow pattern, distinguished from full agent autonomy.
