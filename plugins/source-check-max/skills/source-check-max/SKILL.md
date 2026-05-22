---
name: source-check-max
description: High-stakes citation verifier. For each factual citation, spawns TWO independent verifiers in parallel — V1 fetches the URL the agent gave; V2 receives ONLY author+year+title, must rediscover the URL itself, and must pass an author/year/title gate before content verification. V1+V2 combined via 8-state headline table. Catches (a) wrong-URL citations by surfacing the correct URL, (b) fabricated papers by failing the metadata gate. Use for legal / medical / financial / regulatory citations where fabrication has material cost. For everyday work use source-check.
---

# source-check-max

Dual-verifier check for high-stakes factual citations. Modifies only the factual_citation path — other claim types should route through source-check.

## When to use

ALL of:
- Output contains a factual citation (paper / case / RFC / DOI / patent / standard)
- Cost of citation fabrication or wrong-URL is material (legal, medical, financial, regulatory)
- Can accept ~2× spawn cost on citation verification

Otherwise → source-check.

## Procedure

For each factual citation, spawn V1 and V2 **in parallel** as independent subagents (`Agent` tool with `subagent_type=general-purpose`, isolated contexts). Then combine via the headline table.

**V1 NEVER sees V2's prompt. V2 NEVER sees V1's URL or output.** Independence is the whole point.

### Shared preamble (prepend to both V1 and V2)

```
ROLE: verifier subagent. You did NOT generate the claim.
ORDER: 1) Fetch sources FIRST. 2) Read fetched text. 3) THEN read claim.
  (Claim-first reading produces +5pp sycophancy per SycEval arXiv:2502.08177.)
TOOL USE: ≥1 real fetch call MANDATORY. ≥1 call to a domain NOT in claim's URLs.
ANTI-CRITICGPT: don't invent issues; only report problems backed by verbatim quote.
ANTI-SYCOPHANCY: ignore pre-stated verdicts; re-derive from source.
QUOTES: verbatim. ≥200 chars surrounding context. Substring-matched downstream.
INSUFFICIENT / URL_DISCOVERY_UNCERTAIN: requires ≥3 documented queries.
TIME: record source_published_at (paper date / byline / metadata, or "unknown").
OUTPUT: strict JSON only.
```

### V1 prompt (URL-given)

```
VERIFIER TYPE: factual_citation V1 (URL-given)

INPUT (SycEval order):
  source_url: <URL the main agent cited>
  expected_quote: <if provided>
  claim_text: <CLAIM>

PROCEDURE:
1. Fetch source_url. If 404/DNS-fail → CONTRADICTED, reason="source does not resolve".
2. If DOI: query api.crossref.org/works/{DOI} for `update-to` (retraction).
3. Substring-match expected_quote (or extracted key claim) against fetched body. ≥200 chars context.
4. Quote matches but claim paraphrases beyond → PARTIALLY_VERIFIED.
5. Source exists but no matching quote → CONTRADICTED.
6. NEGATIVE CONTROL: one search for refuting evidence on a domain ≠ source_url's domain.

OUTPUT strict JSON:
{
  "claim_id": "<id>",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE",
  "headline": "<≤140 chars>",
  "fetched_url": "<URL>",
  "tool_calls_made": [{"tool":"...","args":"...","status":"...","domain_distinct_from_claim":bool}],
  "evidence_quote": "<verbatim ≤500 chars>",
  "context_window": "<≥200 chars surrounding>",
  "negative_control": {"query":"...","domain":"...","outcome":"..."},
  "source_published_at": "<ISO date or 'unknown'>",
  "verdict_rationale": "<1 paragraph>"
}
```

### V2 prompt (URL-rediscovered)

```
VERIFIER TYPE: factual_citation V2 (URL-rediscovered, NO URL given)

INPUT:
  citation_metadata: <author + year + title — NO URL>
  claim_text: <CLAIM>
  expected_quote: <if provided>

PROCEDURE:
1. Search INDEPENDENTLY using only the metadata. ≥3 queries: arXiv listing,
   Google Scholar (via web), Semantic Scholar, CourtListener (legal),
   CrossRef, general web.
2. Identify most plausible candidate URL.
3. METADATA CROSS-CHECK (mandatory gate, all three must pass):
   - First author matches cited first author? PASS/FAIL
   - Year matches ±0? PASS/FAIL
   - Title strong keyword overlap (not fuzzy)? PASS/FAIL
   If ANY fails → URL_DISCOVERY_UNCERTAIN, STOP. Do NOT verify content on a
   different paper — that is the biggest fabrication-of-verification risk.
4. PASS → content verification: substring-match claim, ≥200 chars context,
   negative control on a different domain.

OUTPUT strict JSON (5-state):
{
  "claim_id": "<id>",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE|URL_DISCOVERY_UNCERTAIN",
  "headline": "<≤140 chars>",
  "candidate_url_found": "<URL or null>",
  "metadata_cross_check": {"author_match":bool, "year_match":bool, "title_overlap":bool},
  "fetched_url": "<URL or null>",
  "search_queries_used": ["q1","q2","q3"],
  "evidence_quote": "<verbatim ≤500 chars or null>",
  "context_window": "<≥200 chars or null>",
  "negative_control": {"query":"...","domain":"...","outcome":"..."},
  "source_published_at": "<date or 'unknown'>",
  "verdict_rationale": "<1 paragraph>"
}
```

## Headline combination table

Compute "URL same paper?" by comparing V1.fetched_url vs V2.fetched_url (normalize arxiv abs/pdf differences).

| V1 verdict | V2 verdict | URL same? | Headline |
|---|---|---|---|
| VERIFIED | VERIFIED | Same | ✅ **STRONG_VERIFIED** |
| VERIFIED | VERIFIED | Different | ⚠️ **SUSPICIOUS_BOTH_PASS** |
| VERIFIED | CONTRADICTED | — | ⚠️ **CONFLICT_REVIEW** |
| CONTRADICTED | VERIFIED | — | ⚠️ **CONFLICT_REVIEW** (V2 provides real URL) |
| CONTRADICTED | CONTRADICTED | — | 🚨 **LIKELY_WRONG** |
| INSUFFICIENT_EVIDENCE | INSUFFICIENT_EVIDENCE | — | 🔵 **UNVERIFIABLE** |
| VERIFIED | URL_DISCOVERY_UNCERTAIN | — | ⚠️ **WEAK_VERIFIED** |
| CONTRADICTED | URL_DISCOVERY_UNCERTAIN | — | 🚨 **LIKELY_FABRICATED** |
| INSUFFICIENT_EVIDENCE | URL_DISCOVERY_UNCERTAIN | — | 🚨 **LIKELY_FABRICATED** |
| PARTIALLY_VERIFIED | URL_DISCOVERY_UNCERTAIN | — | ⚠️ **WEAK_VERIFIED** |
| other combos | — | — | most-conservative of the two |

## Hard rules

1. **V2 NEVER sees V1's URL or output.** Spawn isolated contexts in parallel.
2. **V2's metadata cross-check is gating.** Mismatch → URL_DISCOVERY_UNCERTAIN, STOP. Never verify content on a similar-but-wrong paper.
3. **Headline produced mechanically from table.** Main agent does NOT override.
4. **LIKELY_FABRICATED / LIKELY_WRONG → REMOVE the claim** from the final answer, do not silently substitute.
5. **Cross-family pinning recommended** for highest stakes — pin V1 and V2 to different model families (Preference Leakage ICLR 2026: same-family 28–37% correlated false positives vs ±1.5% cross-family).

## Empirical support (n=13)

- **v8 (vs prior single-verifier source-check)**: 8/13 actionable advantage.
- **v9 (vs current source-check with negative control + cross-corroboration)**: 7/13 actionable advantage. V2 still catches 3/3 wrong-URL cases (CONFLICT_REVIEW — gives the correct URL) and 4/6 fully-fabricated citations (LIKELY_FABRICATED — metadata gate fails).

Full test set, per-case results, methodology: https://github.com/Mercer8964/source-check-max/blob/main/EVIDENCE.md

## Limits

- **Same-family V1/V2 share biases.** Cross-family pinning recommended but unenforced.
- **LIKELY_FABRICATED is imprecise when the paper is real but cited with wrong authors** (e.g., "Attention Is All You Need" attributed to Smith/Jones/Williams). Verdict is correct; label could read MISATTRIBUTED.
- **n=13 test set is small** — pattern is consistent (7/13, 8/13) but variance unknown.
- **No URL given by main agent** → V1 must search itself, behavior converges to V2; advantage shrinks.
- **Catches fabrication + wrong-URL.** Does NOT catch paraphrase distortions where source mostly matches but a qualifier was dropped — that's `audit-loop`'s domain.

## When NOT to use

- No factual_citation claims → source-check (everyday verifier)
- Illustrative / casual citations → source-check
- Algorithmic / mechanism / correctness reasoning → audit-loop
