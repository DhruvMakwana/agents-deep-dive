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

=== "Combined Scenario Check"

    28 questions from every page on this site, one combined pass instead of opening each page separately. Every question shows which page it's from — go re-read that page for anything you get wrong.

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
        }
      ]
    }
    </script>
    </div>

=== "Flashcards"

    56 flashcards from every page with a deck so far — click a card to flip it, shuffle for random order.

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
        }
      ]
    }
    </script>
    </div>

