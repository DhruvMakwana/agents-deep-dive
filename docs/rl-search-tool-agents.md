# RL for Search and Tool Agents

!!! example "Hands-on"
    Full runnable recipe: [`rl-search-tool-agents/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/rl-search-tool-agents) in the companion cookbook — a real repro of ToolRL's reward-design claim on real tool-call completions. Actually training these methods needs GPU-hours this cookbook doesn't have, so — per this site's own standing decision for training content — the rest stays conceptual, grounded in each paper's own verified abstract and results.

??? abstract "TL;DR — quick revision"
    - **Search-R1 trains an LLM to interleave real search calls with reasoning, using outcome-only reward.** Real, verified: *"Search-R1 optimizes LLM reasoning trajectories with multi-turn search interactions... Experiments on seven question-answering datasets show that Search-R1 improves performance by 41% (Qwen2.5-7B) and 20% (Qwen2.5-3B) over various RAG baselines."* R1-Searcher and ReSearch train the same joint reasoning+search behavior via RL with no supervised reasoning-step data at all — R1-Searcher via a two-stage outcome-only process, ReSearch by treating search as guided by the model's own "text-based thinking."
    - **ToolRL studied reward design for tool use directly, and found coarse reward isn't enough.** Real, verified: *"coarse-grained reward signals, such as answer matching, fail to offer the finegrained feedback required for effective learning."* Its fix — decomposing reward into tool-name, parameter-name, and parameter-value components — produced *"a 17% improvement over base models and a 15% gain over SFT models."* This page's own repro reproduces the exact mechanism on real tool calls.
    - **ReTool shows RL can teach WHEN to reach for a tool, not just how.** Real, verified: on AIME, *"67% accuracy with 400 training steps"* vs. a text-only RL baseline's *"40% accuracy, 1080 steps"* — higher accuracy in roughly a third of the training steps — with the paper describing an emergent *"aha moment"* where the model starts self-correcting code mid-reasoning without being explicitly taught to.
    - **RAGEN names a real, recurring multi-turn RL failure mode: the Echo Trap.** Real, verified: *"a recurring instability pattern, Echo Trap, where agents overfit to locally rewarded reasoning patterns, marked by reward variability collapse, entropy drop, and gradient spikes."* Models converge to *"near-identical phrasing... without justification"* — real, multi-turn RL training doesn't just get slower when it goes wrong, it can collapse into repeating a shallow, memorized pattern that happens to score well locally.

## The shared mechanism: outcome-only RL for interleaved reasoning and search

Three real, separate 2026 papers converge on the same shape: train an LLM to decide *for itself*, during reasoning, when to issue a search query — using only whether the final answer was right, no supervised data on the reasoning steps themselves.

**Search-R1** is explicit about the mechanism: the model "learns to autonomously generate (multiple) search queries during step-by-step reasoning with real-time retrieval," using dedicated tokens to mark search calls and the retrieved results returned into the same rollout, trained with either PPO or GRPO and "a simple outcome-based reward function." The real, verified headline: *"Search-R1 improves performance by 41% (Qwen2.5-7B) and 20% (Qwen2.5-3B) over various RAG baselines under the same setting"* — across seven real QA datasets.

**R1-Searcher** takes a two-stage route to the same behavior: "a novel two-stage outcome-based RL approach designed to enhance the search capabilities of LLMs," which "relies exclusively on RL, without requiring process rewards or distillation for a cold start" — stage one teaches the model to invoke search at all, stage two teaches it to use search effectively toward a correct answer. Its real, verified claim: the method "significantly outperforms previous strong RAG methods, even when compared to the closed-source GPT-4o-mini."

**ReSearch** frames the same joint training slightly differently: search isn't a separately triggered action bolted onto reasoning, it's "guided by text-based thinking" as an integral part of the reasoning chain itself, with "search results subsequently influenc[ing] further reasoning." Real and notable: the paper reports the trained models "naturally elicit advanced reasoning capabilities such as reflection and self-correction" as a side effect of the RL process — nobody wrote a reflection-training objective; it emerged from training the model to get multi-hop answers right via search.

The common thread across all three: none of them supervise *how* to search. They only ever reward whether the final answer was correct, and let RL discover the search strategy — which is exactly why reward design for the intermediate tool calls themselves becomes its own real, separate research question.

## ToolRL: reward design isn't an afterthought, it's the actual lever

ToolRL asks a narrower, sharper question: given that a tool call itself (not just a final text answer) is what's being scored, what should the reward function for *that* actually look like? Its own framing of the gap: *"multiple tools may be invoked with diverse parameters, and coarse-grained reward signals, such as answer matching, fail to offer the finegrained feedback required for effective learning."* The paper's fix, verified: decompose the reward into per-component scores — tool selection, parameter names, parameter values — rather than one all-or-nothing match. Trained with GRPO, the real, verified result: *"a 17% improvement over base models and a 15% gain over SFT models."*

## Repro: what binary reward can't see, on real tool calls

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/rl-search-tool-agents/rl_search_tool_agents_docs.py:reward-functions"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/rl-search-tool-agents/rl_search_tool_agents_docs.py:trials"
```

Four real trials against the identical target action — a 30-minute "Team Sync" event on 2026-10-05 — each phrased to elicit a different, real kind of imperfection: fully specified, a vaguely-worded duration, a request that could plausibly map to a different available tool, and a paraphrased duration.

!!! success "A real run, unforced — binary vs. fine-grained reward on the same real completions"
    | Trial | Real call | Binary reward | Fine-grained reward |
    |---|---|---|---|
    | Fully specified | Exact match | 1.0 | 1.0 |
    | Vague duration ("keep it brief") | Right tool, right params, guessed `duration_minutes: 15` vs. expected `30` | **0.0** | **0.8889** |
    | Decoy tool available | Called `create_reminder` instead of `create_calendar_event` | **0.0** | **0.2222** |
    | Paraphrased duration ("half an hour") | Exact match | 1.0 | 1.0 |

    The real finding is the middle two rows: binary reward scores the "vague duration" call and the "decoy tool" call **identically** — both 0.0 — even though one call got the tool right and every parameter right except a single reasonably-guessed value, and the other picked an entirely different tool. A policy trained on binary reward alone gets zero gradient signal telling those two real, different completions apart. Fine-grained reward gives them real, different scores — 0.8889 vs. 0.2222 — the exact differentiation ToolRL's own abstract argues coarse reward can't provide.

## ReTool: RL can teach *when*, not just *how*

ReTool targets a different question: given a model that already knows how to write and run code, when during a reasoning chain should it actually reach for that tool? Its approach combines a cold-start phase — "synthetic cold-start data generation to produce code-augmented long-form reasoning traces" — with RL that "leverages task outcomes as rewards to iteratively refine the model's tool use strategy, enabling autonomous discovery of optimal tool invocation patterns." The real, verified numbers, on AIME with a 32B model: **67% accuracy with 400 training steps**, against a text-only RL baseline's **40% accuracy at 1080 steps** — meaningfully higher accuracy in roughly a third of the training steps. The paper also documents a real, named emergent behavior: "code self-correction, signaling an 'aha moment' in which the model autonomously masters adaptive tool use" — the model wasn't explicitly trained to notice and fix its own buggy code mid-reasoning; that behavior showed up as a byproduct of outcome-based RL over enough training steps.

## RAGEN: naming what goes wrong in multi-turn agent RL

Every method above assumes multi-turn RL training actually converges cleanly. RAGEN studies what happens when it doesn't, across four stylized multi-turn agent environments, and names a real, recurring failure mode directly: *"a recurring instability pattern, Echo Trap, where agents overfit to locally rewarded reasoning patterns, marked by reward variability collapse, entropy drop, and gradient spikes."* Concretely, the paper reports models converging "to near-identical phrasing focused on choosing [an action] without justification" — the policy doesn't fail by getting worse in an obvious way; it collapses into a narrow, repetitive, shallow pattern that happens to score adequately under the local reward signal, at the cost of the actual reasoning the training was meant to produce.

RAGEN's proposed fix, StarPO-S, addresses this with "trajectory filtering, critic incorporation, and gradient stabilization." The paper's third real finding sharpens exactly why ToolRL's reward-design question and RAGEN's stability question are connected: *"without fine-grained, reasoning-aware reward signals, agent reasoning hardly emerge[s] through multi-turn RL and they may show shallow strategies or hallucinated thoughts."* Coarse reward doesn't just fail to differentiate good completions from bad ones (ToolRL's finding) — across many real turns of RL training, it can actively drive the policy toward the Echo Trap's collapsed, shallow behavior, because a shallow, repeated pattern is often the path of least resistance to a locally-adequate reward.

## What this means in practice

The six real methods on this page split cleanly into two real, connected layers. Search-R1, R1-Searcher, and ReSearch answer "should the model decide *when* to call a tool during reasoning?" — yes, and outcome-only RL is enough to teach that, without any supervised reasoning-step data. ToolRL and ReTool answer "once it's decided to call a tool, how do you reward *that specific action* well?" — coarse, all-or-nothing reward isn't enough; decomposed, fine-grained reward (ToolRL) and outcome-based iterative refinement over many real training steps (ReTool) both do measurably better. RAGEN answers the question underneath both: "what happens when the reward signal training either of these is too coarse, over many multi-turn rollouts?" — real, documented instability, named and diagnosable, not just "training got noisy."

## Interview angle

**Weak answer** to "how would you train an agent to use tools better with RL?": *"Give it a reward of 1 for a correct final answer and 0 otherwise, and run PPO."* This page's own real repro is a direct, measured counterexample at the level of a single tool call: binary reward assigned the identical score (0.0) to a call that was almost entirely correct and a call that used the wrong tool entirely — real, zero differentiation between two very different real mistakes.

**Strong answer**: name the two real, separate design questions this page distinguishes — whether to let the model decide *when* to act (Search-R1/R1-Searcher/ReSearch show outcome-only RL is sufficient for that) versus how to reward the *specific action* once taken (ToolRL and ReTool show this needs finer-grained signal, and ReTool shows RL can improve *timing* specifically, not just correctness). Then name the real risk of getting the reward too coarse across many turns: RAGEN's Echo Trap, a real, documented case where multi-turn RL collapses into a shallow, locally-rewarded pattern rather than the reasoning it was meant to produce.

**Follow-up to expect**: "how would you detect an Echo Trap in your own training run before it wastes a lot of compute?" RAGEN's own real, verified signature gives a direct, checkable answer: watch for reward variability collapsing, entropy dropping, and gradient spikes appearing together — not just a flattening loss curve, which could mean convergence rather than collapse. The qualitative tell this page's own material adds: sample actual rollouts periodically and check whether the model's stated reasoning is getting more repetitive and less justified over training, since RAGEN's own finding is that the model can keep scoring adequately while doing exactly that.

## Build it yourself — 30 minutes

1. Pick a real tool-calling task with more than one parameter, and write both a binary reward function (exact match) and a decomposed reward function (component-by-component, following ToolRL's tool-name/parameter-name/parameter-value split).
2. Generate a handful of real completions across deliberately varied prompt phrasings — some unambiguous, some vague on exactly one detail, some that could plausibly trigger a different but available tool.
3. Score every completion both ways and look specifically for pairs where binary reward ties two completions that a human would clearly rank differently — that tie is the real, concrete cost of coarse reward, visible without training anything.
4. If you have access to a small RL setup, try training on binary-only reward vs. the fine-grained version on the identical task, and watch specifically for RAGEN's named signature (reward variability collapse, entropy drop, gradient spikes) under whichever reward function is coarser — that's the real, connected risk this page's material predicts.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: RL for Search and Tool Agents">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro scored two different real tool-call completions against the identical target action using both binary reward (exact match only) and a fine-grained, decomposed reward (tool name / parameter names / parameter values). One completion used the right tool with one reasonably guessed wrong value; the other used a completely different tool. Binary reward scored both 0.0.",
      "question": "What is the most precise real problem this identical binary score demonstrates?",
      "options": [
        "Binary reward assigns zero gradient signal distinguishing a near-correct completion from a completely wrong one",
        "Binary reward is always a worse choice than fine-grained reward for every possible RL training scenario",
        "The fine-grained reward function contains a bug, since two genuinely different completions should never both score low",
        "Binary reward only fails when exactly two tools are available for the model to choose between"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, measured finding is that binary reward gave the SAME score (0.0) to two meaningfully different real completions -- one nearly perfect, one entirely wrong -- meaning a policy trained on it alone has no signal telling those two cases apart, exactly the gap ToolRL's own verified claim describes.",
        "Overstates the claim -- the page doesn't argue binary reward is categorically worse in every scenario (e.g. it's fine for genuinely all-or-nothing tasks); the specific, demonstrated problem is about DIFFERENTIATING degrees of correctness within multi-parameter tool calls.",
        "Backwards -- the fine-grained reward function correctly gave the two completions DIFFERENT scores (0.8889 vs 0.2222), which is it working as intended; it's binary reward that failed to differentiate them, not a bug in the fine-grained one.",
        "Not what the repro shows or claims -- the identical-score problem is about reward GRANULARITY (exact-match vs. component-wise), not about how many tools happen to be available in any specific trial; it would recur with any number of available tools."
      ]
    },
    {
      "scenario": "RAGEN's real, verified abstract names a recurring instability pattern in multi-turn LLM agent RL training called the 'Echo Trap,' described as being 'marked by reward variability collapse, entropy drop, and gradient spikes,' where models converge to near-identical phrasing without justification.",
      "question": "What would be the most accurate way to distinguish a real Echo Trap from ordinary, healthy training convergence, based on RAGEN's own described signature?",
      "options": [
        "Ordinary convergence and the Echo Trap are indistinguishable from training metrics alone and require no further investigation",
        "The Echo Trap is defined purely by the loss curve flattening out over time, regardless of any other training signal",
        "Any drop in entropy during training is sufficient on its own to confirm an Echo Trap has occurred",
        "Reward variability collapse, entropy drop, and gradient spikes together signal the Echo Trap specifically"
      ],
      "correct": 3,
      "explanations": [
        "Contradicted directly -- RAGEN's own abstract gives a specific, named, checkable signature (the three co-occurring signals), which is precisely a way to distinguish it from other training dynamics, not evidence that it can't be distinguished at all.",
        "Incomplete -- a flattening loss curve alone is ambiguous between genuine convergence and collapse; RAGEN's own named signature requires the CO-OCCURRENCE of reward variability collapse, entropy drop, AND gradient spikes together, not loss flattening in isolation.",
        "Overstates a single signal -- entropy naturally decreases somewhat during normal training as a policy becomes more confident and less exploratory; RAGEN's own definition requires it to co-occur with reward variability collapse and gradient spikes, not to appear alone.",
        "Correct. RAGEN's own quoted definition names three co-occurring signals as the Echo Trap's signature, and the paper's own qualitative description adds that models converge to near-identical, unjustified phrasing -- checking rollout text directly for that repetition, alongside the three metrics, is the most complete match to what RAGEN actually documented."
      ]
    },
    {
      "scenario": "Search-R1, R1-Searcher, and ReSearch are all real, separate 2026 papers that train LLMs to interleave search calls with reasoning using reinforcement learning.",
      "question": "What real, shared design choice do all three papers make regarding supervision of the reasoning process itself?",
      "options": [
        "All three require large amounts of supervised data labeling the correct reasoning steps before any RL training begins",
        "All three rely only on final-answer correctness, without supervising intermediate reasoning or search steps",
        "All three require a human reviewer to approve each individual search query before it is allowed to execute",
        "All three use a separately trained critic model to score every intermediate reasoning step during training"
      ],
      "correct": 1,
      "explanations": [
        "Directly contradicted -- the whole point of outcome-based RL in these papers is to AVOID needing supervised data on the reasoning steps; R1-Searcher's own quote is explicit that it works 'without requiring process rewards or distillation for a cold start,' and Search-R1/ReSearch make the same real design choice.",
        "Correct. All three papers' real, verified descriptions confirm this shared choice: Search-R1 uses 'a simple outcome-based reward function,' R1-Searcher is explicitly 'outcome-based... without requiring process rewards,' and ReSearch trains 'without using any supervised data on reasoning steps' -- only the final answer's correctness supervises training in each case.",
        "Not a real mechanism described in any of the three papers -- these are automated RL training loops without a human-in-the-loop approval step on individual search queries during training.",
        "Not accurate for this shared design choice -- GRPO (used by at least Search-R1 and ReSearch) specifically avoids a separate critic model, estimating the baseline from the sampled group instead; a critic-based approach is the PPO-style alternative these methods are contrasted against, not what they share."
      ]
    },
    {
      "scenario": "ReTool reports a real, verified result on AIME: a 32B model reaches 67% accuracy after 400 training steps, compared to a text-only RL baseline reaching 40% accuracy after 1080 steps.",
      "question": "What is the most precise interpretation of what this specific comparison demonstrates about ReTool's contribution?",
      "options": [
        "ReTool proves that tool-augmented models are always more accurate than text-only models on any reasoning benchmark",
        "The 1080-step baseline failed to converge at all, which is why its accuracy remained lower than ReTool's",
        "ReTool reached higher accuracy while needing meaningfully fewer training steps than the text-only baseline here",
        "ReTool's higher accuracy came entirely from a larger model size rather than from its RL training approach"
      ],
      "correct": 2,
      "explanations": [
        "Overgeneralizes a single benchmark result into a universal claim -- the real, verified numbers are specific to AIME with this particular model and setup; the page doesn't claim this result generalizes to 'any reasoning benchmark.'",
        "Not stated or implied -- there's no claim the baseline 'failed to converge'; it's presented as reaching a real, specific accuracy figure (40%) after its own training budget, a comparison point, not a description of a broken run.",
        "Correct. The real, verified numbers show BOTH a higher accuracy (67% vs. 40%) AND fewer training steps (400 vs. 1080) for ReTool's approach on this benchmark -- roughly a third of the steps for a substantially better result, which is precisely the comparison the page reports.",
        "Not supported by the given comparison -- the page attributes the result to ReTool's RL training approach (the cold-start-plus-RL method and outcome-based refinement), not to model size; no model-size confound is described in this comparison."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- ["Search-R1: Training LLMs to Reason and Leverage Search Engines with Reinforcement Learning"](https://arxiv.org/abs/2503.09516) (arXiv 2503.09516) — the real, verified outcome-based multi-turn search RL framework and its 41%/20% headline numbers.
- ["R1-Searcher"](https://arxiv.org/abs/2503.05592) (arXiv 2503.05592) — the real, verified two-stage outcome-only RL approach to eliciting search capability.
- ["ReSearch: Learning to Reason with Search for LLMs via Reinforcement Learning"](https://arxiv.org/abs/2503.19470) (arXiv 2503.19470) — the real, verified framing of search as integral to the reasoning chain, and its emergent reflection/self-correction finding.
- ["ToolRL: Reward is All Tool Learning Needs"](https://arxiv.org/abs/2504.13958) (arXiv 2504.13958) — the real, verified reward-design study this page's own repro is built directly around.
- ["ReTool: Reinforcement Learning for Strategic Tool Use in LLMs"](https://arxiv.org/abs/2504.11536) (arXiv 2504.11536) — the real, verified cold-start-plus-RL method and its AIME numbers.
- ["RAGEN: Understanding Self-Evolution in LLM Agents via Multi-Turn Reinforcement Learning"](https://arxiv.org/abs/2504.20073) (arXiv 2504.20073) — the real, verified Echo Trap failure mode and the StarPO-S stabilization fix.
- [Training Agents: Reward and Credit](training-agents.md) — GRPO, DAPO, outcome vs. process reward, and DPO; the general RL-training mechanics this page's six methods apply specifically to search and tool use.
- [Tool Design](tool-design.md) — endpoint-wrapper vs. consolidated vs. code-execution tool surfaces, a complementary, non-RL angle on what makes a tool call easy or hard to get right in the first place.
