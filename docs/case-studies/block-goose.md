# Block's goose: A Regulated Fintech Bets on an Open, Model-Agnostic Agent

Block (formerly Square) is a regulated financial-services company — not the profile most people picture betting early on an open-source AI agent framework. In January 2025, Block's Open Source Program Office did exactly that, releasing "codename goose" publicly rather than building an internal, closed tool.

## What they built

goose is a real, open-source (Apache-2.0) AI agent framework, built in Rust, designed to be model-agnostic rather than tied to one provider, and connecting to external capabilities through MCP — Block's own choice to build on an open, standardized tool-connection protocol rather than a proprietary integration layer. As of this writing, the project has drawn **54.6k GitHub stars** and has since moved to community governance under the Agentic AI Foundation at the Linux Foundation — the same governance body [Skills, A2A and Other Protocols](../skills-a2a-protocols.md) covers for AGENTS.md, a pattern worth noticing: several agent-ecosystem projects that started as one company's internal tool have moved to neutral, cross-vendor governance once adoption grew past that one company.

## Why, in Block's own words

Block's own open-source lead, Manik Surtani, framed the decision as being about more than developer goodwill: open source is the *"best way to build software,"* and — the more specific, company-relevant point — *"even more important for a financial services company to be building on open principles and open standards."* For a regulated fintech, the more usual instinct is to keep infrastructure proprietary and tightly controlled; Block's own stated reasoning runs the other way, treating openness and auditability as the more defensible choice given the regulatory stakes, not a luxury a less-regulated company could better afford.

A real, necessary caveat: several secondary sources circulate claims that goose "replaced 40% of Block's engineering team" or "scaled to 60% of the company" — these numbers could not be verified against any primary Block source and read as unsourced content-farm amplification. They're worth flagging explicitly as *not* usable rather than silently repeating them, the same standard this site holds every other claim to.

!!! success "The lesson"
    A model-agnostic, MCP-native architecture is a real, deliberate hedge against vendor lock-in specifically — goose's own design means Block isn't betting its internal tooling on any single model provider's roadmap, which matters more, not less, for a company operating under regulatory scrutiny where switching infrastructure later isn't a quick decision. The broader, more general lesson: when a framework choice claims real numbers, check whether they trace to the company itself or to secondary "AI news" amplification — this case study's own real, verified facts (the star count, the Linux Foundation governance move, Surtani's own quote) are worth far more than the unverifiable adoption-percentage claims that get repeated alongside them.

## Sources

- [Block: "Block Open Source Introduces codename goose"](https://block.xyz/inside/block-open-source-introduces-codename-goose)
- [InfoQ: "Block Releases codename goose"](https://www.infoq.com/news/2025/02/codename-goose/)
- [github.com/block/goose](https://github.com/block/goose)
