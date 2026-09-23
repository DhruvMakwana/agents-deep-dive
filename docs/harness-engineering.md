# Harness Engineering

!!! example "Hands-on"
    Full runnable recipe: [`harness-engineering/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/harness-engineering) in the companion cookbook — a real multi-session continuation via a shared progress file, and a self-verification repro against a deliberately ambiguous tool result.

??? abstract "TL;DR — quick revision"
    - **A harness is everything around the model that makes a long-running agent actually work** — Anthropic's own framing: *"the system prompt, set of tools, and overall agent harness"* together, not the model alone. Their own real finding on why this matters: *"even a frontier coding model like Opus 4.5... will fall short... if it's only given a high-level prompt"* — the failure modes were one-shotting too much at once (context exhaustion mid-task) and prematurely declaring work complete.
    - **The documented fix is a specific artifact set, not a vague "give it more context" instruction**: an initializer session creates *"an `init.sh` script, a claude-progress.txt file that keeps a log of what agents have done, and an initial git commit"* — three concrete things a later, otherwise-blank session reads before doing anything.
    - **A real repro of this exact pattern worked end to end, and caught a real bug along the way.** A budget-limited first session made all its real lookups but was cut off before recording anything; a genuinely fresh second session — no memory of the first — read the shared progress file, re-derived the lost work from scratch, and correctly finished the task, flagging one deliberately ambiguous result as needing follow-up, unprompted.
    - **"Compaction isn't sufficient" on its own, per Anthropic's own real finding** — and a real bug in this page's own recipe demonstrated exactly the adjacent risk: treating "the response stopped" as "the agent finished" without checking *why* it stopped silently corrupted a state-tracking loop, mishandling a genuine token-budget truncation as if the agent had genuinely completed its turn.
    - **A real self-verification test — instructed discipline vs. none — was a clean, honest negative.** Anthropic's cited practice: *"Self-verify all features. Only mark features as 'passing' after careful testing."* Tested directly against a deliberately unhelpful real tool result, both the plain and the explicitly-instructed condition correctly declined to mark it "passing" — Sonnet 5's baseline judgment was already sufficient here.

## Why "just give it a bigger context window" doesn't solve long-running agents

The obvious-seeming fix for an agent that needs to work longer than one context window fits is compaction — summarize the old conversation, keep going. Anthropic's own real finding, from building a genuinely long-running coding agent (a claude.ai clone, over 200 real features), is that this isn't enough on its own: *"even a frontier coding model like Opus 4.5 running on the Claude Agent SDK in a loop across multiple context windows will fall short of building a production-quality web app if it's only given a high-level prompt."* Two concrete failure patterns showed up: the agent tried to do too much in one continuous push, running out of context mid-implementation, and — separately — later sessions would prematurely declare the work done. Compaction alone doesn't fix either: it keeps the conversation going, but *"compaction doesn't always pass perfectly clear instructions to the next agent"* — the summary itself can lose exactly the specificity a fresh session needs to avoid repeating the same two failures.

The real fix Anthropic documents is structural, not just "compact better": a dedicated initializer session sets up three concrete artifacts before any real work starts — *"an `init.sh` script, a claude-progress.txt file that keeps a log of what agents have done, and an initial git commit that shows what files were added."* Every subsequent session reads the progress file first, works from a known state, and updates it before it might run out of room — the harness, not the model's raw context window, is what actually survives across sessions.

## Repro: a genuine multi-session continuation, with a real bug caught along the way

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/harness-engineering/harness_engineering_docs.py:progress-file"
```

!!! success "A real run — a checklist filled across two genuinely separate sessions, only sharing a progress file"
    **Task**: fill in a real 6-item policy-comparison checklist (3 fictional products × 2 criteria) using a real `lookup_policy` tool, recording each finding via `update_progress_file` — a minimal, real version of the documented feature-checklist pattern.

    **Session 1**, deliberately turn-limited to force a real reset: made all 6 real `lookup_policy` calls, then ran out of its turn budget before recording a single finding. `remaining: 6` — nothing persisted.

    **Session 2**, a genuinely fresh conversation with zero memory of session 1's tool calls: read the current progress file (still empty), **re-did all 6 lookups from scratch** — because only the progress file, not the raw conversation, survives a real reset — then recorded all 6 real findings, correctly flagging the deliberately ambiguous Vertex Sync data-retention result (*"Data retention varies by plan tier; contact support for specifics"* — not an actual answer) as `needs_follow_up` rather than `passing`, unprompted. `remaining: 0` — the checklist finished correctly.

    **A real bug surfaced and got fixed before this result was trusted.** The first version of the session loop treated any non-`tool_use` stop reason as "the agent is done." Session 1's real transcript included the model's own text — *"Now recording all six findings"* — immediately followed by zero actual `update_progress_file` calls: the response had been cut off by `max_tokens` mid-generation, not genuinely concluded, and the loop was silently treating a truncation as a clean finish. Fixed by explicitly checking `stop_reason == "max_tokens"` as a distinct case from real completion. This is itself a real, on-topic harness-engineering lesson, not an incidental implementation detail: a harness that can't tell "the agent decided it's done" apart from "the agent got cut off mid-sentence" will silently corrupt whatever state it's tracking across sessions — precisely the class of failure a progress file is meant to protect against, undermined at the one place nobody was checking.

## Repro: self-verification, tested against a result designed to tempt a false "passing"

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/harness-engineering/harness_engineering_docs.py:self-verify-system-prompt"
```

Anthropic's own cited discipline for avoiding the "prematurely declares work complete" failure mode: *"Self-verify all features. Only mark features as 'passing' after careful testing."* The natural question: does stating this explicitly change real behavior, or is a capable model's baseline judgment already good enough?

!!! success "A real run — the identical ambiguous result, with and without an explicit self-verify instruction"
    Both conditions ran the full 6-item checklist against the identical real tool results, including the same deliberately unhelpful Vertex Sync response. **Without** an explicit self-verify instruction: *"needs_follow_up — Policy states data retention varies by plan tier and directs users to contact support for specifics — no concrete retention period was returned, so this is not a usable, comparable answer."* **With** the explicit instruction: *"needs_follow_up — Policy response was vague... Does not state an actual retention period or rule — needs follow-up to get a concrete answer per plan tier."*

    Both real runs reached the identical, correct classification, with real, independently-reasoned explanations each time. This is a clean, honest negative: the explicit self-verification instruction didn't change the outcome here, because Sonnet 5's default judgment about an ambiguous result was already sufficient. Worth reporting exactly as it happened rather than adjusted to fit the assumption that stating a verification rule out loud would matter.

## What this means in practice

The two repros point at the same underlying idea from different angles: a harness's real job is protecting the parts of a long-running task that a model's own judgment can't be relied on to protect by default, and figuring out which parts those actually are requires testing, not assuming. This page's own bug is the sharpest illustration — a harness component (the session loop) failed not because the model behaved badly, but because the *harness itself* conflated two genuinely different real conditions (truncation vs. completion) that only a careful implementation distinguishes. Meanwhile, the self-verification instruction — the part of the documented pattern explicitly aimed at model *judgment* — turned out not to be load-bearing in this specific test, because the model's default judgment already handled it. The progress-file continuation mechanism, by contrast, is load-bearing by construction: there's no way for a genuinely fresh session to know what a prior session did except by reading what was actually persisted, regardless of how good the model's judgment is. Harness engineering is deciding which of these two categories a given risk falls into — a structural gap only the harness can close, or a judgment call the model already makes well — and this page's own repros show both categories are real, and neither should be assumed without checking.

## Interview angle

**Weak answer** to "how would you build an agent that can work on a task for hours across multiple sessions?": *"Use a model with a big context window and summarize when it gets full."* This treats the problem as purely a context-size problem, and doesn't explain what survives a real reset (a fresh session has no access to a *summary* of the old conversation unless something outside the model explicitly wrote one down) or how a fresh session avoids repeating the same mistakes the last one made.

**Strong answer**: long-running agent reliability depends on a small set of persistent artifacts that survive independent of any single conversation's context — this page's own repro demonstrates the mechanism directly: a progress file that a genuinely fresh session reads first, works from, and updates before it might run out of room, the same pattern Anthropic's own documented harness uses (`init.sh`, `claude-progress.txt`, a git commit). The harder, more interesting engineering problem isn't the model's reasoning — it's building the surrounding loop robustly enough that it doesn't silently corrupt the state it's supposed to protect, which is exactly the class of bug this page's own recipe caught: a session loop that couldn't distinguish "genuinely done" from "cut off mid-response."

**Follow-up to expect**: "if the self-verification instruction didn't change the outcome in your test, why does Anthropic bother documenting it as a real practice?" Because "didn't matter in one specific, low-stakes test" isn't the same as "never matters" — this page's own test used a single, moderately capable model (Sonnet 5) on a fairly legible ambiguity (a policy answer that explicitly says "contact support" instead of giving a number). A harder, more subtly wrong result, a less capable model, or a task where getting it wrong is more consequential could plausibly show the instruction mattering — the honest, tested conclusion from this page's own repro is scoped to exactly what was tested, not a blanket claim that self-verification instructions are unnecessary everywhere.

## Build it yourself — 30 minutes

1. Design a real multi-step task and a shared, persistent artifact (a plain dict or file standing in for a progress file) that survives across two completely independent conversations — no shared message history, only the artifact. Run session 1 with a deliberately tight turn or token limit to force a real, unplanned stopping point.
2. Run session 2 as a genuinely fresh conversation, giving it only the current state of the shared artifact. Check whether it correctly identifies what's left to do and finishes the task, the way this page's own repro's session 2 re-derived and completed the checklist session 1 never got to record.
3. Before trusting any result from a loop like this, check what happens when a response gets cut off mid-generation rather than genuinely concluding. This page's own bug — conflating `max_tokens` truncation with real completion — is exactly the kind of thing worth checking explicitly rather than assuming your stop-reason handling is already correct.
4. Inject one deliberately unhelpful, ambiguous real result into your task (the way this page's Vertex Sync policy does) and compare a plain instruction against an explicit self-verification instruction. Don't assume the explicit instruction will matter — this page's own test found it didn't, for this specific model and this specific ambiguity, and reported that plainly.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Harness Engineering">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro's session loop initially treated any non-tool_use stop reason as 'the agent is done.' A real session hit its token budget mid-generation (stop_reason == 'max_tokens') right after the model's own text said 'Now recording all six findings' -- but zero actual recording tool calls followed, because the response had been cut off, not concluded.",
      "question": "What is the most precise description of the actual failure this bug caused?",
      "options": [
        "The model refused to complete the task once its budget ran low",
        "A real truncation was silently mishandled as if it were genuine completion",
        "The lookup_policy tool returned incorrect data for one of the six items",
        "The progress file's underlying data structure had a bug in how it stored items"
      ],
      "correct": 1,
      "explanations": [
        "Not what happened -- the model was actively trying to proceed (it stated an intent to record findings); there's no refusal in the real transcript, only an incomplete response due to running out of budget.",
        "Correct. The real bug was in the harness's OWN interpretation logic: it didn't check stop_reason carefully enough to distinguish a genuine, deliberate stop (stop_reason == 'tool_use' absent because the agent chose to conclude) from a forced cutoff (stop_reason == 'max_tokens'), and treated both identically as 'finished.'",
        "Unrelated to the actual bug -- lookup_policy's fictional data was correct throughout; the bug was entirely in the session loop's control-flow logic for interpreting API responses, not in the tool's returned data.",
        "Not where the bug lived -- ProgressFile's own update/render/remaining logic was dry-tested separately and worked correctly; the bug was specifically in _run_session's handling of the API response's stop_reason."
      ]
    },
    {
      "scenario": "A real repro ran session 1 with a tight turn limit, causing it to make all 6 real tool lookups but get cut off before recording any findings. Session 2 started as a genuinely fresh conversation -- with no access to session 1's actual conversation history -- and had to re-do all 6 lookups from scratch before it could record any findings.",
      "question": "What does session 2 needing to re-do the lookups most directly illustrate about what a real context reset actually discards?",
      "options": [
        "It illustrates that only the persisted progress artifact survives a reset, not history",
        "It illustrates a bug -- a well-designed harness should never need to repeat any work",
        "It shows the lookup_policy tool has a bug causing inconsistent results across sessions",
        "It proves that progress files are an ineffective mechanism for multi-session continuation"
      ],
      "correct": 0,
      "explanations": [
        "Correct. This is exactly the real mechanism the demo makes concrete: a progress file is the ONLY thing that survives between sessions in this design -- the raw tool-call history and conversation from session 1 is genuinely gone, so anything not written to the shared artifact has to be redone.",
        "Overstates the claim -- some repeated work is a real, inherent cost of this pattern (the raw conversation genuinely doesn't survive), not necessarily evidence of a flawed design; the alternative (persisting the entire conversation) has its own real costs the progress-file pattern is specifically designed to avoid.",
        "Contradicts the setup -- lookup_policy returns the identical fictional data every time it's called; session 2's repeated lookups returned the same real results as session 1's, just to a different, fresh conversation.",
        "Backwards -- the progress-file mechanism worked correctly here: the checklist ended fully and correctly filled (remaining: 0) specifically because session 2 could read what session 1 had NOT yet recorded and act accordingly; ineffectiveness would look like session 2 not knowing what remained, which didn't happen."
      ]
    },
    {
      "scenario": "A real repro tested a plain instruction against an explicit self-verification instruction ('only mark features as passing after careful testing'), both applied to the identical deliberately ambiguous tool result. Both conditions independently and correctly classified the result as needing follow-up rather than passing.",
      "question": "What is the most defensible conclusion to draw from this specific comparison?",
      "options": [
        "The test is invalid since a real experiment should find a difference between conditions",
        "Self-verification instructions are proven unnecessary for any model or task",
        "In this specific test, the explicit instruction didn't change the real outcome",
        "Anthropic's own documented practice must be incorrect based on this result"
      ],
      "correct": 2,
      "explanations": [
        "Backwards reasoning -- a real experiment can legitimately produce a null result; assuming a valid test must always find a difference would bias reporting toward confirming expectations rather than reporting what actually happened.",
        "Overgeneralizes a single, scoped, honest negative result into a sweeping universal claim -- this test used one model, one task, one specific ambiguity; it doesn't establish anything about harder ambiguities, less capable models, or higher-stakes tasks.",
        "Correct. The precise, defensible, scoped claim is exactly this: for this specific model, this specific ambiguous result, and this specific pair of instructions, the explicit self-verification instruction didn't change the real outcome -- a genuine, honestly-reported finding, not evidence about self-verification in general.",
        "A non sequitur -- one narrow test not finding a difference doesn't invalidate a documented practice drawn from a much larger, different real deployment (a 200+ feature production coding agent); the two are different scales and different specific claims."
      ]
    },
    {
      "scenario": "Anthropic's own real finding states that even a frontier model running in a loop across multiple context windows will fall short of a complex task if only given a high-level prompt, with two named failure patterns: attempting too much at once (context exhaustion) and prematurely declaring work complete.",
      "question": "Which specific artifact from the documented three-part initializer pattern (init.sh, claude-progress.txt, initial git commit) most directly targets the SECOND failure pattern (premature declarations of completion)?",
      "options": [
        "init.sh, since it sets up the environment before any session begins",
        "The initial git commit, since it establishes a restorable baseline state",
        "None of the three artifacts relate to premature completion specifically",
        "claude-progress.txt, since it gives a session concrete status to verify"
      ],
      "correct": 3,
      "explanations": [
        "Not the most direct connection -- init.sh addresses environment setup and reproducibility, not the specific failure of a session wrongly believing work is finished.",
        "Addresses a different, related but distinct risk -- the git commit's documented role is enabling recovery ('use git to revert bad code changes and recover working states'), which is more directly about undoing bad changes than about preventing premature 'done' declarations.",
        "Incorrect -- the progress file's real, documented role (a checklist with explicit passing/not-passing status per item) is specifically what gives a session concrete grounds to check before claiming completion, directly countering vague, ungrounded 'I'm done' judgments.",
        "Correct. A concrete, itemized progress file -- especially one following the real documented feature-checklist pattern with an explicit passing/not-passing status per item -- is what lets a session check specific, verifiable status rather than relying on its own possibly-premature sense that the task is complete."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Effective harnesses for long-running agents"](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — the initializer session pattern, why compaction alone isn't sufficient, the git-based recovery mechanism, and the self-verification discipline this page's second repro tests directly.
- [Memory Architectures](memory-architectures.md) — Claude's real memory tool and the semantic/episodic taxonomy, a closely related but distinct persistence mechanism from this page's progress-file pattern.
- [Planning and Decomposition](planning-and-decomposition.md) — the verifier-in-the-loop pattern, directly complementary to this page's self-verification repro.
- [Durable Execution](durable-execution.md) — event logs, replay, and idempotency; a lower-level, protocol-focused counterpart to this page's application-level progress-file pattern.
