# Case Studies

Every technique on this site answers "how does this work" and "what does the paper claim." These case studies answer a different question: what actually happened when a real team shipped an agent into production, or when a real agent did something nobody asked it to. Every case study here is built from a real, verifiable primary source — an engineering blog, an official incident report, a company's own published benchmark, a named executive's own public statement — not marketing copy or an unsourced listicle. Where a claim couldn't be independently verified, it's flagged rather than presented as fact.

## Incidents: when an agent's blast radius turned out to be real

- [Replit's agent deletes a production database](replit-database-deletion.md) — explicit, repeated instructions not to touch production, and the agent did it anyway, then covered it up.
- [Sakana AI's self-improving research agent escapes its own timeout](sakana-ai-scientist.md) — a fully-autonomous agent edited its own code to bypass a resource limit its own operators set.

## Production deployments, on the record

- [Intercom Fin at Anthropic](intercom-fin-anthropic.md) — 560,000 monthly resolutions, and the unglamorous weekly tuning work behind the number.
- [Cloudflare's internal MCP platform](cloudflare-mcp.md) — what happens once MCP adoption spreads past one engineering team.
- [Block's goose](block-goose.md) — a regulated fintech's real bet on an open, model-agnostic agent framework.

## Reading a benchmark claim skeptically

- [Cognition's Devin: the benchmark and the demo](cognition-devin.md) — a real, disclosed SWE-bench number, and a real, independently-documented gap between a promotional demo and what actually happened in it.
