# Intercom Fin at Anthropic: 560,000 Resolutions a Month, and the Unglamorous Work Behind It

Most public agent-deployment stories stop at a headline resolution rate. Anthropic's own account of running Intercom's Fin agent for its customer support — as told by Isabel Larrow, Anthropic's Head of AI Support — is unusual for naming the ongoing, weekly operational work behind the number, not just the number itself.

## The numbers

Real, named-source figures: Fin handles **560,000 resolutions per month** for Anthropic, at a **79% resolution rate**, with **63%** of incoming queries fully automated end to end. Fin is exposed to **80% of all incoming support queries** — the team deliberately doesn't route everything through it.

## What they actually did, not just what they bought

The real, quoted framing from Anthropic's own account: *"We knew that an AI-native strategy will always outperform an AI-added strategy."* This isn't a generic AI-first slogan in context — it's a stated design choice about building support workflows around an agent as the primary interface rather than bolting one onto an existing human-first process.

The more concrete, more instructive detail is what happens *after* deployment. Larrow's own account is explicit that the resolution rate isn't a fixed property of the tool: *"Fin isn't a 'set it and forget it' technology. It's only as good as the investment you put in."* The team runs a continuous tuning process — reviewing failed or low-confidence resolutions, updating source content, adjusting routing logic — and a single internal hack-week focused specifically on Fin tuning produced a real, measured **10% resolution rate increase**. That's a substantial jump from process work alone, with no change to the underlying model.

A real, disclosed caveat worth including rather than omitting: at least one third-party review of Intercom's broader customer base reports "real" resolution rates in the 45–53% range, against Intercom's own marketed average closer to 76% — a reminder that a headline number from one well-instrumented customer (Anthropic, reviewing and tuning weekly) doesn't necessarily generalize to every deployment of the same product.

!!! success "The lesson"
    The real, useful lesson here isn't "79% resolution rate is achievable" — it's that 79% was the result of continuous, deliberate operational investment (weekly review, a dedicated tuning hack-week, active routing-logic maintenance), not a number the agent arrived at on its own after initial setup. This is the same real point [Evaluating Agents](../evaluating-agents.md) makes about outcome-based grading requiring an actual eval loop, not a one-time launch check — a resolution rate is a live metric a team has to keep earning, not a static spec the product ships with.

## Sources

- [Fin.ai: Anthropic customer story, "Anthropic's Transformation"](https://fin.ai/customers/anthropic-transformation)
