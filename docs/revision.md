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

=== "Combined Scenario Check"

    8 questions from every page on this site, one combined pass instead of opening each page separately. Every question shows which page it's from — go re-read that page for anything you get wrong.

    <div class="quiz-widget" data-title="Combined Scenario Check — All Pages">
    <script type="application/json">
    {
      "questions": [
        {
          "scenario": "A support system does classify -> retrieve -> generate over 50 FAQ categories, chosen by embedding similarity. When the best match's similarity score falls below a fixed threshold, the code shows the customer 3 candidate articles instead of generating one answer.",
          "question": "Does the threshold branch make this system an agent?",
          "options": [
            "Yes \u2014 showing multiple candidates instead of one means the system is adapting its behavior to retrieval quality, which is the decision-point behavior that defines an agent",
            "No \u2014 the branch (one answer vs. three candidates) is triggered by a fixed numeric threshold check in code, not by the model choosing what to do next based on its own judgment",
            "It depends on whether the embedding model or the generation model owns the threshold",
            "Yes, because the system now has more than one possible output path depending on the input"
          ],
          "correct": 1,
          "explanations": [
            "This is the tempting trap: the system's output genuinely does vary with live data (the similarity score), which sounds like the 'depends on the outcome of a previous action' criterion. But varying output isn't the same as the model deciding \u2014 a lookup table that branches on a number is still just a lookup table, however adaptive it looks from outside.",
            "Correct. A fixed threshold in code is choosing the next step, not the model. Nothing here lets the model itself decide what happens next based on its own read of the situation \u2014 replace the threshold with any other number and the architecture is identical.",
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
            "It's an agent \u2014 an LLM is making the 'good enough' judgment, and that judgment is a real decision",
            "It's a workflow \u2014 the LLM's yes/no only gates a branch that fixed code already wired in (retry once, then stop and return whatever you have); the model never gets to choose a different action, only fill in a pre-built slot",
            "It's an agent on the second attempt only, since that's the point where branching happens",
            "It's ambiguous, and the answer depends on which provider is used for the judgment call"
          ],
          "correct": 1,
          "explanations": [
            "This is the most common wrong intuition on this whole page: an LLM call is involved in the decision, so it feels agentic. But this is exactly the pattern Anthropic names 'evaluator-optimizer' and classifies as a workflow \u2014 the LLM fills in a yes/no gate whose consequences (retry once, cap at 2, return regardless of the second verdict) were fully decided by the code author in advance.",
            "Correct. The action space here has exactly two pre-wired outcomes (retry once, or don't), chosen by code, not model-selected from live options. Compare this to this page's own recipe: the judge step there feeds into a decision where the model picks among BROADEN/CLARIFY/ESCALATE \u2014 a genuinely open action set, not a single fixed retry slot.",
            "Branching happening at a specific point doesn't retroactively make earlier or later steps agentic or non-agentic \u2014 the whole system is one architecture, evaluated by the same rule throughout.",
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
            "Build it as a workflow \u2014 the correct procedure for each of the 12 cases is already known and doesn't branch on live results, so an agent's dynamic decision-making adds cost, latency, and non-determinism without buying anything this task needs; revisit the decision if and when new request types actually appear",
            "It must be an agent, because customer-facing workflows are inherently unpredictable",
            "Build it as an agent, because letting the model choose the procedure is more reliable than hard-coded step selection on a fully known task"
          ],
          "correct": 1,
          "explanations": [
            "A real and common engineering trap \u2014 over-building for imagined future flexibility the task doesn't currently need. If new request types show up later, that's the moment to reconsider, not a reason to pay agentic costs today for a fully solved, fully verified problem.",
            "Correct. This is the 'When you would not build an agent' trade-off from this page, applied to a concrete case instead of asked in the abstract \u2014 the task is fully known and doesn't need branching on live feedback, so a fixed pipeline is strictly more reliable and cheaper here.",
            "An unfounded generalization \u2014 the scenario explicitly states the procedures are verified and deterministic. 'Customer-facing' doesn't imply unpredictable; the two are independent.",
            "False, and worth catching directly: an LLM choosing among known, verified procedures is not more reliable than the verified procedures themselves \u2014 on a fully known task, hard-coded logic doesn't fail in the ways a model's live judgment can."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "A candidate is asked how LLM agents improve over a deployment's lifetime, given they don't get gradient updates from a reward signal the way RL agents do. The candidate answers: 'They don't really improve \u2014 each session starts fresh with the same weights, so there's no real analog to RL's learning.'",
          "question": "What's the best critique of this answer?",
          "options": [
            "The candidate is basically right \u2014 without fine-tuning, there's no improvement mechanism at all",
            "The candidate is missing the in-context mechanisms (reflection notes, accumulated memory, refined tools/prompts carried forward across sessions) plus the fact that RL-style training (GRPO, DPO on trajectories) is increasingly applied on top of the deployed loop, not as a one-time pretraining step",
            "The candidate is wrong because model weights are automatically updated after every conversation",
            "The candidate is right for closed-source models but wrong for open-weight models, which retrain continuously between sessions"
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
            "Nothing is wrong with it \u2014 this is an accurate description",
            "The model never directly invokes anything. It generates a structured request (a tool_use block naming a function and arguments); your own code parses that, runs the real function, and sends the result back as a distinct new message the model reads on its next turn. The SDK transports messages, it doesn't execute your functions",
            "It's correct for Anthropic specifically, but OpenAI's API does execute functions server-side, so the explanation doesn't generalize",
            "It's mostly right, except the return value gets appended silently into the same message instead of arriving as a new one"
          ],
          "correct": 1,
          "explanations": [
            "This is exactly the weak interview answer this page's Interview angle section warns about \u2014 it skips the entire mechanism.",
            "Correct. No mainstream provider's tool-calling API executes your functions for you \u2014 Anthropic, OpenAI, and Google all require the caller to parse the request, run the real code, and send the result back explicitly. That round trip is the whole mechanism.",
            "A tempting 'maybe it varies by vendor' hedge, but false \u2014 execution is the caller's responsibility across every major provider's tool-calling API, not just Anthropic's.",
            "A specific, plausible-sounding technical detail that happens to be wrong: results travel back as their own distinct message (a tool_result), not appended silently onto an existing one \u2014 this page's own loop code makes that explicit."
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
        },
        {
          "scenario": "A team has 40 tools registered for one agent and notices frequent wrong-tool selection and invented arguments. An engineer proposes collapsing all 40 into a single 'do_anything' tool that takes one free-text 'instruction' string and dispatches internally via keyword matching, reasoning that a single tool means the model can't pick the wrong one.",
          "question": "What's the strongest critique of this proposal?",
          "options": [
            "It's a good fix \u2014 fewer tools in the schema means less chance of tool confusion",
            "It doesn't remove the selection problem, it moves it \u2014 from the model choosing among 40 well-described, schema-validated tools, into a keyword matcher inside your own code choosing among 40 possible actions from unstructured text. That trades a visible, improvable failure mode for a hidden, harder-to-debug one, and throws away the argument validation the model was actually doing reasonably well",
            "It's correct, because with a single tool the model never has to make a selection decision at all",
            "It's wrong only because free-text instructions can't be validated by JSON Schema \u2014 everything else about the idea is sound"
          ],
          "correct": 1,
          "explanations": [
            "The naive read: fewer visible tools, less confusion. But the selection decision hasn't gone away, it's just moved somewhere you can't see it or fix it the same way.",
            "Correct. The model still has to figure out which of 40 things you mean from a free-text instruction, and now that decision happens inside an internal keyword matcher instead of the model's tool selection \u2014 which is typically less capable at exactly this kind of disambiguation and much harder to debug when it picks wrong.",
            "The selection decision still exists, it's just been relocated from the model's tool choice into your dispatch code's keyword matching \u2014 'no decision' is not what happened here.",
            "A half-right trap: schema validation loss is real, but framing it as the *only* problem misses the bigger one \u2014 you've hidden the selection logic where you can no longer inspect, test, or improve it the way you could with 40 separate tool descriptions."
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
        },
        {
          "scenario": "An agent's loop is capped at max_iterations=10. A teammate objects: 'That's not a real safety net \u2014 nothing stops a single iteration from being slow. A model can request a tool that hangs for 90 seconds, and your 10-iteration cap does nothing about that.'",
          "question": "Is the teammate right?",
          "options": [
            "No \u2014 max_iterations bounds the number of LLM round-trips, which is what actually matters for cost, and per-tool latency is a separate, unrelated concern",
            "Yes \u2014 an iteration cap bounds how many times the loop goes around, but says nothing about how long any single tool call is allowed to take. A production loop needs a per-call timeout on each tool execution too, independent of the iteration count",
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
            "The baseline actually got every arithmetic step right on its own. What it lacked was the exchange rate, which isn't a math problem at all \u2014 it's a live-data problem that no amount of arithmetic skill solves without a real source, which is exactly what the currency tool provided",
            "The colleague is right, but only for multiplication and division \u2014 the baseline's addition and subtraction steps were reliable",
            "The gap is because Haiku is too small a model for this task \u2014 a larger model would have matched the tool-using answer with no tools at all"
          ],
          "correct": 1,
          "explanations": [
            "The clich\u00e9 conclusion, and specifically the wrong read of this run: the recipe README's real trace shows the baseline's tip, total, and per-person arithmetic were all correct \u2014 the number it was honestly unsure about was the exchange rate, not any arithmetic step.",
            "Correct \u2014 and this is the point worth remembering past this page: a gap that looks like a capability failure is sometimes a data-access failure instead, and the fix (a tool that supplies the missing data) is different from the fix for a genuine reasoning failure (a better model, or a different prompt).",
            "An oddly specific and fabricated distinction \u2014 nothing in the actual run supports operation-by-operation reliability differences; the model's addition, multiplication, and division were all correct in this trace.",
            "A tempting but wrong appeal to scale: model size doesn't create access to a live exchange rate that was never in the prompt or the model's training data at query time \u2014 this is a knowledge-access gap, not a capacity gap, and no larger model closes it without an actual data source."
          ],
          "source": "The Agent Loop From Scratch",
          "sourceUrl": "agent-loop-from-scratch.md"
        }
      ]
    }
    </script>
    </div>

=== "Flashcards"

    16 flashcards from every page with a deck so far — click a card to flip it, shuffle for random order.

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
        }
      ]
    }
    </script>
    </div>

