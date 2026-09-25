# MCP Deep Dive

!!! example "Hands-on"
    Full runnable recipe: [`mcp-deep-dive/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/mcp-deep-dive) in the companion cookbook — two real repros against the official MCP Python SDK (`mcp==2.2.0`, targeting the 2026-07-28 spec revision). No API key needed, pure protocol mechanics: this page is about how MCP itself keeps (or now, deliberately doesn't keep) track of a conversation between requests — not about calling any model.

??? abstract "TL;DR — quick revision"
    - **MCP's 2026-07-28 revision makes the protocol stateless at the wire level**: the `initialize`/`notifications/initialized` handshake and the `Mcp-Session-Id` header are removed from Streamable HTTP; every request carries its own protocol version and capabilities; list endpoints no longer vary per connection. A real, dated finding: `mcp==2.2.0` (the SDK that explicitly targets this spec) still uses `Mcp-Session-Id` **by default** — spec-compliant statelessness is real and working, but is an opt-in flag (`stateless_http=True`), not the default.
    - **Multi Round-Trip Requests (MRTR) replaced server-initiated requests** (`roots/list`, `sampling/createMessage`, `elicitation/create`) with a request/retry pattern: a server returns `InputRequiredResult`, the client retries the original request carrying the answer plus an opaque `requestState` token the server minted and must re-verify.
    - **A real repro of the SDK's own `requestState` security held on every check**: tampering with a sealed token is rejected (AEAD authentication failure), replaying a token against a different tool argument is rejected (request-binding), and — the important one — replaying one user's token as a different user is rejected (principal-binding). That last check is the real, working mitigation for the spec's own named "State Handle Hijacking" vulnerability: *"MCP servers **MUST NOT** treat possession of a state handle as authentication."*
    - **A real, current list of deprecations matters for anything built today**: HTTP+SSE transport (migrate to Streamable HTTP), Roots/Sampling/Logging features (migrate to tool parameters, direct provider APIs, and OpenTelemetry respectively), and OAuth Dynamic Client Registration (migrate to Client ID Metadata Documents) are all now formally Deprecated under a twelve-month removal window, not just "discouraged."
    - **Token passthrough is explicitly forbidden, not just risky**: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server"* — a server that blindly forwards a client-supplied token downstream breaks a real OAuth security boundary and reintroduces the confused-deputy problem the rest of the spec's auth model is built to prevent.

## The big picture, before the mechanics

This page walks through one architectural decision and its direct, unavoidable consequence. MCP's 2026-07-28 spec revision deletes the mechanism a server used to remember who it was talking to between requests — that's the "statelessness" change in the first repro below. Deleting that mechanism doesn't delete the *need* for state, though: some real flows (like a server that needs to pause mid-task and ask the client for more information) still have to carry information from one request to the next. The spec's answer is to make that leftover state an explicit, ordinary value instead of invisible connection tracking — which is exactly why it now needs real cryptography to stay safe, covered in the second repro (`requestState`). Same underlying idea, applied twice: stop relying on the transport to remember things for you, and anything that still needs remembering has to protect itself.

## What a "session" was, and why the spec removes it

Concretely, here's what MCP session state meant before this revision. A client's very first message to a server was a handshake — `initialize`, then `notifications/initialized` — where the two sides agreed on a protocol version and a set of capabilities, much like logging into a website. In exchange, the server handed back a session ID, carried in an `Mcp-Session-Id` header, that the client then had to attach to every request after that — the same job a browser's login cookie does. The server kept its own memory per session ID: which tools it had already listed for this client, what capabilities this particular handshake had settled on. Two clients hitting the same server could genuinely get different answers, purely because of what each one's handshake happened to accumulate.

The 2026-07-28 revision deletes that mechanism outright, not just trims it: *"Make MCP stateless: remove the `initialize`/`notifications/initialized` handshake. Every request now carries its protocol version and client capabilities in `_meta`"* and *"Remove protocol-level sessions and the `Mcp-Session-Id` header from the Streamable HTTP transport. List endpoints (`tools/list`, `resources/list`, `prompts/list`) no longer vary per-connection. Servers that need cross-call state use explicit, server-minted handles passed as ordinary tool arguments."*

That last sentence is the important design move: statelessness at the protocol level doesn't mean servers can't have state — it means state stops being implicit, connection-tracked protocol machinery, and becomes an explicit value the server hands the client and the client hands back, like any other tool argument. That shift is exactly what makes this page's second repro (the `requestState` envelope) both necessary and interesting: once state is just a value passed around, it needs its own integrity and authorization guarantees, because nothing about the transport protects it anymore.

## Repro 1: statelessness in practice, and a real gap between spec and default

The cookbook's `mcp==2.2.0` install explicitly targets the 2026-07-28 spec — its own SDK documentation states it was built *"to support the 2026-07-28 MCP specification (and every earlier revision)."* The obvious question: does the SDK's actual default behavior match the spec's statelessness mandate? Concretely, that means sending one raw HTTP request — a plain `tools/list` call, with no handshake and no session ID attached, because the spec says none of that should be needed anymore — first against the server's default settings, then again with its explicit `stateless_http=True` mode turned on, and comparing what actually comes back.

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/mcp-deep-dive/mcp_deep_dive_docs.py:running-server"
```

!!! success "A real run — raw HTTP requests against the SDK's own real server, default mode vs. explicit stateless mode"
    **Default mode** (`streamable_http_app()`, no arguments) — a raw `tools/list` POST with no prior handshake, sent directly with `httpx2` (the real HTTP client, no MCP-level session management):

    ```json
    {"jsonrpc":"2.0","id":null,"error":{"code":-32600,"message":"Bad Request: Missing session ID"}}
    ```

    In plain terms: the server refused to answer at all. It rejected the request outright, and the response carried a real `mcp-session-id` header — the server was insisting on a session ID that this raw request never had a chance to obtain, because there was no handshake to obtain it from. **This is the SDK's default**, in a release that explicitly targets a spec that says sessions and this exact header are removed.

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

## Repro 2: `requestState` — the replacement for server-initiated requests

Picture an ordinary web form that's missing one required field: a well-built site doesn't throw away everything you typed — it asks you for just the missing piece and lets you resubmit. That's the shape of the pattern MCP adopted here. Before this revision, a server could interrupt a request mid-flight and directly ask the client a question (`sampling/createMessage`, `elicitation/create`); the new spec removes that server-initiated interruption and replaces it with a request/retry loop the *client* drives instead, called **Multi Round-Trip Requests (MRTR)**: *"Servers return an `InputRequiredResult` (`resultType: "input_required"`) whose `inputRequests` field carries the requests for the additional information needed to process the request. Clients respond with `inputResponses` on a retry of the original request providing the requested information."* The catch is exactly what you'd expect from a resubmitted form on the open internet: the server needs a way to know, on that retry, that it genuinely continues the same interrupted request rather than an attacker's unrelated request smuggled in — that's what the `requestState` token carries.

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

## Deprecations that matter for anything built today

The 2026-07-28 revision formalizes a feature lifecycle policy — Active, Deprecated, Removed, with a minimum twelve-month deprecation window — and uses it immediately on several features real systems still use:

- **HTTP+SSE transport** (deprecated since `2025-03-26`, now formally reclassified Deprecated): *"Migrate to Streamable HTTP."* Anything built against the older SSE-based transport is now on a formal removal clock, not just informally discouraged.
- **Roots, Sampling, and Logging features**: *"remain fully functional during the deprecation window but new implementations should not add support for them."* Suggested migrations: pass directories via tool parameters instead of Roots, integrate directly with LLM provider APIs instead of Sampling, log to `stderr` or use OpenTelemetry instead of the Logging feature.
- **OAuth 2.0 Dynamic Client Registration**: deprecated as the default client-registration mechanism *"in favor of Client ID Metadata Documents"* — though it *"remains available for backwards compatibility with authorization servers that do not support"* the new approach.

None of these break existing deployments immediately — that's the point of a twelve-month window — but a new integration built today against the deprecated path is building on a clock that's already running.

## Auth: token passthrough is forbidden, not discouraged

MCP's authorization model rests on a boundary that's easy to blur in an MCP server that proxies to a third-party API: a token issued *to* the MCP server is not the same thing as a token that's safe to *forward* unmodified to a downstream service. The spec's security document is unambiguous about this specific anti-pattern: *"'Token passthrough' is an anti-pattern where an MCP server accepts tokens from an MCP client without validating that the tokens were properly issued to the MCP server and passes them through to the downstream API."* The real risk isn't abstract — it directly reintroduces the confused-deputy problem this page's `requestState` repro is adjacent to: *"the downstream API may incorrectly trust the token as if it came from the MCP server or assume the token was validated by the upstream API."* The mitigation is stated as a hard requirement, not a suggestion: *"MCP servers **MUST NOT** accept any tokens that were not explicitly issued for the MCP server."*

## What this means in practice

As the big picture up top promised: statelessness and `requestState` aren't two unrelated features — they're the same design decision, applied twice. Removing protocol-level sessions means a server can no longer lean on the transport to remember who's talking to it between requests; anything that needs to persist across a round trip has to become an explicit, self-contained value the client carries — and once that value exists, it needs its own cryptographic guarantees, because nothing about the wire protects it anymore. That's exactly what `requestState`'s AEAD sealing, request-binding, and principal-binding provide. A team building an MCP server today gets this security property "for free" if they use the SDK's own `RequestStateSecurity`/`RequestStateBoundary` middleware as designed — but this page's first repro is the reason "the SDK targets the new spec" isn't the same claim as "the SDK behaves per the new spec by default": the statelessness half of the same design decision required an explicit flag to actually take effect.

## Interview angle

**Weak answer** to "what changed in the newest MCP spec?": *"It removed sessions to make the protocol stateless."* True, but incomplete in a way that misses the actual design insight — it doesn't explain what replaced the thing sessions used to provide (cross-call state for flows like elicitation), or why that replacement needs cryptography that a session-ID string never did.

**Strong answer**: removing protocol-level sessions doesn't remove the need for cross-call state — it moves the responsibility for protecting that state from the transport (an opaque session ID the server tracks server-side) to an explicit, self-describing value the client carries (a `requestState` token the server must cryptographically verify on every use). That's a strictly harder problem to get right than session tracking was, which is exactly why the SDK ships a real AEAD-sealed, request-bound, principal-bound implementation rather than leaving it to each server author — and exactly why "state handle hijacking" earned its own named entry in the spec's security document: an unprotected handle is a much easier target than a server-side session table ever was.

**Follow-up to expect**: "if the SDK already targets the new spec, why does it still default to session-based mode?" Because "targets the spec" describes what the SDK is *capable* of, not what ships enabled — this page's own real test showed exactly that gap: `stateless_http=True` produces fully spec-compliant behavior, but it's opt-in, and a team that assumes an up-to-date SDK version implies up-to-date default behavior would ship the old session-based semantics without realizing it. The lesson generalizes past MCP: "supports the new spec" and "defaults to the new spec's behavior" are different claims, and only checking the actual runtime behavior — not the changelog, not the version number — tells you which one you're getting.

## Build it yourself — 30 minutes

1. Install `mcp` and build a two-tool `MCPServer` (the README's own `add`/`greeting` example is enough). Run it with `streamable_http_app()` (default) and send one raw `tools/list` POST with a plain HTTP client, no MCP session library involved. Read the actual error.
2. Re-run the same raw request against `streamable_http_app(stateless_http=True)`. Compare the real response headers side by side — this page's own repro is exactly this comparison, just automated.
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

- Model Context Protocol, ["Key Changes — 2026-07-28"](https://modelcontextprotocol.io/specification/2026-07-28/changelog) — the full changelog: statelessness (SEP-2575), MRTR (SEP-2322), the feature lifecycle and deprecation policy (SEP-2596), and every deprecation named on this page.
- Model Context Protocol, ["Security Best Practices"](https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices) — State Handle Hijacking, Token Passthrough, the Confused Deputy Problem, SSRF, and the rest of the spec's named attacks and mitigations.
- [`modelcontextprotocol/python-sdk`](https://github.com/modelcontextprotocol/python-sdk) — the official SDK (`mcp==2.2.0` at the time this page was written), including `mcp/server/request_state.py`, read directly for this page's second repro.
- [Agent Security](agent-security.md) — the lethal trifecta, Rule of Two, and tool poisoning; a related but distinct set of agent-level (not protocol-level) security failure modes.
- [Tools at Scale](tools-at-scale.md) — Anthropic's code-execution-with-MCP pattern, presenting MCP servers as code APIs; complements this page's protocol-level focus with a context-efficiency angle on MCP usage.
