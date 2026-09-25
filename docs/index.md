---
title: Home
---

# Agents, Deep Dive

A complete, working reference on LLM agents — 36 pages from the workflow-vs-agent distinction up through tool use, context and memory, multi-agent systems, production engineering, security and governance, and training agents with reinforcement learning. A companion to [RAG, Deep Dive](https://dhruvmakwana.github.io/rag-deep-dive/).

Every page follows the same standard: real, tested code where a technique can be demoed (not pseudocode), current research rather than recycled tutorial content, and an honest report of what actually happened when the code ran — including the times it surfaced something genuinely surprising, or didn't work at all. Where a claim is measured, the numbers on the page are from a real run, not an estimate. Fast-moving claims (a library version, a protocol spec, a benchmark score) are stamped with the date they were checked, because this field moves within months.

!!! tip "How to use this site"
    Each page is self-contained: problem it solves → mechanism → worked example → trade-offs → an interview-ready one-liner. Most pages end with a scenario-based quiz to check whether the concept actually stuck, and an interview angle naming what a weak versus strong answer looks like. New here? Start with [**What Is an Agent?**](what-is-an-agent.md). Prepping for an interview? Jump straight to the [**Revision**](revision.md) page — every page's TL;DR, quiz, and flashcards in one place. Every page also links to a real, runnable recipe in the [companion cookbook](https://github.com/DhruvMakwana/agents-cookbook).

## Foundations

- [**What Is an Agent?**](what-is-an-agent.md) — the workflow-vs-agent distinction, worked through a real example, with contested definitions named rather than papered over.
- [**The Agent Loop From Scratch**](agent-loop-from-scratch.md) — tool-calling mechanics, stop conditions, and a provider-neutral loop, built by hand.
- [**Workflow Patterns**](workflow-patterns.md) — chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer.
- [**Reasoning Paradigms**](reasoning-paradigms.md) — ReAct, Reflexion, Plan-and-Solve, ReWOO, LLM Compiler, ToT and LATS.
- [**Choosing a Framework**](choosing-a-framework.md) — LangChain/LangGraph, OpenAI Agents SDK, Pydantic AI, Claude Agent SDK, and the own-the-loop debate.

## Tools and Context

- [**Tool Design**](tool-design.md) — schema as prompt, errors as observations, the agent-computer interface.
- [**Context Engineering**](context-engineering.md) — write, select, compress, isolate; context rot and the four context failures.
- [**Tools at Scale**](tools-at-scale.md) — tool retrieval, programmatic tool calling, code execution over MCP.
- [**MCP Deep Dive**](mcp-deep-dive.md) — the current spec, transports, auth, security, servers and clients.
- [**MCP and Tool Ecosystem**](mcp-tool-ecosystem.md) — registries, gateways, remote servers, browser infrastructure, tool platforms.
- [**Skills, A2A and Other Protocols**](skills-a2a-protocols.md) — Agent Skills and progressive disclosure, A2A, AG-UI, ACP, AGENTS.md.
- [**KV-Cache Economics**](kv-cache-economics.md) — cache-stable prompts, tool masking versus removal.
- [**Memory Architectures**](memory-architectures.md) — the four-type taxonomy, MemGPT, Mem0, graph memory, memory poisoning.
- [**Models for Agents**](models-for-agents.md) — frontier, small and local models, tool-calling reliability, routing.

## Systems

- [**Planning and Decomposition**](planning-and-decomposition.md) — plan-and-execute, replanning triggers, the granularity rule.
- [**Multi-Agent Systems**](multi-agent-systems.md) — real patterns, the Cognition-vs-Anthropic debate, MAST failure modes.
- [**Harness Engineering**](harness-engineering.md) — harness anatomy, long-horizon agents, resets versus compaction.
- [**Coding Agents: Mechanisms**](coding-agents-mechanisms.md) — the agent-computer interface, sandboxes, review loops.
- [**Coding Agent Products and Configuration**](coding-agent-config.md) — Claude Code, Codex, Cursor, and how they're actually configured.
- [**Deep Research Agents**](deep-research-agents.md) — Anthropic's research system, subagent division of labor, STORM, BrowseComp.
- [**Computer-Use and Browser Agents**](computer-use-browser-agents.md) — grounding, OSWorld, UI-TARS, and real browser-agent safety incidents.
- [**Personal and Always-On Agents**](personal-always-on-agents.md) — OpenClaw, Hermes Agent, and the changed blast radius.

## Production

- [**Evaluating Agents**](evaluating-agents.md) — outcome versus trajectory, pass@k versus pass^k, infrastructure noise.
- [**Benchmark Atlas**](benchmark-atlas.md) — what SWE-bench, tau-bench, GAIA, OSWorld and more actually measure.
- [**Observability and Debugging**](observability-debugging.md) — traces, OpenTelemetry GenAI conventions, the 3 a.m. debugging question.
- [**Agent Security**](agent-security.md) — the lethal trifecta, indirect injection, CaMeL, the Rule of Two.
- [**Guardrails and Human-in-the-Loop**](guardrails-human-in-the-loop.md) — layered guardrails, risk-tiered approval gates.
- [**Identity, Governance and Compliance**](identity-governance-compliance.md) — agent identity, OWASP's agentic list, NIST, the EU AI Act, MITRE ATLAS.
- [**Sandboxes and Permissions**](sandboxes-permissions.md) — the isolation ladder, credential proxying, theft versus misuse.
- [**Durable Execution**](durable-execution.md) — event logs, replay, idempotency.
- [**Cost and Latency**](cost-latency.md) — routing, caching, step budgets, quadratic transcript growth.

## Training

- [**Training Agents: Reward and Credit**](training-agents.md) — outcome versus process reward, GRPO and DAPO, DPO pair construction.
- [**RL for Search and Tool Agents**](rl-search-tool-agents.md) — Search-R1, ToolRL, ReTool, and RAGEN's failure modes.
- [**Data and Environments**](data-environments.md) — trajectory synthesis, SWE-Gym, SWE-smith, and how verifiers work.
- [**Self-Improving Agents**](self-improving-agents.md) — Reflexion, ACE, Voyager, the Darwin Gödel Machine, AlphaEvolve.

## Case Studies

- [**Case Studies**](case-studies/index.md) — real, primary-sourced stories of agents in production and in incidents: a database deleted despite explicit instructions not to, a self-improving agent editing its own resource limit, a real 560,000-resolution-a-month support deployment, and more.

## Tutorials & Code

- [**Tutorials & Code**](tutorials/index.md) — end-to-end builds combining several pages into one real, production-shaped project, starting with a support agent that ties together tool design, memory, and risk-tiered guardrails.

## Interview prep

- [**Interview Playbook**](interview-playbook.md) — an evidence-graded synthesis of real interview reports, company-published process changes, and job-description data, not a recycled question list.
- [**Revision**](revision.md) — every page's TL;DR, quiz, and flashcards, merged into one page for a fast pass before an interview.
