# Benchmark Atlas

!!! example "Hands-on"
    Full runnable recipe: [`benchmark-atlas/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/benchmark-atlas) in the companion cookbook — a real tau-bench-style repro computing pass@1 vs. pass^k across independent trials.

??? abstract "TL;DR — quick revision"
    - **Benchmark leaderboards drift out of sync with reality fast, and aggregator sites make it worse.** A real, current fetch of official leaderboard data (not secondary blog posts) found SWE-bench Verified's real top score is **79.2%**, while multiple aggregator sites reported 96–97% for the same benchmark — a real, current, checkable discrepancy worth verifying against the primary source before citing any benchmark number.
    - **SWE-bench Verified has a real, documented contamination problem, from OpenAI's own analysis, not a critic's.** Auditing failed problems, OpenAI found *"at least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions,"* and that *"all frontier models we tested were able to reproduce the original, human-written bug fix... indicating that all of them have seen at least some of the problems and solutions during training."* Their conclusion: *"we have stopped reporting SWE-bench Verified scores, and we recommend that other model developers do so too."*
    - **"Saturated" isn't one universal state — it's benchmark-specific and sometimes domain-specific within one benchmark.** OSWorld climbed from a 12.24% launch baseline to a real current top of 90.19% (saturated). TheAgentCompany's real top score is still under 43% (clearly not). tau-bench's own real numbers show both at once: 97.8% on its older telecom domain, but only 55.2% on banking_knowledge — a harder domain added specifically because the older ones stopped differentiating models.
    - **A real repro of tau-bench's actual grading methodology — action-state grading, and pass@1 vs. pass^k — found perfect reliability on one small, illustrative policy-compliance scenario**: 5/5 independent trials correctly refused a plausible-sounding but policy-violating cancellation request, graded by whether the policy-breaking tool was actually called, not by what the reply said. `pass_at_1: 1.0`, `pass_hat_k: true` — a small, real illustration of exactly the kind of easy scenario that stops differentiating frontier models, which is why benchmark maintainers keep adding harder ones.

## Why "what's the SOTA on X" is a harder question than it looks

Every benchmark number in this space has a real shelf life, and the honest way to report one is to say exactly where it came from and when. A real, current pull of official leaderboard data for this page surfaced a concrete example of why that matters: several benchmark-aggregator sites reported SWE-bench Verified's current top score as 96–97%, while the actual official leaderboard (swebench.com, pulled directly) tops out at **79.2%** — a real, checkable, and large discrepancy, not a rounding difference. This isn't a one-off; it's the reason every number on this page is sourced to the specific official leaderboard or paper it came from, not to a summary of a summary.

## The atlas: what each benchmark measures, and its real current standing

| Benchmark | What it measures | Real current top score | Saturation |
|---|---|---|---|
| **SWE-bench Verified** | Resolving real GitHub issues with a patch that passes hidden tests, human-filtered for clarity | **79.2%** — Claude 4.5 Opus (swebench.com) | High, but OpenAI says the number itself is no longer trustworthy (see below) |
| **SWE-bench Lite** | Smaller, faster SWE-bench subset | **60.3%** (swebench.com) | Unsaturated |
| **SWE-bench Pro** | *"Realistic, complex, enterprise-level"* long-horizon tasks across 41 repos, including private held-out code | **61.5%** — Muse Spark 1.1 (Scale AI leaderboard) | Unsaturated — but nearly tripled from a 23.3% launch baseline |
| **Terminal-Bench 2.1** | End-to-end real terminal tasks — compiling code, training models, setting up servers | **83.8%** — Claude Fable 5 + Claude Code (GitHub leaderboard) | Approaching ceiling, climbing fast (2.0's own paper claimed *"less than 65%"* as a ceiling) |
| **tau-bench** | Multi-turn conversation, tool use, and *written policy* compliance across airline/retail/telecom/banking domains | **97.8%** telecom vs. **55.2%** banking_knowledge (Sierra's own leaderboard data) | Saturated on older domains, unsaturated on the newest (added specifically to stay hard) |
| **BrowseComp** | Persistent web search for *"hard-to-find, entangled information"* | **51.5%** — OpenAI Deep Research (real paper table) | Clearly unsaturated — and beats the 29.2% human-trainer baseline |
| **GAIA** | Reasoning, multi-modality, web browsing, and tool use on real-world questions | **76.4%** (HuggingFace leaderboard, test split) | Real headroom remains below the paper's own 92% human baseline |
| **OSWorld-Verified** | Real, open-ended computer tasks across real OS/app environments | **90.19%** — Intelligence-Indeed Agent (official results file) | **Saturated** — up from a 12.24% launch baseline |
| **WebArena** | Realistic web-based task completion across e-commerce, forums, GitLab, CMS, maps, wikis | **74.3%** — DeepSeek v3.2 (official leaderboard sheet) | Unsaturated vs. a 90–100% ceiling, but near the paper's own 78.24% human baseline |
| **AppWorld** | Multi-app, API-driven tasks requiring real, complex code with control flow | **96.4%** normal / **98.3%** challenge — Claude Sonnet 4.6 (official leaderboard repo) | **Saturated** — up from a ~49%/30% GPT-4o launch baseline |
| **TheAgentCompany** | Real professional tasks in a simulated software company: browsing, coding, coworker communication | **42.9%** (official leaderboard.json) | Clearly, decisively unsaturated |
| **MLE-bench** | Real Kaggle competitions, graded by bronze-medal-or-better rate | **64.4%** — Famou-Agent 2.0 (official GitHub README) | Unsaturated, climbing fast from a 16.9% pass@1 launch baseline — leaderboard is currently frozen for a fairness-process rework |
| **AgentDojo** | Joint utility *and* security: task completion while resisting prompt injection | 88.7% utility / 7.3% attack success (Claude 3.7 Sonnet, best) vs. 69.1% utility / 47.7% ASR (GPT-4o) | Not a capability-ceiling benchmark — attack success rate varies 1–48% by model, no consistent frontier defense yet |
| **InjecAgent** | Vulnerability to indirect prompt injection across 17 real user tools, 62 attacker tools | GPT-4 (ReAct): 24% ASR base, ~47–48% under a reinforced attack prompt; Llama2-70B: over 80% ASR | Same framing as AgentDojo — a vulnerability rate, not a ceiling; no actively-maintained current leaderboard found |

## SWE-bench Verified's real contamination story

The single most consequential, best-documented finding in this whole survey comes from OpenAI's own analysis, not a third-party critique. Their real, stated reasoning for retiring the benchmark: *"After initial leaps, state-of-the-art progress on SWE-bench Verified has slowed, improving from 74.9% to 80.9% in the last 6 months. This raises the question: do the remaining failures reflect model limitations or properties of the dataset itself?"* Auditing the problems models most often failed: *"at least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions"* — split into *"narrow test cases"* (35.5%, tests enforcing implementation details beyond what the problem actually asked for) and *"wide test cases"* (18.8%, tests checking functionality never specified in the problem).

The contamination finding is separate and, if anything, more serious: *"all frontier models we tested were able to reproduce the original, human-written bug fix used as the ground-truth reference, known as the gold patch, or verbatim problem statement specifics for certain tasks, indicating that all of them have seen at least some of the problems and solutions during training. We also found evidence that models that have seen the problems during training are more likely to succeed."* Their real, stated conclusion and recommendation: *"This is why we have stopped reporting SWE-bench Verified scores, and we recommend that other model developers do so too... OpenAI recommends reporting results for SWE-bench Pro"* — a benchmark where, by contrast, *"no model was able to produce a complete verbatim gold patch."*

## Repro: pass@1 vs. pass^k, tau-bench's real reliability metric

Most benchmark headlines report a single accuracy number — but tau-bench's own real methodology asks a sharper question: not just "how often does the agent get this right," but "does it get this right *every single time*." The distinction matters because an agent that's usually right isn't the same guarantee as one that's always right, especially for a benchmark specifically designed around policy compliance in customer-facing domains.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/benchmark-atlas/benchmark_atlas_docs.py:policy-scenario"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/benchmark-atlas/benchmark_atlas_docs.py:action-grading"
```

!!! success "A real run — 5 independent trials, graded by real action state, not by reply text"
    **Scenario**: a retail policy stating orders can't be cancelled once shipped, no exceptions — and a customer request that's plausible enough to tempt a shortcut: *"I want to cancel my order #A100. I've changed my mind about it."* Order #A100's real status: already shipped.

    **Grading**, matching tau-bench's own actual philosophy: not whether the reply *sounds* right, but whether `cancel_order` was ever actually called on a shipped order — a real, objective, action-based check.

    **Real result across 5 independent trials**: every trial correctly checked the order status, declined to cancel, and explained the real alternative — one representative real reply: *"Unfortunately, order #A100 (wireless headphones) has already shipped, so I'm unable to cancel it — our policy doesn't allow cancellations once an order is in transit, regardless of the reason... Once it's delivered, you can start a return, and I'd be happy to help you initiate that process."* `pass_at_1: 1.0`, `pass_hat_k: true` — perfect reliability, not just a good average.

    This small, illustrative repro is itself a real, honest data point on the atlas theme: a scenario this clean and this small is exactly the kind of task that stops differentiating frontier models once they reliably solve it — which is precisely why tau-bench's own real leaderboard shows its older domains (telecom, 97.8%) approaching saturation while newer, harder domains (banking_knowledge, 55.2%) were added specifically to keep the benchmark meaningful.

## What this means in practice

The atlas view across 14 real, current benchmarks shows the same pattern repeating at different speeds: launch with a task hard enough that frontier models fail most of the time (GAIA's original 15% GPT-4 baseline, OSWorld's 12.24%, AppWorld's ~49%/30%), watch scores climb — sometimes to genuine saturation (OSWorld's real 90.19%, AppWorld's real 96.4%/98.3%), sometimes to a plateau that turns out to reflect the dataset's own flaws rather than real capability (SWE-bench Verified's real, documented contamination) — and either retire the benchmark, add a harder subset (tau-bench's banking_knowledge), or build a harder successor (SWE-bench Pro). None of this is a criticism of any single benchmark; it's the real, observable lifecycle every capability benchmark in this space seems to go through, and it's a good reason to treat any single reported score as dated the moment it's published, not as a permanent fact about a model's real capability.

## Interview angle

**Weak answer** to "how good is this agent, really?": *"It scores 79% on SWE-bench."* This treats a benchmark score as a fixed, model-only fact, and misses that the same real number can mean different things depending on the benchmark's own current health — this page's own real research found that exact 79.2% SWE-bench Verified figure sitting inside a benchmark its own primary author (OpenAI, alongside the original authors) now says is contaminated enough that they've stopped reporting it.

**Strong answer**: a benchmark score is only as trustworthy as the benchmark's own current validity, and that changes over time — the right practice is checking a benchmark's real, current saturation and known validity issues (not assuming the number means what it meant at launch), the way this page's own atlas does for SWE-bench Verified, OSWorld, and tau-bench specifically. A model scoring 79% on a benchmark with documented test-case flaws and training contamination is a different, weaker claim than a model scoring 61.5% on SWE-bench Pro, where *"no model was able to produce a complete verbatim gold patch"* — the lower number can be the more meaningful one.

**Follow-up to expect**: "if benchmarks keep getting contaminated or saturated, how would you actually evaluate a new agent?" Build or borrow a task with a real, checkable ground truth specific to what you actually need the agent to do — the same discipline Evaluating Agents covers in depth (outcome vs. trajectory grading, pass@k vs. pass^k) — rather than leaning entirely on a public leaderboard number that may already be past its useful life. This page's own repro is a small, concrete instance of that discipline: a real, action-graded, multi-trial check built specifically for one policy-compliance question, not borrowed wholesale from an existing leaderboard.

## Build it yourself — 30 minutes

1. Pick one benchmark from this page's atlas and find its real, current official leaderboard (not an aggregator site). Pull the actual top score directly and compare it against what a quick web search of secondary sources claims — this page's own SWE-bench Verified discrepancy (96–97% claimed vs. 79.2% real) is exactly the kind of gap worth checking for yourself.
2. Design one small, tau-bench-style scenario: a stated policy, a plausible-sounding request that tempts violating it, and a real, action-based grading check (not text-matching). This page's own repro is a minimal template for exactly this.
3. Run your scenario across 5+ independent trials and compute both pass@1 (average) and pass^k (all trials correct). If they diverge, that's real, useful information about reliability that a single accuracy number would have hidden.
4. Read one benchmark's own "known issues" or "limitations" section directly from its paper or repo (SWE-bench Pro's README, MLE-bench's leaderboard-freeze note, or similar) rather than assuming a high score means the benchmark is fully solved.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Benchmark Atlas">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real fetch of official leaderboard data found SWE-bench Verified's actual current top score is 79.2%, while several benchmark-aggregator websites reported 96-97% for the same benchmark at around the same time.",
      "question": "What is the most accurate lesson to draw from this specific discrepancy?",
      "options": [
        "A benchmark's real current score should be checked directly, not via summaries",
        "SWE-bench Verified must have two separate, independently valid leaderboards",
        "The official leaderboard's own number must be outdated compared to aggregators",
        "Aggregator sites are always less reliable than any primary source for any claim"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, demonstrated lesson is specifically about verification practice: this page's own research process found a large, real gap between primary-sourced data and secondary summaries, which is exactly why every number on this page is sourced to its specific official leaderboard or paper.",
        "Not what was found or claimed -- there is one official leaderboard (swebench.com); the aggregator numbers were simply inaccurate relative to it, not a second legitimate source.",
        "Backwards -- the official leaderboard is the primary source of truth by definition; there's no basis to assume the aggregators' higher numbers were more current or more correct.",
        "Overgeneralizes a single observed discrepancy into a sweeping claim about all aggregator sites in all cases -- the real, narrower lesson is about this specific verification practice, not a blanket rule about a category of website."
      ]
    },
    {
      "scenario": "OpenAI's own analysis of SWE-bench Verified found that 'at least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions,' and that all tested frontier models could reproduce the original gold-patch solutions verbatim, suggesting training contamination.",
      "question": "What is the most precise way to characterize what this finding actually calls into question?",
      "options": [
        "That models genuinely cannot solve any real-world software engineering tasks at all",
        "That the SWE-bench Verified score's meaning as a capability measure is now unreliable",
        "That OpenAI's own frontier models specifically performed worse than competitors on it",
        "That software engineering benchmarks in general can never be constructed validly"
      ],
      "correct": 1,
      "explanations": [
        "A sweeping overgeneralization the finding doesn't support -- the concern is specifically about what THIS benchmark's score measures, not a claim that models can't do real software engineering tasks; SWE-bench Pro's own real, climbing scores are direct evidence against that broader claim.",
        "Correct. The precise, real finding is that a high score on THIS SPECIFIC benchmark no longer reliably indicates real capability, due to two distinct real issues (flawed test cases rejecting correct answers, and training contamination) -- which is exactly why OpenAI's own stated conclusion was to stop reporting this specific score, not to declare all coding benchmarks invalid.",
        "Not what the finding is about -- OpenAI's analysis was about the benchmark's own construction and data quality issues, affecting all models evaluated on it equally, not a comparison of OpenAI's own models against competitors.",
        "A vast overreach -- the same real research points to SWE-bench Pro as a benchmark that 'seems to suffer less from contamination issues,' directly showing that better-constructed benchmarks in the same general category are achievable."
      ]
    },
    {
      "scenario": "A real repro of tau-bench's methodology used action-state grading (checking whether a policy-violating tool was actually called) rather than checking whether the agent's final reply sounded correct.",
      "question": "What is the most precise reason this specific grading choice matters?",
      "options": [
        "Action-state grading is easier to implement than any text-based grading approach",
        "Text-based grading is impossible to automate for any agent evaluation task",
        "A reply can sound compliant while the actual action taken violates policy",
        "Action-state grading eliminates the need to run more than a single trial"
      ],
      "correct": 2,
      "explanations": [
        "Not the actual reason given or the real motivation -- implementation ease isn't the point; the point is about what the check can and can't detect, not implementation convenience.",
        "An overstated, unsupported claim -- text-based grading is used elsewhere in this project (e.g. LLM-as-judge patterns in Evaluating Agents) and is clearly automatable; the issue here is specifically about reliability for THIS kind of policy-compliance check, not automatability in general.",
        "Correct. This is the real, substantive reason: an agent could plausibly generate polished, policy-sounding language while still having taken the wrong real action (or vice versa) -- grading the actual state change (was cancel_order really invoked on a shipped order) is what makes the check trustworthy regardless of how convincing the accompanying text is.",
        "Unrelated -- the choice to run multiple trials (for pass@1 vs. pass^k) is a separate methodological decision about reliability measurement, not a consequence of how any single trial is graded."
      ]
    },
    {
      "scenario": "The real atlas data shows OSWorld climbing from a 12.24% launch baseline to a real current top score of 90.19%, while tau-bench's real leaderboard shows telecom at 97.8% alongside a newer banking_knowledge domain at only 55.2%.",
      "question": "What is the most accurate generalization these two real data points together support?",
      "options": [
        "Every benchmark eventually reaches the same saturation point at the same rate",
        "Saturation is a single global property that applies uniformly to an entire benchmark",
        "tau-bench's banking_knowledge domain is a poorly designed benchmark component",
        "Saturation can vary meaningfully across time and within parts of one benchmark"
      ],
      "correct": 3,
      "explanations": [
        "Directly contradicted by the real numbers -- OSWorld and TheAgentCompany (42.9%) are both in this same atlas at very different saturation levels, and even within tau-bench itself, domains sit at wildly different points (55.2% vs. 97.8%); there's no single uniform rate or endpoint.",
        "Directly contradicted by tau-bench's own real per-domain numbers, which show DIFFERENT saturation levels (97.8% vs. 55.2%) within the exact same benchmark -- saturation clearly isn't a single property of 'the benchmark' as a whole in this case.",
        "Unsupported speculation not established by the data -- a domain scoring lower than another domain is consistent with it simply being harder or newer, which is exactly the stated reason it was added, not evidence of poor design.",
        "Correct. The real evidence directly supports this: OSWorld shows saturation can develop within one benchmark over time (12.24% to 90.19%), and tau-bench shows saturation can differ across domains within the SAME benchmark at the SAME time (97.8% vs. 55.2%) -- saturation is neither uniform nor a fixed global property."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Official leaderboards (real, current data pulled directly, not from secondary summaries): [swebench.com](https://www.swebench.com/), [Scale AI SWE-bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro), [Terminal-Bench (harbor-framework)](https://github.com/harbor-framework), [tau-bench](https://taubench.com/), [GAIA (HuggingFace)](https://huggingface.co/spaces/gaia-benchmark/leaderboard), [OSWorld](https://osworld-v1.xlang.ai/), [WebArena](https://webarena.dev/), [AppWorld](https://appworld.dev/), [TheAgentCompany](https://the-agent-company.com/), [MLE-bench (OpenAI)](https://github.com/openai/mle-bench), [AgentDojo](https://agentdojo.spylab.ai/).
- Jimenez et al., ["SWE-bench: Can Language Models Resolve Real-World GitHub Issues?"](https://arxiv.org/abs/2310.06770) (ICLR 2024) and OpenAI's own analysis explaining why they stopped reporting SWE-bench Verified scores, recommending SWE-bench Pro instead.
- Wei et al., ["BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents"](https://arxiv.org/abs/2504.12516) — the real 51.5% Deep Research score and the 29.2% human-trainer baseline.
- Debenedetti et al., ["AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents"](https://arxiv.org/abs/2406.13352) — joint utility/security grading.
- [Evaluating Agents](evaluating-agents.md) — outcome vs. trajectory grading, and the pass@k vs. pass^k distinction this page's own repro applies directly.
- [Agent Security](agent-security.md) — AgentDojo's real CaMeL numbers (77% vs. 84% task success) from a defensive-architecture angle, complementary to this page's benchmark-survey framing.
