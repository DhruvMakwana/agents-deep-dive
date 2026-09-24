# Guardrails and Human-in-the-Loop

!!! example "Hands-on"
    Full runnable recipe: [`guardrails-human-in-the-loop/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/guardrails-human-in-the-loop) in the companion cookbook — a real 3-way comparison of guardrail architectures on the identical task.

??? abstract "TL;DR — quick revision"
    - **Guardrails work in real, layered stages, not as one blanket switch** — a real, current framing names the stack directly: data and context guardrails, design-time governance, runtime guardrails and gateways, identity/access/security, and human-in-the-loop oversight as the final layer, not the only one.
    - **Risk tiering is a real, specific practice, not just "be careful with risky stuff"**: *"Define explicit risk tiers for agent use cases and apply proportional controls. Low-risk tasks like data summarization can run with lighter oversight. High-risk actions involving financial transactions, PII, or policy changes require multi-step verification, human approval, and comprehensive audit trails."*
    - **A real repro found the exact tradeoff risk-tiering is supposed to solve, playing out concretely**: flat-autonomous (no gates) let a $250 refund execute with zero review. Flat-gated (every action requires approval, no distinction by risk) stopped the agent after only 2 harmless lookup calls — it never even reached the risky refund step. Risk-tiered gating (only actions above a real threshold require approval) let both safe lookups proceed immediately and correctly blocked only the $250 refund — confirmed on two separate full runs.
    - **The real, sharper finding isn't "flat gating adds friction" — it's that undifferentiated gating can stall an agent before it even reaches the point where review matters.** The agent under flat-gating didn't slowly grind through extra approval steps; it stopped making progress entirely, two calls in, having never attempted the one action that actually needed a human.

## Guardrails are a stack, and human review is the last layer, not the only one

The instinct to protect an agent with "add a human approval step" treats guardrails as a single binary switch — either a human checks everything, or nothing is checked. Real, current guardrail practice describes something more structured: a real five-layer stack — data and context guardrails, design-time governance, runtime guardrails and gateways, identity/access/security controls, and human-in-the-loop oversight as the final layer. Human review sits at the *top* of that stack, catching what the earlier, cheaper, automated layers didn't already handle — not standing in for all of them.

This matters because it changes what "guardrails" actually means in practice: most of the real protective work happens *before* anything reaches a human, in scoped permissions, runtime checks, and automated gating — human-in-the-loop is specifically for the residual risk those earlier layers can't resolve on their own, which is precisely why it needs to be reserved for the cases that actually warrant it, not applied uniformly to everything.

## Risk tiering: proportional controls, not uniform ones

The real, specific practice this page's repro tests directly: *"Define explicit risk tiers for agent use cases and apply proportional controls. Low-risk tasks like data summarization can run with lighter oversight. High-risk actions involving financial transactions, PII, or policy changes require multi-step verification, human approval, and comprehensive audit trails."* This is a real, concrete design principle, not a vague call for caution — it says explicitly that *not* gating low-risk work is part of doing this correctly, not a corner being cut.

A closely related, real, and equally concrete guideline: organizations should *"require human approval on any irreversible action like moving money or deleting data"* — naming reversibility, not just dollar amount or category, as a real criterion for what belongs in the highest risk tier.

## Repro: flat-autonomous vs. flat-gated vs. risk-tiered, on the identical task

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/guardrails-human-in-the-loop/guardrails_human_in_the_loop_docs.py:refund-scenario"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/guardrails-human-in-the-loop/guardrails_human_in_the_loop_docs.py:risk-tier"
```

Risk here is computed per real tool call, from the actual arguments — not just which tool got named. A $10 `issue_refund` call and a $250 `issue_refund` call are the identical tool with very different real risk, exactly matching the "specific action... weighed against its current context" framing real risk-based gating uses.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/guardrails-human-in-the-loop/guardrails_human_in_the_loop_docs.py:gate-loop"
```

!!! success "A real run — the identical refund task, three real gating conditions, confirmed on two separate full runs"
    **Task**: *"Please process a refund for order #A200. Look up the order, check refund eligibility, and if eligible, issue a full refund of $250."* Three real tool calls of genuinely different risk: two lookups (low risk) and one $250 refund (high risk — above a real $50 human-approval threshold).

    **Flat-autonomous** (no gates at all): all 3 real calls executed immediately, including the $250 refund. `risky_refund_executed_without_review: true` — the high-risk action went through with zero review, indistinguishable in the raw trace from the two harmless lookups around it.

    **Flat-gated** (every call requires approval, no distinction by risk): the real trace shows only **2 total calls** — both harmless lookups, both immediately returned `PENDING_HUMAN_APPROVAL`. The agent never even attempted the refund call. `gated_count: 2`, `total_calls: 2`.

    **Tiered** (only the human-required tier gates): all 3 real calls were attempted. `gated_count: 1` — only the $250 refund. `risky_refund_executed_without_review: false`. The agent completed everything it safely could, and stopped exactly at the one point that genuinely needed human review.

    The sharper finding is in the flat-gated condition's call count, not just its gate count: undifferentiated gating didn't produce a slower, more cautious version of the same task — it produced an agent that made real progress on nothing beyond two harmless lookups, having never reached the point where gating was actually protecting anything.

## What this means in practice

The three real conditions map cleanly onto the actual design space guardrails occupy: flat-autonomous is functional but unsafe (the risky action goes through unreviewed, indistinguishable from safe ones in a raw success/failure view); flat-gated is safe but non-functional (protection is real, but so is the cost — the agent stalls before doing anything the review layer was meant to protect); risk-tiered is the only condition that was both, in this real run — safe (the risky action was caught) and functional (the agent still made real progress on everything that didn't need review). This is the concrete, measured version of what "proportional controls" actually buys: not lower total safety than uniform gating, but the same safety on the risky action while preserving real throughput on the work that was never risky to begin with.

## Interview angle

**Weak answer** to "how would you add human oversight to an autonomous agent?": *"Have a human approve every action before it executes."* This page's own real repro is a direct, measured counterexample — the flat-gated condition applied exactly this policy and the agent never even reached the one action that genuinely needed a human's attention, because it stalled on two harmless lookups first.

**Strong answer**: define real risk tiers, computed from the actual action and its actual parameters (not just which tool was called), and gate proportionally — matching the real, cited guideline that *"low-risk tasks... can run with lighter oversight"* while *"high-risk actions... require multi-step verification, human approval, and comprehensive audit trails."* This page's own repro demonstrates the concrete payoff directly: the same real task, with the same real risky action, was both correctly caught and correctly *not* obstructed on everything else — a result flat gating, run on the identical task, couldn't produce, because it never got far enough to matter.

**Follow-up to expect**: "how would you decide the actual threshold — why $50, why not $500?" That's a real business and risk-tolerance decision, not something an engineering pattern determines on its own — but the reversibility criterion from real guidance ("any irreversible action like moving money or deleting data") gives a defensible starting heuristic independent of a specific dollar figure: gate what can't be cleanly undone, regardless of its apparent size, and calibrate the size-based threshold from there based on your organization's actual tolerance for a wrong autonomous action at that scale.

## Build it yourself — 30 minutes

1. Pick a real task with at least one clearly low-risk action and one clearly high-risk action (a lookup and a payment, a read and a delete, a draft and a send). Run it fully autonomous first and confirm the risky action really does execute unreviewed.
2. Add a flat gate: every tool call requires approval, with no distinction by risk. Run the identical task and check not just how many calls got gated, but how far the agent actually got — this page's own repro found the real, sharper cost here.
3. Add real risk-tiering: compute risk from the actual arguments of each call (an amount, a scope, a target), not just the tool name, and gate only above a real threshold. Run the identical task a third time and compare all three real outcomes side by side.
4. If your own repro shows the tiered condition reaching further than the flat-gated one while still catching the risky action, that's the real, working version of "proportional controls" — not a theoretical benefit, a measured one on your own task.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Guardrails and Human-in-the-Loop">
<script type="application/json">
{
  "questions": [
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
      ]
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
      ]
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
      ]
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
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Atlan, ["AI Agent Risks & Guardrails: 2026 Enterprise Security Guide"](https://atlan.com/know/ai-agent-risks-guardrails/) — the real five-layer guardrail framework and the risk-tiering guidance this page's repro tests directly.
- OWASP Top 10 for Agentic Applications (2026) — human approval requirements for irreversible, high-impact actions.
- [Agent Security](agent-security.md) — the lethal trifecta and Rule of Two, a closely related but distinct framing of layered, structural risk controls.
- [Coding Agent Products and Configuration](coding-agent-config.md) — the real hooks-vs-prompted-rule repro, the binary predecessor to this page's three-tier comparison.
- [Harness Engineering](harness-engineering.md) — the progress-file and self-verification patterns, complementary harness-level controls to this page's action-gating focus.
