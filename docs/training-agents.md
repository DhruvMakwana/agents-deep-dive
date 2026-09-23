# Training Agents: Reward and Credit

!!! example "Hands-on"
    Full runnable recipe: [`training-agents/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/training-agents) in the companion cookbook — real DPO preference-pair construction from real completions, plus exact GRPO and DAPO formulas run on toy reward groups so their real documented failure mode and fix are actually visible in the numbers. No GPU, no gradient update — per this site's own standing decision, training content stays conceptual plus a CPU-only toy.

??? abstract "TL;DR — quick revision"
    - **GRPO replaces a critic with the group itself.** DeepSeekMath's own framing: "GRPO foregoes the critic model, instead estimating the baseline from group scores" — rewards for a group of sampled completions to the same prompt are "normalized by subtracting the group average and dividing by the group standard deviation," and that normalized value becomes every token's advantage.
    - **A real, exact run of that formula reproduces GRPO's own documented failure mode.** A group where every sample got the identical reward produced an advantage of exactly zero for all eight samples — DAPO's own paper names this the "gradient-decreasing problem": "if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients."
    - **DAPO's fix, run for real on the same degenerate group, is exactly what the paper describes**: over-sample and filter out any group whose accuracy is exactly 0 or exactly 1, keeping only groups with genuine variance — "leaving all prompts in the batch with effective gradients."
    - **Outcome reward and process reward answer different questions, with a measured real gap between them.** "Process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset" — a process-supervised model reported solving 78% of a representative MATH subset. A real toy repro shows exactly why: outcome-only credit can't tell a uniformly-bad trajectory from one where only the middle step failed; step-level credit can.
    - **DPO turns preference pairs directly into a classification loss, no separate reward model or RL loop needed** — "solve the standard RLHF problem with only a simple classification loss," reported to match or exceed PPO-based RLHF while being "stable, performant, and computationally lightweight." A real repro constructed one such pair from two real, independently sampled completions and a real judge call.

## GRPO: the group is the baseline

Proximal Policy Optimization (PPO), the RLHF-era default, needs a learned critic (value function) to estimate how much better or worse an action was than expected — a second model, trained alongside the policy, adding real memory and instability cost. DeepSeekMath's Group Relative Policy Optimization sidesteps that entirely: "GRPO foregoes the critic model, instead estimating the baseline from group scores." Sample a group of completions to the same prompt, score each one, and use the group's own statistics as the baseline — "rewards are normalized by subtracting the group average and dividing by the group standard deviation," and "sets the advantages of all tokens in the output as the normalized reward." No critic, no separate value network — the group tells you, relative to itself, which samples were actually better.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/training-agents/training_agents_docs.py:grpo"
```

!!! success "A real run — the exact formula, a real mixed group, and a real degenerate one"
    **Input** — a group of 8 binary rewards for a genuinely hard prompt (a mix of correct and incorrect samples): `[1, 1, 0, 1, 0, 0, 1, 0]`.

    **Output — real computed advantages**: `[1.0, 1.0, -1.0, 1.0, -1.0, -1.0, 1.0, -1.0]`. Every reward-1 sample gets a positive advantage, every reward-0 sample gets a negative one — exactly the group-relative signal the formula is supposed to produce, computed for real, not asserted.

    **Input** — a second group, for a prompt the model already always answers correctly: `[1, 1, 1, 1, 1, 1, 1, 1]`.

    **Output**: `[0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]`. Every advantage is exactly zero — not a bug in this recipe's code, a real, exact reproduction of the mechanism DAPO's paper names and explains next.

## DAPO: the gradient-decreasing problem, and the fix

That all-zero result above isn't a corner case nobody worries about — it's common enough in practice that DAPO built a named fix for it. The paper's own diagnosis: "Existing RL algorithm suffers from the gradient-decreasing problem when some prompts have accuracy equal to 1. For example for GRPO, if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients, shrinking the magnitude and increasing the noise sensitivity of the batch gradient, thereby degrading sample efficiency." A model that's already mastered a prompt (or one it always fails, symmetrically) contributes nothing to training from that prompt at all, while still costing a full group of real sampling compute to discover that.

DAPO's fix, dynamic sampling, is exactly what it sounds like: "we propose to over-sample and filter out prompts with the accuracy equal to 1 and 0 ... leaving all prompts in the batch with effective gradients and keeping a consistent number of prompts." Keep sampling until you find groups with real internal variance, and only train on those. This is one of DAPO's four named techniques (alongside Clip-Higher, Token-Level Policy Gradient Loss, and Overlong Reward Shaping), and the paper reports the combination reaching 50 points on AIME 2024 with a Qwen2.5-32B base model — ahead of DeepSeek-R1-Zero-Qwen-32B's 47 points, using only half the training steps.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/training-agents/training_agents_docs.py:dapo"
```

!!! success "A real run — the exact degenerate group, filtered and replaced"
    **Input** — three candidate groups, simulating repeated sampling attempts on the same prompt: two degenerate (`[1,1,1,1,1,1,1,1]`, twice) and one genuinely mixed (`[1,0,1,1,0,1,0,1]`).

    **Output**: the filter correctly skipped both degenerate groups (accuracy exactly 1.0, per DAPO's own stated condition) and kept the third — accuracy 0.625 — computing real, non-degenerate advantages from it: `[0.775, -1.291, 0.775, 0.775, -1.291, 0.775, -1.291, 0.775]`. The same formula that produced all-zeros above now produces a real, usable gradient signal, because the input it was given this time actually had variance to normalize against.

## Outcome reward vs. process reward

Both are answers to "what should the model be rewarded for," and they disagree about the unit that gets scored. Outcome supervision scores the final result — did the proof conclude correctly, did the task actually succeed. Process supervision scores each intermediate step on its own merits. PRM800K's own comparison is direct: "process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset," with a process-supervised model reported solving 78% of a representative MATH test subset — and the cost of that improvement is real too: 800,000 human step-level labels went into building it. Process reward models aren't free to build well, either — a later paper's own finding: Monte-Carlo-estimated PRM labels are flawed, and consensus filtering across multiple estimates is needed to get usable step-level signal at all.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/training-agents/training_agents_docs.py:credit_assignment"
```

!!! success "A real run — a toy trajectory, both credit-assignment methods, run on the identical steps"
    **Input** — a 3-step toy trajectory where only the middle step was actually the mistake: step 1 (correct sub-action), step 2 (the actual mistake), step 3 (correct sub-action). The trajectory failed overall, because of step 2 — final outcome reward: `-1`.

    **Output — outcome-only credit**, the single final reward broadcast across all three steps: `[-1, -1, -1]`. `outcome_only_can_distinguish_steps: false` — this credit assignment is bitwise identical to what a trajectory where *all three* steps were bad would have produced. There's no way to look at this output and recover which step actually caused the failure.

    **Output — process-level credit**, each step scored on its own: `[1, -1, 1]`. `process_reward_can_distinguish_steps: true` — the actual culprit is visible directly in the numbers, not just asserted.

This is the concrete version of the credit assignment problem: reinforcement learning needs to know which action in a trajectory deserves credit or blame, and a single scalar at the end of a multi-step trajectory structurally cannot answer that question on its own, no matter how well-tuned the final reward is. A 2026 survey of the field counts 69 papers (56 core methods) trying to close exactly this gap — from GRPO's group-relative baseline to methods like GiGPO's "anchor state grouping," which groups identical states that recur across different rollouts to get finer-grained, step-level credit while keeping GRPO's critic-free, low-memory properties.

## DPO: preference pairs, no separate reward model

Classic RLHF is a two-stage pipeline: train a separate reward model on human preference data, then run PPO against it. Direct Preference Optimization collapses this into one step — the paper's own framing: the approach "enable[s] extraction of the corresponding optimal policy in closed form, allowing us to solve the standard RLHF problem with only a simple classification loss." No reward model, no RL sampling loop during training — just a loss computed directly over `(prompt, chosen, rejected)` triples. The reported trade-off is favorable, not just simpler: DPO is described as "stable, performant, and computationally lightweight, eliminating the need for sampling from the LM during fine-tuning," while "exceed[ing] PPO-based RLHF in ability to control sentiment of generations" and matching or improving quality in summarization and dialogue.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/training-agents/training_agents_docs.py:dpo_pair"
```

!!! success "A real run — two real completions, a real judge verdict, a real preference pair"
    **Input**, identical for both completions: *"Write a one-sentence tagline for a fictional productivity app called Nimbus."* — no temperature was set explicitly (this SDK version's Messages API has no top-level `temperature` parameter at all), and default sampling still produced two genuinely different completions on two independent calls.

    **Output — completion A**: *"Nimbus: Your thoughts organized, your tasks flowing, your potential limitless."*
    **Output — completion B**: *"Nimbus: Rise above the chaos and let your tasks drift into focus."*

    **Output — the real judge verdict**: *"B is more concrete and specific with vivid metaphors ('drift into focus', 'rise above chaos') that better differentiate the product, while A relies on generic aspirational language that could apply to almost any productivity tool."*

    **The resulting DPO pair**, constructed directly from that verdict: `chosen` = completion B, `rejected` = completion A. This is the exact shape DPO's loss consumes — no reward model was trained to produce it, just one comparison call over two real samples.

## Interview angle

**Weak answer** to "how would you train an agent to be more reliable at a multi-step task": *"Just run RL with a reward for task success."* This names the reward signal and skips the actual hard part this page's own real run demonstrates twice: a single outcome reward can't tell which step in a multi-step trajectory deserves the blame, and (separately) a naive group-relative baseline silently contributes zero gradient on every prompt the model already handles perfectly or fails completely — which, on a real model partway through training, is not a rare edge case.

**Strong answer**: outcome reward and process reward solve different problems and have different costs. Outcome reward is cheap to compute (often just a verifier) but structurally can't do credit assignment across steps — this page's own toy trajectory shows the exact failure: identical credit for a uniformly-bad run and a run where only one step failed. Process reward fixes that but costs real labeling or estimation effort (PRM800K needed 800,000 human labels; MC-estimated alternatives need consensus filtering to be trustworthy at all). Separately, whichever reward you pick, the optimizer itself needs real variance to learn from — GRPO's group-relative baseline goes to exactly zero gradient on a zero-variance group, which is why DAPO's dynamic sampling (filter out accuracy-0-or-1 groups, keep sampling until you find real variance) is a documented, measured fix, not a hypothetical one.

**Follow-up to expect**: "when would you reach for DPO instead of a full RL loop like GRPO?" DPO's real advantage is that it needs only static preference pairs, not a live sampling loop against an evolving policy — cheaper and more stable, which is exactly why the paper reports it matching or beating PPO-based RLHF while eliminating "the need for sampling from the LM during fine-tuning." GRPO (and its multi-turn agentic descendants) earn their extra complexity specifically when the reward signal itself needs to come from actually running the policy — executing a multi-step trajectory, checking whether a tool call succeeded, verifying an intermediate state — which a static, pre-collected preference dataset structurally cannot capture.

## Build it yourself — 30 minutes

1. Implement `grpo_advantages` exactly as the formula reads: group mean, group standard deviation, `(reward - mean) / std`. Test it against a group with real variance, then deliberately test it against a group where every reward is identical — read the actual numbers, don't just assume what happens.
2. Build the dynamic-sampling filter: given several candidate groups, keep only the ones with accuracy strictly between 0 and 1. Feed it a sequence where the first few groups are degenerate and a later one isn't, and verify it actually skips the useless ones rather than training on them.
3. Construct a toy multi-step trajectory (3-5 steps) with a clear ground-truth "which step was actually wrong." Implement outcome-only credit assignment (broadcast the final reward to every step) and process-level credit (score each step individually), and check programmatically whether each method's output can actually distinguish the bad step from the good ones — don't just eyeball it.
4. Generate two real completions to the same prompt and have a real judge call rank them into a `(chosen, rejected)` pair. Read both completions and the judge's stated reason before trusting the pair — a judge call can fail to parse or produce a verdict that doesn't hold up, the same way any other LLM call can.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Training Agents">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real run computed GRPO advantages for a group where every one of 8 sampled completions received the identical reward (all correct). The result was an advantage of exactly 0.0 for all 8 samples.",
      "question": "What is the most accurate description of what this result demonstrates?",
      "options": [
        "DAPO's 'gradient-decreasing problem' -- zero variance, zero gradient",
        "A bug in the advantage formula needing a special case for perfect groups",
        "That the model has nothing left to learn from any prompt it handles well",
        "That group sizes of 8 are too small to produce meaningful advantages"
      ],
      "correct": 0,
      "explanations": [
        "Correct. DAPO's own paper names this exactly: 'if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients.'",
        "Misdiagnoses a real, documented property of the formula as an implementation error -- the zero result is mathematically correct given the formula, not a flaw needing a workaround.",
        "Overgeneralizes from one group's result to a permanent claim about the prompt -- the issue is this specific batch having no variance, not that the prompt is unlearnable from in general.",
        "Introduces an unsupported claim about group size -- the zero result occurs at any group size whenever every sample receives an identical reward; size isn't the variable at fault."
      ]
    },
    {
      "scenario": "DAPO's dynamic sampling filters out any group whose accuracy is exactly 0 or exactly 1, keeping only groups with accuracy strictly between those values. A team proposes simplifying this to: 'Just filter out any group with accuracy below 50%, since low-accuracy groups are clearly not useful.'",
      "question": "What's the strongest problem with that proposed simplification?",
      "options": [
        "The proposal is equivalent to DAPO's actual rule, just phrased differently",
        "It would discard groups with usable variance that DAPO's rule actually keeps",
        "The 50% threshold should be higher, closer to 90%, to match DAPO's behavior",
        "Filtering by accuracy is never a valid strategy for any RL training approach"
      ],
      "correct": 1,
      "explanations": [
        "Misstates the comparison -- a group with accuracy 0.3 is kept by the real rule and discarded by the proposed one, so the two are not equivalent.",
        "Correct. DAPO's actual condition keeps any group with accuracy strictly between 0 and 1 -- including accuracy 0.3, which still has real reward variance and produces a real, non-zero gradient. A 50% floor would discard genuinely useful training signal DAPO's real rule correctly retains.",
        "Proposes an arbitrary alternative threshold with no grounding -- DAPO's real rule isn't a threshold at all, it specifically excludes only the two boundary values (0 and 1).",
        "Overgeneralizes into an absolute claim the material contradicts -- DAPO's own dynamic sampling IS a form of filtering by accuracy, so filtering by accuracy is demonstrably valid as published."
      ]
    },
    {
      "scenario": "A real toy repro showed outcome-only credit assignment producing identical values ([-1, -1, -1]) for a 3-step trajectory where only the middle step was actually the mistake, while process-level credit correctly produced [1, -1, 1]. A candidate concludes: 'This proves outcome-only reward should never be used, since it always fails to identify which step is bad.'",
      "question": "What's the most accurate pushback on this conclusion?",
      "options": [
        "The conclusion is correct -- outcome-only reward provides no information at all",
        "Process-level reward is strictly better in every respect, including cost",
        "It shows one limitation (can't localize the bad step), not that it's worthless",
        "The result only applies to exactly 3-step trajectories, not longer ones"
      ],
      "correct": 2,
      "explanations": [
        "Overstates the finding -- outcome reward does provide real information (whether the trajectory succeeded overall), just not step-level localization within it.",
        "Contradicts the material directly -- process reward's real documented cost (800,000 human labels, or MC-estimation needing consensus filtering) is exactly why it isn't simply better in every respect.",
        "Correct. The demo's specific limitation is that outcome-only credit can't distinguish WHICH step caused a failure -- it says nothing about whether outcome reward is useful at all. It remains cheap (often just a verifier) and gives a valid trajectory-level signal.",
        "Fabricates a scope restriction with no basis -- the structural issue applies to any trajectory with more than one step, not specifically three."
      ]
    },
    {
      "scenario": "A real DPO-pair repro generated two completions to the same prompt with no explicit temperature setting, and the two completions were genuinely different. A candidate explains: 'This must mean the SDK silently applied a high default temperature to force diversity for this demo.'",
      "question": "What's the most accurate explanation of what actually happened?",
      "options": [
        "SDKs commonly apply hidden temperature overrides for preference-pair use cases",
        "Two identical completions would have been expected, so this suggests an error",
        "The judge call itself introduced the variation, not the two completion calls",
        "Default sampling alone can produce real variance, with no temperature setting"
      ],
      "correct": 3,
      "explanations": [
        "Fabricates an unsupported claim about hidden, use-case-specific SDK behavior -- there's no support for such a mechanism, and this SDK version doesn't even expose a top-level temperature parameter to set in the first place.",
        "Assumes determinism that isn't supported -- nothing about the API guarantees identical outputs across independent calls by default, and the real repro directly demonstrates otherwise.",
        "Misattributes the source of variation -- the judge call happens AFTER both completions already exist and differ; it evaluates the difference, it doesn't create it.",
        "Correct. Two independent calls with no temperature specified still produced genuinely different completions -- consistent with default sampling variance being normal model behavior, not something requiring explicit configuration to observe."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Shao et al., ["DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models"](https://arxiv.org/abs/2402.03300) (arXiv 2402.03300, 2024-02) — GRPO's group-relative advantage, critic-free baseline.
- Yu et al., ["DAPO: An Open-Source LLM Reinforcement Learning System at Scale"](https://arxiv.org/abs/2503.14476) (arXiv 2503.14476, 2025-03) — the gradient-decreasing problem, dynamic sampling, and the other three named techniques.
- Rafailov et al., ["Direct Preference Optimization: Your Language Model is Secretly a Reward Model"](https://arxiv.org/abs/2305.18290) (arXiv 2305.18290, 2023-05) — preference pairs as a direct classification loss.
- Lightman et al., ["Let's Verify Step by Step"](https://arxiv.org/abs/2305.20050) (arXiv 2305.20050, 2023-05, PRM800K) — process supervision vs. outcome supervision on MATH.
- Zhang et al., ["Lessons of Developing Process Reward Models in Mathematical Reasoning"](https://arxiv.org/abs/2501.07301) (arXiv 2501.07301, 2025-01) — Monte-Carlo PRM pitfalls and consensus filtering.
- Feng et al., ["GiGPO"](https://arxiv.org/abs/2505.10978) (arXiv 2505.10978, 2025-05) — anchor-state grouping for fine-grained, step-level credit assignment while keeping GRPO's critic-free properties.
- Zhang et al., ["Credit Assignment in Reinforcement Learning for LLMs"](https://arxiv.org/abs/2604.09459) (arXiv 2604.09459, 2026-04) — a survey of 69 papers (56 core credit-assignment methods).
