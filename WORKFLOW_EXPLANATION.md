# Workflow Explanation - AI vs Deterministic, branch by branch

The central design decision: **AI is used only where open-ended language reasoning is unavoidable; everything that can be a rule, is a rule.** This makes the system predictable, debuggable, and cheaper, and it is what separates this from "a chatbot." Of the 28 nodes, exactly **three** call a model - the other 25 are deterministic.

## Where AI is used (and why it *must* be AI)

| Step | Role | Why a deterministic rule can't do it |
|------|------|--------------------------------------|
| **Summarizer Agent** *(map)* | Generator, per chunk | Compressing arbitrary prose into a faithful summary and isolating atomic factual claims is open-ended natural-language generation. No rule can do it. Runs once per chunk. |
| **Synthesize Summary** *(reduce)* | Editor / merger | Merging several partial summaries and overlapping candidate claims into one coherent, deduplicated summary requires judgement about meaning and redundancy. |
| **Verifier Agent** | Independent critic | Judging whether a claim is *entailed* by a passage is semantic reasoning (paraphrase, negation, scope). Regex/string-match would miss paraphrases and false-match on keywords. |

The agents are **separate nodes with separate prompts and separate roles**. Crucially, the Verifier is a *different* role from the generators: its prompt states it did *not* write the summary and must be skeptical. This is the core "agentic role decomposition" - a model is bad at catching its own hallucinations, so an independent pass with a different objective does the catching. Splitting generation into **map (Summarizer) + reduce (Synthesizer)** is what lets the system handle documents far longer than a single context-friendly prompt.

Every agent is constrained to **structured JSON output** so downstream deterministic nodes can act on it reliably.

## Where deterministic logic is used (and why it *must not* be AI)

| Step | Why it's deterministic |
|------|------------------------|
| **Has URL? / Retry Synthesis? / Route by Trust Tier** | Routing and looping must be 100% predictable and auditable. |
| **Fetch + Extract Text** | HTTP + HTML stripping is mechanical; AI here would be wasteful and unreliable. |
| **Validate & Prepare Input** | A hard length check is a guarantee, not a judgement call. |
| **Chunk Source** | Splitting text at fixed overlapping offsets is pure arithmetic - the **map** fan-out. |
| **Parse Chunk Summary / Parse & Validate Synthesis** | JSON parsing/repair + schema checks must be exact, with deterministic fallbacks if a model returns junk. |
| **Merge Chunk Summaries** | The **reduce** prep - dedup and concatenation - is set logic, not reasoning. |
| **Score & Decide Route** | The trust score and the tier thresholds are **policy**. Policy must be code so it can't drift run-to-run and can be reviewed/changed deliberately. |
| **Build Final Report / Sheet / Email** | Formatting and delivery are mechanical. |

**Rule of thumb applied throughout:** AI proposes (summaries, verdicts); deterministic code disposes (chunking, scores, thresholds, routing, the retry decision, delivery). The model never decides *what happens next* - it only produces content that the rules then evaluate.

## The map-reduce summarization

1. **Chunk Source** splits the validated text into ~4,000-char chunks with a 200-char overlap and emits one item per chunk.
2. **Summarizer Agent** runs once per chunk (n8n executes a node per input item), producing a per-chunk summary + claims.
3. **Merge Chunk Summaries** gathers every chunk's output with `$input.all()`, dedups claims by normalized text, and concatenates the partial summaries.
4. **Synthesize Summary** reduces all of that into one final 3-5 sentence summary and a canonical 4-8 claim set.

A short article produces a single chunk and still flows through the same path - the design is uniform, not special-cased.

## The self-correction loop

`Parse & Validate Synthesis` parses the Synthesizer's JSON. If it is unusable it sets `needs_retry = true`, but **only on the first attempt** - it reads n8n's `$runIndex` (0 on the first pass, 1 after one retry) as a built-in, tamper-proof attempt counter. `Retry Synthesis?` (IF) then either loops back to `Synthesize Summary` (whose prompt detects the retry via `$runIndex` and becomes stricter) or proceeds. After the single retry budget is spent, the parse node falls back to a deterministic claim set built from the merged candidate claims - the run never hard-fails here. The loop is therefore bounded to at most two synthesis calls.

## Branches

1. **Input branch - `Has URL?`**: URL → fetch + extract; otherwise use pasted text. Both converge on the same validation node, so the rest of the workflow is source-agnostic.
2. **Self-correction branch - `Retry Synthesis?`**: loops on invalid synthesis JSON (bounded by `$runIndex < 1`), else proceeds.
3. **Trust branch - `Route by Trust Tier`** (Switch, 3-way) driven by the deterministic policy in `Score & Decide Route`:
   - `unsupported > 0` **or** `weighted_score < 70` → **HARD_REVIEW** (human-in-the-loop; Gmail *Send & Wait* pauses the run and emails Approve/Reject buttons - the reply resumes it).
   - else `trust_score < 90` **or** any PARTIAL **or** any low-confidence SUPPORTED → **LIGHT_REVIEW** (auto-publish, but stamped `AUTO_PUBLISHED_FLAGGED` in the audit log).
   - else → **AUTO_PUBLISH** (no human in the path, no bottleneck).
4. **Decision branch - after HARD_REVIEW**: `Apply Reviewer Decision` reads the approval boolean and stamps `APPROVED_BY_REVIEWER` or `NEEDS_REVISION`.

All branches reconverge at **Build Final Report**, so there is exactly one reporting/logging path regardless of route.

## Trust scoring (confidence-weighted)

```
trust_score    = round(100 * supported / total)                       # simple supported ratio
weighted_score = round(100 * Σ(conf · w) / total)                     # w = 1.0 SUPPORTED, 0.5 PARTIAL, 0 UNSUPPORTED
low_conf       = any SUPPORTED claim with confidence < 0.5
```

Using both a plain ratio and a confidence-weighted score means a summary full of low-confidence "supported" claims can't sail through on the headline number - it drops to LIGHT_REVIEW or HARD_REVIEW.

## Fallback / error handling

- **Bad URL:** `Fetch URL Content` uses `neverError`/`continueOnFail` → flows on instead of crashing; the validation node then catches empty content.
- **Too-short / missing input:** `Validate & Prepare Input` throws an explicit `FALLBACK:` error with a user-actionable message.
- **Malformed chunk JSON:** `Parse Chunk Summary` is lenient - a bad chunk contributes no claims rather than failing the whole run.
- **Malformed synthesis JSON:** the self-correction loop retries once, then falls back to deterministic claims built from the merged candidates.
- **Unusable Verifier output:** `Score & Decide Route` defaults every claim to `UNSUPPORTED`, which forces the safe path - HARD_REVIEW - rather than silently passing.
- **Gmail/Sheets unavailable:** those nodes `continueOnFail`, so the report is still produced and shown on the completion page.

## Structured output schemas

**Summarizer (per chunk) →**
```json
{ "chunk_summary": "string", "claims": [ "string" ] }
```
**Synthesizer →**
```json
{ "title": "string", "summary": "string", "key_claims": [ { "id": 1, "claim": "string" } ] }
```
**Verifier →**
```json
{ "verifications": [ { "id": 1, "claim": "string", "verdict": "SUPPORTED|PARTIAL|UNSUPPORTED", "evidence": "string", "confidence": 0.0 } ] }
```
**Final state →** adds `stats` (`trust_score`, `weighted_score`, counts, `n_chunks`), `route`, `status`, `reviewer_note`, `report_markdown`.
