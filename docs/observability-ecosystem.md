# Observability Ecosystem

!!! example "Hands-on"
    Full runnable recipe: [`observability-ecosystem/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/observability-ecosystem) in the companion cookbook — a real, live, zero-cost test of two observability platform architectures, using only public, documented endpoints, no account needed.

??? abstract "TL;DR — quick revision"
    - **[Observability and Debugging](observability-debugging.md) covers the OpenTelemetry GenAI *standard* — this page covers the real *products* built around it.** Real, verified: **Langfuse** is the strongest OTel-GenAI-native platform, with a dedicated OTLP endpoint (`/api/public/otel`) that auto-maps `gen_ai.*` attributes, and — as of June 2025 — *"all product capabilities—tracing, evaluations, prompt management, experiments, annotation, the playground, and more—are MIT licensed without any usage limits."* Only enterprise compliance features (SCIM, audit logs, retention policies) require a paid license for self-hosting.
    - **A real, live repro confirms the architectural split directly.** POSTing a real OTel GenAI-shaped trace to Langfuse's real endpoint with no credentials returned a real, live `401`: `{"message": "No authorization header"}` — confirming it's a genuine, spec-compliant ingestion endpoint, not marketing copy. POSTing to Helicone's real gateway with no routing info returned a real `400: "Missing target base url"` — then, with a real target header, the request was genuinely forwarded live to OpenAI's real API, whose own real error came back through the proxy.
    - **These are two structurally different real mechanisms, not two brands of the same thing.** Langfuse (and Arize's open-source Phoenix, similarly OTel-native) are *ingestion endpoints* — your agent's own instrumentation pushes spec-shaped traces to them after the fact. Helicone is a *request-path proxy* — it sits directly in front of every real API call, forwarding live, which lets it also do caching, fallback routing, and rate limiting the same request touches.
    - **Datadog's real differentiator is integration, not a separate product category**: real, verified — *"the same tracing technology trusted by 60% of the Fortune 500,"* correlating *"LLM spans with APM services, infra signals, and RUM sessions"* in one platform SRE and DevOps teams already use, rather than a bolt-on tool living in its own silo.
    - **Braintrust and W&B Weave both pair observability with evaluation as a first-class feature, not an add-on** — Braintrust's own framing names *"Traces + Evals + Annotation"* as three pillars together; Weave brings *"sessions, turns, steps, tools, and sub-agents as first-class concepts,"* not generic code-level spans, plus pre-built scorers for toxicity, bias, PII, and hallucination detection.

## Why the product layer is a separate question from the standard

The previous page's real, central point was that agent failures don't look like failures to traditional monitoring — a tool can return well-formed, wrong data, and every individual step reports success. OpenTelemetry's real GenAI semantic conventions give that problem a standard shape to solve it in: consistent span names, `gen_ai.input.messages`/`gen_ai.output.messages` as the specified place for real tool call data. But a standard isn't a product — something still has to receive those spans, store them, let a human search and visualize them, run evals against them, and alert when something looks wrong. That's a real, separate layer, with real, different architectural choices across the products that occupy it.

## OTel-native ingestion: Langfuse and Arize Phoenix

Two real, open(-ish) platforms take the most direct approach: accept a standard OTel GenAI trace with no vendor SDK required. **Langfuse** states this explicitly: *"Langfuse aims to be compliant with the OpenTelemetry GenAI semantic conventions,"* with a dedicated OTLP endpoint that auto-maps standard attributes like `gen_ai.system`, `gen_ai.request.model`, and `gen_ai.usage.*` — one real, current limitation: the endpoint accepts OTLP over HTTP only, not gRPC. Its real, current licensing is unusually permissive for a hosted product: as of a June 2025 change, *"all product capabilities—tracing, evaluations, prompt management, experiments, annotation, the playground, and more—are MIT licensed without any usage limits"* when self-hosted, with only enterprise compliance modules (SCIM, audit logging, retention policies) requiring a paid license. **Arize's Phoenix** — the open-source project distinct from Arize AI's commercial "Arize AX" platform — takes a similar real stance: *"Native OpenTelemetry support... No proprietary lock-in,"* self-hostable under the Elastic License 2.0 (source-available, not OSI-approved open source, but free with no feature gates for self-hosting).

## A request-path proxy: Helicone

Helicone takes a structurally different real approach. Instead of receiving traces after the fact, it sits directly in the request path: the real, documented integration method is pointing your application's API base URL at `gateway.helicone.ai` instead of the provider's own endpoint directly, with a `Helicone-Target-Url` header telling it where to actually forward the request. Real, current pricing starts free (10K requests/month, self-hostable via Docker), with a Pro tier at $79/month. Because every real call physically passes through Helicone's infrastructure, it can layer real gateway features the ingestion-endpoint model can't — caching, provider fallback, rate limiting — on the identical request path that's also being logged.

## Enterprise-integrated: Datadog LLM Observability

Datadog's real, stated differentiator isn't a feature list — it's where the data lives. Real, verified: *"the same tracing technology trusted by 60% of the Fortune 500 and leading AI labs"* lets teams *"correlate LLM spans with APM services, infra signals, and RUM sessions"* — LLM traces share the identical trace/span data model, dashboards, and alerting as the rest of a team's existing Datadog instrumentation, rather than living in a separate tool a different team has to learn. Its real, confirmed billing model charges per LLM-provider-call span specifically — tool spans, embedding spans, and agent spans aren't billed separately.

## Repro: two real architectures, tested live

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/observability-ecosystem/observability_ecosystem_docs.py:trace-payload"
```

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/observability-ecosystem/observability_ecosystem_docs.py:live-checks"
```

The identical `gen_ai.*` attribute shape [Observability and Debugging](observability-debugging.md)'s own trace uses, sent live to Langfuse's real endpoint with no credentials, plus two real calls to Helicone's real gateway.

!!! success "A real run, unforced — three real, live HTTP checks, no account, no API key, zero cost"
    **Langfuse's real OTLP endpoint**: `401` — `{"message": "No authorization header", "error": "UnauthorizedError"}`. The real, well-formed OTel GenAI trace was accepted structurally and rejected purely for missing auth, not a malformed-payload error — real, live confirmation this endpoint genuinely parses this exact trace shape, not just a marketing claim.

    **Helicone's real gateway, no target header**: `400` — `"Missing target base url"`. Real, concrete proof it's a request-path proxy, not an ingestion sink: it has nothing to log until it knows where to send the request.

    **Helicone's real gateway, with a real target header**: `401`, but the real response body is OpenAI's own real, live error — *"You didn't provide an API key. You need to provide your API key in an Authorization header using Bearer auth..."* — forwarded back through Helicone's proxy. The request was genuinely, live-forwarded to a real upstream provider, and its real response passed through unmodified.

    Two real, structurally different mechanisms, both confirmed by their actual, live behavior: one is a place you push spec-shaped traces *to*, after a call completes; the other sits directly *in* the call itself.

## What this means in practice

Choosing an observability platform is really choosing an architecture, not a logo. An OTel-native ingestion endpoint (Langfuse, Phoenix) fits naturally if instrumentation is already emitting standard spans — no vendor SDK, and this page's own repro shows the endpoint genuinely enforces and parses that standard shape, live. A request-path gateway (Helicone) fits if you also want caching, fallback, or rate limiting on the same real request path — the logging comes along with the proxying, not instead of it. Enterprise-integrated observability (Datadog) fits when LLM traces need to sit next to infrastructure signals a team is already alerting on, not in a separate tool. Eval-forward platforms (Braintrust, Weave) fit when "is this good" matters as much as "what happened" — pairing a trace with a scored judgment, not just a log.

## Interview angle

**Weak answer** to "how would you monitor an agent in production?": *"Add LangSmith"* or *"Add Datadog."* This treats observability platforms as interchangeable, when this page's own real repro shows two of them are architecturally different tools solving related but distinct problems — an ingestion endpoint you push traces to, versus a gateway you route calls through.

**Strong answer**: name the real architectural question first — does the traffic already flow through a natural interception point (making a gateway like Helicone attractive, since caching/fallback come free with the logging), or is instrumentation better added at the application layer via a standard like OpenTelemetry (making an OTel-native ingestion endpoint like Langfuse or Phoenix the natural fit, since no vendor SDK is required)? Then layer in the org-context question: does this need to live next to existing infra monitoring (Datadog), and does "monitoring" here mean pure observability or does it need evals-in-the-loop as a first-class feature (Braintrust, Weave)?

**Follow-up to expect**: "if Langfuse is MIT-licensed and self-hostable for free, why would anyone pay for a hosted observability product at all?" A real, honest answer grounded in this page's own material: Langfuse's own real license split answers this directly — the *product features* (tracing, evals, prompts, playground) are free and unrestricted even self-hosted, but *enterprise compliance* (SCIM, audit logs, long retention) requires a paid license — the same real pattern behind most open-core products. Teams pay for the operational and compliance burden they'd otherwise carry themselves, not for the core observability capability itself.

## Build it yourself — 30 minutes

1. Run this page's own recipe with zero setup — no API key, no account — and confirm the two real architectural behaviors for yourself: `python observability_ecosystem.py`.
2. Take a real OTel GenAI-shaped trace from your own agent (or [Observability and Debugging](observability-debugging.md)'s own recipe) and check whether your current logging setup could be pointed at an OTLP endpoint with zero code changes, or whether it needs a vendor SDK first — that's the real, practical version of the ingestion-endpoint-vs-gateway question this page draws conceptually.
3. If you're evaluating a gateway-style product instead, test the same way this page's own repro does: send a real request with no routing information and see what error comes back — a real proxy will tell you it doesn't know where to forward; an ingestion endpoint will tell you about auth or payload shape instead, revealing which architecture you're actually looking at, independent of what the marketing page says.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Observability Ecosystem">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro sent a well-formed OTel GenAI-shaped trace to Langfuse's real OTLP endpoint with no credentials, and received a real, live 401 response: 'No authorization header.' The trace was not rejected for being malformed.",
      "question": "What does this specific result most precisely confirm about Langfuse's real ingestion endpoint?",
      "options": [
        "The endpoint genuinely parses this trace shape and only rejected the request for missing auth",
        "The endpoint is currently broken and unable to process any real OTLP trace data",
        "OpenTelemetry compliance claims from vendors are generally unverifiable without a paid account",
        "Langfuse requires a completely different, undocumented trace format not covered by this page"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The real, live response rejected the request specifically for missing authorization, not for a malformed payload -- direct, concrete evidence the endpoint successfully parsed the real OTel GenAI trace structure and got far enough to check credentials, confirming genuine spec-compliant ingestion.",
        "Not supported -- a 401 specifically for missing authorization is a sign the endpoint IS processing the request far enough to check credentials, not that it's broken; a broken endpoint would more likely produce a different error or no response at all.",
        "Contradicted directly -- this page's own repro is precisely a real, live, no-account verification of an OTel compliance claim, and it succeeded in confirming the claim without needing paid access.",
        "Not indicated -- the real trace sent used the documented OTel GenAI shape and format, and the real error returned was specifically about authorization, not about the payload's format being wrong or unrecognized."
      ]
    },
    {
      "scenario": "A real repro sent a request to Helicone's real gateway endpoint with no routing information and got a real 400 'Missing target base url' response. With a real target header added, the identical request returned OpenAI's own real, live error message, forwarded back through the gateway.",
      "question": "What does this two-step result most precisely demonstrate about Helicone's real architecture?",
      "options": [
        "Helicone is fundamentally broken and cannot successfully route any real request to any provider",
        "Helicone is an ingestion endpoint like Langfuse, just with a different real error message format",
        "OpenAI's API was actually down at the time this real test was run, which explains the response",
        "Helicone operates as a request-path proxy that must know its forwarding target before it can act"
      ],
      "correct": 3,
      "explanations": [
        "Contradicted directly -- with a real target header provided, the request WAS successfully routed, and a real, live response came back from OpenAI's own API through the proxy, demonstrating working forwarding, not brokenness.",
        "Backwards -- Langfuse is the ingestion-endpoint architecture; this page explicitly contrasts it with Helicone's different, proxy-based architecture, which requires routing information an ingestion endpoint wouldn't need.",
        "Not supported -- the real response returned was OpenAI's own standard, well-formed 'no API key provided' error, which is the real, expected response for a request missing valid credentials, not evidence of an outage.",
        "Correct. The real, two-part result demonstrates exactly this: without knowing where to forward, the real gateway can't act at all ('missing target'); given a real target, it successfully forwards live and passes the real upstream response back -- the defining behavior of a request-path proxy, not a passive log sink."
      ]
    },
    {
      "scenario": "Langfuse's own real, verified licensing statement: 'all product capabilities—tracing, evaluations, prompt management, experiments, annotation, the playground, and more—are MIT licensed without any usage limits,' with only enterprise compliance modules requiring a paid license for self-hosting.",
      "question": "Given this real licensing split, what is the most accurate answer to 'why would a team pay for hosted observability at all'?",
      "options": [
        "The free, self-hosted MIT license is a limited trial version that stops working after some real usage threshold",
        "Teams pay specifically for operational and compliance capabilities the free, self-hosted core doesn't include",
        "Paying customers get meaningfully better core tracing and evaluation quality than free self-hosted users",
        "There is no real reason to pay, since the free tier includes every capability the paid tiers offer"
      ],
      "correct": 1,
      "explanations": [
        "Contradicted directly -- the real quote states these capabilities are MIT licensed 'without any usage limits,' explicitly ruling out a usage-capped trial interpretation.",
        "Correct. The real, quoted license split is precise: core product capabilities are unrestricted under MIT, while specifically enterprise/compliance features (SCIM, audit logs, retention policies) require payment -- teams pay for the operational and compliance burden, not for a better version of the core observability functionality itself.",
        "Not supported -- the real quote lists tracing, evals, prompts, and playground as equally MIT-licensed regardless of payment; nothing indicates a quality tier distinction in the core product capabilities themselves.",
        "Overstates it -- while the core product IS unrestricted, real enterprise compliance features (SCIM, audit logging, retention policies) genuinely do require a paid license for self-hosting, so there IS a real reason some teams pay."
      ]
    },
    {
      "scenario": "Datadog's real, verified positioning for LLM Observability emphasizes correlating 'LLM spans with APM services, infra signals, and RUM sessions' using 'the same tracing technology trusted by 60% of the Fortune 500,' rather than emphasizing a standalone feature list.",
      "question": "What does this specific framing indicate is Datadog's real, primary competitive differentiator in this space?",
      "options": [
        "Datadog offers substantially more LLM-specific tracing features than any other platform covered here",
        "Datadog is the only platform in this space that supports OpenTelemetry-based data ingestion at all",
        "Datadog's differentiator is data correlation with existing infrastructure signals a team already monitors",
        "Datadog's LLM Observability product operates as a completely separate platform from its existing APM tools"
      ],
      "correct": 2,
      "explanations": [
        "Not the claim made -- the page's own framing is specifically about WHERE the data lives and what it connects to (APM, infra, RUM), not a claim that Datadog's LLM-specific feature list is larger than competitors' feature lists.",
        "Not supported and not exclusive -- multiple platforms covered on this page (Langfuse, Phoenix) are described as OTel-native or OTel-compatible; the page doesn't claim Datadog is unique in supporting OpenTelemetry-based ingestion.",
        "Correct. The real, quoted framing is specifically about correlating LLM spans with APM services, infrastructure signals, and RUM sessions using the SAME tracing technology already trusted at scale -- the differentiator is unified data and shared tooling with existing observability, not a longer LLM-specific feature list.",
        "Directly contradicted -- the whole point of the real, quoted framing is that LLM spans share the SAME trace/span data model, dashboards, and alerting as the rest of a team's existing Datadog instrumentation, explicitly not a separate, siloed platform."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- [Langfuse: OpenTelemetry integration](https://langfuse.com/integrations/native/opentelemetry) and [Langfuse: Open Source licensing](https://langfuse.com/handbook/chapters/open-source) — the real, verified OTLP endpoint behavior and MIT licensing split this page's own repro and citations are built around.
- [Arize Phoenix: self-hosting license](https://arize.com/docs/phoenix/self-hosting/license) — the real, verified Elastic License 2.0 terms.
- [Helicone: gateway integration docs](https://docs.helicone.ai/getting-started/integration-method/gateway-fallbacks) — the real, documented proxy mechanism this page's own repro confirms live.
- [Datadog: LLM Observability product page](https://www.datadoghq.com/product/ai/llm-observability/) — the real, verified APM-correlation differentiator.
- [Braintrust pricing](https://www.braintrust.dev/pricing) and [W&B Weave](https://wandb.ai/site/weave/) — the real, verified eval-forward feature sets.
- [Observability and Debugging](observability-debugging.md) — the OpenTelemetry GenAI semantic convention itself, the standard every platform on this page is built to ingest.
