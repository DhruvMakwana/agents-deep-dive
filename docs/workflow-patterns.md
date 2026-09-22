# Workflow Patterns

!!! example "Hands-on"
    Full runnable recipe: [`workflow-patterns/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/workflow-patterns) in the companion cookbook — four of the five patterns below, each with a real captured run.

??? abstract "TL;DR — quick revision"
    - **Anthropic names five workflow patterns** — chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer — and every one of them stays a workflow under the [What Is an Agent?](what-is-an-agent.md) definition: the control flow is fixed by code, even where an LLM call is involved in a decision along the way.
    - **Parallelization has two distinct shapes.** *Sectioning* splits one task into a fixed, developer-chosen set of independent subtasks. *Voting* runs the same task multiple times and aggregates. This page demos sectioning.
    - **Orchestrator-workers is the pattern most often confused with parallelization**, and the difference isn't concurrency — it's whether the set of subtasks is fixed by code ahead of time (parallelization) or decided by a planning call at runtime, per input (orchestrator-workers).
    - **A real run measured parallelization's actual value**: the same 3 review calls took 4.79 seconds run sequentially and 1.86 seconds run concurrently — a real ~2.6x speedup, not the theoretical 3x, which is the honest number once real network and queueing overhead are in it.
    - **A real run also showed orchestrator-workers doing what it's supposed to**: the same planner, given two different topics, produced two genuinely differently-*shaped* breakdowns — a comparison-structured split for one topic, a derivation-structured split for the other — with nothing in the code telling it which shape to use.

## Why these are workflows, not agents

Every pattern below can involve an LLM call at a decision point — a gate check in chaining, a category classifier in routing, an evaluator in evaluator-optimizer. None of that crosses into agent territory by the [What Is an Agent?](what-is-an-agent.md) definition, because in every case the *consequence* of that LLM call was wired in by the developer ahead of time: chaining's gate can only pass or fail into two pre-built branches, routing's classifier picks among pre-built downstream handlers, evaluator-optimizer's retry count is capped in code regardless of what the evaluator says. The model's judgment fills in a slot; it doesn't choose the slot.

That's not a limitation to work around — it's the reason these patterns are worth knowing by name. A fixed topology is easier to test, debug, and reason about than an open-ended loop, and every one of these patterns solves a real, common shape of problem without paying for full agentic autonomy.

## Chaining

Break a task into an explicit sequence of steps, where each step's output feeds the next — and, critically, insert a **gate**: a check on an intermediate output before continuing, so a bad intermediate result doesn't silently propagate into an expensive final step.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/workflow-patterns/workflow_patterns_docs.py:chaining"
```

!!! success "A real run — input and output shown in full"
    **Input** (the exact instruction sent to generate the outline): *"Write a 3-bullet outline for a short explainer on how hash tables achieve O(1) average-case lookup time, covering: what a hash function does, how collisions are handled, and why average lookup is O(1)."*

    **Output — the outline** the model actually produced:
    > • **Hash Functions and Direct Addressing**: Explain how hash functions map keys to array indices, allowing direct access to stored values in constant time, and why this is faster than searching through unsorted data.
    >
    > • **Collision Handling Strategies**: Describe how collisions (when multiple keys hash to the same index) are managed through techniques like chaining or open addressing, ensuring that collisions don't completely break the O(1) guarantee.
    >
    > • **Average-Case Analysis and Load Factor**: Discuss how O(1) average-case lookup depends on a good hash function and maintaining a low load factor, showing that when collisions are minimized and distributed evenly, the expected chain length or probe sequence remains constant.

    **The gate's verdict**, on that exact outline: `{"hash_function": true, "collisions": true, "big_o": true}` — passed, so the pipeline proceeded to expand it.

    ??? note "Output — the full ~150-word explainer generated from that outline (click to expand)"
        **Hash Tables: Achieving Constant-Time Lookups**

        **Hash Functions and Direct Addressing**
        Hash functions map keys to array indices, enabling direct access to values in O(1) time. Instead of searching through unsorted data sequentially—which requires O(n) comparisons—hash tables compute an index immediately, allowing instant retrieval. This transformation of search into computation makes lookups dramatically faster.

        **Collision Handling Strategies**
        When multiple keys hash to the same index, collisions occur. Two main strategies resolve this: *chaining* stores colliding elements in linked lists at each index, while *open addressing* finds alternative empty slots within the array. Both techniques prevent complete system failure, preserving the O(1) guarantee by limiting collision impact.

        **Average-Case Analysis and Load Factor**
        O(1) average-case performance depends on distributing keys evenly and maintaining a low load factor (ratio of elements to array size). A quality hash function and proper load factor management keep expected chain lengths or probe sequences constant, regardless of dataset size. When collisions remain minimal and uniformly distributed, lookups remain efficient.

    The gate didn't have anything to catch this run, and the expansion stayed faithful to what the outline promised — which is the point of showing it in full: you can check that claim yourself instead of taking it on faith. The gate is there for the run where the outline misses something, and this run shows the happy path working cleanly rather than a forced failure.

## Routing

Classify the input, then dispatch to a specialized downstream handler per category. You've already seen this pattern on this site: the [What Is an Agent?](what-is-an-agent.md) recipe's fixed pipeline — `classify_intent -> retrieve_faq -> generate_response` — *is* routing, and its real captured trace (a misclassification into `storage_limits`, followed by a generated reply stuck using that category's FAQ entry no matter what) is the canonical illustration of this pattern's actual failure mode: routing is only as good as the classification step, and a fixed pipeline has no way to notice a bad route on its own.

## Parallelization

Split one task into independent pieces, dispatch them concurrently, combine the results. Two distinct uses: *sectioning* (different pieces answer different sub-questions about the same input, like the demo below) and *voting* (the same question run multiple times for a more confident aggregate answer). The developer decides the fixed set of pieces ahead of time — that's what separates this from orchestrator-workers, below.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/workflow-patterns/workflow_patterns_docs.py:parallelization"
```

!!! success "A real run — input and output shown in full, so the reviewer claims below are checkable"
    **Input** — the exact paragraph all three reviewers were given (deliberately written with a few real, plantable issues):

    > Binary search works by repeatedly cutting the search space in half. You start with a sorted array, check the middle element, and if its not what your looking for, you eliminate half the remaining elements based on whether the target is bigger or smaller. This continues until you find it or theres nothing left to check, which means the algorithm needs about log base 2 of n steps for n elements, way faster then checking each one by one.

    **4.79 seconds sequential, 1.86 seconds concurrent** for the same 3 calls above — about 2.6x faster, a real, honest number rather than a clean theoretical 3x.

    **Output — each reviewer's full, unedited reply** to that exact paragraph:

    *Technical accuracy* — "**Technical Accuracy: Correct** — **Issues found:** None found. The explanation accurately describes binary search's core mechanism (repeatedly halving the search space), the requirement for a sorted array, the comparison logic, termination conditions, and the O(log n) time complexity. All technical details are factually correct."

    *Clarity for a beginner* — "**Issues found:** 1. **Missing prerequisite clarity**: The text doesn't explicitly state the array *must be sorted first* before binary search works — a beginner might assume it works on any array. 2. **Vague comparison logic**: 'whether the target is bigger or smaller' doesn't clearly explain *which half* gets eliminated based on that comparison (beginners need to know: if target is smaller, search left half; if bigger, search right half)."

    *Grammar and style* — "**Issues found:** 1. 'its not' should be 'it's not' (contraction of 'it is') 2. 'your looking' should be 'you're looking' (contraction of 'you are') 3. 'theres' should be 'there's' (contraction of 'there is') 4. 'then' should be 'than' (comparison, not time reference)"

    Read the input paragraph again against the grammar reviewer's four findings — all four are genuinely there (*its*, *your*, *theres*, *then*). The technical-accuracy reviewer finding nothing wrong is also checkable against the input: the explanation is, in fact, correct. One generic "review this" call would have had to catch all of that in one pass, or split its attention across it — three focused calls each stayed narrow and each caught what was actually in its lane.

## Orchestrator-workers

A planning call looks at the specific input and decides its **own** sub-tasks — not a fixed set chosen by the developer ahead of time, which is exactly what separates this from parallelization above. Workers answer the sub-tasks, a synthesis call combines them.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/workflow-patterns/workflow_patterns_docs.py:orchestrator_workers"
```

!!! success "A real run — the same planner, two topics, the exact sub-questions it chose for each"
    **Input** to the planner for Topic A was just the string `"arrays vs. linked lists"`. **Output** — the planner's own sub-questions, verbatim, no editing:

    1. How do arrays and linked lists store data in memory, and how does this affect access speed (e.g., random access via index)?
    2. How do insertion and deletion operations differ in cost and complexity between arrays and linked lists?
    3. How does memory allocation and overhead (fixed vs. dynamic sizing, pointer storage) differ between the two structures?
    4. In what real-world scenarios or use cases is one structure clearly preferable over the other?

    That's a **comparison-shaped** split — four dimensions to compare side by side.

    **Input** for Topic B was just `"how binary search achieves O(log n) time"`. **Output** — the same planner, same prompt template, different sub-questions:

    1. What is the core idea behind binary search that lets it eliminate half of the remaining elements at each step?
    2. How does repeatedly halving the search space mathematically translate into a logarithmic number of steps?
    3. Why does the number of times you can halve n before reaching 1 equal log base 2 of n?
    4. What real-world implication does O(log n) time have for binary search's efficiency on large datasets?

    That's a **derivation-shaped** split — four steps in a mathematical argument, not four dimensions of a comparison.

    Both happened to land on four sub-questions, which is coincidence, not the finding — read the two lists above and the difference in *kind*, not count, is the actual evidence. Nothing in the code told the planner which shape to use for which topic; the comparison prompt got compared, the derivation prompt got derived, because the planning call actually read the input and designed a decomposition to fit it.

    ??? note "Output — the full synthesized explainer for Topic B, built from those 4 worker answers (click to expand)"
        Binary search exploits **sorted data** to systematically eliminate half the search space at each step. By comparing your target to the middle element, you determine which half contains the answer — then discard the other half entirely.

        This repeated halving translates directly into logarithmic complexity. When you halve a dataset *n* repeatedly, you're answering: how many times must you divide *n* by 2 until reaching 1?

        Mathematically: *n* ÷ 2^*k* = 1

        Solving for *k*: *k* = log₂(*n*)

        Since logarithms are the inverse of exponentiation, the number of halvings equals log base 2 of the original size.

        **Real-world impact:** A 1 million-item dataset requires only ~20 comparisons with binary search versus 500,000 with linear search. This logarithmic growth means performance remains acceptable even as data scales dramatically, making binary search invaluable for massive datasets in databases, file systems, and real-time applications.

        The core insight: each step eliminates exponentially more possibilities, compressing an enormous problem into a tiny number of operations.

## Evaluator-optimizer

Generate, evaluate the output against a concrete criterion, and if it fails, retry with feedback about what to fix — capped at a hard attempt budget. The evaluator doesn't have to be an LLM call; a deterministic check works fine when the criterion is exact.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/workflow-patterns/workflow_patterns_docs.py:evaluator_optimizer"
```

!!! success "A real run — full input and output"
    **Input**: *"Write ONE sentence (under 20 words) describing a to-do list app. Do not use the words 'simple', 'easy', 'simply', or 'easily' — those are banned marketing cliches."*

    **Output, attempt 1**: *"A digital tool that organizes tasks, tracks progress, and helps you prioritize what matters most."*

    **Deterministic evaluator's verdict** on that exact sentence: `{"ok": true, "word_count": 15, "banned_found": []}` — 15 words (under the 20-word limit), no banned word present. Passed on the first attempt.

    Worth stating plainly rather than treating as a letdown: the retry-with-feedback path exists in the code and is real, but this run didn't need it, and a loop that rarely has to retry is what a correctly-scoped safety net looks like — it isn't evidence the loop was pointless to build.

## Interview angle

**Weak answer** to "when would you use orchestrator-workers instead of parallelization": *"orchestrator-workers is when you run things in parallel with more steps."* This conflates concurrency (an implementation detail — either pattern's sub-tasks can run concurrently or sequentially) with the actual distinguishing feature.

**Strong answer**: the question to ask is *who decides the sub-tasks, and when*. If the developer can enumerate the full set of sub-tasks ahead of time, independent of what a specific input looks like, that's parallelization — a fixed shape reused across every input. If the right decomposition genuinely depends on what the specific input is asking, and a planning call has to look at the input to decide it, that's orchestrator-workers. This page's own real run shows the tell: the same planner produced a comparison-shaped split for one topic and a derivation-shaped split for another, because the two topics actually called for different structures.

**Follow-up to expect**: "why not just always use orchestrator-workers, since it's strictly more flexible?" The honest trade-off: a planning call is a real cost (this recipe's planning step needed Sonnet-tier reasoning to produce a sensible breakdown, not Haiku), the resulting structure is less predictable to test against, and if your sub-tasks genuinely are fixed and known ahead of time, paying for a planning call to rediscover a fixed answer every time is waste — parallelization's fixed split is the better default whenever the shape of the problem doesn't actually change with the input.

## Build it yourself — 30 minutes

1. Pick a small task with a clear gate criterion — something with a checkable "did this actually cover what it needed to" property, not just "does this look good."
2. Build chaining first: generate, gate, expand only on pass. Deliberately try an input likely to miss the gate, and check the pipeline actually stops instead of expanding anyway.
3. Add parallelization: pick 2 or 3 fixed, independent angles to review the same piece of content from, dispatch them concurrently, and time it against dispatching them one at a time.
4. Only then try orchestrator-workers, on a task where you genuinely can't fix the sub-tasks ahead of time — and run it on two meaningfully different inputs to see whether the planner's breakdown actually changes shape, not just count.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Workflow Patterns">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A chaining pipeline generates a document outline, gates it against 3 required concepts, and only expands into a full document if the gate passes. A teammate suggests removing the gate: 'Just have the final expansion step handle any gaps itself while writing -- one fewer LLM call, and it can improvise around a missing concept instead of stopping.'",
      "question": "What's the strongest reason to keep the explicit gate?",
      "options": [
        "Removing the gate saves a call and is strictly better, since fewer steps always means lower cost",
        "The gate is unnecessary, because the same model that wrote the outline will remember what it intended when it expands, so nothing is actually at risk of being dropped",
        "The gate should be removed, but only because a 3-item checklist is too rigid -- gates only add value when checking more than 3 concepts",
        "The gate makes a chain failure legible and cheap to catch \u2014 without it, a missing concept becomes the expansion step's problem to silently improvise around"
      ],
      "correct": 3,
      "explanations": [
        "A real cost consideration, but 'fewer steps' isn't automatically 'strictly better' \u2014 it ignores the downstream cost of an unnoticed quality gap reaching the final output undetected.",
        "A plausible-sounding but ungrounded claim \u2014 there's no mechanism that guarantees the expansion step 'remembers' anything beyond what's literally present in the outline text it's given. If the outline is missing a concept, there's nothing to recall.",
        "An arbitrary, fabricated threshold with no basis \u2014 nothing about gate value scales specifically with item count past some cutoff.",
        "Correct. The gate's value isn't the check itself, it's making a specific kind of failure visible and cheap at the point it happens, instead of buried in a longer final output where it's expensive to notice and hard to attribute to a specific missing piece."
      ]
    },
    {
      "scenario": "In this page's real run, the parallel 'technical accuracy' reviewer found no issues, while the 'clarity' and 'grammar' reviewers both found real, correct problems in the same paragraph.",
      "question": "What's the correct conclusion to draw from this?",
      "options": [
        "One dimension finding nothing in a specific run isn't evidence it's worthless \u2014 different runs or inputs can trigger different subsets of real issues",
        "The technical-accuracy dimension is unnecessary and should be dropped from future runs, since it found nothing this time",
        "The reviewers must be poorly prompted or under-specified, since a genuinely useful reviewer should always find at least one real issue to justify running it at all",
        "This proves parallel dispatch produces lower-quality reviews than sequential dispatch would have"
      ],
      "correct": 0,
      "explanations": [
        "Correct. A reviewer correctly reporting nothing wrong is a legitimate, honest outcome \u2014 the same logic as this page's evaluator-optimizer passing on its first attempt. Sectioning's job is giving each dimension a focused, uncontested look, not guaranteeing every dimension produces a finding on every run.",
        "Generalizing from one clean pass to 'drop this dimension permanently' is exactly the kind of overreaction a single data point doesn't support \u2014 a different input, or the same input reviewed again, could easily trigger a real technical-accuracy finding.",
        "A tempting but backwards assumption \u2014 a reviewer's job is to report accurately, and 'accurately found nothing' is not evidence of a bad prompt, it's the correct behavior when there's genuinely nothing to flag on that dimension.",
        "Confuses concurrency (how the calls are dispatched) with content quality (what each call finds) \u2014 these are unrelated. Running the same three prompts sequentially instead of concurrently changes wall-clock time, not what any individual call returns."
      ]
    },
    {
      "scenario": "Two engineers debate whether a system is 'parallelization' or 'orchestrator-workers.' It takes a user's topic and always dispatches exactly 3 fixed, hardcoded reviewer prompts (accuracy, clarity, grammar) concurrently, then combines the results. The second engineer argues: 'That's still parallelization -- it only becomes orchestrator-workers once the SET of sub-tasks itself is decided by a model at runtime instead of being fixed by the code ahead of time.'",
      "question": "Who's right?",
      "options": [
        "The first framing is right \u2014 dispatching multiple concurrent LLM calls to handle different subtasks makes a system orchestrator-workers by definition, no matter how those specific subtasks happened to be chosen",
        "Neither -- the real distinction is whether the sub-tasks run concurrently (parallelization) or sequentially (orchestrator-workers)",
        "The second engineer is right \u2014 orchestrator-workers means a planning call decides the sub-tasks dynamically; a fixed, developer-chosen set is parallelization no matter the count",
        "The first engineer is right, but only because there are exactly 3 fixed subtasks -- orchestrator-workers specifically requires more than 3"
      ],
      "correct": 2,
      "explanations": [
        "This is the exact confusion this page's Interview angle section names directly \u2014 concurrency is an implementation detail available to both patterns, not what separates them.",
        "A genuinely tempting but wrong mechanism claim \u2014 orchestrator-workers' worker calls can also run concurrently (this recipe's do), and parallelization's calls could in principle run sequentially too. Execution order is orthogonal to this distinction.",
        "Correct. Who decides the sub-tasks, and when, is the actual distinguishing feature \u2014 a hardcoded set of 3 dispatched concurrently is parallelization regardless of scale; the moment a planning call reads the input and decides the breakdown itself, it's orchestrator-workers.",
        "An arbitrary, fabricated numeric threshold \u2014 nothing about the pattern's definition depends on a specific sub-task count."
      ]
    },
    {
      "scenario": "A team adds an evaluator-optimizer loop (generate, check length and banned words, retry up to 3 times) to a product-description generator after occasionally seeing outputs exceed the word limit. After shipping, the loop passes on the first attempt over 95% of the time.",
      "question": "What's the most defensible reaction to this data?",
      "options": [
        "Remove the loop -- if it almost never retries, it isn't doing anything useful",
        "Keep the loop as-is \u2014 a low retry rate is what a correctly-tuned safety net looks like: cheap on the common case, catching the rare real failures it exists for",
        "The high pass rate proves the evaluator's criteria are too lenient and should be made stricter to justify the loop's existence",
        "The low retry rate means the model's outputs have become reliable enough now that the deterministic checks can safely be removed entirely, leaving no checks in place at all"
      ],
      "correct": 1,
      "explanations": [
        "This is the exact 'if it rarely fires it must be useless' trap this page's own evaluator-optimizer run was written to push back on directly \u2014 a rare failure is still a real failure, and the loop's cost on the 95%+ common case is negligible.",
        "Correct. The loop was built because occasional real violations were observed; a low retry rate after shipping means it's catching those rare cases cheaply, which is success, not evidence the loop is unnecessary.",
        "Backwards reasoning \u2014 a low failure rate doesn't imply the bar is too easy; the bar was presumably set at the actual requirement (word limit, banned words), and rarely failing it is the desired outcome, not a sign to tighten further.",
        "Confuses 'usually passes' with 'will always pass' \u2014 removing a cheap, deterministic check because failures are rare (not impossible) reintroduces exactly the bug the loop was built to catch, the next time a rare case shows up."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents) (2024-12-19) — the five named workflow patterns this page is built from.
