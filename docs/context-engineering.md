# Context Engineering

!!! example "Hands-on"
    Full runnable recipe: [`context-engineering/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/context-engineering) in the companion cookbook — minimal real repros of Drew Breunig's four context failures, each with a real fix applied and a real before/after against an independently computed ground truth.

??? abstract "TL;DR — quick revision"
    - **Context engineering is curating the optimal set of tokens for each inference call**, not writing one good prompt once. Anthropic's framing: models have an "attention budget" — a real constraint from the transformer's n² pairwise token relationships — and **context rot** is what happens as that budget gets spent on tokens that don't help.
    - **Four verbs cover most of what you actually do**: write context (save it outside the window), select context (pull the right thing back in), compress context (keep only what's needed), isolate context (split it up) — LangChain's framework, and Breunig's specific fixes (RAG, tool loadout, quarantine, pruning, summarization, offloading) are each an instance of one of these four.
    - **A real repro of context poisoning worked exactly as the theory predicts**: a hallucinated fact (a fake founding year) in context produced a wrong downstream answer (17 years old instead of 12); quarantining the bad turn — not just adding a correction, replacing it — fixed it cleanly.
    - **A real repro of context distraction produced a genuine failure, but not the hypothesized one** — six turns of a consistently wrong pattern in context didn't make the model mechanically repeat that exact pattern; it produced a *different* wrong answer (the raw, undivided sum) in the same terse style the flawed history modeled. Reported honestly rather than smoothed into matching the prediction.
    - **Real repros of confusion and clash did not reproduce as failures in this run** — both honest negative results, with real, disclosed reasons why (a 12-tool test below the literature's reported ~30-tool confusion threshold; a clash repro that gave the model an explicit "supersedes" cue, making it resolvable rather than genuinely ambiguous).

## What context engineering means, and why it isn't just prompt engineering

Anthropic's own framing draws a real distinction: prompt engineering is about writing and organizing instructions for best results; context engineering is the broader practice of curating and maintaining the optimal set of tokens present during inference, across every turn of an agent's run — the system prompt, the tools, the examples, and the message history that accumulates as the agent works. The reason this needs a name at all: **context rot**. As the number of tokens in the context window grows, a model's ability to accurately recall and use information from that context degrades — not because the context window has a hard limit, but because the transformer's attention mechanism computes n² pairwise relationships between tokens, so there's a real, finite "attention budget" being spent, and every irrelevant or redundant token is budget not spent on what matters.

The practical implication is the one this page spends its real experiments on: more context is not free, and the wrong context can actively make an agent worse, not just less efficient. That's a different claim from "keep your prompts short" — it's specifically about *what kind* of tokens degrade a response, which is exactly what Breunig's four named failure modes each describe.

## Four verbs: write, select, compress, isolate

LangChain's framework for context engineering strategies gives four categories, each with a one-line definition:

- **Write** — saving information outside the context window to help the agent later. Scratchpads (state saved via tool calls mid-task) and memories (facts persisted across sessions) are both this.
- **Select** — pulling the right information *into* the context window when it's needed. Retrieval, tool selection via RAG over tool descriptions, and just-in-time loading via lightweight identifiers (a file path, a query, a resource ID — not the full content) instead of pre-loading everything are all this.
- **Compress** — retaining only the tokens actually required. Summarization (recursive or hierarchical compaction of a long trajectory) and trimming (dropping old or superseded messages by a heuristic) are both this.
- **Isolate** — splitting context up rather than keeping it all in one window. Multi-agent systems (each sub-agent gets its own isolated context, returning a condensed summary — often 1,000-2,000 tokens — to the parent) and sandboxed execution (keeping token-heavy intermediate objects like full file contents outside the model's context entirely) are both this.

Breunig's own six fixes for the four failure modes below map cleanly onto these four verbs — RAG and tool loadout are both *select*; quarantine is *isolate*; pruning and summarization are both *compress*; offloading is *write*. The four failure modes are the specific problems; write/select/compress/isolate are the general toolkit for fixing them.

## Context poisoning

Breunig's definition: context poisoning is when a hallucination makes it into the context. The danger isn't the hallucination alone — it's that once it's in the conversation history, every subsequent turn treats it as an established fact, and the error compounds instead of getting caught once and discarded.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/context-engineering/context_engineering_docs.py:poisoning"
```

!!! success "A real run — the exact poisoned and quarantined histories, both full answers shown"
    **Input**, identical question for both conditions: *"As of 2026, how many years old is Meridian Robotics?"* Ground truth (independently computed, not from the model): the company was founded in 2014, so 12 years old.

    **Poisoned condition** — the conversation history includes a fake tool result: *"[tool_result: company_lookup] Meridian Robotics was founded in 2009."* Given that, the model's real answer: *"As of 2026, Meridian Robotics would be 17 years old (2026 - 2009 = 17 years)."* Confidently wrong, because the false premise was never questioned — the model reasoned correctly *from* a poisoned fact.

    **Quarantined condition** — the same conversation, except the tool result is corrected to *"[tool_result: company_lookup, corrected after a data error was caught] Meridian Robotics was founded in 2014."* The bad turn isn't left in context alongside a correction — it's replaced, the way a real quarantine step would catch and remove it before it ever reaches a downstream question. Real answer: *"Based on the information provided, Meridian Robotics was founded in 2014. As of 2026, that would make it 12 years old (2026 - 2014 = 12)."* Correct.

    The fix worked because it addressed the actual mechanism: the model wasn't unable to do the arithmetic in the poisoned condition — the arithmetic on the wrong premise was completely correct. The failure was upstream of reasoning, in what the reasoning was given to work with.

## Context distraction

Breunig's definition: context distraction is when the context overwhelms the training — when accumulated context pulls a model toward repeating a pattern it has seen many times in the conversation instead of reasoning about the current input fresh.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/context-engineering/context_engineering_docs.py:distraction"
```

!!! success "A real run — the flawed history in full, and an honestly-reported surprise"
    **Input** — six prior turns, each following the identical (wrong) pattern of dividing by N-1 instead of N, all stated with no shown work: *"What's the average of 2, 4, 6?" → "The average is 6."* (actual average: 4) *"What's the average of 10, 20?" → "The average is 30."* (actual: 15) — and four more of the same shape, ending with a fresh question: *"What's the average of 10, 20, 30, 40?"* Correct average: 25. What mechanically continuing the demonstrated N-1 pattern would give: 33.33.

    **Distracted condition** (full flawed history included) — real answer: *"The average is 100."* Neither 25 nor 33.33 — 100 is the raw, undivided sum (10+20+30+40). A genuine failure, but not the one hypothesized going in: the flawed history didn't get mechanically copied, but it still produced a real wrong answer, delivered in the same terse, no-work-shown style all six prior "wrong" answers modeled.

    **Compressed condition** (that history dropped, only the fresh question asked) — real answer: *"The average is 25. (10 + 20 + 30 + 40) ÷ 4 = 100 ÷ 4 = 25"* — correct, and showing its work this time, unprompted.

    Reported exactly as it happened rather than adjusted to fit the prediction: the specific *mechanism* of distraction here wasn't pattern-copying the wrong arithmetic, it was something closer to style-copying (terse, unjustified answers) that happened to also drop a step. The fix — compress away the flawed history — worked regardless of which specific mechanism caused the failure, which is itself worth noting: you don't have to know exactly *how* bad context corrupts a response to know that removing it helps.

## Context confusion

Breunig's definition: context confusion is when superfluous context — information that's present but not relevant to the task — influences the response, most commonly documented as degraded tool-selection accuracy once a tool library grows large (RAG-MCP reports accuracy roughly tripling, from about 13.6% to about 43.1%, once tool selection is filtered down instead of exposing everything).

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/context-engineering/context_engineering_docs.py:confusion"
```

!!! success "A real run — an honest negative result, with the real numbers and the real reason it didn't reproduce"
    **Input**, identical for both conditions: *"How many US dollars is 200 EUR worth, using the live rate?"* Ground truth: $216.00 (200 × 1.08).

    **Clean condition** — one tool, `get_live_exchange_rate`. Real result: called the correct tool, answered *"200 EUR is worth 216 USD"* — correct.

    **Confused condition** — 12 tools: the correct tool, a plausible stale-data decoy (`get_exchange_rate_estimate`, described as "cached... may be stale," which would have returned a wrong $230 if picked), and 10 unrelated filler tools (currency symbols, crypto prices, stock quotes, shipping costs — genuinely irrelevant to this question). Real result: **still called the correct tool**, **still answered $216** — identical to the clean condition.

    This is an honest negative result, not a hidden one, and the real reason is worth stating precisely: RAG-MCP's reported confusion effect shows up past roughly 30 tools; this repro used 12, deliberately kept small for a cheap, fast run. It's a genuine, disclosed limitation of this specific repro's scale — not evidence the failure mode doesn't exist, and not something this page can claim to have ruled out.

## Context clash

Breunig's definition: context clash is when parts of the context disagree — two statements in the same conversation that contradict each other, which can happen from information arriving in shards over a conversation rather than all at once (one paper reports a large model's score dropping from 98.1 to 64.1 when the same task's information was split across multiple turns instead of given complete upfront).

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/context-engineering/context_engineering_docs.py:clash"
```

!!! success "A real run — another honest negative result, with the real reason disclosed"
    **Input**, identical question for both conditions: *"What is the refund window, in days?"* Ground truth: 14 (the current policy).

    **Clashing condition** — both facts present: *"[policy_doc v1, published Jan 2026] The refund window is 30 days."* followed by *"[policy_doc v2, published Aug 2026, supersedes v1] The refund window is 14 days."* Real answer: *"According to policy_doc v2 (published Aug 2026, which supersedes v1), the refund window is 14 days."* — correct, and it explicitly cited the supersession reasoning.

    **Pruned condition** — only the current fact present, the stale one removed. Real answer: *"According to the policy document, the refund window is 14 days."* — also correct.

    Both conditions got it right, and the honest reason is a real limitation of this specific repro, disclosed rather than glossed over: the word "supersedes" in the input is an explicit resolution cue, which makes this a resolvable disagreement, not a genuinely ambiguous one. A harder, more faithful repro of context clash would give the model two contradicting facts with *no* signal about which one is current — that's real future work this page doesn't claim to have already done.

## What this means in practice

Two of these four repros showed a clean failure-and-fix; two showed the model handling the situation fine at the scale and clarity this page tested. That split is itself the finding, not a weakness in the demo: context engineering isn't "always apply all six fixes defensively" — it's understanding which failure mode a given situation actually risks, at what scale, and applying the corresponding verb (write, select, compress, isolate) only where the risk is real. Quarantine matters the moment any tool result could plausibly be wrong. Compression matters once a transcript gets long enough to start modeling its own bad patterns back at the model. Select (tool loadout) matters once a tool library crosses the scale where confusion has actually been measured to bite — which, per RAG-MCP, is a real, specific, much higher number than 12. And a genuinely deployed clash-detection strategy needs to handle silent, unlabeled contradictions, not just the labeled kind this page's repro tested.

## Interview angle

**Weak answer** to "how do you prevent context rot in a long-running agent": *"Just summarize the conversation periodically to keep it short."* This names one real technique (compress) but treats context engineering as a single lever, and it can't explain why this page's own poisoning fix (quarantine, not summarization) was what actually worked there — summarizing a poisoned history would just compress the wrong fact more efficiently, not remove it.

**Strong answer**: the fix has to match the failure mode, not just "make context smaller." A hallucinated fact needs to be caught and removed at the source (quarantine/isolate) — summarizing around it preserves the error. A long transcript that's started modeling its own bad patterns needs the flawed history actually dropped, not condensed (compress). A tool library that's grown past the point where selection degrades needs filtering down to what's relevant for this task (select). Two contradicting facts need the stale one identified and pruned once a newer one supersedes it (compress/prune) — or, if there's no reliable "which one is current" signal, that's a genuinely harder problem this page's own repro didn't solve, worth naming as an open issue rather than glossing over.

**Follow-up to expect**: "if two of your four repros didn't show a failure, doesn't that undercut the whole premise?" No — it's the more credible result. A demo where every hypothesized failure reproduces exactly as predicted, every time, at trivial scale, would be a bigger red flag than one where two out of four genuinely didn't reproduce, with disclosed, specific reasons (scale below a documented threshold; an explicit resolution cue present). The value of running real experiments instead of just citing the taxonomy is finding out exactly where the edge is, not confirming the theory always looks bad in a toy example.

## Build it yourself — 30 minutes

1. Pick one of the four failure modes and design the smallest possible repro with an independently-computable ground truth — a fact you know is false, a pattern you know is wrong, a tool you know is the right one, two facts you know contradict.
2. Run the "broken" condition first and read the actual real output, not just whether it matched a pass/fail check — this page's distraction repro only surfaced its real finding (a different failure than hypothesized) by reading the text, not just checking it wasn't 25.
3. Apply exactly one fix (matching the verb: write, select, compress, or isolate) and run the identical question again. Compare the real before/after, not an assumed one.
4. If the failure doesn't reproduce, don't discard the repro — report why, honestly, the same way this page did for confusion and clash. A disclosed negative result at a stated scale is real information; a silently-abandoned repro is not.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Context Engineering">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro showed a poisoned fact (a fake 2009 founding year) producing a wrong downstream age calculation (17 instead of 12). The fix that worked was replacing the bad tool-result message entirely with a corrected one, not appending a correction after it.",
      "question": "Why does replacing the bad message work where appending a correction might not?",
      "options": [
        "Shorter context is always better, regardless of what the tokens actually say",
        "Both facts staying in context turns poisoning into an unresolved clash instead",
        "The arithmetic itself was the real problem, so neither fix approach would work",
        "Appending corrections is fine generally -- only dates needed replacement here"
      ],
      "correct": 1,
      "explanations": [
        "Token count isn't the mechanism here -- a short context with a wrong fact in it is still wrong; the issue is correctness of content, not length.",
        "Correct. Leaving both statements in context turns one already-understood problem (poisoning) into a second, less predictable one (clash) -- and this page's own clash repro showed resolution isn't automatic even when it happens to work; it depends on the model correctly weighting which fact is current, which isn't guaranteed.",
        "Contradicts the real run directly -- the poisoned condition's arithmetic (2026 - 2009 = 17) was completely correct; the model reasoned correctly FROM a false premise, which is precisely what makes this poisoning and not a reasoning failure.",
        "Introduces an arbitrary, unsupported distinction -- nothing about this mechanism is specific to dates; the same logic applies to a fake fact of any kind sitting in context."
      ]
    },
    {
      "scenario": "A real distraction repro hypothesized the model would repeat a wrong pattern (divide by N-1) demonstrated across six prior turns, but the real result was different: the model answered with the raw, undivided sum instead. A reader concludes: 'This means the demo failed -- distraction isn't a real phenomenon, since the model didn't reproduce the predicted pattern.'",
      "question": "What's the strongest problem with that conclusion?",
      "options": [
        "It's correct -- an unmatched predicted mechanism means the phenomenon wasn't actually shown",
        "The demo should be rerun repeatedly until the exact predicted pattern occurs",
        "The compressed condition's correct answer proves the distracted answer was secretly correct too",
        "A real wrong answer occurred with flawed history, and vanished once it was removed"
      ],
      "correct": 3,
      "explanations": [
        "Conflates the specific predicted mechanism with the general phenomenon -- a wrong answer that appears only when the flawed history is present, and disappears when it's removed, is real evidence of context-caused degradation regardless of whether the exact wrong number matches a prior guess.",
        "Not how this page's own stated frugality works, and not how the actual finding was reached -- rerunning until a hypothesis is confirmed would bias toward the prediction rather than reporting what happened.",
        "A fabricated, unsupported claim -- 100 and 25 are different numbers; nothing in the real transcripts suggests the distracted answer was secretly equivalent to the correct one.",
        "Correct. The result (100, an unexplained wrong answer under the distracted condition; 25, correct with shown work under the compressed condition) is real evidence of context-caused failure and a real working fix -- the fact that the WRONG answer wasn't the specific one predicted doesn't erase that a real failure and a real fix both occurred."
      ]
    },
    {
      "scenario": "A real confusion repro (12 tools: 1 correct, 1 stale decoy, 10 irrelevant fillers) found no measurable confusion -- the model called the right tool and got the right answer in both the clean and confused conditions. A team concludes from this: 'Tool confusion isn't a real risk for our agent, since a real experiment already disproved it.'",
      "question": "What's the strongest flaw in that conclusion?",
      "options": [
        "12 tools is well below the roughly 30-tool threshold the cited literature reports",
        "The team is misreading their own page's results -- confusion was actually found",
        "Fictional tools make any confusion repro invalid, regardless of how many tools are used",
        "The result only applies to currency-conversion tasks specifically, not any other domain"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The repro's own real number (12 tools) sits below the specific threshold the cited literature (RAG-MCP) reports degradation starting around -- so a clean result at 12 tools doesn't contradict the literature's claim about 30+, it's simply a different, smaller-scale test that wasn't positioned to detect the same effect.",
        "Misreads the real result -- the page explicitly reports both conditions answered identically and correctly; there was no confusion detected in this specific run.",
        "An overly strong, unsupported rule -- illustrative or fictional tools are the same pattern this entire cookbook uses precisely to test mechanisms cheaply and safely; the real limitation here is scale, not tool realism.",
        "Overly narrow -- nothing about the tested mechanism (many similar-sounding tool descriptions increasing selection difficulty) is inherently specific to currency conversion; the domain choice here was arbitrary and illustrative."
      ]
    },
    {
      "scenario": "A real clash repro gave the model two contradictory policy statements, one explicitly labeled as superseding the other ('policy_doc v2... supersedes v1'). The model correctly resolved the conflict and answered with the current value. A candidate cites this in an interview as proof that 'LLMs reliably resolve contradictory context correctly.'",
      "question": "What's the most accurate pushback on that claim?",
      "options": [
        "The claim is accurate -- one successful resolution is sufficient evidence of general reliability",
        "The result should be dismissed entirely -- policy documents are too narrow a domain",
        "The repro gave an explicit resolution cue; it didn't test a genuinely unlabeled, ambiguous clash",
        "The model didn't resolve anything -- it just output the numerically smaller value by chance"
      ],
      "correct": 2,
      "explanations": [
        "Overgeneralizes from a single, favorable, labeled case to a much broader and unproven claim about ambiguous cases -- one success under easy conditions doesn't establish reliability under harder ones.",
        "Too sweeping -- the domain (policy documents) isn't the limiting factor; the labeled-versus-unlabeled distinction is, and that applies across domains, not just this one.",
        "Correct. This page's own real run disclosed exactly this limitation: the word 'supersedes' is an explicit resolution cue, which makes the test one of instruction-following under a clear signal, not of resolving a genuinely unresolvable or unlabeled clash -- a meaningfully different, harder problem the repro didn't test.",
        "An unsupported, fabricated explanation with no basis in the transcript -- the model's answer explicitly cited the supersession reasoning ('which supersedes v1'), which is not consistent with an arbitrary numeric coincidence."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Drew Breunig, ["How Long Contexts Fail"](https://www.dbreunig.com/2025/06/22/how-contexts-fail-and-how-to-fix-them.html) (2025-06-22) — the four named failure modes: poisoning, distraction, confusion, clash.
- Drew Breunig, ["How to Fix Your Context"](https://www.dbreunig.com/2025/06/26/how-to-fix-your-context.html) (2025-06-26) — the six fixes: RAG, tool loadout, quarantine, pruning, summarization, offloading.
- Anthropic, ["Effective context engineering for AI agents"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (2025-09-29) — the attention budget, context rot, just-in-time retrieval, compaction, structured note-taking, sub-agent architectures.
- LangChain, ["Context Engineering for Agents"](https://www.langchain.com/blog/context-engineering-for-agents) — the write / select / compress / isolate framework.
- Tang et al., ["RAG-MCP"](https://arxiv.org/abs/2505.03275) (arXiv 2505.03275) — tool-selection accuracy roughly tripling (43.13% vs. 13.62% baseline) once tool retrieval filters a large tool library instead of exposing it all.
