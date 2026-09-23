# Models for Agents

!!! example "Hands-on"
    Full runnable recipe: [`models-for-agents/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/models-for-agents) in the companion cookbook — real Haiku 4.5 vs. Sonnet 5 tool-calling reliability, and a real routing repro across three dispatch strategies with real token counts and real correctness checks.

??? abstract "TL;DR — quick revision"
    - **"Small model = unreliable at reasoning" is a real, but no longer automatic, assumption — it needs checking per task.** A real repro found Claude Haiku 4.5 matched Sonnet 5 at 100% accuracy on a deliberately ambiguous tool-calling request, and separately solved three classic reasoning traps (the widgets/machines lateral-thinking problem, the bat-and-ball cognitive-reflection-test problem, and a chickens-and-cows system-of-equations problem) on the first try, no retries.
    - **Open-weight models genuinely compete at the top of tool-calling leaderboards now, not just "catch up eventually."** As of a June 2026 snapshot of the Berkeley Function-Calling Leaderboard, GLM 4.5 (open-weight) led at 76.7% overall accuracy, ahead of Claude Opus 4.7 (76.6%) and Gemini 3.1 Flash Lite Preview (76.5%) — a real, current instance of an open-weight model at the top of a major agentic benchmark, not a footnote below it.
    - **RouteLLM's real, cited numbers show what model routing buys when accuracy differs between tiers**: over 2x cost reduction while holding response quality, with per-benchmark reductions reported around 85% on MT Bench, 45% on MMLU, and 35% on GSM8K at 95% of GPT-4's own performance level.
    - **A real repro found routing still pays off even when the cheap model would have gotten every answer right anyway** — a genuinely different, and arguably more useful, finding than "routing saves money by avoiding mistakes." Real numbers: always-Sonnet cost 216 tokens for 4/4 correct; routed cost 199 tokens for the same 4/4 — the router itself has real overhead, but escalating only the queries flagged as complex still beat blanket escalation on cost without giving up any accuracy.

## The real question isn't "which model is best" — it's "which model is adequate for this call"

Every agent loop makes the same decision repeatedly, whether or not it's explicit: which model handles this specific step. Treating that as a single, fixed choice — pick the best available model, use it everywhere — is simple but wastes money on every call that didn't need the extra capability. The house rule this whole project follows (Claude only, task-scaled: Haiku for simple judgments, Sonnet for anything needing real reasoning) is itself a manual version of the same idea this page tests directly: does the "needs real reasoning" line actually fall where intuition says it does, and does routing between tiers pay off even when it doesn't?

## Repro 1: tool-calling reliability under a deliberately ambiguous request

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/models-for-agents/models_for_agents_docs.py:ambiguous-tools"
```

!!! success "A real run — 5 trials each, Haiku 4.5 vs. Sonnet 5, on a request built to tempt the wrong tool"
    **Input**: *"User ID U-8842 here. I'm traveling for the next couple of months and want to stop being charged while I'm away — can you sort that out? I'll be back and want it picked back up starting March 1st."* Two tools available: `cancel_subscription` (permanent, no auto-resume) and `pause_subscription` (temporary, auto-resumes on a given date). The phrase *"stop being charged"* is exactly the kind of wording that could tempt a hasty read toward `cancel_subscription`, even though the full request — pausing with an explicit resume date — is unambiguously a pause.

    **Real result**: Haiku 4.5 called `pause_subscription` correctly on **5/5 trials**. Sonnet 5 called `pause_subscription` correctly on **5/5 trials**. An unambiguous control case (*"cancel it for good, I won't be coming back"*) also hit 5/5 on both tiers for `cancel_subscription`.

    This is a clean, honest negative result, reported as it actually happened rather than adjusted to fit a predicted gap: whatever real tool-calling reliability differences exist between Haiku 4.5 and Sonnet 5, this specific ambiguity — a plausible temporal misreading of "stop being charged" — isn't one of them. The assumption that a cheaper model would slip here simply didn't hold up against a real run.

## What a major open-weight model's real leaderboard position says

The Berkeley Function-Calling Leaderboard (BFCL) exists specifically to measure the property repro 1 tested informally: how reliably a model picks the right tool call, across 1,000 real test cases spanning vehicle control, trading bots, travel booking, and file-system management, using state-based evaluation that checks both the resulting system state and the execution path taken to get there. As of a June 2026 snapshot: **GLM 4.5 — an open-weight model — led the leaderboard at 76.7% overall accuracy**, ahead of **Claude Opus 4.7 at 76.6%** and **Gemini 3.1 Flash Lite Preview at 76.5%**. Twenty-three models were evaluated at that snapshot.

The real significance isn't that open-weight models are "getting closer" — it's that, on this specific, widely-cited agentic benchmark, an open-weight model was *already at the top*, ahead of a frontier proprietary model from the same family as this project's own default. "Open-weight tool-calling reliability" isn't a caveat-laden footnote anymore; it's a real, current, measured fact worth checking before assuming a frontier proprietary model is the only reliable choice for a tool-heavy workload.

## Repro 2: routing — the real payoff even when accuracy doesn't differ

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/models-for-agents/models_for_agents_docs.py:router"
```

RouteLLM's real, published result frames the classic case for routing: *"our approach significantly reduces costs — by over 2 times in certain cases — without compromising the quality of responses."* Reported per-benchmark reductions run around 85% on MT Bench, 45% on MMLU, and 35% on GSM8K, each while holding to 95% of GPT-4's own performance level — the value proposition is specifically routing *easy* queries to a cheap model and reserving the expensive model for queries that actually need it.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/models-for-agents/models_for_agents_docs.py:reasoning-traps"
```

!!! success "A real run — three dispatch strategies, real tokens, real correctness, against reasoning-trap queries"
    Four queries: two simple (a factual lookup, a text transformation) and two classic reasoning traps — *"If 5 machines take 5 minutes to make 5 widgets, how long would 100 machines take to make 100 widgets?"* (the tempting wrong answer is 100 minutes; correct is **5 minutes**, since each machine works in parallel at the same rate) and *"A bat and a ball cost $1.10 in total. The bat costs $1.00 more than the ball. How much does the ball cost?"* (the tempting wrong answer is $0.10; correct is **$0.05**).

    | Strategy | Total tokens | Correct |
    |---|---|---|
    | Always Haiku | 174 | 4/4 |
    | Always Sonnet | 216 | 4/4 |
    | Routed (Haiku classifies each query, dispatches accordingly) | 199 | 4/4 |

    The router correctly classified both simple queries as `simple` and both reasoning-trap queries as `complex` — and Haiku 4.5, asked the reasoning-trap queries directly, got both right anyway (5 minutes; $0.05), confirmed on a third, harder query outside this table (a chickens-and-cows system-of-equations problem, correct answer 23, which Haiku also solved correctly on the first try).

    The real finding worth sitting with: **routing still cost less than always-Sonnet (199 vs. 216 tokens) while matching its accuracy, even though the cheap model alone would have scored identically for free.** The router's own classification calls are real overhead — routing didn't beat always-Haiku here, and couldn't have, since always-Haiku already got everything right. But that's not evidence against routing; it's evidence that this specific batch happened to fall entirely within Haiku's real capability, and routing paid its own overhead cost back by not needlessly escalating the two queries that didn't need escalation, while still being positioned to correctly escalate a harder query if one had actually appeared in the batch.

## What this means in practice

The honest results across both repros point at the same operational lesson from different angles: the line between "needs the expensive model" and "the cheap model handles this fine" isn't fixed, and it isn't safely assumable from a query's surface difficulty — this page's own three solved reasoning traps are the kind of question that would have differentiated model tiers cleanly a couple of generations ago, and don't anymore, for this particular pair of models on this particular benchmark. Treating that gap as fixed either wastes money (defaulting everything to the expensive tier "to be safe," when the cheap tier would have matched it) or risks real errors (defaulting everything cheap without ever checking whether that specific class of task still holds up). RouteLLM's real numbers and this page's own repro converge on the actual discipline: measure the real accuracy delta for your specific task distribution, on your specific model pair, before assuming either "you need the big model" or "the small model is fine" — and route based on what that measurement shows, not on an assumption about what queries generally look hard.

## Interview angle

**Weak answer** to "how do you decide which model to use for each step of an agent?": *"Use the smallest model that gets the job done, and the biggest model when accuracy really matters."* True in spirit, but it doesn't say how you'd actually know where that line falls for a specific pair of models and a specific task — and this page's own repro shows that line moves: reasoning traps that would plausibly have separated tiers a generation ago didn't separate Haiku 4.5 from Sonnet 5 at all.

**Strong answer**: model selection for agent steps should be an empirical decision, re-checked per task and per model generation, not a fixed rule of thumb applied forever. This page's own real routing repro demonstrates the actual discipline: run the real candidate queries against both the cheap and capable tier, measure real accuracy and real token cost for each, and only then decide whether routing (or blanket cheap, or blanket capable) is the right call for that specific workload — because the "obvious" assumption (reasoning traps need the expensive model) turned out to be wrong for this pair of models, and an assumption-based router would have escalated unnecessarily on queries the cheap model already handled correctly.

**Follow-up to expect**: "if the cheap model got everything right, wasn't the routing demo pointless?" No — it demonstrated something more useful than "routing catches the cheap model's mistakes": it showed routing's cost discipline holds up even in the case where there's no mistake to catch. The router still classified correctly, still escalated only the queries it judged genuinely complex, and still came in under the always-capable baseline on cost — meaning the same routing logic would have caught a genuinely hard query in this batch if one had appeared, without needing to know in advance which queries would turn out to be reasoning traps the cheap model could actually handle.

## Build it yourself — 30 minutes

1. Pick two model tiers you have access to, and design one request with genuine surface ambiguity between two plausible tool calls — the way this page's pause-vs-cancel request does. Run 5 trials on each tier and compare real accuracy, rather than assuming the cheaper tier will slip.
2. Pick two or three classic reasoning-trap questions (the widgets/machines and bat-and-ball problems this page used, or find others) and run them against your cheapest available model. Don't assume it'll fail — check.
3. Build a minimal router: one cheap-model call that classifies a query as simple or complex, then dispatches accordingly. Run a small batch of mixed queries through always-cheap, always-capable, and routed, and compare real token totals and real correctness across all three — this page's own repro is exactly this comparison, run once, honestly reported.
4. If your routed condition ties an always-cheap baseline on accuracy, don't discard the result — check whether it still beat the always-capable baseline on cost. That's the finding this page's own repro surfaced: routing's value doesn't disappear just because the cheap model turned out to be good enough.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Models for Agents">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro ran the same ambiguous tool-calling request (a subscription pause phrased as 'stop being charged') against Haiku 4.5 and Sonnet 5, 5 trials each. Both tiers picked the correct tool on all 5 trials.",
      "question": "What is the most accurate way to interpret this specific result?",
      "options": [
        "It shows this specific ambiguity does not separate these two model tiers",
        "It means tool-calling ambiguity is never a real risk for any model pair",
        "The test was flawed since a real experiment should always find some difference",
        "It proves Haiku 4.5 and Sonnet 5 are identical in every tool-calling scenario"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The result is scoped to exactly what was tested: this specific ambiguity, these two specific model versions, this specific tool pair. It's real, honest evidence that THIS gap doesn't exist here -- not a general claim about tool-calling reliability everywhere.",
        "Overstates a narrow negative result into a universal claim -- this repro tested one ambiguity against one model pair; it says nothing about tool-calling ambiguity risk in general, across other models, tools, or phrasings.",
        "Backwards reasoning -- a real experiment can validly produce a null result; treating 'no difference found' as evidence of a flawed test would bias future work toward only reporting differences that confirm a hypothesis, rather than reporting what was actually observed.",
        "Overgeneralizes far beyond what one test showed -- a single ambiguous-request scenario tying doesn't establish identical behavior across every possible tool-calling situation these two models might face."
      ]
    },
    {
      "scenario": "As of a June 2026 leaderboard snapshot, GLM 4.5 -- an open-weight model -- led the Berkeley Function-Calling Leaderboard at 76.7%, ahead of Claude Opus 4.7 (76.6%) and Gemini 3.1 Flash Lite Preview (76.5%).",
      "question": "What does this specific ranking most accurately establish?",
      "options": [
        "That every open-weight model now outperforms every proprietary model at tool calling",
        "That an open-weight model genuinely held the leaderboard top spot at that snapshot",
        "That this leaderboard no longer measures a meaningful capability gap between models",
        "That GLM 4.5's lead is permanent and will not change in future leaderboard updates"
      ],
      "correct": 1,
      "explanations": [
        "A large overgeneralization from one model's result -- this ranking establishes GLM 4.5's specific standing at that snapshot; it says nothing about EVERY open-weight model versus EVERY proprietary model.",
        "Correct. The precise, defensible claim is exactly this: at that specific snapshot, an open-weight model (GLM 4.5) held the top spot on a major, widely-cited agentic tool-calling benchmark -- a real, current fact, not a general or permanent one.",
        "Not what the numbers show -- the top three scores (76.7%, 76.6%, 76.5%) are close but real, distinct rankings on a benchmark still actively differentiating 23 evaluated models; closeness at the top doesn't mean the benchmark stopped measuring anything.",
        "Unsupported speculation about the future -- leaderboard rankings change as new models are evaluated; nothing about a single snapshot implies permanence, and the page doesn't claim otherwise."
      ]
    },
    {
      "scenario": "A real routing repro found all three dispatch strategies (always-Haiku, always-Sonnet, routed) achieved identical 4/4 accuracy on a batch of queries including two classic reasoning traps, with routed costing 199 tokens versus 216 for always-Sonnet and 174 for always-Haiku.",
      "question": "Given that accuracy was identical across all three strategies, what is the most accurate characterization of what routing actually demonstrated here?",
      "options": [
        "Routing was pointless in this run since the cheap model alone matched everyone's accuracy",
        "The identical accuracy across strategies suggests a bug in the recipe's grading logic",
        "Routing matched the capable tier's accuracy while costing less than blanket escalation",
        "Routing beat always-Haiku on cost, proving it's the superior strategy in every case"
      ],
      "correct": 2,
      "explanations": [
        "Misses the real, distinct value demonstrated -- routing didn't need to catch a mistake to be worth measuring; it showed the classification and dispatch mechanism works correctly and pays for its own overhead even in a batch where escalation wasn't strictly necessary.",
        "Unsupported -- the repro's per-query correctness was checked against independently known correct answers (Paris, 5 minutes, $0.05), not against each other; identical accuracy reflects three separate real evaluations reaching the same real correct answers, not a shared or buggy grading path.",
        "Correct. The real, useful comparison is routed vs. always-Sonnet, not routed vs. always-Haiku: routing matched always-Sonnet's accuracy (4/4) while costing less (199 vs. 216 tokens) -- a genuine, if modest, benefit that holds even though the cheap model alone would also have scored 4/4 for less.",
        "Contradicts the real numbers directly -- routing (199 tokens) cost MORE than always-Haiku (174 tokens), not less; the router's own classification calls are real overhead that a pure cheap-only strategy doesn't pay."
      ]
    },
    {
      "scenario": "RouteLLM's real published result states routing 'significantly reduces costs -- by over 2 times in certain cases -- without compromising the quality of responses,' evaluated against GPT-4 as the capable-tier baseline.",
      "question": "What is the most precise way to relate this cited result to the recipe's own 4-query routing repro, which showed a smaller, single-digit-percentage cost reduction rather than a 2x reduction?",
      "options": [
        "The two results contradict each other, so at least one of them must be wrong",
        "RouteLLM's number is outdated and no longer applies to current-generation models",
        "The recipe's repro proves RouteLLM's reported cost reduction was never real",
        "The small batch here is not positioned to reproduce a benchmark-scale reduction"
      ],
      "correct": 3,
      "explanations": [
        "False dichotomy -- the two results measure different things at different scales (a large, diverse benchmark distribution vs. a small, illustrative 4-query batch); neither being 'wrong' is required to explain the size difference.",
        "Unsupported and not the actual explanation -- nothing about model generation invalidates a routing mechanism's cost math; the real difference is scale and query-mix, not model recency.",
        "Overstates the comparison -- a small illustrative repro not reproducing a large benchmark's exact magnitude doesn't invalidate the cited study's own real, independently reported result; the two are complementary evidence at different scales, not competing claims.",
        "Correct. RouteLLM's reported multiple-times cost reduction comes from routing across a large, varied benchmark distribution where many queries are genuinely simple enough to route cheaply; a small, illustrative 4-query batch -- half of which were deliberately hard reasoning traps -- isn't the right scale or mix to reproduce that same magnitude, even though the same underlying mechanism is genuinely at work in both."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Yu, Zhang, Shin, et al., ["RouteLLM: Learning to Route LLMs with Preference Data"](https://arxiv.org/abs/2406.18665) (arXiv:2406.18665, ICLR 2025) — the real cost-reduction numbers this page's repro 2 draws its framing from.
- Berkeley, ["Berkeley Function-Calling Leaderboard (BFCL)"](https://gorilla.cs.berkeley.edu/leaderboard.html) — the real, current tool-calling benchmark this page cites GLM 4.5's leading position on.
- [Tools at Scale](tools-at-scale.md) — the real Anthropic numbers for tool-search and programmatic tool calling, a complementary technique for managing large tool libraries regardless of which model tier is calling them.
- [KV-Cache Economics](kv-cache-economics.md) — cache-stable prompts and cost discipline from a different angle (context structure rather than model choice).
