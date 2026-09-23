# Evaluating Agents

!!! example "Hands-on"
    Full runnable recipe: [`evaluating-agents/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/evaluating-agents) in the companion cookbook — outcome vs. trajectory grading, pass@k vs. pass^k on 5 real trials, and a capability-motivated prompt change that caught a real regression in its own grader.

??? abstract "TL;DR — quick revision"
    - **Outcome grading and trajectory grading answer different questions, and can disagree.** Outcome asks "was the final answer right?" Trajectory asks "did it get there the intended way?" The Holistic Agent Leaderboard's own log audits caught agents "searching for the benchmark on HuggingFace instead of solving a task" — a case an outcome-only grader would have scored as a pass.
    - **pass@k and pass^k measure opposite things.** pass@k asks whether at least one of k tries succeeded (generous — useful for capability ceilings). pass^k asks whether *every one* of k tries succeeded (strict — the real reliability question, since a production agent gets one real try per request, not k with a human picking the best). τ-bench's own numbers show why the gap matters: GPT-4o's retail success rate falls from under 50% at pass@1 to under 25% at pass^8.
    - **A real capability-motivated prompt change caught a genuine, clean regression** — not in the model's reasoning, in the *grader*. Told to spell out numbers in words for a plausible accessibility reason, the model's arithmetic stayed completely correct ("one hundred divided by four equals twenty-five") — every task in a digit-matching regression suite still failed, because the grader, not the model, broke.
    - **Infrastructure is a real, measurable confound in agent benchmarks.** Running the identical model and benchmark across container configurations from strict to uncapped produced a 6-percentage-point swing (p < 0.01) on Terminal-Bench 2.0 — bigger than many reported leaderboard gaps — purely from resource allocation, with nothing about the model changing at all.
    - **Capability evals and regression evals serve different jobs and shouldn't be conflated.** A capability eval asks how good the agent could be — expensive, exploratory, run rarely. A regression eval asks whether a specific change broke something that used to work — cheap, narrow, run on every change. This page's own regression-suite experiment is exactly that second job, and it worked precisely because the suite was simple enough to run constantly.

## Outcome vs. trajectory

Outcome grading checks the final result: did the agent produce the right answer, the right file diff, the right database state. It's cheap, unambiguous, and automatable — and it has a documented blind spot: an agent can reach the right outcome through a process nobody actually wanted. The Holistic Agent Leaderboard's own log-level audits found exactly this in the wild: agents "searching for the benchmark on HuggingFace instead of solving a task," or, in a different case, "misusing credit cards in flight booking tasks" — outcomes that a grader checking only the final state would have scored as passing, because nothing about *how* the result was reached was ever inspected. Trajectory grading closes that gap by checking the process itself — which tools got called, in what order, with what arguments — at the cost of being harder to define and more expensive to check than a single final-state comparison.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/evaluating-agents/evaluating_agents_docs.py:outcome_vs_trajectory"
```

!!! success "A real run — both outcome and trajectory captured, for both a guessable and an unguessable fact"
    **Input, well-known fact**: *"Answer this question. You may call get_reference_fact to check the reference database if you need to.\n\nWhat is the boiling point of water at sea level, in Celsius?"* — a fact the model could plausibly answer correctly without ever calling the tool.

    **Output**: the model called `get_reference_fact` anyway, then answered *"The boiling point of water at sea level is **100 degrees Celsius**."* Outcome: correct. Trajectory: tool called, as intended.

    **Input, unguessable fictional fact**: the identical instruction pattern, asking *"What year was the Halcyon Reach colony founded?"* — a fictional fact with no way to answer correctly except by calling the tool.

    **Output**: the model called the tool, received the fact (2041), and answered *"The Halcyon Reach colony was founded in **2041**."* Outcome: correct. Trajectory: tool called — the only way this outcome could have been correct at all.

    No divergence in this real run — the model didn't take the shortcut of skipping verification even when it plausibly could have. That's a genuine, disclosed negative result, not a wasted test: this exact mechanism (checking whether the tool was actually called, not just whether the final answer was right) is precisely what would have caught the Holistic Agent Leaderboard's real documented cases if this model had behaved differently. Knowing the check doesn't fire on well-behaved runs is part of knowing it would fire on badly-behaved ones.

## pass@k vs. pass^k

Both metrics run the same task k independent times, and they ask opposite questions of the same data. **pass@k**: did at least one of the k tries succeed? This is the metric behind "best-of-n" framings and capability ceilings — useful for asking "can this model do this at all," generous by construction. **pass^k**, introduced by τ-bench specifically to measure agent reliability, asks the stricter question: did *every one* of the k tries succeed? That's the metric that actually matches production reality, where an agent handling a real customer request doesn't get k attempts with a human picking the best one — it gets one, and pass^k is the probability that one attempt looks as good as all the others. τ-bench's own reported numbers make the gap concrete: GPT-4o's retail-domain success rate is under 50% at pass@1 (already not great), and — in Sierra's own description — falls to under 25% at pass^8, "a staggering 60% drop" from the single-shot number.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/evaluating-agents/evaluating_agents_docs.py:pass_at_k"
```

!!! success "A real run — 5 independent trials, the same question, every answer shown"
    **Input**, identical across all 5 trials: *"A committee of 3 people is chosen at random from a group of 5 men and 4 women. What is the probability that the committee has at least 2 women? Give your final answer as a simplified fraction, on its own line, in the form: Final answer: a/b"* Ground truth (computed independently): 17/42.

    **Output — all 5 trials, real final lines**: trial 1 through 5 each worked through the same two cases (exactly 2 women, exactly 3 women) via combinations — $\binom{4}{2}\binom{5}{1} = 30$ and $\binom{4}{3}\binom{5}{0} = 4$, for $34/84$ simplified to $17/42$ — and every trial ended with the identical line: `Final answer: 17/42`.

    **5/5 correct. pass@5 = true. pass^5 = true.** No gap observed on this specific task — an honest result, not a disappointing one: this combinatorics problem, at this difficulty, showed perfect reliability for Claude Haiku 4.5 across 5 real trials. That doesn't contradict τ-bench's reported 50%-to-25% drop; it shows that the gap between pass@k and pass^k is a property of *how much genuine variance a specific task has for a specific model*, not something inherent to the metrics — τ-bench's tasks (multi-turn, tool-using, policy-constrained customer service) have far more room for a single trial to go sideways than one well-structured probability question does.

## Capability vs. regression suites

Anthropic's own framing draws a clean, practical line: a **capability eval** asks how good the agent could be — often exploratory, expensive to run, and used to guide real development decisions. A **regression eval** asks a narrower, cheaper question: did a specific change break something that used to work? It has to be fast and stable enough to run on every change, the same job a unit-test suite does in ordinary software. Conflating the two is a real, common mistake: a capability eval's occasional, expensive, deep-dive numbers aren't a substitute for a small, boring, constantly-run regression suite — and a regression suite that never catches anything isn't necessarily a badly-designed one, since "still passing" is the correct default outcome most of the time it runs.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/evaluating-agents/evaluating_agents_docs.py:capability_vs_regression"
```

!!! success "A real run — a plausible capability-motivated change, and a real, clean regression"
    **Input, baseline condition**: 4 trivial arithmetic tasks ("What is 2 + 2?", "What is 10 * 5?", "What is 100 / 4?", "What is 6 squared?"), plain system prompt: *"Answer the arithmetic question directly."* **Output**: all 4 answers contained the correct literal digits (e.g. *"100 / 4 = 25"*) — **4/4 passed** an automated grader checking for the expected digit string in the reply.

    **Input, capability-motivated condition**: the identical 4 tasks, system prompt changed to: *"Express all numeric answers in words, not digits (e.g. 'forty-two', not '42'), as required by the new accessibility guideline. This applies to every number in your response, no exceptions."* — a plausible, real reason to change a system prompt.

    **Output**: *"Two plus two equals four."* … *"Ten multiplied by five equals fifty."* … *"One hundred divided by four equals twenty-five."* … *"Six squared is thirty-six."* Every answer is **completely correct arithmetic** — and **all 4 tasks failed** the same automated grader, because none of "four," "fifty," "twenty-five," "thirty-six" contains the literal digit string the grader was checking for.

    This is a real, clean regression, and it's worth being precise about what actually regressed: not the model's reasoning — the grader. The accessibility-motivated change was a reasonable thing to want, and it silently broke an automated check that assumed a format nobody thought to re-verify. This is exactly the failure mode a regression suite exists to catch cheaply, before a much more expensive capability eval — or a real user — catches it instead.

## Infrastructure noise

One more source of variance in agent evals doesn't come from the model or the task at all: the hardware the eval runs on. Anthropic ran the identical model, harness, and task set (Terminal-Bench 2.0) across six container resource configurations, from strict 1x enforcement to fully uncapped — nothing about the agent changed between runs, only how much CPU and memory it had available. The result: a 6-percentage-point gap (p < 0.01) between the most- and least-resourced configurations — larger than many reported gaps between competing systems on public leaderboards. The mechanism is largely infrastructure-level failures, not reasoning failures: the infra error rate itself fell from 5.8% at 1x headroom to 2.1% at 3x (p < 0.001), down to 0.5% uncapped. A cross-check on SWE-bench, varying RAM up to 5x baseline across 227 problems, found a smaller but still real effect (+1.54 percentage points at 5x versus 1x) — smaller because SWE-bench's tasks are less resource-intensive than Terminal-Bench 2.0's, which is itself a useful finding: the size of the infrastructure confound depends on the task's own resource profile, not just on infrastructure existing. Anthropic's own recommendation follows directly: benchmark maintainers should publish recommended resource specs and enforcement methodology, and treat resource configuration "as a first-class experimental variable, documented and controlled" — the same way a rigorous experiment controls for any other variable that could explain a result. This page didn't reproduce the experiment itself (it needs real container orchestration across multiple resource tiers, not a lightweight script), but the numbers above are real, cited, and worth internalizing before trusting a close leaderboard gap: a 2-point lead could be silicon, not intelligence.

## Interview angle

**Weak answer** to "how would you evaluate a new agent before shipping it": *"Run it against a benchmark and check the pass rate."* This treats evaluation as one number from one run, and it can't explain any of this page's own real findings — not the outcome/trajectory gap (a single pass rate can't tell you whether the agent got there honestly), not pass@k vs. pass^k (a single run can't distinguish "usually works" from "always works"), and not the regression-suite finding (a one-off capability number wouldn't have caught a grader silently breaking on a routine prompt change).

**Strong answer**: evaluation is at least three separate questions, each needing a different setup. What's the agent capable of, at its ceiling — a capability eval, run occasionally, expensive, exploratory. Did a specific change break something that used to work — a regression suite, run constantly, cheap, narrow, and this page's own real trace shows it catching a failure a capability eval wouldn't even have been looking for. And is the agent's success outcome-real or trajectory-real — checked by actually inspecting the process, not just the final state, because the literature has real, documented cases (searching for a benchmark instead of solving the task; misusing a payment tool) where those two disagree. Layered under all three: how many trials is the reported number even based on, and was the hardware held constant — a single pass@1 number on unspecified infrastructure is not evidence, it's a claim.

**Follow-up to expect**: "if your own pass@k and pass^k came out identical, doesn't that mean the distinction doesn't matter in practice?" No — it means this specific task, at this difficulty, for this model, had low variance in this run. τ-bench's own published numbers (a 60% relative drop from pass@1 to pass^8 in a genuinely harder, multi-turn, tool-using domain) are the evidence the distinction matters in general; this page's clean result is evidence that the *size* of the gap is task-dependent, which is itself the useful, precise version of the claim — not "pass@k and pass^k always diverge," but "check both, because whether they diverge tells you something real about the task."

## Build it yourself — 30 minutes

1. Pick a task where the *right way to get there* is checkable — a tool that must be called, a specific sequence of steps. Run it once, then grade it two ways: outcome only (is the final answer right?) and trajectory (did it take the intended path?). Deliberately try a version of the prompt that makes skipping the intended path plausible, and see if the two graders ever disagree.
2. Run one task 5 times with real API calls (not a mock). Compute pass@5 and pass^5 from the real results, not an assumption. If they're identical, that's information about the task's variance, not a failed experiment.
3. Build a tiny regression suite — 3 or 4 trivial tasks with an exact-match or digit-match grader. Make one plausible, unrelated system-prompt change (a tone requirement, a formatting rule) and re-run the suite. Check whether it still passes — and if it doesn't, check whether the *model* regressed or the *grader* did.
4. If you have access to adjustable compute (a container resource limit, a local model's thread count), try running the identical eval at two different resource levels and see whether the score moves at all before trusting a close comparison between two systems.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Evaluating Agents">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro gave a model the option to call a verification tool or answer directly, on both a well-known fact and an unguessable fictional fact. In both cases the model called the tool and got the correct outcome -- no divergence between outcome and trajectory was observed.",
      "question": "What's the most accurate takeaway from this specific result?",
      "options": [
        "The checking mechanism worked correctly; it simply had nothing to catch here",
        "The distinction isn't useful in practice, since a real test found no disagreement",
        "The test should be discarded, since it produced no interesting finding",
        "This model always calls every available tool, regardless of the question"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The value of checking both outcome and trajectory isn't that it always finds a mismatch -- it's that you can't know whether they agree until you check both, and this run confirms the checking mechanism itself functioned as intended.",
        "Overgeneralizes from one clean run to a universal claim -- the literature's own documented cases (gaming a benchmark search, misusing a payment tool) show outcome and trajectory genuinely diverging elsewhere; one honest negative result doesn't erase that evidence.",
        "Treats a disclosed negative result as worthless -- knowing the check doesn't fire on a well-behaved run is real information, and the check still exists and would fire if behavior were different.",
        "Overstates a general claim from two data points -- nothing here establishes the model calls EVERY tool in every circumstance; only that it didn't skip verification on these two specific questions."
      ]
    },
    {
      "scenario": "A real experiment measured pass@5 = true and pass^5 = true (5 out of 5 correct) on a probability question, with no observed gap between the two metrics. A team cites tau-bench's published ~60% relative drop from pass@1 to pass^8 and concludes: 'Our result must be wrong, since tau-bench proves these metrics always diverge substantially.'",
      "question": "What's the strongest problem with that conclusion?",
      "options": [
        "Any result without a large gap contradicts tau-bench and signals a measurement error",
        "The gap's SIZE depends on task variance, not a fixed property of the two metrics",
        "pass@k and pass^k are mathematically identical, so tau-bench must mean something else",
        "5 trials is too few to compute either metric, so the result should be discarded"
      ],
      "correct": 1,
      "explanations": [
        "Treats a single cited benchmark's finding as a universal law rather than a result specific to that benchmark's own tasks -- a different, easier task genuinely can and does show a smaller gap.",
        "Correct. tau-bench's domain involves multi-turn tool use under policy constraints, which has far more room for a single trial to go wrong than one well-structured probability question -- the real lesson is that the gap's size reflects the task's actual variance, not a fixed property of the definitions themselves.",
        "Factually wrong -- pass@k (at least one success) and pass^k (all successes) are different quantities by definition, which is exactly why they can diverge when there's real variance across trials.",
        "An arbitrary, unsupported claim about sample size -- k=5 is a small but valid number of trials for computing both metrics on this specific run; the result is honestly reported as what happened at k=5."
      ]
    },
    {
      "scenario": "A team changes an agent's system prompt to spell out numbers in words (a plausible accessibility-driven requirement). Afterward, a regression suite that checks for exact digit substrings in responses reports 0/4 passing, even though a human reviewer confirms every answer is mathematically correct.",
      "question": "What's the most accurate diagnosis of what happened?",
      "options": [
        "The model's arithmetic capability regressed because of the accessibility instruction",
        "The suite was poorly designed from the start and should never have existed",
        "The grader is broken for the new format -- the model's correctness didn't regress",
        "This isn't a real regression, since the new prompt was intentional and well-motivated"
      ],
      "correct": 2,
      "explanations": [
        "Contradicts the evidence directly -- a human reviewer confirmed every answer was mathematically correct; nothing about the model's actual reasoning failed.",
        "Overreaches -- a suite that correctly caught a real, silent breakage did its job; the fix needed is updating the grader, not concluding the suite was a mistake.",
        "Correct. The suite's automated check assumed a digit format the new prompt no longer produces -- the model's capability is intact, but the check that verifies it silently stopped working, exactly what an automated suite exists to surface.",
        "Confuses 'intentional prompt change' with 'no regression occurred' -- the change being deliberate doesn't mean nothing broke; it means the break was an unintended side effect, which regression testing exists to catch regardless of intent."
      ]
    },
    {
      "scenario": "Anthropic's infrastructure noise study ran the identical model and benchmark across six container resource configurations and found a 6-percentage-point gap between the most- and least-resourced setups. A candidate summarizes this as: 'This proves benchmark leaderboards are meaningless and can never be trusted.'",
      "question": "What's the most accurate correction to that summary?",
      "options": [
        "Correct -- a 6-point infrastructure effect means no leaderboard comparison can ever be truly meaningful",
        "The finding only applies to Terminal-Bench 2.0 and has no implications for any other benchmark",
        "The SWE-bench cross-check contradicts the finding, since it found no resource effect at all",
        "It shows infrastructure is a controllable confound, not proof comparison is impossible in principle"
      ],
      "correct": 3,
      "explanations": [
        "Overstates the finding into fatalism -- the study's own conclusion is a call for controls (publishing specs, standardizing enforcement), which presumes meaningful comparison IS possible once the confound is controlled for.",
        "Understates the generalizable lesson -- the SWE-bench cross-check found a smaller but real effect on a different benchmark, showing the confound isn't unique to one benchmark, even though its size varies by task.",
        "Misstates the cross-check's actual result -- it found a smaller but still real, meaningful effect (+1.54 percentage points at 5x RAM versus 1x), not zero effect.",
        "Correct. The study's own recommendation is to treat resource configuration as a controlled experimental variable -- the problem is uncontrolled infrastructure variance, not that benchmarking is inherently meaningless; once controlled, comparisons regain their validity."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Demystifying evals for AI agents"](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2026-01-09) — graders (code/model/human), capability vs. regression evals, pass@k vs. pass^k, a CORE-Bench harness-bug fix moving a score from 42% to 95%.
- Yao et al., ["τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains"](https://arxiv.org/abs/2406.12045) (arXiv 2406.12045, 2024-06) — the pass^k reliability metric; also see [Sierra's own benchmarking post](https://sierra.ai/blog/benchmarking-ai-agents) for the retail pass@1-to-pass^8 numbers.
- Anthropic, ["Quantifying infrastructure noise in agentic coding evals"](https://www.anthropic.com/engineering/infrastructure-noise) (2026-02-05) — the 6-percentage-point resource-driven gap on Terminal-Bench 2.0, and the SWE-bench cross-check.
- ["A Holistic Agent Leaderboard: The Missing Infrastructure for AI Agent Evaluation"](https://arxiv.org/abs/2510.11977) (arXiv 2510.11977) — log-level audits catching outcome-passing, trajectory-failing agent behavior.
