# MCP and Tool Ecosystem

!!! example "Hands-on"
    Full runnable recipe: [`mcp-tool-ecosystem/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/mcp-tool-ecosystem) in the companion cookbook — a real repro of registry-driven server discovery against the live official MCP registry, not a mock.

??? abstract "TL;DR — quick revision"
    - **A real, live, official MCP registry exists and is publicly callable right now**: `registry.modelcontextprotocol.io`, launched in preview 2025-09-08, backed by Anthropic, GitHub, PulseMCP, Block, and others — described in its own launch post as "an open catalog and API for publicly available MCP servers." Unauthenticated reads via `GET /v0/servers?search=...`. A real repro called this exact live API.
    - **The real repro found the registry's search is closer to keyword/brand matching than semantic search.** Natural-language capability queries (`"browser automation form filling"`, `"payment charges status"`) reliably returned **zero results**; single-keyword, brand-style queries (`"Stripe"`, `"playwright"`, `"browser"`) worked. An agent had to burn 6 real search attempts, 4 of them dead ends, before finding a usable browser-automation server.
    - **A real, separately-confirmed finding: even a genuinely popular server can be absent.** Microsoft's own official `playwright-mcp` (37.5k+ GitHub stars) does not currently appear in the registry under any query tried, including a direct `search=microsoft`. As a real preview-stage product, the registry's population is honest but incomplete.
    - **Gateways, remote-hosted servers, and tool platforms are the real, separate layers that exist because raw discovery and raw protocol aren't enough on their own**: gateways add auth/rate-limiting/routing in front of many servers; companies like Stripe, GitHub, Notion, and Cloudflare now host their own official *remote* MCP servers (reachable over HTTP, not run locally); and platforms like Composio aggregate hundreds of third-party integrations behind one MCP interface — each solving a real, different gap the base protocol and the registry alone leave open.

## The registry: real, live, and honestly incomplete

[MCP Deep Dive](mcp-deep-dive.md) covered the protocol itself — the 2026-07-28 spec, transports, auth, servers and clients. This page covers the layer built *around* that protocol: how an agent actually finds, reaches, and manages access to the servers implementing it, once there are thousands of them.

The most direct answer to "how does discovery work" is the real, official MCP registry — `registry.modelcontextprotocol.io`, in preview since 2025-09-08, its own description calling it "an open catalog and API for publicly available MCP servers." Reads are unauthenticated: `GET /v0/servers?search=<query>&limit=<n>` returns real, currently-listed servers with their name, description, and remote endpoint URLs.

## Repro: what an agent actually finds, searching the real thing

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-tool-ecosystem/mcp_tool_ecosystem_docs.py:registry-search"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-tool-ecosystem/mcp_tool_ecosystem_docs.py:discovery-tasks"
```

An agent with exactly one tool — `search_mcp_registry`, wired to the real live API above — given a real task, has to decide what to search for, call the real registry, and pick a real result.

!!! success "A real run, unforced — against the live registry and Claude Sonnet 5"
    **Payment task**: first real query, `"payment charges status"` — **0 results**. Adjusted to `"Stripe"` — **5 real results**. Picked the official `com.stripe/mcp`, reasoning explicitly that a third-party hosted wrapper it also found (`eu.nordicmcp/stripe`) added "a trust/dependency layer you don't need when the official one exists" — a real, security-aware distinction the agent drew on its own, unprompted.

    **Browser automation task**: **6 real search queries** before a usable pick. Four of six — `"browser automation form filling"`, `"web page interaction click form"`, `"Microsoft official playwright mcp"`, `"stagehand browserbase automate website"` — all **0 results**. Only `"playwright"`, `"puppeteer"`, and `"browser"` returned anything. Final pick: `ai.smithery/browserbasehq-mcp-browserbase`, reasoned as a good fit because it uses Stagehand's natural-language action model rather than raw selector-based automation.

    **A separate, direct check** (not from the agent's own search): does Microsoft's own official `playwright-mcp` — a genuinely popular server — actually appear in the registry? A direct `search=microsoft` query, and every variant the agent itself tried, returned **nothing matching it**. The agent's pick wasn't a reasoning failure; the registry genuinely doesn't list that server yet, under any of the terms tried.

The real, combined finding: brand/product-name queries work, natural-language capability descriptions mostly don't, and the registry's real population — as an honest preview-stage product — has real gaps, including for well-known servers. All three are current, verifiable facts about the registry today, not permanent properties of the concept.

## Gateways: the layer the raw protocol and registry don't provide

Neither the MCP spec nor the registry says anything about auth policy, rate limiting, or routing across many servers at once — that's what real, named gateway products add. Current examples include Bifrost, TrueFoundry's MCP Gateway, Docker's and Microsoft's own gateway offerings, and open-source options like ToolHive and agentgateway. Their real, shared job: sit in front of multiple MCP servers and add authN/authZ, per-tool or per-team rate limiting, audit logging, and traffic routing — positioned as the "enterprise readiness" layer the base protocol deliberately leaves out. (Specific vendor-published latency numbers — sub-millisecond claims are common — are vendor benchmarks, not independently reproduced here; treat them as marketing claims to verify yourself before relying on them.)

## Remote servers: the API moved from stdio to HTTP

A real, significant shift already underway: major companies now run their *own* official MCP servers as hosted, remote endpoints — reachable over Streamable HTTP, not spawned as a local stdio process. Real, currently-live examples, each independently confirmed reachable: GitHub's `api.githubcopilot.com/mcp/` (GA'd 2025-09-04), Stripe's `mcp.stripe.com` (dual OAuth/Agent-API-key auth; from 2026-10-31 it stops accepting non-Agent-tagged secret keys entirely), Notion's `mcp.notion.com/mcp` (its older self-hosted stdio server is no longer actively maintained — Notion recommends the hosted endpoint for new integrations), and Cloudflare's own suite of over a dozen scoped remote servers (`docs.mcp.cloudflare.com`, `bindings.mcp.cloudflare.com`, `browser.mcp.cloudflare.com`, and more), each on its own subdomain rather than one monolithic server. The real, shared implication: an agent's "tool access" increasingly means holding a valid OAuth token or Agent API key for someone else's hosted endpoint, not managing a local process at all.

## Browser infrastructure: two real, different philosophies

Two real, named approaches to browser automation over MCP, both confirmed in this page's own repro results: Microsoft's `playwright-mcp` runs locally and drives Playwright's accessibility tree directly — real, verified: "no vision models needed, operates purely on structured data" — with tool names like `browser_click`, `browser_snapshot`, and `browser_fill_form`. Browserbase's MCP server instead runs actual browser sessions in Browserbase's managed cloud, built on Stagehand, which resolves natural-language actions ("click the login button") to real DOM interactions rather than requiring explicit selectors up front — the same server this page's own repro picked for its browser task, precisely because of that higher-level action model.

## Tool platforms: aggregation as a product

A real, separate category from both gateways and the registry: platforms like Composio that aggregate large numbers of third-party integrations behind one MCP interface, so an agent developer integrates once rather than once per API. Composio's own marketing states figures like 1,000+ toolkits, 3,000+ tools, and a separate "500+ managed MCP servers" claim for its hosted gateway product — self-reported numbers that differ across its own pages depending on which product tier is being described, worth verifying directly rather than citing as one settled figure. The real, shared value proposition across this whole category: trade direct control over which exact server you're talking to for a single integration surface and (usually) managed auth across many providers at once.

## What this means in practice

Four real, separate layers, solving four real, separate problems: the **registry** answers "what servers exist" (imperfectly, today); **gateways** answer "how do I control and observe access across many of them"; **remote servers** answer "where does the code actually run" (increasingly: not on your machine); **tool platforms** answer "how do I avoid integrating with each one individually." None of them substitute for the others — a well-known, unlisted server (like this page's own Playwright finding) is a registry gap a gateway or platform doesn't fix; a rate-limit or audit requirement is a gateway problem a bigger registry doesn't fix either.

## Interview angle

**Weak answer** to "how would an agent find the right MCP server for a task?": *"Search the MCP registry for what it needs."* This page's own real repro is a direct, measured complication: half the agent's own natural-language search attempts returned zero results, and the one server it might have most wanted — Microsoft's own popular Playwright server — wasn't even findable, through no fault of the agent's search strategy.

**Strong answer**: name the real, current limitation precisely — the registry's search behaves like keyword/brand matching, not semantic search, so a good discovery strategy needs either brand-aware query generation, a curated allow-list layered on top (what tool platforms and gateways effectively provide), or fallback to direct knowledge of major vendors' known remote endpoints. This page's own repro demonstrates the gap directly rather than asserting it: real dead-end queries, a real successful recovery via brand-name search, and a real, independently-confirmed case of a popular server simply not being listed.

**Follow-up to expect**: "would you trust an agent to autonomously pick and connect to a server it found in the registry?" A real, honest answer grounded in this page's own repro: the agent's own reasoning (favoring `com.stripe/mcp`'s official namespace over a third-party wrapper specifically to avoid an extra intermediary near payment data) shows the *right instinct* — but instinct isn't a security boundary. A real production system would still want a gateway or an explicit allow-list enforcing which servers are actually reachable, rather than relying on the agent's judgment about trust signals in a registry listing alone.

## Build it yourself — 30 minutes

1. Hit the real registry yourself: `curl "https://registry.modelcontextprotocol.io/v0/servers?search=<your term>&limit=5"` — try both a natural-language phrase and a brand/product name for the same real capability, and compare result counts.
2. Wire the real endpoint into a single-tool agent (as in this page's own repro) and give it 2-3 real tasks needing different capabilities. Log every search query it tries, not just its final answer.
3. Check specifically for zero-result queries and note whether the agent adjusted its search strategy on its own, the way this page's own real run did (natural language → brand name).
4. Pick one well-known MCP server you already know exists (by GitHub stars or word of mouth) and check directly whether the registry actually lists it. If it doesn't, that's a real, current registry gap your own agent would hit too.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: MCP and Tool Ecosystem">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro against the live official MCP registry found that natural-language queries like 'browser automation form filling' and 'payment charges status' returned zero results, while brand-name queries like 'Stripe' and 'playwright' returned real matches.",
      "question": "What is the most precise real conclusion this pattern supports about the registry's current search behavior?",
      "options": [
        "The registry's search behaves closer to keyword or brand matching than to semantic search",
        "The registry's search is broken and should not be used by any agent for any purpose",
        "The registry only indexes servers made by large, well-known companies and excludes smaller developers",
        "Natural-language queries always fail while single-word queries always succeed, without exception"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, repeated pattern -- natural-language capability descriptions failing, specific product/brand names succeeding -- is precisely the signature of keyword/substring-style matching rather than a semantic search that understands intent or synonyms.",
        "Overstates the finding -- the repro shows a specific, characterizable LIMITATION (keyword-style matching), not that the search is non-functional; brand-name queries worked correctly and returned real, relevant results.",
        "Not what the evidence shows -- the successful queries were about MATCHING TERMS (brand names), not about company size; nothing in the repro tested or claims a bias toward large companies specifically.",
        "Overgeneralizes into an absolute rule -- the repro reports what happened on the SPECIFIC queries tried, not a proven universal law; the page is careful to characterize this as an observed pattern on real trials, not an exceptionless guarantee."
      ]
    },
    {
      "scenario": "A direct, separate check of the real MCP registry found that Microsoft's own official playwright-mcp server -- a genuinely popular project with 37.5k+ GitHub stars -- does not appear in the registry under any search term tried, including a direct search for 'microsoft'.",
      "question": "What is the most accurate interpretation of what this specific finding demonstrates?",
      "options": [
        "Microsoft's playwright-mcp is not a real or trustworthy MCP server, which is why it's absent",
        "The registry is a fully complete and authoritative source of every MCP server that currently exists",
        "The agent's search strategy in the repro was flawed, which is why it failed to find this server",
        "The registry, as a real preview-stage product, has genuine gaps even for well-known servers today"
      ],
      "correct": 3,
      "explanations": [
        "Not supported and not the claim -- popularity (37.5k+ stars) is independent evidence the server is real and widely used; its absence from the registry says something about the REGISTRY's completeness, not the server's legitimacy.",
        "Directly contradicted -- this is exactly the finding that disproves completeness; a well-known, popular server missing entirely is direct evidence against the registry being authoritative or exhaustive at this stage.",
        "Not supported -- the check that found this was a DIRECT query for 'microsoft' itself, not a flawed derived query; the absence held regardless of search phrasing, pointing to the registry's own data rather than a searcher's mistake.",
        "Correct. The registry is explicitly described as a real, preview-stage product (launched 2025-09-08), and this finding is direct, confirmed evidence of a real current gap in its population -- consistent with 'honest but incomplete,' not a flaw in how anyone searched it."
      ]
    },
    {
      "scenario": "This page distinguishes four real, separate layers in the MCP ecosystem: the registry (discovery), gateways (auth/rate-limiting/routing), remote servers (where code runs), and tool platforms (integration aggregation).",
      "question": "Based on the page's own framing, what happens when a well-known server is missing from the registry (as with the Playwright example), and a team ALSO adds a gateway product in front of their MCP servers?",
      "options": [
        "The gateway automatically expands the registry's listings, so the missing server becomes discoverable through the registry itself",
        "The registry gap remains unaffected by the gateway, since gateways address access control, not discovery completeness",
        "Adding a gateway makes the registry unnecessary, since gateways independently solve the discovery problem too",
        "The gateway and the registry are two names for the same underlying system, so fixing one automatically fixes the other"
      ],
      "correct": 1,
      "explanations": [
        "Not how these layers relate per the page -- gateways sit in front of servers for auth/routing/rate-limiting; they don't write to or expand the registry's own listings, which is a separate real system entirely.",
        "Correct. The page explicitly frames these as solving DIFFERENT real problems -- 'a well-known, unlisted server... is a registry gap a gateway or platform doesn't fix' -- a gateway operates on servers you already know about and have configured, not on expanding what's discoverable in the registry.",
        "Overstates a gateway's role -- the page frames gateways as an access-control/routing layer over KNOWN servers, not as a discovery mechanism that could substitute for the registry's cataloging function.",
        "Contradicted directly -- the page treats these as four distinct, separately real layers solving different problems, not interchangeable names for the same system."
      ]
    },
    {
      "scenario": "In the real repro's payment task, the agent chose the official com.stripe/mcp server over a third-party hosted wrapper (eu.nordicmcp/stripe) that also appeared in the real search results and offered a broader toolset (18 tools).",
      "question": "What was the real, stated reasoning behind this choice, according to the agent's own actual output?",
      "options": [
        "The third-party wrapper was reported as non-functional or broken in the real search results returned",
        "The official server was the only one of the two that had any remote endpoint URL listed at all",
        "The official server avoids the added trust and dependency cost of a nearby third-party intermediary",
        "The third-party wrapper had fewer tools available, making it strictly less capable for the task"
      ],
      "correct": 2,
      "explanations": [
        "Not indicated anywhere -- both real servers appeared as valid, listed entries in the real search results; nothing in the repro's data suggests the third-party option was broken or non-functional.",
        "Factually reversed from what's described -- the third-party wrapper DID have a real remote endpoint listed (a token-based URL); the distinguishing factor wasn't presence/absence of an endpoint.",
        "Correct. The agent's own real output explicitly reasoned about trust: choosing the first-party server specifically to avoid 'a trust/dependency layer you don't need when the official one exists' -- a security-aware judgment about intermediary risk near sensitive payment data, not a capability comparison.",
        "Backwards -- the third-party wrapper was described as offering a BROADER toolset (18 tools including reporting), not a narrower one; capability breadth wasn't the deciding factor and if anything favored the wrapper, which the agent still declined in favor of trust."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- MCP Registry launch post, ["Introducing the MCP Registry"](https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/) and [github.com/modelcontextprotocol/registry](https://github.com/modelcontextprotocol/registry) — the real, verified registry this page's own repro calls directly.
- [MCP Deep Dive](mcp-deep-dive.md) — the underlying protocol spec, transports, auth, and security this page's ecosystem layer builds on top of.
- [Tools at Scale](tools-at-scale.md) — tool retrieval and tool search over a fixed, pre-configured library, the complementary problem to this page's dynamic, registry-driven discovery.
- [github.com/microsoft/playwright-mcp](https://github.com/microsoft/playwright-mcp) — the real, popular server this page's own repro confirmed is currently unlisted in the official registry.
- [Agent Security](agent-security.md) — the lethal trifecta and Rule of Two, relevant to the real trust judgment this page's repro shows an agent making on its own about a third-party payment-data intermediary.
