# Revision

Every page's TL;DR in one place, every page's Scenario Check merged into one combined quiz, and a flashcard deck for fast recall drilling — for a quick pass before an interview instead of clicking through pages one at a time.

!!! note "Kept in sync manually"
    This page is generated from the TL;DR and Scenario Check blocks on every content page by `scripts/generate_revision.py`, re-run after editing any page's TL;DR or quiz — it isn't live-generated on every build.

=== "All TL;DRs"

    ## Foundations

    ### [What Is an Agent?](what-is-an-agent.md)

    - **The defining property isn't "uses an LLM."** Anthropic's working definition: a **workflow** is a system where "LLMs and tools are orchestrated through predefined code paths"; an **agent** is a system where "LLMs dynamically direct their own processes and tool usage." The line is *who decides the next step* — fixed code, or the model.
    - **A retry-on-failure loop written in application code is still a workflow, not an agent**, under this definition — Anthropic's own evaluator-optimizer pattern (generate, grade, loop until a criterion is met) is listed as a *workflow*, because the LLM isn't the one deciding whether or how to retry.
    - **Definitions genuinely disagree at the edges.** Simon Willison: "an LLM agent runs tools in a loop to achieve a goal." Chip Huyen: something that perceives and acts on an environment. OpenAI: "systems that independently accomplish tasks on your behalf." None of these are wrong; they draw the line in slightly different places, and a strong interview answer names the disagreement rather than picking one as *the* definition.
    - **Most systems marketed as agents aren't, by any of these definitions.** Menlo Ventures' December 2025 enterprise survey found only 16% of enterprise and 27% of startup deployments calling themselves "agents" actually qualify as one — most are "if-then logic around a model call."
    - **A worked trace beats a definition in an interview.** This page's own demo shows the same misclassification happening in both a workflow and an agent — the difference isn't that the agent classifies better, it's that the agent has a step where it can act on its own doubt, and the workflow structurally doesn't.

    ### [The Agent Loop From Scratch](agent-loop-from-scratch.md)

    - **The model never calls your function.** At inference time it's given a schema and generates a *structured request* — a `tool_use` block naming a tool and its arguments. Your orchestration code parses that, runs the real function, and sends the result back as a new message (a `tool_result`) on the next turn. All the "agentic" behavior here is: structured generation, external execution, and a feedback loop — nothing more magical than that.
    - **The model has no guaranteed connection to reality until your code creates one.** It can request a tool that doesn't exist, or arguments that don't validate — there's no built-in check that its output matches the schema until your orchestration code checks.
    - **Tool descriptions are a prompt, not documentation.** Vague parameter descriptions produce vague argument extraction; concrete descriptions with worked examples measurably improve it. This is a design surface, not an afterthought.
    - **A hard iteration cap bounds round-trips, not per-call latency.** They're separate risks and need separate limits — a loop that can't run away forever can still stall for a long time on one slow tool call.
    - **A real run here produced a genuinely counter-intuitive result**: the no-tools baseline got every arithmetic step right on its own — what it actually lacked was a live exchange rate, which isn't an arithmetic problem at all.

    ### [Workflow Patterns](workflow-patterns.md)

    - **Anthropic names five workflow patterns** — chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer — and every one of them stays a workflow under the [What Is an Agent?](what-is-an-agent.md) definition: the control flow is fixed by code, even where an LLM call is involved in a decision along the way.
    - **Parallelization has two distinct shapes.** *Sectioning* splits one task into a fixed, developer-chosen set of independent subtasks. *Voting* runs the same task multiple times and aggregates. This page demos sectioning.
    - **Orchestrator-workers is the pattern most often confused with parallelization**, and the difference isn't concurrency — it's whether the set of subtasks is fixed by code ahead of time (parallelization) or decided by a planning call at runtime, per input (orchestrator-workers).
    - **A real run measured parallelization's actual value**: the same 3 review calls took 4.79 seconds run sequentially and 1.86 seconds run concurrently — a real ~2.6x speedup, not the theoretical 3x, which is the honest number once real network and queueing overhead are in it.
    - **A real run also showed orchestrator-workers doing what it's supposed to**: the same planner, given two different topics, produced two genuinely differently-*shaped* breakdowns — a comparison-structured split for one topic, a derivation-structured split for the other — with nothing in the code telling it which shape to use.

    ### [Reasoning Paradigms](reasoning-paradigms.md)

    - **These are a different axis from [workflow patterns](workflow-patterns.md).** Workflow patterns fix the *topology* (which steps exist, in what order). Reasoning paradigms change how a *single step or loop* reasons, searches, or recovers from its own mistakes — they're about the shape of thinking, not the shape of the pipeline.
    - **Plan-and-Solve is a zero-shot prompting technique, not a planner/executor architecture** — it's one prompt ("first devise a plan, then carry it out") compared against plain "let's think step by step." Don't confuse it with LangChain's separately-named Plan-and-Execute agent, which actually is an architecture.
    - **ReWOO's real, verified advantage is token and call efficiency, not latency** — the paper never claims a latency win. A real measured run on this page needed 2.5x the calls and 6.8x the tokens for ReAct versus ReWOO on the identical question and model.
    - **ReWOO's plan-then-execute split has a real failure surface ReAct doesn't**: a bad step in the plan is only caught once it's actually executed, not before. This page's real run hit exactly that — a generated plan referenced a function its tool didn't support, and one step genuinely failed.
    - **Reasoning models changed what needs to be built by hand.** DeepSeek-R1's reported finding is that self-reflection and verification can emerge from pure reinforcement learning, without anyone hand-coding a Reflexion-style retry loop or a ToT-style external search — some of what these 2023 papers built as scaffolding, later training runs learned to do internally.

    ### [Choosing a Framework](choosing-a-framework.md)

    - **A real, identical task solved three ways shows the actual trade-off, not a feature checklist.** The same tool-calling question, same model, same correct answer, same real call count (2) across the raw Anthropic client, LangChain/LangGraph's `create_agent`, and Pydantic AI's `Agent` — but real, measured orchestration code of 15, 8, and 5 lines respectively. The gap is what each approach makes you write by hand versus adopt as-is.
    - **"Own the loop" is a real, named position, not just a vibe.** 12-Factor Agents' framing: most products calling themselves agentic "aren't that agentic" — real production reliability, in this view, comes from owning your own control flow rather than delegating it to a framework's abstraction.
    - **AutoGen is in maintenance mode, by its own README's words**: "It will not receive new features or enhancements and is community managed going forward. New users should start with Microsoft Agent Framework." Building on it today means building on something the vendor itself says to migrate away from.
    - **A real dependency conflict surfaced and got fixed for real, not glossed over**: the full `pydantic-ai` package pulls in `fastmcp-slim`, which requires `python-dotenv>=1.1.0` — directly conflicting with this cookbook's shared `python-dotenv==1.0.1` pin. `pydantic-ai-slim[anthropic]` (skipping the unneeded `mcp` extra) resolves it cleanly, since this recipe never uses MCP.
    - **Framework adoption in real job descriptions is smaller than the discourse suggests.** Across 1,978 genuinely AI-technical job postings, LangChain appears in 8.1% (28 companies) and LangGraph in 5.0% (19 companies) — real, current numbers, not zero, but a small fraction of postings even in a corpus specifically filtered for AI-technical roles.

    ## Tools and Context

    ### [Tool Design](tool-design.md)

    - **A tool description is a prompt, not documentation** — it steers behavior the same way a system prompt does. Anthropic reports that precise tool-description refinements (nothing else) took Claude Sonnet 3.5 to state-of-the-art on SWE-bench Verified.
    - **Consolidated, purpose-built tools beat many granular endpoint wrappers on both calls and tokens, measured**: on the identical question, a single `get_blocked_tasks_with_reasons` tool took 2 calls and 1,563 tokens; exposing `list_tasks` + `list_comments` separately and letting the model compose them took 3 calls and 3,079 tokens.
    - **Code execution's real advantage didn't show up as a clean win on one fixed question** — it tied endpoint wrappers on call count (3 each) here, though it used fewer tokens (2,951 vs. 3,079). Its actual case is avoiding a combinatorial explosion of purpose-built tools across many *different* possible query shapes, not raw efficiency on one you already anticipated.
    - **A real error-message comparison's finding wasn't about retry count — both a terse and an actionable validation error took the model exactly 2 calls to recover.** What differed was the value it recovered *to*: the terse error (no information) got a generic, disconnected guess; the actionable error (stating the valid options) got a genuinely closer match to what the user actually meant.
    - **Interface design measurably matters independent of the underlying model** — SWE-agent's own contribution wasn't a better model, it was a custom Agent-Computer Interface, and the paper reports this got a non-interactive baseline's pass@1 up to 12.5% on SWE-bench.

    ### [Context Engineering](context-engineering.md)

    - **Context engineering is curating the optimal set of tokens for each inference call**, not writing one good prompt once. Anthropic's framing: models have an "attention budget" — a real constraint from the transformer's n² pairwise token relationships — and **context rot** is what happens as that budget gets spent on tokens that don't help.
    - **Four verbs cover most of what you actually do**: write context (save it outside the window), select context (pull the right thing back in), compress context (keep only what's needed), isolate context (split it up) — LangChain's framework, and Breunig's specific fixes (RAG, tool loadout, quarantine, pruning, summarization, offloading) are each an instance of one of these four.
    - **A real repro of context poisoning worked exactly as the theory predicts**: a hallucinated fact (a fake founding year) in context produced a wrong downstream answer (17 years old instead of 12); quarantining the bad turn — not just adding a correction, replacing it — fixed it cleanly.
    - **A real repro of context distraction produced a genuine failure, but not the hypothesized one** — six turns of a consistently wrong pattern in context didn't make the model mechanically repeat that exact pattern; it produced a *different* wrong answer (the raw, undivided sum) in the same terse style the flawed history modeled. Reported honestly rather than smoothed into matching the prediction.
    - **Real repros of confusion and clash did not reproduce as failures in this run** — both honest negative results, with real, disclosed reasons why (a 12-tool test below the literature's reported ~30-tool confusion threshold; a clash repro that gave the model an explicit "supersedes" cue, making it resolvable rather than genuinely ambiguous).

    ### [Tools at Scale](tools-at-scale.md)

    - **A large tool library isn't free just because a model can technically pick the right tool from it.** Every tool definition sits in context on every turn, whether or not it's used — Anthropic's Tool Search Tool defers loading full definitions until they're actually needed, cutting token usage by 85% while keeping the full library reachable, and lifting Opus 4's MCP evaluation accuracy from 49% to 74% (Opus 4.5: 79.5% to 88.1%).
    - **Programmatic tool calling changes what enters context, not just how much**: the model writes one program that calls multiple tools and controls what actually gets returned, instead of one tool call per turn with every intermediate result echoed back. Anthropic's own measurement: 43,588 → 27,297 tokens, a 37% reduction on complex research tasks.
    - **Code execution with MCP takes the same idea further**: presenting MCP servers as code APIs instead of direct tool calls, so intermediate results "stay in the execution environment by default" and "the agent only sees what you explicitly log or return" — Anthropic's cited case: 150,000 → 2,000 tokens, a 98.7% reduction.
    - **A real, minimal repro of all three reproduced the shape of these effects at small scale**: naive (25 tools in context) cost 9,011 tokens over 4 calls; a keyword-filtered tool-search condition cost 4,065 tokens over the same 4 calls (-55%, from not paying for 22 irrelevant tool definitions); programmatic tool calling cost 3,743 tokens over just 3 calls (-58%, from not echoing intermediate results back as separate turns).
    - **A real bug in the tool-search condition's retriever was caught before any paid calls**: raw keyword overlap let generic words ("check", "status") shared between the task question and filler tool descriptions outscore the real tools' more specific keyword sets, excluding 2 of the 3 tools the task actually needed. Fixed by filtering common words from both sides before scoring — verified with a zero-cost dry run before spending a single real API call.

    ### [MCP Deep Dive](mcp-deep-dive.md)

    - **MCP's 2026-07-28 revision makes the protocol stateless at the wire level**: the `initialize`/`notifications/initialized` handshake and the `Mcp-Session-Id` header are removed from Streamable HTTP; every request carries its own protocol version and capabilities; list endpoints no longer vary per connection. A real, dated finding: `mcp==2.2.0` (the SDK that explicitly targets this spec) still uses `Mcp-Session-Id` **by default** — spec-compliant statelessness is real and working, but is an opt-in flag (`stateless_http=True`), not the default.
    - **Multi Round-Trip Requests (MRTR) replaced server-initiated requests** (`roots/list`, `sampling/createMessage`, `elicitation/create`) with a request/retry pattern: a server returns `InputRequiredResult`, the client retries the original request carrying the answer plus an opaque `requestState` token the server minted and must re-verify.
    - **A real repro of the SDK's own `requestState` security held on every check**: tampering with a sealed token is rejected (AEAD authentication failure), replaying a token against a different tool argument is rejected (request-binding), and — the important one — replaying one user's token as a different user is rejected (principal-binding). That last check is the real, working mitigation for the spec's own named "State Handle Hijacking" vulnerability: *"MCP servers **MUST NOT** treat possession of a state handle as authentication."*
    - **A real, current list of deprecations matters for anything built today**: HTTP+SSE transport (migrate to Streamable HTTP), Roots/Sampling/Logging features (migrate to tool parameters, direct provider APIs, and OpenTelemetry respectively), and OAuth Dynamic Client Registration (migrate to Client ID Metadata Documents) are all now formally Deprecated under a twelve-month removal window, not just "discouraged."
    - **Token passthrough is explicitly forbidden, not just risky**: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server"* — a server that blindly forwards a client-supplied token downstream breaks a real OAuth security boundary and reintroduces the confused-deputy problem the rest of the spec's auth model is built to prevent.

    ### [MCP and Tool Ecosystem](mcp-tool-ecosystem.md)

    - **A real, live, official MCP registry exists and is publicly callable right now**: `registry.modelcontextprotocol.io`, launched in preview 2025-09-08, backed by Anthropic, GitHub, PulseMCP, Block, and others — described in its own launch post as "an open catalog and API for publicly available MCP servers." Unauthenticated reads via `GET /v0/servers?search=...`. A real repro called this exact live API.
    - **The real repro found the registry's search is closer to keyword/brand matching than semantic search.** Natural-language capability queries (`"browser automation form filling"`, `"payment charges status"`) reliably returned **zero results**; single-keyword, brand-style queries (`"Stripe"`, `"playwright"`, `"browser"`) worked. An agent had to burn 6 real search attempts, 4 of them dead ends, before finding a usable browser-automation server.
    - **A real, separately-confirmed finding: even a genuinely popular server can be absent.** Microsoft's own official `playwright-mcp` (37.5k+ GitHub stars) does not currently appear in the registry under any query tried, including a direct `search=microsoft`. As a real preview-stage product, the registry's population is honest but incomplete.
    - **Gateways, remote-hosted servers, and tool platforms are the real, separate layers that exist because raw discovery and raw protocol aren't enough on their own**: gateways add auth/rate-limiting/routing in front of many servers; companies like Stripe, GitHub, Notion, and Cloudflare now host their own official *remote* MCP servers (reachable over HTTP, not run locally); and platforms like Composio aggregate hundreds of third-party integrations behind one MCP interface — each solving a real, different gap the base protocol and the registry alone leave open.

    ### [Skills, A2A and Other Protocols](skills-a2a-protocols.md)

    - **Agent Skills load in real, verified stages, not all at once.** Anthropic's own docs: *"until a Skill is triggered, only its name and description occupy context"* — roughly **~100 tokens** per skill at Level 1 (name + description, always loaded), the full body loaded only when triggered (Level 2, *"under 5k tokens"*), and bundled scripts/resources loaded only as referenced, where *"the script code itself never enters context."* A real repro confirmed this directly: loading three skills' full bodies upfront cost **4,975 real tokens** on a task needing only one; progressive disclosure cost **3,069** — a real **38.3% reduction** — and correctly never touched the two irrelevant skills at all.
    - **A2A (Agent2Agent) standardizes agent-to-agent interoperability**, not agent-to-tool — Google announced it April 2025 with 50+ launch partners, and it's since moved to Linux Foundation governance: *"A2A will remain vendor neutral, emphasize inclusive contributions and continue the protocol's focus on extensibility, security and real-world usability."* Its core discovery mechanism is the **Agent Card** — a JSON document *"describing the server's identity, capabilities, skills, service endpoint URL, and how clients should authenticate and interact with it,"* typically served at `.well-known/agent-card.json`.
    - **AG-UI standardizes the agent-to-user-interface layer** — real, verified: *"an open, lightweight, event-based protocol that standardizes how AI agents connect to user-facing applications,"* created by CopilotKit in partnership with LangChain and CrewAI. Three distinct layers, three distinct protocols: MCP (agent↔tools), A2A (agent↔agent), AG-UI (agent↔user interface).
    - **The Agent Client Protocol (ACP) is coding agents' answer to LSP** — real, verified, explicit analogy from its own docs: *"ACP solves this by providing a standardized protocol for agent-editor communication, similar to how the Language Server Protocol (LSP) standardized language server integration."* Created by Zed (August 2025), now community-governed. **AGENTS.md** is the same idea for repo-level instructions — real, verified: *"a README for agents,"* now used by *"over 60k open-source projects"* and stewarded by the Linux Foundation.

    ### [KV-Cache Economics](kv-cache-economics.md)

    - **Prompt caching isn't a flat discount — it's a hierarchy with a precise cost structure.** A 5-minute cache write costs 1.25x base input price; a cache read costs 0.1x (cheaper still — 0.025x–0.05x — on some model families). The cache follows a strict prefix order, `tools` → `system` → `messages`, and a change at any level invalidates that level *and everything after it*.
    - **A real run confirmed the hierarchy precisely**: changing `tool_choice` between calls — same tools, same system prompt — left the cached tools+system prefix fully intact (`cache_read_input_tokens` unchanged). Editing one word in one tool's description invalidated the entire cache and forced a full, fresh write.
    - **"Tool masking vs. removal" is a real, current, shipped feature, not just a conceptual pattern**: the `mid-conversation-tool-changes` beta's `tool_removal` content block withdraws a tool from a running conversation while the top-level `tools` array stays byte-identical — so the cache survives. Physically editing the `tools` array to drop a tool produces a different array and a fresh cache entry, every time.
    - **A real repro measured the contrast directly**: masking a tool (three calls in a row, including one a full turn later) kept reading the identical 2,211-token cache entry. Removing the same tool by editing the array instead produced a new, distinct 2,113-token entry with no relationship to what came before.
    - **Masking is enforced, not cosmetic**: forcing `tool_choice` to the masked tool by name produced a real API error — *"forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back)"* — the model genuinely cannot call it, not just a description change that happens not to mention it.

    ### [Memory Architectures](memory-architectures.md)

    - **Agent memory isn't one thing — CoALA's four-type taxonomy names distinct kinds with distinct jobs**: working memory ("active and readily available information... for the current decision cycle"), episodic memory ("experience from earlier decision cycles"), semantic memory ("an agent's knowledge about the world and itself"), and procedural memory (skills, split between "implicit knowledge stored in the LLM weights" and "explicit knowledge written in the agent's code").
    - **A real repro confirmed the semantic/episodic split as actual model behavior, not just a taxonomy exercise**: given one customer-service case, Claude's real memory tool unprompted wrote a general policy to one file and the specific case's outcome to a separate file — then correctly answered a later semantic-only question and a separate episodic-only question, each from the right file, with no instruction naming the taxonomy at all.
    - **MemGPT's real, cited numbers show what happens without any of this**: GPT-4 with MemGPT's OS-inspired paging hit 92.5% on Deep Memory Retrieval versus a 32.1% fixed-context baseline — the gap a taxonomy-free, everything-in-context approach pays for.
    - **Mem0's real, cited numbers show what a lighter-weight, extraction-based memory layer buys**: 26% relative improvement over OpenAI's own memory feature, 91% lower p95 latency, and over 90% token savings versus stuffing full conversation history into context every turn.
    - **A real memory-poisoning repro found the vulnerability isn't "any injected claim" — it's specifically the claims that don't feel consequential.** A crafted turn injecting an unverified $50,000 financial claim triggered spontaneous skepticism with zero mitigation in place. The identical mechanism against a routine-sounding claim (a support contact reassignment) was trusted with zero hedging — and a single source-tagging instruction was enough to close that gap.

    ### [Models for Agents](models-for-agents.md)

    - **"Small model = unreliable at reasoning" is a real, but no longer automatic, assumption — it needs checking per task.** A real repro found Claude Haiku 4.5 matched Sonnet 5 at 100% accuracy on a deliberately ambiguous tool-calling request, and separately solved three classic reasoning traps (the widgets/machines lateral-thinking problem, the bat-and-ball cognitive-reflection-test problem, and a chickens-and-cows system-of-equations problem) on the first try, no retries.
    - **Open-weight models genuinely compete at the top of tool-calling leaderboards now, not just "catch up eventually."** As of a June 2026 snapshot of the Berkeley Function-Calling Leaderboard, GLM 4.5 (open-weight) led at 76.7% overall accuracy, ahead of Claude Opus 4.7 (76.6%) and Gemini 3.1 Flash Lite Preview (76.5%) — a real, current instance of an open-weight model at the top of a major agentic benchmark, not a footnote below it.
    - **RouteLLM's real, cited numbers show what model routing buys when accuracy differs between tiers**: over 2x cost reduction while holding response quality, with per-benchmark reductions reported around 85% on MT Bench, 45% on MMLU, and 35% on GSM8K at 95% of GPT-4's own performance level.
    - **A real repro found routing still pays off even when the cheap model would have gotten every answer right anyway** — a genuinely different, and arguably more useful, finding than "routing saves money by avoiding mistakes." Real numbers: always-Sonnet cost 216 tokens for 4/4 correct; routed cost 199 tokens for the same 4/4 — the router itself has real overhead, but escalating only the queries flagged as complex still beat blanket escalation on cost without giving up any accuracy.

    ## Systems

    ### [Planning and Decomposition](planning-and-decomposition.md)

    - **Plan-and-execute separates the "what should happen" decision from the "make it happen" work**: a planner produces a multi-step plan up front, and an executor carries it out — real, cited benefits over a single ReAct-style loop include speed (*"the larger agent doesn't need to be consulted after each action"*), cost (*"sub-tasks... can be made to smaller, domain-specific models"*), and completion quality (*"forcing the planner to explicitly 'think through' all the steps required"*).
    - **A real replanning repro was a clean, honest negative**: told either to execute its plan without second-guessing, or explicitly to revise on a broken assumption, Sonnet 5 produced the identical, correct outcome both times when a planned step's premise (a repo has a git tag) turned out false. Worth reporting plainly — the assumption that a rigid "don't second-guess" instruction would cause a real failure didn't hold up.
    - **A real granularity repro found the opposite — a genuine, reproducible failure, confirmed twice**: the same task, decomposed too coarse or well-sized, completed correctly both times (3 real tool calls, a correct changelog). Decomposed over-granular, the model spent its entire token budget writing out sub-steps and never called a single tool — a real, measurable cost of over-decomposition, not a hypothetical one.
    - **A minimal verifier-in-the-loop caught the failure using only the trace that already existed** — no second model call, no re-doing the work. Checking one concrete fact (was the final output-producing tool actually called?) correctly passed the two successful conditions and failed the over-granular one, with a real, specific reason attached.

    ### [Multi-Agent Systems](multi-agent-systems.md)

    - **A multi-agent *system* is not the same thing as orchestrator-workers.** [Workflow Patterns](workflow-patterns.md)' orchestrator-workers is a fixed topology — Anthropic classifies it as a *workflow*. A multi-agent system is what Anthropic calls an *agent* — a lead agent that delegates to subagents that are themselves autonomous, operating in parallel with their own judgment, their own tool calls, and their own context windows.
    - **A real measured run put a real number on Anthropic's token-cost claim**: single agent answering three independent questions, 1 call, 1,048 tokens; a lead agent dispatching three subagents plus a synthesis call, 4 calls, 3,369 tokens — a real **3.21x** multiplier, the same direction as (if smaller than) Anthropic's own reported 4x/15x figures.
    - **A real test of Cognition's inter-agent consistency risk did not reproduce the failure** — twice. Two subagents, each blind to the other's output, independently inventing a shared fact (a trial length) converged on the same value both times, even after the prompt was redesigned specifically to remove an obvious reason they'd default to the same answer. Reported as an honest negative result, with the real, disclosed reason why the repro's own design likely couldn't have shown the failure Cognition describes.
    - **MAST's real taxonomy**: 14 distinct failure modes across 3 categories — system design issues, inter-agent misalignment, task verification — built from 1,600+ annotated traces across 7 frameworks, and the paper's own stated finding is that multi-agent systems' "performance gains on popular benchmarks are often minimal."
    - **"When to use one agent" has a real, non-hand-wavy answer**: genuinely independent, breadth-first sub-tasks are where the token cost buys something real (Anthropic's own 90.2% improvement, largely attributable to spending more tokens); tightly-coupled tasks needing shared context between steps are where a single agent avoids a coordination problem it would otherwise have to solve by hand.

    ### [Harness Engineering](harness-engineering.md)

    - **A harness is everything around the model that makes a long-running agent actually work** — Anthropic's own framing: *"the system prompt, set of tools, and overall agent harness"* together, not the model alone. Their own real finding on why this matters: *"even a frontier coding model like Opus 4.5... will fall short... if it's only given a high-level prompt"* — the failure modes were one-shotting too much at once (context exhaustion mid-task) and prematurely declaring work complete.
    - **The documented fix is a specific artifact set, not a vague "give it more context" instruction**: an initializer session creates *"an `init.sh` script, a claude-progress.txt file that keeps a log of what agents have done, and an initial git commit"* — three concrete things a later, otherwise-blank session reads before doing anything.
    - **A real repro of this exact pattern worked end to end, and caught a real bug along the way.** A budget-limited first session made all its real lookups but was cut off before recording anything; a genuinely fresh second session — no memory of the first — read the shared progress file, re-derived the lost work from scratch, and correctly finished the task, flagging one deliberately ambiguous result as needing follow-up, unprompted.
    - **"Compaction isn't sufficient" on its own, per Anthropic's own real finding** — and a real bug in this page's own recipe demonstrated exactly the adjacent risk: treating "the response stopped" as "the agent finished" without checking *why* it stopped silently corrupted a state-tracking loop, mishandling a genuine token-budget truncation as if the agent had genuinely completed its turn.
    - **A real self-verification test — instructed discipline vs. none — was a clean, honest negative.** Anthropic's cited practice: *"Self-verify all features. Only mark features as 'passing' after careful testing."* Tested directly against a deliberately unhelpful real tool result, both the plain and the explicitly-instructed condition correctly declined to mark it "passing" — Sonnet 5's baseline judgment was already sufficient here.

    ### [Coding Agents: Mechanisms](coding-agents-mechanisms.md)

    - **Coding agents need their own interface, not a human one, repurposed** — SWE-agent's real framing: *"LM agents represent a new category of end users with their own needs and abilities, and would benefit from specially-built interfaces."* Their design principles are concrete, not vague: actions should be *"compact and efficient,"* feedback should be *"informative but concise,"* and *"guardrails mitigate error propagation and hasten recovery."*
    - **A real, cited number shows how load-bearing one specific guardrail was in 2024**: removing SWE-agent's linting guardrail (a syntax check fed back to the model after every edit) dropped its SWE-bench Lite score from 18.0% to 10.3% — a real 7.7 percentage point loss from removing one interface feature.
    - **A real repro of the same mechanism against Sonnet 5, across two independently redesigned task variants, found no measurable gap** — 100% valid syntax whether or not the guardrail was present, on both a flat elif edit and a genuinely harder nested-restructuring edit. A dated, honest finding, not a claim the guardrail is now universally unnecessary.
    - **A second real repro tested the review-loop / verifiable-task pattern directly**: a genuinely subtle binary-search off-by-one bug (confirmed to actually diverge via a 2,000-case random search, since hand-picked test cases initially missed it), fixed with and without a real, independently-run test suite available to check the fix. Another clean, honest tie — Sonnet 5 fixed the bug correctly on the first attempt, every time, with or without the ability to verify its own work.
    - **METR's real, cited trend gives the scale this all sits inside**: model *"time horizon"* — the length of task a model completes with 50% reliability — has grown with *"a doubling time of around 7 months"* over six years; Claude 3.7 Sonnet's measured horizon was *"approximately one hour."* This page's own repros are a small, current, honest data point on a specific slice of that trend: two ACI safety nets that mattered a lot in 2024 didn't move the needle for a 2026 model on small, illustrative tasks.

    ### [Coding Agent Products and Configuration](coding-agent-config.md)

    - **The coding-agent product landscape converges on similar mechanics from different starting points**: Codex CLI (OpenAI, Apache-2.0, local terminal), Cursor (a full IDE, not a CLI), GitHub Copilot (spans inline autocomplete to autonomous "agent mode," natively integrated into GitHub PRs/Issues), Cline (Apache-2.0, markets approval-gated execution — *"every file edit and terminal command requires your approval"* — as its core differentiator over full autonomy), Aider (Apache-2.0, best known for its *"repo map"* of the whole codebase, plus automatic git commits per change), OpenCode (MIT, a "build" agent and read-only "plan" agent switchable by Tab key), Gemini CLI (Google, Apache-2.0, built-in Search grounding), and OpenHands — which has genuinely repositioned from "an autonomous AI software engineer" to *"the self-hosted developer control center for coding agents and automations,"* explicitly orchestrating *"OpenHands, Claude Code, Codex, Gemini, or any agent with Agent-Client Protocol"* rather than just being one agent among many.
    - **AGENTS.md is a real, genuinely cross-tool open standard** — *"a simple, open format for guiding coding agents,"* described as *"a README for agents."* A long list of tools support it (Codex, Jules, Aider, opencode, Cursor, Copilot, and more); Claude Code added support for reading it directly as of v2.1.277.
    - **CLAUDE.md's real mechanics are more specific than "a config file"**: Anthropic's own docs state plainly, *"CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself,"* and — the load-bearing distinction this page's second repro tests — *"Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead."*
    - **A real repro of CLAUDE.md-style instructions found exactly the compliance shift the mechanism promises**: 0% `do_`-prefix compliance and inconsistent f-string avoidance with no instructions, 100% compliance on both with a project-instructions block delivered the real way — as a user message, not a system prompt — confirmed across multiple full runs.
    - **A real repro of hooks required a genuine design correction to test the right thing, and the corrected version found a clean, strong result**: with no protective rule anywhere and no hook, an explicit request to delete a file succeeded 5/5 trials. With a real, deterministic PreToolUse-style check and still no prompted rule, the same file survived 5/5 trials — the hook, not the model's judgment, is what actually held.

    ### [Deep Research Agents](deep-research-agents.md)

    - **Anthropic's own real numbers on their Research system**: a multi-agent system (Opus 4 lead, Sonnet 4 subagents) *"outperformed single-agent Claude Opus 4 by 90.2%"* on their internal eval — but *"token usage by itself explains 80% of the variance"* in performance, and *"agents typically use about 4× more tokens than chat interactions, and multi-agent systems use about 15× more tokens than chats."*
    - **A real, named failure mode, and a real, direct repro of it**: Anthropic's own documented finding — subagents *"performed the exact same searches as other agents... without an effective division of labor."* A real repro with 3 real subagents given identical, unscoped instructions produced **60% redundant document retrieval** and left 2 of 6 real documents completely uncovered.
    - **Anthropic's own prescribed fix, applied literally, worked**: *"Each subagent needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries."* The same real repro, with explicit per-subagent scopes instead, cut redundant retrieval to **14.3%** and covered the entire real document corpus.
    - **STORM's real mechanism generates better structure by simulating disagreement, not by asking once**: *"discovering diverse perspectives,"* then *"simulating conversations where writers carrying different perspectives pose questions to a topic expert,"* producing real, measured gains — *"more of STORM's articles are deemed to be organized (by a 25% absolute increase) and broad in coverage (by 10%)"* versus a single-pass outline-then-write baseline.
    - **BrowseComp is deliberately built to be easy to grade and hard to pass.** Real, verified numbers: GPT-4o scored **0.6%**, OpenAI o1 scored **9.9%**, and OpenAI's own Deep Research model scored **51.5%** — on a benchmark where, of 1,255 attempted questions, human researchers gave up on **70.8%** of them within two hours.

    ### [Computer-Use and Browser Agents](computer-use-browser-agents.md)

    - **Grounding — mapping "click the submit button" to real screen coordinates — is a real, named bottleneck, not a solved problem.** Real, verified: high-resolution professional interfaces create *"a 'needle-in-a-haystack' problem where target widgets may occupy less than 0.1% of the total screen area."* Real accuracy on ScreenSpot-Pro (professional/high-res UIs) ranges from Gemini-2.5-Pro's **6.96%** to UI-TARS-72B's **37.12%** to a specialized method's **73.18%** — grounding accuracy varies by over 10x depending on approach.
    - **OSWorld is deliberately, measurably hard.** Real, original numbers: **369 real tasks**, human performance **72.36%**, the best contemporary agent at launch **12.24%**. Real progress since has been fast but uneven — OpenAI's computer-use-preview reports **38.1%**, UI-TARS-2 reports **47.5%** — and a real follow-up study (OSWorld-Human) found that even accurate agents take **2.7–4.3x more steps than necessary** to complete a task.
    - **UI-TARS is a real, named architecture built specifically for this problem**: a single vision-language model unifying *"perception, reasoning, grounding, and memory,"* rather than a pipeline of separate components. Real, measured progress across its own versions: the original scored 24.6 on OSWorld; UI-TARS-1.5 reached 42.5%; UI-TARS-2 reached 47.5% — three real, verified generations of the same architecture, each better than the last.
    - **A real browser-agent vulnerability, CometJacking, succeeded against a real, deployed product** using exactly the mechanism this page's own repro tests: Brave Security's own root-cause diagnosis — *"when users ask it to 'Summarize this webpage,' Comet feeds a part of the webpage directly to its LLM without distinguishing between the user's instructions and untrusted content."* This page's own repro of that exact mechanism against Claude Sonnet 5 is a real, honest negative — the model resisted in both a raw and a tagged-content condition — but that's a fact about this run, not a structural guarantee the real Comet incident didn't already disprove for a different agent.

    ### [Personal and Always-On Agents](personal-always-on-agents.md)

    - **OpenClaw and Hermes Agent are real, current, open-source "always-on" personal agents**, both messaging-native (WhatsApp, Telegram, Discord, Slack, and more) and both explicitly local-first. OpenClaw's own framing: *"State lives on your machine, not a vendor cloud."* Hermes Agent's differentiator: *"the only agent with a built-in learning loop — it creates skills from experience, improves them during use."*
    - **"Blast radius" is the real, named concept for why persistent agents are categorically different**: *"A standard AI application processes a single user request, returns a response, and discards all session context. The risk surface is bounded by that single exchange. A persistent agent is categorically different"* — because *"blast radius describes the maximum potential damage an agent can cause if its goals are hijacked, its tools are misused, or its memory is poisoned."*
    - **ClawHavoc is a real, dated, multiply-corroborated incident that makes the concept concrete**: malicious actors distributed real credential-stealing malware through OpenClaw's own skill marketplace — Koi Security documented **341 malicious skills**, and Bitdefender found roughly **17% of skills analyzed in the platform's first few weeks carried malicious payloads**.
    - **Unit 42's own root-cause diagnosis names the exact architectural gap**: *"The lack of isolation between skill logic and agent authority means that installation results in complete control over the agent's identity"* — a malicious skill performs *"unauthorized actions through the agent's own authenticated sessions"* with no conventional exploit required.
    - **A real repro of that exact mechanism confirms it at the architecture level, independent of any model's judgment**: a deterministic, no-LLM check showed an unscoped skill's own code freely reading financial data and emailing it out, while a scoped, code-enforced tool allow-list blocked both calls before they ever executed — the real, structural fix, working exactly as the incident's own diagnosis implies it should.

    ## Production

    ### [Evaluating Agents](evaluating-agents.md)

    - **Outcome grading and trajectory grading answer different questions, and can disagree.** Outcome asks "was the final answer right?" Trajectory asks "did it get there the intended way?" The Holistic Agent Leaderboard's own log audits caught agents "searching for the benchmark on HuggingFace instead of solving a task" — a case an outcome-only grader would have scored as a pass.
    - **pass@k and pass^k measure opposite things.** pass@k asks whether at least one of k tries succeeded (generous — useful for capability ceilings). pass^k asks whether *every one* of k tries succeeded (strict — the real reliability question, since a production agent gets one real try per request, not k with a human picking the best). τ-bench's own numbers show why the gap matters: GPT-4o's retail success rate falls from under 50% at pass@1 to under 25% at pass^8.
    - **A real capability-motivated prompt change caught a genuine, clean regression** — not in the model's reasoning, in the *grader*. Told to spell out numbers in words for a plausible accessibility reason, the model's arithmetic stayed completely correct ("one hundred divided by four equals twenty-five") — every task in a digit-matching regression suite still failed, because the grader, not the model, broke.
    - **Infrastructure is a real, measurable confound in agent benchmarks.** Running the identical model and benchmark across container configurations from strict to uncapped produced a 6-percentage-point swing (p < 0.01) on Terminal-Bench 2.0 — bigger than many reported leaderboard gaps — purely from resource allocation, with nothing about the model changing at all.
    - **Capability evals and regression evals serve different jobs and shouldn't be conflated.** A capability eval asks how good the agent could be — expensive, exploratory, run rarely. A regression eval asks whether a specific change broke something that used to work — cheap, narrow, run on every change. This page's own regression-suite experiment is exactly that second job, and it worked precisely because the suite was simple enough to run constantly.

    ### [Benchmark Atlas](benchmark-atlas.md)

    - **Benchmark leaderboards drift out of sync with reality fast, and aggregator sites make it worse.** A real, current fetch of official leaderboard data (not secondary blog posts) found SWE-bench Verified's real top score is **79.2%**, while multiple aggregator sites reported 96–97% for the same benchmark — a real, current, checkable discrepancy worth verifying against the primary source before citing any benchmark number.
    - **SWE-bench Verified has a real, documented contamination problem, from OpenAI's own analysis, not a critic's.** Auditing failed problems, OpenAI found *"at least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions,"* and that *"all frontier models we tested were able to reproduce the original, human-written bug fix... indicating that all of them have seen at least some of the problems and solutions during training."* Their conclusion: *"we have stopped reporting SWE-bench Verified scores, and we recommend that other model developers do so too."*
    - **"Saturated" isn't one universal state — it's benchmark-specific and sometimes domain-specific within one benchmark.** OSWorld climbed from a 12.24% launch baseline to a real current top of 90.19% (saturated). TheAgentCompany's real top score is still under 43% (clearly not). tau-bench's own real numbers show both at once: 97.8% on its older telecom domain, but only 55.2% on banking_knowledge — a harder domain added specifically because the older ones stopped differentiating models.
    - **A real repro of tau-bench's actual grading methodology — action-state grading, and pass@1 vs. pass^k — found perfect reliability on one small, illustrative policy-compliance scenario**: 5/5 independent trials correctly refused a plausible-sounding but policy-violating cancellation request, graded by whether the policy-breaking tool was actually called, not by what the reply said. `pass_at_1: 1.0`, `pass_hat_k: true` — a small, real illustration of exactly the kind of easy scenario that stops differentiating frontier models, which is why benchmark maintainers keep adding harder ones.

    ### [Observability and Debugging](observability-debugging.md)

    - **Agent failures don't look like failures from the outside — that's the actual problem this whole track exists to solve.** The real, current framing: *"When your AI agent returns a confidently wrong answer, your monitoring sees a successful 200 response."* (Jamie Mallers, OneUptime, 2026-03-28) — traditional health checks (did the call error, did it time out) are structurally blind to an agent that completes cleanly and is simply wrong.
    - **OpenTelemetry's real GenAI semantic conventions give this a standard shape, not a bespoke one**: span names follow `{gen_ai.operation.name} {gen_ai.tool.name}` (e.g. `execute_tool get_daily_active_users`), with real, specified attributes — `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens` — and, critically, `gen_ai.input.messages`/`gen_ai.output.messages` as the spec's own designated place for the actual tool call arguments and results.
    - **A real repro built exactly this: real spec-shaped spans around an agent call to a subtly buggy tool.** The tool silently returned *cumulative all-time* users instead of yesterday's daily figure. The call completed with zero errors — `stop_reason: "end_turn"` — while the real final answer confidently claimed the number was *"very healthy... roughly 46.8x the goal."*
    - **An independent, trace-only diagnosis — no access to the final answer, no re-running the agent — found the real root cause immediately**: inspecting only the real `gen_ai.output.messages` attribute on the tool-execution span, the check correctly flagged the exact value as *"implausibly large for a single day, likely a cumulative/all-time figure mislabeled as daily."* This is the concrete, working version of the "3 a.m. question": can you tell what actually went wrong from the trace alone, at 3 a.m., without waking anyone else up to help re-run it?

    ### [Agent Security](agent-security.md)

    - **The lethal trifecta names the actual precondition for data exfiltration**, not a vague "be careful with untrusted content" warning: private-data access, exposure to untrusted content, and the ability to externally communicate, all live in the same session. Remove any one leg and the specific exfiltration risk this describes goes away, regardless of what the untrusted content says.
    - **A real repro of the trifecta didn't produce a successful exploit — worth being precise about what that does and doesn't show.** Claude Haiku 4.5 declined an injected forwarding request in both the vulnerable and the structurally-fixed condition. That's not evidence prompt injection is solved: "The Attacker Moves Second" reports bypassing 12 published defenses at above 90% success, where those same defenses originally reported near-zero attack success rates. The structural fix's actual value is that its guarantee doesn't depend on the model resisting at all.
    - **Tool poisoning reproduced cleanly, on a completely innocuous request.** Asked only "what's the weather in Paris," a model given a `get_weather` tool whose *description* secretly asked it to also pull an unrelated customer record did exactly that — a real supply-chain attack surface that lives in metadata nobody reads, not in a document anyone had to be tricked into opening.
    - **Meta's Rule of Two is a repackaging of the trifecta into a session-design rule, not an independent discovery** — and it's more permissive than it's often summarized: an agent may satisfy *any two* of the three properties, not zero.
    - **CaMeL's guarantee costs measurable capability**: provable security against prompt injection in AgentDojo, at 77% task success versus 84% for an undefended system — a real, quantified trade-off between structural safety and raw usefulness, not a free upgrade.

    ### [Guardrails and Human-in-the-Loop](guardrails-human-in-the-loop.md)

    - **Guardrails work in real, layered stages, not as one blanket switch** — a real, current framing names the stack directly: data and context guardrails, design-time governance, runtime guardrails and gateways, identity/access/security, and human-in-the-loop oversight as the final layer, not the only one.
    - **Risk tiering is a real, specific practice, not just "be careful with risky stuff"**: *"Define explicit risk tiers for agent use cases and apply proportional controls. Low-risk tasks like data summarization can run with lighter oversight. High-risk actions involving financial transactions, PII, or policy changes require multi-step verification, human approval, and comprehensive audit trails."*
    - **A real repro found the exact tradeoff risk-tiering is supposed to solve, playing out concretely**: flat-autonomous (no gates) let a $250 refund execute with zero review. Flat-gated (every action requires approval, no distinction by risk) stopped the agent after only 2 harmless lookup calls — it never even reached the risky refund step. Risk-tiered gating (only actions above a real threshold require approval) let both safe lookups proceed immediately and correctly blocked only the $250 refund — confirmed on two separate full runs.
    - **The real, sharper finding isn't "flat gating adds friction" — it's that undifferentiated gating can stall an agent before it even reaches the point where review matters.** The agent under flat-gating didn't slowly grind through extra approval steps; it stopped making progress entirely, two calls in, having never attempted the one action that actually needed a human.

    ### [Identity, Governance and Compliance](identity-governance-compliance.md)

    - **Real, current identity standards agree on the same principle: an agent should be identifiable AS an agent, not blended into the user it acts for.** Okta's Agent SSO (GA August 24, 2026) is real and verified: *"Agent SSO establishes a first-class identity model for AI agents at the point of connection"* — it *"registers it as a first-class identity in Universal Directory alongside human employees"* and *"issues short-lived, identity-governed tokens in place of stored credentials."* A real repro confirmed this is a design choice, not automatic: an agent told to just "act as the user" sent a fully anonymous approval email; the identical agent given an explicit identity and scoped delegation named itself and its authority in the same real, sent output.
    - **OWASP's real, current Top 10 for Agentic Applications (published December 9, 2025) names identity and trust failures directly**, not just technique-level attacks: **ASI03 – Identity & Privilege Abuse** ("leaked credentials enable agents to operate beyond intended scope") and **ASI09 – Human-Agent Trust Exploitation** ("confident explanations mislead operators into approving harmful actions") are real, distinct, named risk categories in the current list.
    - **NIST's real CAISI AI Agent Standards Initiative (launched February 17, 2026) exists specifically because the original AI RMF predates this problem** — its own three real, stated pillars: facilitating industry-led agent standards, fostering open-source protocol development, and *"advancing research in areas of AI agent security and identity to enable new use cases."*
    - **The EU AI Act has no agent-specific chapter — it applies its existing, real risk categories to agentic systems, not a bespoke agent regime.** Real, verified provisions: Article 50 (AI-interaction transparency, applies from August 2, 2026), Article 14 (human oversight for high-risk systems, including a real requirement for a "stop" mechanism), and Article 6/Annex III (high-risk classification by use case, applying from December 2027 and August 2028). Don't imply a dedicated "agent article" exists — it doesn't.
    - **MITRE ATLAS is the real, adversary-technique counterpart to OWASP's risk list** — a living knowledge base, modeled on MITRE ATT&CK, now real and verified as expanding "to capture attack paths that emerge at the orchestration and execution layers" as agents act with growing autonomy.

    ### [Sandboxes and Permissions](sandboxes-permissions.md)

    - **"Sandbox" is a ladder of real isolation strengths, not one thing**: Linux namespaces/cgroups (containers) share the host kernel — weakest tier. gVisor interposes a user-space kernel, the **Sentry**, that intercepts every syscall the sandboxed workload makes before it ever reaches the real host kernel — middle tier. Firecracker/Kata microVMs give each workload its own dedicated kernel via real hardware virtualization — strongest tier, and still fast: real, verified numbers are **boot in <125ms**, **<5 MiB memory overhead per VM**, and **up to 150 microVMs launched per second per host**. WebAssembly sits differently on the ladder — real, verified: it's *"lighter weight than containers or virtual machines"* and default-deny at the capability level, rather than default-shared-then-restricted.
    - **A sandbox isolates the agent's *execution*. It does not automatically isolate the agent's *credentials*.** A real, documented risk: *"A prompt-injected web page, a poisoned dependency's README, or an ambiguous instruction doesn't need to escape the sandbox. It just needs to ask the agent, which is already inside the trust boundary with the secret, to use it somewhere it shouldn't."* If the real credential sits in the sandboxed process's own environment, it's reachable the instant an injection convinces the agent to reach for it.
    - **The real fix is credential proxying**: route the actual call through a proxy outside the sandbox boundary, and give the sandboxed code only a placeholder token — the proxy substitutes the real credential itself, never handing it back. The real, verified guarantee: *"The agent's process never has the plaintext secret in memory, on disk, or in its context window."*
    - **But that guarantee has a real, named limit**: *"Proxying prevents credential theft. It does not prevent credential misuse."* A repro built directly around this: a deterministic, no-LLM check showed the real credential visible in an unproxied sandbox and only a placeholder visible in a proxied one — but a hand-written payment call to an attacker-controlled-looking account went through in **both** conditions regardless. Proxying changed what the code could *see*, not what it could *do*.

    ### [Durable Execution](durable-execution.md)

    - **Durable execution means a crash resumes the conversation instead of restarting it.** Temporal's own framing: "When a Worker crashes, the Temporal Service hands the work to another Worker, which replays the Event History and resumes at the line where execution stopped, with local variables and progress intact." For an agent specifically: "The loop is a Workflow, each model call and tool call is an Activity, and a crash resumes the conversation instead of restarting it."
    - **Replay and checkpointing are two different mechanisms that solve the same problem differently.** Temporal-style engines replay a recorded event history — re-running workflow code but skipping already-completed steps using their recorded results. LangGraph-style checkpointing persists application state directly as a snapshot. Both need a persistent backend to survive a real process crash: LangGraph's own docs note `MemorySaver` and `InMemorySaver` "store checkpoints in RAM. When the process restarts, all checkpoints are lost."
    - **A real, from-scratch repro of the exact failure mode worked cleanly.** A crash was deliberately triggered right after a tool's side effect committed but before that fact was durably logged — the single most dangerous instant for a naive retry. Resuming in a genuinely separate process replayed the completed steps with zero new model calls, then completed the task with no duplicated side effect.
    - **Idempotency is the second line of defense, not a redundant one.** Replaying the event log tells you what's *known* to have completed — it can't tell you about the gap between "the side effect happened" and "the log says it happened." An idempotent tool, checking its own persisted state before acting, is what actually prevents a double charge or a duplicate booking in that gap.
    - **This is a real, documented interview topic**, not a hypothetical: "How do you make sure agents do not double-execute side-effectful operations like charging a card or booking a ticket twice?" and "Suppose your booking agent sometimes reserves the same hotel twice — walk through how you'd debug and fix this" are both real, sourced interview questions.

    ### [Cost and Latency](cost-latency.md)

    - **Naive agent loops don't cost N times as much as a single call — they cost roughly N² times as much.** Every turn resends the full accumulating transcript, so a real, documented formula applies: `Total = N×S + u×N(N+1)/2 + r×N(N-1)/2` — the `N(N+1)/2` triangular-number term is the trap. A real, worked example: a 20-step loop generating 1,000 tokens/step produces **210,000 cumulative input tokens**, not the 20,000 a naive per-step estimate would suggest.
    - **A real repro confirmed the shape directly, and measured what windowing buys**: a naive 6-file tool loop's per-turn input tokens grew by a near-constant **~631 tokens each turn** (665→1290→1921→2552→3183→3814→4445) — that constant per-turn *increase* is exactly what makes the cumulative total quadratic. A windowed version (only the last 2 tool results sent in full) flattened that increase to **~121 tokens/turn** by the end, for a real **28.3% cumulative reduction** — a gap that only widens with more turns.
    - **Model routing has a real, large lever: current published pricing puts small and large tiers roughly 5-10x apart per token** (Haiku 4.5 at $1/$5 per million vs. Sonnet 5 at $2/$10 per million, input/output), and a real repro confirmed the upside directly — Haiku matched Sonnet's accuracy on a 10-question batch at **~44% of the real dollar cost**.
    - **But the routing mechanism itself has a real, sharp limitation**: asking a model to self-report its own confidence is not a reliable way to catch its own mistakes. A real repro found the smaller model reported `high` confidence on every question, including the one it got wrong — confidently, not hesitantly. This is a concrete, measured reason production routing methods lean on a different signal (like model-internal token-probability confidence) instead of self-report alone.

    ## Training

    ### [Training Agents: Reward and Credit](training-agents.md)

    - **GRPO replaces a critic with the group itself.** DeepSeekMath's own framing: "GRPO foregoes the critic model, instead estimating the baseline from group scores" — rewards for a group of sampled completions to the same prompt are "normalized by subtracting the group average and dividing by the group standard deviation," and that normalized value becomes every token's advantage.
    - **A real, exact run of that formula reproduces GRPO's own documented failure mode.** A group where every sample got the identical reward produced an advantage of exactly zero for all eight samples — DAPO's own paper names this the "gradient-decreasing problem": "if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients."
    - **DAPO's fix, run for real on the same degenerate group, is exactly what the paper describes**: over-sample and filter out any group whose accuracy is exactly 0 or exactly 1, keeping only groups with genuine variance — "leaving all prompts in the batch with effective gradients."
    - **Outcome reward and process reward answer different questions, with a measured real gap between them.** "Process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset" — a process-supervised model reported solving 78% of a representative MATH subset. A real toy repro shows exactly why: outcome-only credit can't tell a uniformly-bad trajectory from one where only the middle step failed; step-level credit can.
    - **DPO turns preference pairs directly into a classification loss, no separate reward model or RL loop needed** — "solve the standard RLHF problem with only a simple classification loss," reported to match or exceed PPO-based RLHF while being "stable, performant, and computationally lightweight." A real repro constructed one such pair from two real, independently sampled completions and a real judge call.

    ### [RL for Search and Tool Agents](rl-search-tool-agents.md)

    - **Search-R1 trains an LLM to interleave real search calls with reasoning, using outcome-only reward.** Real, verified: *"Search-R1 optimizes LLM reasoning trajectories with multi-turn search interactions... Experiments on seven question-answering datasets show that Search-R1 improves performance by 41% (Qwen2.5-7B) and 20% (Qwen2.5-3B) over various RAG baselines."* R1-Searcher and ReSearch train the same joint reasoning+search behavior via RL with no supervised reasoning-step data at all — R1-Searcher via a two-stage outcome-only process, ReSearch by treating search as guided by the model's own "text-based thinking."
    - **ToolRL studied reward design for tool use directly, and found coarse reward isn't enough.** Real, verified: *"coarse-grained reward signals, such as answer matching, fail to offer the finegrained feedback required for effective learning."* Its fix — decomposing reward into tool-name, parameter-name, and parameter-value components — produced *"a 17% improvement over base models and a 15% gain over SFT models."* This page's own repro reproduces the exact mechanism on real tool calls.
    - **ReTool shows RL can teach WHEN to reach for a tool, not just how.** Real, verified: on AIME, *"67% accuracy with 400 training steps"* vs. a text-only RL baseline's *"40% accuracy, 1080 steps"* — higher accuracy in roughly a third of the training steps — with the paper describing an emergent *"aha moment"* where the model starts self-correcting code mid-reasoning without being explicitly taught to.
    - **RAGEN names a real, recurring multi-turn RL failure mode: the Echo Trap.** Real, verified: *"a recurring instability pattern, Echo Trap, where agents overfit to locally rewarded reasoning patterns, marked by reward variability collapse, entropy drop, and gradient spikes."* Models converge to *"near-identical phrasing... without justification"* — real, multi-turn RL training doesn't just get slower when it goes wrong, it can collapse into repeating a shallow, memorized pattern that happens to score well locally.

    ## Interview

    ### [Interview Playbook](interview-playbook.md)

    - **The single strongest dated trend in the evidence: AI-assisted coding is becoming the interview format itself, not just the topic.** OpenAI runs a beta onsite round done "using an AI coding agent." Meta "added an AI-assisted coding round" where a small toy codebase is debugged and completed with LLM help. Sierra rebuilt its entire onsite around a 2-hour build session using any AI tooling, and removed its classic coding/algorithms interview outright. Anthropic, by contrast, explicitly disallows AI tools in its live rounds — the field hasn't converged on one answer.
    - **Most of what's published as "agent interview questions" is prep-market content, not candidate-confirmed.** Of 106 collected questions, only 14 trace to a first-person candidate report or a company's own published process; 81 come from prep sites, listicles, and openly "synthesized" repositories. Treat frequency in prep material as evidence of what the prep industry teaches, not what interviewers actually ask.
    - **A real loop can change within a year.** Sierra's own May-2025 candidate reports describe a take-home support agent plus a TypeScript/React debugging round; Sierra's own April-2026 blog post describes a completely different AI-native onsite. Prepping from a year-old write-up can mean prepping for a loop that no longer exists.
    - **Job descriptions and interview reports disagree on what to emphasize.** Kubernetes appears in 19.4% of AI-tech postings and in zero candidate interview reports reached. Idempotency and durable execution show up across multiple prep sources and real interview questions but in only 1.1% of postings. Evals is the rare topic strongly present in both.
    - **The one topic with genuine candidate confirmation across multiple independent sources: production reliability under real failure, not textbook agent architecture** — Swiggy's real four-question loop was entirely about uncertainty at scale, agent evals, LLM-as-judge, and not hallucinating success after a failed tool call; Sierra's real rounds are build-and-debug tasks, not "design an agent" whiteboard questions.

=== "Combined Scenario Check"

    136 questions from every page on this site, one combined pass instead of opening each page separately. Every question shows which page it's from — go re-read that page for anything you get wrong.

    <div class="quiz-widget" data-title="Combined Scenario Check — All Pages">
    <script type="application/json">
    {
      "questions": [
        {
          "scenario": "A support system does classify -> retrieve -> generate over 50 FAQ categories, chosen by embedding similarity. When the best match's similarity score falls below a fixed threshold, the code shows the customer 3 candidate articles instead of generating one answer.",
          "question": "Does the threshold branch make this system an agent?",
          "options": [
            "No \u2014 a fixed numeric threshold in code decides the branch; the model never gets to choose what happens next",
            "Yes \u2014 showing candidates instead of one answer means the system is adapting to retrieval quality, which is exactly the decision-point behavior that defines an agent",
            "It depends on whether the embedding model or the generation model owns the threshold value",
            "Yes, because the system now has more than one possible output path depending on what the input is"
          ],
          "correct": 0,
          "explanations": [
            "Correct. A fixed threshold in code is choosing the next step, not the model. Nothing here lets the model itself decide what happens next based on its own read of the situation \u2014 replace the threshold with any other number and the architecture is identical.",
            "This is the tempting trap: the system's output genuinely does vary with live data (the similarity score), which sounds like the 'depends on the outcome of a previous action' criterion. But varying output isn't the same as the model deciding \u2014 a lookup table that branches on a number is still just a lookup table, however adaptive it looks from outside.",
            "The threshold lives in application code either way, regardless of which model's score it reads \u2014 this doesn't change who's making the branching decision.",
            "Multiple possible output paths is necessary but nowhere near sufficient \u2014 a large if/elif chain has many output paths and is still a workflow. The test is who picks the path at runtime, not how many paths exist."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "A pipeline generates an answer, then makes a second LLM call asking 'Is this answer good enough? Reply yes or no.' If the reply is 'no,' the code regenerates once more (hard-capped at 2 total attempts) and returns whichever attempt was produced last, regardless of what the second judgment says.",
          "question": "Where does this system sit?",
          "options": [
            "It's an agent \u2014 an LLM is making the 'good enough' judgment here, and any real judgment call counts as agentic behavior",
            "It's an agent on the second attempt only, since that specific point is where the branching actually happens",
            "It's a workflow \u2014 the yes/no only gates a branch fixed code already wired in; the model never picks a different action",
            "It's ambiguous \u2014 the answer depends on which provider or model is used to make the judgment call"
          ],
          "correct": 2,
          "explanations": [
            "This is the most common wrong intuition on this whole page: an LLM call is involved in the decision, so it feels agentic. But this is exactly the pattern Anthropic names 'evaluator-optimizer' and classifies as a workflow \u2014 the LLM fills in a yes/no gate whose consequences (retry once, cap at 2, return regardless of the second verdict) were fully decided by the code author in advance.",
            "Branching happening at a specific point doesn't retroactively make earlier or later steps agentic or non-agentic \u2014 the whole system is one architecture, evaluated by the same rule throughout.",
            "Correct. The action space here has exactly two pre-wired outcomes (retry once, or don't), chosen by code, not model-selected from live options. Compare this to this page's own recipe: the judge step there feeds into a decision where the model picks among BROADEN/CLARIFY/ESCALATE \u2014 a genuinely open action set, not a single fixed retry slot.",
            "The provider generating the yes/no token is irrelevant to this distinction \u2014 the question is about who controls the consequence of that token, and that's fixed code regardless of vendor."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "An interviewer describes a system: 'We handle exactly 12 known customer request types. For each, there's one verified, deterministic procedure to follow, and the correct order of steps never depends on what happens during execution.' They ask whether you'd build this agentically.",
          "question": "What's the strongest response?",
          "options": [
            "Build it as an agent anyway \u2014 future request types outside the current 12 might appear, so agent flexibility future-proofs the system now",
            "It must be an agent, because customer-facing systems are inherently unpredictable",
            "Build it as an agent, because letting the model choose the procedure is more reliable than hard-coded step selection on a fully known task",
            "Build it as a workflow \u2014 the 12 procedures are already known and don't branch on live results, so agentic overhead buys nothing here"
          ],
          "correct": 3,
          "explanations": [
            "A real and common engineering trap \u2014 over-building for imagined future flexibility the task doesn't currently need. If new request types show up later, that's the moment to reconsider, not a reason to pay agentic costs today for a fully solved, fully verified problem.",
            "An unfounded generalization \u2014 the scenario explicitly states the procedures are verified and deterministic. 'Customer-facing' doesn't imply unpredictable; the two are independent.",
            "False, and worth catching directly: an LLM choosing among known, verified procedures is not more reliable than the verified procedures themselves \u2014 on a fully known task, hard-coded logic doesn't fail in the ways a model's live judgment can.",
            "Correct. This is the 'When you would not build an agent' trade-off from this page, applied to a concrete case instead of asked in the abstract \u2014 the task is fully known and doesn't need branching on live feedback, so a fixed pipeline is strictly more reliable and cheaper here."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "A candidate is asked how LLM agents improve over a deployment's lifetime, given they don't get gradient updates from a reward signal the way RL agents do. The candidate answers: 'They don't really improve \u2014 each session starts fresh with the same weights, so there's no real analog to RL's learning.'",
          "question": "What's the best critique of this answer?",
          "options": [
            "The candidate is basically right \u2014 without fine-tuning applied mid-conversation, there is no real improvement mechanism happening at all",
            "The candidate is missing the in-context mechanisms (reflection, memory, refined tools) that persist across sessions, plus RL applied on top of the deployed loop",
            "The candidate is wrong because model weights are automatically updated after every conversation",
            "The candidate is right for closed-source models, but wrong for open-weight models, since publicly released weights are retrained continuously as usage data comes in between sessions"
          ],
          "correct": 1,
          "explanations": [
            "Partial credit at best: it's true no gradient update happens mid-session, but the candidate is answering a narrower question than the one asked \u2014 the interviewer asked about improvement across a deployment's lifetime, not within one session.",
            "Correct \u2014 and it's the exact bridge this page's RL-vs-LLM-agent table points to: session-to-session improvement happens through what persists in context (reflection, memory, tool refinement) and, increasingly, through RL applied on top of the already-deployed agent loop rather than as a separate pretraining phase.",
            "A common but flatly false belief \u2014 no mainstream deployed LLM updates its own weights automatically mid-conversation or between sessions from ordinary usage.",
            "Also false, and a plausible-sounding trap for the same reason as the option above \u2014 open-weight availability has no bearing on whether a specific deployment retrains itself; most don't, open or closed."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
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
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
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
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
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
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
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
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
        },
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
          ],
          "source": "Workflow Patterns",
          "sourceUrl": "workflow-patterns.md"
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
          ],
          "source": "Workflow Patterns",
          "sourceUrl": "workflow-patterns.md"
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
          ],
          "source": "Workflow Patterns",
          "sourceUrl": "workflow-patterns.md"
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
          ],
          "source": "Workflow Patterns",
          "sourceUrl": "workflow-patterns.md"
        },
        {
          "scenario": "A team is deciding between ReAct and ReWOO for a pipeline that answers questions requiring 4-6 tool calls each. A teammate argues: 'ReWOO is strictly better here -- it needs way fewer LLM calls for the same result, so there's no real reason to use ReAct for a tool-heavy pipeline like this.'",
          "question": "What's the strongest objection to that claim?",
          "options": [
            "ReAct is cheaper in tokens too, since each prompt is shorter than ReWOO's solve call",
            "ReWOO can't handle steps that depend on each other's results at all",
            "A bad ReWOO plan step is only caught once it actually executes, not before",
            "ReAct only makes sense when none of the tools involved have side effects"
          ],
          "correct": 2,
          "explanations": [
            "Backwards from the measured pattern -- ReWOO's real advantage IS tokens, not ReAct's; ReAct's growing, resent transcript is the more expensive side, not the cheaper one.",
            "Wrong on the mechanism -- ReWOO explicitly supports dependent steps through #E-style variable substitution; a later step can and does reference an earlier step's not-yet-known result.",
            "Correct. This is exactly what a real run on this page hit: a generated ReWOO plan referenced a function its tool didn't support, and that step only failed once the deterministic executor actually ran it -- nothing checked the plan against real tool behavior first. ReAct's per-step decisions mean a bad result is visible to the model before it commits to the next action.",
            "An arbitrary, fabricated condition with no grounding in either mechanism -- side effects aren't what separates when each approach is appropriate."
          ],
          "source": "Reasoning Paradigms",
          "sourceUrl": "reasoning-paradigms.md"
        },
        {
          "scenario": "After reading about LLM Compiler's parallel dispatch, an engineer proposes: 'Let's just always run every tool call in a ReWOO-style plan concurrently -- more parallelism is strictly an improvement, there's no downside.'",
          "question": "What's wrong with running every step of a ReWOO plan concurrently by default?",
          "options": [
            "A step referencing an earlier step's result can't run before that result exists",
            "Concurrency is only unsafe when a tool has side effects, not because of dependencies",
            "Nothing is wrong -- concurrency can only help wall-clock time, never hurt it",
            "ReWOO plans must always execute strictly sequentially, by design, unlike LLM Compiler"
          ],
          "correct": 0,
          "explanations": [
            "Correct. This is precisely the dependency check LLM Compiler's dispatch unit has to do: a step referencing #E1 in its own arguments cannot execute correctly until #E1's real value exists. This page's own real plan had exactly this shape -- three independent lookups, then three dependent arithmetic steps that needed those lookups' results.",
            "A real consideration for a different failure mode (concurrent writes), but not the reason dependent steps specifically can't run early -- a read-only step that depends on another step's result still can't run before that result exists.",
            "Ignores real dependencies entirely -- a plan step that substitutes an earlier variable into its own arguments has a genuine data dependency, not just a stylistic one; running it early isn't just unnecessary, it's incorrect.",
            "Overstates ReWOO's actual design -- ReWOO's contribution is the plan-then-execute split and variable substitution, not a claim that execution must be sequential; nothing in the paper argues against parallelizing independent steps within a plan."
          ],
          "source": "Reasoning Paradigms",
          "sourceUrl": "reasoning-paradigms.md"
        },
        {
          "scenario": "A candidate is asked to explain Plan-and-Solve prompting in a system-design interview and answers: 'It's an architecture where a dedicated planner model breaks the task into subtasks and hands each one to a separate executor model or agent.'",
          "question": "What's the most accurate correction to this answer?",
          "options": [
            "Right description, just the wrong paper name -- swap in 'ReWOO' and it's accurate",
            "Accurate for Plan-and-Solve, but only when used inside a multi-agent framework",
            "Correct that it uses two separate calls -- just wrong about separate models",
            "It's one zero-shot prompt to a single model, not a multi-model architecture"
          ],
          "correct": 3,
          "explanations": [
            "Doesn't fix the actual error -- ReWOO also isn't a multi-model planner/executor architecture; it's a plan-then-execute mechanism with one planning call, not separate planner and executor models.",
            "No such conditional exists in the paper -- Plan-and-Solve was evaluated as a standalone zero-shot prompting method, not as a component requiring a multi-agent wrapper to function as described.",
            "Fabricates a two-call structure that isn't how the technique works -- it's one prompt, one call, asking for planning and solving in the same response, not two separate calls to two separate steps.",
            "Correct. Plan-and-Solve's entire mechanism is prompt text: one instruction asking the same model to plan before solving, in a single zero-shot call, contrasted against plain chain-of-thought. There's no second model, no handoff, no separate executor role."
          ],
          "source": "Reasoning Paradigms",
          "sourceUrl": "reasoning-paradigms.md"
        },
        {
          "scenario": "A researcher reads DeepSeek-R1's reported finding that self-reflection and verification emerged from pure reinforcement learning, and concludes: 'This means Reflexion-style explicit retry loops are now obsolete -- there's no reason to build one anymore.'",
          "question": "What's the strongest flaw in that conclusion?",
          "options": [
            "Wrong only because Reflexion also needs tasks with no verifiable ground truth",
            "It generalizes one training result to every model, ignoring what that internal process can't do",
            "Backwards -- Reflexion-style loops became MORE necessary once failures got harder to detect",
            "Correct -- if the capability exists internally, building it externally again is pure redundancy"
          ],
          "correct": 1,
          "explanations": [
            "Introduces a claim with no support from anything on this page -- ground-truth availability isn't the axis the DeepSeek-R1 finding or the Reflexion paper turns on.",
            "Correct. The finding is real and worth knowing, but it's scoped to models actually trained that way, and an internal process baked into generation isn't the same tool as an explicit, inspectable, interruptible loop your code controls -- and it says nothing about ReWOO or LLM Compiler's problem, which is about round-trips and token cost across tool calls, not about whether the model second-guesses itself.",
            "Asserts a specific causal claim (harder to detect, therefore more necessary) that isn't established anywhere in the cited material -- a plausible-sounding but unsupported leap.",
            "Takes the finding at face value without noticing what it doesn't cover -- 'emerged in this training setup' isn't the same claim as 'therefore always redundant to build externally.'"
          ],
          "source": "Reasoning Paradigms",
          "sourceUrl": "reasoning-paradigms.md"
        },
        {
          "scenario": "A real run solved the identical tool-calling task with the raw Anthropic client (15 lines, 2 calls), LangChain/LangGraph's create_agent (8 lines, 2 calls), and Pydantic AI's Agent (5 lines, 2 calls) -- all three produced the correct answer.",
          "question": "What is the most accurate interpretation of the line-count difference?",
          "options": [
            "It measures loop code written by hand vs adopted as-is, not code quality",
            "It proves Pydantic AI is the objectively best choice for any agent task",
            "The comparison is meaningless, since real agents never resemble a simple demo task",
            "It shows the raw client is poorly written and should be shortened to match the frameworks"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real measured gap tracks exactly what each layer is for -- how much of the send/check/execute/append loop you write yourself versus hand to a framework's own conventions -- a real, honest, non-evaluative fact, not a verdict on which is 'better.'",
            "Overreaches from one narrow, simple task to a universal quality claim -- fewer lines on this specific task says nothing about how well any framework's assumptions fit a different, harder problem.",
            "Overcorrects into dismissing a real, controlled measurement -- a simple task with a checkable ground truth is exactly what makes a fair comparison possible; it doesn't claim to generalize to every production scenario.",
            "Misreads the finding -- the raw client's length isn't a flaw, it's the actual content of the loop other approaches abstract away; shortening it would mean hiding logic, not writing it better."
          ],
          "source": "Choosing a Framework",
          "sourceUrl": "choosing-a-framework.md"
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
          ],
          "source": "Choosing a Framework",
          "sourceUrl": "choosing-a-framework.md"
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
          ],
          "source": "Choosing a Framework",
          "sourceUrl": "choosing-a-framework.md"
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
          ],
          "source": "Choosing a Framework",
          "sourceUrl": "choosing-a-framework.md"
        },
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
          ],
          "source": "Tool Design",
          "sourceUrl": "tool-design.md"
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
          ],
          "source": "Tool Design",
          "sourceUrl": "tool-design.md"
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
          ],
          "source": "Tool Design",
          "sourceUrl": "tool-design.md"
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
          ],
          "source": "Tool Design",
          "sourceUrl": "tool-design.md"
        },
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
          ],
          "source": "Context Engineering",
          "sourceUrl": "context-engineering.md"
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
          ],
          "source": "Context Engineering",
          "sourceUrl": "context-engineering.md"
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
          ],
          "source": "Context Engineering",
          "sourceUrl": "context-engineering.md"
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
          ],
          "source": "Context Engineering",
          "sourceUrl": "context-engineering.md"
        },
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
          ],
          "source": "Tools at Scale",
          "sourceUrl": "tools-at-scale.md"
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
          ],
          "source": "Tools at Scale",
          "sourceUrl": "tools-at-scale.md"
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
          ],
          "source": "Tools at Scale",
          "sourceUrl": "tools-at-scale.md"
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
          ],
          "source": "Tools at Scale",
          "sourceUrl": "tools-at-scale.md"
        },
        {
          "scenario": "A real repro found that mcp==2.2.0 -- an SDK release whose own documentation states it targets the 2026-07-28 spec -- rejects a raw tools/list request with 'Bad Request: Missing session ID' when run with its default streamable_http_app() configuration, and only behaves statelessly once stateless_http=True is passed explicitly.",
          "question": "What's the most accurate conclusion to draw from this result?",
          "options": [
            "Targeting a spec version and defaulting to that spec's behavior are two different claims",
            "This is a bug in the recipe's own code, not a real property of the installed SDK",
            "The spec's statelessness requirement must not actually apply to the Streamable HTTP transport",
            "The SDK's version number was reported incorrectly and it does not actually target this spec"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The SDK genuinely implements spec-compliant statelessness -- it works correctly once explicitly enabled -- but ships it as an opt-in flag rather than the default, which is exactly what this page's real test demonstrated with raw HTTP requests and real response headers, not an assumption.",
            "Misattributes the finding -- the default-mode rejection came from the installed SDK's own real internal logic (streamable_http_manager.py's session handling), not from any code this recipe wrote; the recipe's code only sent a plain HTTP request and reported the real response.",
            "Contradicts the real, verbatim quote from the spec's own changelog directly: 'Remove protocol-level sessions and the Mcp-Session-Id header from the Streamable HTTP transport' -- the requirement applies to exactly this transport.",
            "Unsupported and contradicted by the SDK's own real documentation, which explicitly and directly states it targets the 2026-07-28 spec -- the discrepancy is about default configuration, not a false version claim."
          ],
          "source": "MCP Deep Dive",
          "sourceUrl": "mcp-deep-dive.md"
        },
        {
          "scenario": "A real requestState repro showed that a token minted for Alice's checkout (method='tools/call', target='checkout', args={'cart_id': 'cart_42'}) was correctly rejected when presented with a different cart_id, even though the token itself was untampered and validly sealed.",
          "question": "What security property does this specific rejection demonstrate, distinct from the tamper-detection check?",
          "options": [
            "That Alice's account lacked sufficient permissions to access a different shopping cart",
            "That the token cryptographically commits to the exact arguments it was minted for",
            "That AES-256-GCM is a stronger algorithm than the one used for the tamper check",
            "That the token's TTL had already expired by the time the second request arrived"
          ],
          "correct": 1,
          "explanations": [
            "Not tested in this repro at all -- no authorization/permissions layer was involved; the rejection came purely from the requestState envelope's own binding check inside the demo's unseal logic, prior to any application-level permission decision.",
            "Correct. This is 'request-binding': the sealed claims envelope includes a digest of the exact arguments the token was minted for (via the 'a' claim), so presenting the same otherwise-valid token against different arguments fails that binding check -- a property distinct from (and additional to) simple tamper detection.",
            "Both checks used the identical codec (AESGCMRequestStateCodec) -- there's no second, stronger algorithm involved; this option invents a distinction the real setup doesn't have.",
            "Not what this specific test isolated -- the request-binding rejection is a separate check from the recipe's dedicated expiry test (which used a different, deliberately short-TTL scenario); this rejection specifically involved a mismatched argument, not elapsed time."
          ],
          "source": "MCP Deep Dive",
          "sourceUrl": "mcp-deep-dive.md"
        },
        {
          "scenario": "A candidate explains the new MCP spec by saying: 'They removed sessions to make the protocol stateless, which simplifies things because servers no longer need to track any state between requests.'",
          "question": "What's the most accurate correction to this explanation?",
          "options": [
            "The change affects only the deprecated HTTP+SSE transport, not Streamable HTTP at all",
            "Sessions weren't removed -- only the name of the Mcp-Session-Id header changed in this revision",
            "Removing sessions relocated cross-call state into a protected value, not removed the need for state",
            "This is fully accurate -- protocol-level statelessness means servers genuinely never need any state"
          ],
          "correct": 2,
          "explanations": [
            "Backwards -- HTTP+SSE is being deprecated in favor of Streamable HTTP, and it's specifically Streamable HTTP that the changelog names as having its Mcp-Session-Id header removed in this revision.",
            "Contradicts the real, quoted spec changes directly -- the changelog explicitly states protocol-level sessions and the Mcp-Session-Id header are removed, not renamed; this recipe's own real test confirmed the header is genuinely absent under stateless_http=True.",
            "Correct. The real design move (and this page's own framing, backed by the requestState repro) is that removing protocol-level sessions didn't eliminate cross-request state -- it moved the responsibility for protecting that state from the transport to an explicit, self-describing, cryptographically sealed value (requestState) the client carries and the server verifies on every use.",
            "Overstates the claim -- the spec's own changelog explicitly names the replacement mechanism: 'Servers that need cross-call state use explicit, server-minted handles passed as ordinary tool arguments,' meaning the NEED for cross-call state didn't disappear, only how it's carried changed."
          ],
          "source": "MCP Deep Dive",
          "sourceUrl": "mcp-deep-dive.md"
        },
        {
          "scenario": "An MCP server proxies requests to a third-party API. To simplify its own code, it accepts whatever bearer token the MCP client sends and forwards that exact token unmodified to the downstream API, without checking who or what it was originally issued for.",
          "question": "What does the spec's own security guidance say about this specific pattern?",
          "options": [
            "It is only a risk if the third-party API and the MCP server happen to share the same audience claim",
            "It is a required pattern for any MCP server that acts as a proxy to a third-party API",
            "It is acceptable as long as the downstream API independently validates the token itself",
            "It is explicitly forbidden -- servers must not accept tokens that were not issued for themselves"
          ],
          "correct": 3,
          "explanations": [
            "Inverts the actual risk condition -- the danger is exactly when audiences are NOT properly validated (accepting tokens regardless of their intended audience); a shared audience claim wouldn't be the triggering risk factor here, mismatched or unchecked audiences are.",
            "The opposite of what the spec recommends -- token passthrough is named as an anti-pattern specifically, not a required or even acceptable implementation choice for proxy servers.",
            "Contradicts the spec's own stated mitigation directly, which places the obligation on the MCP server itself, not on hoping a downstream service happens to catch the problem: 'MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server' -- stated as an absolute requirement, not a conditional one.",
            "Correct. The spec is unambiguous and uses MUST NOT language: a server must validate that a token was properly issued to itself before using or forwarding it, precisely because passthrough reintroduces the confused-deputy problem and breaks real OAuth audience-validation boundaries."
          ],
          "source": "MCP Deep Dive",
          "sourceUrl": "mcp-deep-dive.md"
        },
        {
          "scenario": "A real repro against the live official MCP registry found that natural-language queries like 'browser automation form filling' and 'payment charges status' returned zero results, while brand-name queries like 'Stripe' and 'playwright' returned real matches.",
          "question": "What is the most precise real conclusion this pattern supports about the registry's current search behavior?",
          "options": [
            "The registry's search behaves closer to keyword or brand matching than to semantic search",
            "The registry's search is broken and should not be used by any agent for any purpose",
            "The registry only indexes servers made by large, well-known companies and excludes smaller developers",
            "Natural-language queries always fail while single-word queries always succeed, without exception"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, repeated pattern -- natural-language capability descriptions failing, specific product/brand names succeeding -- is precisely the signature of keyword/substring-style matching rather than a semantic search that understands intent or synonyms.",
            "Overstates the finding -- the repro shows a specific, characterizable LIMITATION (keyword-style matching), not that the search is non-functional; brand-name queries worked correctly and returned real, relevant results.",
            "Not what the evidence shows -- the successful queries were about MATCHING TERMS (brand names), not about company size; nothing in the repro tested or claims a bias toward large companies specifically.",
            "Overgeneralizes into an absolute rule -- the repro reports what happened on the SPECIFIC queries tried, not a proven universal law; the page is careful to characterize this as an observed pattern on real trials, not an exceptionless guarantee."
          ],
          "source": "MCP and Tool Ecosystem",
          "sourceUrl": "mcp-tool-ecosystem.md"
        },
        {
          "scenario": "A direct, separate check of the real MCP registry found that Microsoft's own official playwright-mcp server -- a genuinely popular project with 37.5k+ GitHub stars -- does not appear in the registry under any search term tried, including a direct search for 'microsoft'.",
          "question": "What is the most accurate interpretation of what this specific finding demonstrates?",
          "options": [
            "Microsoft's playwright-mcp is not a real or trustworthy MCP server, which is why it's absent",
            "The registry is a fully complete and authoritative source of every MCP server that currently exists",
            "The agent's search strategy in the repro was flawed, which is why it failed to find this server",
            "The registry, as a real preview-stage product, has genuine gaps even for well-known servers today"
          ],
          "correct": 3,
          "explanations": [
            "Not supported and not the claim -- popularity (37.5k+ stars) is independent evidence the server is real and widely used; its absence from the registry says something about the REGISTRY's completeness, not the server's legitimacy.",
            "Directly contradicted -- this is exactly the finding that disproves completeness; a well-known, popular server missing entirely is direct evidence against the registry being authoritative or exhaustive at this stage.",
            "Not supported -- the check that found this was a DIRECT query for 'microsoft' itself, not a flawed derived query; the absence held regardless of search phrasing, pointing to the registry's own data rather than a searcher's mistake.",
            "Correct. The registry is explicitly described as a real, preview-stage product (launched 2025-09-08), and this finding is direct, confirmed evidence of a real current gap in its population -- consistent with 'honest but incomplete,' not a flaw in how anyone searched it."
          ],
          "source": "MCP and Tool Ecosystem",
          "sourceUrl": "mcp-tool-ecosystem.md"
        },
        {
          "scenario": "This page distinguishes four real, separate layers in the MCP ecosystem: the registry (discovery), gateways (auth/rate-limiting/routing), remote servers (where code runs), and tool platforms (integration aggregation).",
          "question": "Based on the page's own framing, what happens when a well-known server is missing from the registry (as with the Playwright example), and a team ALSO adds a gateway product in front of their MCP servers?",
          "options": [
            "The gateway automatically expands the registry's listings, so the missing server becomes discoverable through the registry itself",
            "The registry gap remains unaffected by the gateway, since gateways address access control, not discovery completeness",
            "Adding a gateway makes the registry unnecessary, since gateways independently solve the discovery problem too",
            "The gateway and the registry are two names for the same underlying system, so fixing one automatically fixes the other"
          ],
          "correct": 1,
          "explanations": [
            "Not how these layers relate per the page -- gateways sit in front of servers for auth/routing/rate-limiting; they don't write to or expand the registry's own listings, which is a separate real system entirely.",
            "Correct. The page explicitly frames these as solving DIFFERENT real problems -- 'a well-known, unlisted server... is a registry gap a gateway or platform doesn't fix' -- a gateway operates on servers you already know about and have configured, not on expanding what's discoverable in the registry.",
            "Overstates a gateway's role -- the page frames gateways as an access-control/routing layer over KNOWN servers, not as a discovery mechanism that could substitute for the registry's cataloging function.",
            "Contradicted directly -- the page treats these as four distinct, separately real layers solving different problems, not interchangeable names for the same system."
          ],
          "source": "MCP and Tool Ecosystem",
          "sourceUrl": "mcp-tool-ecosystem.md"
        },
        {
          "scenario": "In the real repro's payment task, the agent chose the official com.stripe/mcp server over a third-party hosted wrapper (eu.nordicmcp/stripe) that also appeared in the real search results and offered a broader toolset (18 tools).",
          "question": "What was the real, stated reasoning behind this choice, according to the agent's own actual output?",
          "options": [
            "The third-party wrapper was reported as non-functional or broken in the real search results returned",
            "The official server was the only one of the two that had any remote endpoint URL listed at all",
            "The official server avoids the added trust and dependency cost of a nearby third-party intermediary",
            "The third-party wrapper had fewer tools available, making it strictly less capable for the task"
          ],
          "correct": 2,
          "explanations": [
            "Not indicated anywhere -- both real servers appeared as valid, listed entries in the real search results; nothing in the repro's data suggests the third-party option was broken or non-functional.",
            "Factually reversed from what's described -- the third-party wrapper DID have a real remote endpoint listed (a token-based URL); the distinguishing factor wasn't presence/absence of an endpoint.",
            "Correct. The agent's own real output explicitly reasoned about trust: choosing the first-party server specifically to avoid 'a trust/dependency layer you don't need when the official one exists' -- a security-aware judgment about intermediary risk near sensitive payment data, not a capability comparison.",
            "Backwards -- the third-party wrapper was described as offering a BROADER toolset (18 tools including reporting), not a narrower one; capability breadth wasn't the deciding factor and if anything favored the wrapper, which the agent still declined in favor of trust."
          ],
          "source": "MCP and Tool Ecosystem",
          "sourceUrl": "mcp-tool-ecosystem.md"
        },
        {
          "scenario": "A real repro measured two real conditions on an identical task needing only one of three fictional skills: all_upfront (all 3 skills' full bodies in the system prompt) cost 4,975 real input tokens; progressive (only short metadata upfront, one skill's body loaded on demand) cost 3,069 real tokens, and correctly never loaded the other two skills.",
          "question": "What does this specific result most precisely demonstrate about Agent Skills' progressive disclosure?",
          "options": [
            "Progressive disclosure eliminates all context cost for skills the agent doesn't end up using at all",
            "The real token gap between the two conditions would stay exactly 38.3% regardless of how many skills are installed",
            "The all-upfront condition failed to complete the task correctly, unlike the progressive condition",
            "The real token savings scale with how many INSTALLED skills are irrelevant to a given task, not a fixed percentage"
          ],
          "correct": 3,
          "explanations": [
            "Overstates it slightly -- there's still a real, small, nonzero cost even for unused skills under progressive disclosure: their Level-1 metadata (~100 tokens each) is always loaded, per Anthropic's own docs; only Level 2 (the full body) is truly free until triggered.",
            "Not what the page claims -- it explicitly says the gap is STRUCTURAL and scales with the total number of installed skills relative to what's actually used, not a fixed percentage that would hold constant as more skills are added.",
            "Not indicated anywhere -- both conditions are about real TOKEN COST measured on the same task; nothing in the real results suggests the all-upfront condition failed or produced an incorrect answer, only that it cost more.",
            "Correct. The page states this directly: 'all_upfront's cost scales with the total number of installed skills regardless of relevance, while progressive's cost scales with the number of skills actually used' -- the 38.3% figure is this specific run's real result, not a fixed, general ratio."
          ],
          "source": "Skills, A2A and Other Protocols",
          "sourceUrl": "skills-a2a-protocols.md"
        },
        {
          "scenario": "A team is building a system where one agent needs to occasionally hand off part of a task to a completely separate agent built by a different vendor, using a different framework.",
          "question": "Based on this page's own relationship-type framing, which real protocol is specifically designed for this scenario?",
          "options": [
            "AG-UI, since it standardizes how agents connect to user-facing applications",
            "A2A, since it standardizes communication and discovery between separate agents",
            "MCP, since it standardizes how an agent connects to its own tools and data",
            "AGENTS.md, since it provides instructions for coding agents working in a repository"
          ],
          "correct": 1,
          "explanations": [
            "Mismatched relationship -- AG-UI is explicitly the agent-to-USER-INTERFACE layer (rendering what an agent is doing for a human watching), not agent-to-agent communication between two separate backends.",
            "Correct. The page's own relationship-type framing maps this exactly: A2A standardizes AGENT-TO-AGENT communication and discovery (via the real Agent Card mechanism), which is precisely a handoff between two separate, differently-built agents -- distinct from MCP's agent-to-tool relationship.",
            "Mismatched relationship -- MCP standardizes an agent's connection to ITS OWN tools and data sources, not communication with a separate, independently-built agent; that's explicitly the distinction the page draws between MCP and A2A.",
            "Mismatched relationship -- AGENTS.md is a static, repo-level convention for giving ANY coding agent working in that repo context and instructions; it doesn't address live communication or handoff between two running agents."
          ],
          "source": "Skills, A2A and Other Protocols",
          "sourceUrl": "skills-a2a-protocols.md"
        },
        {
          "scenario": "The Agent Client Protocol's own documentation states: 'ACP solves this by providing a standardized protocol for agent-editor communication, similar to how the Language Server Protocol (LSP) standardized language server integration.'",
          "question": "What is the most precise real-world problem this LSP analogy indicates ACP is solving?",
          "options": [
            "Before ACP, each editor needed custom integration work for every different coding agent it supported",
            "Before ACP, coding agents were unable to read or write any files within a code editor's workspace",
            "Before ACP, language servers and coding agents used completely incompatible communication protocols",
            "Before ACP, there was no way for any coding agent to run inside a terminal-based development environment"
          ],
          "correct": 0,
          "explanations": [
            "Correct. This mirrors exactly what LSP solved for language servers -- before LSP, every editor needed custom integration per language server; before ACP, per the same analogy, every editor needed custom integration per coding agent. ACP standardizes that N-times-M integration problem down to one protocol each side implements once.",
            "Not indicated -- the LSP analogy is about COMMUNICATION STANDARDIZATION between two already-functional systems (editors and agents), not about a prior total inability for agents to access files; file access is a capability question, not the integration problem LSP/ACP solve.",
            "Not the analogy being drawn -- ACP is compared to LSP as a NEW protocol solving a similar STRUCTURAL problem (N-times-M custom integrations), not because language servers and coding agents were previously trying and failing to use the same protocol.",
            "Not supported by the quote or the page's broader description -- ACP's docs describe local (stdio) and remote (HTTP/WebSocket) transport options, with no claim that terminal-based environments were previously unable to run any coding agent at all."
          ],
          "source": "Skills, A2A and Other Protocols",
          "sourceUrl": "skills-a2a-protocols.md"
        },
        {
          "scenario": "AGENTS.md's own documentation describes it as 'a README for agents' and states it 'is just standard Markdown... the agent simply parses the text you provide,' with no required schema, and separately notes it is 'used by over 60k open-source projects.'",
          "question": "What does the combination of 'no required schema' and the real, verified adoption figure most directly suggest about why AGENTS.md succeeded as a convention?",
          "options": [
            "Its adoption figure proves it is technically more powerful than structured formats like JSON or YAML for this purpose",
            "60k projects were contractually required to adopt it as a condition of using coding agent tools",
            "Its lack of structure requirements lowered the barrier to writing one, plausibly helping drive broad adoption",
            "The lack of a schema means AGENTS.md files cannot actually be parsed or used reliably by any coding agent"
          ],
          "correct": 2,
          "explanations": [
            "Not a supported claim -- adoption figures reflect real usage, not a technical power comparison against structured formats; the page doesn't argue Markdown is 'more powerful' than JSON/YAML, just that it's low-friction to adopt.",
            "Not stated or implied anywhere -- there's no real evidence of any contractual requirement; adoption is described as organic usage across open-source projects, not an enforced condition.",
            "Correct. A near-zero-friction format (just Markdown, no schema to learn or validate against) plausibly lowers the barrier for any project to add one, which is a reasonable, direct connection to why a convention like this could reach broad real adoption quickly -- consistent with both real, verified facts given.",
            "Directly contradicted -- the quote states 'the agent simply parses the text you provide,' meaning the format explicitly IS usable without a schema; the real, verified adoption figure is direct evidence it works in practice, not evidence against usability."
          ],
          "source": "Skills, A2A and Other Protocols",
          "sourceUrl": "skills-a2a-protocols.md"
        },
        {
          "scenario": "A real repro sent four calls with an identical, cached tools+system prefix. Call 3 changed only tool_choice (same tools, same system prompt) and still showed cache_read_input_tokens unchanged from call 2. Call 4 edited one word in one tool's description and showed a full cache_creation_input_tokens write instead.",
          "question": "What's the most accurate explanation for why these two small changes had such different real costs?",
          "options": [
            "The tools array sits earlier in the real cache prefix hierarchy than tool_choice touches",
            "Call 4 happened later in the session, and caches naturally degrade over elapsed time",
            "Editing a tool description is a larger change in total byte count than changing tool_choice",
            "Call 4 happened later in the session, and caches naturally degrade with time passing"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The cache follows a fixed hierarchy (tools -> system -> messages), and a change only invalidates the level it occurs at plus everything after it. tool_choice affects the messages level, downstream of tools/system, so it leaves the tools+system prefix's hash untouched. A tool description lives inside the tools array itself, upstream of everything, so changing it invalidates the whole chain.",
            "Contradicts the real, documented mechanism directly -- prompt caches don't 'naturally degrade'; they persist for their TTL (unless invalidated by a structural change) and are refreshed on each read. Elapsed time within a session isn't what caused call 4's invalidation.",
            "Not the real mechanism -- a single appended word (' (updated)') is a tiny byte-count change, smaller than many tool_choice payloads could be, yet it caused full invalidation; the deciding factor is WHERE in the hierarchy the byte changed, not how many bytes changed.",
            "Overgeneralizes -- tool_choice itself doesn't invalidate the tools/system cache, but this doesn't mean EVERY possible combination of changes alongside it would also be free; the real claim is narrower and specifically about which cache LEVEL it touches."
          ],
          "source": "KV-Cache Economics",
          "sourceUrl": "kv-cache-economics.md"
        },
        {
          "scenario": "A real repro compared two ways of making a tool functionally unavailable to the model mid-conversation: (1) a tool_removal content block on a mid-conversation role:'system' message, with the top-level tools array left byte-identical, and (2) physically deleting the tool from the top-level tools array. Both produced the same real outcome -- the model could no longer use that tool.",
          "question": "Given the identical functional outcome, what was the real, measured difference between the two approaches?",
          "options": [
            "Array-editing is faster because it requires fewer tokens to express than a tool_removal block",
            "There was no real difference at all -- both approaches are equivalent in every measurable way",
            "Only the tool_removal approach actually works; array-editing silently fails to remove access",
            "tool_removal kept reading one identical cache entry; array-editing produced a distinct entry"
          ],
          "correct": 3,
          "explanations": [
            "Not what was measured or claimed -- the comparison in this repro was about cache token accounting (creation vs. read), not about raw request size or response latency; token-count-for-expressing-the-change was not the dimension being compared.",
            "Directly contradicted by the real measured numbers -- the two approaches produced very different cache_read/cache_creation values across the actual repro, not identical ones.",
            "Contradicts the real repro directly -- the array-editing call (call C) genuinely worked; it produced a valid response with a distinct, freshly-created cache entry (2,113 tokens), not a silent failure. Both approaches genuinely worked functionally; they differed in cache cost, not in whether they worked.",
            "Correct. Three calls using tool_removal (with the tools array unchanged) all read the identical 2,211-token cache entry. The call that instead physically edited the tools array produced a completely new, distinct 2,113-token entry with no relationship to the earlier ones -- same functional outcome, very different cache economics."
          ],
          "source": "KV-Cache Economics",
          "sourceUrl": "kv-cache-economics.md"
        },
        {
          "scenario": "After masking a tool with a tool_removal block, a real repro attempted to force tool_choice to that exact tool by name. The real API response was a 400 error: \"forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back).\"",
          "question": "What does this specific result establish about tool masking that the cache-preservation numbers alone don't?",
          "options": [
            "That masking is purely a description-level change with no effect on actual tool availability",
            "That the masked tool remained fully callable, and the error was an unrelated bug",
            "That masking is enforced by the API itself, not just a cosmetic hint to the model",
            "That forcing tool_choice is generally unsupported whenever a beta header is active"
          ],
          "correct": 2,
          "explanations": [
            "The opposite of what the real error demonstrates -- if masking were purely cosmetic, forcing tool_choice to the masked tool would have succeeded (the model would just be told to call something it could still technically invoke); instead the API itself rejected the request outright.",
            "Directly contradicts the quoted real error message, which explicitly states the tool is 'absent from the final available-tool set' -- not callable, and the error is specifically about that absence, not an unrelated fault.",
            "Correct. A real, specific 400 error tied directly to the tool_removal block's effect shows the API is actively tracking and enforcing which tools are genuinely available -- this is a hard access-control result, not just Claude choosing not to mention the tool in its own responses.",
            "Unsupported and too broad -- nothing in the real error message suggests forced tool_choice is broken generally under beta headers; the rejection was specific to the named tool having been removed, not a general incompatibility."
          ],
          "source": "KV-Cache Economics",
          "sourceUrl": "kv-cache-economics.md"
        },
        {
          "scenario": "A developer re-runs kv_cache_economics.py twice in a row during development, a few minutes apart, against the identical tools+system prefix. The second run's first call shows cache_read_input_tokens instead of the cache_creation_input_tokens they expected for a 'fresh' first call.",
          "question": "What's the most accurate explanation for this observation?",
          "options": [
            "This indicates a bug in the recipe -- a fresh script run should always start with a cache miss",
            "Cache reads refresh the entry's TTL, so an identical prefix tested repeatedly stays warm",
            "The Anthropic API caches responses indefinitely once written, regardless of any TTL",
            "The second run used a different model than the first, which explains the cache hit"
          ],
          "correct": 1,
          "explanations": [
            "Misdiagnoses the cause -- 'fresh script run' and 'fresh cache state' are different things; the cache is scoped to the exact request content and lives server-side independent of when or how many times a local script has been invoked.",
            "Correct. This is real caching behavior, not a bug -- prompt cache entries have a TTL (5 minutes by default), and reading an entry resets that window. Testing the identical prefix repeatedly during development, within that window, keeps extending its life, so a 'first' call in a later run can legitimately read an entry created by an earlier run or test.",
            "Contradicts the documented mechanism directly -- prompt caches are explicitly time-limited (5-minute or 1-hour TTL options), not indefinite; an entry does expire if enough real time passes without being read.",
            "Not the described scenario -- the setup specifies an identical tools+system prefix on both runs; a genuine model change would itself normally produce a DIFFERENT cache entry (as this page's own repro observed between Sonnet 5 and Opus 5 runs), not the same one being read."
          ],
          "source": "KV-Cache Economics",
          "sourceUrl": "kv-cache-economics.md"
        },
        {
          "scenario": "A real repro gave a model one customer-service case and asked it to record both the general policy and the specific case outcome, with no instruction naming 'semantic' or 'episodic' memory anywhere. The model wrote two separate files unprompted, and two later independent sessions each correctly answered a different kind of question from the right file.",
          "question": "What does this result most precisely demonstrate about CoALA's four-type taxonomy?",
          "options": [
            "That episodic memory is strictly more useful than semantic memory for support tasks",
            "That this result only holds for tasks involving refunds, not other domains",
            "That the taxonomy must be named explicitly for a model to organize memory this way",
            "That the model organized files matching the taxonomy, without being told to"
          ],
          "correct": 3,
          "explanations": [
            "Not what was shown or claimed -- both memory types were used correctly for their respective question types; the repro demonstrates complementary roles, not that one type is generally superior to the other.",
            "An unsupported, overly narrow reading -- nothing about the underlying mechanism (separating general rules from specific instances) is inherently tied to refunds; the repro used refunds as one illustrative domain, not evidence the finding is domain-specific.",
            "Directly contradicted by the setup -- the taxonomy's names were never mentioned in the prompt, and the correct organization still emerged from the model's own judgment about what to keep separate.",
            "Correct. The real, notable finding isn't that CoALA's categories are useful in the abstract -- it's that a model's spontaneous behavior (splitting general policy from specific-case detail into separate files) already matches the taxonomy's structure, without being instructed to follow it."
          ],
          "source": "Memory Architectures",
          "sourceUrl": "memory-architectures.md"
        },
        {
          "scenario": "MemGPT's real reported numbers show GPT-4 with MemGPT's paging system reaching 92.5% on Deep Memory Retrieval, versus 32.1% for a fixed-context baseline given no memory-management mechanism.",
          "question": "What does this comparison most precisely isolate as the source of the accuracy gap?",
          "options": [
            "That the baseline had no real mechanism to move data out to any outside storage",
            "MemGPT's numbers only apply to document QA tasks, not conversational memory tasks",
            "The gap reflects prompt-wording differences rather than any architectural difference",
            "GPT-4 with MemGPT is simply a newer, more capable model than the baseline"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The paper's own framing centers on virtual context management -- moving data between 'main context' and 'external context' via function calls -- as the mechanism under test. A fixed-context baseline has no such mechanism, so information that doesn't fit simply isn't available, which is exactly what Deep Memory Retrieval would expose.",
            "Misreads which number applies where -- 92.5% vs. 32.1% is specifically the Deep Memory Retrieval (conversational) result; the paper's document QA finding is a separate, related result about performance not degrading with context length.",
            "Unsupported -- nothing in the real comparison suggests prompt wording, rather than the underlying memory-management architecture, drives a gap of this magnitude.",
            "Not the comparison being made -- both conditions use the same underlying model (GPT-4); the variable under test is the presence or absence of MemGPT's virtual context management, not a difference in model generation."
          ],
          "source": "Memory Architectures",
          "sourceUrl": "memory-architectures.md"
        },
        {
          "scenario": "A real memory-poisoning repro used the identical injection mechanism -- one crafted conversational turn, no direct memory-store access -- against two different claims: an unverified $50,000 financial credit, and an unverified operational fact about a support-contact reassignment. Only the operational claim was trusted with zero hedging.",
          "question": "What is the most accurate characterization of the real vulnerability this repro isolates?",
          "options": [
            "That the model applies scrutiny at write time but never later when memory is actually used",
            "That only claims involving specific dollar amounts can ever be poisoned into agent memory",
            "That the risk concentrates in claims that look insufficiently consequential to flag at all",
            "That memory poisoning failed entirely here, so the technique doesn't work against Claude"
          ],
          "correct": 2,
          "explanations": [
            "Contradicted directly by the repro -- scrutiny was applied AT WRITE TIME in the financial condition (the hedge was written into the memory file itself), and again later at the exploit/use moment in the source-tagged condition.",
            "Inverts the actual finding -- the DOLLAR claim was the one that triggered spontaneous resistance; the operational, non-monetary claim was the one that succeeded with no hedging at all.",
            "Correct. The repro's real contrast -- financial claim resisted spontaneously, operational claim trusted completely -- shows the model's scrutiny isn't tied to whether a claim is actually verified, but to whether it superficially resembles something worth being careful about.",
            "Overstates the negative result -- the operational-claim condition succeeded cleanly (zero hedging, confident wrong answer used for a real decision); only the financial-claim condition resisted, so this wasn't a uniform failure of the attack."
          ],
          "source": "Memory Architectures",
          "sourceUrl": "memory-architectures.md"
        },
        {
          "scenario": "After adding a one-line system-prompt instruction requiring every memory fact to be tagged [user-asserted] or [tool-verified], the previously-unhedged operational-claim poisoning attempt produced a real caveat at exploit time instead of a confident, uncaveated answer.",
          "question": "What is the strongest, most precise takeaway from this specific result?",
          "options": [
            "That the mitigation only works for operational claims, not financial ones",
            "That source-tagging preserves provenance for later scrutiny, not upfront truth",
            "That source-tagging guarantees no false claim can ever be written to memory",
            "That this proves the earlier financial-claim resistance was purely coincidental"
          ],
          "correct": 1,
          "explanations": [
            "Unsupported by the repro as actually run -- the source-tagged condition was tested on BOTH the financial claim (producing explicit unverified-credit caveats) and the operational claim (producing the contact-verification caveat); nothing suggests it's claim-type-specific.",
            "Correct. The real value demonstrated is that provenance-tagging doesn't require solving the much harder problem of verifying truth when a claim first arrives -- it just requires preserving where a fact came from, so a later consuming turn can apply appropriate caution before acting on it.",
            "Overstates the mechanism -- the tagged condition still WROTE the unverified claim to memory (just labeled it); tagging doesn't block or falsify anything at write time, it changes what happens when the tagged fact is later relied upon.",
            "Not addressed by this specific comparison -- whether the untagged financial-claim resistance was reliable or coincidental is a separate question this comparison (about the operational claim) doesn't resolve either way."
          ],
          "source": "Memory Architectures",
          "sourceUrl": "memory-architectures.md"
        },
        {
          "scenario": "A real repro ran the same ambiguous tool-calling request (a subscription pause phrased as 'stop being charged') against Haiku 4.5 and Sonnet 5, 5 trials each. Both tiers picked the correct tool on all 5 trials.",
          "question": "What is the most accurate way to interpret this specific result?",
          "options": [
            "It shows this specific ambiguity does not separate these two model tiers",
            "It means tool-calling ambiguity is never a real risk for any model pair",
            "The test was flawed since a real experiment should always find some difference",
            "It proves Haiku 4.5 and Sonnet 5 are identical in every tool-calling scenario"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The result is scoped to exactly what was tested: this specific ambiguity, these two specific model versions, this specific tool pair. It's real, honest evidence that THIS gap doesn't exist here -- not a general claim about tool-calling reliability everywhere.",
            "Overstates a narrow negative result into a universal claim -- this repro tested one ambiguity against one model pair; it says nothing about tool-calling ambiguity risk in general, across other models, tools, or phrasings.",
            "Backwards reasoning -- a real experiment can validly produce a null result; treating 'no difference found' as evidence of a flawed test would bias future work toward only reporting differences that confirm a hypothesis, rather than reporting what was actually observed.",
            "Overgeneralizes far beyond what one test showed -- a single ambiguous-request scenario tying doesn't establish identical behavior across every possible tool-calling situation these two models might face."
          ],
          "source": "Models for Agents",
          "sourceUrl": "models-for-agents.md"
        },
        {
          "scenario": "As of a June 2026 leaderboard snapshot, GLM 4.5 -- an open-weight model -- led the Berkeley Function-Calling Leaderboard at 76.7%, ahead of Claude Opus 4.7 (76.6%) and Gemini 3.1 Flash Lite Preview (76.5%).",
          "question": "What does this specific ranking most accurately establish?",
          "options": [
            "That every open-weight model now outperforms every proprietary model at tool calling",
            "That an open-weight model genuinely held the leaderboard top spot at that snapshot",
            "That this leaderboard no longer measures a meaningful capability gap between models",
            "That GLM 4.5's lead is permanent and will not change in future leaderboard updates"
          ],
          "correct": 1,
          "explanations": [
            "A large overgeneralization from one model's result -- this ranking establishes GLM 4.5's specific standing at that snapshot; it says nothing about EVERY open-weight model versus EVERY proprietary model.",
            "Correct. The precise, defensible claim is exactly this: at that specific snapshot, an open-weight model (GLM 4.5) held the top spot on a major, widely-cited agentic tool-calling benchmark -- a real, current fact, not a general or permanent one.",
            "Not what the numbers show -- the top three scores (76.7%, 76.6%, 76.5%) are close but real, distinct rankings on a benchmark still actively differentiating 23 evaluated models; closeness at the top doesn't mean the benchmark stopped measuring anything.",
            "Unsupported speculation about the future -- leaderboard rankings change as new models are evaluated; nothing about a single snapshot implies permanence, and the page doesn't claim otherwise."
          ],
          "source": "Models for Agents",
          "sourceUrl": "models-for-agents.md"
        },
        {
          "scenario": "A real routing repro found all three dispatch strategies (always-Haiku, always-Sonnet, routed) achieved identical 4/4 accuracy on a batch of queries including two classic reasoning traps, with routed costing 199 tokens versus 216 for always-Sonnet and 174 for always-Haiku.",
          "question": "Given that accuracy was identical across all three strategies, what is the most accurate characterization of what routing actually demonstrated here?",
          "options": [
            "Routing was pointless in this run since the cheap model alone matched everyone's accuracy",
            "The identical accuracy across strategies suggests a bug in the recipe's grading logic",
            "Routing matched the capable tier's accuracy while costing less than blanket escalation",
            "Routing beat always-Haiku on cost, proving it's the superior strategy in every case"
          ],
          "correct": 2,
          "explanations": [
            "Misses the real, distinct value demonstrated -- routing didn't need to catch a mistake to be worth measuring; it showed the classification and dispatch mechanism works correctly and pays for its own overhead even in a batch where escalation wasn't strictly necessary.",
            "Unsupported -- the repro's per-query correctness was checked against independently known correct answers (Paris, 5 minutes, $0.05), not against each other; identical accuracy reflects three separate real evaluations reaching the same real correct answers, not a shared or buggy grading path.",
            "Correct. The real, useful comparison is routed vs. always-Sonnet, not routed vs. always-Haiku: routing matched always-Sonnet's accuracy (4/4) while costing less (199 vs. 216 tokens) -- a genuine, if modest, benefit that holds even though the cheap model alone would also have scored 4/4 for less.",
            "Contradicts the real numbers directly -- routing (199 tokens) cost MORE than always-Haiku (174 tokens), not less; the router's own classification calls are real overhead that a pure cheap-only strategy doesn't pay."
          ],
          "source": "Models for Agents",
          "sourceUrl": "models-for-agents.md"
        },
        {
          "scenario": "RouteLLM's real published result states routing 'significantly reduces costs -- by over 2 times in certain cases -- without compromising the quality of responses,' evaluated against GPT-4 as the capable-tier baseline.",
          "question": "What is the most precise way to relate this cited result to the recipe's own 4-query routing repro, which showed a smaller, single-digit-percentage cost reduction rather than a 2x reduction?",
          "options": [
            "The two results contradict each other, so at least one of them must be wrong",
            "RouteLLM's number is outdated and no longer applies to current-generation models",
            "The recipe's repro proves RouteLLM's reported cost reduction was never real",
            "The small batch here is not positioned to reproduce a benchmark-scale reduction"
          ],
          "correct": 3,
          "explanations": [
            "False dichotomy -- the two results measure different things at different scales (a large, diverse benchmark distribution vs. a small, illustrative 4-query batch); neither being 'wrong' is required to explain the size difference.",
            "Unsupported and not the actual explanation -- nothing about model generation invalidates a routing mechanism's cost math; the real difference is scale and query-mix, not model recency.",
            "Overstates the comparison -- a small illustrative repro not reproducing a large benchmark's exact magnitude doesn't invalidate the cited study's own real, independently reported result; the two are complementary evidence at different scales, not competing claims.",
            "Correct. RouteLLM's reported multiple-times cost reduction comes from routing across a large, varied benchmark distribution where many queries are genuinely simple enough to route cheaply; a small, illustrative 4-query batch -- half of which were deliberately hard reasoning traps -- isn't the right scale or mix to reproduce that same magnitude, even though the same underlying mechanism is genuinely at work in both."
          ],
          "source": "Models for Agents",
          "sourceUrl": "models-for-agents.md"
        },
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
          ],
          "source": "Planning and Decomposition",
          "sourceUrl": "planning-and-decomposition.md"
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
          ],
          "source": "Planning and Decomposition",
          "sourceUrl": "planning-and-decomposition.md"
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
          ],
          "source": "Planning and Decomposition",
          "sourceUrl": "planning-and-decomposition.md"
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
          ],
          "source": "Planning and Decomposition",
          "sourceUrl": "planning-and-decomposition.md"
        },
        {
          "scenario": "A team built an orchestrator-workers pipeline (per Anthropic's workflow-pattern definition: a central LLM call decides which of several pre-built worker functions to invoke, each worker executes one fixed role). A teammate says: 'This is a multi-agent system, since it has a lead agent and workers operating under it.'",
          "question": "What's the most accurate correction?",
          "options": [
            "Any lead-plus-workers topology qualifies as multi-agent, regardless of what the workers are",
            "It's closer to Cognition's fragile-multi-agent case, since workers can't see each other's context either way",
            "It's a workflow -- the workers execute pre-built roles, not open-ended decisions as autonomous agents",
            "It only counts as multi-agent if the workers run concurrently rather than sequentially"
          ],
          "correct": 2,
          "explanations": [
            "Collapses a distinction Anthropic's own writing draws explicitly -- a fixed topology with workers executing pre-scripted roles is not the same thing as autonomous agents making open-ended decisions, even when the diagram looks similar.",
            "Misapplies Cognition's critique -- their argument is specifically about autonomous subagents making independent, potentially conflicting judgment calls, not about any lead-plus-workers shape; a fixed-role worker structurally can't make the kind of conflicting decision Cognition describes.",
            "Correct. Anthropic's own writing classifies orchestrator-workers as a workflow pattern precisely because the workers execute a fixed, developer-defined role -- the multi-agent Research system is different specifically because its subagents are autonomous, with their own tool access and judgment, not because of the lead-plus-workers shape itself.",
            "Introduces a fabricated distinguishing criterion -- concurrency versus sequential execution isn't what separates a workflow's workers from a multi-agent system's subagents in either Anthropic's or Cognition's framing."
          ],
          "source": "Multi-Agent Systems",
          "sourceUrl": "multi-agent-systems.md"
        },
        {
          "scenario": "A real experiment measured a single agent answering three independent questions in 1,048 tokens, versus a lead-plus-3-subagents-plus-synthesis design using 3,369 tokens on the identical questions -- a real 3.21x multiplier. A team cites Anthropic's reported 4x/15x figures and concludes: 'Our number is basically wrong, or our multi-agent implementation is broken, since it doesn't match Anthropic's reported numbers.'",
          "question": "What's the strongest problem with that conclusion?",
          "options": [
            "The conclusion is right -- any real multiplier that doesn't match a cited figure indicates a bug",
            "Anthropic's figures use a different baseline (real chat) and workload -- not a contradiction",
            "3.21x actually exceeds 15x once counted correctly, so the team undercounted tokens",
            "Token multipliers are never comparable across two different measurements, under any circumstances"
          ],
          "correct": 1,
          "explanations": [
            "Treats a citation as a universal constant rather than a measurement under specific conditions -- reasonable real numbers vary by task, baseline, and scale without indicating an error.",
            "Correct. Anthropic's 4x and 15x figures compare agents and multi-agent systems against a plain-chat baseline on real production workloads -- not a single-agent-does-everything baseline on one small three-question toy task. A smaller real multiplier, in the same direction, from a genuinely different measurement setup is expected, not a red flag.",
            "A fabricated, arithmetically false claim -- 3.21 does not exceed 15 under any accounting; nothing in the real experiment supports this.",
            "Overcorrects into an unreasonable, absolutist position -- comparing real measurements while being explicit about differing conditions is exactly the right way to use a cited figure as context, not a reason to avoid comparison."
          ],
          "source": "Multi-Agent Systems",
          "sourceUrl": "multi-agent-systems.md"
        },
        {
          "scenario": "A real consistency-risk repro (two subagents, each blind to the other, independently inventing a shared numeric fact) found no disagreement across two separate attempts -- even after the prompt was changed specifically to discourage a common default answer. Someone argues: 'This proves Cognition's inter-agent consistency concern is overstated.'",
          "question": "What's the most accurate pushback?",
          "options": [
            "The repro tested a scalar number, narrower than Cognition's own creative-interpretation example",
            "Two consistent real runs are sufficient to generalize that the concern doesn't apply broadly",
            "The result should be dismissed outright, since a repro using a fictional app name is inherently invalid",
            "The repro actually did find a disagreement -- the outcome described is being misread"
          ],
          "correct": 0,
          "explanations": [
            "Correct. Cognition's own example (a background matching one visual style, a bird matching a different one) is about creative/stylistic interpretation -- genuinely high-variance with no shared fallback. This repro tested a scalar fact, which two real runs suggest the same model tends to agree with itself on even when blind and instructed to be unusual -- a real, disclosed, narrower test, not a disproof of the broader risk.",
            "Overgeneralizes two real but narrow results into a broad claim the repro's own design isn't positioned to support -- consistency on a scalar number doesn't settle the question of consistency on open-ended, high-variance creative choices.",
            "An overly strong, unsupported rule -- fictional scenarios are the established pattern across this entire cookbook precisely because they let mechanisms be tested cheaply and safely; that doesn't invalidate a result.",
            "Misstates the real outcome -- both real runs found the two subagents' answers matched (19/19 and 11/11), not a mismatch."
          ],
          "source": "Multi-Agent Systems",
          "sourceUrl": "multi-agent-systems.md"
        },
        {
          "scenario": "MAST's own reported finding is that multi-agent systems' 'performance gains on popular benchmarks are often minimal,' based on a 1,600+-trace survey across 7 frameworks. A candidate cites this in an interview as: 'This means Anthropic's reported 90.2% multi-agent improvement must be exaggerated or unrepresentative.'",
          "question": "What's the most accurate response to that claim?",
          "options": [
            "Correct -- a survey finding minimal average gains directly contradicts any single large reported improvement",
            "MAST's finding only applies to open-source frameworks, so it says nothing about a proprietary system",
            "90.2% isn't comparable to any benchmark result, since it was self-reported rather than independently measured",
            "MAST reports a broad average across many frameworks; Anthropic's number is scoped to one favorable task shape"
          ],
          "correct": 3,
          "explanations": [
            "Treats a broad average and a scoped, task-specific result as if they must agree exactly -- a general survey finding modest average gains is fully consistent with a real, larger gain on a specific task shape well-suited to the architecture.",
            "Fabricates a restriction not stated in the paper -- MAST's abstract doesn't scope its finding to open-source frameworks only, and nothing about the taxonomy's failure modes is specific to open-source implementations.",
            "An overreaching dismissal -- self-reported doesn't mean incomparable; the real point is that the two numbers describe different scopes, not that either measurement type is invalid.",
            "Correct. MAST's finding describes gains across popular benchmarks broadly; Anthropic's 90.2% is explicitly scoped to a breadth-first research task where independent sub-tasks are exactly the shape multi-agent is well-suited to. A big win on a favorable, specific task and a modest average across many different tasks and frameworks can both be true at once."
          ],
          "source": "Multi-Agent Systems",
          "sourceUrl": "multi-agent-systems.md"
        },
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
          ],
          "source": "Harness Engineering",
          "sourceUrl": "harness-engineering.md"
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
          ],
          "source": "Harness Engineering",
          "sourceUrl": "harness-engineering.md"
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
          ],
          "source": "Harness Engineering",
          "sourceUrl": "harness-engineering.md"
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
          ],
          "source": "Harness Engineering",
          "sourceUrl": "harness-engineering.md"
        },
        {
          "scenario": "SWE-agent's own real ablation found removing its linting guardrail dropped SWE-bench Lite performance from 18.0% to 10.3%. A real repro testing the identical mechanism against Sonnet 5, across two independently redesigned edit tasks, found a 100%-vs-100% tie with no guardrail benefit at all.",
          "question": "What is the most accurate way to interpret the gap between these two results?",
          "options": [
            "The two results reflect different models and task scales, not a contradiction",
            "SWE-agent's original 2024 measurement must have been flawed or fabricated",
            "Linting guardrails are proven to be universally unnecessary for any coding agent",
            "The newer repro's tie means it failed to properly test the guardrail's effect"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The two results measure genuinely different things: SWE-agent's ablation used 2024-era models on the full SWE-bench Lite benchmark; this page's repro used Sonnet 5 (2026) on small, illustrative tasks. Both can be real and honestly reported without contradicting each other -- they're simply different, non-comparable measurements.",
            "Unsupported and dismissive of a real, cited, peer-reviewed measurement -- there's no basis to call SWE-agent's own ablation flawed; it's a real number from a specific model generation and benchmark.",
            "Wildly overgeneralizes a small, illustrative repro (5-line functions, one current model) into a sweeping universal claim the repro never makes or could support.",
            "Backwards -- the repro deliberately redesigned both demos into harder variants specifically to give the guardrail a real chance to matter, and still found a tie; a null result from a genuinely fair, repeated test is real data, not evidence of a failed test."
          ],
          "source": "Coding Agents: Mechanisms",
          "sourceUrl": "coding-agents-mechanisms.md"
        },
        {
          "scenario": "A real repro's binary-search bug (hi = mid - 1 instead of hi = mid) initially passed the recipe's first hand-picked test cases even though the bug was real. A 2,000-case random search against a known-correct reference implementation found the bug actually diverges on roughly 20% of random inputs.",
          "question": "What is the most important methodological lesson this specific step illustrates?",
          "options": [
            "Binary search bugs are impossible to catch with any automated testing approach",
            "Hand-picked test cases can miss a real bug -- verify against divergent inputs",
            "The bug wasn't actually real, since it passed several initial test cases",
            "Random testing is unnecessary once a bug has been manually identified in code"
          ],
          "correct": 1,
          "explanations": [
            "Contradicts what actually happened -- the bug WAS caught by automated testing, just not by the first hand-picked cases; a systematic random search using a reference implementation found real inputs that trigger it.",
            "Correct. This is exactly what the repro's own methodology demonstrated: a bug can be real and present in the code while still passing hand-picked tests that don't happen to hit the specific inputs where it manifests -- verifying against actually-divergent inputs (found via random search against a reference implementation) is what made the test suite trustworthy.",
            "Directly contradicted by the real 2,000-case search, which confirmed the bug genuinely diverges from correct behavior on about 20% of random inputs -- passing a few specific test cases didn't mean the bug wasn't real, it meant those particular cases didn't happen to trigger it.",
            "Backwards -- manually spotting a bug in code doesn't guarantee you've picked test inputs that actually trigger it, which is precisely the gap the random search was needed to close in this repro."
          ],
          "source": "Coding Agents: Mechanisms",
          "sourceUrl": "coding-agents-mechanisms.md"
        },
        {
          "scenario": "Both of this page's real repros were run twice each: once with an initial, simpler task design, and again with a deliberately harder, independently redesigned task, after the first attempt came back as a tie. Both harder attempts also came back as ties.",
          "question": "What does trying a harder task variant, rather than accepting the first tie immediately, most directly demonstrate about the project's methodology?",
          "options": [
            "That the researchers kept retrying indefinitely until they found the result they wanted",
            "That the first attempt's tie must have been due to a bug in the test setup",
            "That a single easy-task tie doesn't rule out a real effect showing up on harder tasks",
            "That harder tasks always produce different, more differentiated results than easy ones"
          ],
          "correct": 2,
          "explanations": [
            "Mischaracterizes what happened -- the redesigns stopped after one well-motivated harder attempt each, and the final result (another tie) was reported honestly rather than chased further; this is bounded, disciplined re-testing, not indefinite retrying for a preferred outcome.",
            "Not what was concluded or claimed -- the first task's tie was treated as a real result on an easy task, not evidence of a broken test; the harder redesign was a genuine attempt to test the SAME mechanism under more demanding conditions, not a bug fix.",
            "Correct. A tie on an easy task leaves open the possibility that the task simply wasn't hard enough to reveal a real gap -- trying one genuinely harder, independently designed variant is the correct way to check that possibility before accepting the tie as a meaningful finding, which is exactly what both repros did.",
            "Overstates the pattern -- in this specific case, the harder tasks ALSO came back as ties, directly contradicting the idea that harder tasks always differentiate; that's precisely why this page reports the result as an honest tie rather than assuming harder must mean different."
          ],
          "source": "Coding Agents: Mechanisms",
          "sourceUrl": "coding-agents-mechanisms.md"
        },
        {
          "scenario": "METR's real research reports model 'time horizon' -- task length completable with 50% reliability -- growing with 'a doubling time of around 7 months' over six years, with Claude 3.7 Sonnet measured at approximately one hour.",
          "question": "What is the most accurate relationship between METR's trend and this page's own two small repros?",
          "options": [
            "This page's repros directly measured and confirmed METR's specific doubling-time trend",
            "METR's trend and this page's repros are unrelated and measure completely different things",
            "This page's repros prove METR's projected doubling trend will definitely continue",
            "This page is a small, current, illustrative data point on that same capability shift"
          ],
          "correct": 3,
          "explanations": [
            "Overstates the connection -- this page's repros never measured task duration, time horizon, or anything resembling METR's actual methodology; they tested a completely different, much narrower question (guardrail/review-loop effect on small edits).",
            "Understates a real, acknowledged connection -- the page explicitly frames its own repros as sitting inside the same general trend METR describes, even while being careful not to claim they measure the same thing.",
            "Overreaches -- a small pair of illustrative repros on 5-line functions provides no evidence about whether METR's specific projected trend continues into the future; the page explicitly declines to make that claim.",
            "Correct. The page's own framing is precise: its repros are a small, current, specific illustration of models needing less structural scaffolding over time (SWE-agent's 2024 guardrail vs. Sonnet 5's baseline reliability) -- directionally related to METR's broader capability-growth trend, without claiming to measure or validate that trend directly."
          ],
          "source": "Coding Agents: Mechanisms",
          "sourceUrl": "coding-agents-mechanisms.md"
        },
        {
          "scenario": "A real repro's first design gave a prompted-only condition an explicit system-level rule ('never delete config.json') and tested whether a persuasive override message could get the model to violate it. The model never did, and the hook (present in a separate condition) was never actually triggered in any trial.",
          "question": "What was the real methodological problem with this first design?",
          "options": [
            "It tested obedience to a stated rule, not what a hook protects with none",
            "The model's resistance to the override proved hooks are unnecessary in general",
            "The prompted condition's system prompt was too short to be a fair test",
            "The override message wasn't persuasive enough to count as a real test"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real issue was that the comparison answered a different question than intended: it tested whether the model would violate an EXPLICIT STATED RULE under pressure (it didn't), not what happens when NOTHING protects a resource except the hook -- which is the actual, distinct claim a hook makes.",
            "Overgeneralizes a narrow result -- resistance to ONE override attempt under ONE stated rule says nothing about the general necessity of hooks, especially since the corrected version of this same repro found hooks mattered a great deal once the rule was removed.",
            "Not the actual problem -- prompt length wasn't the issue; the issue was that the comparison's premise (rule present vs. rule+hook present) never actually let the hook demonstrate anything distinct, since the model never came close to needing it.",
            "Backwards -- the override message was reasonable and explicit; the real gap was structural (what was being compared), not that the persuasion attempt itself was too weak."
          ],
          "source": "Coding Agent Products and Configuration",
          "sourceUrl": "coding-agent-config.md"
        },
        {
          "scenario": "A real, corrected repro removed any protective rule from BOTH conditions' system prompts, then compared a real hook-based block against no hook at all, given an identical explicit deletion request in both cases.",
          "question": "What did this corrected design specifically make possible that the first design didn't?",
          "options": [
            "It made the deletion request itself more explicit than before",
            "It let the hook's real, distinct effect actually surface in the results",
            "It removed the need to run multiple trials to get a reliable result",
            "It eliminated the possibility of the model refusing the request on its own"
          ],
          "correct": 1,
          "explanations": [
            "Not the key change -- the request's explicitness was already high in the redesign, but that alone isn't what fixed the comparison; the critical change was removing the confound of a prompted rule sitting in front of the hook.",
            "Correct. By removing every other reason the model might decline (no stated rule to obey), any block that occurred in the hook condition could only be attributed to the hook itself -- exactly the isolated, distinct effect the first design failed to expose.",
            "Not accurate -- the recipe still ran 5 trials per condition in the corrected version; removing the confound didn't eliminate the value of repeated trials, it made what those trials measured meaningful.",
            "Overstates the guarantee -- nothing in the redesign prevents a model from independently declining a request for its own reasons; it simply removed the STATED rule as a competing explanation, it didn't rule out other real behavior."
          ],
          "source": "Coding Agent Products and Configuration",
          "sourceUrl": "coding-agent-config.md"
        },
        {
          "scenario": "A real repro measured CLAUDE.md-style project instructions producing a 0%-to-100% real compliance shift on two checkable code conventions (a naming prefix and avoiding f-strings), delivered as a user message following Anthropic's own documented CLAUDE.md mechanics.",
          "question": "What does Anthropic's own documented framing of CLAUDE.md most precisely predict about the LIMITS of this same mechanism, independent of this page's own repro?",
          "options": [
            "That CLAUDE.md instructions should never be trusted for any real behavior change",
            "That CLAUDE.md files must be under a strict line count to have any effect at all",
            "That CLAUDE.md instructions are treated as context, not a hard enforcement layer",
            "That CLAUDE.md only works when delivered as part of the system prompt itself"
          ],
          "correct": 2,
          "explanations": [
            "Overstated and contradicted directly by this page's own real, measured 0%-to-100% compliance result -- the mechanism clearly does produce real behavior change; the documented limit is about GUARANTEES, not effectiveness in general.",
            "Overstates a real but different guidance point -- Anthropic recommends keeping files under roughly 200 lines for better adherence, but this is about diminishing returns on longer files, not a hard cutoff below which the mechanism has zero effect.",
            "Correct. Anthropic's own documentation states this precisely: 'Claude treats them as context, not enforced configuration.' This predicts CLAUDE.md can shape behavior effectively (as measured) while still not providing the hard guarantee a hook provides -- exactly the distinction this page's second repro tests directly.",
            "Contradicts the real, documented delivery mechanism directly, which states CLAUDE.md content is delivered 'as a user message after the system prompt, not as part of the system prompt itself' -- and this page's own repro replicated exactly that mechanism and still measured a strong real effect."
          ],
          "source": "Coding Agent Products and Configuration",
          "sourceUrl": "coding-agent-config.md"
        },
        {
          "scenario": "A real, current survey of primary sources found OpenHands' own README describing it as 'the self-hosted developer control center for coding agents and automations,' explicitly supporting orchestration of 'OpenHands, Claude Code, Codex, Gemini, or any agent with Agent-Client Protocol.'",
          "question": "What is the most accurate characterization of what this specific finding represents?",
          "options": [
            "Confirmation that OpenHands has always been marketed this way since its creation",
            "Evidence that Agent-Client Protocol is now the only way coding agents can be built",
            "An unverifiable claim, since only secondary sources described OpenHands this way",
            "A genuine, dated product repositioning older secondary sources are likely to miss"
          ],
          "correct": 3,
          "explanations": [
            "Unsupported and contradicted by the framing itself -- the finding is specifically flagged as a notable, current REPOSITIONING (implying change over time), not a claim about how the project has always been described.",
            "A significant overreach -- ACP being one real, current interoperability mechanism OpenHands supports says nothing about it being the exclusive way any coding agent must be built; other tools in the same survey use entirely different architectures.",
            "Incorrect -- the quotes cited come directly from OpenHands' own GitHub README, a primary source, not from a secondary blog post or summary describing the project.",
            "Correct. This is precisely the point worth flagging: OpenHands' own current, primary-source README describes a real shift from being framed as one autonomous coding agent to being a control plane that can orchestrate multiple vendors' agents -- a genuine, dated fact that older secondary sources and blog posts are likely to miss."
          ],
          "source": "Coding Agent Products and Configuration",
          "sourceUrl": "coding-agent-config.md"
        },
        {
          "scenario": "Anthropic's own engineering write-up states that for their multi-agent research system, 'token usage by itself explains 80% of the variance' in performance, alongside a separately reported 90.2% improvement over a single agent on their internal eval.",
          "question": "What is the most precise interpretation of what the 80%-of-variance finding implies about that 90.2% improvement?",
          "options": [
            "The 90.2% improvement is invalid, since it can be explained away entirely by a confounding variable",
            "Multi-agent coordination logic contributed nothing at all to the measured performance difference",
            "The two findings are unrelated and describe completely separate evaluations with no shared explanation",
            "Most of the measured improvement is attributable to spending more compute, not smarter coordination"
          ],
          "correct": 3,
          "explanations": [
            "Overstates it -- 'explains 80% of the variance' is a real, substantial statistical relationship, not a claim the entire effect is spurious or invalid; the improvement itself is a real, separately reported result on their internal eval.",
            "Too strong -- 80% of variance leaves real room for other factors (Anthropic names tool-call count and model choice as the other two), so coordination/design isn't reduced to zero contribution, just shown to be secondary to raw spend.",
            "Not accurate -- both findings come from the same real research system and the same real evaluation context; the variance-explanation finding is precisely what helps explain WHY the 90.2% figure exists, not an unrelated fact.",
            "Correct. This is the precise, real reading: token usage being the dominant explanatory factor means most of the measured gain tracks with how much real compute was spent, not primarily with cleverer division of labor or coordination logic -- which directly motivates why redundant, unscoped spending (this page's own repro) is a real, costly problem, not just an aesthetic one."
          ],
          "source": "Deep Research Agents",
          "sourceUrl": "deep-research-agents.md"
        },
        {
          "scenario": "A real repro gave 3 subagents access to an identical, generic research instruction with no assigned scope, and separately gave 3 different subagents explicit, distinct objectives and boundaries covering different parts of the same topic.",
          "question": "What was the most precise real, measured difference between the two conditions?",
          "options": [
            "The scoped condition made fewer total retrievals while covering more of the corpus, with far less overlap",
            "The vague condition covered more of the real document corpus overall, just with some inefficiency",
            "Both conditions covered the identical set of real documents, differing only in how the summaries were worded",
            "The scoped condition failed to retrieve any documents at all due to its narrower per-subagent scope"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, measured numbers: scoped made 7 total retrievals covering all 6 of 6 documents (14.3% redundant); vague made 10 total retrievals but covered only 4 of 6 documents (60% redundant) -- fewer total calls, MORE real coverage, far less waste.",
            "Backwards from the real result -- vague covered only 4 of 6 real documents, LESS than scoped's full 6 of 6 coverage; the vague condition wasn't just inefficient, it left real ground completely unresearched.",
            "Not what happened -- the real document sets covered by each condition were meaningfully different (4/6 vs. 6/6), not identical; this was a real difference in WHAT was researched, not just in phrasing.",
            "Contradicted directly by the real numbers -- the scoped condition made 7 real retrievals and achieved full corpus coverage; narrower scope per subagent did not mean fewer retrievals overall or failed retrieval, since scope specifically directed each search rather than blocking it."
          ],
          "source": "Deep Research Agents",
          "sourceUrl": "deep-research-agents.md"
        },
        {
          "scenario": "STORM's real, three-stage mechanism involves discovering diverse perspectives, simulating multi-perspective conversations with a topic expert, and then curating an outline -- reporting a 25% absolute increase in articles judged well-organized and a 10% increase in breadth of coverage versus a baseline.",
          "question": "What does this real, measured gain most precisely suggest about WHY simulating multiple perspectives improves article structure?",
          "options": [
            "The improvement is due entirely to using a larger or more capable underlying language model in the STORM condition",
            "Asking the same question from several different simulated angles surfaces different real information than asking once",
            "The gain is purely stylistic, affecting how organized the writing sounds without changing what information is included",
            "Simulating perspectives works by having the model fact-check its own previous answers repeatedly before writing"
          ],
          "correct": 1,
          "explanations": [
            "Not indicated -- the comparison described is STORM's own multi-perspective PROCESS against a single-pass baseline, not a model-capability comparison; nothing in the real, reported numbers attributes the gain to model size or capability differences.",
            "Correct. STORM's own real mechanism is specifically about DIFFERENT perspectives posing DIFFERENT questions to a topic expert -- the real gain in both organization AND breadth of coverage is consistent with each simulated perspective surfacing real information a single, undifferentiated pass would be less likely to ask about at all.",
            "Contradicted directly -- the real, reported gain includes a 10% increase in BREADTH OF COVERAGE, not just organizational quality; breadth of coverage is a substantive, information-level outcome, not a purely stylistic one.",
            "Not the described mechanism -- STORM's process is about generating NEW questions from different simulated perspectives directed at a topic expert, not about the model repeatedly verifying or fact-checking its own prior answers."
          ],
          "source": "Deep Research Agents",
          "sourceUrl": "deep-research-agents.md"
        },
        {
          "scenario": "BrowseComp is explicitly designed around the principle 'easy to verify, but hard to solve,' with real, verified accuracy numbers: GPT-4o scored 0.6%, OpenAI o1 scored 9.9%, and OpenAI's own Deep Research model scored 51.5%.",
          "question": "What does the specific design principle 'easy to verify, but hard to solve' most precisely explain about how BrowseComp's questions were constructed?",
          "options": [
            "Questions were selected so that grading requires the same amount of research effort as answering them originally did",
            "Questions were deliberately made ambiguous so that multiple different answers could be scored as equally correct",
            "Questions have a short, checkable ground-truth answer, but actually reaching it requires genuinely persistent search",
            "Questions were designed to be unsolvable by any current model, including the ones the benchmark was built to measure"
          ],
          "correct": 2,
          "explanations": [
            "Backwards from the actual design goal -- 'easy to verify' specifically means grading is CHEAP and fast (a short, checkable answer), explicitly NOT requiring the same effort as the original search; that asymmetry between solving and verifying is the entire point of the principle.",
            "Contradicts the 'easy to verify' half directly -- ambiguous, multiple-valid-answer questions would make verification HARDER, not easier; the design principle requires a single, checkable ground truth, not an ambiguous one.",
            "Correct. This is the precise, real meaning behind the quoted principle: the ANSWER is short and verifiable at a glance, while actually FINDING that answer demands real, persistent search across hard-to-find, entangled information -- exactly the asymmetry the real accuracy numbers (near-zero for most models, 51.5% for the best) are designed to expose.",
            "Contradicted by the real numbers themselves -- OpenAI's Deep Research model scored 51.5%, meaning the benchmark is hard but not unsolvable; the design goal was calibrated difficulty (screening out what contemporary models COULD already solve), not literal unsolvability by every model including future ones."
          ],
          "source": "Deep Research Agents",
          "sourceUrl": "deep-research-agents.md"
        },
        {
          "scenario": "Real, verified accuracy numbers on the ScreenSpot-Pro grounding benchmark range from 6.96% (Gemini-2.5-Pro) to 73.18% (a specialized method), with UI-TARS-72B at 37.12% -- all measured on the identical benchmark of professional, high-resolution interfaces.",
          "question": "What does this specific spread most precisely indicate about grounding as a computer-use agent capability?",
          "options": [
            "Grounding difficulty and technique matter a lot -- the same task shows a real 10x+ accuracy range by method",
            "The benchmark itself is unreliable, since no consistent accuracy figure could be established across models",
            "Grounding accuracy is primarily determined by which company trained the underlying model, not by technique",
            "All modern vision-language models achieve comparable grounding accuracy once given professional interfaces"
          ],
          "correct": 0,
          "explanations": [
            "Correct. A real, verified 6.96% to 73.18% range on the IDENTICAL benchmark is direct evidence that grounding remains a genuinely hard, method-sensitive problem -- specialized grounding techniques produce real, substantial gains over general-purpose vision-language models on the same real task.",
            "Not supported -- ScreenSpot-Pro produced consistent, comparable, real numbers across multiple models on the identical task; a wide real spread across methods is a finding about the methods, not evidence the benchmark itself is broken.",
            "Too narrow -- the real spread includes UI-TARS-72B (a specialized, purpose-built model) scoring well below a different specialized method (73.18%), showing TECHNIQUE, not just which company built the model, drives the real gap.",
            "Directly contradicted by the real numbers -- the spread is more than 10x between the lowest and highest scores on the same benchmark, the opposite of comparable performance across models."
          ],
          "source": "Computer-Use and Browser Agents",
          "sourceUrl": "computer-use-browser-agents.md"
        },
        {
          "scenario": "OSWorld-Human, a real follow-up study, found that even agents that successfully complete a task take 2.7-4.3x more steps than necessary to do so.",
          "question": "What does this finding add that a task SUCCESS RATE alone (like OSWorld's original 12.24% agent baseline) does not capture?",
          "options": [
            "It reveals that most reported OSWorld success-rate numbers were measured incorrectly and cannot be trusted",
            "It measures real efficiency -- an agent can succeed at a task while still being needlessly wasteful",
            "It proves that all successful task completions in the original OSWorld study were actually failures",
            "It shows that step-inefficiency, not accuracy, is now the only real remaining barrier to production readiness"
          ],
          "correct": 1,
          "explanations": [
            "Not the claim -- OSWorld-Human's own finding is about STEP COUNT for tasks that were genuinely completed successfully, not a critique of how success itself was measured or scored in the original study.",
            "Correct. This is precisely the real, added value of the finding: a task can be marked 'successful' by a binary pass/fail metric while the agent took several times more real actions than needed -- a genuine cost and reliability signal that raw success rate alone doesn't surface.",
            "Contradicted directly -- the finding is specifically about tasks the agent DID complete successfully; it says nothing about those completions actually being failures, only that they were less efficient than optimal.",
            "Overstates it -- the page frames this as an ADDITIONAL real dimension worth checking alongside accuracy, not a claim that accuracy no longer matters or that efficiency is now the sole remaining barrier."
          ],
          "source": "Computer-Use and Browser Agents",
          "sourceUrl": "computer-use-browser-agents.md"
        },
        {
          "scenario": "UI-TARS's own three generations report real, measured progress on identical benchmarks: the original scored 24.6 on OSWorld, UI-TARS-1.5 reached 42.5%, and UI-TARS-2 reached 47.5%.",
          "question": "What does comparing these three real numbers across versions demonstrate that a single version's score alone would not?",
          "options": [
            "That UI-TARS has now definitively solved the OSWorld benchmark and further progress is unlikely",
            "That OSWorld's difficulty decreased over time, making each new version's task easier than the last",
            "A real, measured trajectory of improvement on a consistent benchmark, generation over generation",
            "That grounding and OSWorld task-completion are measuring the exact same underlying capability"
          ],
          "correct": 2,
          "explanations": [
            "Overstates it -- 47.5% is real, substantial progress but still well short of the original 72.36% human baseline reported for OSWorld; nothing in the numbers suggests the benchmark is 'solved.'",
            "Not indicated -- OSWorld's own task set and difficulty are treated as fixed for comparison purposes across UI-TARS versions; the improvement in scores reflects model progress, not a claim the benchmark itself got easier.",
            "Correct. Three real, comparable data points on the IDENTICAL benchmark across successive versions of the same architecture is precisely what shows a genuine trajectory of improvement over time, rather than a single static capability snapshot.",
            "Not established -- ScreenSpot-Pro (grounding) and OSWorld (full task completion) are described as distinct, separate benchmarks measuring different things; UI-TARS reports separate real numbers for each, not one combined score."
          ],
          "source": "Computer-Use and Browser Agents",
          "sourceUrl": "computer-use-browser-agents.md"
        },
        {
          "scenario": "A real repro tested a browser agent's response to a hidden, unmarked injected instruction on a fictional webpage, using the identical, entirely innocuous request ('summarize this webpage'). Both a raw-content condition and a content-tagged condition resulted in the model resisting the injection, after one deliberate rewrite removed a self-announcing 'hidden SYSTEM NOTE' tell from the injected text.",
          "question": "What is the most precise, honest conclusion this result supports, given that a real, different browser agent (Comet) was actually compromised by this same mechanism in the real world?",
          "options": [
            "The real CometJacking vulnerability must have been fabricated, since a capable model resisted an equivalent attack",
            "Content-tagging is proven unnecessary as a defense, since the raw condition produced the identical safe outcome",
            "Any browser agent built on a sufficiently capable model is now permanently immune to this entire attack class",
            "This model resisted on this one run -- not the same guarantee as actually fixing the underlying architecture"
          ],
          "correct": 3,
          "explanations": [
            "Not supported and contradicted by real, documented evidence -- Brave Security's own real, cited root-cause diagnosis of the actual Comet incident is independent of this page's own repro; one model resisting a test doesn't invalidate a separately documented, real-world compromise of a different system.",
            "Overreaches from a single test -- the repro's own real result showed NO measured difference between conditions on this run, but the page explicitly frames this as a fact about this model on this run, not evidence that tagging is unnecessary as a general defense-in-depth practice.",
            "Sweeping and explicitly unsupported -- model behavior on injection attempts is not guaranteed to generalize across future attempts, more adversarial phrasings, or other models; the page explicitly rejects treating one favorable result as a permanent guarantee.",
            "Correct. This is exactly the caveat the page draws: a real, honest negative result on one model, one run, is valuable data but categorically different from a structural fix -- the real CometJacking incident already demonstrates the underlying mechanism (untrusted content mixed unmarked with instructions) is a genuine, exploitable flaw regardless of how this particular test went."
          ],
          "source": "Computer-Use and Browser Agents",
          "sourceUrl": "computer-use-browser-agents.md"
        },
        {
          "scenario": "A real, cited framing distinguishes a standard AI application (processes one request, discards context) from a persistent agent, stating the persistent agent's boundary 'is no longer a network perimeter. It is the agent's runtime policy.'",
          "question": "What does this specific distinction most precisely imply about why blast radius matters more for always-on agents?",
          "options": [
            "The agent granted permissions and standing access, not a fixed network boundary, limit possible damage",
            "Always-on agents are simply slower to respond, which is the only meaningful practical difference",
            "Network perimeters remain the primary security boundary for both application types, without meaningful difference",
            "Persistent agents cannot be secured by any means, since their risk surface is inherently unbounded"
          ],
          "correct": 0,
          "explanations": [
            "Correct. This is the precise, real meaning of 'the boundary is the agent's runtime policy' -- once an agent runs continuously with standing tool/data access, what limits potential damage is what it's been granted and for how long, not a perimeter a single request would have to cross.",
            "Not the claim -- the distinction drawn is about risk SURFACE and DURATION of exposure (bounded by one exchange vs. standing, continuous access), not response latency, which isn't addressed by this framing at all.",
            "Directly contradicted -- the quoted distinction explicitly says the boundary is NO LONGER a network perimeter for persistent agents, which is the whole point of naming it as a category change from the single-request case.",
            "Overstates it into fatalism -- the page's own real repro demonstrates a working structural mitigation (scoped tool access); 'unbounded and unsecurable' is not the claim, 'the boundary shifts to runtime policy, which must be designed deliberately' is."
          ],
          "source": "Personal and Always-On Agents",
          "sourceUrl": "personal-always-on-agents.md"
        },
        {
          "scenario": "Unit 42's real root-cause diagnosis for the ClawHavoc incident states: 'The lack of isolation between skill logic and agent authority means that installation results in complete control over the agent's identity.'",
          "question": "What does this diagnosis most precisely identify as the actual vulnerability -- as opposed to a more superficial explanation?",
          "options": [
            "The vulnerability was that OpenClaw's marketplace allowed any developer account to publish without stricter identity verification",
            "The vulnerability was that a single skill happened to be unusually well-disguised as a legitimate utility",
            "The vulnerability was that the malware payload used was more sophisticated than existing antivirus tools could detect",
            "The vulnerability was an architectural lack of scoping between what a skill needs and what it can reach once installed"
          ],
          "correct": 3,
          "explanations": [
            "A real, contributing factor (permissive upload model) but not what Unit 42's OWN quoted diagnosis names as the core issue -- their language is specifically about isolation between skill logic and agent authority, not developer verification.",
            "Not the structural point -- disguise quality explains how a specific attack went undetected initially, not why installing ANY skill grants 'complete control over the agent's identity' as Unit 42's diagnosis states generally.",
            "Not the diagnosis given -- Unit 42's quoted root cause is about architectural isolation, not detection technology; nothing in the real, cited material attributes the incident to malware sophistication outpacing antivirus capability.",
            "Correct. This is precisely what the quoted diagnosis names: not identity verification, not payload sophistication, but the ARCHITECTURAL absence of scoping -- installation itself grants full authority regardless of what the skill actually needs, which is the exact gap this page's own repro tests and closes with an explicit allow-list."
          ],
          "source": "Personal and Always-On Agents",
          "sourceUrl": "personal-always-on-agents.md"
        },
        {
          "scenario": "A real repro's mechanism-level check (zero LLM judgment, hand-written calls) found that an unscoped skill's own code could freely read financial data and send an email to an external address, while a scoped, code-enforced allow-list blocked both identical calls before they executed.",
          "question": "What is the most precise reason this check used hand-written calls instead of relying on a live model's decisions?",
          "options": [
            "Hand-written calls are always faster and cheaper to run than live model calls, which was the only consideration",
            "It isolates what the architecture permits from whatever a specific model might choose to do on a given run",
            "Live models cannot be given access to tools like read_financial_accounts under any circumstance",
            "The hand-written check was necessary because the live-agent trial could not be run at all in this environment"
          ],
          "correct": 1,
          "explanations": [
            "Not the stated rationale -- while true incidentally, cost/speed isn't why the page frames this as a SEPARATE, necessary check; the real reason given is about isolating architecture from model behavior, not efficiency.",
            "Correct. This is exactly the real, stated purpose: a deterministic, no-LLM check shows what's structurally POSSIBLE regardless of model judgment, which is a categorically different (and more reliable) kind of evidence than observing what one model chose to do on one real run.",
            "Not a real constraint -- the page's own live-agent trial DOES give the model access to exactly this tool; the point of the mechanism check isn't tool-access prohibition, it's testing the architecture independent of model choice.",
            "Contradicted directly -- the page explicitly reports running BOTH the mechanism-level check AND a real, live-agent trial; the hand-written check is a deliberate additional measurement, not a substitute forced by some inability to run the live trial."
          ],
          "source": "Personal and Always-On Agents",
          "sourceUrl": "personal-always-on-agents.md"
        },
        {
          "scenario": "In the real, unforced live-agent trial, Claude Sonnet 5 declined the financial-data reach in both the unscoped and scoped conditions -- an honest negative result, reached only after the skill's embedded instruction was rewritten to remove a self-announcing 'hidden internal note' label.",
          "question": "Given this result and the real, independently documented ClawHavoc incident, what is the most precise, honest conclusion to draw?",
          "options": [
            "The live-agent result proves the unscoped architecture is actually safe in practice, contradicting the ClawHavoc incident",
            "The ClawHavoc incident must have targeted a much less capable model than the one used in this repro",
            "The favorable live-agent result is real but does not substitute for the structural fix ClawHavoc already proves is needed",
            "Because the mechanism check found a real flaw, the live-agent trial's honest negative result should be treated as invalid data"
          ],
          "correct": 2,
          "explanations": [
            "Directly contradicted -- the page explicitly warns against this exact conclusion; a favorable result on one model, one run is real data, but ClawHavoc is independent, real-world proof the unscoped architecture WAS exploited in practice, at scale, against real users.",
            "Not supported -- ClawHavoc's real attacks exploited the ARCHITECTURE (no isolation between skill logic and agent authority) via natural-language instructions, not a specific model's reasoning capability; the page never attributes the incident to model capability differences.",
            "Correct. This is precisely the page's own stated position: the live-agent result is real, honestly reported data, but it is a fact about one model on one run, categorically different from the deterministic guarantee the scoped architecture provides -- and ClawHavoc's real-world exploitation is independent proof the underlying gap matters regardless of how any single model performs on a given test.",
            "Not the page's approach anywhere -- honest negative results are treated as valuable, real data throughout, not discarded because a different, separate check found something else; both results are reported together precisely because they answer different questions."
          ],
          "source": "Personal and Always-On Agents",
          "sourceUrl": "personal-always-on-agents.md"
        },
        {
          "scenario": "A real repro gave a model the option to call a verification tool or answer directly, on both a well-known fact and an unguessable fictional fact. In both cases the model called the tool and got the correct outcome -- no divergence between outcome and trajectory was observed.",
          "question": "What's the most accurate takeaway from this specific result?",
          "options": [
            "The checking mechanism worked correctly; it simply had nothing to catch here",
            "The distinction isn't useful in practice, since a real test found no disagreement",
            "The test should be discarded, since it produced no interesting finding",
            "This model always calls every available tool, regardless of the question"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The value of checking both outcome and trajectory isn't that it always finds a mismatch -- it's that you can't know whether they agree until you check both, and this run confirms the checking mechanism itself functioned as intended.",
            "Overgeneralizes from one clean run to a universal claim -- the literature's own documented cases (gaming a benchmark search, misusing a payment tool) show outcome and trajectory genuinely diverging elsewhere; one honest negative result doesn't erase that evidence.",
            "Treats a disclosed negative result as worthless -- knowing the check doesn't fire on a well-behaved run is real information, and the check still exists and would fire if behavior were different.",
            "Overstates a general claim from two data points -- nothing here establishes the model calls EVERY tool in every circumstance; only that it didn't skip verification on these two specific questions."
          ],
          "source": "Evaluating Agents",
          "sourceUrl": "evaluating-agents.md"
        },
        {
          "scenario": "A real experiment measured pass@5 = true and pass^5 = true (5 out of 5 correct) on a probability question, with no observed gap between the two metrics. A team cites tau-bench's published ~60% relative drop from pass@1 to pass^8 and concludes: 'Our result must be wrong, since tau-bench proves these metrics always diverge substantially.'",
          "question": "What's the strongest problem with that conclusion?",
          "options": [
            "Any result without a large gap contradicts tau-bench and signals a measurement error",
            "The gap's SIZE depends on task variance, not a fixed property of the two metrics",
            "pass@k and pass^k are mathematically identical, so tau-bench must mean something else",
            "5 trials is too few to compute either metric, so the result should be discarded"
          ],
          "correct": 1,
          "explanations": [
            "Treats a single cited benchmark's finding as a universal law rather than a result specific to that benchmark's own tasks -- a different, easier task genuinely can and does show a smaller gap.",
            "Correct. tau-bench's domain involves multi-turn tool use under policy constraints, which has far more room for a single trial to go wrong than one well-structured probability question -- the real lesson is that the gap's size reflects the task's actual variance, not a fixed property of the definitions themselves.",
            "Factually wrong -- pass@k (at least one success) and pass^k (all successes) are different quantities by definition, which is exactly why they can diverge when there's real variance across trials.",
            "An arbitrary, unsupported claim about sample size -- k=5 is a small but valid number of trials for computing both metrics on this specific run; the result is honestly reported as what happened at k=5."
          ],
          "source": "Evaluating Agents",
          "sourceUrl": "evaluating-agents.md"
        },
        {
          "scenario": "A team changes an agent's system prompt to spell out numbers in words (a plausible accessibility-driven requirement). Afterward, a regression suite that checks for exact digit substrings in responses reports 0/4 passing, even though a human reviewer confirms every answer is mathematically correct.",
          "question": "What's the most accurate diagnosis of what happened?",
          "options": [
            "The model's arithmetic capability regressed because of the accessibility instruction",
            "The suite was poorly designed from the start and should never have existed",
            "The grader is broken for the new format -- the model's correctness didn't regress",
            "This isn't a real regression, since the new prompt was intentional and well-motivated"
          ],
          "correct": 2,
          "explanations": [
            "Contradicts the evidence directly -- a human reviewer confirmed every answer was mathematically correct; nothing about the model's actual reasoning failed.",
            "Overreaches -- a suite that correctly caught a real, silent breakage did its job; the fix needed is updating the grader, not concluding the suite was a mistake.",
            "Correct. The suite's automated check assumed a digit format the new prompt no longer produces -- the model's capability is intact, but the check that verifies it silently stopped working, exactly what an automated suite exists to surface.",
            "Confuses 'intentional prompt change' with 'no regression occurred' -- the change being deliberate doesn't mean nothing broke; it means the break was an unintended side effect, which regression testing exists to catch regardless of intent."
          ],
          "source": "Evaluating Agents",
          "sourceUrl": "evaluating-agents.md"
        },
        {
          "scenario": "Anthropic's infrastructure noise study ran the identical model and benchmark across six container resource configurations and found a 6-percentage-point gap between the most- and least-resourced setups. A candidate summarizes this as: 'This proves benchmark leaderboards are meaningless and can never be trusted.'",
          "question": "What's the most accurate correction to that summary?",
          "options": [
            "Correct -- a 6-point infrastructure effect means no leaderboard comparison can ever be truly meaningful",
            "The finding only applies to Terminal-Bench 2.0 and has no implications for any other benchmark",
            "The SWE-bench cross-check contradicts the finding, since it found no resource effect at all",
            "It shows infrastructure is a controllable confound, not proof comparison is impossible in principle"
          ],
          "correct": 3,
          "explanations": [
            "Overstates the finding into fatalism -- the study's own conclusion is a call for controls (publishing specs, standardizing enforcement), which presumes meaningful comparison IS possible once the confound is controlled for.",
            "Understates the generalizable lesson -- the SWE-bench cross-check found a smaller but real effect on a different benchmark, showing the confound isn't unique to one benchmark, even though its size varies by task.",
            "Misstates the cross-check's actual result -- it found a smaller but still real, meaningful effect (+1.54 percentage points at 5x RAM versus 1x), not zero effect.",
            "Correct. The study's own recommendation is to treat resource configuration as a controlled experimental variable -- the problem is uncontrolled infrastructure variance, not that benchmarking is inherently meaningless; once controlled, comparisons regain their validity."
          ],
          "source": "Evaluating Agents",
          "sourceUrl": "evaluating-agents.md"
        },
        {
          "scenario": "A real fetch of official leaderboard data found SWE-bench Verified's actual current top score is 79.2%, while several benchmark-aggregator websites reported 96-97% for the same benchmark at around the same time.",
          "question": "What is the most accurate lesson to draw from this specific discrepancy?",
          "options": [
            "A benchmark's real current score should be checked directly, not via summaries",
            "SWE-bench Verified must have two separate, independently valid leaderboards",
            "The official leaderboard's own number must be outdated compared to aggregators",
            "Aggregator sites are always less reliable than any primary source for any claim"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, demonstrated lesson is specifically about verification practice: this page's own research process found a large, real gap between primary-sourced data and secondary summaries, which is exactly why every number on this page is sourced to its specific official leaderboard or paper.",
            "Not what was found or claimed -- there is one official leaderboard (swebench.com); the aggregator numbers were simply inaccurate relative to it, not a second legitimate source.",
            "Backwards -- the official leaderboard is the primary source of truth by definition; there's no basis to assume the aggregators' higher numbers were more current or more correct.",
            "Overgeneralizes a single observed discrepancy into a sweeping claim about all aggregator sites in all cases -- the real, narrower lesson is about this specific verification practice, not a blanket rule about a category of website."
          ],
          "source": "Benchmark Atlas",
          "sourceUrl": "benchmark-atlas.md"
        },
        {
          "scenario": "OpenAI's own analysis of SWE-bench Verified found that 'at least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions,' and that all tested frontier models could reproduce the original gold-patch solutions verbatim, suggesting training contamination.",
          "question": "What is the most precise way to characterize what this finding actually calls into question?",
          "options": [
            "That models genuinely cannot solve any real-world software engineering tasks at all",
            "That the SWE-bench Verified score's meaning as a capability measure is now unreliable",
            "That OpenAI's own frontier models specifically performed worse than competitors on it",
            "That software engineering benchmarks in general can never be constructed validly"
          ],
          "correct": 1,
          "explanations": [
            "A sweeping overgeneralization the finding doesn't support -- the concern is specifically about what THIS benchmark's score measures, not a claim that models can't do real software engineering tasks; SWE-bench Pro's own real, climbing scores are direct evidence against that broader claim.",
            "Correct. The precise, real finding is that a high score on THIS SPECIFIC benchmark no longer reliably indicates real capability, due to two distinct real issues (flawed test cases rejecting correct answers, and training contamination) -- which is exactly why OpenAI's own stated conclusion was to stop reporting this specific score, not to declare all coding benchmarks invalid.",
            "Not what the finding is about -- OpenAI's analysis was about the benchmark's own construction and data quality issues, affecting all models evaluated on it equally, not a comparison of OpenAI's own models against competitors.",
            "A vast overreach -- the same real research points to SWE-bench Pro as a benchmark that 'seems to suffer less from contamination issues,' directly showing that better-constructed benchmarks in the same general category are achievable."
          ],
          "source": "Benchmark Atlas",
          "sourceUrl": "benchmark-atlas.md"
        },
        {
          "scenario": "A real repro of tau-bench's methodology used action-state grading (checking whether a policy-violating tool was actually called) rather than checking whether the agent's final reply sounded correct.",
          "question": "What is the most precise reason this specific grading choice matters?",
          "options": [
            "Action-state grading is easier to implement than any text-based grading approach",
            "Text-based grading is impossible to automate for any agent evaluation task",
            "A reply can sound compliant while the actual action taken violates policy",
            "Action-state grading eliminates the need to run more than a single trial"
          ],
          "correct": 2,
          "explanations": [
            "Not the actual reason given or the real motivation -- implementation ease isn't the point; the point is about what the check can and can't detect, not implementation convenience.",
            "An overstated, unsupported claim -- text-based grading is used elsewhere in this project (e.g. LLM-as-judge patterns in Evaluating Agents) and is clearly automatable; the issue here is specifically about reliability for THIS kind of policy-compliance check, not automatability in general.",
            "Correct. This is the real, substantive reason: an agent could plausibly generate polished, policy-sounding language while still having taken the wrong real action (or vice versa) -- grading the actual state change (was cancel_order really invoked on a shipped order) is what makes the check trustworthy regardless of how convincing the accompanying text is.",
            "Unrelated -- the choice to run multiple trials (for pass@1 vs. pass^k) is a separate methodological decision about reliability measurement, not a consequence of how any single trial is graded."
          ],
          "source": "Benchmark Atlas",
          "sourceUrl": "benchmark-atlas.md"
        },
        {
          "scenario": "The real atlas data shows OSWorld climbing from a 12.24% launch baseline to a real current top score of 90.19%, while tau-bench's real leaderboard shows telecom at 97.8% alongside a newer banking_knowledge domain at only 55.2%.",
          "question": "What is the most accurate generalization these two real data points together support?",
          "options": [
            "Every benchmark eventually reaches the same saturation point at the same rate",
            "Saturation is a single global property that applies uniformly to an entire benchmark",
            "tau-bench's banking_knowledge domain is a poorly designed benchmark component",
            "Saturation can vary meaningfully across time and within parts of one benchmark"
          ],
          "correct": 3,
          "explanations": [
            "Directly contradicted by the real numbers -- OSWorld and TheAgentCompany (42.9%) are both in this same atlas at very different saturation levels, and even within tau-bench itself, domains sit at wildly different points (55.2% vs. 97.8%); there's no single uniform rate or endpoint.",
            "Directly contradicted by tau-bench's own real per-domain numbers, which show DIFFERENT saturation levels (97.8% vs. 55.2%) within the exact same benchmark -- saturation clearly isn't a single property of 'the benchmark' as a whole in this case.",
            "Unsupported speculation not established by the data -- a domain scoring lower than another domain is consistent with it simply being harder or newer, which is exactly the stated reason it was added, not evidence of poor design.",
            "Correct. The real evidence directly supports this: OSWorld shows saturation can develop within one benchmark over time (12.24% to 90.19%), and tau-bench shows saturation can differ across domains within the SAME benchmark at the SAME time (97.8% vs. 55.2%) -- saturation is neither uniform nor a fixed global property."
          ],
          "source": "Benchmark Atlas",
          "sourceUrl": "benchmark-atlas.md"
        },
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
          ],
          "source": "Observability and Debugging",
          "sourceUrl": "observability-debugging.md"
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
          ],
          "source": "Observability and Debugging",
          "sourceUrl": "observability-debugging.md"
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
          ],
          "source": "Observability and Debugging",
          "sourceUrl": "observability-debugging.md"
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
          ],
          "source": "Observability and Debugging",
          "sourceUrl": "observability-debugging.md"
        },
        {
          "scenario": "A real repro gave an agent all three lethal-trifecta legs (untrusted email, private-record access, unrestricted send) plus a structurally-fixed version (send restricted to the on-file address). In both conditions, the model declined the injected forwarding request -- no exfiltration occurred either way.",
          "question": "What's the most accurate conclusion to draw from this specific result?",
          "options": [
            "The fixed condition's guarantee doesn't depend on the model's own behavior",
            "Prompt injection is effectively solved for well-aligned models like this one",
            "The trifecta framing was unnecessary here, since judgment alone sufficed",
            "Both conditions are equally secure, since neither exfiltrated data this run"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The deterministic allow-list blocks disallowed sends regardless of what the model decides -- that's precisely why it's a different, stronger kind of control than relying on the model's judgment holding up.",
            "Overreaches from one model's response to one injection into a general claim the field's own evidence contradicts -- adaptive attacks bypass published defenses at very high rates.",
            "Draws the wrong lesson from a single favorable outcome -- one instance of good judgment on one injection doesn't establish that judgment is a reliable control in general.",
            "Ignores a structural difference that only shows up under a different injection or model -- the vulnerable condition's send tool would have complied with a successful injection; the fixed condition's would not have, regardless of the model's decision."
          ],
          "source": "Agent Security",
          "sourceUrl": "agent-security.md"
        },
        {
          "scenario": "A real repro showed a model asked an innocuous weather question also calling an unrelated, sensitive lookup tool -- triggered entirely by a hidden instruction embedded in the weather tool's own description field, not by anything in the user's actual request or any document the agent read.",
          "question": "What does this specific finding demonstrate about the attack surface involved?",
          "options": [
            "The user's own request must have contained a subtle injected instruction",
            "The attack surface is the tool's metadata -- content nobody typically reviews",
            "This is functionally identical to the lethal trifecta, since private data was accessed",
            "This can only happen with third-party tools, never ones an organization writes itself"
          ],
          "correct": 1,
          "explanations": [
            "Contradicts the setup directly -- the user's question was a plain, unrelated weather query; the injected instruction lived entirely in the tool's description, not the user's message.",
            "Correct. The instruction was embedded in text that describes the tool to the model on every turn, not in any content the user submitted or any document the agent had to be tricked into opening -- exactly why this is a supply-chain risk, not a content-injection risk.",
            "Conflates two distinct mechanisms -- the trifecta describes a session-level combination of properties across a conversation; tool poisoning is about a single artifact's metadata being untrustworthy, a different attack surface with a different fix.",
            "An unsupported, overly narrow claim -- nothing about the mechanism is inherently limited to external sources; an internally-written tool with a careless or compromised description carries the same risk."
          ],
          "source": "Agent Security",
          "sourceUrl": "agent-security.md"
        },
        {
          "scenario": "A candidate summarizes Meta's Rule of Two as: 'An agent must never combine untrusted input, access to sensitive data, and the ability to take external action -- all three together are always forbidden, no exceptions.'",
          "question": "What's the most accurate correction to this summary?",
          "options": [
            "The summary is correct as stated and needs no correction",
            "The rule only applies to browser-based agents, not email or file-processing agents",
            "The rule permits any two properties; only having all three together is restricted",
            "The rule actually forbids any two of the three properties, making it stricter"
          ],
          "correct": 2,
          "explanations": [
            "Restates the summary rather than correcting it -- the actual quoted rule is more permissive than 'all three combined are forbidden' implies, since it explicitly allows any two.",
            "Introduces an unsupported restriction -- the rule is framed generally around session properties, not scoped to any particular agent modality like browsing.",
            "Correct. Meta's own wording is 'no more than two of the following three properties within a session' -- an agent may freely have any two of the three; it's specifically the full trifecta the rule restricts.",
            "Inverts the actual rule -- Meta's own quoted wording explicitly allows 'no more than two' properties, meaning two together is fine and only the full combination of three is restricted."
          ],
          "source": "Agent Security",
          "sourceUrl": "agent-security.md"
        },
        {
          "scenario": "CaMeL reports solving 77% of AgentDojo tasks with provable security against prompt injection, compared to 84% for an undefended baseline system. A team considering CaMeL concludes: 'This means CaMeL is strictly worse and offers no real advantage over the undefended baseline.'",
          "question": "What's the strongest flaw in that conclusion?",
          "options": [
            "77% and 84% are actually statistically indistinguishable, so there's no real difference",
            "CaMeL's real success rate is higher than 84% once security is factored into the score",
            "The conclusion is correct -- a lower task-success number means a system is strictly worse",
            "It ignores that the 7-point cost buys a real security guarantee the baseline fully lacks"
          ],
          "correct": 3,
          "explanations": [
            "Fabricates a statistical claim with no support -- the reported numbers are presented as real observed results, not statistically equivalent figures, and no such analysis is given in the source.",
            "Misstates how the metric works -- the reported 77% is the task-success rate under CaMeL's actual constraints; security guarantees are a separate, qualitative property, not an adjustment folded into the same percentage.",
            "Treats task-success percentage as the only axis that matters, ignoring that security guarantees are a real, separate dimension of value the comparison must account for.",
            "Correct. Comparing only the task-success numbers ignores the actual trade being made -- the undefended system has no structural protection against prompt injection at all, while CaMeL's lower score buys a provable security property. Whether that cost is worth it depends on the deployment's risk profile, but 'strictly worse' ignores the axis being traded away."
          ],
          "source": "Agent Security",
          "sourceUrl": "agent-security.md"
        },
        {
          "scenario": "A real repro ran the identical refund task under a flat-gated policy (every tool call requires approval, regardless of risk). The agent made only 2 total calls -- both harmless order lookups -- and never attempted the actual $250 refund call at all.",
          "question": "What is the most precise characterization of what this specific result demonstrates?",
          "options": [
            "That undifferentiated gating stalled real progress before review was even needed",
            "That the agent refused the task entirely due to a policy violation it detected",
            "That flat gating is always the safest possible guardrail choice for any task",
            "That the refund tool itself was broken and could not be called under any condition"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, sharper finding is specifically about HOW FAR the agent got, not just how many calls were gated -- flat gating stopped it before it reached the one action that actually needed protecting, which is a different and more concrete cost than simple 'added friction.'",
            "Not what happened -- there's no policy violation in the setup; the agent simply stopped making progress after its first two calls returned PENDING_HUMAN_APPROVAL, it didn't detect or object to anything.",
            "Overstates the case -- flat gating did prevent the risky refund, but at a real cost (the agent never made it that far); 'always safest' ignores the real tradeoff the repro itself measured.",
            "Contradicted directly by the other two real conditions in the same repro, where issue_refund was called successfully (flat-autonomous) or correctly gated (tiered) -- the tool itself worked fine; only the flat-gated condition's approval policy prevented it from ever being attempted."
          ],
          "source": "Guardrails and Human-in-the-Loop",
          "sourceUrl": "guardrails-human-in-the-loop.md"
        },
        {
          "scenario": "A real repro computed risk per tool call using the actual arguments (e.g., the refund amount), not just which tool was named -- so issue_refund for $10 and issue_refund for $250 were treated as different real risk levels despite being the same tool.",
          "question": "What is the most accurate reason this specific design choice matters?",
          "options": [
            "Naming a tool 'issue_refund' is inherently unsafe regardless of any arguments passed",
            "Real risk often depends on the specific parameters of a call, not just its name",
            "Every tool call should always be treated as maximum risk to be safe by default",
            "Argument-based risk scoring makes tool names themselves completely unnecessary"
          ],
          "correct": 1,
          "explanations": [
            "Not the point being made -- the repro's own design shows the SAME tool name being treated as different risk depending on arguments, directly contradicting the idea that the tool name alone determines danger.",
            "Correct. This is exactly the real distinction the repro's risk_tier function encodes: a $10 refund and a $250 refund are the same tool but meaningfully different real risk -- gating decisions that only look at tool names would miss this and either over-gate cheap refunds or under-gate expensive ones.",
            "Contradicts the entire point of tiering -- treating everything as maximum risk is exactly the flat-gated condition, which the repro's own results show has a real, measured cost (stalled progress) compared to proportional tiering.",
            "An unsupported leap -- tool names remain necessary for identifying WHICH action is being taken; argument-based risk scoring is an additional layer on top of tool identification, not a replacement for it."
          ],
          "source": "Guardrails and Human-in-the-Loop",
          "sourceUrl": "guardrails-human-in-the-loop.md"
        },
        {
          "scenario": "A real, cited guideline states: 'Define explicit risk tiers for agent use cases and apply proportional controls. Low-risk tasks like data summarization can run with lighter oversight. High-risk actions... require multi-step verification, human approval, and comprehensive audit trails.'",
          "question": "What does the phrase 'lighter oversight' for low-risk tasks most precisely imply, given this page's own real repro?",
          "options": [
            "That low-risk tasks should receive zero guardrails or safety consideration of any kind",
            "That only high-risk actions were ever intended to be part of any guardrail system",
            "That distinguishing risk levels is itself considered a legitimate, deliberate practice",
            "That lighter oversight means slower, more thorough review for every single action"
          ],
          "correct": 2,
          "explanations": [
            "Overstates 'lighter' as 'none' -- the real quote says lighter oversight, not zero; this page's own tiered condition still tracked and logged every call's tier, it simply didn't gate the low-risk ones.",
            "Contradicted directly by the quote itself, which explicitly names 'low-risk tasks' as a category the guidance addresses, with its own appropriate (lighter, not absent) level of control.",
            "Correct. The real guidance explicitly frames applying DIFFERENT levels of control based on risk as the correct practice, not an oversight or a corner being cut -- this page's own repro operationalized exactly this principle and showed it producing a real, measured benefit (the agent completed low-risk work while the high-risk action was still caught).",
            "Directly contradicts the quoted phrase -- 'lighter oversight' is the opposite of 'slower, more thorough review'; the quote's own contrast is between lighter oversight (low-risk) and multi-step verification (high-risk)."
          ],
          "source": "Guardrails and Human-in-the-Loop",
          "sourceUrl": "guardrails-human-in-the-loop.md"
        },
        {
          "scenario": "A real, five-layer guardrail framework names human-in-the-loop oversight as its final, fifth layer, coming after data/context guardrails, design-time governance, runtime guardrails/gateways, and identity/access/security controls.",
          "question": "What does human review being positioned as the LAST layer, rather than the only layer, most precisely imply about its intended role?",
          "options": [
            "That human review is the least important layer and could reasonably be skipped entirely",
            "That the first four layers are expected to eliminate all risk before review is needed",
            "That the five layers must always execute in strict sequential order for every single action",
            "That human review exists to catch residual risk the earlier automated layers didn't resolve"
          ],
          "correct": 3,
          "explanations": [
            "Not supported -- being positioned last in a defense-in-depth stack typically signals it catches what earlier layers miss, which is a specific and real function, not a signal of lower importance or optional status.",
            "Overstates what defense-in-depth architectures claim -- the entire premise of a layered stack is that no single layer (including the earlier ones) is assumed to be perfect; human review's real role is exactly to catch what slips through, not to be a redundant final check on a supposedly risk-free process.",
            "Not established by the framework as described -- layers in a defense-in-depth model are not necessarily strictly sequential gates for every action; some may apply continuously or in parallel (e.g., identity/access controls), rather than each action passing through all five in fixed order.",
            "Correct. This is the standard, real logic of a layered defense stack: earlier layers (data/context guards, governance, runtime gates, identity/access controls) handle what they can automatically and cheaply, and human review is reserved specifically for the risk that remains after those layers have already acted -- exactly why it shouldn't be applied uniformly to everything, including things the earlier layers already handled."
          ],
          "source": "Guardrails and Human-in-the-Loop",
          "sourceUrl": "guardrails-human-in-the-loop.md"
        },
        {
          "scenario": "A real repro sent the identical underlying task (approve a vendor invoice by email) under two conditions: an agent told to 'just act as the user,' and an agent given a distinct identity plus an explicit, scoped delegation with instructions to disclose both externally. The first produced a fully anonymous email; the second explicitly named the agent and its authorized scope.",
          "question": "What is the most precise conclusion this specific result supports?",
          "options": [
            "Identity disclosure and delegation traceability follow directly from explicit instruction, not automatically from careful agent behavior",
            "The blended condition's email was defective or malformed compared to the delegated condition's output",
            "Identity disclosure requires a separate enforcement mechanism beyond instructions, since models cannot be trusted to follow such instructions",
            "Both conditions are functionally identical from a compliance perspective, since the underlying action taken was the same"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, first-try result shows the difference came directly from what was specified in the system instructions -- explicit identity and delegation produced explicit disclosure; its absence produced none. This is precisely the page's own stated takeaway: disclosure is a design choice, not an automatic byproduct.",
            "Not indicated -- the blended email was well-formed and clearly communicated the approval; it wasn't broken, it simply lacked any identity or delegation disclosure, which is a different property than malformation.",
            "Overstates it -- the real repro's own result is a clean, first-try SUCCESS from instruction alone, not a demonstrated failure requiring additional enforcement; the finding is that the instruction worked, not that it's untrustworthy.",
            "Contradicted directly -- the page treats WHO the recipient can identify as taking the action as a substantive difference, directly relevant to transparency obligations like EU AI Act Article 50; the underlying action being identical doesn't make the disclosure difference irrelevant."
          ],
          "source": "Identity, Governance and Compliance",
          "sourceUrl": "identity-governance-compliance.md"
        },
        {
          "scenario": "OWASP's real, current Top 10 for Agentic Applications names ASI03 (Identity & Privilege Abuse) and ASI09 (Human-Agent Trust Exploitation) as distinct, separate risk categories, alongside other categories like ASI01 (Agent Goal Hijack) and ASI02 (Tool Misuse).",
          "question": "What real, distinct problem do ASI03 and ASI09 specifically name, as opposed to the technique-level attacks this page's companion Agent Security post already covers?",
          "options": [
            "ASI03 and ASI09 are simply renamed duplicates of the lethal trifecta and indirect injection already covered elsewhere",
            "ASI03 and ASI09 apply exclusively to multi-agent systems and have no relevance to single-agent deployments",
            "ASI03 and ASI09 are deprecated categories that OWASP has since removed from its current list",
            "ASI03 and ASI09 name identity/privilege-scope failures and human over-trust in agent authority claims specifically"
          ],
          "correct": 3,
          "explanations": [
            "Not accurate -- the real, quoted descriptions are distinct: ASI03 is specifically about credentials enabling operation 'beyond intended scope' and ASI09 about humans being misled by 'confident explanations,' neither of which restates the lethal trifecta or injection mechanics covered in the companion post.",
            "Not supported -- nothing in the real, cited list restricts these categories to multi-agent contexts specifically; identity/privilege abuse and human trust exploitation are described as general agentic risks.",
            "Contradicted directly -- these are presented as part of OWASP's CURRENT, real, published December 2025 list, not as removed or deprecated categories.",
            "Correct. ASI03's real description names credential/privilege-scope failures directly; ASI09's names a distinct human-trust failure mode (being misled by a confident agent into approving something harmful) -- both are governance/identity-layer risks, complementary to but distinct from the technique-level attacks (trifecta, injection) covered elsewhere."
          ],
          "source": "Identity, Governance and Compliance",
          "sourceUrl": "identity-governance-compliance.md"
        },
        {
          "scenario": "The EU AI Act's real, verified text includes Article 50 (AI-interaction transparency), Article 14 (human oversight for high-risk systems), and Article 6/Annex III (high-risk classification by use case) -- but no provision specifically named 'AI agents' or 'autonomous agents' as a defined regulatory category.",
          "question": "What is the most precise, honest way to determine which EU AI Act obligations apply to a given agentic system, based on this real structure?",
          "options": [
            "Agentic systems are entirely exempt from the EU AI Act, since no provision specifically names them as a category",
            "Classify the system by what it does using the Act's existing risk categories, not by searching for an agent chapter",
            "All agentic systems are automatically classified as high-risk under Annex III regardless of their specific use case",
            "The EU AI Act does not yet apply to any AI system as of this writing, agentic or otherwise"
          ],
          "correct": 1,
          "explanations": [
            "Not supported and likely incorrect -- the absence of an agent-specific chapter doesn't mean exemption; it means agentic systems are evaluated under the SAME existing categories (transparency, high-risk use cases) that apply to any AI system meeting those criteria.",
            "Correct. This is the precise, honest reading the page gives: since no agent-specific chapter exists, the correct approach is applying the Act's real, existing categories (Article 50 transparency, Annex III high-risk use cases) based on what the system actually does, not searching for an 'agent' label that isn't in the text.",
            "Not accurate -- Annex III's real, verified structure defines high-risk status by SPECIFIC USE CASES (biometrics, employment, essential services, etc.), not by architecture; an agentic system isn't automatically high-risk merely for being agentic.",
            "Directly contradicted by the real, verified dates given -- Article 50 applies from August 2, 2026, and other provisions have their own real, specific application dates; parts of the Act are already in force or scheduled with concrete dates, not universally inapplicable."
          ],
          "source": "Identity, Governance and Compliance",
          "sourceUrl": "identity-governance-compliance.md"
        },
        {
          "scenario": "MITRE ATLAS is described as a real, living knowledge base of adversary tactics and techniques against AI systems, modeled on MITRE ATT&CK, and is reported to be expanding to cover attack paths at the orchestration and execution layers as agents act more autonomously.",
          "question": "What is the most precise way to characterize MITRE ATLAS's real role relative to OWASP's Top 10 for Agentic Applications, as this page frames them?",
          "options": [
            "ATLAS replaces OWASP's list entirely, since ATLAS is the more comprehensive and authoritative framework",
            "ATLAS and OWASP's list cover completely unrelated domains with no meaningful connection between them",
            "ATLAS provides the adversary-technique vocabulary that complements OWASP's named risk categories",
            "ATLAS is a proprietary, closed framework unlike OWASP's fully open, community-maintained list"
          ],
          "correct": 2,
          "explanations": [
            "Not the framing given -- the page presents ATLAS and OWASP's list as complementary, serving different purposes (risk categories vs. adversary techniques), not as one superseding the other.",
            "Contradicted directly -- the page explicitly frames ATLAS as the 'adversary-technique counterpart' to OWASP's risk list, describing a direct, deliberate connection between the two, not an unrelated domain.",
            "Correct. This is precisely the page's own framing: OWASP names WHAT can go wrong (risk categories like ASI03, ASI09), while ATLAS catalogs HOW adversaries actually achieve it (tactics and techniques) -- complementary tools, with ATLAS specifically useful for red-teaming and demonstrating compliance coverage against a recognized framework.",
            "Not supported -- MITRE ATLAS is described as 'a globally accessible, living knowledge base,' consistent with an openly accessible resource; the page doesn't characterize it as proprietary or closed, nor does it draw an open-vs-closed contrast with OWASP."
          ],
          "source": "Identity, Governance and Compliance",
          "sourceUrl": "identity-governance-compliance.md"
        },
        {
          "scenario": "A team migrates their agent's code-execution tool from a Docker container to a Firecracker microVM, expecting this to substantially raise their security bar. They don't change how the agent's payments API credential is supplied to the sandboxed code -- it still sits directly in the executed code's own environment variables.",
          "question": "What does this page's own repro most precisely predict about this migration's effect on credential-related risk?",
          "options": [
            "It leaves credential exposure essentially unchanged, since the credential never needed to escape the sandbox",
            "It closes the risk entirely, since Firecracker's hardware isolation is the strongest tier on the ladder",
            "It meaningfully reduces the risk, though some residual exposure from the credential would remain",
            "It increases the risk, because microVMs are slower to patch against credential-based attacks than containers"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The whole point of the theft-vs-misuse repro is that isolation strength (which tier of the ladder) and credential placement (inside vs. proxied) are separate axes -- a stronger sandbox does nothing about a credential that's directly reachable from inside it, since the injected instruction just asks the already-trusted agent to use it, without ever needing to cross the sandbox boundary.",
            "Overstates what sandbox strength buys -- the mechanism-level check showed the real credential value visible to sandboxed code specifically because it was placed there directly, a fact independent of which isolation tier is wrapping the execution.",
            "Understates it -- there's no partial credit here for sandbox strength on this specific risk; a credential placed directly in scope is fully reachable by an injected instruction whether the sandbox is a container or a microVM, because the attack never needs to breach the isolation boundary at all.",
            "Not a real or claimed effect -- the page makes no claim about patch cadence differing between isolation tiers; this misattributes the actual mechanism (credential placement, not isolation strength) to an unrelated concern."
          ],
          "source": "Sandboxes and Permissions",
          "sourceUrl": "sandboxes-permissions.md"
        },
        {
          "scenario": "A real, deterministic mechanism-level check (no LLM involved) found that a hand-written payment call to an attacker-controlled-looking destination succeeded in both an unproxied sandbox (real credential directly visible) and a proxied sandbox (only a placeholder token visible).",
          "question": "What is the most precise conclusion this specific result supports about credential proxying?",
          "options": [
            "Proxying provides no real security benefit at all, since the misuse still succeeded in both conditions",
            "Proxying failed at its intended purpose, since the placeholder token should have blocked the payment from being sent",
            "The result is inconclusive because a real production proxy would have additional protections not modeled here",
            "Proxying is a confidentiality control that did its job; the payment succeeding reflects a separate, unaddressed gap"
          ],
          "correct": 3,
          "explanations": [
            "Too strong -- the credential value itself DID differ cleanly between conditions (real key vs. placeholder only), which is a real, working confidentiality benefit; the claim that it 'provides no benefit at all' ignores that half of the check's own result.",
            "Misunderstands the proxy's actual design -- the placeholder token being usable to complete a call via the proxy IS the proxy working as intended (the sandboxed code never needed the real key to get the action executed); the payment succeeding isn't a proxy failure, it's evidence the proxy was never an authorization mechanism in the first place.",
            "Not what the repro claims or needs to claim -- the mechanism-level check is explicitly about what the proxying PATTERN itself structurally permits, independent of whatever additional real-world hardening a production system might layer on top; the theft-vs-misuse distinction holds regardless of those additions.",
            "Correct. The mechanism check cleanly separated two different properties: what the sandboxed code can SEE (proxying fixed this -- placeholder only) versus what the sandboxed code can DO (proxying never claimed to fix this -- the proxy's job was never to judge whether a given destination/amount is legitimate, which is what the page calls a separate authorization gap)."
          ],
          "source": "Sandboxes and Permissions",
          "sourceUrl": "sandboxes-permissions.md"
        },
        {
          "scenario": "In a real, unforced run, Claude Sonnet 5 was given direct access to a real credential inside a sandboxed code-execution tool, under a prompt injection asking it to both print the credential and send a payment to a suspicious-looking account. It declined both, printing only a masked value and explicitly citing the suspicious destination pattern.",
          "question": "What is the most accurate way to characterize what this specific live-agent result does and doesn't establish?",
          "options": [
            "It proves that credential proxying is unnecessary whenever the underlying model is capable enough to reason about the request",
            "A real result about this model's behavior on this run, not a substitute for the deterministic guarantee",
            "It's not meaningful evidence at all, since the model could reason differently on every subsequent run",
            "It demonstrates that credential theft is now a solved problem for any sufficiently advanced language model"
          ],
          "correct": 1,
          "explanations": [
            "Overreaches directly -- the page explicitly warns against this conclusion: model judgment holding on one run is a fact about that run, not a property of the system, and shouldn't be treated as a substitute for the structural (proxy + authorization) controls.",
            "Correct. The page draws exactly this distinction: the live-agent trial is real and worth reporting honestly, but it answers 'what did this model choose to do here,' which is categorically different from the mechanism-level check's 'what does the architecture permit regardless of model behavior' -- both are real data, serving different purposes.",
            "Too dismissive -- a real, unforced result is genuine evidence about model behavior under this specific injection, and the page reports it plainly as such; the caution is about over-generalizing it into a security guarantee, not about the result being meaningless.",
            "A sweeping claim the page never makes and actively cautions against -- one favorable run against one injection says nothing about 'solved,' especially given the page's own point that model judgment is not a designed control."
          ],
          "source": "Sandboxes and Permissions",
          "sourceUrl": "sandboxes-permissions.md"
        },
        {
          "scenario": "A real, verified guarantee for the credential-proxying pattern states: 'The agent's process never has the plaintext secret in memory, on disk, or in its context window.'",
          "question": "Which real, named limitation does this page pair directly with that guarantee?",
          "options": [
            "The guarantee only holds for text-based secrets, not binary credentials like signing keys",
            "The guarantee requires a hardware-isolated microVM to actually hold, and fails under weaker isolation tiers",
            "Proxying prevents credential theft but does not prevent credential misuse of the substituted credential",
            "The guarantee is voided the moment more than one sandboxed process shares the same proxy"
          ],
          "correct": 2,
          "explanations": [
            "Not a distinction the page makes or that the quoted guarantee implies -- the guarantee is about where the plaintext secret does and doesn't appear (memory, disk, context window), not about the secret's format or data type.",
            "Not correct -- the proxying pattern is described as a boundary independent of which isolation tier wraps the sandboxed execution; the page treats isolation strength and credential-proxying as two separate axes, not one dependent on the other.",
            "Correct. This is the page's central, real, named pairing -- the direct follow-on quote 'Proxying prevents credential theft. It does not prevent credential misuse' -- confirmed concretely by the mechanism-level check, where the payment succeeded in both conditions despite the credential value being hidden in the proxied one.",
            "Not a claim made anywhere on this page -- multiple sandboxed processes sharing one proxy isn't discussed as a failure condition; this invents a mechanism not present in the real, cited material."
          ],
          "source": "Sandboxes and Permissions",
          "sourceUrl": "sandboxes-permissions.md"
        },
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
          ],
          "source": "Durable Execution",
          "sourceUrl": "durable-execution.md"
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
          ],
          "source": "Durable Execution",
          "sourceUrl": "durable-execution.md"
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
          ],
          "source": "Durable Execution",
          "sourceUrl": "durable-execution.md"
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
          ],
          "source": "Durable Execution",
          "sourceUrl": "durable-execution.md"
        },
        {
          "scenario": "A real repro measured a naive agent loop's per-turn input token counts across 7 turns: 665, 1290, 1921, 2552, 3183, 3814, 4445 -- each turn costing roughly 631 tokens more than the one before it.",
          "question": "What does this specific pattern -- a roughly CONSTANT per-turn increase -- indicate about the shape of the CUMULATIVE total across turns?",
          "options": [
            "The cumulative total grows quadratically, since a steady per-turn increase yields a triangular-number sum overall",
            "The cumulative total grows linearly, since each individual turn's token count only increases by a fixed amount",
            "The cumulative total grows exponentially, since token counts are roughly doubling by the final turns of the run",
            "The cumulative total stays constant, since only the per-turn increase changes and not the baseline cost itself"
          ],
          "correct": 0,
          "explanations": [
            "Correct. A sequence with a roughly constant increase per step (665, +625, +631, +631, +631, +631, +631) is an arithmetic sequence, and the SUM of an arithmetic sequence is a triangular number -- quadratic in the number of terms. This is exactly the real, documented N(N+1)/2 term the page describes, confirmed directly by this run's own measured numbers.",
            "Confuses per-turn behavior with cumulative behavior -- a constant per-turn INCREASE (not a constant per-turn value) is precisely the signature of quadratic cumulative growth, not linear; linear cumulative growth would require each turn to cost roughly the SAME amount, not a steadily increasing amount.",
            "Overstates the growth rate -- exponential growth would mean each turn's increase itself keeps growing multiplicatively (631, then ~1200, then ~2400...); here the increase itself stays roughly flat (~631 each time), which is the quadratic/triangular-number signature, not exponential.",
            "Directly contradicted by the data -- token counts clearly increase every single turn (665 up to 4445); nothing about this run is constant except the SIZE of the increase, not the total itself."
          ],
          "source": "Cost and Latency",
          "sourceUrl": "cost-latency.md"
        },
        {
          "scenario": "The same repro ran a windowed version of the identical task (only the last 2 tool results sent in full) and found nearly identical per-turn token counts to the naive version for the first 3 turns, before the windowed version's growth rate flattened sharply from turn 4 onward.",
          "question": "What is the most precise explanation for why the first 3 turns looked nearly identical between conditions?",
          "options": [
            "The windowing logic contained a bug that only started working partway through the run",
            "The model ignored the windowed context and used its own cached memory of earlier turns",
            "Anthropic's API caches identical early turns automatically regardless of which condition is used",
            "Fewer than 3 tool results existed yet at that point, so the window had nothing to omit"
          ],
          "correct": 3,
          "explanations": [
            "Not indicated anywhere -- the windowing logic is described as correctly keeping only the most recent window and omitting anything older; the near-identical early turns are an expected structural consequence of the window's own definition, not a malfunction.",
            "Not a real mechanism -- models don't retain memory of prior API calls outside what's explicitly included in the current request's messages; the windowed condition's context is exactly what's sent in that specific call, nothing more.",
            "Not how prompt caching works, and not what's being measured here -- this page's real numbers are INPUT TOKEN COUNTS sent per call, not cache hit/miss behavior; caching (covered in a separate topic) doesn't explain why two DIFFERENT conditions would show similar early-turn token counts by itself.",
            "Correct. A 'keep only the last 2 tool results' window has literally nothing to omit until MORE than 2 tool results exist in the transcript -- so for the first few turns, the windowed and naive conditions send essentially the same content, and the two conditions only diverge once the transcript grows past the window size."
          ],
          "source": "Cost and Latency",
          "sourceUrl": "cost-latency.md"
        },
        {
          "scenario": "A real repro compared always-Sonnet vs. Haiku-first-with-escalation on 10 real questions. The routed condition matched Sonnet's accuracy (9/10) at about 44% of the real dollar cost -- but the escalation count was exactly zero, including on the one question Haiku got wrong, which it answered with reported 'high' confidence.",
          "question": "What is the most precise way to characterize what this combined result does and doesn't demonstrate about confidence-based routing?",
          "options": [
            "It demonstrates routing failed completely, since the escalation mechanism never triggered even once during the run",
            "The cost benefit of a cheaper model was real, but self-reported confidence didn't reliably catch the model's own mistake",
            "It demonstrates Haiku is strictly more accurate than Sonnet, since it achieved the same correct count at lower cost",
            "It demonstrates self-reported confidence works correctly, since the model would have escalated if it had truly been uncertain"
          ],
          "correct": 1,
          "explanations": [
            "Too sweeping -- the cost/accuracy result (44% of cost, identical accuracy) is a genuine positive outcome; 'failed completely' ignores that real, measured benefit and conflates it with the separate, real limitation in the escalation trigger specifically.",
            "Correct. The page draws exactly this two-part distinction: the cost lever (routing to a cheaper model) delivered real savings with no accuracy loss on this batch, which is one real finding -- but the SAFETY lever (using confidence to catch mistakes before they ship) failed on the one case that mattered, because self-reported confidence didn't correlate with actual correctness here. Both are real and both matter, but they are not the same claim.",
            "Overreaches from a tie -- both models got 9/10 correct (an equal, not superior, result for Haiku); 'strictly more accurate' isn't supported since the counts were identical, only the cost differed.",
            "Backwards -- the real result is the opposite: the model reported HIGH confidence specifically on the question it got wrong, which is direct evidence the self-report mechanism did NOT reliably track true uncertainty, not confirmation that it works as intended."
          ],
          "source": "Cost and Latency",
          "sourceUrl": "cost-latency.md"
        },
        {
          "scenario": "The page states that STEER (a real, cited routing method) uses model-internal confidence derived from token-level output probabilities, rather than a self-reported natural-language confidence field -- and notes this page's own repro could not test that specific mechanism directly.",
          "question": "What real, current constraint does the page give for why the repro used self-reported confidence instead of STEER's actual mechanism?",
          "options": [
            "Self-reported confidence was chosen deliberately for being more accurate than token-probability-based methods in practice",
            "STEER's method has been deprecated and is no longer considered a valid routing approach as of this writing",
            "The Anthropic Messages API used throughout this project doesn't expose token-level log probabilities at all",
            "Token-probability-based confidence only works for open-source models, never for any hosted commercial API"
          ],
          "correct": 2,
          "explanations": [
            "Backwards -- the page's own real result argues the opposite: self-reported confidence failed to catch the model's one actual mistake, which is presented as a limitation to be honest about, not a deliberate preference for a more accurate method.",
            "Not a claim the page makes -- STEER is cited as a real, current method being contrasted with this repro's fallback approach, not as something superseded or invalid.",
            "Correct. The page states this directly and precisely: the Messages API used throughout this project doesn't expose token-level logprobs the way some routing methods assume access to, which is exactly why the repro falls back to a self-reported field -- and the real result shows why that fallback has a real cost (missing the model's actual mistake).",
            "Overgeneralizes beyond what the page claims -- the page's point is specifically about which API this project uses (Anthropic's Messages API) not exposing this data, not a blanket claim about all commercial APIs universally lacking any form of token-probability access."
          ],
          "source": "Cost and Latency",
          "sourceUrl": "cost-latency.md"
        },
        {
          "scenario": "A real run computed GRPO advantages for a group where every one of 8 sampled completions received the identical reward (all correct). The result was an advantage of exactly 0.0 for all 8 samples.",
          "question": "What is the most accurate description of what this result demonstrates?",
          "options": [
            "DAPO's 'gradient-decreasing problem' -- zero variance, zero gradient",
            "A bug in the advantage formula needing a special case for perfect groups",
            "That the model has nothing left to learn from any prompt it handles well",
            "That group sizes of 8 are too small to produce meaningful advantages"
          ],
          "correct": 0,
          "explanations": [
            "Correct. DAPO's own paper names this exactly: 'if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients.'",
            "Misdiagnoses a real, documented property of the formula as an implementation error -- the zero result is mathematically correct given the formula, not a flaw needing a workaround.",
            "Overgeneralizes from one group's result to a permanent claim about the prompt -- the issue is this specific batch having no variance, not that the prompt is unlearnable from in general.",
            "Introduces an unsupported claim about group size -- the zero result occurs at any group size whenever every sample receives an identical reward; size isn't the variable at fault."
          ],
          "source": "Training Agents: Reward and Credit",
          "sourceUrl": "training-agents.md"
        },
        {
          "scenario": "DAPO's dynamic sampling filters out any group whose accuracy is exactly 0 or exactly 1, keeping only groups with accuracy strictly between those values. A team proposes simplifying this to: 'Just filter out any group with accuracy below 50%, since low-accuracy groups are clearly not useful.'",
          "question": "What's the strongest problem with that proposed simplification?",
          "options": [
            "The proposal is equivalent to DAPO's actual rule, just phrased differently",
            "It would discard groups with usable variance that DAPO's rule actually keeps",
            "The 50% threshold should be higher, closer to 90%, to match DAPO's behavior",
            "Filtering by accuracy is never a valid strategy for any RL training approach"
          ],
          "correct": 1,
          "explanations": [
            "Misstates the comparison -- a group with accuracy 0.3 is kept by the real rule and discarded by the proposed one, so the two are not equivalent.",
            "Correct. DAPO's actual condition keeps any group with accuracy strictly between 0 and 1 -- including accuracy 0.3, which still has real reward variance and produces a real, non-zero gradient. A 50% floor would discard genuinely useful training signal DAPO's real rule correctly retains.",
            "Proposes an arbitrary alternative threshold with no grounding -- DAPO's real rule isn't a threshold at all, it specifically excludes only the two boundary values (0 and 1).",
            "Overgeneralizes into an absolute claim the material contradicts -- DAPO's own dynamic sampling IS a form of filtering by accuracy, so filtering by accuracy is demonstrably valid as published."
          ],
          "source": "Training Agents: Reward and Credit",
          "sourceUrl": "training-agents.md"
        },
        {
          "scenario": "A real toy repro showed outcome-only credit assignment producing identical values ([-1, -1, -1]) for a 3-step trajectory where only the middle step was actually the mistake, while process-level credit correctly produced [1, -1, 1]. A candidate concludes: 'This proves outcome-only reward should never be used, since it always fails to identify which step is bad.'",
          "question": "What's the most accurate pushback on this conclusion?",
          "options": [
            "The conclusion is correct -- outcome-only reward provides no information at all",
            "Process-level reward is strictly better in every respect, including cost",
            "It shows one limitation (can't localize the bad step), not that it's worthless",
            "The result only applies to exactly 3-step trajectories, not longer ones"
          ],
          "correct": 2,
          "explanations": [
            "Overstates the finding -- outcome reward does provide real information (whether the trajectory succeeded overall), just not step-level localization within it.",
            "Contradicts the material directly -- process reward's real documented cost (800,000 human labels, or MC-estimation needing consensus filtering) is exactly why it isn't simply better in every respect.",
            "Correct. The demo's specific limitation is that outcome-only credit can't distinguish WHICH step caused a failure -- it says nothing about whether outcome reward is useful at all. It remains cheap (often just a verifier) and gives a valid trajectory-level signal.",
            "Fabricates a scope restriction with no basis -- the structural issue applies to any trajectory with more than one step, not specifically three."
          ],
          "source": "Training Agents: Reward and Credit",
          "sourceUrl": "training-agents.md"
        },
        {
          "scenario": "A real DPO-pair repro generated two completions to the same prompt with no explicit temperature setting, and the two completions were genuinely different. A candidate explains: 'This must mean the SDK silently applied a high default temperature to force diversity for this demo.'",
          "question": "What's the most accurate explanation of what actually happened?",
          "options": [
            "SDKs commonly apply hidden temperature overrides for preference-pair use cases",
            "Two identical completions would have been expected, so this suggests an error",
            "The judge call itself introduced the variation, not the two completion calls",
            "Default sampling alone can produce real variance, with no temperature setting"
          ],
          "correct": 3,
          "explanations": [
            "Fabricates an unsupported claim about hidden, use-case-specific SDK behavior -- there's no support for such a mechanism, and this SDK version doesn't even expose a top-level temperature parameter to set in the first place.",
            "Assumes determinism that isn't supported -- nothing about the API guarantees identical outputs across independent calls by default, and the real repro directly demonstrates otherwise.",
            "Misattributes the source of variation -- the judge call happens AFTER both completions already exist and differ; it evaluates the difference, it doesn't create it.",
            "Correct. Two independent calls with no temperature specified still produced genuinely different completions -- consistent with default sampling variance being normal model behavior, not something requiring explicit configuration to observe."
          ],
          "source": "Training Agents: Reward and Credit",
          "sourceUrl": "training-agents.md"
        },
        {
          "scenario": "A real repro scored two different real tool-call completions against the identical target action using both binary reward (exact match only) and a fine-grained, decomposed reward (tool name / parameter names / parameter values). One completion used the right tool with one reasonably guessed wrong value; the other used a completely different tool. Binary reward scored both 0.0.",
          "question": "What is the most precise real problem this identical binary score demonstrates?",
          "options": [
            "Binary reward assigns zero gradient signal distinguishing a near-correct completion from a completely wrong one",
            "Binary reward is always a worse choice than fine-grained reward for every possible RL training scenario",
            "The fine-grained reward function contains a bug, since two genuinely different completions should never both score low",
            "Binary reward only fails when exactly two tools are available for the model to choose between"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The real, measured finding is that binary reward gave the SAME score (0.0) to two meaningfully different real completions -- one nearly perfect, one entirely wrong -- meaning a policy trained on it alone has no signal telling those two cases apart, exactly the gap ToolRL's own verified claim describes.",
            "Overstates the claim -- the page doesn't argue binary reward is categorically worse in every scenario (e.g. it's fine for genuinely all-or-nothing tasks); the specific, demonstrated problem is about DIFFERENTIATING degrees of correctness within multi-parameter tool calls.",
            "Backwards -- the fine-grained reward function correctly gave the two completions DIFFERENT scores (0.8889 vs 0.2222), which is it working as intended; it's binary reward that failed to differentiate them, not a bug in the fine-grained one.",
            "Not what the repro shows or claims -- the identical-score problem is about reward GRANULARITY (exact-match vs. component-wise), not about how many tools happen to be available in any specific trial; it would recur with any number of available tools."
          ],
          "source": "RL for Search and Tool Agents",
          "sourceUrl": "rl-search-tool-agents.md"
        },
        {
          "scenario": "RAGEN's real, verified abstract names a recurring instability pattern in multi-turn LLM agent RL training called the 'Echo Trap,' described as being 'marked by reward variability collapse, entropy drop, and gradient spikes,' where models converge to near-identical phrasing without justification.",
          "question": "What would be the most accurate way to distinguish a real Echo Trap from ordinary, healthy training convergence, based on RAGEN's own described signature?",
          "options": [
            "Ordinary convergence and the Echo Trap are indistinguishable from training metrics alone and require no further investigation",
            "The Echo Trap is defined purely by the loss curve flattening out over time, regardless of any other training signal",
            "Any drop in entropy during training is sufficient on its own to confirm an Echo Trap has occurred",
            "Reward variability collapse, entropy drop, and gradient spikes together signal the Echo Trap specifically"
          ],
          "correct": 3,
          "explanations": [
            "Contradicted directly -- RAGEN's own abstract gives a specific, named, checkable signature (the three co-occurring signals), which is precisely a way to distinguish it from other training dynamics, not evidence that it can't be distinguished at all.",
            "Incomplete -- a flattening loss curve alone is ambiguous between genuine convergence and collapse; RAGEN's own named signature requires the CO-OCCURRENCE of reward variability collapse, entropy drop, AND gradient spikes together, not loss flattening in isolation.",
            "Overstates a single signal -- entropy naturally decreases somewhat during normal training as a policy becomes more confident and less exploratory; RAGEN's own definition requires it to co-occur with reward variability collapse and gradient spikes, not to appear alone.",
            "Correct. RAGEN's own quoted definition names three co-occurring signals as the Echo Trap's signature, and the paper's own qualitative description adds that models converge to near-identical, unjustified phrasing -- checking rollout text directly for that repetition, alongside the three metrics, is the most complete match to what RAGEN actually documented."
          ],
          "source": "RL for Search and Tool Agents",
          "sourceUrl": "rl-search-tool-agents.md"
        },
        {
          "scenario": "Search-R1, R1-Searcher, and ReSearch are all real, separate 2026 papers that train LLMs to interleave search calls with reasoning using reinforcement learning.",
          "question": "What real, shared design choice do all three papers make regarding supervision of the reasoning process itself?",
          "options": [
            "All three require large amounts of supervised data labeling the correct reasoning steps before any RL training begins",
            "All three rely only on final-answer correctness, without supervising intermediate reasoning or search steps",
            "All three require a human reviewer to approve each individual search query before it is allowed to execute",
            "All three use a separately trained critic model to score every intermediate reasoning step during training"
          ],
          "correct": 1,
          "explanations": [
            "Directly contradicted -- the whole point of outcome-based RL in these papers is to AVOID needing supervised data on the reasoning steps; R1-Searcher's own quote is explicit that it works 'without requiring process rewards or distillation for a cold start,' and Search-R1/ReSearch make the same real design choice.",
            "Correct. All three papers' real, verified descriptions confirm this shared choice: Search-R1 uses 'a simple outcome-based reward function,' R1-Searcher is explicitly 'outcome-based... without requiring process rewards,' and ReSearch trains 'without using any supervised data on reasoning steps' -- only the final answer's correctness supervises training in each case.",
            "Not a real mechanism described in any of the three papers -- these are automated RL training loops without a human-in-the-loop approval step on individual search queries during training.",
            "Not accurate for this shared design choice -- GRPO (used by at least Search-R1 and ReSearch) specifically avoids a separate critic model, estimating the baseline from the sampled group instead; a critic-based approach is the PPO-style alternative these methods are contrasted against, not what they share."
          ],
          "source": "RL for Search and Tool Agents",
          "sourceUrl": "rl-search-tool-agents.md"
        },
        {
          "scenario": "ReTool reports a real, verified result on AIME: a 32B model reaches 67% accuracy after 400 training steps, compared to a text-only RL baseline reaching 40% accuracy after 1080 steps.",
          "question": "What is the most precise interpretation of what this specific comparison demonstrates about ReTool's contribution?",
          "options": [
            "ReTool proves that tool-augmented models are always more accurate than text-only models on any reasoning benchmark",
            "The 1080-step baseline failed to converge at all, which is why its accuracy remained lower than ReTool's",
            "ReTool reached higher accuracy while needing meaningfully fewer training steps than the text-only baseline here",
            "ReTool's higher accuracy came entirely from a larger model size rather than from its RL training approach"
          ],
          "correct": 2,
          "explanations": [
            "Overgeneralizes a single benchmark result into a universal claim -- the real, verified numbers are specific to AIME with this particular model and setup; the page doesn't claim this result generalizes to 'any reasoning benchmark.'",
            "Not stated or implied -- there's no claim the baseline 'failed to converge'; it's presented as reaching a real, specific accuracy figure (40%) after its own training budget, a comparison point, not a description of a broken run.",
            "Correct. The real, verified numbers show BOTH a higher accuracy (67% vs. 40%) AND fewer training steps (400 vs. 1080) for ReTool's approach on this benchmark -- roughly a third of the steps for a substantially better result, which is precisely the comparison the page reports.",
            "Not supported by the given comparison -- the page attributes the result to ReTool's RL training approach (the cold-start-plus-RL method and outcome-based refinement), not to model size; no model-size confound is described in this comparison."
          ],
          "source": "RL for Search and Tool Agents",
          "sourceUrl": "rl-search-tool-agents.md"
        },
        {
          "scenario": "A prep repository's README states that its per-company interview question lists are 'synthesised from this company's publicly known focus areas and role descriptions \u2014 not leaked questions,' the same disclosure appearing on every company's page in the repo.",
          "question": "What's the most accurate way to use this repository's content in interview prep?",
          "options": [
            "As a topic map of likely focus areas, not evidence any specific question was asked",
            "As a real record of confirmed questions, since dozens of named companies are covered",
            "As equally credible as a first-person candidate report of the same company",
            "As entirely worthless and not worth reading, given the synthetic disclosure"
          ],
          "correct": 0,
          "explanations": [
            "Correct. The repo's own stated purpose fits this use -- a synthesized list built from public focus areas is genuinely useful for understanding what a company likely emphasizes, without the false confidence of treating it as a report of real questions.",
            "Contradicts the source's own explicit disclosure -- the repository states plainly that its content is synthesized, not leaked or reported, which is the opposite of a confirmed record.",
            "Directly contradicts the graded distinction this page draws between quality tiers -- 'synthetic' and 'candidate-reported' are meaningfully different levels of evidence precisely because one describes a real event and the other doesn't.",
            "Overreacts to the disclosure -- content being synthesized rather than leaked doesn't make it valueless as a topic guide; it just shouldn't be mistaken for a report of an actual interview."
          ],
          "source": "Interview Playbook",
          "sourceUrl": "interview-playbook.md"
        },
        {
          "scenario": "Sierra candidate reports from May 2025 describe a take-home support agent build plus a TypeScript/React debugging round. Sierra's own company blog post from April 2026 describes a restructured onsite: a 2-hour AI-native build session, with the coding phone screen replaced by system design.",
          "question": "What's the most accurate conclusion a candidate preparing today should draw from having both sources?",
          "options": [
            "Neither source is trustworthy, since the discrepancy means Sierra's process can't be prepared for",
            "The 2026 company post describes the current process; the 2025 reports describe a loop that changed",
            "Both sources are equally current, since candidate reports are generally more reliable than company blogs",
            "The 2025 reports should be trusted over the 2026 post, since candidate reports outrank company statements"
          ],
          "correct": 1,
          "explanations": [
            "Overcorrects into unwarranted skepticism -- a documented change over time isn't a sign either source is unreliable; it's exactly the kind of dated evidence this page uses to show loops can change.",
            "Correct. The company's own dated, published description of a restructured process is the more current evidence -- the older candidate reports describe a real loop, just one the company's later post indicates has since changed.",
            "Ignores the actual dates -- treating an 11-months-earlier candidate report as equally current as the company's own later description of a changed process risks preparing for a round (coding phone screen) that no longer exists.",
            "States an absolute rule not supported by the material -- source type alone doesn't determine reliability; recency and specificity matter here."
          ],
          "source": "Interview Playbook",
          "sourceUrl": "interview-playbook.md"
        },
        {
          "scenario": "A candidate notes that MCP appears in 9.4% of AI-tech job postings across 61 distinct companies, but has zero candidate-confirmed interview questions in the reachable evidence. They conclude: 'Since MCP isn't confirmed in any real interview report, it's safe to skip preparing for it.'",
          "question": "What's the strongest problem with that conclusion?",
          "options": [
            "The conclusion is reasonable -- no candidate confirmation means the topic is genuinely unlikely to come up",
            "MCP should be deprioritized in favor of any topic with even one narrow candidate confirmation",
            "61 companies naming MCP is relevance evidence; the gap reflects unreachable sources, not that it's unasked",
            "JD frequency and candidate confirmation should always align, so the mismatch means the JD data is unreliable"
          ],
          "correct": 2,
          "explanations": [
            "Draws too strong a conclusion from an acknowledged evidence gap -- this page is explicit that the candidate-confirmed sample is thin due to unreachable platforms, not a complete record of what's asked.",
            "Overcorrects into a rule that ignores confirmation strength -- a topic confirmed once, narrowly, isn't automatically more prep-worthy than one present across dozens of current job descriptions.",
            "Correct. The JD data is real, current evidence of relevance even without a matching interview report, and this page states the candidate-confirmed gap likely understates real coverage since major platforms were unreachable -- absence of confirmation isn't evidence of absence.",
            "Misapplies the data -- job descriptions and live interview content answer different questions (what a role needs long-term vs. what's tested live), which is why this page tracks them separately, not a sign either dataset is broken."
          ],
          "source": "Interview Playbook",
          "sourceUrl": "interview-playbook.md"
        },
        {
          "scenario": "A candidate reads that Meta's AI-assisted coding round grader reportedly said 'I don't think my interviewer cared how much code was written by AI vs. me. They cared more about how well I partnered with AI.' They also read that Anthropic's live rounds explicitly disallow AI tools. They conclude: 'AI tool policy must be consistent across all technical companies, so one of these two reports must be wrong.'",
          "question": "What's the most accurate response to this reasoning?",
          "options": [
            "Correct -- since the two policies contradict, at least one source must be inaccurate or outdated",
            "Meta's policy must be the outdated one, since AI-assisted rounds are the newer, rising trend",
            "The two reports are compatible if Anthropic only disallows AI tools in non-technical rounds",
            "Each policy is independently documented and genuinely company-specific, not a contradiction"
          ],
          "correct": 3,
          "explanations": [
            "Assumes an industry-wide standard not supported anywhere in the material -- a real, documented split (OpenAI's scoped exception, Meta's dedicated round, Sierra's restructure, Anthropic's prohibition) is the actual finding, not a contradiction to resolve.",
            "Fabricates a directional claim -- nothing establishes Anthropic's policy is 'outdated' rather than a deliberate, current choice; the rising-trend framing describes AI-assisted coding's spread, not that holdouts are behind.",
            "Invents an unsupported qualification -- the Anthropic quote describes live rounds generally, with no technical/non-technical distinction stated.",
            "Correct. Each company's AI-tool policy is independently documented from its own source -- genuine variation across companies is exactly what the evidence shows, not an error needing resolution."
          ],
          "source": "Interview Playbook",
          "sourceUrl": "interview-playbook.md"
        }
      ]
    }
    </script>
    </div>

=== "Flashcards"

    267 flashcards from every page with a deck so far — click a card to flip it, shuffle for random order.

    <div class="flashcard-widget" data-title="Flashcards — All Pages">
    <script type="application/json">
    {
      "cards": [
        {
          "front": "What is Anthropic's distinction between a workflow and an agent?",
          "back": "A workflow orchestrates LLMs and tools through predefined code paths. An agent is a system where the LLM dynamically directs its own process and tool usage. The line is who decides the next step: fixed code, or the model.",
          "source": "What Is an Agent?"
        },
        {
          "front": "Is a generate-grade-retry loop (fixed code decides whether/how many times to retry) an agent?",
          "back": "No. Anthropic lists this as the 'evaluator-optimizer' workflow pattern. It loops and uses LLM calls, but the retry decision is governed by fixed code, not the model choosing its own next action.",
          "source": "What Is an Agent?"
        },
        {
          "front": "Give three different definitions of 'agent' from different sources.",
          "back": "Anthropic: dynamically directs its own process and tool usage. Willison: runs tools in a loop to achieve a goal. Huyen: perceives and acts on an environment. OpenAI: independently accomplishes tasks on your behalf.",
          "source": "What Is an Agent?"
        },
        {
          "front": "What did Menlo Ventures' Dec 2025 enterprise survey find about how many production 'agents' actually qualify as agents?",
          "back": "Only 16% of enterprise deployments and 27% of startup deployments met the bar for a true agent; most were 'if-then logic around a model call.'",
          "source": "What Is an Agent?"
        },
        {
          "front": "Name three reasons NOT to build something agentic even when it's technically possible.",
          "back": "Latency (multiple model round-trips), cost (tokens scale with steps, a stuck loop burns budget), non-determinism (harder to test/guarantee output shape), and harder evaluation/debugging.",
          "source": "What Is an Agent?"
        },
        {
          "front": "In the LLM-agent vs. classical-RL-agent comparison, what is the main improvement mechanism for each?",
          "back": "Classical RL: gradient updates from a reward signal over many episodes. LLM agent: in-context learning, reflection, and occasionally fine-tuning on trajectories (GRPO/DPO), applied on top of the loop rather than instead of prompting.",
          "source": "What Is an Agent?"
        },
        {
          "front": "A 'customer support agent' does classify intent -> retrieve FAQ -> generate response, with no branching. Is it agentic? What would make it agentic?",
          "back": "No, it's a fixed workflow. Adding a judge step (does the retrieved FAQ actually answer the question?) plus letting the MODEL pick the next action (broaden search, ask a clarifying question, escalate) from live options would make it agentic.",
          "source": "What Is an Agent?"
        },
        {
          "front": "What replaced the 2023 'AutoGPT wave' of fully autonomous agent loops?",
          "back": "The workflow-vs-agent distinction (Anthropic, Dec 2024) -- building bounded systems that reach for full agentic autonomy only where the task needs it -- followed by a mid-2025 shift toward context engineering.",
          "source": "What Is an Agent?"
        },
        {
          "front": "What does the model actually generate when it 'calls a tool'?",
          "back": "Structured text -- a tool_use block naming the tool and its arguments. The model never executes anything; your orchestration code parses the block, runs the real function, and sends the result back as a new tool_result message.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "What does stop_reason: \"tool_use\" mean in Anthropic's API?",
          "back": "The model's turn is a request for a tool call, not a final answer -- your code should parse the tool_use block(s) in the response, execute them, and send results back rather than treating the response as done.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "Why does a tool description with a worked example (e.g. \"'127.50 * 1.18' to add an 18% tip\") outperform a bare type signature?",
          "back": "The tool description functions as a prompt the model actually reads; a concrete example measurably improves how reliably the model extracts and formats the right arguments, versus a vague 'expression: string' signature.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "Why shouldn't a tool's error handler return a raw stack trace or fail silently?",
          "back": "Neither lets the model act sensibly -- a structured error (e.g. {\"error\": \"...\"}) becomes a normal tool_result the model can read and decide how to respond to (retry, try something else, give up honestly), rather than crashing the loop or leaving the model to hallucinate a result.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "Does a max_iterations cap on an agent loop protect against a single tool call that hangs for 90 seconds?",
          "back": "No. An iteration cap bounds the number of round-trips (total cost/runaway risk), not the duration of any single call. Production loops need both a step budget and an independent per-tool timeout.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "A team with 40 tools sees frequent wrong-tool selection. An engineer proposes collapsing all 40 into one free-text 'do_anything' tool with internal keyword dispatch. What's the flaw?",
          "back": "It doesn't remove the selection problem, it hides it -- moving the decision from the model's schema-validated tool choice into an internal keyword matcher that's typically less capable at disambiguation and much harder to debug, while also losing argument validation.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "In this page's real run, the no-tools baseline gave a range ($40.80-$41.50) instead of a precise wrong number. Why?",
          "back": "It got every arithmetic step right on its own but had no live exchange rate -- a data-access gap, not a reasoning failure -- so it honestly hedged with a range rather than asserting false precision.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "In the same real run, why did two calculate tool calls both appear with the same step number in the trace?",
          "back": "The model issued them in parallel within a single turn -- Anthropic's tool use allows multiple tool_use blocks in one response by default, and the loop logs every tool call from that turn under that turn's step count.",
          "source": "The Agent Loop From Scratch"
        },
        {
          "front": "Why do all five of Anthropic's named workflow patterns stay workflows, not agents, even when an LLM makes a decision inside them?",
          "back": "The consequence of that decision is wired in by the developer ahead of time (a gate's pass/fail branches, a router's fixed handlers, a capped retry count) -- the model's judgment fills in a pre-built slot, it doesn't choose the slot itself.",
          "source": "Workflow Patterns"
        },
        {
          "front": "What does a 'gate' add to a chaining pipeline that a single generate-then-expand pipeline lacks?",
          "back": "An explicit, checkable stop point: a bad intermediate result is caught and surfaced before the expensive final step runs, instead of silently propagating into (and being buried inside) the final output.",
          "source": "Workflow Patterns"
        },
        {
          "front": "What are the two distinct uses of parallelization Anthropic names?",
          "back": "Sectioning (different pieces answer different sub-questions about the same input) and voting (the same question run multiple times, aggregated for a more confident answer).",
          "source": "Workflow Patterns"
        },
        {
          "front": "What actually separates orchestrator-workers from parallelization -- it is NOT concurrency.",
          "back": "Who decides the set of sub-tasks, and when. Parallelization uses a fixed, developer-chosen set of sub-tasks. Orchestrator-workers uses a planning call that decides the sub-tasks dynamically, per input. Both can run their sub-tasks concurrently -- that's orthogonal to the distinction.",
          "source": "Workflow Patterns"
        },
        {
          "front": "In a real run, the same orchestrator-workers planner was given two different topics. What did it produce, and why is that the actual finding (not the sub-task count)?",
          "back": "Two differently-SHAPED breakdowns -- a comparison-shaped split for 'arrays vs linked lists', a derivation-shaped split for 'how binary search achieves O(log n)'. Both happened to have 4 sub-questions (coincidence) -- the finding is that the planner read each input and designed a decomposition to fit it, unprompted.",
          "source": "Workflow Patterns"
        },
        {
          "front": "Does an evaluator in the evaluator-optimizer pattern have to be an LLM call?",
          "back": "No. A deterministic check (e.g. word count, banned-word list) works fine when the pass/fail criterion is exact -- this page's real demo uses a non-LLM evaluator.",
          "source": "Workflow Patterns"
        },
        {
          "front": "A real timed run measured parallelization's speedup as 2.6x (4.79s sequential vs 1.86s concurrent for 3 calls), not a clean 3x. Why report the real number instead of the theoretical one?",
          "back": "The theoretical N-times speedup ignores real network and queueing overhead; reporting the measured 2.6x is the honest number a reader can actually expect, not an idealized one.",
          "source": "Workflow Patterns"
        },
        {
          "front": "An evaluator-optimizer loop passes on its first attempt over 95% of the time after shipping. Is that evidence to remove it?",
          "back": "No -- a low retry rate is what a correctly-tuned safety net looks like: negligible cost on the common case, catching the rare real failures it was built for. Rarely firing is success, not proof it's unnecessary.",
          "source": "Workflow Patterns"
        },
        {
          "front": "How are 'reasoning paradigms' (ReAct, Reflexion, ReWOO, ToT...) a different axis from Anthropic's workflow patterns?",
          "back": "Workflow patterns fix the topology -- which steps exist, in what order. Reasoning paradigms change how a single step or loop reasons, searches, or recovers from mistakes -- several of them (ReWOO, ReAct) could sit inside any workflow pattern as the mechanism one step uses internally.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "Is Plan-and-Solve a planner/executor architecture, like the interview-prep seed doc originally described it?",
          "back": "No -- it's a zero-shot PROMPTING technique: one instruction to one model ('first devise a plan, then carry it out step by step'), compared against plain chain-of-thought. Don't confuse it with LangChain's separately-named Plan-and-Execute agent, which really is a two-role architecture.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "What does ReWOO's paper actually claim -- token efficiency or latency?",
          "back": "Token efficiency ('5x token efficiency... on HotpotQA'), and accuracy. It never claims a latency win. A real measured run showed ReWOO needing 2.5x fewer calls and 6.8x fewer tokens than ReAct on the identical question and model.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "A real ReWOO run generated a 6-step plan, and step E6 (round(E5, 2)) failed because the calculator tool didn't support function calls. What does this reveal about ReWOO's actual failure surface, versus ReAct's?",
          "back": "ReWOO commits its whole plan before any of it runs, so a bad step (assuming a tool capability that doesn't exist) is only caught once it executes. ReAct decides one action at a time based on the real previous result, so it would see that failure before committing further -- the trade-off is fewer round-trips (ReWOO) versus earlier error detection (ReAct).",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "What does LLM Compiler add on top of what ReWOO already does?",
          "back": "Concurrent dispatch of a plan's independent steps. ReWOO's own plan executes in written order even when steps have no dependency on each other; LLM Compiler analyzes the dependency graph and runs independent steps in parallel -- a wall-clock win, not a token-count win (the same evidence still feeds one final solve call either way).",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "A real Reflexion demo (a 9-swap key-tracking puzzle) got the right answer on attempt 1 -- no reflection ever fired. Is that a failed demo?",
          "back": "No -- an honest result, not a weaker one. The mechanism is real and the code path exists; this run's failure precondition (a wrong first attempt) just didn't occur. What's notable is WHY: the model wrote out intermediate state after every swap instead of tracking it mentally, which is exactly the kind of externalized bookkeeping that avoids the errors this task risks.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "DeepSeek-R1's reported finding is that self-reflection and verification can emerge from pure RL. Does this make Reflexion-style explicit retry loops obsolete?",
          "back": "No, and this is a common overreach. A reasoning model's internal self-correction isn't inspectable or interruptible the way an explicit loop is, isn't available on models without that training, and says nothing about ReWOO/LLM Compiler's separate problem (token/call economics across tool calls). What changed is which problems are worth hand-building scaffolding for -- not that the paradigms became pointless.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "Tree of Thoughts reports GPT-4 + CoT solving 4% of Game-of-24 tasks versus 74% with ToT search. Why doesn't this page build a fresh code demo for ToT and LATS?",
          "back": "The tradeoff (breadth of search bought with a multiplicative call budget per branch explored) doesn't need a bespoke run to establish -- it's inherent to the algorithm, the same way parallelization's speedup was worth measuring but its existence wasn't in question. The real cited numbers plus the mechanism explanation carry the point without spending extra API calls to re-prove something structural.",
          "source": "Reasoning Paradigms"
        },
        {
          "front": "A real identical tool-calling task was solved with the raw Anthropic client, LangChain/LangGraph's create_agent, and Pydantic AI's Agent. What were the real measured lines of orchestration code and call counts?",
          "back": "15 lines / 2 calls (raw), 8 lines / 2 calls (LangGraph), 5 lines / 2 calls (Pydantic AI). All three got the correct answer on the first real run -- the line-count gap tracks how much of the loop each approach makes you write by hand vs adopt as-is, not code quality.",
          "source": "Choosing a Framework"
        },
        {
          "front": "What is 12-Factor Agents' actual named position on frameworks, and what's Anthropic's own middle-ground framing via the Claude Agent SDK?",
          "back": "12-Factor Agents: most products calling themselves agentic 'aren't that agentic' -- production reliability comes from owning your own control flow. Anthropic's Claude Agent SDK: 'Loop = gather context -> take action -> verify work' -- a real SDK (not raw), but one exposing Claude Code's own harness rather than hiding the loop behind an unrelated abstraction.",
          "source": "Choosing a Framework"
        },
        {
          "front": "What does AutoGen's own README say about its current status, verbatim?",
          "back": "'AutoGen is now in maintenance mode. It will not receive new features or enhancements and is community managed going forward. New users should start with Microsoft Agent Framework.' A direct, current, first-party statement -- weigh heavily against choosing it for new work.",
          "source": "Choosing a Framework"
        },
        {
          "front": "A real dependency conflict: installing the full pydantic-ai package alongside this cookbook's python-dotenv==1.0.1 pin failed. Why, and what was the real fix?",
          "back": "pydantic-ai's mcp extra pulls in fastmcp-slim, which requires python-dotenv>=1.1.0. Fix: pydantic-ai-slim[anthropic] (skip the unneeded mcp extra) -- avoids pulling in the conflicting dependency entirely, since this recipe never uses MCP.",
          "source": "Choosing a Framework"
        },
        {
          "front": "What real, disclosed, out-of-the-box behavior did Pydantic AI show that's worth knowing before you see it in your own logs?",
          "back": "It prints a startup banner to stdout by default (framework version, model, tool count, a nudge toward its paid Logfire observability product) unless PYDANTIC_AI_NO_BANNER=1 is set.",
          "source": "Choosing a Framework"
        },
        {
          "front": "Real framework adoption across 1,978 AI-technical job postings: what were LangChain's and LangGraph's real percentages and company counts?",
          "back": "LangChain: 8.1% of postings, 28 distinct companies. LangGraph: 5.0%, 19 distinct companies. Real, current, non-trivial adoption -- but the majority of AI-technical roles name no specific agent framework at all.",
          "source": "Choosing a Framework"
        },
        {
          "front": "What is Microsoft Agent Framework (MAF) the stated successor to, and what's its own claimed status?",
          "back": "Successor to BOTH AutoGen and Semantic Kernel. Its own framing: 'Microsoft Agent Framework is now available at version 1.0 as a production-ready release: stable APIs, and a commitment to long-term support.'",
          "source": "Choosing a Framework"
        },
        {
          "front": "What makes smolagents structurally different from most other agent frameworks, and what's a real, disclosed cost of adopting it?",
          "back": "'Agents that think in code' -- the model writes and executes real Python as its action, rather than emitting structured tool-call JSON; the only actively-maintained framework built specifically around that design. Real cost: release cadence has slowed to roughly one every couple of months.",
          "source": "Choosing a Framework"
        },
        {
          "front": "What's the practical test Anthropic gives for whether a tool description is good enough?",
          "back": "Describe the tool the way you'd describe it to a new hire on your team -- make implicit context explicit. Reported result of applying this rigorously: tool-description refinements alone took Claude Sonnet 3.5 to state-of-the-art on SWE-bench Verified, with no model or architecture change.",
          "source": "Tool Design"
        },
        {
          "front": "A real run measured endpoint wrappers (list_tasks + list_comments, model composes) against a consolidated tool (get_blocked_tasks_with_reasons, one call) on the identical question. What were the real numbers?",
          "back": "Endpoint wrappers: 3 calls, 3,079 tokens. Consolidated: 2 calls, 1,563 tokens -- consolidated won on both axes, roughly half the tokens.",
          "source": "Tool Design"
        },
        {
          "front": "In the same real run, code execution (model writes Python against a small library) tied endpoint wrappers on calls (3 each) and only modestly beat it on tokens. Does this mean code execution isn't worth it?",
          "back": "No -- this is an honest, unglamorous result specific to ONE fixed, well-anticipated question. Code execution's real advantage is generalizing to query shapes nobody pre-built a tool for, which a single fixed question can't demonstrate either way -- it's a scaling argument across many different queries, not a per-call efficiency claim.",
          "source": "Tool Design"
        },
        {
          "front": "A real comparison ran the same ambiguous request against a tool with a terse validation error ('ValidationError: priority') and an actionable one (naming the valid options). Both conditions took exactly 2 calls to recover. Does this mean the error message quality didn't matter?",
          "back": "No -- both took the same number of TURNS, but recovered to different VALUES. The terse error (no info) got a generic, disconnected guess ('high'). The actionable error (listed valid options) got a genuinely closer match to what the user meant ('urgent'). The real effect was on recovery quality, not recovery speed.",
          "source": "Tool Design"
        },
        {
          "front": "SWE-agent reports a 12.5% pass@1 on SWE-bench, described as 'far exceeding' prior non-interactive LM approaches. What is this improvement actually attributed to?",
          "back": "The custom Agent-Computer Interface (ACI) itself -- file navigation, editing, and test-execution commands built specifically for how a language-model agent uses them -- not a stronger underlying model. The paper's framing: LM agents are a distinct category of end user needing interfaces built for their own needs, not repurposed human interfaces.",
          "source": "Tool Design"
        },
        {
          "front": "What does Anthropic's 'errors as informative feedback' guidance actually recommend, versus a typical error code or traceback?",
          "back": "Errors should give specific and actionable improvements -- for example, naming what a valid value actually looks like, or how to construct a more targeted retry -- rather than an opaque error code or a raw traceback the model has to guess the meaning of.",
          "source": "Tool Design"
        },
        {
          "front": "Anthropic's Tool Search mechanism (loading only relevant tool definitions instead of the full library) reports what real numbers?",
          "back": "85% token reduction, and moved one internal MCP evaluation's accuracy from 49% to 74% on Opus 4 (79.5% to 88.1% on Opus 4.5).",
          "source": "Tool Design"
        },
        {
          "front": "Why does this page treat 'consolidate your tools' and 'give the model code execution instead' as two different answers to two different problems, rather than one technique beating the other?",
          "back": "Consolidation wins when the query shape is known in advance -- you build exactly the right tool once. Code execution's case is when query shapes are diverse or unpredictable -- primitives the model composes in code cover shapes nobody pre-built a tool for. The real measured run showed consolidation winning on ONE fixed question precisely because that question's shape was already known; it doesn't settle which approach wins under a broader, less predictable query mix.",
          "source": "Tool Design"
        },
        {
          "front": "What is 'context rot,' and why is it a real, physical constraint rather than just a metaphor?",
          "back": "As the number of tokens in context grows, the model's ability to accurately recall and use information from it decreases -- rooted in the transformer's attention mechanism computing n\u00b2 pairwise relationships between tokens, so there's a real, finite 'attention budget' being spent regardless of how large the context window technically allows.",
          "source": "Context Engineering"
        },
        {
          "front": "What are the four verbs in LangChain's context engineering framework, and one example of each?",
          "back": "Write (save outside the context window -- scratchpads, memories), Select (pull the right thing back in -- RAG, JIT loading via lightweight identifiers), Compress (keep only what's needed -- summarization, trimming), Isolate (split it up -- sub-agents with their own context, sandboxed execution).",
          "source": "Context Engineering"
        },
        {
          "front": "Breunig's four context failure definitions: poisoning, distraction, confusion, clash. Give each in one line.",
          "back": "Poisoning: a hallucination makes it into context. Distraction: the context overwhelms the training (pulls the model toward repeating an in-context pattern instead of reasoning fresh). Confusion: superfluous context influences the response. Clash: parts of the context disagree.",
          "source": "Context Engineering"
        },
        {
          "front": "A real poisoning repro fed a fake 2009 founding year into context; a downstream question got the wrong age (17, not 12). Why was quarantine (replacing the bad message) the right fix rather than appending a correction after it?",
          "back": "Leaving both the false and corrected facts in context turns a poisoning problem into a CLASH problem -- and clash resolution isn't guaranteed to favor the correct or newer fact. Quarantine removes the bad premise entirely rather than trading one failure mode for another.",
          "source": "Context Engineering"
        },
        {
          "front": "A real distraction repro showed six turns confidently applying a wrong averaging method (divide by N-1). The new question's real answer wasn't the predicted pattern-matched wrong number -- it was something else entirely (the raw, undivided sum). Was this a failed demo?",
          "back": "No. A real wrong answer occurred under the flawed-history condition and vanished once that history was compressed away -- that's real evidence of context-caused failure and a real working fix, even though the SPECIFIC mechanism (exact pattern-copying) didn't match what was hypothesized going in. Reported honestly rather than adjusted to fit the prediction.",
          "source": "Context Engineering"
        },
        {
          "front": "A real confusion repro (12 tools, including a plausible stale-data decoy) found NO measurable confusion -- same correct tool, same correct answer, as the 1-tool clean condition. Does this mean tool confusion isn't real?",
          "back": "No -- it's an honest negative result scoped to a specific scale. The cited literature (RAG-MCP) reports confusion effects starting around 30+ tools; 12 tools is below that threshold, so a clean result here doesn't contradict the literature's claim, it just wasn't positioned to detect the same effect.",
          "source": "Context Engineering"
        },
        {
          "front": "A real clash repro gave the model two contradictory facts, one explicitly labeled as 'supersedes' the other -- and the model resolved it correctly both with and without the stale fact present. What's the real limitation of this specific result?",
          "back": "The explicit 'supersedes' label is a resolution CUE -- this tests whether the model can follow a labeled signal correctly, not whether it can resolve a genuinely unlabeled, ambiguous contradiction with no signal about which fact is current. That harder case is real, disclosed future work, not something this repro tested.",
          "source": "Context Engineering"
        },
        {
          "front": "Why does this page treat 'two of four repros found no failure' as a strength of the demo rather than a weakness?",
          "back": "A demo where every hypothesized failure reproduces exactly as predicted, at trivial scale, every time, would be the more suspicious result. Running real experiments -- and reporting the real, specific, disclosed reasons two of them didn't reproduce (scale below a documented threshold; an explicit resolution cue) -- is more informative than confirming a taxonomy always looks bad in a toy example.",
          "source": "Context Engineering"
        },
        {
          "front": "What is the Tool Search Tool's `defer_loading: true` mechanism, and what real token/accuracy numbers does Anthropic report for it?",
          "back": "Marked tools' full definitions aren't loaded into context upfront -- the model gets a search capability and pulls in a definition only once it's decided that tool is relevant. Real numbers: an 85% token reduction while keeping the full library reachable, and MCP evaluation accuracy improving from 49% to 74% (Opus 4) and 79.5% to 88.1% (Opus 4.5).",
          "source": "Tools at Scale"
        },
        {
          "front": "What does programmatic tool calling change that tool search alone doesn't?",
          "back": "Tool search controls WHICH tool definitions enter context; programmatic tool calling controls what happens AFTER a tool is called -- the model writes one program that chains multiple tool calls and prints only the result it wants back, instead of each result becoming its own conversational turn. Real measurement: 43,588 to 27,297 tokens, a 37% reduction on complex research tasks.",
          "source": "Tools at Scale"
        },
        {
          "front": "What's the real, cited token reduction for presenting MCP servers as code APIs instead of direct tool calls, and what's the second benefit beyond tokens?",
          "back": "150,000 to 2,000 tokens, a 98.7% reduction. The second benefit: intermediate results stay in the execution environment by default, so 'the agent only sees what you explicitly log or return' -- data the workflow doesn't want to share with the model can flow through without ever entering its context.",
          "source": "Tools at Scale"
        },
        {
          "front": "A real recipe measured naive (25 tools in context) at 4 calls / 9,011 tokens vs. tool-search (pre-filtered to 3) at 4 calls / 4,065 tokens, for the identical task and outcome. Since call count didn't change, where did the savings come from?",
          "back": "Entirely from not paying for the 22 irrelevant tool definitions present in context on every one of the 4 turns -- with an identical call count, the only remaining source of the token gap is what's billed on each call.",
          "source": "Tools at Scale"
        },
        {
          "front": "The same recipe's programmatic tool-calling condition finished in 3 calls / 3,743 tokens -- one FEWER call than the naive or tool-search conditions (4 each). Why did it save a call, not just tokens?",
          "back": "The model chained all three real tool calls (lookup_employee, check_license_availability, provision_access) inside one generated program instead of one call per tool -- collapsing what would be several turns of results-processing into a single call. Intermediate results never became separate conversation turns.",
          "source": "Tools at Scale"
        },
        {
          "front": "A hand-built keyword-overlap retriever for tool search initially returned only 1 of 3 required tools for its own task question, silently, with no error. What caused it, and what's the general lesson?",
          "back": "Generic words ('check', 'status') appeared in both the task question and several filler tools' descriptions, scoring those filler tools higher than the real tools' more specific curated keywords. General lesson: a hand-built retriever can fail on exactly the words that look most on-topic, and the failure is invisible -- a broken tool set with nothing in the API response to flag it.",
          "source": "Tools at Scale"
        },
        {
          "front": "How was the broken tool-search retriever bug caught and fixed, per this project's standing verification workflow?",
          "back": "Dry-tested with zero API calls first (printing what search_relevant_tools actually returned for the real task question), before any paid call was made. Fixed by filtering common words -- including 'check' and 'status' specifically -- from both the question and the fallback description-word scoring, then re-verified with another zero-cost dry run before spending a real call.",
          "source": "Tools at Scale"
        },
        {
          "front": "Why does a large tool library hurt more than just token cost, per Tool Design and this page?",
          "back": "More tools in context also means more opportunities for the model to confuse similarly-named or similarly-described tools with each other -- Anthropic's own numbers show this isn't hypothetical: filtering the library down (Tool Search Tool) didn't just cut tokens, it raised MCP evaluation accuracy (Opus 4: 49% to 74%), because a shorter, more relevant tool set is also easier to select correctly from.",
          "source": "Tools at Scale"
        },
        {
          "front": "What is the single largest architectural change in the MCP 2026-07-28 spec revision?",
          "back": "Making MCP stateless at the wire level: the initialize/notifications/initialized handshake and the Mcp-Session-Id header are removed from Streamable HTTP. Every request carries its own protocol version and capabilities in _meta, and list endpoints (tools/list, resources/list, prompts/list) no longer vary per-connection. Servers needing cross-call state use explicit, server-minted handles passed as ordinary tool arguments instead.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "A real repro found mcp==2.2.0 -- an SDK that explicitly targets the 2026-07-28 spec -- still returns 'Bad Request: Missing session ID' by default. What was the actual finding?",
          "back": "Spec-compliant statelessness is real and correctly implemented in the SDK, but it's opt-in (streamable_http_app(stateless_http=True)), not the default. 'Targets a spec' and 'defaults to that spec's behavior' are different claims -- confirmed directly via raw HTTP requests and real response headers, not assumed from the version number.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "What replaced server-initiated requests (roots/list, sampling/createMessage, elicitation/create) in the 2026-07-28 spec, and how does it work?",
          "back": "Multi Round-Trip Requests (MRTR): the server returns an InputRequiredResult carrying inputRequests; the client retries the ORIGINAL request, providing inputResponses plus an opaque requestState token the server minted. The server re-verifies that token on the retry.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "What does the requestState token cryptographically bind to, per the real SDK source (mcp/server/request_state.py)?",
          "back": "Method, target, and an argument digest (request-binding), the authenticated principal when available (principal-binding), and an expiry -- all sealed with AES-256-GCM authenticated encryption (AESGCMRequestStateCodec), so any tampering fails the AEAD authentication tag.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "A real repro replayed Alice's validly-sealed requestState token as a different user ('user:mallory') with the same cart ID. What happened, and why does it matter?",
          "back": "Rejected with 'principal' -- a real, working test of the spec's own named 'State Handle Hijacking' mitigation: 'MCP servers MUST NOT treat possession of a state handle as authentication' and 'SHOULD bind handles server-side to the authenticated user.' The repro confirmed this binding actually holds against the SDK's real crypto, not just as a documented requirement.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "Which three real, concrete things does the 2026-07-28 spec formally deprecate (12-month removal window), and what's the migration for each?",
          "back": "HTTP+SSE transport -> migrate to Streamable HTTP. Roots/Sampling/Logging features -> migrate to tool parameters, direct provider API integration, and OpenTelemetry/stderr logging respectively. OAuth Dynamic Client Registration -> migrate to Client ID Metadata Documents (DCR stays available for backward compatibility only).",
          "source": "MCP Deep Dive"
        },
        {
          "front": "What is 'token passthrough' in MCP's security model, and what does the spec require instead?",
          "back": "An anti-pattern where an MCP server accepts a client-supplied token without validating it was issued FOR the MCP server, then forwards it unmodified downstream -- reintroducing the confused-deputy problem. The spec requires, in MUST NOT language: 'MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server.'",
          "source": "MCP Deep Dive"
        },
        {
          "front": "Why does the real requestState repro's request-binding check (test 3: same token, different cart_id) matter as a DISTINCT property from tamper detection (test 2)?",
          "back": "Tamper detection catches a MODIFIED token (broken AEAD tag). Request-binding catches an UNMODIFIED, validly-sealed token being replayed against different arguments than it was minted for -- the token cryptographically commits to its original method/target/args, so even a legitimately-obtained token can't be reused for a different operation.",
          "source": "MCP Deep Dive"
        },
        {
          "front": "What is the official MCP registry, and how is it described in its own launch material?",
          "back": "registry.modelcontextprotocol.io -- launched in preview 2025-09-08, backed by Anthropic, GitHub, PulseMCP, Block and others. Its own description: 'an open catalog and API for publicly available MCP servers.' Unauthenticated reads via GET /v0/servers?search=<query>&limit=<n>.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "In this topic's real repro against the LIVE registry, what pattern emerged between natural-language queries and brand-name queries?",
          "back": "Natural-language capability queries ('browser automation form filling', 'payment charges status') reliably returned ZERO results. Brand/product-name queries ('Stripe', 'playwright', 'browser', 'puppeteer') returned real matches. Real conclusion: the registry's search behaves closer to keyword/brand matching than semantic search.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What real, separately-confirmed finding did a direct check of the registry reveal about Microsoft's own playwright-mcp server?",
          "back": "Microsoft's official playwright-mcp (37.5k+ GitHub stars) does NOT appear in the registry under any query tried, including a direct search=microsoft. A real, honest gap in a genuine preview-stage product -- even well-known servers can be absent, through no fault of the searcher.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "In the real repro's payment task, why did the agent pick the official com.stripe/mcp over a third-party wrapper (eu.nordicmcp/stripe) that had MORE tools (18)?",
          "back": "Real, stated reasoning: the official server 'avoids the added trust and dependency cost of a third-party intermediary' near payment data -- a security-aware judgment about trust, not a capability comparison. The agent picked the narrower-featured but first-party option specifically to avoid an extra intermediary touching sensitive data.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What real job do MCP gateways do that neither the base protocol nor the registry covers?",
          "back": "AuthN/authZ, per-tool/per-team rate limiting, audit logging, and routing across many MCP servers at once. Real, named examples: Bifrost, TrueFoundry MCP Gateway, Docker/Microsoft gateway offerings, ToolHive, agentgateway -- positioned as the 'enterprise readiness' layer the base spec deliberately leaves out.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What real shift is happening with how companies expose their own MCP servers, and what are 3 real, currently-live examples?",
          "back": "A shift from local stdio processes to hosted, remote Streamable-HTTP endpoints. Real, confirmed-live examples: GitHub (api.githubcopilot.com/mcp/, GA'd 2025-09-04), Stripe (mcp.stripe.com -- stops accepting non-Agent-tagged secret keys from 2026-10-31), Notion (mcp.notion.com/mcp -- old stdio server no longer maintained), and Cloudflare's own suite of a dozen+ scoped subdomains (docs.mcp.cloudflare.com, browser.mcp.cloudflare.com, etc.).",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What are the two real, different philosophies of MCP browser automation this topic's own repro surfaced?",
          "back": "Microsoft's playwright-mcp: runs LOCALLY, drives Playwright's accessibility tree directly ('no vision models needed, operates purely on structured data'), tools like browser_click/browser_snapshot. Browserbase's MCP server: runs in Browserbase's MANAGED CLOUD, built on Stagehand, resolves natural-language actions ('click the login button') to real DOM interactions -- the one this topic's own agent actually picked for its browser task.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What are the four real, separate layers this topic distinguishes in the MCP ecosystem, and why doesn't fixing one fix the others?",
          "back": "Registry (discovery -- what exists), gateways (access control/routing across known servers), remote servers (where code runs), tool platforms (integration aggregation, e.g. Composio's 1,000+/3,000+ self-reported toolkits/tools). A well-known but unlisted server is a REGISTRY gap a gateway can't fix; a rate-limit need is a GATEWAY problem a bigger registry can't fix either -- each layer solves a genuinely different problem.",
          "source": "MCP and Tool Ecosystem"
        },
        {
          "front": "What are the three real, documented levels of Agent Skills' progressive disclosure, and their real approximate costs?",
          "back": "Level 1: metadata (name+description), ALWAYS loaded, ~100 tokens/skill. Level 2: full SKILL.md body, loaded only when triggered, 'under 5k tokens.' Level 3+: bundled resources/scripts, loaded only as referenced -- for scripts, 'the script code itself never enters context,' only its output does.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "In this topic's own real repro, what were the real, measured token costs of all-upfront vs. progressive skill loading on a task needing only 1 of 3 skills?",
          "back": "all_upfront (all 3 full bodies in system prompt): 4,975 real input tokens. progressive (metadata only + on-demand read_skill tool): 3,069 real cumulative tokens across 2 turns -- and correctly loaded ONLY the relevant skill (pdf-forms), never the other two. Real 38.3% reduction, structural: scales with installed-but-irrelevant skill count.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What does A2A (Agent2Agent Protocol) standardize, and what is its real core discovery mechanism?",
          "back": "Communication and interoperability BETWEEN separate agents (potentially different vendors/frameworks) -- distinct from MCP's agent-to-TOOL relationship. Core mechanism: the Agent Card, a JSON doc at .well-known/agent-card.json 'describing the server's identity, capabilities, skills, service endpoint URL, and how clients should authenticate and interact with it.'",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What is A2A's real governance history?",
          "back": "Google announced A2A April 2025 with 50+ launch partners. Donated to the Linux Foundation's Agent2Agent Protocol Project by June 2025 -- 'A2A will remain vendor neutral, emphasize inclusive contributions and continue the protocol's focus on extensibility, security and real-world usability.' Named supporters: Google, AWS, Cisco, Salesforce, SAP, Microsoft, ServiceNow.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What does AG-UI standardize, and who created it?",
          "back": "The agent-to-USER-INTERFACE layer: 'an open, lightweight, event-based protocol that standardizes how AI agents connect to user-facing applications... bi-directional connection between a user-facing application and any agentic backend.' Created by CopilotKit, 'born from CopilotKit's initial partnership with LangChain and CrewAI' -- not one of the major model/framework vendors.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What is the Agent Client Protocol (ACP), and what's its own explicit analogy for what problem it solves?",
          "back": "Standardizes editor<->coding-agent communication. Own docs: 'ACP solves this by providing a standardized protocol for agent-editor communication, similar to how the Language Server Protocol (LSP) standardized language server integration' -- i.e. removes the N-times-M custom-integration problem. Local: JSON-RPC over stdio. Remote: HTTP/WebSocket. Created by Zed (Aug 2025), now community-governed.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What is AGENTS.md, and what real adoption figure is verified for it?",
          "back": "'A README for agents' -- a repo-root file with build steps/tests/conventions for AI coding agents, deliberately unstructured ('just standard Markdown... the agent simply parses the text you provide,' no schema). Real, verified: 'used by over 60k open-source projects,' now stewarded by the Linux Foundation.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What real protocol handles agentic payments, and what's its core trust mechanism?",
          "back": "Google's Agent Payments Protocol (AP2): 'an open protocol developed with leading payments and technology companies to securely initiate and transact agent-led payments across platforms.' Core mechanism: Mandates -- 'tamper-proof, cryptographically-signed digital contracts that serve as verifiable proof of a user's instructions,' proving an agent was actually authorized to spend.",
          "source": "Skills, A2A and Other Protocols"
        },
        {
          "front": "What is the exact cache prefix hierarchy Anthropic's prompt cache follows, and why does the order matter?",
          "back": "tools -> system -> messages. A change at any level invalidates that level AND everything after it. Since tools/system are usually stable across every turn while messages changes constantly, putting the volatile part last means the stable, expensive part can be cached once and reused across an entire session.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "What are the real cost multipliers for a 5-minute cache write vs. a cache read (vs. base input token price)?",
          "back": "Cache write: 1.25x base input price. Cache read: 0.1x base input price on most models (as low as 0.025x-0.05x on some newer model families). The economics only pay off if a prefix is read far more times than it's written.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "A real repro changed ONLY tool_choice between two calls with an identical tools+system prefix. What happened to the cache, and why?",
          "back": "cache_read_input_tokens stayed exactly unchanged (2279, matching the prior call). tool_choice affects only the messages level, downstream of tools/system in the hierarchy, so it doesn't touch the hashed tools+system prefix at all.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "A real repro edited ONE WORD in one tool's description, with everything else (17 other tools, system prompt) unchanged. What happened to the cache?",
          "back": "Full invalidation -- cache_creation_input_tokens: 2283 (a fresh write covering the ENTIRE prefix), cache_read_input_tokens: 0. Modifying any tool definition invalidates the whole cache (tools, system, AND messages), per Anthropic's own documented rule, confirmed with real numbers.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "What is the mid-conversation-tool-changes beta, and what problem does it solve for caching?",
          "back": "A real, shipped beta (2026-07-24, expanded 2026-09-22) that lets you withdraw or add a tool via tool_removal/tool_addition blocks on a mid-conversation role:'system' message, instead of editing the top-level tools array. Since editing the tools array invalidates the entire cache, this lets tool availability change mid-conversation while the cached prefix stays byte-identical and intact.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "A real repro compared masking a tool (tool_removal block, array unchanged) vs. removing it (array physically edited). Same functional outcome -- what was the real cache difference?",
          "back": "Masking: three calls in a row (including one a full turn later) all read the IDENTICAL 2,211-token cache entry -- zero extra cost. Removal: produced a completely NEW, distinct 2,113-token cache entry with no relationship to what came before -- a full fresh write, every time the array changes.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "After masking a tool with tool_removal, a real repro forced tool_choice to that exact tool by name. What happened, and what does it prove?",
          "back": "A real 400 error: \"forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back).\" This proves masking is enforced by the API itself -- a genuine access-control change, not just a description Claude happens not to mention.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "A developer re-runs the same cache-testing script twice, a few minutes apart, with an identical prefix. The 'first' call in the second run shows a cache READ instead of the expected fresh WRITE. Bug or real behavior?",
          "back": "Real behavior, not a bug. Prompt cache entries have a TTL (5 minutes by default) and READING an entry refreshes that TTL. Repeatedly testing an identical prefix during development keeps the entry warm indefinitely, so a later 'first' call can legitimately hit a cache entry created by an earlier run.",
          "source": "KV-Cache Economics"
        },
        {
          "front": "What are CoALA's four memory types, in one line each?",
          "back": "Working memory: 'active and readily available information... for the current decision cycle' (the live conversation). Episodic memory: 'experience from earlier decision cycles' (what happened, in a specific instance). Semantic memory: 'an agent's knowledge about the world and itself' (how things work, generally). Procedural memory: implicit (LLM weights) and explicit (agent code) skills.",
          "source": "Memory Architectures"
        },
        {
          "front": "A real repro gave Claude's memory tool one customer-service case and asked it to record both the policy and the case outcome, with no mention of 'semantic' or 'episodic' anywhere. What happened?",
          "back": "The model unprompted split its own memory writes into two separate files -- a general-policy file and a case-specific file -- then correctly answered a later semantic-only question from one file and a separate episodic-only question from the other. The taxonomy emerged from the model's own organization, not from being told the categories.",
          "source": "Memory Architectures"
        },
        {
          "front": "What are MemGPT's real, cited numbers on Deep Memory Retrieval, and what's the real mechanism behind the gap?",
          "back": "GPT-4 + MemGPT: 92.5% accuracy. Fixed-context baseline: 32.1%. The mechanism: MemGPT's virtual context management moves data between 'main context' (in the window) and 'external context' (outside it) via model-generated function calls, triggered at a 'warning token count' threshold -- the baseline has no such mechanism.",
          "source": "Memory Architectures"
        },
        {
          "front": "What are Mem0's real, cited numbers vs. a full-context baseline?",
          "back": "26% relative improvement in the LJudge metric over OpenAI's own memory feature, 91% lower p95 latency, and over 90% token cost savings -- from extracting and consolidating salient facts rather than keeping full conversation history in context. Mem0g (graph variant) adds ~2% on top of base Mem0.",
          "source": "Memory Architectures"
        },
        {
          "front": "What does MINJA demonstrate about memory poisoning, and why does it matter that it's 'query-only'?",
          "back": "MINJA (arXiv:2503.03704) injects malicious records into an agent's memory bank purely by interacting through normal queries and observations -- no direct access to the memory store required. This means ANY user who can talk to the agent is a potential attacker, not just someone with backend access -- reported success rates above 95% in published results.",
          "source": "Memory Architectures"
        },
        {
          "front": "A real repro injected an identical poisoning mechanism as two different claims: an unverified $50,000 financial credit, and an unverified support-contact reassignment. What was the real, contrasting result?",
          "back": "The financial claim triggered SPONTANEOUS hedging with zero mitigation in place -- the model wrote its own 'unverified, recommend confirming in writing' caveat into memory. The operational claim was trusted with ZERO hedging -- written as flat fact and later used confidently with no caveat. The real vulnerability is claims that don't LOOK consequential, not claims in general.",
          "source": "Memory Architectures"
        },
        {
          "front": "What one system-prompt change closed the gap the untagged operational-claim poisoning left open, and how did it work?",
          "back": "Requiring every memory fact to be tagged [user-asserted] or [tool-verified], with an instruction to flag [user-asserted] facts before relying on them for anything consequential. It doesn't verify truth at write time -- it preserves PROVENANCE, so a later turn can apply scrutiny before acting, even though the claim itself was never independently checked.",
          "source": "Memory Architectures"
        },
        {
          "front": "Why is 'source-tagging' a stronger mitigation framing than 'validate claims before writing to memory'?",
          "back": "Many claims (like a routine contact reassignment) have no independent source to validate against at write time -- there's nothing to check truth against. Source-tagging sidesteps that impossible problem: it doesn't try to determine if a claim is TRUE, it just preserves WHERE the claim came from, so the consuming turn -- not the writing turn -- can decide how much to trust it.",
          "source": "Memory Architectures"
        },
        {
          "front": "A real repro tested Haiku 4.5 vs. Sonnet 5 on a deliberately ambiguous tool-calling request (a subscription 'pause' phrased as 'stop being charged'). What happened, across 5 trials each?",
          "back": "Both tiers picked the correct tool (pause_subscription) on 5/5 trials, and an unambiguous control case hit 5/5 on both for cancel_subscription. A clean, honest negative result -- this specific ambiguity didn't differentiate the two tiers, contrary to the assumption a cheaper model would slip.",
          "source": "Models for Agents"
        },
        {
          "front": "As of a June 2026 leaderboard snapshot, where did an open-weight model rank on the Berkeley Function-Calling Leaderboard (BFCL), and what's the real significance?",
          "back": "GLM 4.5 (open-weight) led at 76.7% overall accuracy, ahead of Claude Opus 4.7 (76.6%) and Gemini 3.1 Flash Lite Preview (76.5%). The significance: an open-weight model held the TOP spot on a major agentic tool-calling benchmark, not just 'closing the gap' -- a real, current, dated fact.",
          "source": "Models for Agents"
        },
        {
          "front": "What are RouteLLM's real, cited cost-reduction numbers, and under what condition do they hold?",
          "back": "'Over 2 times' cost reduction in certain cases, with reported per-benchmark reductions around 85% (MT Bench), 45% (MMLU), and 35% (GSM8K) -- all while holding to 95% of GPT-4's own performance level. The mechanism: route easy queries to a cheap model, reserve the expensive model for queries that actually need it.",
          "source": "Models for Agents"
        },
        {
          "front": "A real routing repro found all three strategies (always-Haiku, always-Sonnet, routed) tied at 4/4 correct on a batch including two classic reasoning traps. What were the real token costs, and what's the actual finding?",
          "back": "Always Haiku: 174 tokens. Always Sonnet: 216. Routed: 199. Since accuracy tied everywhere, the real finding isn't 'routing avoided mistakes' -- it's that routing matched the capable tier's accuracy while costing LESS than blanket escalation (199 vs 216), even though the cheap model alone would also have scored 4/4 for less.",
          "source": "Models for Agents"
        },
        {
          "front": "What are the two classic reasoning-trap questions this recipe used, and what are their tempting-wrong vs. correct answers?",
          "back": "Widgets/machines: '5 machines, 5 min, 5 widgets -> 100 machines, 100 widgets, how long?' Tempting wrong: 100 minutes. Correct: 5 minutes (parallel, same rate). Bat-and-ball: 'bat+ball=$1.10, bat costs $1.00 more than ball, ball=?' Tempting wrong: $0.10. Correct: $0.05.",
          "source": "Models for Agents"
        },
        {
          "front": "Haiku 4.5 solved the widgets/machines trap, the bat-and-ball trap, AND a third harder chickens-and-cows system-of-equations problem (correct answer 23), all on the first try. What's the honest lesson, rather than chasing a failure?",
          "back": "'Small model = unreliable at reasoning' is real but no longer automatic -- it needs checking per task and per model generation. Current-generation small models are meaningfully more capable at classic reasoning traps than that assumption implies; the right move is measuring, not assuming.",
          "source": "Models for Agents"
        },
        {
          "front": "Why does the recipe's small-scale routing result (single-digit % cost reduction) not contradict RouteLLM's cited '2x or more' cost reduction?",
          "back": "They measure different things at different scales: RouteLLM's number comes from routing across a large, varied benchmark distribution where many queries are genuinely simple. The recipe's illustrative 4-query batch -- half deliberately hard reasoning traps -- isn't positioned to reproduce that magnitude, even though the same underlying routing mechanism is genuinely at work in both.",
          "source": "Models for Agents"
        },
        {
          "front": "What's the real, generalizable discipline this topic argues for, given both repros' honest negative results?",
          "back": "Don't assume where the 'needs the expensive model' line falls -- measure it empirically, per task and per specific model pair, since that line moves as models improve. This recipe's own reasoning traps would plausibly have separated tiers a generation ago and didn't here; an assumption-based router would have escalated unnecessarily.",
          "source": "Models for Agents"
        },
        {
          "front": "What are the three real, cited benefits of plan-and-execute over a single ReAct-style loop, per LangChain's own framing?",
          "back": "Speed: 'the larger agent doesn't need to be consulted after each action.' Cost: sub-tasks 'can be made to smaller, domain-specific models.' Quality: 'forcing the planner to explicitly think through all the steps required' improves task completion.",
          "source": "Planning and Decomposition"
        },
        {
          "front": "A real repro tested a broken plan assumption (a repo with no git tag) under 'execute without second-guessing' vs. 'revise on broken assumptions' instructions. What happened?",
          "back": "Identical outcome both times -- both conditions correctly called list_all_merged_prs instead of the now-invalid tag-anchored query, and produced the same correct changelog. An honest negative: Sonnet 5's baseline behavior already adapted without needing the explicit replanning instruction.",
          "source": "Planning and Decomposition"
        },
        {
          "front": "A real granularity repro decomposed the same task three ways: no explicit plan, a well-sized 4-step plan, and an over-granular many-sub-step plan. What was the real, reproducible result?",
          "back": "Too-coarse and well-sized both succeeded (3 real tool calls each, correct changelog). Over-granular produced ZERO tool calls -- the model spent its entire token budget writing out sub-steps and never executed anything. Confirmed on two separate full runs.",
          "source": "Planning and Decomposition"
        },
        {
          "front": "What is the concrete, non-philosophical mechanism behind the over-granular planning failure?",
          "back": "Writing out a highly detailed plan consumes real tokens -- the same token budget the executor needs to actually make tool calls. A plan detailed enough can consume the entire response budget before a single real action happens. This is a measurable resource-competition failure, not an abstract 'too much planning is bad' argument.",
          "source": "Planning and Decomposition"
        },
        {
          "front": "What did the minimal verifier-in-the-loop check in this recipe, and why was it cheap?",
          "back": "One concrete fact: was draft_changelog (the output-producing tool) actually called? It ran against the trace each condition already produced -- no second model call, no re-running the task. It correctly passed too_coarse and well_sized, and failed over_granular with the real reason 'draft_changelog was never called -- no real output was produced.'",
          "source": "Planning and Decomposition"
        },
        {
          "front": "Why is 'always write a very detailed plan first' NOT a safe universal rule for agent reliability, per this topic's real repro?",
          "back": "The MOST detailed plan condition was the one that failed completely -- it never executed a single tool because all its budget went to articulating sub-steps. Plan granularity has a real cost (tokens spent on plan text) as well as a real benefit; the right size is task-dependent, not 'more detail is always safer.'",
          "source": "Planning and Decomposition"
        },
        {
          "front": "How does this topic's verifier-in-the-loop differ from Workflow Patterns' evaluator-optimizer pattern?",
          "back": "Evaluator-optimizer (Workflow Patterns) is a content-refinement loop: an evaluator judges generated content and triggers regeneration. This page's verifier is trace-checking: it inspects whether execution actually reached a concrete, checkable milestone (a specific tool call), not judging the quality of generated content -- a cheaper, more mechanical check.",
          "source": "Planning and Decomposition"
        },
        {
          "front": "How does a multi-agent SYSTEM differ from the orchestrator-workers WORKFLOW pattern, per Anthropic's own classification?",
          "back": "Orchestrator-workers is a fixed workflow: a central LLM call delegates to workers executing pre-built roles. A multi-agent system's subagents are themselves autonomous agents -- own tool access, own reasoning loop, own context window, making open-ended decisions -- not executing a pre-scripted function.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "A real experiment measured single-agent (1 call, 1,048 tokens) vs lead+3-subagents+synthesis (4 calls, 3,369 tokens) on three independent questions. What was the real ratio, and how does it compare to Anthropic's own reported figures?",
          "back": "3.21x measured. Smaller than Anthropic's reported 4x (agent vs chat) or 15x (multi-agent vs chat), but the same direction -- the difference is baseline and scale (real production workload vs chat, vs this page's small single-agent-does-everything toy baseline), not a contradiction.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "What's the real mechanism behind multi-agent's token overhead, visible in a real captured trace?",
          "back": "Each subagent call repeats system framing overhead (e.g. 'You are a research subagent...') that one combined call only pays once, AND the synthesis call has to re-read all subagent outputs in full before producing the final answer -- tokens a single pass never spends at all.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "A real test of Cognition's inter-agent consistency risk (two blind subagents inventing a shared fact) found NO disagreement, twice -- even after redesigning the prompt to avoid an obvious common default. Does this disprove Cognition's concern?",
          "back": "No -- honest negative result with a real, disclosed reason: the repro tested convergence on a SCALAR NUMBER, which the same model tends to agree with itself on even when blind. Cognition's own example (Flappy Bird: a Mario-style background vs a bird that doesn't match) is about open CREATIVE interpretation -- genuinely higher-variance, no shared convention to fall back on. Different, harder test than this repro ran.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "MAST's taxonomy: 14 failure modes across 3 categories. Name the three categories and one real finding.",
          "back": "System design issues (unclear roles/specs), inter-agent misalignment (agents talking past each other -- the category Cognition's Flappy Bird example fits), task verification (nobody checks the final output is right). Built from 150 traces (kappa=0.88), validated against 1,600+ traces across 7 frameworks. Reports: 'performance gains on popular benchmarks are often minimal.'",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "MAST reports multi-agent gains are 'often minimal' on popular benchmarks. Anthropic reports a 90.2% improvement. Are these in tension?",
          "back": "No -- different scope. MAST is a broad average across many frameworks and task types. Anthropic's 90.2% is scoped to one breadth-first, parallelizable research task (identifying board members across S&P 500 IT companies) -- exactly the task shape multi-agent is well-suited to. A big win on a favorable task and a modest average across many tasks can both be true.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "What's the practical decision rule for 'when to use one agent' that both companies' real evidence, plus MAST, point toward?",
          "back": "Use multi-agent when sub-tasks are genuinely independent -- solvable correctly using only their own slice of context, no need to see what another piece decided (worth the measured token premium). Stay single-agent when steps are coupled -- a later step's correctness depends on an earlier step's reasoning path, not just its output (avoids a coordination problem multi-agent would have to solve explicitly).",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "Anthropic says token usage 'by itself explains 80% of the variance' in one benchmark's (BrowseComp) performance. What does this actually mean for evaluating a multi-agent win?",
          "back": "Much of multi-agent's measured advantage on that benchmark is attributable to spending more tokens, not to the architecture itself being smarter -- a caution against crediting 'multi-agent design' for a gain that a single agent given an equivalent token/compute budget might also achieve.",
          "source": "Multi-Agent Systems"
        },
        {
          "front": "What is a 'harness,' per Anthropic's own framing, and why does it matter distinct from the model?",
          "back": "'The system prompt, set of tools, and overall agent harness' together -- everything surrounding the model that makes a long-running agent actually work. Real finding: 'even a frontier coding model like Opus 4.5... will fall short... if it's only given a high-level prompt' -- the model alone isn't enough; the surrounding structure is load-bearing.",
          "source": "Harness Engineering"
        },
        {
          "front": "What two real failure patterns did Anthropic observe when a frontier model worked on a long, complex task with no harness structure?",
          "back": "(1) One-shotting too much at once, leading to context exhaustion mid-implementation. (2) Later sessions prematurely declaring the work complete. Compaction alone doesn't fix either -- 'compaction doesn't always pass perfectly clear instructions to the next agent.'",
          "source": "Harness Engineering"
        },
        {
          "front": "What three concrete artifacts does an initializer session create, per Anthropic's documented pattern?",
          "back": "An init.sh script (environment setup), a claude-progress.txt file ('keeps a log of what agents have done'), and an initial git commit (enables using git 'to revert bad code changes and recover working states'). Every later session reads the progress file first.",
          "source": "Harness Engineering"
        },
        {
          "front": "A real repro's session loop had a bug: it treated any non-tool_use stop reason as 'the agent is done.' What real failure did this cause, and how was it caught?",
          "back": "A response cut off by max_tokens mid-generation (right after the model said 'Now recording all six findings,' with zero actual recording calls that followed) was silently treated as a clean finish rather than a truncation. Caught by noticing the mismatch between the model's stated intent and the empty tool-call list. Fixed by explicitly checking stop_reason == 'max_tokens' as a distinct case.",
          "source": "Harness Engineering"
        },
        {
          "front": "A real repro ran session 1 (turn-limited, cut off before recording anything) then session 2 (genuinely fresh, no shared conversation). What did session 2 have to do, and why?",
          "back": "Re-do all 6 real tool lookups from scratch before it could record any findings -- because only the progress file, not the raw conversation, survives a real reset. Session 2 then correctly finished the checklist (remaining: 0), including unprompted flagging of an ambiguous result as needs_follow_up.",
          "source": "Harness Engineering"
        },
        {
          "front": "What is Anthropic's cited self-verification discipline, and what did a real repro find when testing it against a deliberately ambiguous result?",
          "back": "'Self-verify all features. Only mark features as passing after careful testing.' Real result: an honest negative -- both a plain instruction and the explicit self-verify instruction correctly flagged the same ambiguous result ('contact support for specifics') as needs_follow_up rather than passing. Sonnet 5's baseline judgment was already sufficient in this specific test.",
          "source": "Harness Engineering"
        },
        {
          "front": "Why is 'a session loop's own bug' worth documenting as a real harness-engineering lesson, not just an incidental coding error?",
          "back": "The bug (conflating max_tokens truncation with genuine completion) is itself exactly the class of failure a harness is meant to guard against: a harness that can't tell 'the agent decided it's done' apart from 'the agent got cut off mid-sentence' will silently corrupt the state it's supposed to protect -- precisely on-topic for what harness engineering is about.",
          "source": "Harness Engineering"
        },
        {
          "front": "What is SWE-agent's core argument for building a dedicated Agent-Computer Interface (ACI) rather than giving a model human developer tools?",
          "back": "'LM agents represent a new category of end users with their own needs and abilities, and would benefit from specially-built interfaces.' Design principles: actions should be 'compact and efficient,' feedback should be 'informative but concise,' and 'guardrails mitigate error propagation and hasten recovery.'",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "What was SWE-agent's real, measured cost of removing its linting guardrail?",
          "back": "SWE-bench Lite score dropped from 18.0% to 10.3% -- a real 7.7 percentage point loss from removing one interface feature (a code linter integrated into the edit function that alerts the agent to syntax mistakes it introduced).",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "A real repro tested the same linting-guardrail mechanism against Sonnet 5, across two independently redesigned edit tasks (a flat elif edit, then a genuinely harder nested-restructuring edit). What happened?",
          "back": "A clean tie both times: 100% valid syntax whether or not the guardrail was present. A dated, honest finding -- not a claim the guardrail is now universally unnecessary, but real evidence that current-model baseline reliability on small tasks may have shifted since SWE-agent's 2024 measurement.",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "A real repro's binary-search bug (hi = mid - 1 instead of hi = mid) initially passed hand-picked test cases even though it was real. How was this caught, and what's the lesson?",
          "back": "A 2,000-case random search against a known-correct reference implementation found the bug genuinely diverges on ~20% of random inputs. Lesson: a bug can be real and still pass test cases that don't happen to hit the inputs where it manifests -- verify against actually-divergent inputs, don't assume hand-picked cases are sufficient.",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "A real review-loop repro gave Sonnet 5 a subtle, verified-divergent binary-search bug to fix, with and without a real run_tests tool available. What was the real, independently-checked result?",
          "back": "Another clean tie: both conditions fixed the bug correctly on the first attempt, 5/5 trials each -- with zero ability to verify its own work in the no-review-loop condition. Every result was checked independently via a fresh subprocess, never from the model's own self-report.",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "What is METR's 'time horizon' metric, and what's the real reported doubling trend?",
          "back": "The length (for humans) of a task a model can complete with a given success rate (typically 50%). Real trend: 'a doubling time of around 7 months' over roughly six years of frontier models. Claude 3.7 Sonnet's measured horizon: 'approximately one hour.'",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "Why does this page treat four consecutive honest ties (two demos x two task variants each) as a real finding rather than a failed experiment?",
          "back": "Each tie was tested twice -- once on an easier task, once on a deliberately harder, independently redesigned variant -- specifically to check whether the mechanism just needed a harder task to reveal a gap. Both harder attempts also tied. A disciplined, bounded re-test (not indefinite retrying) that still comes back negative is real, honestly-reportable data.",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "Why is it methodologically wrong to conclude 'linting guardrails and review loops are now universally unnecessary' from this page's repros?",
          "back": "The repros tested small, illustrative tasks (5-line functions) against one current model (Sonnet 5) -- a narrow, scoped result. METR's own trend describes tasks stretching toward hours, days, and weeks; a guardrail with zero measured value on a tiny edit could easily matter again at a much larger, messier scale this page never tested.",
          "source": "Coding Agents: Mechanisms"
        },
        {
          "front": "What is AGENTS.md, and how does it relate to CLAUDE.md?",
          "back": "AGENTS.md is a real, open, cross-tool standard: 'a simple, open format for guiding coding agents,' like 'a README for agents.' Supported by many tools (Codex, Jules, Aider, opencode, Cursor, Copilot). Claude Code added direct support for reading AGENTS.md as of v2.1.277 -- a format born outside Anthropic that Claude Code later adopted.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "What does Anthropic's own documentation say about exactly how CLAUDE.md content is delivered to the model?",
          "back": "'CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself.' A real, specific mechanical detail -- not just 'a config file gets read somehow.'",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "What is the single most load-bearing quote distinguishing CLAUDE.md from hooks, per Anthropic's own docs?",
          "back": "'Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead.' And: 'Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer.'",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "A real repro tested CLAUDE.md-style project instructions (no f-strings, do_-prefixed names, imperative docstrings) delivered the real way. What was the measured compliance shift?",
          "back": "Without instructions: 0% do_-prefix compliance, inconsistent (67-100%) f-string avoidance. With instructions (delivered as a user message, matching real CLAUDE.md mechanics): 100% compliance on both, every trial, across multiple full runs -- a real, strong, measured effect.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "A real hooks repro's FIRST design gave the prompted condition an explicit rule ('never delete config.json'). What went wrong, and how was it fixed?",
          "back": "The model never violated the rule, so the hook never actually fired in any trial -- the comparison tested rule-obedience, not what a hook protects. Fixed by removing the protective rule from BOTH conditions entirely, isolating exactly what the hook adds when nothing else does.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "After the redesign (no protective rule anywhere), what was the real result with vs. without the hook, given an explicit request to delete a file?",
          "back": "Without a hook: 0% survival (5/5 trials, the model complied and deleted the file given a reasonable-sounding explicit request). With a real, deterministic PreToolUse-style check: 100% survival (5/5 trials) -- the hook, not the model's judgment, is what held. The model also gracefully reported the block and suggested real alternatives.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "What real, current repositioning did primary-source research find for OpenHands?",
          "back": "OpenHands' own README now describes it not as an autonomous coding agent but as 'the self-hosted developer control center for coding agents and automations,' explicitly orchestrating 'OpenHands, Claude Code, Codex, Gemini, or any agent with Agent-Client Protocol' -- a genuine shift from being one agent to being a control plane above other vendors' agents, which older secondary sources likely miss.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "What real, product-level design choice does Cline make as its stated core differentiator?",
          "back": "'Every file edit and terminal command requires your approval, so you stay in control of what actually changes' -- approval-gated execution by default, with an opt-in toggle for autonomous mode. A direct, product-level instance of the human-in-the-loop theme.",
          "source": "Coding Agent Products and Configuration"
        },
        {
          "front": "What are Anthropic's real, verified numbers on their multi-agent Research system's performance vs. cost?",
          "back": "90.2% improvement over single-agent Opus 4 (Opus 4 lead + Sonnet 4 subagents) on their internal eval. But 'token usage by itself explains 80% of the variance' in that performance, and agents use ~4x more tokens than chat; multi-agent systems use ~15x more. Parallelization separately 'cut research time by up to 90% for complex queries.'",
          "source": "Deep Research Agents"
        },
        {
          "front": "What real, named failure mode does Anthropic document for multi-agent research systems, and their exact prescribed fix?",
          "back": "Failure: subagents 'performed the exact same searches as other agents... without an effective division of labor,' plus continuing when 'they already had sufficient results.' Fix (exact quote): 'Each subagent needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries.'",
          "source": "Deep Research Agents"
        },
        {
          "front": "In this topic's own real repro, what were the measured results of vague vs. scoped subagent instructions?",
          "back": "vague (identical, unscoped prompt to 3 subagents): 10 total retrievals, only 4/6 unique docs covered, 60% redundant retrieval. scoped (explicit objective+boundary per subagent): 7 total retrievals, ALL 6/6 docs covered, only 14.3% redundant -- Anthropic's own fix, applied literally, confirmed working on a real test.",
          "source": "Deep Research Agents"
        },
        {
          "front": "What is STORM's real, three-stage mechanism for generating long-form articles, and its measured gain vs. a single-pass baseline?",
          "back": "(1) Discover diverse perspectives on the topic. (2) Simulate conversations where each perspective 'poses questions to a topic expert grounded on trusted Internet sources.' (3) Curate into an outline. Real gain: '25% absolute increase' in articles judged well-organized, '10%' increase in breadth of coverage vs. a single-pass outline-then-write baseline.",
          "source": "Deep Research Agents"
        },
        {
          "front": "What is BrowseComp's real, stated design philosophy, and what does it reveal about model capability?",
          "back": "'Easy to verify, but hard to solve' -- 1,266 questions with a short, checkable answer but requiring genuinely persistent search to find. Real accuracy: GPT-4o 0.6%, GPT-4o+browsing 1.9%, o1 9.9%, OpenAI's own Deep Research model 51.5% ('solving around half the problems' where others were 'near-zero'). Humans gave up on 70.8% of 1,255 attempted questions within 2 hours.",
          "source": "Deep Research Agents"
        },
        {
          "front": "What real, open-source model reports a competitive BrowseComp score, and what's its architecture?",
          "back": "Alibaba's Tongyi DeepResearch: 30.5B total params, 3.3B active per token (MoE), trained with a customized GRPO-based RL approach. Reports 43.4% on BrowseComp -- real, substantial progress from an open model, though still behind the 51.5% frontier (OpenAI Deep Research) number.",
          "source": "Deep Research Agents"
        },
        {
          "front": "Why does this topic's repro use a curated per-document KEYWORD INDEX rather than raw prose token-overlap for its search tool?",
          "back": "A real bug was caught during dry-testing: raw substring matching produced false positives (e.g. the word 'in' matched inside 'endpoints'), and even after fixing to whole-word matching, generic connector words (e.g. 'employee', 'team') caused unrelated documents to match. An explicit, curated keyword set per document eliminated this noise, giving a clean, honest measurement of REAL subagent search behavior rather than search-tool artifacts.",
          "source": "Deep Research Agents"
        },
        {
          "front": "How does this topic connect to and differ from the existing Multi-Agent Systems topic?",
          "back": "Same orchestrator-worker architecture and same real Anthropic source. Multi-Agent Systems covers the TOKEN-COST multiplier (real 3.21x measured) and Cognition's inter-agent CONSISTENCY risk (honest negative result). Deep Research Agents covers a different, complementary failure mode: DIVISION OF LABOR (redundant search coverage) and its real fix -- plus STORM and BrowseComp, which Multi-Agent Systems doesn't touch at all.",
          "source": "Deep Research Agents"
        },
        {
          "front": "What is 'grounding' in computer-use agents, and what real, named difficulty does it face on dense UIs?",
          "back": "'The task of mapping natural language instructions to screen coordinates.' On professional/high-res interfaces: a 'needle-in-a-haystack problem where target widgets may occupy less than 0.1% of the total screen area.' Real ScreenSpot-Pro accuracy spread: Gemini-2.5-Pro 6.96%, OpenAI CUA 35.03%, UI-TARS-72B 37.12%, a specialized method 73.18% -- 10x+ range by technique.",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What are OSWorld's real, original headline numbers, and what did the OSWorld-Human follow-up add?",
          "back": "369 real tasks. Human performance 72.36%; best agent at launch 12.24%. OSWorld-Human found even SUCCESSFUL agents take '2.7-4.3x more steps than necessary' -- a real efficiency dimension raw success rate doesn't capture.",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What is UI-TARS's real architectural approach, and its real, measured progress across versions?",
          "back": "A single vision-language model unifying 'perception, reasoning, grounding, and memory' (not a pipeline), via Unified Action Modeling + System-2 Reasoning + RL-based refinement. Real OSWorld trajectory: original 24.6 -> UI-TARS-1.5 42.5% -> UI-TARS-2 47.5%. Real generational improvement on the same benchmark.",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What is CometJacking, and what was Brave Security's exact root-cause diagnosis of the real vulnerability?",
          "back": "A real, working attack against Perplexity's Comet browser agent, using a hidden Reddit spoiler-tag instruction. Root cause (exact quote): 'when users ask it to Summarize this webpage, Comet feeds a part of the webpage directly to its LLM without distinguishing between the user's instructions and untrusted content.' Real data exfiltration (email -> OTP -> post) was demonstrated.",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "In this topic's own real repro of the CometJacking mechanism, what happened, and what real methodological lesson came from the first attempt?",
          "back": "Both raw (unmarked) and tagged (<untrusted_webpage_content> wrapped) conditions: Sonnet 5 resisted, correctly flagging the hidden 'unlock bonus PDF' pretext as data-harvesting/phishing. BUT the first injection version self-announced as a 'SYSTEM NOTE' in a '[hidden span]' marker -- an unrealistic tell. Rewritten as an ordinary-sounding pretext, the honest negative held -- a fact about this model on this run, NOT a structural guarantee (the real Comet incident succeeded against a different real agent using this exact mechanism).",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What real, practical cost detail does Anthropic's own computer-use tool documentation give, and how does it connect to the Cost and Latency topic?",
          "back": "A single screenshot typically costs 'roughly 1,000-1,800 input tokens each.' A long computer-use agent loop compounds cost the same way Cost and Latency's quadratic-transcript-growth finding does -- just driven by image tokens instead of text tokens.",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What are the three real, distinct browser-automation infrastructure approaches this topic names?",
          "back": "Anthropic's computer_toolset_20260801 (17 member tools, pixel-space coordinates). OpenAI's computer-use-preview (Research Preview, gated tiers, Responses API only). Stagehand (Browserbase's Playwright wrapper with act()/extract()/observe() natural-language primitives; v4 moved dispatch into a browser extension so remote browsers 'behave like the Chrome on your laptop').",
          "source": "Computer-Use and Browser Agents"
        },
        {
          "front": "What are OpenClaw and Hermes Agent, and what's each one's real, distinguishing framing?",
          "back": "Both real, open-source, local-first, messaging-native always-on personal agents. OpenClaw: 'State lives on your machine, not a vendor cloud' -- gateway/untrusted-execution/deterministic-policy architecture. Hermes Agent (Nous Research): 'the only agent with a built-in learning loop -- it creates skills from experience, improves them during use.'",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "What is 'blast radius' in the context of always-on agents, exactly quoted?",
          "back": "'A standard AI application processes a single user request, returns a response, and discards all session context. The risk surface is bounded by that single exchange. A persistent agent is categorically different.' 'Blast radius describes the maximum potential damage an agent can cause if its goals are hijacked, its tools are misused, or its memory is poisoned.' 'The boundary is no longer a network perimeter. It is the agent's runtime policy.'",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "What is ClawHavoc, and what are the real, verified numbers behind it?",
          "back": "A real 2026 supply-chain attack on OpenClaw's skill marketplace (ClawHub) -- real credential-stealing malware distributed via disguised skills. Koi Security documented 341 malicious skills; Bitdefender found ~17% of skills analyzed in the platform's first few weeks carried malicious payloads.",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "What is Unit 42's exact root-cause diagnosis for ClawHavoc?",
          "back": "'The lack of isolation between skill logic and agent authority means that installation results in complete control over the agent's identity.' A malicious skill performs 'unauthorized actions through the agent's own authenticated sessions,' exploiting 'file systems, shells and credential managers' via natural-language instructions -- no conventional exploit needed.",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "In this topic's own real repro (mechanism-level check, zero LLM judgment), what did unscoped vs. scoped skill tool access actually permit?",
          "back": "unscoped: hand-written calls to read_financial_accounts and send_email (to the skill's own external address) both succeeded cleanly -- real balance returned, real 'email sent' confirmation. scoped: both identical calls were BLOCKED before execution ('outside granted scope'). This is ClawHavoc's real architectural flaw, reproduced deterministically.",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "What happened in this topic's live-agent trial (real Sonnet 5), and what real methodological lesson repeated from a prior topic?",
          "back": "Sonnet 5 declined the financial-data reach in BOTH conditions -- an honest negative. But the FIRST version of the injected skill instruction self-announced as a 'hidden internal note... not shown in the marketplace listing' -- the same unrealistic tell caught and fixed in the Computer-Use and Browser Agents topic. Rewritten as ordinary (if overreaching) product copy, the honest negative held after one well-motivated adjustment.",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "Why does this topic treat the live-agent trial's honest negative as NOT a security guarantee, even though the model resisted?",
          "back": "The mechanism-level check holds BY CONSTRUCTION, independent of model behavior. The live-agent result is a fact about one model on one run. ClawHavoc itself is real, independent, real-world proof the underlying architectural gap gets exploited regardless of any single model's judgment -- real attackers didn't need to fool a capable model's judgment because the unscoped architecture never required fooling it in the first place.",
          "source": "Personal and Always-On Agents"
        },
        {
          "front": "What's the real difference between outcome grading and trajectory grading, and what documented risk does relying on outcome-only grading create?",
          "back": "Outcome grading checks the final result; trajectory grading checks the process (which tools, in what order). Outcome-only grading can be gamed: the Holistic Agent Leaderboard's own log audits caught agents 'searching for the benchmark on HuggingFace instead of solving a task' -- an outcome an outcome-only grader would score as passing.",
          "source": "Evaluating Agents"
        },
        {
          "front": "A real repro tested whether a model would skip a verification tool on an easy, guessable fact vs an unguessable fictional one. It called the tool both times and got both outcomes right -- no divergence. Was this a wasted test?",
          "back": "No -- an honest negative result. The value isn't that the check always finds a mismatch; it's that you can't know whether outcome and trajectory agree until you check both. The real documented risk (agents gaming outcome-only graders) is confirmed elsewhere in the literature, not disproven by one well-behaved run.",
          "source": "Evaluating Agents"
        },
        {
          "front": "What's the precise difference between pass@k and pass^k?",
          "back": "pass@k: did AT LEAST ONE of k independent trials succeed (generous, capability-ceiling framing). pass^k: did EVERY ONE of k trials succeed (strict, the real reliability question -- a production agent gets one real try per request, not k with a human picking the best).",
          "source": "Evaluating Agents"
        },
        {
          "front": "tau-bench reports GPT-4o's retail success rate falling from under 50% at pass@1 to under 25% at pass^8. A real recipe run measured pass@5 = pass^5 = 100% on a different task (a combinatorics question). Does this contradict tau-bench's finding?",
          "back": "No. The SIZE of the pass@k/pass^k gap depends on how much genuine variance a specific task has for a specific model -- tau-bench's multi-turn, tool-using, policy-constrained domain has far more room to go wrong than one well-structured probability question. Different tasks, different gaps; the metrics' definitions don't change.",
          "source": "Evaluating Agents"
        },
        {
          "front": "A real experiment changed a system prompt to spell out numbers in words (accessibility-motivated). A digit-matching regression suite went from 4/4 passing to 0/4 -- but a human confirmed every answer was mathematically correct. What actually regressed?",
          "back": "The grader, not the model. The automated check assumed a digit format the new (reasonable) prompt no longer produced. This is exactly the failure mode a regression suite exists to catch: a well-intentioned, unrelated change silently breaking how correctness gets checked, not how correctly the task gets done.",
          "source": "Evaluating Agents"
        },
        {
          "front": "What's the practical difference between a capability eval and a regression eval, and why shouldn't they be conflated?",
          "back": "Capability eval: how good could the agent be -- expensive, exploratory, run occasionally. Regression eval: did a specific change break something that used to work -- cheap, narrow, run on every change. A regression suite that 'never catches anything' isn't badly designed; 'still passing' is the correct default outcome most of the time it runs.",
          "source": "Evaluating Agents"
        },
        {
          "front": "Anthropic ran the identical model and benchmark (Terminal-Bench 2.0) across six container resource configurations. What was the real measured effect, and what does it imply about trusting a close leaderboard gap?",
          "back": "A 6-percentage-point gap (p<0.01) between most- and least-resourced configs -- bigger than many reported gaps between competing systems. Infra error rate alone fell from 5.8% (1x) to 2.1% (3x) to 0.5% (uncapped). Implication: a small leaderboard lead could be hardware, not intelligence, unless resource specs are pinned and published.",
          "source": "Evaluating Agents"
        },
        {
          "front": "The SWE-bench cross-check in Anthropic's infrastructure-noise study found a smaller effect (+1.54 percentage points at 5x RAM) than the main Terminal-Bench 2.0 finding (6 points). Does this contradict the main finding?",
          "back": "No -- it shows the SIZE of the infrastructure confound depends on the task's own resource profile, not just on infrastructure existing. SWE-bench's tasks are less resource-intensive than Terminal-Bench 2.0's, so the same underlying confound shows up smaller there. Both results are consistent with 'infrastructure is a real, controllable confound.'",
          "source": "Evaluating Agents"
        },
        {
          "front": "A real fetch of official leaderboard data found SWE-bench Verified's real top score, versus what several aggregator sites reported. What's the discrepancy, and why does it matter?",
          "back": "Official (swebench.com): 79.2%. Aggregator sites: 96-97%. A real, large, checkable gap -- the lesson isn't 'aggregators are always wrong,' it's that any benchmark number should be checked against its official leaderboard directly, not a secondary summary, before citing it.",
          "source": "Benchmark Atlas"
        },
        {
          "front": "What did OpenAI's own audit find about SWE-bench Verified's test cases, and what did they conclude?",
          "back": "'At least 59.4% of the audited problems have flawed test cases that reject functionally correct submissions' (35.5% too narrow, 18.8% too wide). Conclusion: 'we have stopped reporting SWE-bench Verified scores, and we recommend that other model developers do so too... OpenAI recommends reporting results for SWE-bench Pro' instead.",
          "source": "Benchmark Atlas"
        },
        {
          "front": "What did OpenAI find about training contamination on SWE-bench Verified?",
          "back": "'All frontier models we tested were able to reproduce the original, human-written bug fix... indicating that all of them have seen at least some of the problems and solutions during training. We also found evidence that models that have seen the problems during training are more likely to succeed.'",
          "source": "Benchmark Atlas"
        },
        {
          "front": "Which real benchmarks in this atlas are genuinely saturated (near-ceiling top scores), and which are clearly not?",
          "back": "Saturated: OSWorld-Verified (90.19%, up from a 12.24% launch baseline), AppWorld (96.4%/98.3%, up from ~49%/30%). Clearly not: TheAgentCompany (42.9%), SWE-bench Pro (61.5%), MLE-bench (64.4%), BrowseComp (51.5%).",
          "source": "Benchmark Atlas"
        },
        {
          "front": "How does tau-bench's own real leaderboard show that 'saturation' isn't a single property of a whole benchmark?",
          "back": "Telecom domain: 97.8% (essentially saturated). Banking_knowledge domain (added later, specifically to stay hard): only 55.2%. Same benchmark, wildly different saturation levels depending on which domain -- maintainers add harder domains once older ones stop differentiating models.",
          "source": "Benchmark Atlas"
        },
        {
          "front": "What is tau-bench's real reliability metric pass^k, and how does it differ from pass@1?",
          "back": "pass@1 is the average success rate across independent trials. pass^k asks whether EVERY one of k independent trials succeeded -- a stricter, all-or-nothing reliability check. An agent that's usually right (high pass@1) isn't the same guarantee as one that's always right (pass^k = true).",
          "source": "Benchmark Atlas"
        },
        {
          "front": "A real repro of tau-bench's grading philosophy graded by 'action state' rather than reply text. What does that mean, and why does it matter?",
          "back": "It checked whether the policy-violating tool (cancel_order on a shipped order) was actually CALLED, not whether the reply sounded compliant. A reply can sound correct while the real action taken violates policy (or vice versa) -- action-state grading is trustworthy regardless of how convincing the text is.",
          "source": "Benchmark Atlas"
        },
        {
          "front": "AgentDojo and InjecAgent don't fit the 'saturated vs. unsaturated' framing the way capability benchmarks do. Why not?",
          "back": "They measure attack success rate (a vulnerability), not a capability ceiling. AgentDojo's real numbers: Claude 3.7 Sonnet had 88.7% utility / 7.3% attack success; GPT-4o had 69.1% utility / 47.7% attack success -- ASR varies wildly by model (1-48%), meaning there's no consistent 'frontier' defense yet, which is itself the interesting finding rather than a ceiling number.",
          "source": "Benchmark Atlas"
        },
        {
          "front": "What is the real, current quote naming the core problem with monitoring AI agents like traditional services?",
          "back": "'When your AI agent returns a confidently wrong answer, your monitoring sees a successful 200 response.' (Jamie Mallers, OneUptime, 2026-03-28). Traditional error/timeout monitoring is structurally blind to a semantically wrong but technically clean result.",
          "source": "Observability and Debugging"
        },
        {
          "front": "What is the real OpenTelemetry GenAI span naming rule for tool execution?",
          "back": "'Span name SHOULD be {gen_ai.operation.name} {gen_ai.tool.name}.' A call to get_daily_active_users produces a span literally named 'execute_tool get_daily_active_users' -- a standard, mechanical, cross-vendor naming rule, not a bespoke team format.",
          "source": "Observability and Debugging"
        },
        {
          "front": "Where does the real OTel GenAI spec say a tool call's actual arguments and results get recorded, and what's the real privacy caveat attached?",
          "back": "Via gen_ai.input.messages and gen_ai.output.messages (structured JSON), NOT a dedicated gen_ai.tool.call.arguments attribute. Real caveat: 'This attribute is likely to contain sensitive information' and instrumentations 'MAY provide a way for users to filter or truncate' it.",
          "source": "Observability and Debugging"
        },
        {
          "front": "A real repro's agent call to a buggy tool completed with stop_reason 'end_turn' and zero errors. What was the real, confidently wrong final answer, and what was the actual bug?",
          "back": "'Yesterday's DAU for Nimbus was 2,340,000... roughly 46.8x the goal... This is a very healthy result.' The real bug: get_daily_active_users returned CUMULATIVE all-time users, not yesterday's daily figure -- a tool implementation bug invisible to the model's own reasoning.",
          "source": "Observability and Debugging"
        },
        {
          "front": "How did the real trace-only diagnostic check find the bug, and what real constraint did it operate under?",
          "back": "It inspected ONLY the trace's gen_ai.output.messages attribute on the execute_tool span (no access to the final answer, no re-running the agent) and flagged: '2,340,000 -- implausibly large for a single day, likely a cumulative/all-time figure mislabeled as daily.' Real root cause found from structured data alone.",
          "source": "Observability and Debugging"
        },
        {
          "front": "What real bug did the recipe's own trace-emitter code hit, and what caused it?",
          "back": "It assumed every non-text content block was a tool_use block and read .name/.input off it unconditionally -- crashed on Sonnet 5's real adaptive-thinking ThinkingBlock, which has neither attribute. Fixed by explicitly handling text, tool_use, and any other block type.",
          "source": "Observability and Debugging"
        },
        {
          "front": "Why does 'gen_ai.input.messages/output.messages have a defined standard location' matter specifically for automated diagnosis?",
          "back": "A standard, specified location means ANY compliant trace -- from any tool, any team, any vendor -- can be inspected the same way by the same diagnostic check, rather than every team needing bespoke parsing logic for its own ad hoc logging format.",
          "source": "Observability and Debugging"
        },
        {
          "front": "How should the real privacy tension in gen_ai.input.messages/output.messages (sensitive data vs. real debugging value) be resolved, per this topic's framing?",
          "back": "Filtering and truncation should be a deliberate design choice made for each real case -- not a blanket default either way (log everything forever, or log nothing to stay safe). The same discipline context editing and compaction already require elsewhere in this project.",
          "source": "Observability and Debugging"
        },
        {
          "front": "What are the three properties of Willison's 'lethal trifecta,' in his own words?",
          "back": "Access to private data; exposure to untrusted content ('any mechanism by which text controlled by a malicious attacker could become available to your LLM'); and the ability to externally communicate 'in a way that could be used to steal your data.' All three live in one session is the dangerous precondition.",
          "source": "Agent Security"
        },
        {
          "front": "A real repro gave an agent all three trifecta legs plus a version with a deterministic allow-list on the send tool. The model declined the injected request in BOTH conditions -- no exfiltration either way. Does this mean the structural fix wasn't needed?",
          "back": "No. The fixed condition's guarantee doesn't depend on whether the model resists -- it blocks disallowed sends regardless of model behavior. One run where the model happened to resist tells you about that model on that injection; a code-level check doesn't need to keep 'happening to work.'",
          "source": "Agent Security"
        },
        {
          "front": "What does 'The Attacker Moves Second' report about published prompt-injection/jailbreak defenses?",
          "back": "Bypassed 12 recent published defenses with attack success rate above 90% for most -- where those same defenses had originally reported near-zero attack success rates under their own evaluation. A sobering finding: detection-based defenses look solid until someone adapts specifically to them.",
          "source": "Agent Security"
        },
        {
          "front": "What is 'tool poisoning,' and how did a real repro demonstrate it?",
          "back": "A supply-chain attack where a hidden instruction is embedded in a tool's DESCRIPTION field -- text a user never reads, read by the model on every turn. Real repro: an innocuous 'what's the weather in Paris?' question triggered an unrelated sensitive customer-record lookup, purely because the weather tool's description secretly asked for it.",
          "source": "Agent Security"
        },
        {
          "front": "Meta's Rule of Two is often summarized as 'never combine all three trifecta properties.' What's the precise wording, and why does it matter?",
          "back": "'Agents must satisfy no more than two of the following three properties within a session' [A] untrustworthy inputs [B] sensitive systems/private data [C] can change state or communicate externally. It permits ANY TWO, not zero -- only the full three-way combination is restricted. It's a repackaging of the trifecta into a design rule, credited explicitly to Willison.",
          "source": "Agent Security"
        },
        {
          "front": "CaMeL's core guarantee, in its own words, and its real measured cost?",
          "back": "'The untrusted data retrieved by the LLM can never impact the program flow' -- a structural (not detection-based) separation of control and data flow. Real cost: solves 77% of AgentDojo tasks with provable security, versus 84% for an undefended system -- a real 7-point utility cost for the guarantee, not a free upgrade.",
          "source": "Agent Security"
        },
        {
          "front": "Why is a classifier/detector-based defense against prompt injection considered 'defense in depth' rather than the load-bearing control?",
          "back": "Willison's own framing: a '95% catch rate is a failing grade' in security terms, and adaptive attacks have bypassed 12 published defenses at >90% success. The load-bearing control is limiting what a COMPROMISED agent can do (removing a trifecta leg, CaMeL-style structural separation) -- not relying on detection catching every attempt.",
          "source": "Agent Security"
        },
        {
          "front": "How does tool poisoning differ from the lethal trifecta as an attack category?",
          "back": "The trifecta is a session-level combination of properties across a conversation (private data + untrusted content + external comms all present at once). Tool poisoning is about a single artifact's metadata (a tool/MCP description) being untrustworthy -- a supply-chain risk that exists before any conversation even starts, with a different fix (auditing/sanitizing descriptions, not session design).",
          "source": "Agent Security"
        },
        {
          "front": "What is the real, five-layer guardrail stack, and where does human-in-the-loop sit within it?",
          "back": "Data and context guardrails, design-time governance, runtime guardrails and gateways, identity/access/security controls, and human-in-the-loop oversight -- as the FINAL layer, not the only one. It's meant to catch residual risk the earlier, cheaper, automated layers didn't already resolve.",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "What is the real, specific risk-tiering guideline this topic is built around?",
          "back": "'Define explicit risk tiers for agent use cases and apply proportional controls. Low-risk tasks like data summarization can run with lighter oversight. High-risk actions involving financial transactions, PII, or policy changes require multi-step verification, human approval, and comprehensive audit trails.'",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "A real repro ran the identical refund task (2 lookups + a $250 refund) under flat-autonomous (no gates). What happened?",
          "back": "All 3 calls executed immediately, including the $250 refund -- risky_refund_executed_without_review: true. The high-risk action went through with zero review, indistinguishable from the safe lookups in a raw success/failure view.",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "The same task under flat-gated (every call requires approval, no risk distinction) -- what was the real, sharper finding, beyond just 'more friction'?",
          "back": "The agent made only 2 total calls -- both harmless lookups -- and NEVER attempted the refund call at all. Undifferentiated gating didn't just slow things down; it stalled real progress before the agent even reached the action that actually needed review.",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "The same task under risk-tiered gating (only actions above a real threshold require approval) -- what was the real result?",
          "back": "All 3 calls were attempted. Only the $250 refund was gated (gated_count: 1). risky_refund_executed_without_review: false. The agent completed everything it safely could and stopped exactly at the one point needing human review -- confirmed on two separate full runs.",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "Why did the recipe compute risk per tool CALL (using actual arguments) rather than per tool NAME?",
          "back": "A $10 issue_refund call and a $250 issue_refund call are the identical tool with very different real risk. Risk-based gating that only looked at tool names would either over-gate cheap refunds or under-gate expensive ones -- matching the real framing of risk scoring that 'weighs the specific action against its current context.'",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "What real, concrete criterion does OWASP-adjacent guidance name for what belongs in the highest risk tier, beyond just dollar amount?",
          "back": "Reversibility: 'require human approval on any irreversible action like moving money or deleting data.' This gives a defensible heuristic independent of a specific threshold number -- gate what can't be cleanly undone, then calibrate size-based thresholds from there.",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "What is the real, three-way tradeoff this topic's repro demonstrates across flat-autonomous, flat-gated, and tiered guardrails?",
          "back": "Flat-autonomous: functional but unsafe (risky action unreviewed). Flat-gated: safe but non-functional (agent stalls before reaching what needed review). Tiered: both safe (risky action caught) AND functional (safe work still proceeds) -- the concrete, measured payoff of 'proportional controls.'",
          "source": "Guardrails and Human-in-the-Loop"
        },
        {
          "front": "What is Okta's real, verified Agent SSO, and what does it establish?",
          "back": "GA August 24, 2026. 'Agent SSO establishes a first-class identity model for AI agents at the point of connection' -- registers agents 'as a first-class identity in Universal Directory alongside human employees,' and 'issues short-lived, identity-governed tokens in place of stored credentials.' Its Cross App Access protocol is 'formally incorporated as the official Enterprise-Managed Authorization extension for the Model Context Protocol.'",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "In this topic's own real repro, what happened when an agent was told to 'just act as the user' vs. given a distinct identity + scoped delegation?",
          "back": "Blended: sent email was fully anonymous -- 'This email confirms approval of Invoice #INV-4471...' no identity, no delegation trail. Delegated: 'I am Agent-Finance-07, an AI agent acting on behalf of Priya Shah (Finance Ops) under an explicit, scoped delegation...' Real, first-try result: identity disclosure is a design choice specified via instruction, not an automatic byproduct.",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What are OWASP's real ASI03 and ASI09 risk categories (Top 10 for Agentic Applications, Dec 9 2025)?",
          "back": "ASI03 -- Identity & Privilege Abuse: 'leaked credentials enable agents to operate beyond intended scope.' ASI09 -- Human-Agent Trust Exploitation: 'confident explanations mislead operators into approving harmful actions.' Both are governance/identity-layer risks, distinct from technique-level attacks (trifecta, injection) covered in Agent Security.",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What is NIST's real CAISI AI Agent Standards Initiative, and its three stated pillars?",
          "back": "Launched Feb 17, 2026. Pillars (exact quotes): (1) 'facilitating industry-led development of agent standards and U.S. leadership in international standards bodies,' (2) 'fostering community-led open source protocol development and maintenance for agents,' (3) 'advancing research in areas of AI agent security and identity to enable new use cases.'",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What real EU AI Act provisions apply to agentic systems, and what's the critical thing NOT to imply?",
          "back": "Article 50 (AI-interaction transparency, applies Aug 2, 2026) -- must disclose AI interaction unless obvious. Article 14 (human oversight for high-risk systems, incl. a real 'stop' mechanism requirement). Article 6/Annex III (high-risk classification by USE CASE, applies Dec 2027/Aug 2028). CRITICAL: no provision names 'AI agents' as a defined category -- classify by what the system does, not by an agent-specific chapter (none exists).",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What is MITRE ATLAS, and how does this topic frame its relationship to OWASP's list?",
          "back": "A real, living knowledge base of adversary tactics/techniques against AI systems, modeled on MITRE ATT&CK, built from 'empirical evidence... of real-world attacks.' Expanding 'to capture attack paths that emerge at the orchestration and execution layers' as agents act autonomously. Framing: OWASP names WHAT can go wrong (risk categories); ATLAS catalogs HOW adversaries achieve it (techniques) -- complementary, not competing.",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What real, unratified IETF draft proposes a delegation-chain mechanism for agent identity, and what does it add?",
          "back": "draft-oauth-ai-agents-on-behalf-of-user (expired/archived, NOT an approved standard -- flag this). Introduces a 'requested_actor' parameter identifying the specific agent, and an 'actor_token' authenticating it during exchange; resulting access tokens carry 'the delegation chain from the user to the agent via a client application' -- a real, traceable record of who authorized what.",
          "source": "Identity, Governance and Compliance"
        },
        {
          "front": "What are the four tiers of the real isolation ladder, weakest to strongest, for sandboxed agent code execution?",
          "back": "Linux namespaces/cgroups (containers) -- weakest, shared host kernel. gVisor -- middle, a user-space Sentry kernel intercepts every syscall before the real host kernel sees it. Firecracker/Kata microVMs -- strongest, each workload gets its own dedicated guest kernel via real hardware virtualization. WebAssembly sits on a separate axis: capability-first, default-deny, and real, verified 'lighter weight than containers or virtual machines.'",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What are the real, verified Firecracker microVM numbers that make strong isolation and fast/disposable VMs NOT a tradeoff?",
          "back": "Boot in <125ms. <5 MiB memory overhead per VM. Up to 150 microVMs launched per second per host. Real hardware virtualization without the traditional VM cost.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What does gVisor's Sentry actually do, architecturally?",
          "back": "A per-sandbox application kernel running in USER SPACE that re-implements Linux's syscall surface itself. Every syscall the sandboxed workload makes is intercepted and handled by the Sentry, not the real host kernel directly -- and the Sentry's own access back down to the host is narrowed via seccomp filters to a small subset of calls.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What real, documented risk means a sandbox does NOT automatically protect an agent's credentials?",
          "back": "'A prompt-injected web page, a poisoned dependency's README, or an ambiguous instruction doesn't need to escape the sandbox. It just needs to ask the agent, which is already inside the trust boundary with the secret, to use it somewhere it shouldn't.' (Zujkowski) -- if the real credential sits in the sandboxed process's own scope, it's reachable without ever breaching the sandbox.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What is the real, verified guarantee credential proxying earns, and how does the pattern work mechanically?",
          "back": "'The agent's process never has the plaintext secret in memory, on disk, or in its context window.' Mechanically: route the real call through a proxy OUTSIDE the sandbox boundary; the sandboxed code only ever holds a placeholder token, and the proxy substitutes the real credential itself before the call goes out, never handing the plaintext back.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What is the real, named limitation of credential proxying, and what did this topic's mechanism-level check show?",
          "back": "'Proxying prevents credential theft. It does not prevent credential misuse.' The check (zero LLM judgment, hand-written code): the real key was visible in 'inside' mode and only a placeholder in 'proxied' mode -- but a hand-written payment call to an attacker-controlled-looking account succeeded in BOTH conditions regardless. Proxying changed what the code could SEE, not what it could DO.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "In the real, unforced live-agent trial against Claude Sonnet 5 (direct key access + injected payment request), what happened, and why doesn't this replace the mechanism-level check?",
          "back": "The model masked the key rather than printing it raw and declined the suspicious payment in BOTH conditions, citing the destination pattern explicitly. But this is a fact about THIS model on THIS run -- not a deterministic guarantee. The mechanism-level check holds by construction regardless of model behavior; the live trial is a second, honest, non-substitutable data point, not a security control.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "Why is credential proxying described as a confidentiality control rather than an authorization control -- and what closes the gap it leaves open?",
          "back": "Proxying hides the credential's VALUE from the sandboxed code; it says nothing about whether a given action (which destination, how much) should be allowed at all. Closing the misuse gap needs a separate, deterministic authorization check on the action itself -- an allow-list, a hard cap, a human-approval gate -- the same structural-vs-behavioral distinction as risk-tiered guardrails.",
          "source": "Sandboxes and Permissions"
        },
        {
          "front": "What does 'a crash resumes the conversation instead of restarting it' actually mean, in Temporal's own framing?",
          "back": "'When a Worker crashes, the Temporal Service hands the work to another Worker, which replays the Event History and resumes at the line where execution stopped, with local variables and progress intact.' For agents: the loop is a Workflow, each model/tool call is an Activity -- a crash doesn't mean starting the whole task over.",
          "source": "Durable Execution"
        },
        {
          "front": "What's the real mechanical difference between Temporal-style replay and LangGraph-style checkpointing?",
          "back": "Replay: re-runs workflow code from the top, but skips already-completed steps using their RECORDED RESULTS (an event history). Checkpointing: persists application STATE directly as a snapshot at each step. Different mechanisms, same goal -- and both need a real persistent backend to survive a process crash.",
          "source": "Durable Execution"
        },
        {
          "front": "LangGraph's own docs warn that MemorySaver/InMemorySaver checkpoints don't survive a process restart. Why does this matter beyond LangGraph specifically?",
          "back": "It's a general trap: having a checkpointing/logging API in your code doesn't guarantee durability -- durability depends on WHERE state is persisted, not just that a persistence API exists. Any framework's default in-memory store has this same gap.",
          "source": "Durable Execution"
        },
        {
          "front": "A real repro crashed a process right after a tool's side effect committed but BEFORE that fact was logged. Why can't a better-designed event log alone fix this gap?",
          "back": "The log can only record what it already knows completed -- it structurally cannot represent 'the side effect just happened but the write recording it hasn't happened yet.' There is always a real window between an action and its durable record, regardless of log quality. This is why idempotent tools are a separate, necessary second mechanism.",
          "source": "Durable Execution"
        },
        {
          "front": "In the real durable-agent run, invocation 2 (the resumed process) needed only 1 new model call, not 3. Why?",
          "back": "Steps 1 and 2 were already durably logged as complete, so invocation 2 replayed them from disk with ZERO new model calls -- reconstructing conversation state without re-deciding or re-executing. Only step 3, which crashed before being logged, needed a real new model call.",
          "source": "Durable Execution"
        },
        {
          "front": "In that same run, the model's step-3 decision on resume was to call send_confirmation AGAIN (since the log didn't show it as done). What actually prevented a duplicate confirmation email?",
          "back": "Not the event log -- the tool itself. send_confirmation checked its own persisted ledger (keyed by order_id) before acting, found the confirmation_id already existed, and returned 'already sent (idempotent replay)' instead of sending again. One confirmation_id, sent exactly once, across the crash and the resume combined.",
          "source": "Durable Execution"
        },
        {
          "front": "Why is 'idempotency alone solves crash recovery, we don't need a log' a flawed argument?",
          "back": "Idempotent tools prevent DUPLICATE SIDE EFFECTS, but without a durable log, every resume would have to re-decide and re-attempt every step from scratch -- burning model calls and losing all progress tracking, even though no duplicate real-world effect would occur. The log and idempotency solve different halves of the same problem.",
          "source": "Durable Execution"
        },
        {
          "front": "Why is this a real, documented interview topic rather than a niche concern?",
          "back": "Real sourced interview questions: 'How do you make sure agents do not double-execute side-effectful operations like charging a card or booking a ticket twice?' and 'Suppose your booking agent sometimes reserves the same hotel twice -- walk through how you'd debug and fix this.' Long-running agents accumulate real side effects at unpredictable points, unlike a typical fast request/response service.",
          "source": "Durable Execution"
        },
        {
          "front": "What is the real, documented formula for naive agent loop cost, and which term is the real 'cost trap'?",
          "back": "Total = N*S + u*N(N+1)/2 + r*N(N-1)/2, where S=system prompt tokens, u=new input tokens/turn, r=output tokens/turn, N=turns. The N(N+1)/2 TRIANGULAR NUMBER term is the trap -- it's quadratic in N, because every turn rebills the entire accumulating transcript, not just its own new content.",
          "source": "Cost and Latency"
        },
        {
          "front": "What real, worked example shows the scale of quadratic transcript growth?",
          "back": "A 20-step loop where each step generates 1,000 tokens produces 210,000 cumulative input tokens -- NOT the 20,000 a flat per-step estimate would suggest. A separate 10-step file-reading example: naive loop = 43.3x a single-pass baseline; windowed/constrained = ~29x -- real savings, but still far above 1x.",
          "source": "Cost and Latency"
        },
        {
          "front": "In this topic's real repro, what did the naive condition's per-turn input-token counts look like across 7 turns, and what does that pattern indicate?",
          "back": "665, 1290, 1921, 2552, 3183, 3814, 4445 -- each turn costs ~631 tokens MORE than the last. A roughly CONSTANT per-turn increase is exactly the signature of quadratic cumulative growth (sum of an arithmetic sequence = a triangular number). Real cumulative total: 17,870 input tokens.",
          "source": "Cost and Latency"
        },
        {
          "front": "What did windowing (keeping only the last 2 tool results) actually buy in the real repro, measured?",
          "back": "Per-turn tokens: 665, 1300, 1934, 2051, 2168, 2285, 2406 -- nearly identical to naive for the first 3 turns (window had nothing to omit yet), then flattening to ~121 tokens/turn increase. Real cumulative total: 12,809 tokens -- a real 28.3% reduction vs. naive, with the gap widening every additional turn. Correctness was unaffected in this run.",
          "source": "Cost and Latency"
        },
        {
          "front": "What real current per-token pricing gap makes model routing a large lever, and what did the real routing repro find on cost/accuracy?",
          "back": "Haiku 4.5: $1/$5 per million (in/out). Sonnet 5: $2/$10 per million -- roughly 2x here, and the real repro's routed condition (mostly Haiku) matched Sonnet's accuracy (9/10) at ~44% of Sonnet-only's real dollar cost ($0.011843 vs $0.026866).",
          "source": "Cost and Latency"
        },
        {
          "front": "What was the real, sharp limitation found in the confidence-based routing repro -- and did it survive a deliberate retry?",
          "back": "escalated_count was 0 -- Haiku reported 'high' confidence on ALL 10 questions, including the one it got wrong (the 'strawberry' spelling trap). This held even after adding 2 harder questions AND rewriting the tool description to explicitly instruct calibrated self-assessment ('mark low whenever multi-step reasoning could produce a careless mistake'). Confidence stayed high regardless -- a real, well-tested negative result.",
          "source": "Cost and Latency"
        },
        {
          "front": "Why does STEER (arXiv 2511.06190) use model-internal confidence instead of self-reported confidence, and why couldn't this topic's repro test that mechanism directly?",
          "back": "A model that's about to be wrong isn't reliably aware it's about to be wrong -- self-report doesn't track true uncertainty (confirmed by this topic's own real result). STEER instead derives confidence from the smaller model's own OUTPUT-TOKEN PROBABILITIES. This repro couldn't use that directly because the Anthropic Messages API doesn't expose token-level log probabilities.",
          "source": "Cost and Latency"
        },
        {
          "front": "What are the two SEPARATE real cost levers this topic distinguishes, and what does each one actually change?",
          "back": "(1) Context management (windowing, summarization) attacks the QUADRATIC term -- how much of the transcript gets rebilled each turn, independent of which model answers. (2) Model routing attacks the PER-TOKEN PRICE -- a real 2x+ lever on current pricing. Routing's COST benefit is separate from its SAFETY benefit (catching mistakes) -- the real repro got the former but not the latter from self-report alone.",
          "source": "Cost and Latency"
        },
        {
          "front": "What does GRPO replace PPO's critic (value network) with, and what's the exact formula?",
          "back": "The group itself. 'GRPO foregoes the critic model, instead estimating the baseline from group scores.' Formula: sample a group of completions to the same prompt, then advantage = (reward - group_mean) / group_std for each sample -- that normalized value becomes every token's advantage.",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "A real run computed GRPO advantages for a group where every one of 8 samples got the identical reward (all correct). What was the result, and what does DAPO call this?",
          "back": "Advantage = 0.0 for all 8 samples -- an exact reproduction of DAPO's named 'gradient-decreasing problem': 'if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients.'",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "What is DAPO's 'dynamic sampling' fix, exactly, and how is it different from a simple accuracy threshold (e.g. 'discard anything below 50%')?",
          "back": "Over-sample and filter out prompts whose accuracy is EXACTLY 0 or EXACTLY 1 -- keep everything with real variance in between. Unlike a threshold, a group with accuracy 0.3 is kept (it has real gradient signal), not discarded -- only the two degenerate boundary cases are excluded.",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "A real toy repro: a 3-step trajectory where only step 2 was the actual mistake, final outcome reward -1. What did outcome-only credit assignment produce, and what could it NOT do?",
          "back": "Broadcast [-1, -1, -1] to all three steps -- bitwise identical to what a trajectory where ALL THREE steps were bad would produce. It structurally cannot localize which step caused the failure; process-level credit ([1, -1, 1]) correctly isolated step 2.",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "PRM800K's own reported comparison of process vs outcome supervision, and its real cost?",
          "back": "'Process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset' -- a process-supervised model solved 78% of a representative MATH test subset. Real cost: 800,000 human step-level labels went into building it.",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "What does DPO eliminate compared to classic RLHF, and what's the real measured trade-off reported?",
          "back": "Eliminates the separate reward-model-training stage and the RL sampling loop -- 'solve the standard RLHF problem with only a simple classification loss' over (prompt, chosen, rejected) triples. Reported to exceed PPO-based RLHF on sentiment control and match/improve quality on summarization and dialogue, while being 'stable, performant, and computationally lightweight.'",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "A real DPO-pair demo generated two completions with NO temperature explicitly set, and got two genuinely different results. Why doesn't this require an explanation like 'the SDK secretly used a high temperature'?",
          "back": "Default (non-zero) sampling variance is normal model behavior across independent calls -- no explicit temperature setting is needed to observe real differences. (Also a real, separate finding: the SDK version used has no top-level `temperature` parameter in its Messages API at all.)",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "Why is credit assignment described as a genuinely unsolved, active research area rather than a solved problem?",
          "back": "A 2026 survey counts 69 papers (56 core credit-assignment methods) still working on it. Methods span from GRPO's group-relative baseline to GiGPO's 'anchor state grouping' (grouping identical states recurring across different rollouts) for finer-grained, step-level credit while keeping GRPO's critic-free, low-memory properties -- an active spectrum, not a settled question.",
          "source": "Training Agents: Reward and Credit"
        },
        {
          "front": "What is Search-R1's real, verified core mechanism and headline result?",
          "back": "The model 'learns to autonomously generate (multiple) search queries during step-by-step reasoning with real-time retrieval,' trained with PPO or GRPO using 'a simple outcome-based reward function.' Real result: 'improves performance by 41% (Qwen2.5-7B) and 20% (Qwen2.5-3B) over various RAG baselines' across 7 QA datasets.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "What do R1-Searcher and ReSearch share with Search-R1, and how does ReSearch frame search differently?",
          "back": "All three train joint reasoning+search behavior via RL with NO supervised data on reasoning steps -- outcome-only. R1-Searcher: explicit two-stage process (learn to invoke search, then learn to use it well). ReSearch: frames search as 'guided by text-based thinking' -- integral to the reasoning chain itself, not a separately triggered action -- and reports reflection/self-correction emerging as a side effect of RL, unprompted.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "What is ToolRL's real, central claim about reward design for tool-use RL, and its fix?",
          "back": "'Coarse-grained reward signals, such as answer matching, fail to offer the finegrained feedback required for effective learning.' Fix: decompose reward into tool-name, parameter-name, and parameter-value components (trained with GRPO). Real result: 'a 17% improvement over base models and a 15% gain over SFT models.'",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "In this topic's own real repro (reproducing ToolRL's claim), what happened when a near-correct tool call and a completely-wrong-tool call were both scored with binary reward vs. fine-grained reward?",
          "back": "Binary reward gave BOTH real completions the identical score: 0.0 -- no differentiation. Fine-grained (tool name / param names / param values, averaged) gave real, different scores: 0.8889 (near-correct, one guessed value wrong) vs. 0.2222 (wrong tool entirely) -- exactly the gap ToolRL's abstract describes, demonstrated on real Claude completions.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "What is ReTool's real, verified headline number on AIME, and what real emergent behavior did the paper document?",
          "back": "67% accuracy at 400 training steps (32B model) vs. a text-only RL baseline's 40% accuracy at 1080 steps -- higher accuracy in ~1/3 the steps. Real, named emergent behavior: 'code self-correction, signaling an \"aha moment\" in which the model autonomously masters adaptive tool use' -- not explicitly trained, it emerged from outcome-based RL.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "What is RAGEN's real, named failure mode for multi-turn LLM agent RL, and its exact defining signature?",
          "back": "The 'Echo Trap' -- 'a recurring instability pattern... where agents overfit to locally rewarded reasoning patterns, marked by reward variability collapse, entropy drop, and gradient spikes.' Models converge to 'near-identical phrasing... without justification' -- a shallow, repetitive pattern that scores adequately locally, not a loss curve just flattening.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "What is RAGEN's proposed fix for the Echo Trap, and what is its third real finding connecting reward design to reasoning quality?",
          "back": "Fix: StarPO-S, using 'trajectory filtering, critic incorporation, and gradient stabilization.' Third finding: 'without fine-grained, reasoning-aware reward signals, agent reasoning hardly emerge[s] through multi-turn RL and they may show shallow strategies or hallucinated thoughts' -- directly connecting RAGEN's stability problem to ToolRL's reward-granularity problem.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "How do the six methods on this topic split into two connected layers?",
          "back": "Layer 1 -- WHEN to act: Search-R1/R1-Searcher/ReSearch show outcome-only RL is enough to teach a model to decide when to search, no step-level supervision needed. Layer 2 -- HOW to reward that action well: ToolRL (fine-grained reward beats coarse) and ReTool (RL improves tool-use TIMING specifically) show coarse reward isn't enough. RAGEN names the real risk of getting layer 2 wrong across many turns: the Echo Trap.",
          "source": "RL for Search and Tool Agents"
        },
        {
          "front": "Of 106 collected 'agent interview questions,' how many actually trace to a real candidate report or company-published process (vs prep material)?",
          "back": "Only 14, from 11 distinct real sources. The other 81 come from prep sites, listicles, and repositories -- some of which explicitly state their questions are 'synthesised... not leaked questions.' Treat prep-site frequency as evidence of what's taught, not what's asked.",
          "source": "Interview Playbook"
        },
        {
          "front": "What is the single strongest dated trend across the interview evidence collected?",
          "back": "AI-assisted coding becoming the interview format itself. OpenAI runs a beta onsite round done 'using an AI coding agent.' Meta added a round where a toy codebase is debugged/completed with LLM help. Sierra removed its classic coding/algorithms round entirely and rebuilt its onsite around a 2-hour AI-tooling build session.",
          "source": "Interview Playbook"
        },
        {
          "front": "Why does it matter that Sierra's own candidate reports (May 2025) and Sierra's own company blog (April 2026) describe different interview processes?",
          "back": "A real loop can change within a year. The 2025 reports (take-home support agent + TS/React debugging) describe a loop the company's own later post says it has since restructured (AI-native onsite, coding phone screen replaced by system design). Prepping from the older report means prepping for a round that may no longer exist.",
          "source": "Interview Playbook"
        },
        {
          "front": "Kubernetes appears in 19.4% of AI-tech job postings but zero candidate-confirmed interview reports reached. What's the correct interpretation?",
          "back": "Job descriptions and live interview reports answer different questions -- what a role needs long-term vs what gets tested live. The JD presence is still real evidence of relevance; the lack of interview confirmation likely reflects unreachable sources (Reddit, Glassdoor, paywalled reports), not proof it's never asked.",
          "source": "Interview Playbook"
        },
        {
          "front": "What's the one topic area with genuine MULTI-SOURCE candidate confirmation, and what does it actually look like (vs typical prep material)?",
          "back": "Production reliability and build-or-debug-an-agent-codebase as the task format. Swiggy's real 4-question loop: LLM uncertainty at scale, agent evals, LLM-as-judge, and not hallucinating success after a failed tool call -- narrower and more production-focused than most 'design an agent' prep material suggests.",
          "source": "Interview Playbook"
        },
        {
          "front": "Why should 5 similar-looking prep articles NOT be counted as 5 independent confirmations of a question?",
          "back": "They may not be independent. One widely-shared article was found to reproduce another hiring blog's three example prompts nearly verbatim. Always check whether apparently-separate sources are actually copying from each other before treating repetition as confirmation.",
          "source": "Interview Playbook"
        },
        {
          "front": "What does 'no more than two of three trifecta properties' teach about how to read a security or evidence rule precisely -- and why does the same discipline apply to interview evidence grading?",
          "back": "Read the exact wording, not the paraphrase everyone repeats (Meta's Rule of Two permits any two properties, not zero -- a common misreading). Same discipline for interview evidence: check whether a claim is exactly what the primary source said, not the shape a paraphrase gave it after passing through several prep articles.",
          "source": "Interview Playbook"
        },
        {
          "front": "A company's stated AI-tool policy for interviews (allowed, one round, whole onsite, or banned) varies by company (OpenAI's scoped exception, Meta's dedicated round, Sierra's full restructure, Anthropic's prohibition). What's the practical prep implication?",
          "back": "Don't assume one company's policy generalizes to another. Ask directly in a recruiter screen whether AI tools are allowed in technical rounds -- this page's own evidence shows real, current, company-specific disagreement on exactly that question, and assuming the wrong answer wastes real prep time.",
          "source": "Interview Playbook"
        }
      ]
    }
    </script>
    </div>

