# Reasoning Paradigms

!!! example "Hands-on"
    Full runnable recipe: [`reasoning-paradigms/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/reasoning-paradigms) in the companion cookbook — Reflexion, ReWOO vs. ReAct measured on real call count and token count, LLM Compiler's concurrent dispatch, and Plan-and-Solve vs. zero-shot CoT, each with a real captured run.

??? abstract "TL;DR — quick revision"
    - **These are a different axis from [workflow patterns](workflow-patterns.md).** Workflow patterns fix the *topology* (which steps exist, in what order). Reasoning paradigms change how a *single step or loop* reasons, searches, or recovers from its own mistakes — they're about the shape of thinking, not the shape of the pipeline.
    - **Plan-and-Solve is a zero-shot prompting technique, not a planner/executor architecture** — it's one prompt ("first devise a plan, then carry it out") compared against plain "let's think step by step." Don't confuse it with LangChain's separately-named Plan-and-Execute agent, which actually is an architecture.
    - **ReWOO's real, verified advantage is token and call efficiency, not latency** — the paper never claims a latency win. A real measured run on this page needed 2.5x the calls and 6.8x the tokens for ReAct versus ReWOO on the identical question and model.
    - **ReWOO's plan-then-execute split has a real failure surface ReAct doesn't**: a bad step in the plan is only caught once it's actually executed, not before. This page's real run hit exactly that — a generated plan referenced a function its tool didn't support, and one step genuinely failed.
    - **Reasoning models changed what needs to be built by hand.** DeepSeek-R1's reported finding is that self-reflection and verification can emerge from pure reinforcement learning, without anyone hand-coding a Reflexion-style retry loop or a ToT-style external search — some of what these 2023 papers built as scaffolding, later training runs learned to do internally.

## Why these are a different axis from workflow patterns

[Workflow Patterns](workflow-patterns.md) covered five ways to wire fixed pipelines: chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer. Every one of those fixes the *sequence of steps* ahead of time — the LLM fills in judgment calls inside a shape the developer already committed to in code.

The paradigms on this page are a different axis entirely: they're about what happens *inside* a single reasoning step or loop, independent of pipeline shape. ReAct is about how one agent loop decides its next action. Reflexion is about what happens after that loop fails. Tree of Thoughts is about exploring more than one line of reasoning before committing to an answer. None of them describe a pipeline topology — they describe a reasoning *strategy*, and several of them (ReWOO, ReAct) could sit inside any of last page's workflow patterns as the mechanism one step uses internally.

## ReAct: the loop you've already built

[The Agent Loop From Scratch](agent-loop-from-scratch.md) *is* a ReAct-style loop — the paper's own framing is generating reasoning traces and actions in an interleaved manner, where the reasoning helps the model track its plan and handle exceptions, and the actions let it reach outside the model into tools or an environment. Nothing new to build here; that page's real captured tool-calling loop is the working reference. The paper (Yao et al., ICLR 2023) reported a 34-percentage-point absolute success-rate improvement over imitation and reinforcement-learning baselines on ALFWorld, and a 10-point improvement on WebShop — evidence for interleaving reasoning and acting specifically, not for any particular provider's tool-calling API, which didn't exist yet in 2022 when this was written.

## Reflexion: solve, verify, reflect, retry

Reflexion agents verbally reflect on a failure signal, then keep that reflection in an episodic memory buffer to inform the next attempt — no gradient update, no fine-tuning, just text carried forward into the next try. On HumanEval, the paper reports 91% pass@1, ahead of the GPT-4 baseline's reported 80% at the time (Shinn et al., 2023).

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/reasoning-paradigms/reasoning_paradigms_docs.py:reflexion"
```

!!! success "A real run — input and output shown in full"
    **Input** — the exact question sent to the model, a 9-swap state-tracking puzzle with a locally-computed (non-LLM) ground truth: *"Nadia starts holding the red key, Omar starts holding the blue key, and Priya starts holding the green key. Then, in this exact order: 1. Nadia and Omar swap whatever key they are currently holding. 2. Omar and Priya swap whatever key they are currently holding. 3. Nadia and Priya swap whatever key they are currently holding."* — repeated in that same 3-swap pattern three times (9 swaps total), then: *"After all the swaps, who holds the red key? End your answer with a final line in the exact form: Final answer: <name>"*

    **Output — attempt 1**, Claude Haiku 4.5, correct on the first try:
    > I'll track which key each person holds after each swap.
    >
    > **Initial state:** Nadia: Red, Omar: Blue, Priya: Green
    >
    > **After swap 1 (Nadia and Omar swap):** Nadia: Blue, Omar: Red, Priya: Green
    >
    > **After swap 2 (Omar and Priya swap):** Nadia: Blue, Omar: Green, Priya: Red
    >
    > *(...six more swaps, each stated explicitly...)*
    >
    > **After swap 9 (Nadia and Priya swap):** Nadia: Red, Omar: Green, Priya: Blue
    >
    > Final answer: Nadia

    Ground truth (computed independently, by literally simulating the swaps in Python, not by the model): **Nadia**. Correct — no reflection needed, so none fired.

    An honest result, not a weaker one: the failure this mechanism exists to catch didn't occur on this run. What's genuinely interesting is *why* it didn't — the model wrote out the full intermediate state after every single swap instead of trying to hold nine sequential updates in its head, which is exactly the kind of externalized bookkeeping that avoids the state-tracking errors this task was designed to risk. That's a real, checkable finding about *how* it avoided the failure, not just that it did.

## Plan-and-Solve: a prompt, not an architecture

The seed interview-prep material this blog started from described Plan-and-Solve as an early planner/executor split. That's not what the paper is: it's a **zero-shot prompting technique** — one instruction, "let's first understand the problem and devise a complete plan, then carry out the plan and solve the problem step by step," compared against the plainer "let's think step by step" (zero-shot chain-of-thought). The paper's target is zero-shot-CoT's three failure modes — calculation errors, missing-step errors, and semantic misunderstanding — and it reports that Plan-and-Solve "consistently outperforms Zero-shot-CoT across all datasets by a large margin" and reaches performance "comparable with 8-shot CoT prompting" on math reasoning (Wang et al., ACL 2023). Don't confuse this with LangChain's separately-named Plan-and-Execute *agent*, which genuinely is a two-role architecture — same words, different thing, published two months apart.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/reasoning-paradigms/reasoning_paradigms_docs.py:plan_and_solve"
```

!!! success "A real run — same problem, two prompts, both shown in full"
    **Input** — the exact word problem, identical for both prompts: *"A bakery bakes 320 loaves of bread each morning. It sells 3/8 of the loaves by noon, then donates 20 loaves to a shelter, then sells half of what remains by evening. The rest goes into the discount bin overnight. Of the loaves in the discount bin, 20% are stale and thrown out the next morning. How many loaves actually make it to the discount bin's shelf for sale?"* Ground truth, computed independently: **72** loaves.

    **Output — zero-shot-CoT** ("Let's think step by step."), Claude Haiku 4.5:
    > Step 1: Calculate loaves sold by noon — 3/8 × 320 = 120 loaves. Step 2: Remaining = 320 − 120 = 200 loaves. Step 3: After donating 20 loaves = 200 − 20 = 180 loaves. Step 4: Sold by evening = 1/2 × 180 = 90 loaves. Step 5: Remaining = 180 − 90 = 90 loaves. Step 6: Stale loaves = 20% × 90 = 18 loaves. Step 7: Available for sale = 90 − 18 = **72 loaves**

    **Output — Plan-and-Solve**, same model:
    > **Problem Analysis and Plan** — Let me break down what happens to the 320 loaves throughout the day: 1. By noon: sell 3/8 of loaves 2. After donation: donate 20 loaves to shelter 3. By evening: sell half of what remains 4. Overnight: remaining loaves go to discount bin 5. Next morning: 20% of discount bin loaves are stale and thrown out.
    >
    > **Step-by-Step Solution** — Step 1: Sold by noon: 3/8 × 320 = 120, remaining 200. Step 2: After donation: 200 − 20 = 180. Step 3: Sold by evening: 1/2 × 180 = 90, remaining 90. Step 4: Discount bin receives 90. Step 5: Stale: 20% × 90 = 18. Step 6: Available for sale: 90 − 18 = **72 loaves**

    Both correct: **72**. An honest negative result for this specific comparison — the paper's claim is distributional (it wins "across all datasets," averaged over many problems), not a guarantee on any single one, and this one problem didn't distinguish the two prompts. The real difference in the transcripts isn't the answer, it's structure: Plan-and-Solve produced an explicit, separated planning section before any arithmetic began, while zero-shot-CoT interleaved planning and computing from the first line.

## ReWOO: plan once, execute deterministically

ReWOO's own abstract frames the problem directly: in an interleaved loop, "an LLM reasons to call an external tool, gets halted to fetch the tool's response, and then decides the next action based on all preceding response tokens" — a pattern that "often leads to huge computation complexity from redundant prompts and repeated execution" (Xu et al., 2023). ReWOO's fix: one planning call produces the *entire* plan up front, using `#E1`, `#E2`-style variables so a later step can reference an earlier step's result before that result exists yet — then the plan executes deterministically, with real values substituted in, and a final call solves from the gathered evidence. Two LLM calls, no matter how many tool steps the plan has. The paper reports "5x token efficiency and 4% accuracy improvement on HotpotQA" against the interleaved baseline — a token and accuracy claim, notably not a latency one.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/reasoning-paradigms/reasoning_paradigms_docs.py:rewoo"
```

!!! success "A real run — the exact question, the real generated plan, and a real partial failure"
    **Input** — the question, sent once, to a planner: *"Using the colony records, what is the combined population of Meridian Station and Halcyon Outpost, expressed as a percentage of Kepler Reach's population? Round to 2 decimal places."* Fictional records the tools read from: Meridian Station 48,200; Halcyon Outpost 15,750; Kepler Reach 92,400.

    **Output — the plan**, generated by Claude Sonnet 5 in one call, verbatim:
    ```json
    [
      {"var": "E1", "tool": "lookup_population", "args": {"entity": "Meridian Station"}},
      {"var": "E2", "tool": "lookup_population", "args": {"entity": "Halcyon Outpost"}},
      {"var": "E3", "tool": "lookup_population", "args": {"entity": "Kepler Reach"}},
      {"var": "E4", "tool": "calculator", "args": {"expression": "E1 + E2"}},
      {"var": "E5", "tool": "calculator", "args": {"expression": "(E4 / E3) * 100"}},
      {"var": "E6", "tool": "calculator", "args": {"expression": "round(E5, 2)"}}
    ]
    ```

    Executing that plan deterministically, with real values substituted for each `#E` reference, produced this evidence: `E1=48200, E2=15750, E3=92400, E4=63950, E5=69.20995670995671`, and then **`E6` genuinely failed**: `"Could not evaluate 'round(69.20995670995671, 2)': Disallowed expression"` — this recipe's calculator, like `agent-loop-from-scratch`'s, is an AST walk that only allows `+ - * / **`, deliberately not function calls. The model's plan assumed a capability the tool didn't have. This is exactly the failure surface the mechanism itself creates: a bad step is only caught once the plan actually runs, because nothing checks the plan against the tool's real capabilities before execution starts — unlike ReAct below, where the model sees each real result before deciding the next action.

    **Output — the final solve call**, given that evidence (including `E6`'s error) and nothing else: `"69.21"`. Correct, and recovered *despite* one step failing — the solve step reasoned from `E5`'s raw unrounded value once `E6` came back unusable, rather than the whole run failing.

## ReAct vs. ReWOO, measured

Same question, same model (Sonnet), same correct final answer — call count and token count measured, not estimated:

| | Calls | Total tokens |
|---|---|---|
| ReWOO | 2 | 756 |
| ReAct | 5 | 5,141 |

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/reasoning-paradigms/reasoning_paradigms_docs.py:react"
```

!!! success "A real run — ReAct's turn-by-turn trace"
    **Methodology, disclosed plainly**: this ReAct loop deliberately executes only the *first* requested tool call per turn, even on a turn where the model requests several — otherwise Claude's native parallel tool-calling could quietly collapse several ReAct "steps" into one API call, which would measure the API's own batching rather than ReAct's actual interleaved-decision mechanism.

    Real trace, one call per line: turn 1 → `lookup_population("Meridian Station")` → `48200`. Turn 2 → `lookup_population("Halcyon Outpost")` → `15750`. Turn 3 → `lookup_population("Kepler Reach")` → `92400`. Turn 4 → `calculator("(48200 + 15750) / 92400 * 100")` → `69.20995670995671`. Turn 5 → final answer: *"The combined population of Meridian Station (48,200) and Halcyon Outpost (15,750) is 63,950, which is **69.21%** of Kepler Reach's population (92,400)."*

    **5 calls, 5,141 total tokens** against ReWOO's **2 calls, 756 total tokens** — 2.5x the calls, 6.8x the tokens, for the identical question, model, and final answer. The gap between the call-count ratio (2.5x) and the token ratio (6.8x) is itself the finding: ReAct's transcript grows every turn and gets resent in full on the next call, so token cost compounds faster than call count does — precisely the "redundant prompts and repeated execution" ReWOO's own abstract names as the thing it's fixing.

## LLM Compiler: parallel dispatch over the plan

ReWOO's plan above executes in written order regardless of whether a step actually depends on an earlier one — `E1`, `E2`, and `E3` don't depend on each other at all, yet nothing in ReWOO dispatches them concurrently. LLM Compiler's contribution is exactly that gap: a planner that produces an execution graph, a dispatch unit that starts any task whose dependencies are already satisfied, and an executor running independent tasks in parallel (Kim et al., ICML 2024). The paper reports "latency speedup of up to 3.7x," "cost savings of up to 6.7x," and "accuracy improvement of up to ~9%" against a ReAct-style baseline across their benchmark suite.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/reasoning-paradigms/reasoning_paradigms_docs.py:llm_compiler"
```

!!! success "A real run — reusing ReWOO's own plan, dispatched differently"
    Given the exact same 6-step plan ReWOO generated above, dependency analysis correctly split it into `E1, E2, E3` (no reference to any other step's result — independent) and `E4, E5, E6` (each references an earlier `#E` — dependent). No new LLM call needed for this: it's a regex check over the plan's own argument strings.

    Dispatching the three independent lookups concurrently: **0.45s**. Running the same three, one at a time: **1.22s** — about a 2.7x speedup for 3 calls. **Disclosed plainly**: these are fictional, in-memory records with an artificial 0.4s delay added to stand in for real I/O latency (a real API call, a real database query), since there's nothing to actually wait on for a dict lookup — this measures the concurrency *mechanism* honestly, not a claim about real network behavior. For a real-network version of the same measurement, see [Workflow Patterns](workflow-patterns.md#parallelization)'s parallelization demo, which measured a 2.6x speedup on 3 genuine API calls (the more realistic number, since queueing and network variance eat into the clean theoretical multiple).

## Tree of Thoughts and LATS: search over thoughts

Tree of Thoughts generalizes chain-of-thought into a search: instead of one linear reasoning path, the model proposes multiple candidate "thoughts" at each step, self-evaluates them, and explores the more promising branches — with the ability to backtrack, unlike a single forward pass (Yao et al., 2023). On the Game of 24 arithmetic puzzle, the paper reports GPT-4 with chain-of-thought prompting solved 4% of tasks, against a 74% success rate with ToT search. That's a genuinely large gap, and it comes with a genuinely large cost: exploring several branches at each of several steps multiplies the number of LLM calls per problem, which is exactly why this page didn't build a fresh demo for it — the tradeoff itself (breadth of search bought with a multiplicative call budget) doesn't need a bespoke run to establish; it's inherent to the algorithm, the same way parallelization's real speedup was worth measuring but its existence wasn't in question.

LATS (Language Agent Tree Search) goes further, combining ReAct-style acting, Reflexion-style self-reflection, and Monte Carlo Tree Search's exploration into one loop, using the language model itself as both the value function scoring a branch and the source of reflections that inform later branches (Zhou et al., 2023). The paper reports 92.7% pass@1 on HumanEval with GPT-4, and a WebShop score the authors describe as comparable to gradient-based fine-tuning — without any gradient update. It's the most capable of the paradigms on this page and the most expensive to run, for the same structural reason as ToT: real tree search over real API calls.

## What reasoning models changed

Every paradigm above except ReAct was built as external scaffolding around a base model that didn't reliably reflect on, search over, or verify its own reasoning unprompted — Reflexion's episodic buffer, ToT's explicit branching, LATS's tree search all exist because the mechanism wasn't happening inside the model on its own. DeepSeek-R1's reported finding changes that premise for at least some of these: "reasoning abilities of LLMs can be incentivized through pure reinforcement learning (RL), obviating the need for human-labeled reasoning trajectories," and the paper reports that "self-reflection, verification, and dynamic strategy adaptation" emerged from that RL training rather than being explicitly programmed (DeepSeek-AI, 2025).

The practical shift: for tasks a modern reasoning model handles well, some of what Reflexion or ToT used to buy through external scaffolding now happens inside one long chain-of-thought, before the model ever emits a final answer — self-checking and backtracking as part of generation, not as a separate loop your code has to build and orchestrate. This doesn't make the paradigms on this page obsolete. A reasoning model's internal self-correction isn't inspectable or interruptible the way an explicit Reflexion loop is, it isn't available at all on models without that training, and it doesn't help with the structural problems ReWOO and LLM Compiler solve — those are about *call and token economics* across tool-using steps, a different axis entirely from whether the model second-guesses itself well. What changed is which problems are worth solving with hand-built scaffolding versus which ones a well-trained model increasingly handles on its own — worth checking per-task, not assuming either way.

## Interview angle

**Weak answer** to "when would you reach for ReWOO instead of ReAct": *"ReWOO is faster since it needs fewer calls."* This states the direction of the real effect but skips the actual trade-off, and this page's own real run shows why that matters: ReWOO's `E6` step failed, and ReWOO structurally could not have caught that until execution — there's no point where the model looks at a real result before committing to the next step, because the whole plan was already committed before any of it ran.

**Strong answer**: the real trade-off is speed-through-fewer-round-trips against a wider blast radius per undetected planning error. ReWOO wins decisively on tokens and calls (this page measured 6.8x and 2.5x on one real run) because the model reasons about the *shape* of the problem once, then never has to re-read a growing transcript. ReAct wins on catching a bad step early, because every decision is made with the real previous result already in hand. The corollary that actually answers "when": ReWOO is the better default when the tool calls are cheap to verify after the fact and errors are recoverable at the solve step (as this run's own `E6` failure was) — ReAct is the better default when a wrong intermediate step could send the rest of the plan somewhere expensive or irreversible to unwind.

**Follow-up to expect**: "doesn't LLM Compiler just make ReWOO's redundant round-trips point moot, since you get parallelism either way?" No — LLM Compiler adds *concurrency* for independent steps, which is a wall-clock win, not a token-count win. ReWOO's own plan in this page's real run had three independent lookups; running them concurrently (LLM Compiler's contribution) sped up wall-clock time, but it didn't touch the token count at all, because the same evidence still gets assembled into the same final solve call either way. Token efficiency and call-latency are separate axes — a system can be strong on one and unremarkable on the other, and this page's own numbers show ReWOO winning specifically on tokens/calls while LLM Compiler wins specifically on wall-clock.

## Build it yourself — 30 minutes

1. Pick a multi-step question that genuinely needs 3+ tool calls, with a ground truth you can compute independently of any model (a calculation, a lookup you control) — like this page's population question.
2. Build the ReAct version first: a loop that decides one action, executes it, observes the real result, and only then decides the next action. Restrict it to one tool call per turn even if your provider supports requesting several, so you're testing the mechanism, not the API's batching.
3. Build ReWOO next, on the identical question: one call that plans every step with `#E`-style variables, deterministic execution with substitution, one call that solves from the gathered evidence. Compare real call counts and real token counts (from your API response's usage field) against step 2 — don't estimate them.
4. Only then try LLM Compiler: take ReWOO's plan, write a simple dependency check (does a step's arguments reference another step's variable?), and dispatch the independent ones concurrently. Time it against running them one at a time.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Reasoning Paradigms">
<script type="application/json">
{
  "questions": [
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
      ]
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
      ]
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
      ]
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
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Yao et al., ["ReAct: Synergizing Reasoning and Acting in Language Models"](https://arxiv.org/abs/2210.03629) (arXiv 2210.03629, submitted 2022-10-06, ICLR 2023) — the interleaved thought/action/observation mechanism every tool-calling agent loop uses today.
- Shinn et al., ["Reflexion: Language Agents with Verbal Reinforcement Learning"](https://arxiv.org/abs/2303.11366) (arXiv 2303.11366, submitted 2023-03-20) — verbal self-reflection stored in an episodic buffer across trials.
- Wang et al., ["Plan-and-Solve Prompting: Improving Zero-Shot Chain-of-Thought Reasoning by Large Language Models"](https://arxiv.org/abs/2305.04091) (arXiv 2305.04091, ACL 2023) — a zero-shot prompting technique, not a planner/executor architecture.
- Xu et al., ["ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models"](https://arxiv.org/abs/2305.18323) (arXiv 2305.18323, submitted 2023-05-23) — plan-then-execute with variable substitution; reports token efficiency, not latency.
- Kim et al., ["An LLM Compiler for Parallel Function Calling"](https://arxiv.org/abs/2312.04511) (arXiv 2312.04511, ICML 2024) — a DAG-based planner and parallel dispatch over independent tool calls.
- Yao et al., ["Tree of Thoughts: Deliberate Problem Solving with Large Language Models"](https://arxiv.org/abs/2305.10601) (arXiv 2305.10601, submitted 2023-05-17) — search over candidate reasoning branches with self-evaluation and backtracking.
- Zhou et al., ["Language Agent Tree Search Unifies Reasoning, Acting, and Planning in Language Models"](https://arxiv.org/abs/2310.04406) (arXiv 2310.04406, submitted 2023-10-06) — Monte Carlo Tree Search combining ReAct-style acting, Reflexion-style self-reflection, and ToT-style search.
- DeepSeek-AI, ["DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning"](https://arxiv.org/abs/2501.12948) (arXiv 2501.12948, submitted 2025-01-22, also published in *Nature* vol. 645) — self-reflection and verification reported to emerge from pure RL, without hand-coded scaffolding.
