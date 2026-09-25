# Cognition's Devin: The Benchmark and the Demo

Cognition's Devin, launched in 2024 as one of the earliest products marketed as an "AI software engineer," is a useful case study for a reason that has nothing to do with whether it's good or bad — it's one of the clearest real examples of the gap between a company's own disclosed benchmark number and the same company's own promotional material, both from primary sources, both worth reading side by side.

## The real, disclosed benchmark number

Cognition's own SWE-bench technical report gives a specific, honestly-scoped result: on a 570-issue sample (roughly 25% of the full SWE-bench set), Devin resolved **13.86%** of issues unassisted. For comparison, Cognition's own report states the prior unassisted baseline was **1.96%**, and the prior best *assisted* system (meaning a human working with model help, not a fully autonomous agent) reached **4.80%**. Given real unit tests upfront, Devin's own pass rate rose to **23%**. Cognition's own report includes a real, disclosed caveat most companies wouldn't volunteer: since the underlying model's training data likely included these public repositories, there's a genuine data-contamination risk inflating the apparent number — stated by Cognition itself, not extracted from them.

## The demo controversy

Separately, Cognition's own promotional material — a video showing Devin completing a real freelance job posted on Upwork — was independently, publicly scrutinized by a software engineer publishing under "Internet of Bugs." The analysis found that Cognition's video *"fed only the first sentence"* of the actual job posting to Devin, while the real requirements were contained in a second part of the posting the video never showed being given to the agent. Separately, the specific bug the video shows Devin "fixing" was one Devin had introduced itself earlier in the same session — not a pre-existing issue in the codebase, as the video's framing implied. The analysis's own conclusion characterized the video as edited to *"put Devin in the best light,"* not a neutral demonstration of an unscripted task.

## Reading both together

Neither of these facts erases the other. The SWE-bench number is real, self-disclosed, and includes an honest caveat about its own limitations — a genuinely more transparent disclosure than most companies give around agent benchmarks. The demo critique is also real, independently sourced, and specific about exactly what was misleading and how. The two together are the actual, complete picture: a real, non-trivial capability improvement over prior baselines, marketed through promotional material that, independently verified, didn't hold up to the same standard of honesty as the benchmark report sitting right next to it.

!!! success "The lesson"
    A company's own benchmark report and a company's own promotional demo are not the same kind of evidence, even when they're about the identical product — one is a disclosed, methodologically-scoped measurement (with Cognition's own report naming its own contamination risk), the other is marketing, optimized to look impressive rather than to disclose limitations. [Benchmark Atlas](../benchmark-atlas.md) makes the general case for reading what a benchmark actually measures before trusting a headline score; this case study is the concrete instance of the companion skill — reading a demo video with the same scrutiny, checking what was and wasn't shown, before trusting that a promotional clip represents an unscripted, complete task the way an actual benchmark run does.

## Sources

- [Cognition: "SWE-bench Technical Report"](https://cognition.com/blog/swe-bench-technical-report)
- [80.lv: "First AI Software Engineer' Creators Are Accused of Lying"](https://80.lv/articles/first-ai-software-engineer-creators-are-accused-of-lying)
