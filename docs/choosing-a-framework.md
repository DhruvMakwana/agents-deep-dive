# Choosing a Framework

!!! example "Hands-on"
    Full runnable recipe: [`choosing-a-framework/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/choosing-a-framework) in the companion cookbook — the deliberate exception to this whole site's "no framework by default" rule. The identical tool-calling task, solved on the same Claude model four real ways, with real lines of code and real call counts compared.

??? abstract "TL;DR — quick revision"
    - **A real, identical task solved four ways shows the actual trade-off, not a feature checklist.** The same tool-calling question, same model, same correct answer, same real call count (2) across the raw Anthropic client, LangChain/LangGraph's `create_agent`, Pydantic AI's `Agent`, and CrewAI's `Agent.kickoff` — but real, measured orchestration code of 15, 8, 5, and 9 lines respectively. The gap is what each approach makes you write by hand versus adopt as-is — and CrewAI's own real jump back up to 9 lines is itself informative: two of those lines are `role=`/`goal=`/`backstory=`, a real, deliberate design choice the other three don't make.
    - **"Own the loop" is a real, named position, not just a vibe.** 12-Factor Agents' framing: most products calling themselves agentic "aren't that agentic" — real production reliability, in this view, comes from owning your own control flow rather than delegating it to a framework's abstraction.
    - **AutoGen is in maintenance mode, by its own README's words**: "It will not receive new features or enhancements and is community managed going forward. New users should start with Microsoft Agent Framework." Building on it today means building on something the vendor itself says to migrate away from.
    - **A real dependency conflict surfaced and got fixed for real, not glossed over**: the full `pydantic-ai` package pulls in `fastmcp-slim`, which requires `python-dotenv>=1.1.0` — directly conflicting with this cookbook's shared `python-dotenv==1.0.1` pin. `pydantic-ai-slim[anthropic]` (skipping the unneeded `mcp` extra) resolves it cleanly, since this recipe never uses MCP.
    - **Framework adoption in real job descriptions is smaller than the discourse suggests.** Across 1,978 genuinely AI-technical job postings, LangChain appears in 8.1% (28 companies) and LangGraph in 5.0% (19 companies) — real, current numbers, not zero, but a small fraction of postings even in a corpus specifically filtered for AI-technical roles.

## The real trade-off, run four ways

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/choosing-a-framework/framework_shootout_docs.py:shared_tool"
```

**Raw Anthropic client** — full manual control over the message list, tool execution, and stop condition:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/choosing-a-framework/framework_shootout_docs.py:raw"
```

**LangChain / LangGraph** — `create_agent` builds a compiled graph that handles the loop:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/choosing-a-framework/framework_shootout_docs.py:langgraph"
```

**Pydantic AI** — a typed `Agent` handles the same loop, with its own conventions:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/choosing-a-framework/framework_shootout_docs.py:pydantic_ai"
```

**CrewAI** — a role-based `Agent`, run directly via `kickoff` (no `Task`/`Crew` wrapper):

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/choosing-a-framework/framework_shootout_docs.py:crewai"
```

!!! success "A real run — the identical question, four implementations, real numbers"
    **Input**, identical for all four: *"What is 15% of 240, plus 30?"* Ground truth (computed independently): 66.0.

    | Implementation | Real lines of orchestration code | Real calls | Real answer |
    |---|---|---|---|
    | Raw Anthropic client | 15 | 2 | *"15% of 240 plus 30 is **66**... 15% of 240 = 0.15 × 240 = 36. 36 + 30 = 66"* |
    | LangChain / LangGraph `create_agent` | 8 | 2 | *"15% of 240 is 36, and when you add 30 to that, you get **66**."* |
    | Pydantic AI `Agent` | 5 | 2 | *"15% of 240 is 36, and when you add 30 to that, you get **66**."* |
    | CrewAI `Agent.kickoff` | 9 | 2 | *"The answer is **66**. Here's the breakdown: 15% of 240 = 36. 36 + 30 = 66."* |

    All four got the correct answer, in the identical number of real calls, on the first real run — no framework needed a retry or produced a wrong result on this task. LangGraph's and Pydantic AI's final answers came back character-for-character identical, most likely coincidental convergence on the same natural phrasing for a simple, well-defined question, not evidence of a shared code path — their internal message handling and tool-calling wiring are genuinely different underneath.

    The line-count gap tracks exactly what each approach is actually for. The raw client's 15 lines are the tool-calling loop itself, in full: send, check `stop_reason`, execute the tool, append the result, repeat — visible and directly modifiable, at the cost of writing it every time. LangGraph's 8 lines hand that loop to a compiled graph, in exchange for adopting its message and state conventions. Pydantic AI's 5 lines go further in the same direction. **CrewAI's 9 lines climb back up, and the real reason why is itself informative**: two of those lines are `role=`, `goal=`, and `backstory=` — CrewAI's own real design asks you to describe an agent as a persona, not just wire a model to a tool list, because its actual target use case is role-based multi-agent orchestration, not a minimal single-tool task. None of this is a quality ranking — it's a real measurement of how much of the mechanism you write by hand versus accept as-is, which is the honest, concrete version of the trade-off "own the loop" debates are actually about.

    **Two real, disclosed side effects worth knowing before they surprise you**: Pydantic AI prints a startup banner to stdout by default — framework version, model, tool count, a nudge toward its paid observability product — unless `PYDANTIC_AI_NO_BANNER=1` is set. CrewAI prints its own real, rich, boxed console output for every agent lifecycle event (`LiteAgent Started`, tool execution, `LiteAgent Completed`) — and a real, tested finding here: passing `verbose=False` to the `Agent` constructor (already done in this recipe) reduces some of it, but real, live runs still show CrewAI's own start/complete banners regardless, suggesting this particular logging isn't fully gated by the per-agent flag. Neither is a bug — both are real default behavior worth knowing about before it shows up unexpectedly in a log stream.

## Own the loop, or don't: a real named position

12-Factor Agents states the position directly, not as a vague preference: most products marketed as agentic "aren't that agentic," and the fix is owning your own control flow rather than handing it to a framework's loop. This isn't an anti-framework stance in the abstract — it's a claim about where production reliability actually comes from: a loop you wrote yourself is a loop you can read, debug, and change at the exact line where something goes wrong, without first understanding a framework's own internal abstractions on top of the model call. Anthropic's own framing of their Claude Agent SDK — "Loop = gather context → take action → verify work" — describes a deliberate middle position: a real SDK, not a raw client, but one built to expose Claude Code's own harness as a library rather than to hide the loop's structure behind an unrelated abstraction.

The honest version of "when do you pick a framework" follows directly from this page's own measured trade-off: a framework earns its abstraction when its conventions (state management, message formatting, tool wiring) save real, repeated work across many similar agents — and costs you real debuggability exactly where your problem doesn't fit its assumptions cleanly. This page's own three real implementations all handled one simple tool call correctly; the real test of a framework's value shows up on a harder, more idiosyncratic task, not a demo-sized one.

## The rest of the landscape

The frameworks below weren't run in this page's real comparison (CrewAI, above, now is), but each has a real, current, verified fact worth knowing:

- **OpenAI Agents SDK** — `Agent(name="Assistant", instructions=...)` with `Runner.run_sync(agent, ...)`; built around Agents, Handoffs, Guardrails, Sessions, and Tracing as named primitives. The most stable of the vendor-specific SDKs' public API surface, though still a 0.x version as of this writing.
- **Claude Agent SDK** — `query(prompt=..., options=ClaudeAgentOptions(...))`; explicitly Claude Code's own harness exposed as a library, not a general multi-provider framework.
- **Google ADK** — `Agent(name=, model=, instruction=, tools=[...])`; reached a 2.0.0 GA release adding a Workflow Runtime and Task API, with the 1.x line still maintained in parallel.
- **Microsoft Agent Framework (MAF)** — the stated successor to both AutoGen and Semantic Kernel: "Microsoft Agent Framework is now available at version 1.0 as a production-ready release: stable APIs, and a commitment to long-term support."
- **smolagents** — "agents that think in code": the model writes and executes real Python as its action, rather than emitting structured tool-call JSON — the only actively-maintained framework built specifically around that design, though its release cadence has slowed to roughly one every couple of months.
- **AutoGen** — in maintenance mode, by its own README: "It will not receive new features or enhancements and is community managed going forward. New users should start with Microsoft Agent Framework." A real, current fact worth internalizing before choosing it for new work.
- **No-code builders** — visual, drag-and-drop agent builders exist and serve a genuinely different audience (non-engineers assembling simple automations); they trade away exactly the code-level control this page's own real comparison measures, which is the right trade for that audience and the wrong one for the production-engineering questions this site is otherwise about.

**CrewAI's own real, separate practical cost, beyond its line count above**: its own install pulls in over 130 packages, and it caps supported Python below 3.14 — a real, practical dependency-footprint cost this page's own line-count table doesn't capture, worth weighing alongside the code-length comparison, not instead of it.

## What real job descriptions say

Across a same-day snapshot of 1,978 genuinely AI-technical job postings (filtered from 12,977 total postings across 107 company boards), framework mentions are real but modest: LangChain appears in 8.1% of postings across 28 distinct companies, LangGraph in 5.0% across 19 companies, LlamaIndex in 3.0%, CrewAI in 0.7%, and AutoGen in 0.5%. Worth reading precisely: these are genuine, current numbers, not evidence any single framework dominates hiring — the majority of AI-technical roles name no specific agent framework at all, which is itself informative about how replaceable framework-specific expertise is treated relative to understanding the underlying mechanisms this whole site is about.

## Interview angle

**Weak answer** to "which agent framework do you use, and why": *"I use [framework], because it's the most popular / has the best documentation."* This answers a marketing question, not an engineering one, and it can't explain this page's own real measurement — popularity and line-count savings are different axes, and neither one alone tells you whether a framework's specific abstractions fit your actual problem.

**Strong answer**: name the real trade-off directly. A framework like LangGraph or Pydantic AI measurably reduces the orchestration code you write — this page's own real numbers show roughly a 2-3x reduction versus a raw client on a simple task — in exchange for your code now depending on that framework's conventions for state, messages, and tool wiring. That's worth it when you're building many similar agents and the framework's assumptions genuinely fit your use case; it's a real cost when your task needs something the framework doesn't anticipate, at which point you're debugging both your logic and the framework's abstraction over it. This page's own real CrewAI number sharpens the point further: its 9 lines *don't* show the same reduction, not because the framework is worse, but because its actual assumption — you're modeling a role-based persona, likely one of several collaborating agents — doesn't pay off on a single-tool task; the "does this framework's assumption fit my problem" question matters more than a generic line-count comparison. The 12-Factor Agents position — own your control flow — is the strongest version of the case for starting raw, and it's worth being able to state precisely, not just cite as a name.

**Follow-up to expect**: "wouldn't it be faster to just always use a framework and only go raw if you hit a wall?" That's a defensible default, but it inverts a real cost this page's own repro exposed for free: a framework dependency conflict (the `fastmcp-slim`/`python-dotenv` clash) that a raw-client-only project would never have hit at all. Frameworks add real surface area — their own dependencies, their own breaking changes, their own version compatibility matrix — that a raw client doesn't carry. The honest trade isn't "framework always wins until it doesn't" — it's that both directions have a real, measurable cost, and this page's job was to make both costs visible with numbers instead of assuming one of them away.

## Build it yourself — 30 minutes

1. Pick one small, well-defined tool-calling task with a checkable ground truth. Implement it with the raw provider client first, and count the real lines of orchestration code (not imports, not the tool itself).
2. Implement the identical task in one framework of your choice, using its `Agent`-equivalent primitive. Count the same thing. Compare the real numbers, not an assumption about which "should" be shorter.
3. Deliberately install just the extras your task needs (not the framework's full package) and check what that changes about the dependency tree — this page's own real fix (`pydantic-ai-slim[anthropic]` instead of `pydantic-ai`) is a genuine, repeatable pattern worth checking for in any framework you adopt.
4. Read one framework's own "why you'd use this" documentation and one critique of framework abstraction (12-Factor Agents is a good one) back to back, and write down, in your own words, which of your own projects' actual requirements each side's argument would change your mind about.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Choosing a Framework">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real run solved the identical tool-calling task with the raw Anthropic client (15 lines, 2 calls), LangChain/LangGraph's create_agent (8 lines, 2 calls), Pydantic AI's Agent (5 lines, 2 calls), and CrewAI's Agent.kickoff (9 lines, 2 calls) -- all four produced the correct answer, with CrewAI's line count climbing back up above LangGraph's and Pydantic AI's.",
      "question": "What is the most accurate interpretation of the line-count differences, including CrewAI's real increase?",
      "options": [
        "It measures code written by hand vs. adopted as-is, not a quality ranking",
        "It proves Pydantic AI is the objectively best choice for any agent task",
        "It shows the raw client is poorly written and should be shortened to match the frameworks",
        "The comparison is meaningless, since real agents never resemble a simple demo task"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real measured gap tracks exactly what each layer is for -- how much of the send/check/execute/append loop you write yourself versus hand to a framework's own conventions -- and CrewAI's own real increase specifically comes from its role/goal/backstory persona model, a genuine design choice for role-based multi-agent orchestration, not a sign of worse code.",
        "Overreaches from one narrow, simple task to a universal quality claim -- fewer lines on this specific task says nothing about how well any framework's assumptions fit a different, harder problem.",
        "Misreads the finding -- the raw client's length isn't a flaw, it's the actual content of the loop other approaches abstract away; shortening it would mean hiding logic, not writing it better.",
        "Overcorrects into dismissing a real, controlled measurement -- a simple task with a checkable ground truth is exactly what makes a fair comparison possible; it doesn't claim to generalize to every production scenario."
      ]
    },
    {
      "scenario": "AutoGen's own README states: 'AutoGen is now in maintenance mode. It will not receive new features or enhancements and is community managed going forward. New users should start with Microsoft Agent Framework.' A team is choosing a framework for a new project.",
      "question": "What's the most accurate way to weigh this fact?",
      "options": [
        "It's irrelevant, since AutoGen still technically works and has a large install base",
        "A real, current, vendor-stated reason to avoid it for new work, favoring its own successor",
        "It only matters for enterprise customers, not for smaller or personal projects",
        "AutoGen should be preferred specifically because it's stable and won't introduce breaking changes"
      ],
      "correct": 1,
      "explanations": [
        "Understates a direct, explicit vendor statement -- 'not receiving new features' and 'community managed' are real, material facts about a project's future trajectory, not something to set aside because old code still runs.",
        "Correct. This is a direct, current, first-party statement from the project itself naming its own successor -- exactly the kind of real, dated evidence that should weigh heavily in a framework choice for new work.",
        "Introduces an unsupported distinction -- the README's guidance makes no enterprise-vs-personal distinction at all.",
        "Inverts the real trade-off -- a maintenance-mode project not changing isn't the same as a supported project being stable; bugs and compatibility gaps with newer model APIs are less likely to be fixed at all."
      ]
    },
    {
      "scenario": "A real dependency conflict occurred when installing the full pydantic-ai package alongside this cookbook's shared python-dotenv==1.0.1 pin, because pydantic-ai's mcp extra pulls in fastmcp-slim, which requires python-dotenv>=1.1.0. Switching to pydantic-ai-slim[anthropic] (omitting the mcp extra) resolved it cleanly.",
      "question": "What's the most accurate lesson to draw from this specific incident?",
      "options": [
        "python-dotenv should be upgraded across the entire cookbook to avoid this in the future",
        "This proves Pydantic AI is poorly engineered compared to other frameworks",
        "Installing only the extras a project needs can avoid conflicts from unused functionality",
        "Dependency conflicts like this are rare and not worth checking for when adopting a framework"
      ],
      "correct": 2,
      "explanations": [
        "Proposes a real possible fix but not the one actually demonstrated -- the repro resolved the conflict by narrowing the install, not by changing the shared pin, which this cookbook explicitly wanted to keep stable across recipes.",
        "Overgeneralizes a single dependency conflict, tied to one specific extra, into a broad engineering-quality judgment not supported by the incident itself.",
        "Correct. The conflict came specifically from the mcp extra (unused by this recipe) pulling in fastmcp-slim's own stricter dotenv requirement -- installing only the needed extra avoided pulling in that unrelated dependency, a real, repeatable pattern.",
        "Contradicts the very incident being described -- a real conflict did occur and did need a real fix; treating this class of issue as rare is exactly the assumption this page's own repro shows can fail."
      ]
    },
    {
      "scenario": "Across 1,978 genuinely AI-technical job postings, LangChain appears in 8.1% (28 companies) and LangGraph in 5.0% (19 companies). A candidate argues: 'Since these percentages are low, frameworks like LangChain and LangGraph are barely used in the industry and not worth learning.'",
      "question": "What's the strongest problem with that conclusion?",
      "options": [
        "The conclusion is correct -- low percentages mean a skill is not worth learning for interviews",
        "The percentages must be undercounted, since these are well-known frameworks",
        "Job description mentions are the only valid measure of a framework's real-world importance",
        "8.1%/5.0% across dozens of real companies is genuine adoption, not 'barely used'"
      ],
      "correct": 3,
      "explanations": [
        "Draws too strong a conclusion from a relative percentage -- 'a minority of postings name it' and 'not worth learning' are very different claims, especially given the real company counts behind those percentages.",
        "Introduces an unsupported claim of measurement error with no evidence -- the page states this is a real, current snapshot from a defined methodology.",
        "Overstates the JD data's role -- this page explicitly notes JD frequency and real interview or production usage are related but distinct signals, not the sole measure of anything.",
        "Correct. 28 and 19 distinct real companies naming these frameworks specifically is genuine, verifiable adoption -- modest relative to all AI-technical postings, but a different, more precise claim than 'barely used.'"
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) — the "own your control flow" position and its framing of what makes an agent genuinely agentic.
- Anthropic, ["Building agents with the Claude Agent SDK"](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) — the gather-context/take-action/verify-work loop framing.
- [`langchain-ai/langgraph`](https://github.com/langchain-ai/langgraph) and [LangChain's `create_agent`](https://python.langchain.com/) documentation.
- [Pydantic AI](https://ai.pydantic.dev/) documentation.
- [`microsoft/autogen`](https://github.com/microsoft/autogen) — the maintenance-mode statement and migration guidance to Microsoft Agent Framework.
- [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) — the stated successor to AutoGen and Semantic Kernel.
