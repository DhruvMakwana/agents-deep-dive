# Interview Playbook

!!! info "What this page is"
    Not a code recipe — an evidence-graded synthesis of real agent-interview material: candidate reports, company-published process changes, job-description term frequency across 12,977 live postings, and a curated question set, each item tagged with exactly how well-sourced it is. Every "asked at X" claim on this page was tied to that company by a source actually read and quoted — no claim is carried from a listicle's untraceable "Asked at" tag.

??? abstract "TL;DR — quick revision"
    - **The single strongest dated trend in the evidence: AI-assisted coding is becoming the interview format itself, not just the topic.** OpenAI runs a beta onsite round done "using an AI coding agent." Meta "added an AI-assisted coding round" where a small toy codebase is debugged and completed with LLM help. Sierra rebuilt its entire onsite around a 2-hour build session using any AI tooling, and removed its classic coding/algorithms interview outright. Anthropic, by contrast, explicitly disallows AI tools in its live rounds — the field hasn't converged on one answer.
    - **Most of what's published as "agent interview questions" is prep-market content, not candidate-confirmed.** Of 106 collected questions, only 14 trace to a first-person candidate report or a company's own published process; 81 come from prep sites, listicles, and openly "synthesized" repositories. Treat frequency in prep material as evidence of what the prep industry teaches, not what interviewers actually ask.
    - **A real loop can change within a year.** Sierra's own May-2025 candidate reports describe a take-home support agent plus a TypeScript/React debugging round; Sierra's own April-2026 blog post describes a completely different AI-native onsite. Prepping from a year-old write-up can mean prepping for a loop that no longer exists.
    - **Job descriptions and interview reports disagree on what to emphasize.** Kubernetes appears in 19.4% of AI-tech postings and in zero candidate interview reports reached. Idempotency and durable execution show up across multiple prep sources and real interview questions but in only 1.1% of postings. Evals is the rare topic strongly present in both.
    - **The one topic with genuine candidate confirmation across multiple independent sources: production reliability under real failure, not textbook agent architecture** — Swiggy's real four-question loop was entirely about uncertainty at scale, agent evals, LLM-as-judge, and not hallucinating success after a failed tool call; Sierra's real rounds are build-and-debug tasks, not "design an agent" whiteboard questions.

## How this evidence is graded

Every item on this page carries two tags. **Quality** describes the source type: `candidate-reported` (a first-person account of an actual interview), `company-published` (the company's own description of its process), `prep-site` (a hiring guide or study resource, not a report of a real loop), or `listicle` (a generic or influencer question list). **Evidence** describes how directly it was verified: `VERIFIED-PRIMARY` means the source was loaded and quoted directly; `VERIFIED-SECONDARY` means a distinctive phrase was confirmed present on a loaded page, but the page itself is prep material, not a report; `UNVERIFIED` means the source could not be loaded this pass (a bot wall, a paywall, a dead link) and the claim is carried forward with that caveat attached, not silently dropped or silently trusted. A company name is attached to a question **only** when a loaded source explicitly ties that question to that company — the "Asked at OpenAI, Anthropic, Google..." tags common on prep sites, with no per-question citation, are treated as marketing and never repeated here as fact.

## The shift to AI-assisted coding rounds

This is the trend with the most dated, independent, cross-company evidence in the entire corpus. OpenAI's onsite includes a beta round: "you're given a prompt, an existing codebase, and a problem too complex to solve by writing code from scratch... the expectation is that you work through them using an AI coding agent" — and, notably, "AI use in OpenAI interviews is strictly prohibited, except in the beta agentic coding interview," meaning this is a deliberate, scoped exception to an otherwise strict no-AI policy. Meta, independently: "Meta added an AI-assisted coding round... you get a much smaller toy codebase plus unit tests, and you're allowed to use an LLM to debug and complete the implementation" — and the candidate reporting this adds a genuinely useful data point about what's actually graded: "I don't think my interviewer cared how much code was written by AI vs. me. They cared more about how well I partnered with AI." Sierra went further than adding a round — it restructured its entire onsite: "We removed our coding and algorithms interviews and replaced them with an AI-native onsite," built around a candidate building a real feature over two hours "using the AI tooling and frameworks of their choice," followed by a review where the interviewer "dig[s] into how they used AI along the way."

This isn't universal. Anthropic's own candidate-reported rounds run the other way: "Anthropic ironically doesn't allow the use of AI tools unlike google or meta" — its live rounds stay practical system-building and infrastructure-focused system design, with a separate behavioral round specifically on AI safety alignment. The honest read: whether AI-assisted coding is *allowed*, *one round among several*, or *the entire onsite* is company-specific and worth asking about directly rather than assuming from a single company's practice.

## What's actually candidate-confirmed, vs. what the prep industry pushes

Of 106 collected questions across this research pass, only 14 trace to an actual first-person report or a company's own published process — 11 distinct real sources in total. The other 81 come from prep sites, listicles, and repositories, several of which are explicit about not being real: one popular question-bank repository states outright, on every company's page, that its questions are "synthesised from this company's publicly known focus areas and role descriptions — not leaked questions." Several prep sources also aren't independent of each other — one widely-shared article reproduces another hiring blog's three example prompts nearly verbatim, so counting them as two separate confirmations would overstate the evidence.

The honest gap, stated plainly: topics like loop termination, agent memory design, human-in-the-loop patterns, multi-agent delegation, and MCP appear across many prep sources with zero candidate-level confirmation in the sources reachable this pass — not proof these are never asked, but a real gap between what's taught and what's confirmed. The topics that *do* have multi-source candidate confirmation are narrower and more concrete than most prep material suggests: production reliability and grounding after tool failure (Swiggy, one detailed real report), and build-or-debug-an-agent-codebase as the task format itself (Sierra, across three independent sources; OpenAI and Meta for AI-assisted coding specifically).

## Job descriptions vs. interview reports

A same-day snapshot of 12,977 live postings across 107 company job boards (Greenhouse, Ashby, Lever), filtered to 1,978 genuinely AI-technical roles across 96 companies, gives a real, current picture of what companies say they want — which is a different question from what gets asked live in a loop. Some real gaps between the two:

- **Named in job descriptions, absent from candidate interview reports reached**: Kubernetes (19.4% of AI-tech postings), Go (14.1%), PyTorch (11.4%), reinforcement learning (11.0%), MCP (9.4%, across 61 distinct companies), LangChain (8.1%), guardrails (11.1%).
- **Present across interview material, rare in job descriptions**: loop termination and runaway-agent handling (no tracked JD term at all, yet appears across 5 independent prep sources); idempotency and durable execution (only 1.1% of postings, but present in real interview questions like "how do you make sure agents do not double-execute side-effectful operations" and in Bhavishya Pandit's widely-shared 25-question list); prompt injection (1.8% of postings, 9 prep sources).
- **Strong in both**: evals — 20.9% of AI-tech postings across 68 distinct companies, and the one topic with genuine candidate confirmation (Swiggy's real "how do you evaluate if your agent is actually working correctly?").

A caution worth repeating explicitly: this JD data is one snapshot on one date. It shows a current baseline, not a trend — a claim like "MCP mentions are rising" would need a second snapshot months apart to actually support, which this page doesn't have.

## Real system-design prompts, with provenance

Every prompt below is shown with exactly how it's sourced — several widely-repeated "design an agent" prompts turn out to come from hiring-side guides, not from any candidate's actual report of being asked them.

| Prompt | Source | Provenance |
|---|---|---|
| Build a real feature over 2 hours using any AI tooling you choose, then demo and defend your choices | Sierra's own blog | company-published |
| Build a small customer-support agent (take-home); be ready to discuss prioritization, metrics, observability | Sierra Agent Engineer candidate report | candidate-reported |
| Debug a small (~4-5 file) agent codebase against a reference diagram showing intended behavior — find the ~3 planted bugs | Sierra Software Engineer (Agent) candidate report | candidate-reported |
| Take an existing codebase and a problem too large to solve by hand, and work it to completion using an AI coding agent | OpenAI onsite (beta round) | insider-sourced prep page |
| Design an autonomous agent loop: reliability compounding across steps, durable execution and replay, the tool-call contract, working vs. long-term memory, prompt injection arriving through tool output, quadratic cost growth, stuck-loop detection | Prep-site guide | prep-site (no candidate confirmation) |
| Design a ticket-resolving support agent with escalation logic and an eval framework distinguishing "resolved" from "deflected" | Hiring-side blog example prompt | prep-site (hiring-side blog) |
| Design a system where multiple agents collaborate to produce a cited research report | Same hiring-side blog | prep-site (hiring-side blog) |

## A short, evidence-graded set to practice with

A deliberately small, high-confidence selection — not the full 106-question bank, but enough to cover the topics with the strongest real backing, each with its actual quality and evidence tag shown:

1. *"How would you solve LLM uncertainty at millions of users scale?"* — **Swiggy, AI Engineer** · candidate-reported · VERIFIED-PRIMARY
2. *"How do you evaluate if your agent is actually working correctly?"* — **Swiggy, AI Engineer** · candidate-reported · VERIFIED-PRIMARY
3. *"How do you stop your agent from hallucinating a successful order when the tool call failed?"* — **Swiggy, AI Engineer** · candidate-reported · VERIFIED-PRIMARY
4. Debug a small, unfamiliar agent codebase against a reference diagram of intended behavior — **Sierra, Software Engineer (Agent)** · candidate-reported · VERIFIED-PRIMARY
5. Build a small customer-support agent as a take-home, then defend prioritization, metrics, and observability choices in a follow-up round — **Sierra, Agent Engineer** · candidate-reported · VERIFIED-PRIMARY
6. *"How do you make sure agents do not double-execute side-effectful operations like charging a card or booking a ticket twice?"* — influencer question list · listicle · VERIFIED-SECONDARY
7. *"Agents can sometimes loop or take unnecessary steps. How do you design to avoid runaway behavior?"* — influencer question list · listicle · VERIFIED-SECONDARY
8. *"What different strategies could you use for long-term memory?"* — hiring-side blog · prep-site · VERIFIED-SECONDARY
9. Implement multi-head attention from scratch, no `nn.MultiheadAttention` — handle KV cache and grouped-query attention in the same hour — **Anthropic** (for the KV cache/GQA detail specifically) · candidate-reported (anonymous) · VERIFIED-PRIMARY
10. *"You have a prototype agent that works for 10 requests a day. How would you redesign it for 10,000 requests a day?"* — influencer question list · listicle · VERIFIED-SECONDARY

## Interview angle

**Weak answer** to "how should I prepare for an agent interview": *"Study the top 20 agent interview questions from [prep site]."* This treats every question on a prep list as equally likely to be asked, when this page's own evidence shows the opposite — 81 of 106 collected questions have no candidate confirmation at all, several widely-shared lists aren't independent of each other, and the one company with a detailed real report (Swiggy) asked something narrower and more production-focused than most prep material emphasizes.

**Strong answer**: apply the same evidence grading to your own prep that this page applies to its sources. Weight real candidate reports and company-published process descriptions over prep-site frequency — and when you only have prep-site material, say so to yourself honestly rather than treating repetition across five similar-looking listicles as five independent confirmations, especially when (as documented here) some of them are directly copying from each other. Separately, ask directly in a recruiter screen whether AI tools are allowed in the technical rounds — this page's own evidence shows real, current, company-specific disagreement on that exact question (OpenAI's scoped exception, Meta's dedicated round, Sierra's full restructure, versus Anthropic's explicit prohibition), and assuming the wrong answer wastes real prep time.

**Follow-up to expect**: "isn't 14-out-of-106 candidate-confirmed a really thin evidence base to build a prep strategy on?" Yes, honestly — and that thinness is itself the finding worth taking seriously, not a flaw to paper over. Reddit, Glassdoor, 1point3acres, and most paywalled experience reports were unreachable in this research pass, so the real gap between "confirmed" and "never asked" is almost certainly narrower than 14-vs-81 suggests. The correct response to that uncertainty isn't to fall back on treating every prep-site question as equally credible — it's to prioritize preparation on the topics with real confirmation (production reliability, evals, build-or-debug-an-agent-codebase, and the AI-assisted-coding format itself), and treat everything else as plausible but unconfirmed, the same distinction this page draws throughout.

## Scenario Check

<div class="quiz-widget" data-title="Scenario Check: Interview Playbook">
<script type="application/json">
{
  "questions": [
    {
      "scenario": "A prep repository's README states that its per-company interview question lists are 'synthesised from this company's publicly known focus areas and role descriptions — not leaked questions,' the same disclosure appearing on every company's page in the repo.",
      "question": "What's the most accurate way to use this repository's content in interview prep?",
      "options": [
        "As a topic map of likely focus areas, not evidence any specific question was asked",
        "As a real record of confirmed questions, since dozens of named companies are covered",
        "As equally credible as a first-person candidate report of the same company",
        "As entirely worthless and not worth reading, given the synthetic disclosure"
      ],
      "correct": 0,
      "explanations": [
        "Correct. The repo's own stated purpose fits this use -- a synthesized list built from public focus areas is genuinely useful for understanding what a company likely emphasizes, without the false confidence of treating it as a report of real questions.",
        "Contradicts the source's own explicit disclosure -- the repository states plainly that its content is synthesized, not leaked or reported, which is the opposite of a confirmed record.",
        "Directly contradicts the graded distinction this page draws between quality tiers -- 'synthetic' and 'candidate-reported' are meaningfully different levels of evidence precisely because one describes a real event and the other doesn't.",
        "Overreacts to the disclosure -- content being synthesized rather than leaked doesn't make it valueless as a topic guide; it just shouldn't be mistaken for a report of an actual interview."
      ]
    },
    {
      "scenario": "Sierra candidate reports from May 2025 describe a take-home support agent build plus a TypeScript/React debugging round. Sierra's own company blog post from April 2026 describes a restructured onsite: a 2-hour AI-native build session, with the coding phone screen replaced by system design.",
      "question": "What's the most accurate conclusion a candidate preparing today should draw from having both sources?",
      "options": [
        "Neither source is trustworthy, since the discrepancy means Sierra's process can't be prepared for",
        "The 2026 company post describes the current process; the 2025 reports describe a loop that changed",
        "Both sources are equally current, since candidate reports are generally more reliable than company blogs",
        "The 2025 reports should be trusted over the 2026 post, since candidate reports outrank company statements"
      ],
      "correct": 1,
      "explanations": [
        "Overcorrects into unwarranted skepticism -- a documented change over time isn't a sign either source is unreliable; it's exactly the kind of dated evidence this page uses to show loops can change.",
        "Correct. The company's own dated, published description of a restructured process is the more current evidence -- the older candidate reports describe a real loop, just one the company's later post indicates has since changed.",
        "Ignores the actual dates -- treating an 11-months-earlier candidate report as equally current as the company's own later description of a changed process risks preparing for a round (coding phone screen) that no longer exists.",
        "States an absolute rule not supported by the material -- source type alone doesn't determine reliability; recency and specificity matter here."
      ]
    },
    {
      "scenario": "A candidate notes that MCP appears in 9.4% of AI-tech job postings across 61 distinct companies, but has zero candidate-confirmed interview questions in the reachable evidence. They conclude: 'Since MCP isn't confirmed in any real interview report, it's safe to skip preparing for it.'",
      "question": "What's the strongest problem with that conclusion?",
      "options": [
        "The conclusion is reasonable -- no candidate confirmation means the topic is genuinely unlikely to come up",
        "MCP should be deprioritized in favor of any topic with even one narrow candidate confirmation",
        "61 companies naming MCP is relevance evidence; the gap reflects unreachable sources, not that it's unasked",
        "JD frequency and candidate confirmation should always align, so the mismatch means the JD data is unreliable"
      ],
      "correct": 2,
      "explanations": [
        "Draws too strong a conclusion from an acknowledged evidence gap -- this page is explicit that the candidate-confirmed sample is thin due to unreachable platforms, not a complete record of what's asked.",
        "Overcorrects into a rule that ignores confirmation strength -- a topic confirmed once, narrowly, isn't automatically more prep-worthy than one present across dozens of current job descriptions.",
        "Correct. The JD data is real, current evidence of relevance even without a matching interview report, and this page states the candidate-confirmed gap likely understates real coverage since major platforms were unreachable -- absence of confirmation isn't evidence of absence.",
        "Misapplies the data -- job descriptions and live interview content answer different questions (what a role needs long-term vs. what's tested live), which is why this page tracks them separately, not a sign either dataset is broken."
      ]
    },
    {
      "scenario": "A candidate reads that Meta's AI-assisted coding round grader reportedly said 'I don't think my interviewer cared how much code was written by AI vs. me. They cared more about how well I partnered with AI.' They also read that Anthropic's live rounds explicitly disallow AI tools. They conclude: 'AI tool policy must be consistent across all technical companies, so one of these two reports must be wrong.'",
      "question": "What's the most accurate response to this reasoning?",
      "options": [
        "Correct -- since the two policies contradict, at least one source must be inaccurate or outdated",
        "Meta's policy must be the outdated one, since AI-assisted rounds are the newer, rising trend",
        "The two reports are compatible if Anthropic only disallows AI tools in non-technical rounds",
        "Each policy is independently documented and genuinely company-specific, not a contradiction"
      ],
      "correct": 3,
      "explanations": [
        "Assumes an industry-wide standard not supported anywhere in the material -- a real, documented split (OpenAI's scoped exception, Meta's dedicated round, Sierra's restructure, Anthropic's prohibition) is the actual finding, not a contradiction to resolve.",
        "Fabricates a directional claim -- nothing establishes Anthropic's policy is 'outdated' rather than a deliberate, current choice; the rising-trend framing describes AI-assisted coding's spread, not that holdouts are behind.",
        "Invents an unsupported qualification -- the Anthropic quote describes live rounds generally, with no technical/non-technical distinction stated.",
        "Correct. Each company's AI-tool policy is independently documented from its own source -- genuine variation across companies is exactly what the evidence shows, not an error needing resolution."
      ]
    }
  ]
}
</script>
</div>

## Sources & further reading

This page synthesizes a dedicated research pass (candidate reports, company-published process descriptions, and a same-day snapshot of 12,977 live job postings across 107 company boards via the Greenhouse, Ashby, and Lever APIs) rather than citing a small fixed set of external posts. The individual sources behind every claim above — LinkedIn posts, Blind threads, company engineering blogs, and job-board API responses — are cited inline by name and role in the text, exactly as read and quoted, per this project's own standing rule that every number and quote is re-verified at writing time and carries its own confidence grade.
