# Tools at Scale

!!! example "Hands-on"
    Full runnable recipe: [`tools-at-scale/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/tools-at-scale) in the companion cookbook — the same 3-step helpdesk task against a 25-tool library three ways, with real call counts and real token counts from a live run, plus a real retriever bug caught and fixed before any paid API calls.

??? abstract "TL;DR — quick revision"
    - **A large tool library isn't free just because a model can technically pick the right tool from it.** Every tool definition sits in context on every turn, whether or not it's used — Anthropic's Tool Search Tool defers loading full definitions until they're actually needed, cutting token usage by 85% while keeping the full library reachable, and lifting Opus 4's MCP evaluation accuracy from 49% to 74% (Opus 4.5: 79.5% to 88.1%).
    - **Programmatic tool calling changes what enters context, not just how much**: the model writes one program that calls multiple tools and controls what actually gets returned, instead of one tool call per turn with every intermediate result echoed back. Anthropic's own measurement: 43,588 → 27,297 tokens, a 37% reduction on complex research tasks.
    - **Code execution with MCP takes the same idea further**: presenting MCP servers as code APIs instead of direct tool calls, so intermediate results "stay in the execution environment by default" and "the agent only sees what you explicitly log or return" — Anthropic's cited case: 150,000 → 2,000 tokens, a 98.7% reduction.
    - **A real, minimal repro of all three reproduced the shape of these effects at small scale**: naive (25 tools in context) cost 9,011 tokens over 4 calls; a keyword-filtered tool-search condition cost 4,065 tokens over the same 4 calls (-55%, from not paying for 22 irrelevant tool definitions); programmatic tool calling cost 3,743 tokens over just 3 calls (-58%, from not echoing intermediate results back as separate turns).
    - **A real bug in the tool-search condition's retriever was caught before any paid calls**: raw keyword overlap let generic words ("check", "status") shared between the task question and filler tool descriptions outscore the real tools' more specific keyword sets, excluding 2 of the 3 tools the task actually needed. Fixed by filtering common words from both sides before scoring — verified with a zero-cost dry run before spending a single real API call.

## The problem: a tool library is context, whether or not it's used

Every tool definition passed to `tools=` in a Messages API call — its name, description, and JSON schema — sits in the model's context on every single turn of a conversation, regardless of whether that tool is ever called. A helpdesk agent with 25 tools pays for the full definitions of all 25 the moment the first message goes out, even if the task only ever touches 3 of them. Anthropic's own framing is direct about the cost: a large tool library can consume 50,000+ tokens before a single user message is even answered, and Tool Design (an earlier page on this site) already covered why that hurts — more tools in context means more opportunities for the model to confuse similarly-named or similarly-described tools with each other, not just more tokens billed.

The three real techniques below all attack the same problem from different angles: what if the model didn't have to see every tool definition, on every turn, whether or not it needed it?

## Technique 1: tool search (defer loading, search on demand)

Anthropic's Tool Search Tool marks individual tool definitions with `defer_loading: true` — instead of every marked tool's full definition being loaded into context upfront, the model gets a lightweight search capability and only pulls in the full definition for a tool once it's actually decided that tool is relevant. The library stays fully reachable; what changes is *when* each definition's tokens get spent.

The real numbers: *"This represents an 85% reduction in token usage while maintaining access to your full tool library."* And the effect isn't just cheaper — it's more accurate: *"Opus 4 improved from 49% to 74%, and Opus 4.5 improved from 79.5% to 88.1% with Tool Search Tool enabled"* on an MCP evaluation. Fewer, more relevant tool definitions in context at once means less chance of picking the wrong one from a crowded, similar-sounding list.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tools-at-scale/tools_at_scale_docs.py:tool-search"
```

!!! success "A real run — the input task, the retriever's real output, and the transcript answer"
    **Input**, identical across all three conditions in this recipe: *"Employee EMP-4471 needs access to DataViz Pro. Look up the employee, check whether a license seat is available for their department, and provision access if one is."* Three real tools solve it (`lookup_employee`, `check_license_availability`, `provision_access`); 22 unrelated filler tools (password resets, hardware orders, badge access, facilities requests) sit alongside them, illustrative of a larger real internal library.

    **The retriever's real output** — `search_relevant_tools(TASK_QUESTION)` on the fixed keyword-overlap scorer: `["lookup_employee", "check_license_availability", "provision_access"]`, exactly the 3 tools the task needs, filtered from 25 down to 3 before the model ever saw the rest.

    **The model's real final answer**, tool-search condition: *"All done! Here's a summary: Employee: Jamie Cole (EMP-4471), Engineering department. License check: 12 seats available for DataViz Pro in Engineering. Result: Access successfully provisioned. Confirmation ID: `prov_EMP-4471_DataVizPro`. Jamie Cole now has access to DataViz Pro."* Correct, verified against the fixture's independently known ground truth (12 seats available, so provisioning should succeed).

    **Real cost**: 4 calls, 4,065 tokens — versus the naive condition's 4 calls, 9,011 tokens for the identical task and identical outcome. Same number of round-trips; the savings came entirely from not paying for 22 irrelevant tool definitions on every turn.

## A real bug this recipe caught before spending a paid call

The tool-search condition depends on a retriever picking the right tools *before* the model ever sees the question — get that wrong, and the model gets handed a broken tool set no amount of reasoning can fix. The first version of `search_relevant_tools` scored every tool by raw word-overlap between the task question and each tool's description. Dry-tested with zero API calls, it returned `['check_printer_status', 'check_license_availability', 'check_ticket_status']` — only 1 of the 3 tools the task actually needed, with `lookup_employee` and `provision_access` (both required to complete the task) excluded entirely.

The cause: the task question contains the words *"check"* and *"whether"*, and so do several filler tools' descriptions — *"Check the status of a support ticket"*, *"Check whether a printer is online"* — so those filler tools scored real overlap on words that have nothing to do with the task, while the real tools' hand-curated keyword sets (`{"employee", "lookup", "department", "id"}` and similar) didn't happen to share as many raw words with the question's exact phrasing. The fix: filter a small set of common words — including "check" and "status" specifically, since they were the ones causing the false matches — out of both the question and the fallback description-word scoring before computing overlap. Re-run with zero API calls, the retriever returned exactly the 3 real tools, confirmed before condition B ever made a real call.

This is worth stating plainly because it's a genuine illustration of a real risk with hand-built retrieval: a keyword filter that looks reasonable in isolation can fail silently on exactly the words that seem most on-topic, and the failure mode isn't a crash — it's a tool set that structurally cannot complete the task, with nothing in the API response to flag that anything went wrong. A production system would use Anthropic's actual Tool Search Tool (a real search index, not hand-tuned keywords) or a proper embedding retriever precisely to avoid this class of bug — this recipe's minimal keyword version is illustrative of the *mechanism* (pre-filter before the model sees the rest), not a recommendation to ship keyword overlap as a retriever.

## Technique 2: programmatic tool calling

Where tool search reduces *which* tool definitions the model sees, programmatic tool calling changes what happens *after* a tool is called. Instead of one tool call per conversational turn — with each tool's result posted back into the transcript as a separate message the model has to read before deciding the next step — the model writes one program that calls several tools in sequence, computes with their results, and prints only the final thing it wants back. Anthropic's own description of the mechanism: the model provides code, *"which is processed in the Code Execution environment rather than Claude's context"* — intermediate values never become separate turns at all.

The real measured effect: *"Average usage dropped from 43,588 to 27,297 tokens, a 37% reduction on complex research tasks."*

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/tools-at-scale/tools_at_scale_docs.py:programmatic-tool"
```

!!! success "A real run — the model's actual generated code, and the transcript answer"
    Given the identical task question and a single `execute_workflow` tool (its description exposes the three real Python functions as a mini API: `lookup_employee(employee_id) -> dict`, `check_license_availability(software, department) -> dict`, `provision_access(employee_id, software) -> dict`), the model's real final answer: *"Here's the summary for EMP-4471: Employee: Jamie Cole, Department: Engineering. License Check: 12 seats available for DataViz Pro in Engineering. Provisioning: Successfully provisioned. Status: `provisioned`. Confirmation ID: `prov_EMP-4471_DataVizPro`."* Correct, matching the same ground truth as both other conditions.

    **Real cost**: 3 calls, 3,743 tokens — one fewer call than either the naive or tool-search condition (4 each), because the model chained all three real tool calls inside a single generated program instead of one call per tool. The intermediate `lookup_employee` and `check_license_availability` results never became separate conversation turns; only the final summary the model chose to print came back into context.

## Technique 3: code execution with MCP

The same idea, pushed to where it matters most for tool libraries that are themselves large and MCP-based: instead of exposing every MCP tool's schema directly to the model, present the whole MCP server as a code API the model can import and call from generated code. Anthropic's framing: *"a solution is to present MCP servers as code APIs rather than direct tool calls."* The cited real case: *"This reduces the token usage from 150,000 tokens to 2,000 tokens—a time and cost saving of 98.7%."*

The privacy implication is a genuine second benefit, not just a token-count one: *"intermediate results stay in the execution environment by default. This way, the agent only sees what you explicitly log or return, meaning data you don't wish to share with the model can flow through your workflow without ever entering the model's context."* A workflow that reads a full customer record to extract one field never has to put the full record in front of the model at all — only the one field the code chooses to return does.

This recipe's `execute_workflow` tool (used in the programmatic tool calling demo above) is a minimal version of exactly this pattern — three plain Python functions exposed through one code-execution tool, rather than an MCP server, but the mechanism is the same: the model's generated code decides what crosses back into context, and everything else — the full employee record, the raw seat count — stays inside the sandboxed execution and is discarded once the program returns.

## What this means in practice

All three techniques attack the same root cause — every tool definition and every intermediate tool result is context, and unfiltered context is either tokens paid for nothing or, per Context Engineering's confusion failure mode, a real source of degraded tool selection — but they attack it at different points: tool search controls *which definitions* load; programmatic tool calling and code execution with MCP control *what results* make it back into the transcript at all. A production agent with a genuinely large tool library (dozens of MCP servers, hundreds of tools) benefits from combining both: defer-load definitions so the model only ever sees the handful relevant to the current task, and route multi-step tool chains through generated code so intermediate results stay out of context by default rather than by discipline.

The scale in this recipe (25 tools, a 3-step task) is deliberately small and cheap to run — the real Anthropic numbers cited above (85% reduction, 150,000 → 2,000 tokens) come from libraries and workflows far larger than what a demo can afford to run repeatedly. The recipe's own real numbers (-55%, -58%) show the same *direction* and the same *mechanism* at a scale a reader can actually reproduce for a few cents, not a claim to match Anthropic's reported magnitude.

## Interview angle

**Weak answer** to "how would you scale an agent from 5 tools to 500?": *"I'd use RAG to retrieve the relevant tools before each call."* This names a real technique (tool search is exactly this) but stops at the first bottleneck — it doesn't address what happens *after* tools are called, which is where this page's own recipe found the larger of its two savings (programmatic tool calling saved a full call, not just tokens, by not echoing intermediate results back as separate turns).

**Strong answer**: two separate bottlenecks scale independently. The first is definition bloat — every tool schema in context on every turn, regardless of use — solved by deferred loading / search, cutting Anthropic's cited case by 85% and improving accuracy (49%→74% on Opus 4) because a shorter, more relevant tool set is also easier to select correctly from. The second is result bloat — every intermediate tool result becoming its own conversational turn — solved by programmatic tool calling or code execution with MCP, where the model writes code that chains calls and controls what actually returns to context, cutting a cited case by 98.7% and, as a real second benefit, keeping data the model doesn't need to see (a full customer record, when only one field is needed) out of context entirely rather than relying on the model to ignore it.

**Follow-up to expect**: "why would a keyword-based retriever specifically be a bad idea in production?" Because it fails silently on exactly the words that look most relevant — this page's own recipe hit this directly: a raw-overlap retriever excluded 2 of 3 required tools because filler tools shared generic words ("check", "status") with the task question, and there was no error, no exception, nothing in the API response to signal the tool set was broken — just a model quietly handed an incomplete toolkit. A real retriever (Anthropic's Tool Search Tool, or a proper embedding index) is worth the engineering cost specifically because that failure mode is invisible until something downstream breaks.

## Build it yourself — 30 minutes

1. Take an agent you already have (or the tool-calling loop from Agent Loop From Scratch) and add 10-20 unrelated filler tool definitions to its `tools=` list — enough to notice a real token cost, without needing a genuinely large library.
2. Measure real `response.usage.input_tokens + output_tokens` for a task, with all tools present versus with a hand-filtered subset containing only the tools the task needs. Compare the real numbers, not an assumed savings.
3. Pick one multi-step task that currently makes 3+ separate tool calls in a loop. Write a single tool whose description exposes the underlying functions as a mini code API (see this recipe's `execute_workflow`), and let the model write one program instead. Compare real call counts and real token counts against the original loop.
4. If you build a keyword or embedding retriever for step 2, dry-test it with zero API calls first — print what it actually returns for your real task question, and check by hand that every tool the task structurally needs is in the result, the way this recipe's own bug was caught before any paid call.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Tools at Scale">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real recipe measured naive (all 25 tool definitions in context) at 4 calls / 9,011 tokens, and a tool-search condition (pre-filtered to 3 relevant tools) at 4 calls / 4,065 tokens for the identical task and identical correct outcome.",
      "question": "Both conditions took the same number of calls. What does that say about where the token savings actually came from?",
      "options": [
        "The savings figure is likely a measurement error, since call count didn't change too",
        "The savings came from the tool-search condition skipping the license-availability check",
        "The savings came from the model reasoning less carefully once fewer tools were shown",
        "The savings came from not paying for the 22 irrelevant tool definitions on every turn"
      ],
      "correct": 3,
      "explanations": [
        "Unsupported -- a real token-usage field from the API (input_tokens + output_tokens) isn't a noisy estimate; a large, consistent gap across a fixed call count is exactly the kind of real signal that measurement would reliably show.",
        "Contradicts the recipe's own reported real answer directly -- both conditions' final answers explicitly confirm '12 seats available' and successful provisioning; neither skipped a step.",
        "Nothing in the real transcripts supports reduced reasoning -- both conditions reached the identical correct outcome (provisioned, same confirmation ID), which is inconsistent with a model reasoning less carefully in one of them.",
        "Correct. With an identical call count, the only remaining source of the token difference is what's paid for on each of those calls -- the naive condition pays for all 25 tool definitions every turn; the filtered condition pays for only 3. The savings is definitional bloat, not fewer round-trips."
      ]
    },
    {
      "scenario": "This recipe's programmatic tool-calling condition finished in 3 calls / 3,743 tokens, one fewer call than both the naive and tool-search conditions (4 calls each) for the same task.",
      "question": "What's the most accurate explanation for why programmatic tool calling saved a full call, not just tokens?",
      "options": [
        "Intermediate tool results were chained inside one program instead of separate turns",
        "The execute_workflow tool has a larger max_tokens budget than the other two conditions",
        "Sonnet inherently requires fewer calls than Haiku regardless of tool structure used",
        "The model skipped verifying license availability before provisioning access"
      ],
      "correct": 0,
      "explanations": [
        "Correct. In the naive and tool-search loops, each of the three real tool calls becomes its own conversational turn (a separate model call to process each result and decide the next step). Programmatic tool calling lets the model write one program that calls all three functions internally and prints only the final result, collapsing what would be several turns of results-processing into a single call.",
        "A red herring -- max_tokens caps the LENGTH of a single response, it doesn't reduce how many calls a multi-step tool-use loop needs; it isn't the mechanism behind fewer calls here.",
        "This recipe used the same model (Sonnet) across all three conditions -- the call-count difference is explained by tool-calling structure, not a model comparison that wasn't actually run.",
        "Contradicts the real transcript directly -- the programmatic condition's final answer explicitly states 'License Check: 12 seats available,' confirming the check ran; nothing was skipped."
      ]
    },
    {
      "scenario": "A team hand-builds a keyword-overlap retriever for tool search. It's dry-tested once against a sample question, returns a plausible-looking set of 3 tools, and ships. Months later, a task fails because the retriever silently excluded a genuinely required tool for a differently-worded question.",
      "question": "What does this page's own real bug (the retriever initially returning 2 wrong tools for its own task question) suggest was the actual risk here?",
      "options": [
        "The failure only happens with fictional or illustrative tool names, not real ones",
        "A hand-picked keyword retriever can fail silently on generic words shared with filler",
        "The team's mistake was not adding more filler tools to test against before shipping",
        "Keyword retrievers always fail eventually, so only embedding search should ever be used"
      ],
      "correct": 1,
      "explanations": [
        "Unsupported and arbitrary -- nothing about the failure mechanism (word-overlap scoring against generic shared vocabulary) is specific to fictional versus real tool names; the same scoring logic would behave identically either way.",
        "Correct. The recipe's own real bug was caused by generic words ('check', 'status') appearing in both the task question and several filler tool descriptions, scoring those filler tools higher than the real tools' more specific keyword sets -- with no exception or error signal that anything was wrong, exactly the kind of silent failure the scenario describes happening later, in production.",
        "Misdiagnoses the fix -- the recipe already had 22 filler tools present when the bug occurred; adding more wouldn't have surfaced the mechanism (generic-word overlap with the real tools' curated keyword sets), which was the actual cause, not insufficient test volume.",
        "Overstates the claim -- this page doesn't argue keyword retrieval always fails, only that it's fragile in a specific, demonstrated way (generic word overlap); a well-designed keyword system with stopword handling, like the one this recipe ended up shipping, worked correctly for its test case."
      ]
    },
    {
      "scenario": "Anthropic's code-execution-with-MCP article states that intermediate results 'stay in the execution environment by default' and 'the agent only sees what you explicitly log or return.'",
      "question": "Beyond the token savings, what real second benefit does this description point to?",
      "options": [
        "It removes the need for any max_tokens limit on the model's own responses",
        "It guarantees the generated code itself is always free of bugs or errors",
        "Data the workflow doesn't want shared with the model can stay out of context",
        "It lets the model skip calling tools whose results aren't immediately needed"
      ],
      "correct": 2,
      "explanations": [
        "Unrelated -- max_tokens governs the length of a model's own generated response; it isn't affected by where intermediate tool results are stored.",
        "Not what the quoted mechanism does or claims -- code execution changes what data reaches the model's context, it has no bearing on whether the generated code itself is correct or error-free.",
        "Correct. As the article states directly, this means 'data you don't wish to share with the model can flow through your workflow without ever entering the model's context' -- a genuine privacy/minimization benefit distinct from the token-count savings, e.g. a full customer record can be processed in code while only one needed field is ever returned to the model.",
        "Misreads the mechanism -- code execution doesn't let a model skip calls it needs; it changes what happens to a call's result afterward (stays in the execution environment unless explicitly returned), not whether the call happens."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Advanced tool use"](https://www.anthropic.com/engineering/advanced-tool-use) — Tool Search Tool, `defer_loading`, the 85% token reduction, and the Opus 4 (49%→74%) / Opus 4.5 (79.5%→88.1%) MCP evaluation accuracy improvements; Programmatic Tool Calling and its 43,588→27,297 token, 37% reduction measurement.
- Anthropic, ["Code execution with MCP: Building more efficient AI agents"](https://www.anthropic.com/engineering/code-execution-with-mcp) — presenting MCP servers as code APIs, the 150,000→2,000 token (98.7%) reduction, and intermediate results staying in the execution environment by default.
- [Tool Design](tool-design.md) — the earlier page on this site covering why more tools in context also degrades tool-selection accuracy, not just cost.
- [Context Engineering](context-engineering.md) — Breunig's context confusion failure mode, directly related to why unfiltered large tool libraries hurt accuracy, not just token count.
