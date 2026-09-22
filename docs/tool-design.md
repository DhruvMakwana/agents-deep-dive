# Tool Design

!!! example "Hands-on"
    Full runnable recipe: [`tool-design/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/tool-design) in the companion cookbook — the same fictional task-tracker data exposed through three tool surfaces, measured on real call count and token count, plus terse vs. actionable tool-error messages compared on real recovery quality.

??? abstract "TL;DR — quick revision"
    - **A tool description is a prompt, not documentation** — it steers behavior the same way a system prompt does. Anthropic reports that precise tool-description refinements (nothing else) took Claude Sonnet 3.5 to state-of-the-art on SWE-bench Verified.
    - **Consolidated, purpose-built tools beat many granular endpoint wrappers on both calls and tokens, measured**: on the identical question, a single `get_blocked_tasks_with_reasons` tool took 2 calls and 1,563 tokens; exposing `list_tasks` + `list_comments` separately and letting the model compose them took 3 calls and 3,079 tokens.
    - **Code execution's real advantage didn't show up as a clean win on one fixed question** — it tied endpoint wrappers on call count (3 each) here, though it used fewer tokens (2,951 vs. 3,079). Its actual case is avoiding a combinatorial explosion of purpose-built tools across many *different* possible query shapes, not raw efficiency on one you already anticipated.
    - **A real error-message comparison's finding wasn't about retry count — both a terse and an actionable validation error took the model exactly 2 calls to recover.** What differed was the value it recovered *to*: the terse error (no information) got a generic, disconnected guess; the actionable error (stating the valid options) got a genuinely closer match to what the user actually meant.
    - **Interface design measurably matters independent of the underlying model** — SWE-agent's own contribution wasn't a better model, it was a custom Agent-Computer Interface, and the paper reports this got a non-interactive baseline's pass@1 up to 12.5% on SWE-bench.

## Schema as prompt

The tool schema — its name, its description, its parameter names and descriptions — is not metadata the model consults once and moves past. It's read on every turn a tool might be relevant, the same way a system prompt is, and it steers behavior with the same weight. Anthropic's own guidance states the practical test plainly: describe a tool the way you'd describe it to a new hire, and make context that's obvious to you but invisible to the model explicit in the text. The reported result of doing this rigorously: refining tool descriptions alone — no model change, no architecture change — took Claude Sonnet 3.5 to state-of-the-art performance on SWE-bench Verified.

Ambiguous parameter names (`user` instead of `user_id`) and thin descriptions don't just risk a wrong call — they cost tokens and turns recovering from one, which the rest of this page measures directly rather than asserting.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tool-design/tool_design_docs.py:tool_surfaces_data"
```

## The agent-computer interface: SWE-agent's finding

SWE-agent's paper frames a claim worth stating precisely, because it's easy to round off into "better prompting": language model agents are "a new category of end users with their own needs and abilities," and would benefit from interfaces built specifically for them — not the same interfaces built for humans, repurposed. The paper's own custom Agent-Computer Interface (ACI) — file navigation and editing commands, test execution, purpose-built for an LM to use reliably — reports a 12.5% pass@1 on SWE-bench, "far exceeding" the previous state of the art with non-interactive language models on the same benchmark family (Yang et al., 2024). The interface, not a stronger model, is the paper's contribution. That's the same idea underneath every comparison on this page: the surface an agent gets isn't a thin wrapper around "the real API" — it's a genuine design decision with a measurable effect on the outcome.

## Endpoint wrappers vs. consolidated vs. code execution, measured

Three ways to expose the same fictional task data to the same model, for the same question: *"Which tasks are currently blocked, and what's the most recent blocking reason mentioned in their comments?"*

**Endpoint wrappers** — one tool per low-level operation, composition left entirely to the model:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tool-design/tool_design_docs.py:endpoint_wrappers"
```

**Consolidated** — one purpose-built tool that does the filtering server-side:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tool-design/tool_design_docs.py:consolidated"
```

**Code execution** — the model writes code against a small library instead of calling named tools one at a time:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tool-design/tool_design_docs.py:code_execution"
```

!!! success "A real run — the exact question, three surfaces, real call and token counts"
    **Input**, identical for all three surfaces: *"Which tasks are currently blocked, and what's the most recent blocking reason mentioned in their comments? List each task's title and that reason."* Ground truth (computed independently, not by the model): 3 blocked tasks — *Fix login redirect bug* (waiting on infra to rotate the session-signing key), *Migrate billing service to new queue* (waiting on legal sign-off on data-retention policy), *Deprecate legacy export endpoint* (an internal dashboard still depends on it).

    | Surface | Calls | Total tokens |
    |---|---|---|
    | Endpoint wrappers | 3 | 3,079 |
    | Consolidated | 2 | 1,563 |
    | Code execution | 3 | 2,951 |

    All three surfaces got the correct answer, Claude Sonnet 5 throughout. **Consolidated won outright** — half the tokens of endpoint wrappers, the fewest calls. **Code execution's result is the honest, unglamorous one**: it tied endpoint wrappers on calls (3 each) and only modestly beat it on tokens (2,951 vs. 3,079) — nowhere near a clean win on this specific question. That's not a failure to report quietly; it's the actual finding. Code execution's real case isn't "cheaper on any one query" — it's that a consolidated surface only stays cheap if you correctly anticipated the query shape in advance and built a bespoke tool for it. Ask a differently-shaped question tomorrow and a fixed set of consolidated tools may not cover it, while code execution's model-written filtering logic generalizes to a new shape for free. One fixed question can't demonstrate that scaling argument — it would take many different query shapes, run against the same three surfaces, to show it honestly, which is future work, not something to claim from this run.

## Errors as observations

A tool's error message is also part of its interface, read by the model as an observation it has to act on — not a log line for a human. Anthropic's own guidance is specific: errors should give "specific and actionable improvements, rather than opaque error codes or tracebacks."

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tool-design/tool_design_docs.py:errors_as_observations"
```

!!! success "A real run — same request, two error conditions, both shown in full"
    **Input**, identical for both conditions: *"Create a task to fix the login bug, due next Friday, pretty urgent. You can't ask me follow-up questions right now — call create_task directly. For the priority field, pass the user's own words exactly as given ('pretty urgent'); make your best guess for the date."* (Deliberately forcing a first attempt with an invalid priority value, so there's a real error to observe and recover from.)

    **Terse condition** — first call: `{"title": "fix the login bug", "priority": "pretty urgent", "due_date": "2025-01-10"}` → `{"error": "ValidationError: priority"}`. Second call, recovering: `{"title": "fix the login bug", "priority": "high", "due_date": "2025-01-10"}` → succeeded. **2 calls.**

    **Actionable condition** — first call: `{"title": "fix the login bug", "priority": "pretty urgent", "due_date": "2025-01-10"}` → `{"error": "Invalid priority 'pretty urgent'. Must be exactly one of: low, medium, high, urgent."}`. Second call, recovering: `{"title": "fix the login bug", "priority": "urgent", "due_date": "2025-01-10"}` → succeeded. **2 calls.**

    Both conditions converged in exactly the same number of calls — the actionable error didn't make recovery faster. What it changed is *which value* the model recovered to: given no information about what's valid, the model played it safe and picked `"high"`, a generic middle value with no real connection to "pretty urgent." Given the actual list of valid options, it picked `"urgent"` — the option that genuinely matches what the user said. An honest, more precise version of "better errors help": not always fewer turns, but a better answer at the end of the same number of turns.

## Tool evals: measuring what actually matters

Everything above is a small version of a practice Anthropic reports running at production scale: building held-out evaluation sets for real tool integrations (Slack, Asana) and iterating tool descriptions and shapes against measured accuracy, the same way one would iterate a prompt. Separately, Anthropic's advanced tool-use work reports concrete numbers for related techniques: a Tool Search mechanism (only loading tool definitions relevant to a query instead of the full library) cutting token usage by 85% and moving one internal MCP evaluation's accuracy from 49% to 74% on Opus 4 (79.5% to 88.1% on Opus 4.5); Programmatic Tool Calling cutting tokens by 37% on a complex research task; and worked examples in a tool's description (not just its schema) moving accuracy on tricky parameter handling from 72% to 90%. The pattern across all of it, including this page's own three small real numbers above: tool design is something you measure and iterate, the same way you would a prompt or a retrieval pipeline — not something you get right once by intuition and leave alone.

## Interview angle

**Weak answer** to "how would you decide how many tools to expose to an agent": *"Fewer tools is always better, since it's less for the model to get confused by."* This states a real, common pressure but not the actual trade-off, and it can't explain this page's own real result — consolidated (1 tool) beat endpoint wrappers (2 tools) on this run, but code execution (also 1 tool) didn't beat endpoint wrappers at all on calls.

**Strong answer**: the real axis isn't tool *count*, it's whether the tool surface matches the shape of the questions actually being asked. A single consolidated tool is cheap exactly when you can predict the query shape in advance and build the tool to answer it directly — which is precisely what this page's real run showed for one fixed question. The moment the range of possible queries gets wide or unpredictable, a fixed set of purpose-built tools either grows without bound (defeating the "fewer tools" instinct) or fails to cover a shape nobody anticipated — which is the actual argument for code execution and endpoint-style primitives: not that they're cheaper per call, but that they compose to answer questions nobody pre-built a tool for.

**Follow-up to expect**: "so should every agent just use code execution instead of purpose-built tools?" No — this page's own numbers are the counter-evidence: code execution didn't win on this specific, single, well-anticipated query. The real decision criterion is the actual diversity of queries the agent will face in production — narrow and predictable favors consolidated tools measured against real evals; broad and unpredictable favors giving the model primitives and letting it compose them, accepting a per-query cost that's sometimes higher than a tool you could have pre-built, in exchange for coverage you couldn't have pre-built for everything.

## Build it yourself — 30 minutes

1. Pick a small, fictional dataset and a fixed question against it — something with an answer you can compute independently, so you can check correctness, not just "did it respond."
2. Build the endpoint-wrapper surface first: one tool per basic operation, no composition helpers. Run the real question, record the real call count and token usage from your API response's usage field.
3. Build one consolidated tool purpose-built for that exact question. Run the same question again, compare the real numbers against step 2 — don't estimate them.
4. Pick one tool in your surface and deliberately give it a validation error with no information (just an error type). Trigger it, then rewrite the error to name the actual valid values or expected format, and trigger it again with the identical ambiguous request. Compare not just how many turns each takes, but what value the model actually recovers to.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Tool Design">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A team building an agent's tool library debates two options: many small endpoint-wrapper tools (list_tasks, list_comments, list_users, ...) versus a handful of purpose-built consolidated tools. One engineer argues: 'Consolidated tools are just strictly better -- this page's own real run showed consolidated beating endpoint wrappers on both calls and tokens.'",
      "question": "What's the strongest problem with generalizing that conclusion?",
      "options": [
        "It's backwards -- endpoint wrappers actually won on tokens, not consolidated tools",
        "Consolidated tools need the query shape anticipated in advance; a new shape may go uncovered",
        "Consolidated tools are strictly worse for security, since server-side composition exposes more data",
        "The real run actually showed code execution winning on every single metric instead"
      ],
      "correct": 1,
      "explanations": [
        "Misstates the actual real numbers -- consolidated used FEWER tokens (1,563) than endpoint wrappers (3,079) in the real measured run, not more.",
        "Correct. The real run measured one fixed, well-anticipated question -- consolidated wins precisely because that question's shape was known ahead of time and a tool was built for it. A different question tomorrow might not be covered by that same fixed tool set at all, which is the actual scaling limitation a single-question measurement can't reveal.",
        "An unsupported, fabricated security claim -- nothing about server-side composition inherently exposes more data than client-side composition; it depends entirely on what each tool chooses to return.",
        "Contradicts the real measured numbers directly -- code execution tied endpoint wrappers on calls (3 each) and only modestly beat it on tokens in this run, not a win on 'every metric.'"
      ]
    },
    {
      "scenario": "After seeing this page's error-message comparison (terse vs. actionable validation errors, both took exactly 2 calls to recover), a developer concludes: 'Since both took the same number of calls, actionable error messages don't actually matter -- it's not worth the effort to write good tool errors.'",
      "question": "What does this conclusion get wrong?",
      "options": [
        "The actionable condition actually took 3 calls, not 2, in the real run",
        "Call counts matched, so error quality truly has no measurable effect here",
        "It conflates 'same turns' with 'no effect' -- the real difference was in recovery value",
        "Actionable errors should be judged only on preventing the first attempt from ever failing"
      ],
      "correct": 2,
      "explanations": [
        "Misstates the real numbers -- both conditions took exactly 2 calls in the actual recorded run, not 3.",
        "Gets the premise right (call counts matched) but the conclusion wrong -- 'no effect on call count' isn't the same claim as 'no effect at all,' and this run showed a real effect on a different dimension.",
        "Correct. The real run's finding wasn't about speed -- it was that the terse error's lack of information led to a generic, disconnected guess ('high'), while the actionable error's explicit valid-options list led to a genuinely closer match to the user's actual words ('urgent'). Recovery quality, not recovery speed, is where the real difference showed up.",
        "Introduces a standard not used anywhere on this page or supported by the data -- neither condition was expected to succeed on the first attempt, since the request was deliberately designed to trigger an error for comparison."
      ]
    },
    {
      "scenario": "A candidate in a system-design interview is asked to justify SWE-agent's reported 12.5% pass@1 improvement on SWE-bench and answers: 'That's just because they used a stronger underlying language model than prior non-interactive approaches.'",
      "question": "What's the most accurate correction?",
      "options": [
        "The paper's own contribution is the custom Agent-Computer Interface itself, not a stronger model",
        "Right, but the interface does contribute a small, secondary boost worth mentioning too",
        "SWE-bench scores aren't comparable across papers, so this claim can't be verified either way",
        "The improvement is attributable to allowing more tool calls per task, unrelated to interface design"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The paper frames language-model agents as a distinct category of end user needing interfaces built for their specific needs and abilities -- the contribution under test is the interface (navigation, editing, test-execution commands), independent of which underlying model runs it.",
        "Inverts the paper's own emphasis -- the ACI is presented as the central contribution being evaluated, not a minor addendum to a model upgrade.",
        "Overstates the uncertainty -- the paper directly compares its interactive, ACI-equipped approach against prior non-interactive LM approaches on the same benchmark family, which is exactly the comparison the pass@1 number is drawn from.",
        "Introduces an unsupported, fabricated mechanism -- call budget isn't the variable the paper frames its contribution around; interface design is."
      ]
    },
    {
      "scenario": "A skeptical reviewer reads this page's tool-surface comparison and says: 'Code execution tied endpoint wrappers on calls and only modestly beat it on tokens -- so code execution is basically pointless here, just use endpoint wrappers.'",
      "question": "What's the strongest flaw in that reasoning?",
      "options": [
        "The reviewer misread the numbers -- code execution actually won on both calls and tokens in the real run",
        "Code execution should always be preferred no matter what, as the more modern technique here",
        "The comparison is invalid, since code execution and endpoint wrappers ran against different underlying data",
        "One anticipated query can't show code execution's real edge -- covering shapes nobody pre-built for"
      ],
      "correct": 3,
      "explanations": [
        "Misstates the real numbers -- code execution tied endpoint wrappers on calls (3 each) and only modestly beat it on tokens (2,951 vs. 3,079), not a win on both.",
        "An unsupported, fabricated preference rule -- this page explicitly measures and reports honest results rather than declaring a technique universally superior regardless of what's measured.",
        "Factually wrong about the recipe -- all three surfaces (endpoint wrappers, consolidated, code execution) ran against the identical fictional dataset and identical question.",
        "Correct. The reviewer's premise is accurate (code execution didn't clearly win here) but the conclusion doesn't follow -- code execution's real case is covering query shapes nobody pre-built a tool for, which a single fixed, well-anticipated question structurally cannot demonstrate either way."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Writing effective tools for agents"](https://www.anthropic.com/engineering/writing-tools-for-agents) (2025-09-11) — schema as prompt, tool consolidation, concise high-signal responses, errors as actionable feedback.
- Yang et al., ["SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering"](https://arxiv.org/abs/2405.15793) (arXiv 2405.15793, submitted 2024-05-06) — interface design measurably matters independent of the underlying model.
- Anthropic, ["Advanced tool use"](https://www.anthropic.com/engineering/advanced-tool-use) (2025-11-24) — Tool Search Tool, Programmatic Tool Calling, and Tool Use Examples, with reported token and accuracy numbers.
