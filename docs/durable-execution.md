# Durable Execution

!!! example "Hands-on"
    Full runnable recipe: [`durable-execution/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/durable-execution) in the companion cookbook — a minimal event-log agent, run as two genuinely separate process invocations, with a real deliberate crash mid-tool-call and a real idempotent resume.

??? abstract "TL;DR — quick revision"
    - **Durable execution means a crash resumes the conversation instead of restarting it.** Temporal's own framing: "When a Worker crashes, the Temporal Service hands the work to another Worker, which replays the Event History and resumes at the line where execution stopped, with local variables and progress intact." For an agent specifically: "The loop is a Workflow, each model call and tool call is an Activity, and a crash resumes the conversation instead of restarting it."
    - **Replay and checkpointing are two different mechanisms that solve the same problem differently.** Temporal-style engines replay a recorded event history — re-running workflow code but skipping already-completed steps using their recorded results. LangGraph-style checkpointing persists application state directly as a snapshot. Both need a persistent backend to survive a real process crash: LangGraph's own docs note `MemorySaver` and `InMemorySaver` "store checkpoints in RAM. When the process restarts, all checkpoints are lost."
    - **A real, from-scratch repro of the exact failure mode worked cleanly.** A crash was deliberately triggered right after a tool's side effect committed but before that fact was durably logged — the single most dangerous instant for a naive retry. Resuming in a genuinely separate process replayed the completed steps with zero new model calls, then completed the task with no duplicated side effect.
    - **Idempotency is the second line of defense, not a redundant one.** Replaying the event log tells you what's *known* to have completed — it can't tell you about the gap between "the side effect happened" and "the log says it happened." An idempotent tool, checking its own persisted state before acting, is what actually prevents a double charge or a duplicate booking in that gap.
    - **This is a real, documented interview topic**, not a hypothetical: "How do you make sure agents do not double-execute side-effectful operations like charging a card or booking a ticket twice?" and "Suppose your booking agent sometimes reserves the same hotel twice — walk through how you'd debug and fix this" are both real, sourced interview questions.

## Why agents need this more than typical request/response services

A normal web request either succeeds or fails within a few hundred milliseconds, and a crash mid-request is usually safe to just retry from scratch. An agent loop is different in a way that makes crash recovery genuinely harder: it can run for minutes, hours, or (per real production harnesses) days, accumulating real side effects — a charge, a reservation, a sent email — at unpredictable points along the way. Restarting "from scratch" after a crash doesn't mean replaying an idempotent GET request; it means re-running a sequence that may have already charged a card once, and might charge it again.

## Replay vs. checkpointing

Two different real mechanisms answer "how does the system know what already happened," and it's worth being precise about which is which, since the terms get used loosely. Temporal's model persists an **event history** — every significant step (starting a workflow, an activity being scheduled, an activity completing) is an event, and recovery means re-executing the workflow's own code from the top while *replaying* already-recorded events instead of re-running the corresponding activities: "replays the Event History and resumes at the line where execution stopped, with local variables and progress intact." LangGraph's model instead **checkpoints application state** directly — a snapshot of the graph's state at each step, persisted so a later run can "resume exactly where it left off." The practical catch, stated plainly in LangGraph's own docs: the default in-memory checkpointers ("`MemorySaver` and `InMemorySaver`") "store checkpoints in RAM. When the process restarts, all checkpoints are lost" — durability, in either model, requires a real persistent backend, not just the presence of a checkpointing API.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/durable-execution/durable_agent_docs.py:event_log"
```

## Idempotent tools: the second line of defense

An event log answers "what does the orchestrator know completed" — but there's a real gap it can't close on its own: the instant between a tool's side effect actually committing and that fact being durably recorded. A crash in that exact window means the orchestrator will, on resume, believe the step never happened and ask for it again. The fix isn't a smarter log — it's a tool that can tell the difference between "do this" and "you already did this," checked against its own persisted state, independent of whatever the orchestrator believes.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/durable-execution/durable_agent_docs.py:idempotent_tools"
```

## A real crash, a real resume, no duplicate side effect

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/durable-execution/durable_agent_docs.py:agent_loop"
```

!!! success "A real run — two genuinely separate process invocations, not two calls in one script"
    **Task**, identical across both invocations: *"Process order ORD-8842: charge the card $49.99, reserve item SKU-772, then send an order confirmation. Do exactly one tool call per turn, in that order, then report done once all three steps are complete."*

    **Invocation 1** — real model decisions, real tool executions, then a deliberate crash:
    ```
    [step 1] real model call decided: charge_card({'order_id': 'ORD-8842', 'amount': 49.99})
    [step 1] tool executed: {'status': 'charged', 'charge_id': 'ch_ORD-8842', 'amount': 49.99}
    [step 2] real model call decided: reserve_item({'order_id': 'ORD-8842', 'sku': 'SKU-772'})
    [step 2] tool executed: {'status': 'reserved', 'reservation_id': 'rsv_ORD-8842', 'sku': 'SKU-772'}
    [step 3] real model call decided: send_confirmation({'order_id': 'ORD-8842'})
    [step 3] tool executed: {'status': 'sent', 'confirmation_id': 'conf_ORD-8842'}
    [crash] simulating a worker crash right after send_confirmation's side effect committed, before this step's completion is durably logged.
    ```
    At the moment of the crash: the confirmation genuinely was sent — its own ledger has the entry — but the event log holds only 2 completed steps, not 3. The process is dead. Nothing about step 3 exists anywhere the orchestrator can see.

    **Invocation 2** — a completely separate process, started fresh, reading only what's on disk:
    ```
    [resume] replayed 2 completed step(s) -- no new model calls for these.
    [step 3] real model call decided: send_confirmation({'order_id': 'ORD-8842'})
    [step 3] tool executed: {'status': 'already sent (idempotent replay)', 'confirmation_id': 'conf_ORD-8842'}
    [done] Done! All three steps have been completed successfully for order ORD-8842:
    1. ✓ Card charged $49.99 (charge_id: ch_ORD-8842)
    2. ✓ Item SKU-772 reserved (reservation_id: rsv_ORD-8842)
    3. ✓ Order confirmation sent (confirmation_id: conf_ORD-8842)
    ```
    Steps 1 and 2 cost **zero** new model calls — reconstructed purely from the durable log, exactly matching Temporal's own framing of what replay buys you. Step 3 needed a real new model call, since it was never durably marked complete, and the model correctly (from its own point of view) asked to send the confirmation again — but the tool's own ledger check caught it: one `confirmation_id`, sent exactly once, across a crash and a full resume. Total real API calls for the entire crash-and-resume story: 4.

## Interview angle

**Weak answer** to "how do you make sure an agent doesn't charge a customer's card twice if it crashes mid-task": *"Just log everything and replay the log on restart."* This names half the mechanism and skips the actual failure mode this page's own real run demonstrates: a crash can happen in the gap between a side effect committing and that fact being logged, and no amount of replaying the log helps with a step the log never recorded as attempted.

**Strong answer**: durability needs two separate mechanisms working together, not one. A durable, replayable event log (or persisted checkpoint) answers "what does the system already know completed," so a resumed run doesn't waste calls or make new decisions for steps that are genuinely finished — this page's own resumed run needed zero new model calls for its first two steps. Idempotent tools, keyed on something stable like an order ID or an explicit idempotency key, answer the harder question the log can't: what happens when the orchestrator asks for a step again that already silently succeeded. The real production discipline is designing every side-effecting tool call to be safe to receive twice, not just designing the orchestrator to avoid asking twice.

**Follow-up to expect**: "doesn't the crash simulation in your demo feel artificial — how often does a crash really land in that exact instant?" It doesn't need to land there often to matter: a long-running agent making many tool calls over hours has many such instants, and the cost of getting it wrong (a real double charge, a real duplicate booking) is asymmetric — one occurrence in production is a real incident, however rare the timing. That asymmetry is exactly why production durable-execution engines (Temporal, DBOS, Restate) exist as dedicated infrastructure rather than something teams build ad hoc per agent, and why the real interview questions on this topic ("how do you make sure agents do not double-execute side-effectful operations") are framed around debugging exactly this class of rare-but-costly failure.

## Build it yourself — 30 minutes

1. Pick a multi-step task with at least one side-effecting action per step, and design each step's tool to check its own persisted state (a local file, a dict, a real database in production) before acting — keyed by something stable across retries, not a fresh UUID generated each attempt.
2. Build the durable event log: append a record after every completed step, and write a replay function that reconstructs conversation state from the log before making any new model call.
3. Deliberately crash the process (a hard `os._exit`, not a caught exception) right after a tool executes but before you log that it did — the specific instant this page's own repro targets.
4. Run the exact same command again as a genuinely new process invocation, not a retry loop inside the same script. Verify two things separately: that already-completed steps needed zero new model calls, and that the step that was mid-flight during the crash didn't produce a second real side effect.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Durable Execution">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro deliberately crashed a process right after a `send_confirmation` tool's side effect committed, but before that fact was written to the durable event log. On resume in a fresh process, the model decided to call `send_confirmation` again, and the tool returned 'already sent (idempotent replay)' instead of sending a second time.",
      "question": "What does this specific sequence demonstrate about the event log's own limits?",
      "options": [
        "The log can only tell you what it knows completed, not what just now happened",
        "The event log should have caught this itself -- its absence here is a design bug",
        "The event log correctly recorded the send, so this scenario couldn't have occurred",
        "This proves event logs are unnecessary as long as every tool is idempotent"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The event log answers 'what do we already know completed' -- it has no way to represent 'a side effect just happened but the write recording it hasn't happened yet,' which is exactly the window a real crash can land in.",
        "Misdiagnoses the mechanism -- a durable log fundamentally cannot record an event before that event's own write completes; there is always a real window between a side effect and its durable record, regardless of implementation quality.",
        "Contradicts the setup directly -- the whole point of the crash's timing was that the log had NOT yet recorded step 3's completion when the process died.",
        "Overreaches -- the log still provided real value in this same run, letting the first two completed steps replay with zero new model calls; idempotency and the log solve different parts of the problem, not substitutes for one another."
      ]
    },
    {
      "scenario": "A team built an agent with a durable event log and checkpointing, but used LangGraph's default `InMemorySaver` for their checkpoint backend. They believe their agent can survive a process crash and resume without data loss.",
      "question": "What's the most accurate assessment of this setup?",
      "options": [
        "Correct -- any checkpointer, by definition, guarantees durability across restarts",
        "In-memory checkpoints are lost the moment a restart happens, progress included",
        "Fine, as long as the agent also logs to stdout, since terminal output persists",
        "Checkpointing is irrelevant here; only idempotent tools matter for crash recovery"
      ],
      "correct": 1,
      "explanations": [
        "Overstates what a checkpointer guarantees -- the mechanism (persisting state as checkpoints) is separate from WHERE it's persisted; an in-memory store doesn't survive the process that holds it dying.",
        "Correct. LangGraph's own documentation states plainly that in-memory checkpointers 'store checkpoints in RAM. When the process restarts, all checkpoints are lost' -- durability requires a persistent backend, not just a checkpointing API being present in the code.",
        "Introduces an irrelevant and incorrect claim -- stdout output isn't a structured, replayable state store, and nothing about this scenario suggests it would substitute for a real persistence layer.",
        "Understates checkpointing's real role -- it's what lets completed steps be replayed without new model calls, a genuine part of the solution alongside (not replaced by) idempotent tools."
      ]
    },
    {
      "scenario": "A candidate explains Temporal's durability model as: 'When a worker crashes, Temporal just restarts the workflow function from the very beginning, using the exact same input, and relies on the activities being fast enough that this is cheap.'",
      "question": "What's the most accurate correction to this explanation?",
      "options": [
        "Correct as stated -- restarting from the beginning with the same input is exactly the model",
        "Temporal has no concept of recovery at all -- workflows simply cannot survive a crash",
        "Temporal replays the recorded event history, resuming progress rather than restarting",
        "This only holds for single-activity workflows; multi-activity ones work differently"
      ],
      "correct": 2,
      "explanations": [
        "Restates the misconception rather than correcting it -- 'restart from the beginning' is precisely what Temporal's design avoids for already-completed work.",
        "Directly contradicts Temporal's stated purpose -- durable, crash-resistant workflow execution is the core feature being described, not something absent from the system.",
        "Correct. Temporal's own documentation describes the recovering worker replaying the Event History and resuming 'at the line where execution stopped, with local variables and progress intact' -- completed activities are not re-run, they're reconstructed from their recorded results.",
        "Introduces an arbitrary, unsupported distinction -- the replay mechanism described applies to workflows generally, not conditioned on activity count."
      ]
    },
    {
      "scenario": "A team argues: 'Since our tools are all idempotent, we don't need a durable event log at all -- idempotency alone solves crash recovery.'",
      "question": "What's the strongest flaw in that argument?",
      "options": [
        "The argument is correct -- idempotent tools alone are complete and sufficient",
        "Idempotency and event logs solve the exact same problem, making one redundant",
        "Idempotency only matters for financial transactions, not general tool calls",
        "Without a log, every resume re-decides and re-attempts each step from scratch"
      ],
      "correct": 3,
      "explanations": [
        "Ignores the real cost this page's own repro measured directly -- the resumed run needed zero new model calls for completed steps specifically because a log existed; without one, that savings disappears entirely.",
        "Treats the two mechanisms as interchangeable when they solve different halves of the problem -- one prevents duplicate side effects, the other prevents redundant re-decision and preserves progress.",
        "An arbitrary, unsupported restriction -- nothing about the idempotency mechanism is specific to financial operations; any side-effecting tool call (booking, sending, provisioning) carries the same risk.",
        "Correct. Idempotent tools would indeed prevent a step from having a duplicate real-world effect if re-attempted -- but without a log, EVERY resume would have to re-decide and re-attempt every single step from the start, burning model calls and losing all progress tracking, even though no duplicate side effect would occur."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Temporal, ["Why Temporal"](https://docs.temporal.io/evaluate/why-temporal) — the Event History replay model and the agent-specific framing (loop as Workflow, calls as Activities).
- LangChain, ["Durable execution"](https://docs.langchain.com/oss/python/langgraph/durable-execution) — checkpointing, and the explicit caveat that in-memory checkpointers don't survive a process restart.
- DBOS — ["Database-Backed Durable Python Workflows"](https://github.com/dbos-inc/dbos-transact-py), an alternative durable-execution approach backed directly by a database rather than a separate orchestration service.
- Restate — [`restatedev/sdk-python`](https://github.com/restatedev/sdk-python), a third real durable-execution SDK in the same space.
