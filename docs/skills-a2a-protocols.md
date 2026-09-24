# Skills, A2A and Other Protocols

!!! example "Hands-on"
    Full runnable recipe: [`skills-a2a-protocols/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/skills-a2a-protocols) in the companion cookbook — a real repro of Agent Skills' progressive-disclosure claim, measured in real tokens.

??? abstract "TL;DR — quick revision"
    - **Agent Skills load in real, verified stages, not all at once.** Anthropic's own docs: *"until a Skill is triggered, only its name and description occupy context"* — roughly **~100 tokens** per skill at Level 1 (name + description, always loaded), the full body loaded only when triggered (Level 2, *"under 5k tokens"*), and bundled scripts/resources loaded only as referenced, where *"the script code itself never enters context."* A real repro confirmed this directly: loading three skills' full bodies upfront cost **4,975 real tokens** on a task needing only one; progressive disclosure cost **3,069** — a real **38.3% reduction** — and correctly never touched the two irrelevant skills at all.
    - **A2A (Agent2Agent) standardizes agent-to-agent interoperability**, not agent-to-tool — Google announced it April 2025 with 50+ launch partners, and it's since moved to Linux Foundation governance: *"A2A will remain vendor neutral, emphasize inclusive contributions and continue the protocol's focus on extensibility, security and real-world usability."* Its core discovery mechanism is the **Agent Card** — a JSON document *"describing the server's identity, capabilities, skills, service endpoint URL, and how clients should authenticate and interact with it,"* typically served at `.well-known/agent-card.json`.
    - **AG-UI standardizes the agent-to-user-interface layer** — real, verified: *"an open, lightweight, event-based protocol that standardizes how AI agents connect to user-facing applications,"* created by CopilotKit in partnership with LangChain and CrewAI. Three distinct layers, three distinct protocols: MCP (agent↔tools), A2A (agent↔agent), AG-UI (agent↔user interface).
    - **The Agent Client Protocol (ACP) is coding agents' answer to LSP** — real, verified, explicit analogy from its own docs: *"ACP solves this by providing a standardized protocol for agent-editor communication, similar to how the Language Server Protocol (LSP) standardized language server integration."* Created by Zed (August 2025), now community-governed. **AGENTS.md** is the same idea for repo-level instructions — real, verified: *"a README for agents,"* now used by *"over 60k open-source projects"* and stewarded by the Linux Foundation.

## Agent Skills: progressive disclosure, verified in real tokens

Anthropic's own Agent Skills architecture is built on a real, named principle the docs call progressive disclosure: *"Claude loads information in stages as needed, rather than consuming context upfront."* Three real, documented levels: **Level 1** is metadata — a skill's `name` and `description`, always loaded into the system prompt, at roughly **~100 tokens per skill**. **Level 2** is the skill's full instructions (its `SKILL.md` body), loaded only when a request actually matches the description — *"under 5k tokens."* **Level 3+** is bundled resources and scripts, loaded only as referenced, with a real, notable detail: for executable scripts, *"the script code itself never enters context"* — only its output does. The real, stated payoff: *"This lightweight approach means you can install many Skills without context penalty: until a Skill is triggered, only its name and description occupy context."*

## Repro: measuring the real gap between all-upfront and progressive

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/skills-a2a-protocols/skills_a2a_protocols_docs.py:skills-and-task"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/skills-a2a-protocols/skills_a2a_protocols_docs.py:two-conditions"
```

Three fictional skills, each with a real, sized `SKILL.md`-style body, on a task that's only actually relevant to one of them.

!!! success "A real run, unforced — identical task, two real conditions"
    **`all_upfront`** (all three skills' full bodies injected into the system prompt on every call): **4,975 real input tokens** — paid regardless of relevance, on a task that only needed one of the three.

    **`progressive`** (only the three short name+description pairs in the system prompt; a real `read_skill` tool loads a body only when needed): **3,069 real cumulative input tokens** across 2 real turns — and the agent correctly loaded only `pdf-forms`, never touching `invoice-processing` or `expense-report` at all.

    A real **38.3% token reduction** on just three candidate skills, one relevant. The gap isn't a fixed number — it's structural: `all_upfront`'s cost scales with the *total number of installed skills*, `progressive`'s cost scales with the number *actually used*. The more skills installed that aren't relevant to a given task, the wider this real gap gets.

## A2A: interoperability between different agents, not between an agent and its tools

Where MCP standardizes how one agent talks to its tools, A2A standardizes how *separate agents* — potentially built on different frameworks, by different vendors — discover and talk to each other. Google announced it in April 2025 with 50+ launch partners, and by June 2025 had donated it to the Linux Foundation's Agent2Agent Protocol Project, with a real, verified governance commitment: *"A2A will remain vendor neutral, emphasize inclusive contributions and continue the protocol's focus on extensibility, security and real-world usability."* Named supporting organizations include Google, AWS, Cisco, Salesforce, SAP, Microsoft, and ServiceNow.

The real, core discovery mechanism is the **Agent Card** — a JSON document, typically served at `.well-known/agent-card.json`, *"describing the server's identity, capabilities, skills, service endpoint URL, and how clients should authenticate and interact with it."* An agent looking for a collaborator fetches another agent's card the same way a browser might fetch a site's manifest — real, structured self-description rather than out-of-band documentation.

## AG-UI: the layer between an agent and the human watching it work

A real, separate concern from both MCP and A2A: once an agent is reasoning and calling tools, how does a *user-facing application* — a chat UI, a dashboard — actually render what's happening in real time? AG-UI's own real description: *"an open, lightweight, event-based protocol that standardizes how AI agents connect to user-facing applications... the general-purpose, bi-directional connection between a user-facing application and any agentic backend."* Real origin: *"AG-UI was born from CopilotKit's initial partnership with LangChain and CrewAI"* — built by an applied UI-integration team, not one of the major model or agent-framework vendors, which is itself a real, notable data point about where this particular gap was felt most acutely.

The three protocols on this page so far cleanly split by relationship: MCP is agent↔tools, A2A is agent↔agent, AG-UI is agent↔user-interface. Three different real problems, none of which substitute for the others.

## ACP and AGENTS.md: two real conventions for coding agents specifically

The **Agent Client Protocol** answers a narrower, practical question: how should a code editor talk to a coding agent, regardless of which agent or which editor? Its own docs make the analogy explicit: *"ACP solves this by providing a standardized protocol for agent-editor communication, similar to how the Language Server Protocol (LSP) standardized language server integration."* Real mechanics: *"Local agents run as sub-processes of the code editor, communicating via JSON-RPC over stdio. Remote agents can be hosted in the cloud or on separate infrastructure, communicating over HTTP or WebSocket."* Created by Zed (announced August 2025), now governed at a community org rather than solely by Zed.

**AGENTS.md** solves an adjacent but distinct problem: not how an agent talks to an editor, but how a repository tells *any* coding agent how to work in it. Real, verified framing: *"Think of AGENTS.md as a README for agents: a dedicated, predictable place to provide the context and instructions to help AI coding agents work on your project"* — build steps, test commands, conventions that would clutter a human-facing README. It's deliberately unstructured: *"AGENTS.md is just standard Markdown... the agent simply parses the text you provide"* — no required schema. Real, verified adoption: *"used by over 60k open-source projects,"* now stewarded by the Linux Foundation rather than any single company.

## Payments: one real protocol worth naming

Agentic commerce — letting an agent actually initiate a payment on a user's behalf — has its own real, emerging protocol layer. Google's **Agent Payments Protocol (AP2)** is real and verified: *"an open protocol developed with leading payments and technology companies to securely initiate and transact agent-led payments across platforms,"* built around *"Mandates — tamper-proof, cryptographically-signed digital contracts that serve as verifiable proof of a user's instructions"* — a real mechanism for an agent to prove it was actually authorized to spend, not just that it decided to.

## What this means in practice

Every protocol on this page exists because one specific, narrow gap wasn't covered by the others. Agent Skills' progressive disclosure is a *within-agent* context-efficiency mechanism — this page's own real repro shows exactly what it buys, in tokens, not just in principle. A2A is *between* agents. AG-UI is *agent-to-human*. ACP and AGENTS.md are both coding-agent-specific, but at different layers — one live/interactive (editor↔agent), one static (repo↔agent). None of these compete with each other or with MCP; a single production agent system plausibly touches all of them at once: MCP for tools, Skills for its own capability loading, A2A if it needs to hand off to another vendor's agent, AG-UI if a human is watching, and an AGENTS.md file telling any coding agent that touches the repo how to behave.

## Interview angle

**Weak answer** to "what's the difference between MCP and A2A?": *"They're both agent protocols."* This page's own material gives the precise, correct distinction: MCP standardizes an agent's connection to tools and data sources; A2A standardizes communication *between different agents themselves*, via a real, structured discovery mechanism (the Agent Card) rather than tool schemas.

**Strong answer**: name the real, verified numbers behind Agent Skills specifically, since it's the one claim on this page directly measurable — Anthropic's own documented ~100-token metadata cost per skill versus this page's own real repro showing a 38.3% real reduction from progressive disclosure on just three candidate skills. Then place the other protocols correctly by *relationship type*, not just by name: agent↔tools (MCP), agent↔agent (A2A), agent↔user-interface (AG-UI), editor↔agent (ACP), repo↔agent (AGENTS.md) — a candidate who can map each protocol to the specific relationship it standardizes is demonstrating structural understanding, not a list of buzzwords.

**Follow-up to expect**: "would you actually need all of these in one system?" A real, honest answer grounded in this page's own framing: only if the system actually has all those relationships — a single-agent CLI tool with no UI and no peer agents needs none of A2A, AG-UI, or ACP, just MCP and possibly Skills. The right question isn't "which protocols are trendy," it's "which of these five relationships does my system actually have," since each protocol solves exactly one of them and nothing else.

## Build it yourself — 30 minutes

1. Take a real skill or capability your agent already has hardcoded in its system prompt, and split it into a short name+description plus a separate, larger body — mirroring Agent Skills' own Level 1/Level 2 split.
2. Give the agent a `read_skill`-style tool to load the body on demand, and measure real `input_tokens` on a task that only needs that one skill, both with and without the split.
3. Add 2-3 more fictional skills the task doesn't need, and re-measure — this page's own repro predicts the gap should widen as more irrelevant skills are added to the all-upfront condition.
4. If you're building a multi-agent system, sketch out which of A2A, AG-UI, ACP, or AGENTS.md your system would actually need, using the relationship-type framing above — most systems need at most two or three of the five real protocols on this page.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Skills, A2A and Other Protocols">
<script type="application/json">
{
  "questions": [
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
      ]
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
      ]
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
      ]
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
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Claude Docs, ["Agent Skills Overview"](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) — the real, verified progressive-disclosure architecture this page's own repro measures directly.
- Linux Foundation, ["Linux Foundation Launches the Agent2Agent Protocol Project"](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents) and [a2a-protocol.org](https://a2a-protocol.org/latest/specification/) — A2A's real governance and the Agent Card mechanism.
- [docs.ag-ui.com](https://docs.ag-ui.com/introduction) and [github.com/ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui) — AG-UI's real protocol description and origin.
- [agentclientprotocol.com](https://agentclientprotocol.com/get-started/introduction) and [zed.dev/acp](https://zed.dev/acp) — ACP's real LSP analogy and transport mechanics.
- [agents.md](https://agents.md/) — the real, verified AGENTS.md convention and its adoption figure.
- Google Cloud, ["Announcing Agents to Payments (AP2) Protocol"](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol) — the real Mandates mechanism for agentic payments.
- [MCP Deep Dive](mcp-deep-dive.md) and [MCP and Tool Ecosystem](mcp-tool-ecosystem.md) — the agent↔tools protocol and its surrounding infrastructure, the complementary layer to every protocol on this page.
