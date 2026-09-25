# Cloudflare's Internal MCP Platform: What Happens Once More Than One Team Uses It

[MCP and Tool Ecosystem](../mcp-tool-ecosystem.md) covers the registry, gateway, and remote-server layer of the MCP ecosystem in general terms. Cloudflare's own engineering blog describes what that layer actually looks like from the inside, once a real company moves MCP adoption from one engineering team's experiment to a company-wide platform spanning engineering, product, sales, marketing, and finance.

## The problem

Cloudflare's own stated starting point: *"Very early on, we realized that locally-hosted MCP servers were a security liability."* A locally-run MCP server, spun up ad hoc by whichever team wants to connect an agent to some internal system, has no consistent access control, no centralized audit trail, and no per-team scoping — the same server configuration effectively grants the same access to everyone who can run it, regardless of what they should be allowed to touch.

## What they built

Cloudflare's real fix was architectural, not a policy memo: centrally-hosted, remote MCP servers instead of locally-run ones, fronted by an internal "portal" model that lets each team get a differently-scoped view of the same underlying tools. The concrete example given: finance gets read-only access to a repository; engineering gets read/write access to the same repository, through the same portal, via different, real, enforced permission grants — not a shared credential with an honor-system boundary. Authentication runs through Cloudflare Access, and cost/usage control runs through Cloudflare's own AI Gateway product, rather than being left to whatever the underlying MCP server happens to expose.

## The numbers

A real, measured detail from Cloudflare's own post: their "Code Mode" approach to tool definitions reduced token consumption by **94%** on one portal serving 4 internal MCP servers and 52 tools — from roughly 9,400 tokens down to roughly 600 tokens just for tool definitions, before a single real task-relevant token is spent. Adoption itself is real and named as having spread organically past the original engineering use case into product, sales, marketing, and finance teams — not a projection, a description of what already happened internally.

!!! success "The lesson"
    "Wire up an MCP server" is a very different problem from "let MCP adoption spread past one team," and Cloudflare's own real experience shows the gap concretely: the moment a second team wants to use the same tool surface, you're not maintaining a server anymore, you're maintaining a permissions and cost-governance platform, whether you planned to build one or not. The specific architectural choice — one centrally-hosted, access-controlled portal per differently-scoped audience, rather than one server per team — is the real, load-bearing decision, and it's the same underlying principle [Identity, Governance and Compliance](../identity-governance-compliance.md) argues for at the level of individual agent identity: scoped, traceable, centrally-governed access beats a shared credential everyone happens to have equal reach through.

## Sources

- [Cloudflare Blog: "Bringing enterprise-ready MCP to the whole company"](https://blog.cloudflare.com/enterprise-mcp/)
