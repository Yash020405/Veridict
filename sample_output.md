# Sample Output

Two representative runs of the map-reduce workflow. (Exact wording varies by model; the structure and routing are stable. Verified locally against the sample input - see the run notes at the bottom.)

---

## Run A - high trust → AUTO_PUBLISH (no human needed)

The short sample produces a single chunk, so the **Summarizer** runs once and the **Synthesizer** cleans it into the canonical summary below. (A long document would produce several chunks that the Synthesizer merges.)

**Synthesize Summary output (JSON):**
```json
{
  "title": "Mediterranean Diet and Cardiovascular Risk",
  "summary": "A multicenter randomized trial of 7,447 high-risk participants compared a Mediterranean diet (with olive oil or nuts) against a low-fat control diet. After a median 4.8 years, the Mediterranean groups had roughly a 30% relative reduction in major cardiovascular events, driven mainly by fewer strokes. No significant reduction in all-cause mortality was found.",
  "key_claims": [
    {"id": 1, "claim": "The trial enrolled 7,447 participants aged 55 to 80 at high cardiovascular risk."},
    {"id": 2, "claim": "Participants followed one of three diets: Mediterranean+olive oil, Mediterranean+nuts, or low-fat control."},
    {"id": 3, "claim": "Median follow-up was about 4.8 years."},
    {"id": 4, "claim": "The Mediterranean groups saw ~30% relative reduction in major cardiovascular events."},
    {"id": 5, "claim": "The benefit was driven largely by reduced stroke incidence."},
    {"id": 6, "claim": "There was no significant reduction in all-cause mortality."}
  ]
}
```

**Verifier Agent output (JSON):**
```json
{
  "verifications": [
    {"id": 1, "verdict": "SUPPORTED", "evidence": "enrolled 7,447 participants aged 55 to 80 who were at high cardiovascular risk", "confidence": 0.99},
    {"id": 2, "verdict": "SUPPORTED", "evidence": "assigned to one of three diets...", "confidence": 0.98},
    {"id": 3, "verdict": "SUPPORTED", "evidence": "median follow-up of 4.8 years", "confidence": 0.99},
    {"id": 4, "verdict": "SUPPORTED", "evidence": "relative risk reduction of approximately 30 percent", "confidence": 0.97},
    {"id": 5, "verdict": "SUPPORTED", "evidence": "driven largely by a reduction in the incidence of stroke", "confidence": 0.95},
    {"id": 6, "verdict": "SUPPORTED", "evidence": "did not find a statistically significant reduction in death from any cause", "confidence": 0.96}
  ]
}
```

**Score & Decide Route (deterministic):** `supported 6/6 → trust_score 100%`, `weighted_score ~97%`, `unsupported 0`, no partials, no low-confidence → **route = AUTO_PUBLISH**.

**Final report:**
```
# Verified Research Summary: Mediterranean Diet and Cardiovascular Risk
Status: AUTO_PUBLISHED  |  Route: AUTO_PUBLISH
Trust score: 100% supported  ·  Confidence-weighted: 97%
Claims: 6/6 supported, 0 partial, 0 unsupported  ·  Source chunks analysed: 1
... (claim-by-claim table follows)
```

---

## Run B - hallucination injected → HARD_REVIEW (human gate)

Submitting `scenario_B` fills the *Inject Test Claim* field, which deterministically appends one extra claim the source never makes: *"The diet also reversed type 2 diabetes in 80% of patients."*

**Verifier flags it:**
```json
{"id": 7, "verdict": "UNSUPPORTED", "evidence": "", "confidence": 0.05}
```

**Score & Decide Route:** `supported 6/7 → trust_score 86%`, `weighted_score ~82%`, `unsupported 1`. Because `unsupported > 0`, the policy forces **route = HARD_REVIEW** regardless of the headline score.

→ Routes to **Gmail Send & Wait**: the reviewer gets an email -
> *Review needed: Mediterranean Diet... (trust 86%). Unsupported: 1. Approve to publish, or Reject to send it back for revision.*

The execution **pauses** until the reviewer clicks Approve or Reject. On Reject → `status = NEEDS_REVISION`. The final report and Sheet row record the human decision. **This is the workflow's whole reason to exist: the hallucinated diabetes claim never gets silently published.**

> A third tier sits between these two: if every claim is supported but some are PARTIAL or low-confidence (trust < 90%, no UNSUPPORTED), the run takes **LIGHT_REVIEW** - it auto-publishes but is stamped `AUTO_PUBLISHED_FLAGGED` in the audit log so it can be spot-checked later.

---

## Google Sheet log (audit trail)

| timestamp | title | status | route | trust_score | weighted_score | supported | partial | unsupported | n_chunks |
|-----------|-------|--------|-------|-------------|----------------|-----------|---------|-------------|----------|
| 2026-06-04T10:12:00Z | Mediterranean Diet... | AUTO_PUBLISHED | AUTO_PUBLISH | 100 | 97 | 6 | 0 | 0 | 1 |
| 2026-06-04T10:15:42Z | Mediterranean Diet... | NEEDS_REVISION | HARD_REVIEW | 86 | 82 | 6 | 0 | 1 | 1 |

---

> **Verification note:** this workflow was tested **live end-to-end** against a free Groq model (`llama-3.3-70b-versatile`) - all three agent roles called for real, then run through the actual Code-node logic:
> - **Run A** (short, honest) → 1 chunk, 4/4 supported, trust 100% → **AUTO_PUBLISH**.
> - **Run B** (injected false diabetes claim) → 5/6 supported, 1 unsupported, trust 83% → **HARD_REVIEW** (the hallucination was caught).
> - **Long-document run** → split into 2 chunks producing 7 + 5 = 12 candidate claims, which the Synthesizer reduced to 6 canonical claims (`n_chunks = 2`) → all supported → AUTO_PUBLISH. This confirms the map-reduce genuinely consolidates.
>
> Exact claim counts and wording vary per model run; the routing and structure are reproduced deterministically by the rules.
