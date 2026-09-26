# MCP Deep Dive

!!! example "Hands-on"
    Full runnable recipe: [`mcp-deep-dive/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/mcp-deep-dive) in the companion cookbook — a real, minimal "build a server, connect a client, call a tool" demo, plus two deeper repros against the official MCP Python SDK (`mcp==2.2.0`, targeting the current 2026-07-28 spec). No API key needed, pure protocol mechanics: this page is about what MCP is and how it actually works, not about calling any model.

??? abstract "TL;DR — quick revision"
    - **MCP (Model Context Protocol) is an open standard for connecting AI applications to external tools, data, and workflows** — Anthropic's own framing: *"Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems."* One MCP server, built once, works with any MCP-compatible client (Claude, ChatGPT, VS Code, Cursor, and more) — instead of one custom integration per app-times-tool pair.
    - **Architecture: a Host app runs one MCP Client per MCP Server it talks to**, each client holding a dedicated connection. Servers expose three primitives — **Tools** (actions the AI can invoke), **Resources** (data it can read), **Prompts** (reusable templates) — over **JSON-RPC 2.0**, carried on either **stdio** (a local subprocess) or **Streamable HTTP** (a remote server).
    - **MCP is stateless today**: every request carries its own protocol version and capabilities in `_meta`, so a server needs nothing remembered from earlier requests to answer the current one. A real, dated finding: `mcp==2.2.0` (the SDK that targets this exact spec) still requires a session ID **by default** — spec-compliant statelessness is real and works correctly, but is an opt-in flag (`stateless_http=True`), not the default.
    - **Multi Round-Trip Requests (MRTR) is how a server asks the client for more input mid-task** (elicitation): the server returns `InputRequiredResult`, the client retries the original request carrying the answer plus an opaque `requestState` token the server minted and must re-verify.
    - **A real repro of the SDK's own `requestState` security held on every check**: tampering with a sealed token is rejected (AEAD authentication failure), replaying a token against a different tool argument is rejected (request-binding), and — the important one — replaying one user's token as a different user is rejected (principal-binding). That last check is the real, working mitigation for the spec's own named "State Handle Hijacking" vulnerability: *"MCP servers **MUST NOT** treat possession of a state handle as authentication."*
    - **Token passthrough is explicitly forbidden, not just risky**: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server"* — a server that blindly forwards a client-supplied token downstream breaks a real OAuth security boundary and reintroduces the confused-deputy problem the rest of the spec's auth model is built to prevent.

## What MCP actually is

MCP (Model Context Protocol) is an open standard for connecting AI applications to external systems — data sources like local files and databases, tools like search engines and calculators, and workflows like reusable prompts. Anthropic's own framing is the clearest starting point: *"Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems."* The problem that solves is real and specific: without a shared standard, every AI application that wants to use a given tool has to write its own custom integration for it — an app-times-tool explosion of one-off code. Build the tool once as an MCP server, and any MCP-compatible application (Claude, ChatGPT, VS Code, Cursor, and others) can use it without new integration work.

**The architecture has three participants**, defined directly in MCP's own current documentation: *"MCP Host: The AI application that coordinates and manages one or multiple MCP clients. MCP Client: A component that maintains a connection to an MCP server and obtains context from an MCP server for the MCP host to use. MCP Server: A program that provides context to MCP clients."* Concretely: a Host (say, VS Code, or your own agent code) creates one dedicated Client for each Server it wants to talk to — one client per connection, not one client shared across many servers. A local server (reached over **stdio**, standard input/output between two processes on the same machine) typically serves a single client; a remote server (reached over **Streamable HTTP**) typically serves many clients at once.

**Servers expose three core primitives**, again in the spec's own words: *"Tools: Executable functions that AI applications can invoke to perform actions (e.g., file operations, API calls, database queries). Resources: Data sources that provide contextual information to AI applications (e.g., file contents, database records, API responses). Prompts: Reusable templates that help structure interactions with language models (e.g., system prompts, few-shot examples)."* A database-facing MCP server, for instance, might expose a `query` tool, a `schema` resource, and a prompt template for writing queries against that schema. Every one of these primitives is exchanged as a **JSON-RPC 2.0** message — a small, well-defined request/response format that's identical regardless of which transport (stdio or Streamable HTTP) carries it.

## How to use MCP: build a server, connect a client

The smallest real MCP program is one tool, one server, one client call — no cryptography, no protocol edge cases, just the mechanism working:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:build-server"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:basic-usage"
```

!!! success "A real run — one tool, discovered and called over a real connection"
    ```json
    {
      "tools": [{"name": "add", "description": "Add two numbers."}],
      "add(2, 3)": {"result": 5}
    }
    ```

    That's the whole loop: `@server.tool()` registers a plain Python function as a Tool primitive; the SDK's `Client` connects, `list_tools()` performs discovery (the client asking the server what it can do), and `call_tool()` performs execution (the client invoking one of the tools the server just advertised). Everything from here on is what happens once real systems build on top of this loop: how the protocol avoids needing to remember anything between requests, and how it protects the one piece of state a request sometimes genuinely does need to carry.

## How MCP handles requests without remembering you between them

MCP's data layer is, today, a stateless protocol: *"Every request contains all the information needed to process it, so servers infer nothing from previous requests."* Concretely, every request a client sends carries its own protocol version and capabilities in a `_meta` field — the server doesn't look anything up about "this connection," because there's nothing connection-specific to look up. A client that wants to know what a server supports sends a `server/discover` request; from there, `tools/list` performs discovery for the Tools primitive specifically, and `tools/call` performs execution:

```json
{
  "jsonrpc": "2.0", "id": 3, "method": "tools/call",
  "params": {
    "name": "weather_current",
    "arguments": {"location": "San Francisco", "units": "imperial"},
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": {"name": "example-client", "version": "1.0.0"}
    }
  }
}
```

That `_meta` block is the whole point: the server can answer this request correctly using only what's inside it, regardless of what request came before it or from which client. Nothing about who's asking is implicit or tracked server-side across calls.

## Repro 1: statelessness in practice, and a real gap between spec and default

The cookbook's `mcp==2.2.0` install explicitly targets the 2026-07-28 spec — its own SDK documentation states it was built *"to support the 2026-07-28 MCP specification (and every earlier revision)."* The obvious question, given the stateless design just described: does the SDK's actual default behavior match it? Concretely, that means sending one raw HTTP request — a plain `tools/list` call, with no session ID attached, because a stateless server shouldn't need one — first against the server's default settings, then again with its explicit `stateless_http=True` mode turned on, and comparing what actually comes back.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:running-server"
```

!!! success "A real run — raw HTTP requests against the SDK's own real server, default mode vs. explicit stateless mode"
    **Default mode** (`streamable_http_app()`, no arguments) — a raw `tools/list` POST with no prior handshake, sent directly with `httpx2` (the real HTTP client, no MCP-level session management):

    ```json
    {"jsonrpc":"2.0","id":null,"error":{"code":-32600,"message":"Bad Request: Missing session ID"}}
    ```

    In plain terms: the server refused to answer at all. It rejected the request outright, and the response carried a real `mcp-session-id` header — the server was insisting on a session ID this raw request had no way to obtain, since a stateless request is never supposed to need one. **This is the SDK's default**, in a release that explicitly targets a spec whose stateless design doesn't call for this header at all.

    **Explicit stateless mode** (`streamable_http_app(stateless_http=True)`) — two fully independent raw requests, no cookie, no session, no shared state between them:

    ```json
    {
      "call_1": {"status": 200, "has_session_id_header": false},
      "call_2": {"status": 200, "has_session_id_header": false},
      "identical_tool_lists": true
    }
    ```

    Both succeeded, neither response carried `Mcp-Session-Id`, and both independently returned the identical tool list — exactly the spec's stated behavior. And the real `Client` from the SDK's own README works unchanged against this mode: `Client(url)` connects, `list_tools()` and `call_tool("add", {"a": 2, "b": 3})` return `{"result": 5}`, with no explicit handshake code required in the calling application either way.

    **The honest finding**: spec-compliant statelessness is real, implemented, and works correctly in this SDK — it's just not what you get by not passing an argument. A team reading "our SDK targets the 2026-07-28 spec" and assuming the default configuration is therefore stateless would be wrong, as of this SDK version, checked 2026-09-23. This is a fact about the current state of the tooling, not a bug in this recipe's own code — the kind of gap between "spec says X" and "default behavior does X" that's worth checking directly rather than assuming, for any spec revision and any SDK.

## Repro 2: `requestState` — how MCP asks you for more information mid-task

Picture an ordinary web form that's missing one required field: a well-built site doesn't throw away everything you typed — it asks you for just the missing piece and lets you resubmit. That's the shape of **elicitation**, one of MCP's current client-side features: *"Allows servers to request additional information from users... Servers request user input with the `elicitation/create` method."* The mechanism carrying that request/resubmit loop is called **Multi Round-Trip Requests (MRTR)**, and it's entirely client-driven: *"Servers return an `InputRequiredResult` (`resultType: "input_required"`) whose `inputRequests` field carries the requests for the additional information needed to process the request. Clients respond with `inputResponses` on a retry of the original request providing the requested information."* The catch is exactly what you'd expect from a resubmitted form on the open internet: the server needs a way to know, on that retry, that it genuinely continues the same interrupted request rather than an attacker's unrelated request smuggled in — that's what the `requestState` token carries.

The official SDK's `mcp.server.request_state` module implements this with real production cryptography: an `AESGCMRequestStateCodec` (AES-256-GCM, authenticated encryption) sealing a claims envelope that binds the token to the exact method, target, and argument digest it was minted for, plus (when the transport is authenticated) the specific principal who requested it, plus an expiry. Reading that module's own source directly rather than guessing at its behavior:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:request-state-envelope"
```

!!! success "A real run — a checkout flow's requestState, and four real attacks against it"
    **Setup**: a real `RequestStateSecurity.ephemeral(ttl=1.0)` policy (the SDK's own random-key generator) mints a token for Alice's in-progress checkout — `method="tools/call", target="checkout", args={"cart_id": "cart_42"}, principal="user:alice"` — carrying the state `"awaiting_payment_confirmation"`.

    **1. Legitimate round-trip** — Alice retries with the identical method/target/args/principal: recovers `"awaiting_payment_confirmation"` correctly.

    **2. Tampering** — one character of the sealed token flipped before presenting it: rejected with `"seal"` — a real AES-GCM authentication tag failure, the same mechanism that protects the token against any bit-level modification, not just this specific field.

    **3. Request-binding** — the *same, untampered, validly-sealed* token, but presented against a different cart (`cart_id="cart_99_bobs_cart"` instead of `cart_42`): rejected with `"request binding"`. The token cryptographically commits to the exact arguments it was minted for — it can't be replayed against a different operation even by someone who legitimately possesses it.

    **4. Principal-binding — the state-handle-hijacking attack itself** — the same token and the same cart ID, but presented by `"user:mallory"` instead of `"user:alice"`: rejected with `"principal"`. This is a real, working test of the exact attack the spec's security document names: an attacker who somehow obtains another user's handle (a leaked log line, a shared client, a guessed value) and tries to use it to act as that user. The spec's own required mitigation: *"MCP servers **MUST NOT** treat possession of a state handle as authentication"* and *"**SHOULD** bind handles server-side to the authenticated user."* This repro shows that binding actually enforced, against the SDK's real cryptographic implementation, not a description of what it's supposed to do.

    **5. Expiry** — the identical legitimate request, retried 1.2 seconds after a 1-second TTL: rejected with `"expired"`.

    All five checks ran against the SDK's own real `AESGCMRequestStateCodec` for the cryptographic sealing and unsealing — the request-binding, principal-binding, and expiry checks are this recipe's own code, built to mirror the exact claims structure read directly from the installed SDK's `RequestStateBoundary._seal`/`_unseal` source, not a black-box guess at what it does.

## Auth: token passthrough is forbidden, not discouraged

MCP's authorization model rests on a boundary that's easy to blur in an MCP server that proxies to a third-party API: a token issued *to* the MCP server is not the same thing as a token that's safe to *forward* unmodified to a downstream service. The spec's security document is unambiguous about this specific anti-pattern: *"'Token passthrough' is an anti-pattern where an MCP server accepts tokens from an MCP client without validating that the tokens were properly issued to the MCP server and passes them through to the downstream API."* The real risk isn't abstract — it directly reintroduces the confused-deputy problem this page's `requestState` repro is adjacent to: *"the downstream API may incorrectly trust the token as if it came from the MCP server or assume the token was validated by the upstream API."* The mitigation is stated as a hard requirement, not a suggestion: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server."*

## What this means in practice

Statelessness and `requestState` aren't two unrelated features — they're the same design decision, applied twice. Because a server can't lean on the transport to remember who's talking to it between requests, anything that needs to persist across a round trip (like a paused checkout awaiting payment confirmation) has to become an explicit, self-contained value the client carries — and once that value exists outside the protected connection, it needs its own cryptographic guarantees, because nothing about the wire protects it anymore. That's exactly what `requestState`'s AEAD sealing, request-binding, and principal-binding provide. A team building an MCP server today gets this security property "for free" if they use the SDK's own `RequestStateSecurity`/`RequestStateBoundary` middleware as designed — but this page's first repro is the reason "the SDK targets the current spec" isn't the same claim as "the SDK behaves per the current spec by default": statelessness itself required an explicit flag to actually take effect.

## Interview angle

**Weak answer** to "why is MCP described as a stateless protocol?": *"Because it doesn't use sessions."* True, but incomplete in a way that misses the actual design insight — it doesn't explain what handles cross-call state instead (flows like elicitation genuinely need to persist something between requests), or why that replacement needs cryptography that a session-ID string never did.

**Strong answer**: statelessness means the server needs nothing remembered from earlier requests to answer the current one — every request is self-contained via its `_meta` field. That doesn't eliminate the need for cross-call state; it moves the responsibility for protecting that state from the transport (an opaque session ID the server tracks server-side) to an explicit, self-describing value the client carries (a `requestState` token the server must cryptographically verify on every use). That's a strictly harder problem to get right than session tracking was, which is exactly why the SDK ships a real AEAD-sealed, request-bound, principal-bound implementation rather than leaving it to each server author — and exactly why "state handle hijacking" earned its own named entry in the spec's security document: an unprotected handle is a much easier target than a server-side session table ever was.

**Follow-up to expect**: "if the SDK already targets the current spec, why does it still default to session-based mode?" Because "targets the spec" describes what the SDK is *capable* of, not what ships enabled — this page's own real test showed exactly that gap: `stateless_http=True` produces fully spec-compliant behavior, but it's opt-in, and a team that assumes an up-to-date SDK version implies up-to-date default behavior would ship session-based semantics without realizing it. The lesson generalizes past MCP: "supports the spec" and "defaults to the spec's behavior" are different claims, and only checking the actual runtime behavior — not the changelog, not the version number — tells you which one you're getting.

## Build it yourself — 30 minutes

1. Install `mcp`, build a one-tool `MCPServer` with `@server.tool()` (this page's own `add` example is enough), and connect to it with the SDK's `Client` — call `list_tools()` then `call_tool()`. This is the entire mechanism; everything else on this page is what real systems build on top of it.
2. Run that same server with `streamable_http_app()` (default) and send one raw `tools/list` POST with a plain HTTP client, no MCP client library involved. Read the actual error. Then re-run it against `streamable_http_app(stateless_http=True)` and compare the real response headers side by side — this page's own repro is exactly this comparison, just automated.
3. Read your installed SDK's own `request_state.py` (or the equivalent security module for whatever MCP SDK you use) directly, the way this recipe did, rather than trusting a summary of what it does. Find the actual claims fields it binds a token to.
4. Pick one of this page's four `requestState` attacks (tamper, request-binding, principal-binding, expiry) and reproduce it against your own server's real token-minting code, not just this recipe's toy `checkout` example — confirm your own server actually rejects what the spec requires it to reject.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: MCP Deep Dive">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro found that mcp==2.2.0 -- an SDK release whose own documentation states it targets the 2026-07-28 spec -- rejects a raw tools/list request with 'Bad Request: Missing session ID' when run with its default streamable_http_app() configuration, and only behaves statelessly once stateless_http=True is passed explicitly.",
      "question": "What's the most accurate conclusion to draw from this result?",
      "options": [
        "Targeting a spec version and defaulting to that spec's behavior are two different claims",
        "This is a bug in the recipe's own code, not a real property of the installed SDK",
        "The spec's statelessness requirement must not actually apply to the Streamable HTTP transport",
        "The SDK's version number was reported incorrectly and it does not actually target this spec"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The SDK genuinely implements spec-compliant statelessness -- it works correctly once explicitly enabled -- but ships it as an opt-in flag rather than the default, which is exactly what this page's real test demonstrated with raw HTTP requests and real response headers, not an assumption.",
        "Misattributes the finding -- the default-mode rejection came from the installed SDK's own real internal logic (streamable_http_manager.py's session handling), not from any code this recipe wrote; the recipe's code only sent a plain HTTP request and reported the real response.",
        "Contradicts the real, verbatim quote from the spec's own changelog directly: 'Remove protocol-level sessions and the Mcp-Session-Id header from the Streamable HTTP transport' -- the requirement applies to exactly this transport.",
        "Unsupported and contradicted by the SDK's own real documentation, which explicitly and directly states it targets the 2026-07-28 spec -- the discrepancy is about default configuration, not a false version claim."
      ]
    },
    {
      "scenario": "A real requestState repro showed that a token minted for Alice's checkout (method='tools/call', target='checkout', args={'cart_id': 'cart_42'}) was correctly rejected when presented with a different cart_id, even though the token itself was untampered and validly sealed.",
      "question": "What security property does this specific rejection demonstrate, distinct from the tamper-detection check?",
      "options": [
        "That Alice's account lacked sufficient permissions to access a different shopping cart",
        "That the token cryptographically commits to the exact arguments it was minted for",
        "That AES-256-GCM is a stronger algorithm than the one used for the tamper check",
        "That the token's TTL had already expired by the time the second request arrived"
      ],
      "correct": 1,
      "explanations": [
        "Not tested in this repro at all -- no authorization/permissions layer was involved; the rejection came purely from the requestState envelope's own binding check inside the demo's unseal logic, prior to any application-level permission decision.",
        "Correct. This is 'request-binding': the sealed claims envelope includes a digest of the exact arguments the token was minted for (via the 'a' claim), so presenting the same otherwise-valid token against different arguments fails that binding check -- a property distinct from (and additional to) simple tamper detection.",
        "Both checks used the identical codec (AESGCMRequestStateCodec) -- there's no second, stronger algorithm involved; this option invents a distinction the real setup doesn't have.",
        "Not what this specific test isolated -- the request-binding rejection is a separate check from the recipe's dedicated expiry test (which used a different, deliberately short-TTL scenario); this rejection specifically involved a mismatched argument, not elapsed time."
      ]
    },
    {
      "scenario": "A candidate explains the new MCP spec by saying: 'They removed sessions to make the protocol stateless, which simplifies things because servers no longer need to track any state between requests.'",
      "question": "What's the most accurate correction to this explanation?",
      "options": [
        "The change affects only the deprecated HTTP+SSE transport, not Streamable HTTP at all",
        "Sessions weren't removed -- only the name of the Mcp-Session-Id header changed in this revision",
        "Removing sessions relocated cross-call state into a protected value, not removed the need for state",
        "This is fully accurate -- protocol-level statelessness means servers genuinely never need any state"
      ],
      "correct": 2,
      "explanations": [
        "Backwards -- HTTP+SSE is being deprecated in favor of Streamable HTTP, and it's specifically Streamable HTTP that the changelog names as having its Mcp-Session-Id header removed in this revision.",
        "Contradicts the real, quoted spec changes directly -- the changelog explicitly states protocol-level sessions and the Mcp-Session-Id header are removed, not renamed; this recipe's own real test confirmed the header is genuinely absent under stateless_http=True.",
        "Correct. The real design move (and this page's own framing, backed by the requestState repro) is that removing protocol-level sessions didn't eliminate cross-request state -- it moved the responsibility for protecting that state from the transport to an explicit, self-describing, cryptographically sealed value (requestState) the client carries and the server verifies on every use.",
        "Overstates the claim -- the spec's own changelog explicitly names the replacement mechanism: 'Servers that need cross-call state use explicit, server-minted handles passed as ordinary tool arguments,' meaning the NEED for cross-call state didn't disappear, only how it's carried changed."
      ]
    },
    {
      "scenario": "An MCP server proxies requests to a third-party API. To simplify its own code, it accepts whatever bearer token the MCP client sends and forwards that exact token unmodified to the downstream API, without checking who or what it was originally issued for.",
      "question": "What does the spec's own security guidance say about this specific pattern?",
      "options": [
        "It is only a risk if the third-party API and the MCP server happen to share the same audience claim",
        "It is a required pattern for any MCP server that acts as a proxy to a third-party API",
        "It is acceptable as long as the downstream API independently validates the token itself",
        "It is explicitly forbidden -- servers must not accept tokens that were not issued for themselves"
      ],
      "correct": 3,
      "explanations": [
        "Inverts the actual risk condition -- the danger is exactly when audiences are NOT properly validated (accepting tokens regardless of their intended audience); a shared audience claim wouldn't be the triggering risk factor here, mismatched or unchecked audiences are.",
        "The opposite of what the spec recommends -- token passthrough is named as an anti-pattern specifically, not a required or even acceptable implementation choice for proxy servers.",
        "Contradicts the spec's own stated mitigation directly, which places the obligation on the MCP server itself, not on hoping a downstream service happens to catch the problem: 'MCP servers MUST NOT accept any tokens that were not explicitly issued for the MCP server' -- stated as an absolute requirement, not a conditional one.",
        "Correct. The spec is unambiguous and uses MUST NOT language: a server must validate that a token was properly issued to itself before using or forwarding it, precisely because passthrough reintroduces the confused-deputy problem and breaks real OAuth audience-validation boundaries."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Model Context Protocol, ["Introduction"](https://modelcontextprotocol.io/) — the official "what is MCP" page this page's own USB-C analogy and Host/Client/Server framing is quoted directly from.
- Model Context Protocol, ["Architecture overview"](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture) — the current data layer / transport layer split, the Tools/Resources/Prompts primitives, and the full worked `server/discover` → `tools/list` → `tools/call` example this page's own JSON snippet is drawn from.
- Model Context Protocol, ["Key Changes — 2026-07-28"](https://modelcontextprotocol.io/specification/2026-07-28/changelog) — the current spec's own changelog, including statelessness (SEP-2575) and MRTR (SEP-2322).
- Model Context Protocol, ["Security Best Practices"](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices) — State Handle Hijacking, Token Passthrough, the Confused Deputy Problem, SSRF, and the rest of the spec's named attacks and mitigations.
- [`modelcontextprotocol/python-sdk`](https://github.com/modelcontextprotocol/python-sdk) — the official SDK (`mcp==2.2.0` at the time this page was written), including `mcp/server/request_state.py`, read directly for this page's second repro.
- [Agent Security](agent-security.md) — the lethal trifecta, Rule of Two, and tool poisoning; a related but distinct set of agent-level (not protocol-level) security failure modes.
- [Tools at Scale](tools-at-scale.md) — Anthropic's code-execution-with-MCP pattern, presenting MCP servers as code APIs; complements this page's protocol-level focus with a context-efficiency angle on MCP usage.
