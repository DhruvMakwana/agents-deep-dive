# Data and Environments

!!! example "Hands-on"
    Full runnable recipe: [`data-environments/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/data-environments) in the companion cookbook — a real, complete SWE-smith-style task-synthesis-and-verification loop, run start to finish against real code and real tests.

??? abstract "TL;DR — quick revision"
    - **Real trajectory-synthesis methods turn scarce, expensive real data into cheap, scalable synthetic data — with real, automated verification built in, not just generation.** AgentTrek converts web tutorials into agent trajectories at a real, verified cost of *"$0.55 per high-quality trajectory without human annotators,"* using *"a VLM-based evaluator [to verify] trajectory correctness."* ToolACE builds *"a comprehensive API pool of 26,507 diverse APIs"* across 390 domains, checked by a real *"dual-layer verification system combining rule-based and model-based checks."*
    - **SWE-Gym and SWE-smith attack the same real bottleneck — realistic SWE training environments — from two different angles.** SWE-Gym curates *"2,438 real-world Python task instances"* from actual GitHub issues, producing *"up to 19% absolute gains in resolve rate"* on SWE-bench. SWE-smith instead *"automatically synthesizes"* tasks by breaking tests in *"50k instances sourced from 128 GitHub repositories"* — no real historical bug report required at all.
    - **A real repro of SWE-smith's exact mechanism, run start to finish**: a real Claude call introduced a genuine, subtle boundary bug into working code; a real pytest run confirmed the task was real (4 of 5 tests passed, 1 genuinely failed); a second real Claude call, given only the real failing test output, fixed it; a real pytest re-run confirmed all 5 tests passed.
    - **"Verifiers" is the real, general term for what makes RL training at scale possible: a deterministic pass/fail signal instead of an LLM's opinion.** RLVR's own framing: *"for code, a compiler ran the output and returned pass or fail, producing binary rewards: 1 for correct, 0 for wrong."* Prime Intellect's real, open-source `verifiers` library formalizes this as an Environment — *"a dataset of task inputs, a harness for the model... and a reward function or rubric to score the model's performance."*

## Trajectory synthesis: cheap, scalable, and verified

Real agent trajectories — a full sequence of an agent's actions and observations completing a task — are expensive to collect from humans and scarce in the wild. Real, current methods synthesize them instead, with automated verification built directly into the pipeline rather than trusting the generation step alone. AgentTrek's real mechanism: harvest web tutorials, convert them into structured task specs, then *"use a visual-language model (VLM) agent to execute these instructions in real environments, while a VLM-based evaluator verifies trajectory correctness"* — real, verified cost: *"$0.55 per high-quality trajectory without human annotators."* APIGen-MT takes a two-phase approach — task blueprints *"validated through format/execution checks and an LLM committee review,"* then turned into full trajectories *"through simulated human-agent interplay"* — with 5K real trajectories openly released alongside the resulting models.

## ToolACE: scale and verification for tool-calling data specifically

ToolACE targets tool-calling data directly, building *"a comprehensive API pool of 26,507 diverse APIs"* across 390 real domains through *"a novel self-evolution synthesis process,"* with dialogs generated *"through the interplay among multiple agents, guided by a formalized thinking process."* Its real, distinguishing safeguard: a *"dual-layer verification system combining rule-based and model-based checks"* — not trusting either a deterministic rule or a model's judgment alone. At publication, ToolACE-8B topped the BFCL-v1 leaderboard at **91.41%** overall accuracy, ahead of Claude-3.5-Sonnet's 90.53%; worth noting that BFCL has since moved to a harder v3 leaderboard (adding "live" categories), where ToolACE-8B's real score is lower (59.22, rank #3) — a real reminder that a benchmark's own difficulty isn't fixed over time, and a headline number needs its leaderboard version attached.

## SWE-Gym and SWE-smith: real tasks vs. synthesized tasks, at real scale

Two real, complementary approaches to the same problem: training data for software-engineering agents needs a real, executable environment, not just text. SWE-Gym curates *"2,438 real-world Python task instances,"* each *"comprising a codebase with an executable runtime environment, unit tests, and a task specified in natural language,"* sourced from actual GitHub issues — real, and *"the first publicly available training environment combining real-world SWE tasks from GitHub issues with pre-installed dependencies and executable test verification."* Training on it produced real, measured gains: *"up to 19% absolute gains in resolve rate on the popular SWE-Bench Verified and Lite test sets,"* reaching 32.0% and 26.0% respectively with trained verifiers added — a real state-of-the-art for open-weight agents at publication.

SWE-smith takes the opposite real approach: instead of curating existing real bug reports, it *"automatically synthesizes... task instances that break existing test(s) in the codebase"* from arbitrary working repositories — no historical PR or issue required at all. The real scale gap this unlocks is stark: *"50k instances sourced from 128 GitHub repositories,"* against prior work's *"at most 1,000s of training instances from 11 or fewer GitHub repositories."* The resulting model, SWE-agent-LM-32B, reached **40.2% Pass@1** on SWE-bench Verified — state of the art among open-source models at publication. This page's own repro reproduces SWE-smith's exact mechanism directly, at a much smaller scale but with the identical underlying idea.

## Repro: synthesizing a real task, then verifying a real fix

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/data-environments/data_environments_docs.py:starting-code-and-tests"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/data-environments/data_environments_docs.py:real-verifier"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/data-environments/data_environments_docs.py:synthesis-prompt"
```

A real, working `merge_intervals` function with a real, passing 5-test suite. A real Claude call introduces exactly one subtle bug — SWE-smith's own approach, generating a task from arbitrary working code rather than requiring an existing bug report. The real verifier above is the same shape RLVR and SWE-Gym both rely on: actually run the code, don't ask an LLM whether it looks right.

!!! success "A real run, unforced — full synthesis-then-fix loop"
    **Synthesis**: Claude changed one comparison operator, `start <= last_end` to `start < last_end` — a genuinely subtle boundary bug (intervals that exactly touch, like `[1,4]` and `[4,5]`, stop merging). **Real verification**: `4 passed, 1 failed` — specifically `test_touching_intervals`, exactly the case the injected bug breaks, nothing else. A real, well-formed synthesized task: hard enough to be genuine, narrow enough to be solvable.

    **Fix**: given *only* the real failing pytest output — not the original correct code — a second Claude call correctly diagnosed the boundary condition and restored `start <= last_end`. **Real re-verification**: `5 passed` — the fix genuinely worked, confirmed by execution, not by the fixing model's own confidence.

    A real bug was caught and fixed in this recipe's own code along the way: an earlier version indexed `response.content[0].text` directly and broke with `AttributeError: 'ThinkingBlock' object has no attribute 'text'` — the first content block isn't reliably the text block. Fixed by filtering for `block.type == "text"` explicitly, then this clean run was captured.

## Verifiers: the real mechanism that makes any of this trainable at scale

Every method on this page depends on the same real, underlying requirement: a way to score a trajectory or an output *without* a human in the loop, cheaply and reliably enough to run millions of times. RLVR's own framing names the pattern directly: *"for code, a compiler ran the output and returned pass or fail, producing binary rewards: 1 for correct, 0 for wrong"* — the same real mechanism this page's own repro just ran, by hand, on one example. Prime Intellect's real, open-source `verifiers` library formalizes the idea as a reusable unit: an Environment is *"a dataset of task inputs, a harness for the model (tools, sandboxes, context management, etc.), and a reward function or rubric to score the model's performance,"* with a real, stated reason for decoupling verification logic from any specific training stack: *"Environment implementations are often tied to a specific training RL stack and can be difficult to adapt to a new trainer."*

## What this means in practice

The real throughline across every method on this page: data and environments for training agents don't come from finding more real examples, they come from designing a real, automated way to generate *and check* examples at scale. SWE-Gym curates real tasks with executable verification already attached; SWE-smith and AgentTrek synthesize tasks and verify them automatically instead of curating; ToolACE layers rule-based and model-based checks on generated dialogues; the `verifiers` library packages the whole pattern — dataset, harness, reward function — as a reusable unit. This page's own repro is the smallest possible version of the same idea: one real bug synthesized, one real test run to confirm it's genuine, one real fix attempted, one real test run to confirm it worked. Nothing here depended on an LLM's opinion about correctness — every real verdict came from actually executing code.

## Interview angle

**Weak answer** to "how would you get training data for a coding agent?": *"Scrape real GitHub issues and PRs."* SWE-smith's own real, verified numbers are a direct counterexample to treating that as the only real lever — 50k instances from 128 repos, generated by breaking tests in *arbitrary* working code, dwarfs what curating real historical bug reports alone can practically produce (*"at most 1,000s... from 11 or fewer repositories"*).

**Strong answer**: name the real, general pattern underneath every method on this page — synthesize the task, then verify it's genuine and solvable with a deterministic check, not an LLM's judgment. This page's own repro demonstrates the mechanism concretely: real pytest execution, not a model saying "yes, this looks like a real bug" or "yes, this looks fixed." Then connect it to why this matters for RL specifically: RLVR's binary pass/fail reward only scales because the verifier is cheap and automated — the same real property SWE-Gym, SWE-smith, and ToolACE's own verification layers are all built around.

**Follow-up to expect**: "what happens when you can't write a clean pass/fail verifier for a task?" A real, honest answer grounded in this page's own material: that's exactly the gap ToolACE's *dual-layer* verification (rule-based plus model-based) and AgentTrek's *VLM-based evaluator* are built to cover — when a deterministic check isn't available, a model-based verifier is the real fallback, at the real cost of being less reliable than actually running code, which is why the strongest pipelines on this page layer both rather than picking one.

## Build it yourself — 30 minutes

1. Pick a real, small function you have with a real, passing test suite (or write one — 4-5 tests is enough).
2. Have a real LLM call introduce one subtle bug into it, then actually run the test suite against the result — this page's own repro predicts a clean, informative partial failure (some tests pass, the ones testing the broken behavior fail), not a total collapse.
3. Feed only the real failing test output (not the original code) to a second LLM call and ask it to fix the bug. Re-run the real test suite to verify.
4. If you're building anything that needs to score model outputs at scale, check whether a deterministic verifier (execution, exact match) is possible before reaching for an LLM-as-judge — this page's own material argues that's the more reliable, and dramatically cheaper, default whenever the task allows it.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Data and Environments">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro had a Claude call introduce one bug into a working function, then ran the real test suite: 4 of 5 tests passed, and exactly 1 failed -- the specific test covering the broken behavior.",
      "question": "What does this specific pattern of partial failure (not 0 passed, not 5 passed) most precisely indicate about the synthesized task's quality?",
      "options": [
        "The synthesized bug is narrow and realistic, breaking only the specific behavior it targeted",
        "The test suite itself must be broken, since a genuine bug should cause every test to fail",
        "This pattern proves the bug was too subtle to be a genuine, realistic software defect",
        "The result is inconclusive, since partial test failures cannot distinguish a real bug from a flawed test suite"
      ],
      "correct": 0,
      "explanations": [
        "Correct. This is precisely the real, well-formed synthesized task the page describes: narrow enough that most of the function's behavior remains correct (4 passing tests), but genuinely broken in one specific, testable way (1 failing test) -- exactly the shape SWE-smith's approach aims to produce, verified by actual execution rather than assumed.",
        "Not supported -- a real bug in one specific code path (a boundary condition) legitimately only breaks tests that exercise that exact path; expecting every test to fail conflates 'any bug exists' with 'every code path is affected,' which isn't how real bugs work.",
        "Backwards -- a bug that's TOO subtle to matter wouldn't fail any test at all; failing exactly the test that covers the affected boundary condition is evidence the bug is realistic and consequential, not too subtle to be genuine.",
        "Overstates the ambiguity -- the real, deterministic pytest run directly attributes the failure to a specific assertion (test_touching_intervals) with a specific diff, which is concrete, traceable evidence of a real behavioral change, not an inconclusive result."
      ]
    },
    {
      "scenario": "SWE-smith's real, verified approach generates 50k task instances from 128 GitHub repositories by breaking existing tests in arbitrary working code, explicitly without requiring a real historical PR or issue -- contrasted with SWE-Gym's real, curated 2,438 instances sourced from actual GitHub issues.",
      "question": "What is the most precise, real distinction between these two approaches' core mechanisms?",
      "options": [
        "SWE-Gym generates entirely synthetic tasks with no connection to real code, while SWE-smith only uses real historical bugs",
        "SWE-smith curates real historical bug reports at greater scale than SWE-Gym's smaller, hand-picked selection",
        "The two approaches are functionally identical, differing only in which specific GitHub repositories they draw from",
        "SWE-Gym curates tasks from real historical bug reports; SWE-smith synthesizes tasks from arbitrary working code instead"
      ],
      "correct": 3,
      "explanations": [
        "Backwards on both counts -- SWE-Gym's real tasks ARE sourced from actual GitHub issues (real code, real bugs), while SWE-smith is the one that does NOT require any real historical bug report at all, synthesizing tasks instead.",
        "Backwards -- SWE-smith is explicitly the SYNTHESIS approach (breaking tests in arbitrary code, no historical report needed); it's SWE-Gym that curates from real historical GitHub issues, not SWE-smith.",
        "Not supported -- the real, core mechanisms differ fundamentally (curation from real historical issues vs. synthesis from arbitrary working code), not merely which specific repos were selected; this difference is exactly what explains the real scale gap between the two.",
        "Correct. This is precisely the real, distinguishing mechanism: SWE-Gym's 2,438 instances come from real, curated GitHub issues; SWE-smith's 50k instances are SYNTHESIZED by deliberately breaking tests in arbitrary working repositories, with no real historical bug report required -- the core reason SWE-smith reaches a real, dramatically larger scale (50k vs. 2,438)."
      ]
    },
    {
      "scenario": "RLVR's real, quoted framing states: 'for code, a compiler ran the output and returned pass or fail, producing binary rewards: 1 for correct, 0 for wrong.' Separately, ToolACE uses 'a dual-layer verification system combining rule-based and model-based checks.'",
      "question": "What real, practical reason would a pipeline choose ToolACE's dual-layer approach over a pure RLVR-style binary compiler check?",
      "options": [
        "Dual-layer verification is strictly more accurate in every case, so it should always replace binary compiler-based checks",
        "Model-based checks are needed when correctness can't be reduced to a single deterministic pass/fail signal alone",
        "Binary compiler checks are being phased out industry-wide in favor of model-based verification exclusively",
        "The dual-layer approach exists purely for redundancy, providing no capability a single-layer check couldn't already provide"
      ],
      "correct": 1,
      "explanations": [
        "Overclaims universality -- the page frames deterministic execution-based verification (RLVR's compiler check) as the MORE reliable default when available, not something dual-layer checking should always replace; each is suited to different task types.",
        "Correct. This is the precise, real reason the page gives for layering rule-based and model-based checks: some correctness properties (e.g., whether a generated tool-calling dialogue is realistic and diverse, not just syntactically valid) aren't reducible to a single deterministic pass/fail test the way compiled code execution is -- model-based judgment fills that real gap.",
        "Not supported anywhere in the real, cited material -- nothing in the page suggests compiler/execution-based verification is being phased out; RLVR's binary pass/fail approach is presented as real and current, not obsolete.",
        "Contradicted directly -- the page frames the dual-layer approach as covering DIFFERENT kinds of correctness (rule-based for what can be checked deterministically, model-based for what can't), not as redundant duplication of the same capability."
      ]
    },
    {
      "scenario": "A real repro's own recipe code initially broke with 'AttributeError: ThinkingBlock object has no attribute text' when indexing response.content[0].text directly, and was fixed by filtering for block.type == 'text' explicitly before trusting any result.",
      "question": "What does this specific bug and fix illustrate about applying this page's own 'verify, don't assume' principle to the recipe's own development process?",
      "options": [
        "The bug proves LLM API responses are fundamentally unreliable and unsuitable for any verification pipeline",
        "The fix was unnecessary, since the original indexing approach would have worked correctly in most real cases",
        "Catching this via a real run before trusting results applies the same discipline this page argues for",
        "This bug specifically demonstrates a flaw in the SWE-smith method's own task-synthesis mechanism"
      ],
      "correct": 2,
      "explanations": [
        "Overreaches -- one indexing assumption being wrong (position isn't guaranteed) isn't evidence the API itself is unreliable; the response structure was well-defined, just not in the shape the code assumed.",
        "Not accurate -- the error was a real, reproducible crash on an actual run, not a hypothetical edge case; the fix (filtering by type rather than position) was necessary to get a working result at all, not an unnecessary precaution.",
        "Correct. This is precisely the page's own throughline applied reflexively: the bug was only caught because the code's own output was actually run and checked, rather than assumed correct -- the identical discipline ('verify by execution, don't assume') the page argues every synthesized task and every RL reward signal needs.",
        "Not related -- this was a bug in the RECIPE'S OWN Python code for parsing an API response, unrelated to SWE-smith's actual task-synthesis mechanism (which is about generating bugs in target code, not about how this blog's demo code parses API responses)."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- ["AgentTrek: Agent Trajectory Synthesis via Guiding Replay with Web Tutorials"](https://arxiv.org/abs/2412.09605) — the real, verified $0.55-per-trajectory synthesis pipeline.
- ["ToolACE: Winning the Points of LLM Function Calling"](https://arxiv.org/abs/2409.00920) — the real, verified 26,507-API pool and dual-layer verification.
- ["Training Software Engineering Agents and Verifiers with SWE-Gym"](https://arxiv.org/abs/2412.21139) — the real, verified 2,438-instance curated environment.
- ["SWE-smith: Scaling Data for Software Engineering Agents"](https://arxiv.org/abs/2504.21798) — the real, verified 50k-instance synthesis pipeline this page's own repro reproduces the mechanism of.
- [Prime Intellect `verifiers`](https://github.com/PrimeIntellect-ai/verifiers) — the real, open-source Environment/Rubric framework for RL verification at scale.
- [Training Agents: Reward and Credit](training-agents.md) — GRPO, DAPO, and outcome-vs-process reward, the general RL mechanics this page's verification methods feed into.
- [RL for Search and Tool Agents](rl-search-tool-agents.md) — ToolRL's own reward-granularity findings, directly complementary to this page's verifier coverage.
