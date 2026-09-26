# MCP Deep Dive

!!! example "Hands-on"
    Full runnable recipe: [`mcp-deep-dive/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/mcp-deep-dive) in the companion cookbook — five real, progressively-built demos against the official MCP Python SDK (`mcp==2.2.0`, targeting the current 2026-07-28 spec): a first server and client, multiple tools with their real schemas, the same server reached over both transports (stdio and Streamable HTTP), and two deeper repros into how MCP behaves under the hood. No API key needed anywhere on this page — pure protocol mechanics, not a model call.

??? abstract "TL;DR — quick revision"
    - **MCP (Model Context Protocol) is an open standard for connecting AI applications to external tools, data, and workflows** — Anthropic's own framing: *"Think of MCP like a USB-C port for AI applications."* One MCP server, built once, works with any MCP-compatible client (Claude, ChatGPT, VS Code, Cursor, and more).
    - **Three pieces, defined against each other**: a **Host** (the AI application, e.g. your own script, Claude Desktop, VS Code) creates a **Client** for each **Server** it wants to talk to; the Client is a dedicated connection, the Server is the program on the other end that actually owns the tools.
    - **A Tool's schema is generated automatically from ordinary code**: `@server.tool()` on a typed Python function produces a real JSON Schema (`required` fields, optional fields with defaults) — this is the exact structure a model sees when deciding how to call it.
    - **MCP has two transports, and this page shows both working**: **stdio** (the server is a local subprocess, talked to over stdin/stdout — no network at all) and **Streamable HTTP** (the server is a remote endpoint, reachable over a URL). Same client code, same tool calls, different wiring underneath.
    - **Going deeper, for anything built for production**: MCP is stateless today (every request is self-contained via `_meta`) — with a real, dated finding that `mcp==2.2.0`'s default HTTP mode doesn't actually behave statelessly unless you opt in. A `requestState` token (real AES-256-GCM encryption) protects any state that must survive across a multi-step tool call, and a real repro confirms its tamper/request-binding/principal-binding checks all hold.

## What is MCP?

MCP (Model Context Protocol) is an open standard for connecting AI applications to external systems — data sources like local files and databases, tools like search engines and calculators, and workflows like reusable prompts. Anthropic's own framing is the clearest starting point: *"Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect electronic devices, MCP provides a standardized way to connect AI applications to external systems."* The problem that solves is specific: without a shared standard, every AI application that wants to use a given tool has to write its own custom integration for it. Build the tool once as an MCP server, and any MCP-compatible application (Claude, ChatGPT, VS Code, Cursor, and others) can use it without new integration work.

That's the pitch. The rest of this page is about what that actually looks like in running code — starting from the three words you'll see in every sentence about MCP: Host, Client, Server.

## The three pieces, defined against each other

The clearest way to learn these three terms is in the order they depend on each other, not as three unrelated flashcards:

- **Host** — the AI application you actually run or interact with. It could be a polished app like Claude Desktop or VS Code, or it could just be a Python script you write yourself. The Host is the thing that decides, in a conversation, "I should use a tool right now."
- **Client** — a Host doesn't talk to a Server directly. Instead, for every Server it wants to use, the Host creates one **Client**: a dedicated connection object, like a private phone line reserved for talking to exactly one Server. If a Host is using three different servers, it has created three separate Clients, one per line.
- **Server** — the program on the other end of that line. This is the piece that actually owns the tools, data, or prompt templates, and answers whatever the Client asks it.

Concretely, in every example on this page: **the Python script you run is the Host.** It creates one Client. That Client connects to a Server — sometimes defined in the very same file for simplicity, sometimes running as a genuinely separate process (you'll see both below). A real product like Claude Desktop does exactly this, just with a UI wrapped around it and often many Clients running at once, one per configured Server.

A Server exposes what it has through three primitives, defined directly in MCP's own current documentation: *"Tools: Executable functions that AI applications can invoke to perform actions. Resources: Data sources that provide contextual information to AI applications. Prompts: Reusable templates that help structure interactions with language models."* This page focuses on Tools, the most commonly used of the three — everything below builds one, lists it, and calls it, for real.

## Your first MCP server

The smallest real MCP program is a server with a couple of tools:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:build-server"
```

`@server.tool()` is doing more than it looks like. It reads the function's name, its docstring, and its type-hinted parameters, and turns all three into a **Tool primitive** — a structured description any MCP client can discover and call, without ever seeing this Python source. You never write a schema by hand; the schema is derived from ordinary code. The next section shows exactly what that generated schema looks like.

## Connecting a client and calling a tool

Now the other half — a Client that connects to that server, asks it what it can do, and calls one of its tools:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:basic-usage"
```

!!! success "A real run — one server, one client, one tool call"
    ```json
    {
      "tools": [
        {"name": "add", "description": "Add two numbers."},
        {"name": "multiply", "description": "Multiply two numbers."},
        {"name": "greet", "description": "Greet someone by name."}
      ],
      "add(2, 3)": {"result": 5}
    }
    ```

    Two calls happened here, and they're the two things every MCP client does: `list_tools()` is **discovery** — the Client asking the Server "what can you do?" — and `call_tool()` is **execution** — the Client telling the Server "actually do one of those things, with these arguments." Everything else on this page is a variation on exactly this pair.

## Multiple tools, and the real schema behind each one

The server above already has three tools, not one. Here's what a client actually receives when it asks for all of them — the real, auto-generated JSON Schema behind each:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:multi-tool-schema"
```

!!! success "A real run — three tools, three real schemas"
    ```json
    {
      "tool_schemas": [
        {
          "name": "add",
          "description": "Add two numbers.",
          "inputSchema": {
            "type": "object",
            "properties": {"a": {"title": "A", "type": "integer"}, "b": {"title": "B", "type": "integer"}},
            "required": ["a", "b"],
            "title": "addArguments"
          }
        },
        {
          "name": "greet",
          "description": "Greet someone by name.",
          "inputSchema": {
            "type": "object",
            "properties": {
              "name": {"title": "Name", "type": "string"},
              "formal": {"default": false, "title": "Formal", "type": "boolean"}
            },
            "required": ["name"],
            "title": "greetArguments"
          }
        }
      ],
      "add(2, 3)": {"result": 5},
      "greet(name=\"Dhruv\", formal=True)": {"result": "Good day, Dhruv."}
    }
    ```
    (`multiply`'s schema is identical in shape to `add`'s and is omitted above for space — the real run includes it.)

    Notice the difference between the two schemas shown: `add`'s function signature has no defaults, so both `a` and `b` land in `"required"`. `greet`'s `formal: bool = False` parameter does the opposite — it appears in `properties` with `"default": false` and is *absent* from `"required"`, because the Python default made it optional. This is the exact structure a language model is shown when it's deciding how to call a tool — a required-vs-optional distinction you get for free just by how you write the function signature, not something you author separately.

## Two ways to reach a server: stdio and Streamable HTTP

Every example so far ran the Client and Server in the same process, over HTTP on your own machine. That's one of MCP's two real transports — and worth seeing next to the other one, because they solve different problems.

**stdio** is for a server that lives on your own machine, launched by your Host as a child process, and talked to over that process's standard input and output — no network, no port, nothing to expose. This is how most local developer tools work today (a filesystem server, a git server): your Host starts the server program itself and pipes messages to it directly.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/stdio_server.py"
```

That file runs as its own separate program. Here's the client side, connecting to it as a subprocess rather than a URL:

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:stdio-client"
```

!!! success "A real run — the exact same tool calls, now over stdio instead of HTTP"
    ```json
    {
      "tools": ["add", "greet"],
      "add(4, 5)": {"result": 9},
      "greet(name=\"Dhruv\")": {"result": "Hey Dhruv!"}
    }
    ```

    `StdioServerParameters(command=sys.executable, args=[...])` tells the SDK's own `Client` to launch `stdio_server.py` as a real child process and speak MCP over its stdin/stdout, instead of opening an HTTP connection. The calling code barely changes — `list_tools()` and `call_tool()` are identical — only how the Client *reaches* the Server is different.

**Streamable HTTP**, the transport every earlier example on this page already used, is for a server that runs somewhere else — on a different machine, shared across many users — reachable over an ordinary URL, the way GitHub's or Stripe's own official MCP servers work today. A local stdio server typically serves one client at a time; a remote HTTP server typically serves many at once.

**The rule of thumb**: reaching for a tool that only needs to run on your own machine, with nothing to expose to anyone else? stdio. Building or connecting to a server other people or other machines need to reach over the network? Streamable HTTP.

## Going deeper: how MCP behaves under the hood

Everything above is enough to build and use a real MCP server. The rest of this page is optional, deeper material on two things worth knowing before shipping anything for real: how MCP avoids needing to remember you between requests, and how it protects the one piece of state a multi-step tool call sometimes genuinely needs to carry.

### Statelessness: every request is self-contained

MCP's data layer is, today, a stateless protocol: *"Every request contains all the information needed to process it, so servers infer nothing from previous requests."* Concretely, every request a client sends carries its own protocol version and capabilities in a `_meta` field — the server doesn't look anything up about "this connection," because there's nothing connection-specific to look up:

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

That `_meta` block is the whole point: the server can answer this request correctly using only what's inside it, regardless of what request came before it or from which client.

The cookbook's `mcp==2.2.0` install explicitly targets this spec — its own SDK documentation states it was built *"to support the 2026-07-28 MCP specification (and every earlier revision)."* The obvious question, given that stateless design: does the SDK's actual default behavior match it? Concretely, that means sending one raw HTTP request — a plain `tools/list` call, with no session ID attached, because a stateless server shouldn't need one — first against the server's default settings, then again with its explicit `stateless_http=True` mode turned on, and comparing what actually comes back.

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

    Both succeeded, neither response carried `Mcp-Session-Id`, and both independently returned the identical tool list — exactly the spec's stated behavior.

    **The honest finding**: spec-compliant statelessness is real, implemented, and works correctly in this SDK — it's just not what you get by not passing an argument. A team reading "our SDK targets the 2026-07-28 spec" and assuming the default configuration is therefore stateless would be wrong, as of this SDK version, checked 2026-09-23.

### `requestState`: protecting the one thing that does need to persist

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

    **4. Principal-binding — the state-handle-hijacking attack itself** — the same token and the same cart ID, but presented by `"user:mallory"` instead of `"user:alice"`: rejected with `"principal"`. This is a real, working test of the exact attack the spec's security document names: an attacker who somehow obtains another user's handle (a leaked log line, a shared client, a guessed value) and tries to use it to act as that user. The spec's own required mitigation: *"MCP servers **MUST NOT** treat possession of a state handle as authentication"* and *"**SHOULD** bind handles server-side to the authenticated user."*

    **5. Expiry** — the identical legitimate request, retried 1.2 seconds after a 1-second TTL: rejected with `"expired"`.

    All five checks ran against the SDK's own real `AESGCMRequestStateCodec` for the cryptographic sealing and unsealing — the request-binding, principal-binding, and expiry checks are this recipe's own code, built to mirror the exact claims structure read directly from the installed SDK's `RequestStateBoundary._seal`/`_unseal` source, not a black-box guess at what it does.

### Auth: token passthrough is forbidden, not discouraged

MCP's authorization model rests on a boundary that's easy to blur in an MCP server that proxies to a third-party API: a token issued *to* the MCP server is not the same thing as a token that's safe to *forward* unmodified to a downstream service. The spec's security document is unambiguous about this specific anti-pattern: *"'Token passthrough' is an anti-pattern where an MCP server accepts tokens from an MCP client without validating that the tokens were properly issued to the MCP server and passes them through to the downstream API."* The real risk isn't abstract — it directly reintroduces the confused-deputy problem the `requestState` repro above is adjacent to: *"the downstream API may incorrectly trust the token as if it came from the MCP server or assume the token was validated by the upstream API."* The mitigation is stated as a hard requirement, not a suggestion: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server."*

## What this means in practice

Statelessness and `requestState` are the same design decision, applied twice. Because a server can't lean on the transport to remember who's talking to it between requests, anything that needs to persist across a round trip (like a paused checkout awaiting payment confirmation) has to become an explicit, self-contained value the client carries — and once that value exists outside the protected connection, it needs its own cryptographic guarantees, because nothing about the wire protects it anymore. That's exactly what `requestState`'s AEAD sealing, request-binding, and principal-binding provide. A team building an MCP server today gets this security property "for free" if they use the SDK's own `RequestStateSecurity`/`RequestStateBoundary` middleware as designed — but this page's own repro is the reason "the SDK targets the current spec" isn't the same claim as "the SDK behaves per the current spec by default": statelessness itself required an explicit flag to actually take effect.

## Interview angle

**Weak answer** to "why is MCP described as a stateless protocol?": *"Because it doesn't use sessions."* True, but incomplete in a way that misses the actual design insight — it doesn't explain what handles cross-call state instead (flows like elicitation genuinely need to persist something between requests), or why that replacement needs cryptography that a session-ID string never did.

**Strong answer**: statelessness means the server needs nothing remembered from earlier requests to answer the current one — every request is self-contained via its `_meta` field. That doesn't eliminate the need for cross-call state; it moves the responsibility for protecting that state from the transport (an opaque session ID the server tracks server-side) to an explicit, self-describing value the client carries (a `requestState` token the server must cryptographically verify on every use). That's a strictly harder problem to get right than session tracking was, which is exactly why the SDK ships a real AEAD-sealed, request-bound, principal-bound implementation rather than leaving it to each server author — and exactly why "state handle hijacking" earned its own named entry in the spec's security document.

**Follow-up to expect**: "if the SDK already targets the current spec, why does it still default to session-based mode?" Because "targets the spec" describes what the SDK is *capable* of, not what ships enabled — this page's own real test showed exactly that gap: `stateless_http=True` produces fully spec-compliant behavior, but it's opt-in, and a team that assumes an up-to-date SDK version implies up-to-date default behavior would ship session-based semantics without realizing it. The lesson generalizes past MCP: "supports the spec" and "defaults to the spec's behavior" are different claims, and only checking the actual runtime behavior — not the changelog, not the version number — tells you which one you're getting.

## Build it yourself — 30 minutes

1. Build a one-tool `MCPServer` with `@server.tool()` (this page's own `add` example is enough), and connect to it with the SDK's `Client` — call `list_tools()` then `call_tool()`. This is the entire mechanism; everything else on this page builds on it.
2. Add a second tool with an optional, defaulted parameter (this page's own `greet` is one example). Print the real schema `list_tools()` returns for both, and check you can explain, from the schema alone, which arguments are required.
3. Run the same server two ways: as a subprocess over stdio (`StdioServerParameters` + the SDK's `Client`), and over Streamable HTTP (`streamable_http_app()` + a URL). Confirm the same `list_tools()`/`call_tool()` calls work identically both ways.
4. Then go deeper: run that same HTTP server with `streamable_http_app(stateless_http=True)` versus without it, and compare the real response headers side by side — this page's own repro is exactly this comparison, just automated. Read your installed SDK's own `request_state.py` directly and reproduce one of this page's four `requestState` attacks (tamper, request-binding, principal-binding, expiry) against your own server's real token-minting code.

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

- Model Context Protocol, ["Introduction"](https://modelcontextprotocol.io/) — the official "what is MCP" page this page's own USB-C analogy is quoted directly from.
- Model Context Protocol, ["Architecture overview"](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture) — the current Host/Client/Server model, the Tools/Resources/Prompts primitives, stdio and Streamable HTTP transports, and the full worked `server/discover` → `tools/list` → `tools/call` example this page's own JSON snippet is drawn from.
- Model Context Protocol, ["Key Changes — 2026-07-28"](https://modelcontextprotocol.io/specification/2026-07-28/changelog) — the current spec's own changelog, including statelessness (SEP-2575) and MRTR (SEP-2322).
- Model Context Protocol, ["Security Best Practices"](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices) — State Handle Hijacking, Token Passthrough, the Confused Deputy Problem, SSRF, and the rest of the spec's named attacks and mitigations.
- [`modelcontextprotocol/python-sdk`](https://github.com/modelcontextprotocol/python-sdk) — the official SDK (`mcp==2.2.0` at the time this page was written), including `mcp/server/request_state.py`, read directly for this page's requestState repro.
- [MCP and Tool Ecosystem](mcp-tool-ecosystem.md) — once you're comfortable with the basics on this page: the registry, gateways, hosted remote servers, and tool-aggregation platforms built around MCP.
- [Agent Security](agent-security.md) — the lethal trifecta, Rule of Two, and tool poisoning; a related but distinct set of agent-level (not protocol-level) security failure modes.
- [Tools at Scale](tools-at-scale.md) — Anthropic's code-execution-with-MCP pattern, presenting MCP servers as code APIs; complements this page's protocol-level focus with a context-efficiency angle on MCP usage.
