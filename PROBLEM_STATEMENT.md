# Problem Statement

## Who is the user?
Students, researchers, and knowledge workers who summarize a large volume of papers, articles, and reports using AI tools.

## What is the pain point?
LLM summaries are fast but **untrustworthy**: they routinely introduce claims, numbers, or conclusions that are *not present in the source* (hallucination), or overstate hedged findings. The reader has no way to tell which sentences are grounded and which are invented - so the only safe option is to re-read the original, which defeats the purpose of summarizing.

## Why does it matter?
Summaries get copied into literature reviews, decision memos, and study notes. A single hallucinated statistic can propagate into real decisions. The risk is highest exactly when the user is *least* able to verify - long or unfamiliar sources.

## What output should the workflow produce?
A **verified summary** consisting of:
1. A concise 3-5 sentence summary (built map-reduce style so it works on long documents, not just short ones).
2. A claim-by-claim verification table: each extracted claim labelled `SUPPORTED` / `PARTIAL` / `UNSUPPORTED`, with a quoted piece of source evidence and a confidence value.
3. An overall **trust score** (% of claims supported) plus a **confidence-weighted** score.
4. A **status** reflecting one of three routing tiers: `AUTO_PUBLISHED` (high trust), `AUTO_PUBLISHED_FLAGGED` (light review - auto-published but logged for spot-checking), `APPROVED_BY_REVIEWER`, or `NEEDS_REVISION` (after the human gate).

This output is emailed, logged to a Google Sheet (audit trail), and shown back to the submitter.

## Why a workflow and not a single prompt?
A single prompt that "summarizes and self-checks" is circular - the same model that hallucinated the claim is trusted to catch it. The value comes from **separation of roles** (an independent verifier), **deterministic scoring/thresholds** the model can't fudge, and a **human gate** for low-trust output. Those are workflow concerns, not prompt concerns.

## Success criteria
- Hallucinated/unsupported claims are flagged, not silently published.
- High-trust summaries flow through automatically (no human bottleneck).
- Every run leaves an auditable record.
