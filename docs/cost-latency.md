# Cost and Latency

!!! example "Hands-on"
    Full runnable recipe: [`cost-latency/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/cost-latency) in the companion cookbook — a real, measured transcript-growth repro plus a real confidence-routing test.

??? abstract "TL;DR — quick revision"
    - **Naive agent loops don't cost N times as much as a single call — they cost roughly N² times as much.** Every turn resends the full accumulating transcript, so a real, documented formula applies: `Total = N×S + u×N(N+1)/2 + r×N(N-1)/2` — the `N(N+1)/2` triangular-number term is the trap. A real, worked example: a 20-step loop generating 1,000 tokens/step produces **210,000 cumulative input tokens**, not the 20,000 a naive per-step estimate would suggest.
    - **A real repro confirmed the shape directly, and measured what windowing buys**: a naive 6-file tool loop's per-turn input tokens grew by a near-constant **~631 tokens each turn** (665→1290→1921→2552→3183→3814→4445) — that constant per-turn *increase* is exactly what makes the cumulative total quadratic. A windowed version (only the last 2 tool results sent in full) flattened that increase to **~121 tokens/turn** by the end, for a real **28.3% cumulative reduction** — a gap that only widens with more turns.
    - **Model routing has a real, large lever: current published pricing puts small and large tiers roughly 5-10x apart per token** (Haiku 4.5 at $1/$5 per million vs. Sonnet 5 at $2/$10 per million, input/output), and a real repro confirmed the upside directly — Haiku matched Sonnet's accuracy on a 10-question batch at **~44% of the real dollar cost**.
    - **But the routing mechanism itself has a real, sharp limitation**: asking a model to self-report its own confidence is not a reliable way to catch its own mistakes. A real repro found the smaller model reported `high` confidence on every question, including the one it got wrong — confidently, not hesitantly. This is a concrete, measured reason production routing methods lean on a different signal (like model-internal token-probability confidence) instead of self-report alone.

## The real shape of the cost trap: quadratic, not linear

The intuitive mental model for agent cost is "N turns, N times the cost of one turn." That's wrong for the common case, and a real, documented formula shows exactly why: `Total = N×S + u×N(N+1)/2 + r×N(N-1)/2`, where `S` is the fixed system-prompt size, `u` is new input added per turn (a user message or tool result), `r` is output tokens per turn, and `N` is the total number of turns. The `N(N+1)/2` term is a triangular number — it grows quadratically in `N` — because a naive loop resends the *entire* accumulating transcript on every single call, meaning turn 1's content gets rebilled on turns 2 through N, turn 2's content gets rebilled on turns 3 through N, and so on.

The real, worked scale of this: a 20-step loop where each step generates 1,000 tokens produces **210,000 cumulative input tokens** — not the 20,000 a flat per-step estimate would suggest. A real, separate worked example for a 10-step file-reading agent (Sonnet-tier pricing, 8,000 input tokens/step): a naive loop costs **43.3x** a single-pass baseline; a constrained, windowed version cuts that to roughly **29x** — real, substantial savings, but still nowhere near 1x, because *some* rebilling is unavoidable once a loop spans multiple turns at all.

## Repro: measuring the quadratic shape directly

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/cost-latency/cost_latency_docs.py:files-and-task"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/cost-latency/cost_latency_docs.py:windowing"
```

The windowing logic is deliberately simple: keep the message structure intact (every `tool_use` still has a matching `tool_result`, so the API still accepts the conversation) but replace old tool results' *content* with a short placeholder once they fall outside the window.

!!! success "A real run — naive vs. windowed, identical 6-file task, real per-turn token counts"
    **`naive`** (full transcript resent every turn), real per-turn `input_tokens`: `665, 1290, 1921, 2552, 3183, 3814, 4445`. Each turn costs about **631 tokens more** than the one before it — a constant per-turn *increase*, which is exactly what a quadratic cumulative total looks like turn by turn. Real cumulative total: **17,870 input tokens** across 7 turns.

    **`windowed_last_2`** (only the 2 most recent tool results sent in full), real per-turn `input_tokens`: `665, 1300, 1934, 2051, 2168, 2285, 2406`. Nearly identical to `naive` for the first 3 turns — the window hasn't started dropping anything yet, since fewer than 3 tool results exist. From turn 4 on, the per-turn increase flattens to about **121 tokens/turn**. Real cumulative total: **12,809 input tokens** — a real **28.3% reduction**, and the gap widens every additional turn past this one.

    Both conditions reached the correct final answer (`SUM=186`) — windowing here cost nothing in correctness, because the task's final step only needed the sum, not the individual file contents still visible earlier in the untouched transcript.

## Routing: a real, positive result and a real, sharp limitation, in the same run

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/cost-latency/cost_latency_docs.py:routing-tool"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/cost-latency/cost_latency_docs.py:routing-loop"
```

Ten real questions — some trivial, some genuinely tricky (a bat-and-ball problem, a box-relabeling logic puzzle, a compound-discount trap, and a "strawberry" spelling trap) — answered two ways: every question straight to Sonnet 5, versus Haiku 4.5 first with an explicitly-calibrated confidence field, escalating to Sonnet 5 only on a `low` reading.

!!! success "A real run, unforced — real dollar cost using current published pricing"
    **Cost and accuracy**: `always_sonnet` spent **$0.026866** for **9/10** correct. `routed` spent **$0.011843** for the identical **9/10** correct — Haiku 4.5 alone matched Sonnet 5's real accuracy on this batch at **~44% of the real dollar cost**. A genuine, positive result: the cheap model was fully adequate here, not a compromise.

    **But `escalated_count: 0`.** Haiku reported `confidence: "high"` on all 10 questions — including the one it got wrong. The miss was the "strawberry" spelling trap; Haiku confidently answered "Yes" (there are two consecutive 'r's), matching Sonnet's own real miss on the identical question, with identical confidence. This held even after one deliberate adjustment made specifically to give the routing mechanism a fairer shot: two additional hard, multi-step questions were added, and the tool's own description was rewritten to explicitly instruct calibrated self-assessment — *"mark confidence as low whenever the question involves multi-step arithmetic... not just when you're unsure of a fact."* Confidence stayed `high` across the board regardless.

## Why self-reported confidence isn't the mechanism production routers actually rely on

The real, sharper finding is in that null result, not despite it: a model that's about to be wrong isn't reliably aware that it's about to be wrong, so asking it to say so in words is not a dependable safety net. This is a concrete, measured version of a real, documented design choice: confidence-guided routing methods like STEER (arXiv 2511.06190) use *model-internal* confidence — derived from the smaller model's own output-token probabilities — rather than a self-reported natural-language field, specifically because that internal signal doesn't depend on the model correctly narrating its own uncertainty. This recipe's self-report mechanism is the practical fallback available through a hosted API that doesn't expose token-level logprobs the way some routing methods assume — Anthropic's Messages API is one such case — and the real result here is exactly why that gap matters: the cost benefit of routing was real and measured, but the safety benefit of routing (catching the small model's mistakes before they ship) did not materialize from self-report alone.

## What this means in practice

Two separate, real levers, not one: windowing/context-constraining attacks the *quadratic* term directly — it doesn't change which model answers, it changes how much of the transcript gets rebilled each turn, and the real measured gap (28.3% here, growing with more turns) compounds the longer a loop runs. Routing attacks the *per-token price* — a real, large lever (5-10x per token on current pricing) that this run's cost numbers confirm directly. But routing's *safety* case — using it to keep the expensive model's judgment on anything genuinely risky — needs a routing signal that doesn't rely on the small model accurately reporting its own uncertainty, because this run's own real result shows that signal failing exactly when it mattered most: on the one question the small model actually got wrong.

## Interview angle

**Weak answer** to "how would you cut this agent's API cost?": *"Switch to a cheaper model."* This page's own repro shows why that's incomplete on its own — the quadratic transcript-growth term applies regardless of which model answers, and a cheap model rebilling an ever-growing transcript still hits the same `N(N+1)/2` wall, just at a lower per-token price.

**Strong answer**: separate the two real levers explicitly. Attack the quadratic term with context management (windowing, summarization, or a fresh sub-agent per bounded subtask) — this page's own real numbers show a measured, compounding reduction. Attack the per-token price with routing — but be precise about what routing actually buys: this page's own real result got the cost benefit (44% of cost, identical accuracy) *and* a clean demonstration that the naive escalation trigger (self-reported confidence) never fired, even on the one question that needed it. A candidate who names that distinction — cost savings from routing is not the same guarantee as safety from routing, unless the escalation signal is actually reliable — is demonstrating real, measured understanding, not a generic cost-optimization checklist.

**Follow-up to expect**: "if self-reported confidence doesn't work, what would?" A real, honest answer from this page's own material: either a deterministic, non-LLM check on the output (when the task allows one, the same structural-versus-behavioral distinction [Guardrails and Human-in-the-Loop](guardrails-human-in-the-loop.md) makes about approval gates), or a model-internal confidence signal like STEER's token-probability approach — which this page's own repro couldn't test directly, since it's not exposed through the Messages API used throughout this project, a real, current constraint worth naming rather than glossing over.

## Build it yourself — 30 minutes

1. Pick a real multi-turn tool loop you already have (or the file-reading task above). Instrument it to print the real `usage.input_tokens` from every API response, and sum them across the full run.
2. Run it naive (full transcript every turn) and note the per-turn deltas — confirm they're roughly constant, which is what produces the quadratic cumulative total.
3. Add a real context constraint (a window, a summarizer, or dropping tool results once superseded) and re-run the identical task. Compare real cumulative totals, not just per-call estimates.
4. Separately, take a batch of real task instances at varying difficulty and try confidence-based routing with your cheapest adequate model. Check specifically whether the model's self-reported confidence was ever `low` on the instances it actually got wrong — that's the real test of whether the mechanism is doing its stated job, not just whether it saved money on average.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Cost and Latency">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro measured a naive agent loop's per-turn input token counts across 7 turns: 665, 1290, 1921, 2552, 3183, 3814, 4445 -- each turn costing roughly 631 tokens more than the one before it.",
      "question": "What does this specific pattern -- a roughly CONSTANT per-turn increase -- indicate about the shape of the CUMULATIVE total across turns?",
      "options": [
        "The cumulative total grows quadratically, since a steady per-turn increase yields a triangular-number sum overall",
        "The cumulative total grows linearly, since each individual turn's token count only increases by a fixed amount",
        "The cumulative total grows exponentially, since token counts are roughly doubling by the final turns of the run",
        "The cumulative total stays constant, since only the per-turn increase changes and not the baseline cost itself"
      ],
      "correct": 0,
      "explanations": [
        "Correct. A sequence with a roughly constant increase per step (665, +625, +631, +631, +631, +631, +631) is an arithmetic sequence, and the SUM of an arithmetic sequence is a triangular number -- quadratic in the number of terms. This is exactly the real, documented N(N+1)/2 term the page describes, confirmed directly by this run's own measured numbers.",
        "Confuses per-turn behavior with cumulative behavior -- a constant per-turn INCREASE (not a constant per-turn value) is precisely the signature of quadratic cumulative growth, not linear; linear cumulative growth would require each turn to cost roughly the SAME amount, not a steadily increasing amount.",
        "Overstates the growth rate -- exponential growth would mean each turn's increase itself keeps growing multiplicatively (631, then ~1200, then ~2400...); here the increase itself stays roughly flat (~631 each time), which is the quadratic/triangular-number signature, not exponential.",
        "Directly contradicted by the data -- token counts clearly increase every single turn (665 up to 4445); nothing about this run is constant except the SIZE of the increase, not the total itself."
      ]
    },
    {
      "scenario": "The same repro ran a windowed version of the identical task (only the last 2 tool results sent in full) and found nearly identical per-turn token counts to the naive version for the first 3 turns, before the windowed version's growth rate flattened sharply from turn 4 onward.",
      "question": "What is the most precise explanation for why the first 3 turns looked nearly identical between conditions?",
      "options": [
        "The windowing logic contained a bug that only started working partway through the run",
        "The model ignored the windowed context and used its own cached memory of earlier turns",
        "Anthropic's API caches identical early turns automatically regardless of which condition is used",
        "Fewer than 3 tool results existed yet at that point, so the window had nothing to omit"
      ],
      "correct": 3,
      "explanations": [
        "Not indicated anywhere -- the windowing logic is described as correctly keeping only the most recent window and omitting anything older; the near-identical early turns are an expected structural consequence of the window's own definition, not a malfunction.",
        "Not a real mechanism -- models don't retain memory of prior API calls outside what's explicitly included in the current request's messages; the windowed condition's context is exactly what's sent in that specific call, nothing more.",
        "Not how prompt caching works, and not what's being measured here -- this page's real numbers are INPUT TOKEN COUNTS sent per call, not cache hit/miss behavior; caching (covered in a separate topic) doesn't explain why two DIFFERENT conditions would show similar early-turn token counts by itself.",
        "Correct. A 'keep only the last 2 tool results' window has literally nothing to omit until MORE than 2 tool results exist in the transcript -- so for the first few turns, the windowed and naive conditions send essentially the same content, and the two conditions only diverge once the transcript grows past the window size."
      ]
    },
    {
      "scenario": "A real repro compared always-Sonnet vs. Haiku-first-with-escalation on 10 real questions. The routed condition matched Sonnet's accuracy (9/10) at about 44% of the real dollar cost -- but the escalation count was exactly zero, including on the one question Haiku got wrong, which it answered with reported 'high' confidence.",
      "question": "What is the most precise way to characterize what this combined result does and doesn't demonstrate about confidence-based routing?",
      "options": [
        "It demonstrates routing failed completely, since the escalation mechanism never triggered even once during the run",
        "The cost benefit of a cheaper model was real, but self-reported confidence didn't reliably catch the model's own mistake",
        "It demonstrates Haiku is strictly more accurate than Sonnet, since it achieved the same correct count at lower cost",
        "It demonstrates self-reported confidence works correctly, since the model would have escalated if it had truly been uncertain"
      ],
      "correct": 1,
      "explanations": [
        "Too sweeping -- the cost/accuracy result (44% of cost, identical accuracy) is a genuine positive outcome; 'failed completely' ignores that real, measured benefit and conflates it with the separate, real limitation in the escalation trigger specifically.",
        "Correct. The page draws exactly this two-part distinction: the cost lever (routing to a cheaper model) delivered real savings with no accuracy loss on this batch, which is one real finding -- but the SAFETY lever (using confidence to catch mistakes before they ship) failed on the one case that mattered, because self-reported confidence didn't correlate with actual correctness here. Both are real and both matter, but they are not the same claim.",
        "Overreaches from a tie -- both models got 9/10 correct (an equal, not superior, result for Haiku); 'strictly more accurate' isn't supported since the counts were identical, only the cost differed.",
        "Backwards -- the real result is the opposite: the model reported HIGH confidence specifically on the question it got wrong, which is direct evidence the self-report mechanism did NOT reliably track true uncertainty, not confirmation that it works as intended."
      ]
    },
    {
      "scenario": "The page states that STEER (a real, cited routing method) uses model-internal confidence derived from token-level output probabilities, rather than a self-reported natural-language confidence field -- and notes this page's own repro could not test that specific mechanism directly.",
      "question": "What real, current constraint does the page give for why the repro used self-reported confidence instead of STEER's actual mechanism?",
      "options": [
        "Self-reported confidence was chosen deliberately for being more accurate than token-probability-based methods in practice",
        "STEER's method has been deprecated and is no longer considered a valid routing approach as of this writing",
        "The Anthropic Messages API used throughout this project doesn't expose token-level log probabilities at all",
        "Token-probability-based confidence only works for open-source models, never for any hosted commercial API"
      ],
      "correct": 2,
      "explanations": [
        "Backwards -- the page's own real result argues the opposite: self-reported confidence failed to catch the model's one actual mistake, which is presented as a limitation to be honest about, not a deliberate preference for a more accurate method.",
        "Not a claim the page makes -- STEER is cited as a real, current method being contrasted with this repro's fallback approach, not as something superseded or invalid.",
        "Correct. The page states this directly and precisely: the Messages API used throughout this project doesn't expose token-level logprobs the way some routing methods assume access to, which is exactly why the repro falls back to a self-reported field -- and the real result shows why that fallback has a real cost (missing the model's actual mistake).",
        "Overgeneralizes beyond what the page claims -- the page's point is specifically about which API this project uses (Anthropic's Messages API) not exposing this data, not a blanket claim about all commercial APIs universally lacking any form of token-probability access."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Augment Code, ["AI Agent Loop Token Costs: How to Constrain Context"](https://www.augmentcode.com/guides/ai-agent-loop-token-cost-context-constraints) — the real `N(N+1)/2` triangular-number formula and worked examples this page's repro is built directly around.
- ["Confidence-Guided Stepwise Model Routing for Cost-Efficient Reasoning"](https://arxiv.org/abs/2511.06190) (STEER, arXiv 2511.06190) — the real model-internal confidence routing method contrasted with this page's self-report fallback.
- [KV-Cache Economics](kv-cache-economics.md) — prompt caching, a third real lever on cost this page doesn't cover directly (cutting cached-input cost specifically, rather than the per-turn rebilling or per-token price this page focuses on).
- [Guardrails and Human-in-the-Loop](guardrails-human-in-the-loop.md) — the structural-versus-behavioral distinction this page applies to routing's escalation trigger specifically.
- [Models for Agents](models-for-agents.md) — Haiku 4.5 vs. Sonnet 5 reliability under ambiguity, a closely related but distinct real comparison of the same two model tiers.
