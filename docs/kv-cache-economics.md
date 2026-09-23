# KV-Cache Economics

!!! example "Hands-on"
    Full runnable recipe: [`kv-cache-economics/`](https://github.com/DhruvMakwana/agents-cookbook/tree/main/kv-cache-economics) in the companion cookbook — real cache write/read token counts against the live API, the real invalidation hierarchy, and tool masking vs. removal using the real `mid-conversation-tool-changes` beta.

??? abstract "TL;DR — quick revision"
    - **Prompt caching isn't a flat discount — it's a hierarchy with a precise cost structure.** A 5-minute cache write costs 1.25x base input price; a cache read costs 0.1x (cheaper still — 0.025x–0.05x — on some model families). The cache follows a strict prefix order, `tools` → `system` → `messages`, and a change at any level invalidates that level *and everything after it*.
    - **A real run confirmed the hierarchy precisely**: changing `tool_choice` between calls — same tools, same system prompt — left the cached tools+system prefix fully intact (`cache_read_input_tokens` unchanged). Editing one word in one tool's description invalidated the entire cache and forced a full, fresh write.
    - **"Tool masking vs. removal" is a real, current, shipped feature, not just a conceptual pattern**: the `mid-conversation-tool-changes` beta's `tool_removal` content block withdraws a tool from a running conversation while the top-level `tools` array stays byte-identical — so the cache survives. Physically editing the `tools` array to drop a tool produces a different array and a fresh cache entry, every time.
    - **A real repro measured the contrast directly**: masking a tool (three calls in a row, including one a full turn later) kept reading the identical 2,211-token cache entry. Removing the same tool by editing the array instead produced a new, distinct 2,113-token entry with no relationship to what came before.
    - **Masking is enforced, not cosmetic**: forcing `tool_choice` to the masked tool by name produced a real API error — *"forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back)"* — the model genuinely cannot call it, not just a description change that happens not to mention it.

## Why the cache has a hierarchy, not just a size

Anthropic's prompt cache doesn't treat a request as one undifferentiated blob of tokens — it walks the request in a fixed order and hashes prefixes as it goes: *"the cache follows the hierarchy: `tools` → `system` → `messages`. Changes at each level invalidate that level and all subsequent levels."* That ordering is deliberate, not incidental: `tools` and `system` are usually the most stable part of an agent's request — the same tool library and instructions on every single turn — while `messages` is where the actual, ever-changing conversation lives. Placing the volatile part last means the stable, expensive-to-regenerate part can be cached once and reused across an entire session, as long as nothing earlier in that hierarchy changes.

The pricing makes the incentive concrete. A cache write costs *more* than a normal request — *"5-minute cache write tokens are 1.25 times the base input tokens price"* — because the system does real extra work to store the entry. A cache read costs a small fraction of that — *"0.1 times the base input tokens price"* on most models, and as little as *"0.025x the base input price"* on some newer families. The economics only work out if the same prefix gets read many more times than it gets written — which is exactly the shape of a real agent loop: one tool library, one system prompt, dozens of turns.

## Repro 1: cache write, cache read, and the hierarchy under real load

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/kv-cache-economics/kv_cache_economics_docs.py:tool-library"
```

!!! success "A real run — four calls against Claude Sonnet 5, the same ~18-tool library and system prompt throughout"
    **Call 1** (cold cache): `cache_creation_input_tokens: 2279`, `cache_read_input_tokens: 0` — a real write, at the 1.25x price, for the full tools+system prefix.

    **Call 2** (identical tools+system, a different user question): `cache_read_input_tokens: 2279`, `cache_creation_input_tokens: 0` — an exact match to call 1's write size, read at the 0.1x price instead of paid for again in full.

    **Call 3** (same tools+system, `tool_choice` changed to `{"type": "any"}`): `cache_read_input_tokens: 2279` — **unchanged**. Anthropic's own documentation states plainly that *"Changing `tool_choice`"* invalidates only the *"Messages cache"* — this real run confirms the tools+system cache survives a `tool_choice` change completely untouched, exactly as documented.

    **Call 4** (same tools+system, but one tool's description edited by a single word — `" (updated)"` appended): `cache_creation_input_tokens: 2283`, `cache_read_input_tokens: 0` — **full invalidation**. The documented rule: *"Modifying tool definitions (names, descriptions, parameters) invalidates the entire cache."* A one-word change to one tool's description was enough to force a complete, fresh 2,283-token write — nothing from the identical system prompt or the 17 unchanged tools was salvaged.

    The contrast between call 3 and call 4 is the entire hierarchy, demonstrated with real numbers: a parameter that lives in the `messages`-adjacent layer (`tool_choice`) costs nothing extra when changed; a byte anywhere inside the `tools` array costs everything.

## Repro 2: tool masking vs. removal — a real, current beta feature

The obvious follow-up question: if editing the `tools` array invalidates everything, how do you ever change which tools are available mid-conversation without paying full price on every single turn from then on? Anthropic's answer, shipped as a real beta (`mid-conversation-tool-changes-2026-07-01`, expanded further under `inline-tools-2026-09-15`): don't edit the array. Declare the full tool set once, and control what's actually *offered* to the model through `tool_addition` and `tool_removal` content blocks on a mid-conversation `role:"system"` message instead. The documentation is direct about why this matters for caching specifically: *"The `tools` array itself never changes, so the cached prefix stays intact"* — because *"The `tools` array sits even earlier in the hashed request prefix than the top-level `system` field, so editing it invalidates the prompt cache for the entire conversation."*

```python
--8<-- "https://raw.githubusercontent.com/DhruvMakwana/agents-cookbook/main/kv-cache-economics/kv_cache_economics_docs.py:masking-vs-removal"
```

!!! success "A real run — masking vs. removal, against Claude Opus 5 (the beta isn't available on Sonnet 5)"
    **Call A** (baseline): `cache_read_input_tokens: 2211`. This is itself worth being honest about — it shows as a *read*, not a fresh write, because prompt-cache reads refresh the cache's TTL, and earlier development runs of this exact recipe against this exact tools+system prefix had already kept the entry warm. A genuinely cold first call to this same content would show a `cache_creation_input_tokens: 2211` write instead — the same real mechanism repro 1's call 1 demonstrated directly.

    **Call B1** — a `tool_removal` block withdraws `check_feature_flag` via a mid-conversation `role:"system"` message; the top-level `tools` array is byte-identical to call A's: `cache_read_input_tokens: 2211` — **identical to call A**. The masking directive cost nothing in cache terms.

    **Call B2** — the same conversation continued one more real turn (the actual assistant reply from call B1 replayed as history, plus a new question): `cache_read_input_tokens: 2211` — still identical. Masking isn't a one-call trick; it holds across the rest of the conversation.

    **Forced `tool_choice` on the masked tool** — attempting to force the model to call `check_feature_flag` by name, after it had been masked: a real `400` error, not a quiet no-op: `"tool_choice: forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back). Add it back via tool_addition, or change tool_choice."` The API itself enforces the removal — this isn't cosmetic, the tool is genuinely gone from what the model can be made to call.

    **Call C** (contrast — physically drop the tool from the top-level `tools` array instead, a 17-tool array rather than an 18-tool array with a masking directive layered on top): `cache_creation_input_tokens: 2113` — a **fresh, distinct** cache entry. Not a partial read, not a discount on the shared prefix — a completely new write, because the array itself is now different bytes.

    Both paths reach the identical functional outcome: the model cannot use `check_feature_flag` afterward. The cache bill is not identical. Three calls of masking read the same 2,211-token entry over and over; one call of removal wrote an entirely new 2,113-token entry that shares nothing with what came before it.

## What this means in practice

The general principle underneath both repros is the same: **the cache doesn't know what a change means, only where it happened.** A `tool_choice` change and a one-word tool-description edit are both, semantically, small adjustments — but one lives after the `tools`/`system` hash boundary and one lives inside it, so their real costs differ by an entire cache write. Masking a tool via `tool_removal` and physically deleting it from the array produce the identical *functional* result for the model — it can't call that tool either way — but one keeps the expensive prefix intact and one doesn't, purely because of which part of the request the change touches. Designing a cache-stable agent loop means internalizing this distinction concretely: anything that needs to vary turn-to-turn (which tools are currently offered, how the model should choose among them, what's in the conversation history) belongs after the hierarchy's expensive layers, or routed through a mechanism — like `tool_removal` — specifically built to make a real behavioral change without an array edit.

## Interview angle

**Weak answer** to "how would you make a long-running agent's tool use cheaper?": *"Use prompt caching on the system prompt."* True but incomplete — it treats caching as a single on/off switch, and doesn't explain what silently breaks it. A team that ships this understanding will confidently cache their system prompt, then remove a tool from the array between turns (a completely reasonable-looking thing to do) and never notice they just paid full price on every subsequent turn.

**Strong answer**: caching has a real prefix hierarchy (`tools` → `system` → `messages`), and the practical skill is knowing which layer a given change touches before making it — this page's own real numbers show a `tool_choice` change costing nothing extra (messages-level only) while a one-word tool-description edit costs a full re-write (invalidates everything after it in the hierarchy, including `tools` itself). For agents whose available tool set genuinely needs to change mid-conversation, the real, current answer isn't "accept the cache cost" — it's using a mechanism purpose-built to change *availability* without changing the *array*, like the `mid-conversation-tool-changes` beta's `tool_removal` block, which this page's own repro confirmed keeps the cache fully intact across multiple turns while still being a real, API-enforced restriction, not a cosmetic one.

**Follow-up to expect**: "if masking and removal reach the same functional outcome, why does the cache economics difference matter?" Because at agent-loop scale, the difference compounds every single turn. A 2,000+ token tools+system prefix re-written on every turn instead of read once and reused for the rest of a long-running session is exactly the kind of quadratic-feeling cost this whole track (Cost and Latency, a later Tier-2 topic) exists to name directly — and it's avoidable, for free, purely by choosing the mechanism that changes availability without changing bytes.

## Build it yourself — 30 minutes

1. Build a tools array with `cache_control: {"type": "ephemeral"}` on the last tool, large enough to clear your model's minimum cacheable length (1,024 tokens for Sonnet 5). Send the identical request twice and compare real `cache_creation_input_tokens` vs. `cache_read_input_tokens` — this page's own repro 1 is exactly this comparison.
2. Send a third, identical-prefix request but change only `tool_choice`. Confirm the tools+system cache read is unaffected — then send a fourth request with one tool's description edited by one word, and watch the entire cache invalidate.
3. If your model supports the `mid-conversation-tool-changes` beta, withdraw a tool with a `tool_removal` block instead of editing the `tools` array, and compare the real cache cost against physically removing it. Confirm the masked tool genuinely can't be forced via `tool_choice` afterward — a real API error, not just an assumption.
4. Watch for the TTL-refresh effect this page's own repro hit honestly: re-running the same cache-hitting request during development keeps the entry warm, so a "second run" of your own test may show a read where you expected to see a fresh write. That's real caching behavior working correctly, not a bug in your test.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: KV-Cache Economics">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A real repro sent four calls with an identical, cached tools+system prefix. Call 3 changed only tool_choice (same tools, same system prompt) and still showed cache_read_input_tokens unchanged from call 2. Call 4 edited one word in one tool's description and showed a full cache_creation_input_tokens write instead.",
      "question": "What's the most accurate explanation for why these two small changes had such different real costs?",
      "options": [
        "The tools array sits earlier in the real cache prefix hierarchy than tool_choice touches",
        "Call 4 happened later in the session, and caches naturally degrade over elapsed time",
        "Editing a tool description is a larger change in total byte count than changing tool_choice",
        "Call 4 happened later in the session, and caches naturally degrade with time passing"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The cache follows a fixed hierarchy (tools -> system -> messages), and a change only invalidates the level it occurs at plus everything after it. tool_choice affects the messages level, downstream of tools/system, so it leaves the tools+system prefix's hash untouched. A tool description lives inside the tools array itself, upstream of everything, so changing it invalidates the whole chain.",
        "Contradicts the real, documented mechanism directly -- prompt caches don't 'naturally degrade'; they persist for their TTL (unless invalidated by a structural change) and are refreshed on each read. Elapsed time within a session isn't what caused call 4's invalidation.",
        "Not the real mechanism -- a single appended word (' (updated)') is a tiny byte-count change, smaller than many tool_choice payloads could be, yet it caused full invalidation; the deciding factor is WHERE in the hierarchy the byte changed, not how many bytes changed.",
        "Overgeneralizes -- tool_choice itself doesn't invalidate the tools/system cache, but this doesn't mean EVERY possible combination of changes alongside it would also be free; the real claim is narrower and specifically about which cache LEVEL it touches."
      ]
    },
    {
      "scenario": "A real repro compared two ways of making a tool functionally unavailable to the model mid-conversation: (1) a tool_removal content block on a mid-conversation role:'system' message, with the top-level tools array left byte-identical, and (2) physically deleting the tool from the top-level tools array. Both produced the same real outcome -- the model could no longer use that tool.",
      "question": "Given the identical functional outcome, what was the real, measured difference between the two approaches?",
      "options": [
        "Array-editing is faster because it requires fewer tokens to express than a tool_removal block",
        "There was no real difference at all -- both approaches are equivalent in every measurable way",
        "Only the tool_removal approach actually works; array-editing silently fails to remove access",
        "tool_removal kept reading one identical cache entry; array-editing produced a distinct entry"
      ],
      "correct": 3,
      "explanations": [
        "Not what was measured or claimed -- the comparison in this repro was about cache token accounting (creation vs. read), not about raw request size or response latency; token-count-for-expressing-the-change was not the dimension being compared.",
        "Directly contradicted by the real measured numbers -- the two approaches produced very different cache_read/cache_creation values across the actual repro, not identical ones.",
        "Contradicts the real repro directly -- the array-editing call (call C) genuinely worked; it produced a valid response with a distinct, freshly-created cache entry (2,113 tokens), not a silent failure. Both approaches genuinely worked functionally; they differed in cache cost, not in whether they worked.",
        "Correct. Three calls using tool_removal (with the tools array unchanged) all read the identical 2,211-token cache entry. The call that instead physically edited the tools array produced a completely new, distinct 2,113-token entry with no relationship to the earlier ones -- same functional outcome, very different cache economics."
      ]
    },
    {
      "scenario": "After masking a tool with a tool_removal block, a real repro attempted to force tool_choice to that exact tool by name. The real API response was a 400 error: \"forced tool 'check_feature_flag' is absent from the final available-tool set (a tool_removal block removed it without a later add-back).\"",
      "question": "What does this specific result establish about tool masking that the cache-preservation numbers alone don't?",
      "options": [
        "That masking is purely a description-level change with no effect on actual tool availability",
        "That the masked tool remained fully callable, and the error was an unrelated bug",
        "That masking is enforced by the API itself, not just a cosmetic hint to the model",
        "That forcing tool_choice is generally unsupported whenever a beta header is active"
      ],
      "correct": 2,
      "explanations": [
        "The opposite of what the real error demonstrates -- if masking were purely cosmetic, forcing tool_choice to the masked tool would have succeeded (the model would just be told to call something it could still technically invoke); instead the API itself rejected the request outright.",
        "Directly contradicts the quoted real error message, which explicitly states the tool is 'absent from the final available-tool set' -- not callable, and the error is specifically about that absence, not an unrelated fault.",
        "Correct. A real, specific 400 error tied directly to the tool_removal block's effect shows the API is actively tracking and enforcing which tools are genuinely available -- this is a hard access-control result, not just Claude choosing not to mention the tool in its own responses.",
        "Unsupported and too broad -- nothing in the real error message suggests forced tool_choice is broken generally under beta headers; the rejection was specific to the named tool having been removed, not a general incompatibility."
      ]
    },
    {
      "scenario": "A developer re-runs kv_cache_economics.py twice in a row during development, a few minutes apart, against the identical tools+system prefix. The second run's first call shows cache_read_input_tokens instead of the cache_creation_input_tokens they expected for a 'fresh' first call.",
      "question": "What's the most accurate explanation for this observation?",
      "options": [
        "This indicates a bug in the recipe -- a fresh script run should always start with a cache miss",
        "Cache reads refresh the entry's TTL, so an identical prefix tested repeatedly stays warm",
        "The Anthropic API caches responses indefinitely once written, regardless of any TTL",
        "The second run used a different model than the first, which explains the cache hit"
      ],
      "correct": 1,
      "explanations": [
        "Misdiagnoses the cause -- 'fresh script run' and 'fresh cache state' are different things; the cache is scoped to the exact request content and lives server-side independent of when or how many times a local script has been invoked.",
        "Correct. This is real caching behavior, not a bug -- prompt cache entries have a TTL (5 minutes by default), and reading an entry resets that window. Testing the identical prefix repeatedly during development, within that window, keeps extending its life, so a 'first' call in a later run can legitimately read an entry created by an earlier run or test.",
        "Contradicts the documented mechanism directly -- prompt caches are explicitly time-limited (5-minute or 1-hour TTL options), not indefinite; an entry does expire if enough real time passes without being read.",
        "Not the described scenario -- the setup specifies an identical tools+system prefix on both runs; a genuine model change would itself normally produce a DIFFERENT cache entry (as this page's own repro observed between Sonnet 5 and Opus 5 runs), not the same one being read."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

- Anthropic, ["Prompt caching"](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — cache write/read cost multipliers, TTL options, minimum cacheable lengths, breakpoint mechanics, and the `tools → system → messages` invalidation hierarchy.
- Anthropic, ["Tool use with prompt caching"](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-use-with-prompt-caching) — `cache_control` placement on tool definitions, `defer_loading` and cache preservation, and the per-change invalidation table this page's repro 1 verified directly.
- Anthropic API release notes — the `mid-conversation-tool-changes-2026-07-01` beta (2026-07-24) and its `inline-tools-2026-09-15` expansion (2026-09-22): `tool_addition`/`tool_removal` blocks on mid-conversation `role:"system"` messages.
- [Tools at Scale](tools-at-scale.md) — `defer_loading` and tool search, a complementary technique for keeping large tool libraries out of context and cache-stable at the same time.
- [Context Engineering](context-engineering.md) — the "select" and "compress" context-engineering verbs this page's cache-stability principle is a specific, measurable instance of.
