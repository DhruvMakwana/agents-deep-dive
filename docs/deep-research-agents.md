# Deep Research Agents

!!! example "Hands-on"
    Full runnable recipe: [`deep-research-agents/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/deep-research-agents) in the companion cookbook — a real repro of Anthropic's own documented subagent "division of labor" failure mode and their prescribed fix. See [Multi-Agent Systems](multi-agent-systems.md) for the token-cost and inter-agent-consistency angle on the same architecture — this page covers different, complementary ground.

??? abstract "TL;DR — quick revision"
    - **Anthropic's own real numbers on their Research system**: a multi-agent system (Opus 4 lead, Sonnet 4 subagents) *"outperformed single-agent Claude Opus 4 by 90.2%"* on their internal eval — but *"token usage by itself explains 80% of the variance"* in performance, and *"agents typically use about 4× more tokens than chat interactions, and multi-agent systems use about 15× more tokens than chats."*
    - **A real, named failure mode, and a real, direct repro of it**: Anthropic's own documented finding — subagents *"performed the exact same searches as other agents... without an effective division of labor."* A real repro with 3 real subagents given identical, unscoped instructions produced **60% redundant document retrieval** and left 2 of 6 real documents completely uncovered.
    - **Anthropic's own prescribed fix, applied literally, worked**: *"Each subagent needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries."* The same real repro, with explicit per-subagent scopes instead, cut redundant retrieval to **14.3%** and covered the entire real document corpus.
    - **STORM's real mechanism generates better structure by simulating disagreement, not by asking once**: *"discovering diverse perspectives,"* then *"simulating conversations where writers carrying different perspectives pose questions to a topic expert,"* producing real, measured gains — *"more of STORM's articles are deemed to be organized (by a 25% absolute increase) and broad in coverage (by 10%)"* versus a single-pass outline-then-write baseline.
    - **BrowseComp is deliberately built to be easy to grade and hard to pass.** Real, verified numbers: GPT-4o scored **0.6%**, OpenAI o1 scored **9.9%**, and OpenAI's own Deep Research model scored **51.5%** — on a benchmark where, of 1,255 attempted questions, human researchers gave up on **70.8%** of them within two hours.

## Anthropic's own research system: the real numbers behind the architecture

Anthropic's own engineering write-up on building their Research feature gives real, exact numbers for a claim this whole page rests on: multi-agent research works, but it's expensive, and the expense itself is most of why it works. The architecture is the same orchestrator-worker shape covered on [Multi-Agent Systems](multi-agent-systems.md): *"a lead agent coordinates the process while delegating to specialized subagents that operate in parallel."* Real, measured performance: *"multi-agent system with Claude Opus 4 as the lead agent and Claude Sonnet 4 subagents outperformed single-agent Claude Opus 4 by 90.2%"* — and real parallelization gains, separately: *"these changes cut research time by up to 90% for complex queries."*

The real, more surprising number is what actually explains that gain: *"token usage by itself explains 80% of the variance, with the number of tool calls and the model choice as the two other explanatory factors."* Most of the improvement isn't from cleverer coordination — it's from simply spending more real compute in parallel. That's the same real 3.21x-and-4x/15x territory [Multi-Agent Systems](multi-agent-systems.md) already covers in depth; this page picks up where that one leaves off, on the failure mode Anthropic documents *underneath* that headline number.

## The real failure mode: redundant work without a real division of labor

Anthropic's own documented finding, stated plainly: subagents *"performed the exact same searches as other agents... without an effective division of labor,"* and separately, agents would continue *"when they already had sufficient results, using overly verbose search queries, or selecting incorrect tools."* Spending 15x the tokens of a single chat only pays off if those tokens are actually covering *different* ground — if three subagents independently converge on the same handful of obvious searches, the real cost is 15x with barely more real coverage than one agent would have gotten.

Their own real, prescribed fix is specific, not a vague call for "better prompting": *"Each subagent needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries."* Four concrete, checkable things, not a general instruction to "coordinate well."

## Repro: measuring real redundancy, vague vs. scoped

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/deep-research-agents/deep_research_agents_docs.py:corpus-and-search"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/deep-research-agents/deep_research_agents_docs.py:vague-vs-scoped"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/deep-research-agents/deep_research_agents_docs.py:overlap-stats"
```

A fixed, fictional 6-document corpus on one broad research topic — three real subagents get either the identical, generic prompt, or Anthropic's own fix applied literally: a distinct objective and explicit boundary each.

!!! success "A real run, unforced — 3 real subagents per condition"
    **`vague`** (identical, unscoped instructions): 3 subagents made **10 total real retrievals** but covered only **4 of 6 unique documents** — every document that got touched was touched by more than one subagent. Real redundant-retrieval rate: **60%**. All three independently gravitated toward the same obvious angles (productivity, cost, culture), writing three overlapping summaries — a direct, real reproduction of Anthropic's documented failure.

    **`scoped`** (explicit objective + boundary per subagent): 3 subagents made **7 total real retrievals** and covered **all 6 of 6** documents — the full corpus, split cleanly with almost no overlap. Real redundant-retrieval rate: **14.3%**. One subagent's search reached slightly past its assigned scope, and it caught this itself in its own output: *"I excluded a document on cross-team collaboration/mentorship, as that falls under culture, outside this research scope."*

    The real, measured contrast is the finding: identical unscoped instructions left a third of the real corpus completely uncovered while still burning redundant tokens on the same ground three times; Anthropic's own fix, applied literally, covered everything with real tokens to spare.

## STORM: better structure from simulated disagreement, not a single pass

A different, real approach to the same underlying research problem — not multiple agents researching a live topic, but one system producing a well-structured long-form article. STORM's real, three-stage mechanism: first, *"discovering diverse perspectives in researching the given topic,"* then *"simulating conversations where writers carrying different perspectives pose questions to a topic expert grounded on trusted Internet sources,"* and finally *"curating the collected information to create an outline."* The real payoff of asking from multiple simulated angles rather than once: *"more of STORM's articles are deemed to be organized (by a 25% absolute increase) and broad in coverage (by 10%)"* compared to a baseline that outlines once and writes. A real, honestly documented limitation from the same evaluation: editor feedback surfaced *"source bias transfer and over-association of unrelated facts"* as new problems this kind of automated synthesis introduces.

## BrowseComp: hard to solve, deliberately easy to grade

Deep research agents need a real way to measure whether they're actually finding hard-to-find information, not just producing plausible-sounding prose. OpenAI's BrowseComp is built around a specific, real design philosophy: *"Easy to verify, but hard to solve."* Every one of its real 1,266 questions is constructed so the *answer* is checkable at a glance, even though *finding* it requires genuinely persistent search. Real, verified accuracy numbers make the difficulty concrete: GPT-4o scored **0.6%**, GPT-4o with browsing scored **1.9%**, OpenAI o1 scored **9.9%**, and OpenAI's own Deep Research model — built for exactly this kind of task — scored **51.5%**, described in the paper as *"solving around half of the problems"* where every other tested model was *"near-zero."* Even human researchers weren't close to solving it outright: of 1,255 attempted questions, humans *"gave up after two hours"* on **70.8%** of them.

Real, open-source systems are now closing that gap: Alibaba's Tongyi DeepResearch, a 30.5B-parameter model (3.3B active per token, mixture-of-experts) trained with a customized GRPO variant, reports **43.4%** on BrowseComp — real, substantial progress from an openly-released model, though still well behind the 51.5% frontier number above.

## What this means in practice

Every real number on this page points at the same underlying design tension: parallelism buys real coverage, but only if the work is actually divided, and division has to be designed, not assumed. Anthropic's own 80%-of-variance finding says spending more (tokens, tool calls, subagents) is most of what buys the 90.2% improvement — but their own division-of-labor failure mode, and this page's own real repro of it, shows that spending without real task boundaries just pays for redundant coverage of the obvious ground, not the hard-to-find information BrowseComp is specifically built to demand. STORM's real gain comes from the same underlying idea applied to structure rather than search: multiple simulated perspectives asking different questions cover more ground than one pass, for the same real reason multiple well-scoped subagents cover more ground than three overlapping ones.

## Interview angle

**Weak answer** to "how would you build a deep research agent?": *"Spin up multiple subagents to research in parallel, then combine their findings."* This page's own real repro is a direct, measured counterexample to treating that as sufficient — three subagents given the same generic instruction covered only two-thirds of a real, small corpus, at 60% redundant cost, because "in parallel" said nothing about "on different ground."

**Strong answer**: name Anthropic's own real, four-part fix specifically — an explicit objective, an output format, guidance on tools/sources, and clear task boundaries per subagent — and cite this page's own real numbers as evidence it works: real redundancy dropped from 60% to 14.3%, real coverage went from 4/6 to 6/6 documents, on the identical task and corpus. Then connect it to the real cost math: Anthropic's own 80%-of-variance finding means most of the performance gain from a multi-agent system comes from spending tokens, which makes it directly wasteful, not just imperfect, when that spend isn't divided across genuinely different ground.

**Follow-up to expect**: "how would you evaluate whether your research agent is actually finding hard information, not just sounding confident?" BrowseComp's own real design answers this precisely: build evaluation questions that are *easy to verify* (a short, checkable ground-truth answer) but *hard to solve* (genuinely requiring persistent search, not recall) — the same "easy to verify, but hard to solve" principle behind BrowseComp's own real, documented question-construction process, which specifically screened out anything solvable by non-browsing frontier models before it was even included.

## Build it yourself — 30 minutes

1. Pick a real, moderately broad research question with at least 3 natural sub-angles, and build a small, fixed document corpus (5-10 fictional or real documents) covering all of them, with a simple search tool over it.
2. Run 3 real subagents against it with the identical, generic instruction (no scope, no boundaries) and measure real retrieval overlap — this page's own repro predicts heavy redundancy and incomplete real coverage.
3. Re-run with Anthropic's own fix applied literally: give each subagent an explicit objective, an output format, and an explicit boundary naming what it should NOT cover. Compare real coverage and real overlap against the first run.
4. If your own repro shows the same real pattern this page's does — full coverage, minimal overlap — that's Anthropic's documented fix working on your own task, not just a claim to trust.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Deep Research Agents">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "Anthropic's own engineering write-up states that for their multi-agent research system, 'token usage by itself explains 80% of the variance' in performance, alongside a separately reported 90.2% improvement over a single agent on their internal eval.",
      "question": "What is the most precise interpretation of what the 80%-of-variance finding implies about that 90.2% improvement?",
      "options": [
        "The 90.2% improvement is invalid, since it can be explained away entirely by a confounding variable",
        "Multi-agent coordination logic contributed nothing at all to the measured performance difference",
        "The two findings are unrelated and describe completely separate evaluations with no shared explanation",
        "Most of the measured improvement is attributable to spending more compute, not smarter coordination"
      ],
      "correct": 3,
      "explanations": [
        "Overstates it -- 'explains 80% of the variance' is a real, substantial statistical relationship, not a claim the entire effect is spurious or invalid; the improvement itself is a real, separately reported result on their internal eval.",
        "Too strong -- 80% of variance leaves real room for other factors (Anthropic names tool-call count and model choice as the other two), so coordination/design isn't reduced to zero contribution, just shown to be secondary to raw spend.",
        "Not accurate -- both findings come from the same real research system and the same real evaluation context; the variance-explanation finding is precisely what helps explain WHY the 90.2% figure exists, not an unrelated fact.",
        "Correct. This is the precise, real reading: token usage being the dominant explanatory factor means most of the measured gain tracks with how much real compute was spent, not primarily with cleverer division of labor or coordination logic -- which directly motivates why redundant, unscoped spending (this page's own repro) is a real, costly problem, not just an aesthetic one."
      ]
    },
    {
      "scenario": "A real repro gave 3 subagents access to an identical, generic research instruction with no assigned scope, and separately gave 3 different subagents explicit, distinct objectives and boundaries covering different parts of the same topic.",
      "question": "What was the most precise real, measured difference between the two conditions?",
      "options": [
        "The scoped condition made fewer total retrievals while covering more of the corpus, with far less overlap",
        "The vague condition covered more of the real document corpus overall, just with some inefficiency",
        "Both conditions covered the identical set of real documents, differing only in how the summaries were worded",
        "The scoped condition failed to retrieve any documents at all due to its narrower per-subagent scope"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, measured numbers: scoped made 7 total retrievals covering all 6 of 6 documents (14.3% redundant); vague made 10 total retrievals but covered only 4 of 6 documents (60% redundant) -- fewer total calls, MORE real coverage, far less waste.",
        "Backwards from the real result -- vague covered only 4 of 6 real documents, LESS than scoped's full 6 of 6 coverage; the vague condition wasn't just inefficient, it left real ground completely unresearched.",
        "Not what happened -- the real document sets covered by each condition were meaningfully different (4/6 vs. 6/6), not identical; this was a real difference in WHAT was researched, not just in phrasing.",
        "Contradicted directly by the real numbers -- the scoped condition made 7 real retrievals and achieved full corpus coverage; narrower scope per subagent did not mean fewer retrievals overall or failed retrieval, since scope specifically directed each search rather than blocking it."
      ]
    },
    {
      "scenario": "STORM's real, three-stage mechanism involves discovering diverse perspectives, simulating multi-perspective conversations with a topic expert, and then curating an outline -- reporting a 25% absolute increase in articles judged well-organized and a 10% increase in breadth of coverage versus a baseline.",
      "question": "What does this real, measured gain most precisely suggest about WHY simulating multiple perspectives improves article structure?",
      "options": [
        "The improvement is due entirely to using a larger or more capable underlying language model in the STORM condition",
        "Asking the same question from several different simulated angles surfaces different real information than asking once",
        "The gain is purely stylistic, affecting how organized the writing sounds without changing what information is included",
        "Simulating perspectives works by having the model fact-check its own previous answers repeatedly before writing"
      ],
      "correct": 1,
      "explanations": [
        "Not indicated -- the comparison described is STORM's own multi-perspective PROCESS against a single-pass baseline, not a model-capability comparison; nothing in the real, reported numbers attributes the gain to model size or capability differences.",
        "Correct. STORM's own real mechanism is specifically about DIFFERENT perspectives posing DIFFERENT questions to a topic expert -- the real gain in both organization AND breadth of coverage is consistent with each simulated perspective surfacing real information a single, undifferentiated pass would be less likely to ask about at all.",
        "Contradicted directly -- the real, reported gain includes a 10% increase in BREADTH OF COVERAGE, not just organizational quality; breadth of coverage is a substantive, information-level outcome, not a purely stylistic one.",
        "Not the described mechanism -- STORM's process is about generating NEW questions from different simulated perspectives directed at a topic expert, not about the model repeatedly verifying or fact-checking its own prior answers."
      ]
    },
    {
      "scenario": "BrowseComp is explicitly designed around the principle 'easy to verify, but hard to solve,' with real, verified accuracy numbers: GPT-4o scored 0.6%, OpenAI o1 scored 9.9%, and OpenAI's own Deep Research model scored 51.5%.",
      "question": "What does the specific design principle 'easy to verify, but hard to solve' most precisely explain about how BrowseComp's questions were constructed?",
      "options": [
        "Questions were selected so that grading requires the same amount of research effort as answering them originally did",
        "Questions were deliberately made ambiguous so that multiple different answers could be scored as equally correct",
        "Questions have a short, checkable ground-truth answer, but actually reaching it requires genuinely persistent search",
        "Questions were designed to be unsolvable by any current model, including the ones the benchmark was built to measure"
      ],
      "correct": 2,
      "explanations": [
        "Backwards from the actual design goal -- 'easy to verify' specifically means grading is CHEAP and fast (a short, checkable answer), explicitly NOT requiring the same effort as the original search; that asymmetry between solving and verifying is the entire point of the principle.",
        "Contradicts the 'easy to verify' half directly -- ambiguous, multiple-valid-answer questions would make verification HARDER, not easier; the design principle requires a single, checkable ground truth, not an ambiguous one.",
        "Correct. This is the precise, real meaning behind the quoted principle: the ANSWER is short and verifiable at a glance, while actually FINDING that answer demands real, persistent search across hard-to-find, entangled information -- exactly the asymmetry the real accuracy numbers (near-zero for most models, 51.5% for the best) are designed to expose.",
        "Contradicted by the real numbers themselves -- OpenAI's Deep Research model scored 51.5%, meaning the benchmark is hard but not unsolvable; the design goal was calibrated difficulty (screening out what contemporary models COULD already solve), not literal unsolvability by every model including future ones."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic Engineering, ["How we built our multi-agent research system"](https://www.anthropic.com/engineering/multi-agent-research-system) — the real, verified division-of-labor failure mode and prescribed fix this page's repro is built directly around.
- Shao et al., ["Assisting in Writing Wikipedia-like Articles From Scratch with Large Language Models"](https://arxiv.org/abs/2402.14207) (STORM, NAACL 2024) — the real, verified perspective-guided question-asking mechanism and its measured gains.
- Wei et al., ["BrowseComp: A Simple Yet Challenging Benchmark for Browsing Agents"](https://arxiv.org/abs/2504.12516) (OpenAI) — the real, verified "easy to verify, hard to solve" design and headline accuracy numbers.
- [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) — Tongyi DeepResearch's real, open-source BrowseComp result.
- [Multi-Agent Systems](multi-agent-systems.md) — the token-cost multiplier and inter-agent-consistency angle on the same orchestrator-worker architecture this page's repro extends.
- [Guardrails and Human-in-the-Loop](guardrails-human-in-the-loop.md) — the structural-versus-behavioral distinction, applied here to explicit subagent task boundaries rather than approval gates.
