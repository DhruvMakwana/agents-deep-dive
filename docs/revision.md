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

    ## Systems

    ### [Multi-Agent Systems](multi-agent-systems.md)

    - **A multi-agent *system* is not the same thing as orchestrator-workers.** [Workflow Patterns](workflow-patterns.md)' orchestrator-workers is a fixed topology — Anthropic classifies it as a *workflow*. A multi-agent system is what Anthropic calls an *agent* — a lead agent that delegates to subagents that are themselves autonomous, operating in parallel with their own judgment, their own tool calls, and their own context windows.
    - **A real measured run put a real number on Anthropic's token-cost claim**: single agent answering three independent questions, 1 call, 1,048 tokens; a lead agent dispatching three subagents plus a synthesis call, 4 calls, 3,369 tokens — a real **3.21x** multiplier, the same direction as (if smaller than) Anthropic's own reported 4x/15x figures.
    - **A real test of Cognition's inter-agent consistency risk did not reproduce the failure** — twice. Two subagents, each blind to the other's output, independently inventing a shared fact (a trial length) converged on the same value both times, even after the prompt was redesigned specifically to remove an obvious reason they'd default to the same answer. Reported as an honest negative result, with the real, disclosed reason why the repro's own design likely couldn't have shown the failure Cognition describes.
    - **MAST's real taxonomy**: 14 distinct failure modes across 3 categories — system design issues, inter-agent misalignment, task verification — built from 1,600+ annotated traces across 7 frameworks, and the paper's own stated finding is that multi-agent systems' "performance gains on popular benchmarks are often minimal."
    - **"When to use one agent" has a real, non-hand-wavy answer**: genuinely independent, breadth-first sub-tasks are where the token cost buys something real (Anthropic's own 90.2% improvement, largely attributable to spending more tokens); tightly-coupled tasks needing shared context between steps are where a single agent avoids a coordination problem it would otherwise have to solve by hand.

    ## Production

    ### [Evaluating Agents](evaluating-agents.md)

    - **Outcome grading and trajectory grading answer different questions, and can disagree.** Outcome asks "was the final answer right?" Trajectory asks "did it get there the intended way?" The Holistic Agent Leaderboard's own log audits caught agents "searching for the benchmark on HuggingFace instead of solving a task" — a case an outcome-only grader would have scored as a pass.
    - **pass@k and pass^k measure opposite things.** pass@k asks whether at least one of k tries succeeded (generous — useful for capability ceilings). pass^k asks whether *every one* of k tries succeeded (strict — the real reliability question, since a production agent gets one real try per request, not k with a human picking the best). τ-bench's own numbers show why the gap matters: GPT-4o's retail success rate falls from under 50% at pass@1 to under 25% at pass^8.
    - **A real capability-motivated prompt change caught a genuine, clean regression** — not in the model's reasoning, in the *grader*. Told to spell out numbers in words for a plausible accessibility reason, the model's arithmetic stayed completely correct ("one hundred divided by four equals twenty-five") — every task in a digit-matching regression suite still failed, because the grader, not the model, broke.
    - **Infrastructure is a real, measurable confound in agent benchmarks.** Running the identical model and benchmark across container configurations from strict to uncapped produced a 6-percentage-point swing (p < 0.01) on Terminal-Bench 2.0 — bigger than many reported leaderboard gaps — purely from resource allocation, with nothing about the model changing at all.
    - **Capability evals and regression evals serve different jobs and shouldn't be conflated.** A capability eval asks how good the agent could be — expensive, exploratory, run rarely. A regression eval asks whether a specific change broke something that used to work — cheap, narrow, run on every change. This page's own regression-suite experiment is exactly that second job, and it worked precisely because the suite was simple enough to run constantly.

    ### [Agent Security](agent-security.md)

    - **The lethal trifecta names the actual precondition for data exfiltration**, not a vague "be careful with untrusted content" warning: private-data access, exposure to untrusted content, and the ability to externally communicate, all live in the same session. Remove any one leg and the specific exfiltration risk this describes goes away, regardless of what the untrusted content says.
    - **A real repro of the trifecta didn't produce a successful exploit — worth being precise about what that does and doesn't show.** Claude Haiku 4.5 declined an injected forwarding request in both the vulnerable and the structurally-fixed condition. That's not evidence prompt injection is solved: "The Attacker Moves Second" reports bypassing 12 published defenses at above 90% success, where those same defenses originally reported near-zero attack success rates. The structural fix's actual value is that its guarantee doesn't depend on the model resisting at all.
    - **Tool poisoning reproduced cleanly, on a completely innocuous request.** Asked only "what's the weather in Paris," a model given a `get_weather` tool whose *description* secretly asked it to also pull an unrelated customer record did exactly that — a real supply-chain attack surface that lives in metadata nobody reads, not in a document anyone had to be tricked into opening.
    - **Meta's Rule of Two is a repackaging of the trifecta into a session-design rule, not an independent discovery** — and it's more permissive than it's often summarized: an agent may satisfy *any two* of the three properties, not zero.
    - **CaMeL's guarantee costs measurable capability**: provable security against prompt injection in AgentDojo, at 77% task success versus 84% for an undefended system — a real, quantified trade-off between structural safety and raw usefulness, not a free upgrade.

    ### [Durable Execution](durable-execution.md)

    - **Durable execution means a crash resumes the conversation instead of restarting it.** Temporal's own framing: "When a Worker crashes, the Temporal Service hands the work to another Worker, which replays the Event History and resumes at the line where execution stopped, with local variables and progress intact." For an agent specifically: "The loop is a Workflow, each model call and tool call is an Activity, and a crash resumes the conversation instead of restarting it."
    - **Replay and checkpointing are two different mechanisms that solve the same problem differently.** Temporal-style engines replay a recorded event history — re-running workflow code but skipping already-completed steps using their recorded results. LangGraph-style checkpointing persists application state directly as a snapshot. Both need a persistent backend to survive a real process crash: LangGraph's own docs note `MemorySaver` and `InMemorySaver` "store checkpoints in RAM. When the process restarts, all checkpoints are lost."
    - **A real, from-scratch repro of the exact failure mode worked cleanly.** A crash was deliberately triggered right after a tool's side effect committed but before that fact was durably logged — the single most dangerous instant for a naive retry. Resuming in a genuinely separate process replayed the completed steps with zero new model calls, then completed the task with no duplicated side effect.
    - **Idempotency is the second line of defense, not a redundant one.** Replaying the event log tells you what's *known* to have completed — it can't tell you about the gap between "the side effect happened" and "the log says it happened." An idempotent tool, checking its own persisted state before acting, is what actually prevents a double charge or a duplicate booking in that gap.
    - **This is a real, documented interview topic**, not a hypothetical: "How do you make sure agents do not double-execute side-effectful operations like charging a card or booking a ticket twice?" and "Suppose your booking agent sometimes reserves the same hotel twice — walk through how you'd debug and fix this" are both real, sourced interview questions.

    ## Training

    ### [Training Agents: Reward and Credit](training-agents.md)

    - **GRPO replaces a critic with the group itself.** DeepSeekMath's own framing: "GRPO foregoes the critic model, instead estimating the baseline from group scores" — rewards for a group of sampled completions to the same prompt are "normalized by subtracting the group average and dividing by the group standard deviation," and that normalized value becomes every token's advantage.
    - **A real, exact run of that formula reproduces GRPO's own documented failure mode.** A group where every sample got the identical reward produced an advantage of exactly zero for all eight samples — DAPO's own paper names this the "gradient-decreasing problem": "if all outputs of a particular prompt are correct and receive the same reward, the resulting advantage for this group is zero. A zero advantage results in zero policy gradients."
    - **DAPO's fix, run for real on the same degenerate group, is exactly what the paper describes**: over-sample and filter out any group whose accuracy is exactly 0 or exactly 1, keeping only groups with genuine variance — "leaving all prompts in the batch with effective gradients."
    - **Outcome reward and process reward answer different questions, with a measured real gap between them.** "Process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset" — a process-supervised model reported solving 78% of a representative MATH subset. A real toy repro shows exactly why: outcome-only credit can't tell a uniformly-bad trajectory from one where only the middle step failed; step-level credit can.
    - **DPO turns preference pairs directly into a classification loss, no separate reward model or RL loop needed** — "solve the standard RLHF problem with only a simple classification loss," reported to match or exceed PPO-based RLHF while being "stable, performant, and computationally lightweight." A real repro constructed one such pair from two real, independently sampled completions and a real judge call.

    ## Interview

    ### [Interview Playbook](interview-playbook.md)

    - **The single strongest dated trend in the evidence: AI-assisted coding is becoming the interview format itself, not just the topic.** OpenAI runs a beta onsite round done "using an AI coding agent." Meta "added an AI-assisted coding round" where a small toy codebase is debugged and completed with LLM help. Sierra rebuilt its entire onsite around a 2-hour build session using any AI tooling, and removed its classic coding/algorithms interview outright. Anthropic, by contrast, explicitly disallows AI tools in its live rounds — the field hasn't converged on one answer.
    - **Most of what's published as "agent interview questions" is prep-market content, not candidate-confirmed.** Of 106 collected questions, only 14 trace to a first-person candidate report or a company's own published process; 81 come from prep sites, listicles, and openly "synthesized" repositories. Treat frequency in prep material as evidence of what the prep industry teaches, not what interviewers actually ask.
    - **A real loop can change within a year.** Sierra's own May-2025 candidate reports describe a take-home support agent plus a TypeScript/React debugging round; Sierra's own April-2026 blog post describes a completely different AI-native onsite. Prepping from a year-old write-up can mean prepping for a loop that no longer exists.
    - **Job descriptions and interview reports disagree on what to emphasize.** Kubernetes appears in 19.4% of AI-tech postings and in zero candidate interview reports reached. Idempotency and durable execution show up across multiple prep sources and real interview questions but in only 1.1% of postings. Evals is the rare topic strongly present in both.
    - **The one topic with genuine candidate confirmation across multiple independent sources: production reliability under real failure, not textbook agent architecture** — Swiggy's real four-question loop was entirely about uncertainty at scale, agent evals, LLM-as-judge, and not hallucinating success after a failed tool call; Sierra's real rounds are build-and-debug tasks, not "design an agent" whiteboard questions.

=== "Combined Scenario Check"

    52 questions from every page on this site, one combined pass instead of opening each page separately. Every question shows which page it's from — go re-read that page for anything you get wrong.

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

    104 flashcards from every page with a deck so far — click a card to flip it, shuffle for random order.

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

