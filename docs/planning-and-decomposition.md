# Planning and Decomposition

!!! example "Hands-on"
    Full runnable recipe: [`planning-and-decomposition/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/planning-and-decomposition) in the companion cookbook — real plan-and-execute repros: a replanning trigger, a granularity tradeoff that produced a real, reproducible failure, and a minimal verifier-in-the-loop check.

??? abstract "TL;DR — quick revision"
    - **Plan-and-execute separates the "what should happen" decision from the "make it happen" work**: a planner produces a multi-step plan up front, and an executor carries it out — real, cited benefits over a single ReAct-style loop include speed (*"the larger agent doesn't need to be consulted after each action"*), cost (*"sub-tasks... can be made to smaller, domain-specific models"*), and completion quality (*"forcing the planner to explicitly 'think through' all the steps required"*).
    - **A real replanning repro was a clean, honest negative**: told either to execute its plan without second-guessing, or explicitly to revise on a broken assumption, Sonnet 5 produced the identical, correct outcome both times when a planned step's premise (a repo has a git tag) turned out false. Worth reporting plainly — the assumption that a rigid "don't second-guess" instruction would cause a real failure didn't hold up.
    - **A real granularity repro found the opposite — a genuine, reproducible failure, confirmed twice**: the same task, decomposed too coarse or well-sized, completed correctly both times (3 real tool calls, a correct changelog). Decomposed over-granular, the model spent its entire token budget writing out sub-steps and never called a single tool — a real, measurable cost of over-decomposition, not a hypothetical one.
    - **A minimal verifier-in-the-loop caught the failure using only the trace that already existed** — no second model call, no re-doing the work. Checking one concrete fact (was the final output-producing tool actually called?) correctly passed the two successful conditions and failed the over-granular one, with a real, specific reason attached.

## What plan-and-execute buys over a single loop

A ReAct-style agent interleaves reasoning and action in one continuous loop — every tool call happens after a fresh round of "what should I do next" reasoning from the same model. Plan-and-execute splits that: a planner produces the multi-step plan once, up front, and an executor works through it. LangChain's own framing of the three real advantages this buys: *"they can execute multi-step workflow faster, since the larger agent doesn't need to be consulted after each action"* (speed); *"they offer cost savings over ReAct agents. If LLM calls are used for sub-tasks, they typically can be made to smaller, domain-specific models"* (cost); and *"they can perform better overall (in terms of task completions rate and quality) by forcing the planner to explicitly 'think through' all the steps required"* (quality, from the act of planning itself).

The mechanics of revision are explicit in the same framing: *"Once execution is completed, the agent is called again with a re-planning prompt, letting it decide whether to finish with a response or whether to generate a follow-up plan (if the first plan didn't have the desired effect)."* That's the theoretical case for replanning. Whether it's actually *necessary* — whether a model without an explicit replanning step still handles a broken plan assumption reasonably — is an empirical question, not a given.

## Repro 1: a plan's assumption breaks mid-execution — does replanning matter?

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/planning-and-decomposition/planning_and_decomposition_docs.py:release-notes-tools"
```

!!! success "A real run — the same broken assumption, two different instructions, an identical real outcome"
    **Task**: *"Prepare the release notes for the 'api-gateway' repo: find what's changed since the last release and draft a changelog."* The natural plan: get the latest tag, list PRs merged since that tag, draft the changelog. The real trap: `api-gateway` has never been tagged — `get_latest_tag` returns `{"tag": None, "note": "This repo has no tags yet."}` — so the "since the last tag" half of the plan's premise is false before execution even starts.

    **Condition A** (*"execute it as planned rather than second-guessing each step"*): real trace — `get_latest_tag` → `list_all_merged_prs` → `draft_changelog`. No attempt to force the now-broken `list_merged_prs_since_tag` call. Real answer: *"This repository has no prior tags, so this changelog covers all merged PRs to date"* — correct.

    **Condition B** (*"if a step's result reveals that an assumption behind your plan was wrong... stop and revise the rest of your plan"*): identical real trace, identical correct outcome.

    Both conditions produced the same three real tool calls and the same correct changelog, covering the same three real PRs. This is a genuine, honest negative result: whatever value an explicit replanning instruction adds in general, it didn't show up here, because Sonnet 5's baseline behavior already handled the broken assumption correctly without being told to.

## Repro 2: the granularity tradeoff — a real, reproducible failure

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/planning-and-decomposition/planning_and_decomposition_docs.py:granularity-system-prompts"
```

!!! success "A real run — the identical task, three decomposition levels, confirmed on two separate full runs"
    **Too coarse** (no explicit plan structure at all — *"complete the task in whatever way you judge best"*): real trace — 3 tool calls (`get_latest_tag`, `list_all_merged_prs`, `draft_changelog`), 4,951 tokens, a correct changelog.

    **Well-sized** (four concrete, tool-mapped steps spelled out explicitly): real trace — the identical 3 tool calls, 5,441 tokens, a correct, slightly more detailed changelog.

    **Over-granular** (a plan required to break the task into many small sub-steps — separate steps for *"connecting to the repository, authenticating, querying for the tag, parsing the tag response, checking whether the tag is null..."*): real trace — **0 tool calls**. The real `answer` field is a cut-off plan document: *"1. Repository connection & authentication... 2. Determine the last release point... 3. Retrieve merged PR data... 4. Parse PR results... 4.5. Compile a clean list"* — the response hit its token budget mid-plan, before executing a single real action.

    This isn't a one-off: the same three-way split — coarse succeeds, well-sized succeeds, over-granular produces zero tool calls — reproduced identically on a second, independent full run of the recipe. The mechanism is concrete, not philosophical: an exhaustively detailed upfront plan consumes the same token budget the executor needs to actually act, and a plan detailed enough can spend all of it before execution starts.

    ```python
    --8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/planning-and-decomposition/planning_and_decomposition_docs.py:verifier"
    ```

    **The verifier, applied to all three real results**: `too_coarse` → *"passed: draft_changelog was called; a real changelog was produced."* `well_sized` → passed, same reason. `over_granular` → *"passed: false, reason: draft_changelog was never called — no real output was produced."* One concrete check, against the trace each run already produced, no extra model call — and it correctly separated the two real successes from the one real failure.

## What this means in practice

The two repros land in different places, and that's itself informative: replanning, in this run, turned out to be something Sonnet 5 already does without being told — the "broken assumption" scenario didn't need an explicit instruction to handle correctly. Granularity, by contrast, produced a real, repeatable failure with a concrete, checkable cause. The lesson isn't "replanning instructions are useless" or "always keep plans coarse" — it's that these two planning concerns have different real risk profiles for a current-generation model: one degrades gracefully by default, one has a real failure mode that specifically over-engineering the plan itself can trigger. A verifier-in-the-loop is cheap insurance for exactly the second case — it doesn't need to be a second, expensive model call second-guessing the first; checking one concrete fact about what actually happened (was the output-producing step reached at all?) is often enough to catch a real planning failure before it reaches whoever's waiting on the output.

## Interview angle

**Weak answer** to "how would you make a multi-step agent more reliable?": *"Have it write out a detailed plan before acting."* This page's own real repro is a direct counterexample: the *most* detailed plan was the one that never got executed at all — the token budget spent articulating sub-steps was budget not spent calling the tools that would have actually produced the result.

**Strong answer**: plan detail has a real cost as well as a real benefit, and the right granularity is task-dependent, not "more is always safer." This page's own repro shows both directions of the correct principle in one comparison: too coarse didn't fail here (contrary to the intuition that an executor needs explicit guidance), but over-granular did fail, concretely, by consuming the execution budget on planning text. The actual discipline is sizing a plan step to be concrete enough to verify and large enough to be worth a distinct step — and checking that with a real run, the way this page's own recipe did, rather than assuming more detail is strictly better.

**Follow-up to expect**: "if a verifier just checks whether a tool was called, wouldn't a sufficiently clever over-granular plan game that check by calling a tool pointlessly?" Possibly — this page's own verifier is deliberately minimal (one concrete condition: was the actual output-producing step reached), which is exactly why it's cheap and worth having as a floor, not a ceiling. A production system would layer a real content check on top (does the produced changelog actually mention the right PRs, the way Evaluating Agents' outcome-vs-trajectory distinction argues for) — but even the minimal version already caught a real failure this page's own repro produced, for free, using only the trace that already existed.

## Build it yourself — 30 minutes

1. Design a task with one tool whose result can plausibly invalidate the rest of a straightforward plan (this page's null-tag lookup is one pattern — a search that comes back empty, a record that doesn't exist, a status that's the opposite of expected). Run it once with a "commit to your plan" instruction and once with an explicit "revise on broken assumptions" instruction. Compare the real traces, not an assumed difference.
2. Take one real task and write three system prompts for it: no plan structure at all, a well-sized plan with as many steps as there are real tool calls, and a plan required to break every step into several smaller sub-steps. Run all three and compare real tool-call counts, real token totals, and whether each one actually finished.
3. Write one minimal verifier: a plain Python function checking one concrete, checkable fact about a completed run's trace (not re-running the task, not a second model call). Apply it to your own three granularity conditions and see whether it catches what actually went wrong.
4. If your own replanning test comes back a tie the way this page's own repro did, don't force a difference into the write-up — report it as what actually happened, and note what that implies about how much the explicit instruction was doing versus the model's own baseline behavior.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Planning and Decomposition">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro ran the identical broken-plan-assumption scenario under two system prompts: one instructing the model to execute its committed plan without second-guessing, one explicitly instructing it to revise the plan when a step reveals a wrong assumption. Both conditions produced the identical, correct real outcome.",
      "question": "What is the most accurate conclusion to draw from this specific result?",
      "options": [
        "In this run, the model's baseline behavior already handled the broken assumption",
        "The test must have been flawed, since the two conditions used different instructions",
        "It means plan-and-execute architectures never need any replanning mechanism",
        "It proves explicit replanning instructions provide no value in any agent system"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, scoped, defensible conclusion is exactly this: for this specific broken-assumption scenario and this specific model, the explicit replanning instruction didn't change the outcome, because the model's default behavior already adapted correctly.",
        "Backwards reasoning -- deliberately using two different instructions on the identical scenario is exactly the correct experimental design to test whether the instruction matters; a flawed test would be one that couldn't distinguish the conditions, not one that did and found no difference.",
        "Overstates a single honest negative result into a sweeping architectural claim -- this repro doesn't establish that no plan-and-execute system, ever, needs a replanning mechanism; it reports what happened in one specific, real test.",
        "Overgeneralizes a single scoped result into a universal claim -- this repro tested one model, one scenario, one broken assumption; it doesn't establish that replanning instructions are valueless everywhere, only that this specific instruction didn't change the outcome here."
      ]
    },
    {
      "scenario": "A real granularity repro found that a too-coarse plan (no explicit structure) and a well-sized plan (four concrete steps) both completed the task correctly with 3 real tool calls each, while an over-granular plan (many small sub-steps) resulted in zero tool calls -- the model ran out of token budget writing the plan itself.",
      "question": "What is the most precise explanation for why the over-granular condition failed?",
      "options": [
        "The over-granular plan was factually incorrect about what steps were needed",
        "Writing out the detailed plan consumed the same token budget execution needed",
        "The model refused to execute plans that were too detailed as a policy choice",
        "The tools became unavailable specifically under the over-granular system prompt"
      ],
      "correct": 1,
      "explanations": [
        "Not what the real trace shows -- the plan text itself (as far as it got before being cut off) correctly identified the right branching logic (tag exists vs. null); the failure wasn't about the plan's correctness, it was about never finishing articulating it in time to act.",
        "Correct. The real trace shows the response hit its token budget mid-plan, having spent the entire allocation on plan text -- a concrete, measurable resource-consumption failure, not a reasoning or correctness failure. More planning detail requires more tokens to state, and those tokens compete directly with the tokens needed for actual tool calls.",
        "Unsupported and not what happened -- there's no refusal in the real trace; the model was actively producing plan content right up until it ran out of budget, not declining to proceed.",
        "Contradicts the setup -- the same tools were available identically across all three conditions; only the system prompt's planning-detail instruction differed between them."
      ]
    },
    {
      "scenario": "A minimal verifier function checked one condition against each granularity condition's real execution trace -- whether the output-producing tool (draft_changelog) was actually called -- with no additional model call involved.",
      "question": "What is the most accurate description of why this verifier design choice mattered here?",
      "options": [
        "It required a second, more powerful model to judge the quality of each plan",
        "It worked by re-running each task from scratch to check for consistent results",
        "It caught a real failure using only the trace already produced by execution",
        "It could only detect failures in the over-granular condition, not the other two"
      ],
      "correct": 2,
      "explanations": [
        "Contradicts the actual implementation -- the verifier was plain Python logic checking the trace, not a model call of any kind, let alone a more powerful one judging quality.",
        "Not how it worked -- the verifier ran once against each condition's existing trace after execution completed; it did not re-execute or re-run anything.",
        "Correct. The verifier's real value here came from being cheap: it checked one concrete, already-available fact (was draft_changelog called) rather than re-doing any work or spending a new model call -- and that simple check was sufficient to correctly separate the two real successes from the one real failure.",
        "Incorrect -- the verifier was applied uniformly to all three conditions and correctly returned 'passed' for both too_coarse and well_sized, not just a failure signal exclusive to over_granular."
      ]
    },
    {
      "scenario": "LangChain's own real framing of plan-and-execute agents lists three cited benefits over ReAct-style single-loop agents: execution speed, cost savings from using smaller models for sub-tasks, and better task-completion quality from forcing explicit upfront planning.",
      "question": "Which of this page's own two real repros most directly demonstrates a genuine risk specifically tied to the THIRD cited benefit (forcing explicit upfront planning improves quality)?",
      "options": [
        "The replanning repro, since both conditions produced identical correct outcomes",
        "Neither repro relates to the explicit-upfront-planning benefit at all",
        "Both repros equally, since they used the same underlying task and tools",
        "The granularity repro, since detailed planning caused zero real execution"
      ],
      "correct": 3,
      "explanations": [
        "Backwards -- the replanning repro's identical-outcome result doesn't illustrate a risk in the upfront-planning benefit; if anything, it shows the model succeeded regardless of the replanning instruction, unrelated to plan detail level.",
        "Incorrect -- the granularity repro is directly about varying how much explicit upfront planning happens, which is exactly what the cited third benefit claims helps quality.",
        "Overstates the symmetry -- while both repros reuse the same task and tools, only the granularity repro varies the amount of explicit upfront planning; the replanning repro varies a different variable (whether to revise a plan mid-execution) entirely.",
        "Correct. The cited benefit assumes forcing the planner to 'think through' steps explicitly helps completion quality -- the granularity repro shows a real case where pushing that same idea further (forcing MORE explicit, detailed upfront planning) backfired completely, consuming the budget needed to execute at all."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- LangChain, ["Plan-and-Execute Agents"](https://www.langchain.com/blog/planning-agents) — the real, cited speed/cost/quality benefits over ReAct, and the replanning mechanism this page's repro 1 tests directly.
- [Reasoning Paradigms](reasoning-paradigms.md) — ReWOO's dependency-via-variable-substitution mechanism, a related but distinct planning approach from plan-and-execute's serial tool calling.
- [Workflow Patterns](workflow-patterns.md) — the evaluator-optimizer pattern, a related but distinct verification loop (content refinement) from this page's verifier-in-the-loop (execution-trace checking).
- [Evaluating Agents](evaluating-agents.md) — outcome vs. trajectory grading, directly relevant to designing a more thorough verifier than this page's minimal one.
