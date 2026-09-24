# Observability and Debugging

!!! example "Hands-on"
    Full runnable recipe: [`observability-debugging/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/observability-debugging) in the companion cookbook — a real, spec-shaped OpenTelemetry GenAI trace that makes a hidden tool bug diagnosable without re-running anything.

??? abstract "TL;DR — quick revision"
    - **Agent failures don't look like failures from the outside — that's the actual problem this whole track exists to solve.** The real, current framing: *"When your AI agent returns a confidently wrong answer, your monitoring sees a successful 200 response."* (Jamie Mallers, OneUptime, 2026-03-28) — traditional health checks (did the call error, did it time out) are structurally blind to an agent that completes cleanly and is simply wrong.
    - **OpenTelemetry's real GenAI semantic conventions give this a standard shape, not a bespoke one**: span names follow `{gen_ai.operation.name} {gen_ai.tool.name}` (e.g. `execute_tool get_daily_active_users`), with real, specified attributes — `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens` — and, critically, `gen_ai.input.messages`/`gen_ai.output.messages` as the spec's own designated place for the actual tool call arguments and results.
    - **A real repro built exactly this: real spec-shaped spans around an agent call to a subtly buggy tool.** The tool silently returned *cumulative all-time* users instead of yesterday's daily figure. The call completed with zero errors — `stop_reason: "end_turn"` — while the real final answer confidently claimed the number was *"very healthy... roughly 46.8x the goal."*
    - **An independent, trace-only diagnosis — no access to the final answer, no re-running the agent — found the real root cause immediately**: inspecting only the real `gen_ai.output.messages` attribute on the tool-execution span, the check correctly flagged the exact value as *"implausibly large for a single day, likely a cumulative/all-time figure mislabeled as daily."* This is the concrete, working version of the "3 a.m. question": can you tell what actually went wrong from the trace alone, at 3 a.m., without waking anyone else up to help re-run it?

## Why "the call succeeded" is the wrong question for an agent

Traditional service monitoring answers one question well: did this request complete without an error? For a web server, that's usually the right question — a 200 status code is genuinely strong evidence the request worked. For an agent, it isn't. A tool call can return real, valid, well-formed JSON that's simply wrong; a model can reason fluently from that wrong data to a confident, articulate, incorrect conclusion; and every individual step along the way will report success, because nothing actually errored. The real, current framing for this gap: *"When your AI agent returns a confidently wrong answer, your monitoring sees a successful 200 response."* The same source's opening scenario makes the stakes concrete: *"It is 3:07am on a Tuesday. Your AI agent...silently stops working. No alert fires"* — not because nothing happened, but because what happened doesn't look like the kind of failure traditional monitoring is built to catch.

## What OpenTelemetry's real GenAI conventions actually specify

Rather than every team inventing its own tracing schema, OpenTelemetry maintains real, versioned semantic conventions specifically for generative AI and agents. The span-naming rule is concrete and mechanical: for tool execution, *"Span name SHOULD be `{gen_ai.operation.name} {gen_ai.tool.name}`"* — so a call to a tool named `get_daily_active_users` produces a span literally named `execute_tool get_daily_active_users`, not an opaque, team-specific label. The real, specified attributes for that span include `gen_ai.operation.name` (required; `execute_tool` is one of its defined values), `gen_ai.tool.name` (required), `gen_ai.tool.call.id` and `gen_ai.provider.name` (conditionally required), and token usage fields when available.

The detail that matters most for actual debugging — where the real tool call arguments and results get recorded — isn't a dedicated `gen_ai.tool.call.arguments` attribute; per the spec, that information belongs in `gen_ai.input.messages` and `gen_ai.output.messages`, structured JSON attached to the relevant messages, with an explicit, real privacy note attached: *"This attribute is likely to contain sensitive information"* and instrumentations *"MAY provide a way for users to filter or truncate"* it. Agent-level spans follow the same pattern one level up: `invoke_agent {gen_ai.agent.name}`, `create_agent {gen_ai.agent.name}`, `plan {gen_ai.agent.name}` — a consistent, standard shape for every layer of an agent's real execution.

## Repro: a confidently wrong answer, and whether the trace alone can catch it

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/observability-debugging/observability_debugging_docs.py:buggy-tool"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/observability-debugging/observability_debugging_docs.py:trace-emitter"
```

!!! success "A real run — a clean 'success,' a confidently wrong answer, and a real spec-shaped trace"
    **Task**: *"What were yesterday's daily active users for 'Nimbus', and is that healthy against our 50,000 DAU target?"* **The real bug**: `get_daily_active_users` actually returns cumulative all-time users, not yesterday's figure — a bug in the tool's own implementation, invisible to the model, which has no way to know the number it received doesn't mean what its name promises.

    **Traditional monitoring's real view of this run**: `{"status": "success", "stop_reason": "end_turn", "error": null}` — nothing to flag, nothing wrong.

    **The real final answer**: *"Yesterday's DAU for Nimbus was 2,340,000. Compared to your target of 50,000 DAU, that's well above target — roughly 46.8x the goal (2,340,000 vs. 50,000). This is a very healthy result, so either Nimbus is significantly outperforming expectations, or it's worth double-checking that the 50,000 target is still current/accurate for this product's scale."* Confident, well-reasoned, articulate — and wrong, because the underlying number was never a daily figure at all.

    **The real trace**, using the actual spec's attribute names: a `chat claude-sonnet-5` span (`gen_ai.operation.name: "chat"`, `gen_ai.provider.name: "anthropic"`, real token counts) followed by an `execute_tool get_daily_active_users` span carrying the real `gen_ai.input.messages: {"product": "Nimbus"}` and `gen_ai.output.messages: {"product": "Nimbus", "daily_active_users": 2340000}`.

    ```python
    --8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/observability-debugging/observability_debugging_docs.py:diagnose"
    ```

    **The independent, trace-only diagnosis** — run with no access to the final answer, no re-running the agent, only the real trace's own data — inspected the `execute_tool` span's `gen_ai.output.messages` attribute and correctly flagged it: *"get_daily_active_users returned 2,340,000 — implausibly large for a single day, likely a cumulative/all-time figure mislabeled as daily."* The exact tool call, the exact wrong value, and a specific, correct hypothesis — found entirely from a trace built to the real, standard convention.

## What this means in practice

The gap this page's repro makes concrete is the same gap the real "3 a.m." framing names directly: an agent's individual steps succeeding tells you almost nothing about whether the *task* succeeded, because the two things fail independently. A monitoring setup built around "did anything error" will miss this class of failure by construction — it's not a matter of better alerting thresholds, it's that the failure genuinely doesn't produce an error at all. What actually closes the gap is recording the real content of each step — not just that a tool ran, but what it was asked and what it returned — in a structured, standard shape a human (or a second, automated check) can inspect without reproducing the failure. That's the entire practical value of OpenTelemetry's GenAI conventions over an ad hoc logging scheme: `gen_ai.input.messages` and `gen_ai.output.messages` existing as real, specified, cross-vendor attribute names means any tool built against the convention can read a trace produced by any other, and the specific diagnostic check this page's repro ran — "is this value plausible for what it claims to be" — is exactly the kind of check that's only possible when the real inputs and outputs are actually captured, not summarized away.

## Interview angle

**Weak answer** to "how would you monitor an agent in production?": *"Set up alerts for errors and track uptime."* This is the traditional-monitoring answer this page's own repro directly falsifies — the demo's agent call had zero errors and 100% uptime by that definition, while still producing a real, consequential wrong answer that traditional monitoring would never surface.

**Strong answer**: instrument every model call and tool call with structured, standard-shaped spans — specifically the real OpenTelemetry GenAI conventions (`gen_ai.operation.name`, `gen_ai.tool.name`, and the actual inputs/outputs via `gen_ai.input.messages`/`gen_ai.output.messages`) — so that a failure which doesn't produce an error can still be diagnosed from the trace's actual content. This page's own repro demonstrates the mechanism directly: a wrong answer with a clean success status was made diagnosable in one automated check, entirely from trace data, with no need to reproduce the failure live.

**Follow-up to expect**: "isn't logging every tool call's full input and output a privacy or cost problem?" Yes, and the real spec accounts for it directly — it explicitly warns that `gen_ai.input.messages`/`gen_ai.output.messages` are *"likely to contain sensitive information"* and states instrumentations *"MAY provide a way for users to filter or truncate"* them. The practical answer is the same discipline Context Editing and compaction techniques already require elsewhere in this project: capture what you need for real diagnosis, truncate or redact what you don't, and treat that boundary as a deliberate design decision rather than an afterthought — not "log everything forever" and not "log nothing to stay safe," either extreme defeats the purpose.

## Build it yourself — 30 minutes

1. Build one deliberately buggy tool whose output looks plausible but is subtly wrong (a units mismatch, a stale cache, a metric confused with a similarly-named one, the way this page's cumulative-vs-daily bug does). Run an agent against it and confirm the call completes with no errors.
2. Wrap the call in real spans using the actual OpenTelemetry GenAI attribute names — `gen_ai.operation.name`, `gen_ai.tool.name`, `gen_ai.input.messages`, `gen_ai.output.messages` — rather than an ad hoc logging format. This page's own recipe is a minimal, working template.
3. Write one independent diagnostic check that only reads the trace's own data — no access to the agent's final answer, no re-running anything — and see whether it can catch the real bug. This page's repro used a simple plausibility check (is this value too large to be what it claims); yours might check units, ranges, or consistency with another field.
4. If your check catches the bug from the trace alone, that's the real, working version of the "3 a.m. question" — you now know what a genuinely useful trace needs to contain, instead of assuming your current logging already has it.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Observability and Debugging">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro's agent call completed with stop_reason 'end_turn' and no exceptions, while its real final answer confidently misreported a cumulative all-time user count as a healthy daily figure, calling it 'roughly 46.8x the goal' and 'very healthy.'",
      "question": "What does this specific outcome most precisely demonstrate about traditional 'did the call succeed' monitoring for agents?",
      "options": [
        "That such monitoring is structurally blind to failures that don't produce any error",
        "That traditional monitoring is generally useless and should never be used for anything",
        "That the model itself was responsible for a bug that traditional monitoring should catch",
        "That stop_reason values are unreliable and should not be trusted for any purpose"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, demonstrated point is structural: an agent's individual steps can each genuinely succeed (no errors, no timeouts) while the overall task is still wrong, because a monitoring approach built around error detection has no mechanism to catch a semantically wrong but technically clean result.",
        "A sweeping overgeneralization -- traditional monitoring remains genuinely useful for what it's designed to catch (crashes, timeouts, exceptions); the finding is about a specific class of failure it can't see, not a claim that it's worthless everywhere.",
        "Misattributes the bug -- the real root cause was in the TOOL's implementation (returning the wrong metric), not something the model did wrong; the model reasoned correctly from data it had no way to know was mislabeled.",
        "Not what was shown or claimed -- stop_reason 'end_turn' was an accurate, correct report of what actually happened (the model concluded its turn normally); the issue isn't that this signal was wrong, it's that this signal alone is insufficient to detect the real problem."
      ]
    },
    {
      "scenario": "OpenTelemetry's real GenAI semantic conventions specify that a tool call's actual arguments and results are recorded via gen_ai.input.messages and gen_ai.output.messages, rather than via a dedicated attribute like gen_ai.tool.call.arguments.",
      "question": "Why does this specific detail matter for the kind of diagnosis this page's repro performed?",
      "options": [
        "It means tool call data is technically impossible to inspect in any real trace",
        "It means the real content needed for diagnosis has a defined, standard location",
        "It means every GenAI trace automatically redacts all tool call data by default",
        "It means tool calls cannot be distinguished from ordinary chat messages in a trace"
      ],
      "correct": 1,
      "explanations": [
        "Directly contradicted by the repro itself -- the real trace's gen_ai.output.messages attribute was successfully inspected and used to diagnose the bug; the data was fully present and readable, just under a specific attribute name.",
        "Correct. Knowing that the real spec designates gen_ai.input.messages/gen_ai.output.messages as the place tool call data lives is exactly what let the repro's diagnostic check know where to look -- a standard location means any compliant trace, from any tool, can be inspected the same way.",
        "Not accurate -- the spec notes this data 'is likely to contain sensitive information' and instrumentations 'MAY provide a way to filter or truncate' it, meaning redaction is an available, optional capability, not an automatic default behavior.",
        "Unsupported -- the real trace in the repro clearly distinguished a 'chat' operation span from an 'execute_tool' operation span via the gen_ai.operation.name attribute, a real, explicit distinction the spec provides."
      ]
    },
    {
      "scenario": "A real repro ran an independent diagnostic check using ONLY the trace's own span data -- no access to the agent's final text answer, and without re-running the agent -- and it correctly identified the exact tool call and exact bad value responsible for the wrong final answer.",
      "question": "What is the most precise significance of the diagnostic check having no access to the final answer?",
      "options": [
        "It proves the final answer's wording was itself completely irrelevant to the bug",
        "It demonstrates that re-running the agent would have been faster in this case",
        "It shows the real root cause was findable from structured data, not from reading prose",
        "It means the diagnostic check could not have used any information from the trace at all"
      ],
      "correct": 2,
      "explanations": [
        "Overstates the point -- the final answer's wording is a separate, real symptom (it's what a user actually saw), just not what the DIAGNOSTIC check needed to find the cause; both facts can be true without contradiction.",
        "Not addressed or claimed by the repro -- the entire point of building a trace-based check was to AVOID needing to re-run anything; no comparison of relative speed between re-running and trace inspection was made or implied.",
        "Correct. The real, demonstrated significance is that a genuinely useful diagnosis didn't require reading or interpreting the agent's natural-language output at all -- the exact tool call and exact wrong value were both directly present in the trace's own structured attributes, which is precisely what makes automated, scalable diagnosis possible.",
        "Contradicts the setup directly -- the check used exactly the trace's own span data (specifically gen_ai.output.messages) as its sole input; 'no access to the final answer' describes what it excluded, not that it had no data to work with at all."
      ]
    },
    {
      "scenario": "The real OpenTelemetry GenAI spec explicitly states that gen_ai.input.messages and gen_ai.output.messages are 'likely to contain sensitive information' and that instrumentations 'MAY provide a way for users to filter or truncate' them.",
      "question": "What is the most accurate way to reconcile this real privacy note with the value of capturing full tool call data for debugging, as this page frames it?",
      "options": [
        "The privacy note means full data capture for debugging should never actually be done",
        "There's no real tension here since sensitive data and debugging value never overlap",
        "The spec requires all sensitive fields to be automatically encrypted before being stored",
        "Filtering and truncation should be a deliberate design choice made for each real case"
      ],
      "correct": 3,
      "explanations": [
        "Overstates the guidance -- the spec frames filtering/truncation as something instrumentations MAY provide, an optional capability, not an instruction that full capture must never happen; the repro itself captured full data specifically because it was needed for diagnosis.",
        "Understates the real tension the spec itself acknowledges -- the same field ('likely to contain sensitive information') is explicitly the one needed for the kind of diagnosis this page's repro performed, so the overlap is real and directly named in the spec's own text.",
        "Not stated anywhere in the real spec text quoted -- the spec mentions optional filtering or truncation, not a requirement for automatic encryption of any specific fields.",
        "Correct. This is the page's own stated framing: treat what to capture, filter, or truncate as a deliberate decision made with the specific diagnostic need in mind (matching the same discipline context editing and compaction require elsewhere), rather than defaulting to either extreme -- logging everything forever or logging nothing to stay safe."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- OpenTelemetry, [GenAI semantic conventions (semantic-conventions-genai repository)](https://github.com/open-telemetry/semantic-conventions-genai) — the real span naming rules and `gen_ai.*` attribute definitions this page's repro is built against directly.
- Jamie Mallers, ["Your AI Agent Crashed at 3am and Nobody Noticed"](https://oneuptime.com/blog/post/2026-03-28-your-ai-agent-crashed-at-3am-nobody-noticed/view) (OneUptime, 2026-03-28) — the real "3 a.m." framing and the "confidently wrong answer... successful 200 response" quote this whole page is built around.
- [Evaluating Agents](evaluating-agents.md) — outcome vs. trajectory grading, a closely related but distinct discipline: this page is about *seeing* what happened in production; that page is about *grading* whether it was right before deployment.
- [Benchmark Atlas](benchmark-atlas.md) — action-state grading (checking what actually happened, not what the reply claims), the same underlying discipline this page applies to live production traces instead of benchmark tasks.
