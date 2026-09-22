# The Agent Loop From Scratch

!!! example "Hands-on"
    Full runnable recipe: [`agent-loop-from-scratch/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/agent-loop-from-scratch) in the companion cookbook — a real tool-calling loop with the raw Anthropic client, no framework, checked against an independently computed ground truth.

??? abstract "TL;DR — quick revision"
    - **The model never calls your function.** At inference time it's given a schema and generates a *structured request* — a `tool_use` block naming a tool and its arguments. Your orchestration code parses that, runs the real function, and sends the result back as a new message (a `tool_result`) on the next turn. All the "agentic" behavior here is: structured generation, external execution, and a feedback loop — nothing more magical than that.
    - **The model has no guaranteed connection to reality until your code creates one.** It can request a tool that doesn't exist, or arguments that don't validate — there's no built-in check that its output matches the schema until your orchestration code checks.
    - **Tool descriptions are a prompt, not documentation.** Vague parameter descriptions produce vague argument extraction; concrete descriptions with worked examples measurably improve it. This is a design surface, not an afterthought.
    - **A hard iteration cap bounds round-trips, not per-call latency.** They're separate risks and need separate limits — a loop that can't run away forever can still stall for a long time on one slow tool call.
    - **A real run here produced a genuinely counter-intuitive result**: the no-tools baseline got every arithmetic step right on its own — what it actually lacked was a live exchange rate, which isn't an arithmetic problem at all.

## Mechanism: what "the model calls a tool" actually means

The weak version of this explanation — the one that gets corrected in an interview — is "the model calls the function." It doesn't, and there's no version of any major provider's API where it does.

What actually happens: your code sends the model a list of tool schemas (name, description, a JSON Schema for the arguments) alongside the conversation. If the model decides a tool would help, it doesn't execute anything — it generates a structured block saying, in effect, "I'd like to call `calculate` with `expression: '127.50 * 1.18'`." That's still just text generation, constrained to a particular shape. Your code reads that block, actually runs `calculate("127.50 * 1.18")` for real, and sends the return value back to the model as a new message. The model reads that result on its next turn and decides what to do next — answer, or request another tool. Anthropic's response shape makes this explicit: `stop_reason: "tool_use"` marks a turn as a request rather than an answer, and the result has to travel back as its own `tool_result` message, not get silently appended anywhere.

This is the entire mechanism. There's no hidden channel, no direct invocation, and critically: **no guarantee**. The model's tool request is just generated text that happens to be shaped like a function call — it can name a tool that doesn't exist, or supply arguments that don't match the schema, and nothing stops it from doing so except your code checking.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/agent-loop-from-scratch/agent_loop_docs.py:tool_schemas"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/agent-loop-from-scratch/agent_loop_docs.py:agent_loop"
```

## Design principles: the tool schema is a prompt

Two real tools back this page's demo. The first is a genuinely safe calculator — not `eval()` on whatever text the model produces, but an AST walk that only permits numeric literals and `+ - * / **`, so a malformed or adversarial expression can't do anything beyond arithmetic:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/agent-loop-from-scratch/agent_loop_docs.py:safe_calculator"
```

The second is a currency converter over a fixed, illustrative rate table:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/agent-loop-from-scratch/agent_loop_docs.py:currency_tool"
```

Notice both tool descriptions carry a worked example (`'127.50 * 1.18' to add an 18% tip`, `convert_currency(150.45, 'EUR', 'USD')`), not just a type signature. That's deliberate: a schema that only says `expression: string` leaves the model guessing at format; a schema with a concrete example in the description measurably improves how reliably the model extracts the right arguments. The description is doing real work — it's the closest thing this system has to a spec the model actually reads.

The other design choice worth naming: `calculate`'s `except` branch doesn't crash the loop or swallow the problem silently — it returns a structured `{"error": "..."}` dict, which becomes a normal `tool_result` the model can read and act on. A raw stack trace or a silent failure both leave the model unable to tell what went wrong; a structured error message lets it decide whether to retry with a fixed expression, try a different tool, or give up and say so.

## Stop conditions and budgets

The loop in this recipe is capped at a fixed number of iterations, and if the model still hasn't finished, it returns an honest "ran out of steps" result instead of looping forever:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/agent-loop-from-scratch/agent_loop_docs.py:agent_loop"
```

Worth being precise about what this cap does and doesn't cover: it bounds how many round-trips the loop can make, which is what keeps a stuck agent from silently burning an unbounded number of API calls. It says nothing about how long any *single* tool call is allowed to run — a tool that hangs for 90 seconds isn't touched by an iteration cap at all. Production loops need both: a step budget on the loop, and an independent timeout on each tool execution.

!!! success "A real run — one honest surprise"
    Question: *"I'm splitting a €127.50 dinner bill among 4 friends, and we want to add an 18% tip on top before splitting. What does each person owe, in US dollars?"* Ground truth, computed independently in plain Python: **$40.62**. One real run against Claude Haiku 4.5 — see the recipe README for the full trace.

    **No tools**: got every arithmetic step right on its own (18% tip, total, per-person split) — then, with no live exchange rate available, didn't assert a confident wrong number. It gave a range instead: *"approximately $40.80–$41.50 USD (depending on the current EUR/USD exchange rate, which fluctuates daily) ... Check a current converter for the exact amount."*

    **With tools**: converged in 3 turns to **$40.62 — the exact ground truth**. The trace shows something not asked for: the model issued two `calculate` calls in parallel in its first turn (Anthropic allows multiple `tool_use` blocks in one response), and the second one mildly re-did work the first had already computed rather than reusing it — correct, but not the shortest path available.

    The genuinely useful finding here isn't "the baseline can't do math" — it could. It's that the gap was a live-data problem, not an arithmetic one, and no amount of arithmetic skill closes a live-data gap without a real source for that data.

## Interview angle

**Weak answer** to "walk me through exactly what happens between the model deciding to call a tool and the tool's result reaching the model again": *"the model calls the API."* This skips the entire mechanism and reads as not having thought about it.

**Strong answer**: name the two-message round trip explicitly — the model generates a structured `tool_use` request (still just text generation, constrained to a schema), your orchestration code parses and executes it for real, and the result comes back as a distinct `tool_result` message the model reads on its next turn. Mention that nothing guarantees the request is valid until your code checks.

**Follow-up to expect**: a version of "your agent has 40 tools registered and keeps picking the wrong one or inventing arguments — what do you do?" A weak instinct is to collapse many tools into one mega-tool with a free-text instruction field. The stronger answer notices that this doesn't remove the selection problem, it just moves it from the model (choosing among well-described, schema-validated tools) into your own code (a keyword matcher choosing among internal actions from unstructured text) — trading a visible, debuggable failure mode for a hidden one, and losing schema validation in the process.

## Build it yourself — 30 minutes

1. Pick two small, deterministic tools whose outputs you can check independently (a calculator and a unit or currency converter both work well — you want a task where "did the model get this right" isn't a matter of taste).
2. Write the raw loop: send the conversation, check `stop_reason`, execute any requested tool for real, append the result as a new message, repeat with a hard iteration cap.
3. Ask the same question with no tools at all first, and write down what the model says.
4. Run it again with tools, and compare against an answer you computed independently, by hand or in a separate script — not by asking the model to check its own work.
5. Look at *why* the two answers differ, not just whether they differ. It's rarely as simple as "no tools means wrong."

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: The Agent Loop From Scratch">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A candidate explains tool calling this way: 'When the model calls a tool, it directly invokes your Python function through the API \u2014 the SDK handles the actual execution for you, which is why you never see the return value show up as a separate message.'",
      "question": "What's the strongest critique of this explanation?",
      "options": [
        "The model never directly invokes anything \u2014 it generates a structured tool_use request that your own code parses, executes, and sends back as a new message",
        "It's correct for Anthropic's API specifically, but OpenAI's function-calling API actually executes the functions server-side on your behalf, so the explanation doesn't generalize across providers",
        "It's mostly right, except the return value gets appended silently into the same message instead of arriving as a new one",
        "Nothing is wrong with it \u2014 this is an accurate, complete description of how tool calling works"
      ],
      "correct": 0,
      "explanations": [
        "Correct. No mainstream provider's tool-calling API executes your functions for you \u2014 Anthropic, OpenAI, and Google all require the caller to parse the request, run the real code, and send the result back explicitly. That round trip is the whole mechanism.",
        "A tempting 'maybe it varies by vendor' hedge, but false \u2014 execution is the caller's responsibility across every major provider's tool-calling API, not just Anthropic's.",
        "A specific, plausible-sounding technical detail that happens to be wrong: results travel back as their own distinct message (a tool_result), not appended silently onto an existing one \u2014 this page's own loop code makes that explicit.",
        "This is exactly the weak interview answer this page's Interview angle section warns about \u2014 it skips the entire mechanism."
      ]
    },
    {
      "scenario": "A team has 40 tools registered for one agent and notices frequent wrong-tool selection and invented arguments. An engineer proposes collapsing all 40 into a single 'do_anything' tool that takes one free-text 'instruction' string and dispatches internally via keyword matching, reasoning that a single tool means the model can't pick the wrong one.",
      "question": "What's the strongest critique of this proposal?",
      "options": [
        "It's a good fix \u2014 fewer tools in the schema means less chance of tool confusion",
        "It's correct, because collapsing everything into a single tool means the model genuinely never has to make any kind of selection decision at all",
        "It's wrong only because free-text instructions can't be validated by JSON Schema \u2014 everything else about the idea is sound",
        "It doesn't remove the selection problem, it just moves it into a hidden, harder-to-debug keyword matcher inside your own code"
      ],
      "correct": 3,
      "explanations": [
        "The naive read: fewer visible tools, less confusion. But the selection decision hasn't gone away, it's just moved somewhere you can't see it or fix it the same way.",
        "The selection decision still exists, it's just been relocated from the model's tool choice into your dispatch code's keyword matching \u2014 'no decision' is not what happened here.",
        "A half-right trap: schema validation loss is real, but framing it as the *only* problem misses the bigger one \u2014 you've hidden the selection logic where you can no longer inspect, test, or improve it the way you could with 40 separate tool descriptions.",
        "Correct. The model still has to figure out which of 40 things you mean from a free-text instruction, and now that decision happens inside an internal keyword matcher instead of the model's tool selection \u2014 which is typically less capable at exactly this kind of disambiguation and much harder to debug when it picks wrong."
      ]
    },
    {
      "scenario": "An agent's loop is capped at max_iterations=10. A teammate objects: 'That's not a real safety net \u2014 nothing stops a single iteration from being slow. A model can request a tool that hangs for 90 seconds, and your 10-iteration cap does nothing about that.'",
      "question": "Is the teammate right?",
      "options": [
        "No \u2014 max_iterations bounds the number of LLM round-trips, which is what actually matters for cost, and per-tool latency is a separate, unrelated concern",
        "Yes \u2014 an iteration cap bounds how many times the loop goes around, but says nothing about how long any single tool call is allowed to take",
        "No \u2014 the model itself automatically times out and returns control if a tool call runs too long",
        "Yes, but only because this example specifically used 10 iterations \u2014 a higher cap would fix the problem"
      ],
      "correct": 1,
      "explanations": [
        "Partially true (the cap does bound round-trip cost) but wrong to dismiss the teammate's specific point \u2014 round-trip count and per-call duration are genuinely separate risks, and this answer only defends against one of them.",
        "Correct. This page's own loop caps iterations but has no per-tool timeout \u2014 a real production version needs both, because they bound different things: total round-trips versus how long any one of them can take.",
        "There's no such mechanism at the model level. The model doesn't run the tool at all (see the mechanism section above) \u2014 it's the orchestration code's job to enforce any timeout, and nothing does that automatically.",
        "Misdiagnoses the fix \u2014 a higher iteration cap would let the loop run *more* times, which has no bearing at all on how long any single tool call is allowed to hang."
      ]
    },
    {
      "scenario": "After seeing this page's real run (no-tools baseline hedges with a $40.80-$41.50 range; the tool-using loop gets exactly $40.62), a colleague concludes: 'This proves LLMs are just bad at arithmetic, and tools fix that.'",
      "question": "What's the more accurate read of what actually happened?",
      "options": [
        "The colleague is right \u2014 the baseline's arithmetic was wrong, and the calculator tool corrected it",
        "The colleague is right, but only for multiplication and division specifically \u2014 the baseline's addition and subtraction steps happened to be fully reliable throughout",
        "The baseline got every arithmetic step right on its own \u2014 the real gap was the exchange rate, a live-data problem no arithmetic skill solves without a real source",
        "The gap is because Haiku is too small a model for this task \u2014 a larger model would have matched the tool-using answer with no tools at all"
      ],
      "correct": 2,
      "explanations": [
        "The clich\u00e9 conclusion, and specifically the wrong read of this run: the recipe README's real trace shows the baseline's tip, total, and per-person arithmetic were all correct \u2014 the number it was honestly unsure about was the exchange rate, not any arithmetic step.",
        "An oddly specific and fabricated distinction \u2014 nothing in the actual run supports operation-by-operation reliability differences; the model's addition, multiplication, and division were all correct in this trace.",
        "Correct \u2014 and this is the point worth remembering past this page: a gap that looks like a capability failure is sometimes a data-access failure instead, and the fix (a tool that supplies the missing data) is different from the fix for a genuine reasoning failure (a better model, or a different prompt).",
        "A tempting but wrong appeal to scale: model size doesn't create access to a live exchange rate that was never in the prompt or the model's training data at query time \u2014 this is a knowledge-access gap, not a capacity gap, and no larger model closes it without an actual data source."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, [Tool use with Claude](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) — the current tool-use API shape this recipe's code follows directly.
- Anthropic, ["Writing effective tools for agents — with agents"](https://www.anthropic.com/engineering/writing-tools-for-agents) (2025-09-11) — tool description design, namespacing, and returning model-readable context.
- Anthropic, ["Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents) (2024-12-19) — Appendix 2, "Prompt engineering your tools."
