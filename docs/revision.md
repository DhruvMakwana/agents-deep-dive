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

=== "Combined Scenario Check"

    3 questions from every page on this site, one combined pass instead of opening each page separately. Every question shows which page it's from — go re-read that page for anything you get wrong.

    <div class="quiz-widget" data-title="Combined Scenario Check — All Pages">
    <script type="application/json">
    {
      "questions": [
        {
          "scenario": "A team built a support bot: classify the customer's message into one of 8 categories, retrieve the matching help-center article, and have an LLM turn it into a reply. They call it 'our support agent.'",
          "question": "Under Anthropic's workflow-vs-agent distinction, is this an agent?",
          "options": [
            "Yes \u2014 it uses an LLM at every step, so it's agentic",
            "No \u2014 it's a workflow, because the sequence of steps is fixed regardless of what any step returns",
            "It depends on how many categories there are",
            "Yes, because retrieval makes any pipeline agentic"
          ],
          "correct": 1,
          "explanations": [
            "Using an LLM at each step doesn't make a system agentic \u2014 the classic mistake this page opens with. The line is who decides the next step, not how many LLM calls happen.",
            "Correct. There's no point where the outcome of one step changes which step runs next \u2014 that's exactly Anthropic's definition of a workflow, even with LLM calls throughout.",
            "Category count is irrelevant to the workflow-vs-agent distinction \u2014 it's about whether the control flow can branch on live feedback, not scale.",
            "Retrieval is orthogonal to this distinction; RAG pipelines are workflows unless something in the loop can decide to retrieve differently based on what came back."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "An engineer adds a retry: if the generated answer's confidence score (computed by fixed code) is below a threshold, regenerate up to 3 times before returning the best-scoring attempt.",
          "question": "Does adding this retry loop make the system an agent?",
          "options": [
            "Yes \u2014 any loop with more than one attempt is agentic",
            "No \u2014 this is Anthropic's evaluator-optimizer workflow pattern; the retry logic is still fixed code, not a model decision",
            "Yes, because it now has multiple steps",
            "No \u2014 because it uses a threshold instead of an LLM judge"
          ],
          "correct": 1,
          "explanations": [
            "A loop alone doesn't cross the line \u2014 Anthropic explicitly lists a generate-grade-retry loop as a workflow pattern (evaluator-optimizer), not an agent.",
            "Correct. The decision to retry and when to stop is governed by fixed code (a threshold, a step count) \u2014 the model never chooses its own next action from live options.",
            "Step count isn't the criterion; a five-step fixed pipeline is still a workflow, and a one-step agent that picks its own tool is still an agent.",
            "The mechanism used to score the output (a threshold vs. an LLM judge) doesn't matter here \u2014 what matters is who decides whether/how to retry, and that's still the code."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        },
        {
          "scenario": "You're asked in an interview: 'Given unlimited engineering time, would you make every feature in your product agentic?'",
          "question": "What's the strongest answer?",
          "options": [
            "Yes \u2014 agents are strictly more capable, so there's no reason not to",
            "No \u2014 reflexively reaching for agentic architecture adds latency, cost, and non-determinism that a well-scoped fixed pipeline doesn't need, and evaluation gets harder",
            "It depends only on whether the team has experience with agent frameworks",
            "No \u2014 agents are never appropriate for production systems"
          ],
          "correct": 1,
          "explanations": [
            "More capable at open-ended tasks, yes \u2014 but that capability comes with real costs (latency, cost, non-determinism, harder eval) that a fixed pipeline doesn't pay when the task doesn't need branching.",
            "Correct. This is the trade-off question this page's 'When you would not build an agent' section is built around \u2014 interviewers are testing judgment, not enthusiasm.",
            "Framework familiarity is an implementation detail, not the reason to choose an architecture; the decision should be driven by whether the task genuinely needs branching on live feedback.",
            "Too absolute in the other direction \u2014 plenty of production systems (this whole site's Systems and Production sections) are agentic where the task genuinely calls for it."
          ],
          "source": "What Is an Agent?",
          "sourceUrl": "what-is-an-agent.md"
        }
      ]
    }
    </script>
    </div>

=== "Flashcards"

    8 flashcards from every page with a deck so far — click a card to flip it, shuffle for random order.

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
        }
      ]
    }
    </script>
    </div>

