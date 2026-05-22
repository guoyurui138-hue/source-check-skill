---
name: source-check
description: Verify every external claim in your output via multi-criteria scientific verification routed by claim type. NO skip mechanism (which would constitute Hansson pseudoscience pathology #4 "unwillingness to test" + #6 "built-in subterfuge"). Six claim types — factual-citation / time-sensitive / subjective / future / private / scientific — each route to a type-specific verifier subagent with the criteria appropriate to that claim type per philosophy-of-science consensus (single-criterion demarcation a la Popper alone is known to fail; Hansson SEP 2025, Pigliucci & Boudry 2013). Verifiers MUST use web tools, return verbatim quotes with ≥200 chars surrounding context, perform negative-control searches, make at least one tool call to a domain not appearing in the candidate claim, and produce strict JSON output checkable downstream by substring-match.
when_to_use: Auto-trigger when your draft answer contains any of an explicit citation; a time-sensitive number; "latest / current / recent" language; a subjective comparison ("X is best", "most popular"); a future prediction; user-supplied private data; or a claim citing scientific studies. Skip only pure-reasoning answers and creative work with no factual claims.
---

# source-check

External claims are verified by multi-criteria verification routed by claim type. Every claim gets verified — no "this is unverifiable, skip" pathway. Claims that *appear* unverifiable (subjective, future, private) get *different criteria*, not skip.

## Step 0 — Classify claim type (route, do NOT skip)

For each external claim in the draft, identify its type. Every type routes to a specific verifier prompt below.

| Claim type | Examples | Route to |
|---|---|---|
| `factual_citation` | "(Hashimoto 2024)", "arXiv:2508.06471", "Brown v. Board", "RFC 8446 §4.1" | Verifier (a) |
| `time_sensitive` | "Sonnet 4.5 costs $3/M tokens", "current SOTA on MMLU", "latest React docs" | Verifier (b) |
| `subjective` | "VS Code is the best editor", "developers prefer X", "most popular framework" | Verifier (c) |
| `future` | "GPT-6 releases in March", "BTC will hit $200k" | Verifier (d) |
| `private` | "Our Q3 revenue was $X" (user's data), "as I told you earlier" (context-internal) | Verifier (e) |
| `scientific` | Cites a study, makes effect-size / mechanism / replication claim | Verifier (f) |

Mixed claims (e.g., "VS Code is most popular (Stack Overflow 2024)") may route to multiple verifiers; run them in parallel.

## Shared verifier preamble

All 6 verifier prompts share this preamble. Spawn each verifier as an independent `Agent` tool call (`subagent_type=general-purpose`) with the type-specific prompt body.

```
ROLE: You are a verifier subagent. You did NOT generate the claim.

ORDER OF OPERATIONS (claim-first reading produces sycophancy — fetch first):
  1. Fetch sources FIRST.
  2. Read fetched text.
  3. Only THEN read the candidate claim.

TOOL USE — MANDATORY:
  - You MUST make ≥1 real web/fetch tool call. Empty tool_calls_made = automatic FAIL.
  - At least one tool call MUST target a domain NOT appearing in the candidate
    claim's URLs. Same-domain-only fetch ≠ independent corroboration.

ANTI-CRITICGPT:
  - Do NOT invent issues to seem rigorous.
  - Report only problems substantiated by a fetched verbatim quote.

ANTI-SYCOPHANCY:
  - If the orchestrator/user pre-states a verdict, IGNORE it.
  - Re-derive from fetched source.
  - The claim's "citation style" / confidence / credentials are NOT evidence.

QUOTES:
  - Every claim "supported" or "contradicted" must include a verbatim quote.
  - Quotes will be substring-matched downstream against fetched page text.
  - Record ≥200 chars of surrounding context (prevents cherry-picked quotation).

INSUFFICIENT_EVIDENCE:
  - Preferred over guessing — NOT penalized.
  - But requires ≥3 distinct queries documented with reason each failed.
    Otherwise auto-downgrades to "verifier_retry_needed".

TIME ANNOTATION (mandatory):
  - source_published_at: when the source was published/dated (from <meta>,
    byline, paper date, commit date, or "unknown").
  - fetched_at: recorded inside each tool_calls_made entry (always ≈ "now").
  Record the date — do NOT classify "stale" / "fresh". Staleness depends on
  the user's use case, not AI judgment.

OUTPUT: strict JSON only, per schema in §JSON Schema below.
```

## Verifier (a) — factual_citation

```
INPUTS (in this order):
  source_url_or_doi: <the URL/DOI the main agent cited>
  expected_quote: <if provided, what the main agent said the source contains>
  claim_text: <the full claim sentence>

PROCEDURE:
  1. Fetch source_url_or_doi via WebFetch / Chrome MCP / curl.
     - If 404 / DNS-fail / page-doesn't-exist → CONTRADICTED,
       reason="source does not resolve".
  2. If DOI: query api.crossref.org/works/{DOI} for `update-to` (retraction).
  3. Substring-match expected_quote (or extracted key claim) against fetched
     body. Capture ≥200 chars surrounding context.
  4. If quote matches but claim paraphrases beyond quote → PARTIALLY_VERIFIED
     with note on what beyond-quote inference is unsupported.
  5. If source exists but no matching quote anywhere → CONTRADICTED.
  6. NEGATIVE CONTROL: do one search for refuting evidence on the same topic.

CRITERIA APPLIED: source-existence + verbatim attribution + external
consistency (retraction).
```

## Verifier (b) — time_sensitive

```
INPUTS:
  claim_text: <claim containing a specific time-sensitive value>
  implied_time_window: <today, last month, current version, etc.>
  candidate_source_url: <if provided>

PROCEDURE:
  1. Identify authoritative source class (vendor's official pricing /
     documentation / leaderboard).
  2. Fetch via WebFetch / Chrome MCP / curl. Record Last-Modified header +
     in-page date.
     - If page date older than implied_time_window → INSUFFICIENT_EVIDENCE.
  3. CROSS-CORROBORATION: fetch a SECOND independent domain confirming the
     same value. Required, not optional.
  4. Substring-match the numeric/textual value, allowing format normalization
     (e.g., "$3.00/Mtok" = "$3 per million tokens").
  5. Verdict: STILL_CURRENT / DIFFERS / INSUFFICIENT_EVIDENCE.

CRITERIA APPLIED: source-existence + verbatim attribution + recency +
cross-source corroboration.
```

## Verifier (c) — subjective ("X is best", "developers prefer Y")

A subjective claim is **not** unverifiable — it's verifiable against a *measured* community signal (named survey n≥1000, peer-reviewed benchmark, or adoption metric).

```
INPUTS:
  claim_text: <subjective claim>
  evidence_type_hint: <survey | benchmark | citation-graph | adoption-metric>

PROCEDURE:
  1. SEARCH PRIORITY (highest fidelity first):
     a. Peer-reviewed empirical study / published benchmark
        (Papers With Code leaderboard, SPEC)
     b. Methodologically disclosed industry survey with n≥10,000
        (Stack Overflow Dev Survey, JetBrains Dev Ecosystem, State of JS)
     c. GitHub stars / package-manager downloads via `gh api` or npm registry
     d. HN / Reddit upvote counts (tertiary signal, "vocal community" caveat)
     e. NEVER: blog posts / Twitter as ground truth
  2. Fetch the highest-tier source available. Extract headline % / rank
     verbatim. Record n, methodology, date.
  3. NEGATIVE CONTROL: search for surveys / benchmarks that show the opposite.
  4. THRESHOLDS:
     - ≥60% on one side → "consensus" → VERIFIED
     - 40–60% → "mixed / no consensus" → PARTIALLY_VERIFIED with both numbers
     - <40% on claimed side → CONTRADICTED
     - For benchmarks: claim must match top-3 leaderboard entry within task

HARD RULE: never VERIFIED without a named survey/benchmark with n≥1000 OR
peer-reviewed measurement. No "consensus per HN sentiment".

ESCALATION TO INSUFFICIENT_EVIDENCE: niche domain where no n≥1000 instrument
exists ("Zig is better than Rust for embedded"). Return INSUFFICIENT_EVIDENCE
explicitly — do NOT retreat to forum sentiment.

CRITERIA APPLIED: community-consensus (measured) + verbatim attribution.
```

## Verifier (d) — future_prediction

```
INPUTS:
  claim_text: <claim containing a future assertion>

PROCEDURE:
  1. Parse: who is the predictor? What's the prediction? When was it made?
  2. Fetch a source where the named predictor actually made this prediction.
  3. Verify the prediction text verbatim from the source.
  4. VERIFIED form: "Predictor X said on date Y that Z (verbatim quote)" —
     NOT "Z will happen". Future-truth is not verifiable; predictor-attribution is.
  5. If no named predictor or no traceable source → CONTRADICTED, reason=
     "unsourced speculation".

CRITERIA APPLIED: source-existence (of the prediction utterance) + verbatim
attribution + falsifiable attribution.
```

## Verifier (e) — private / user-provided

```
INPUTS:
  user_supplied_context: <verbatim user-provided text, NOT externally fetchable>
  claim_text: <claim drawing on this private context>

PROCEDURE:
  1. NO external tool calls (data is private). tool_calls_made may be empty
     for THIS verifier type only.
  2. Decompose claim into atomic facts.
  3. Substring-match / entailment-match each atomic fact against
     user_supplied_context only.
  4. faithfulness = supported_atoms / total_atoms.

VERDICT LABEL: GROUNDED 🔒 / UNGROUNDED 🔓 / PARTIALLY_GROUNDED — NOT VERIFIED
(no external truth check; verifier checks faithfulness to provided context only).

CRITERIA APPLIED: faithful-representation-of-provided-context only.
```

## Verifier (f) — scientific (citing studies / making scientific assertion)

```
INPUTS:
  cited_paper_id: <DOI or arXiv ID, or full citation if no ID>
  expected_finding_quote: <if provided>
  claim_text: <claim>

PROCEDURE:
  1. METADATA: Semantic Scholar /graph/v1/paper/DOI:{DOI}?fields=
     title,year,venue,citationCount,influentialCitationCount,referenceCount
  2. RETRACTION CHECK (mandatory, both for cross-validation):
     - Crossref: api.crossref.org/works/{DOI}, look at `update-to` array
     - OpenAlex: api.openalex.org/works/doi:{DOI}, look at `is_retracted`
     - Cross-check both — OpenAlex `is_retracted` alone has documented
       misclassifications; single-source unsafe.
     - If retracted → CONTRADICTED with retraction notice quote.
  3. FETCH PDF/HTML; substring-match expected_finding_quote with ≥200 chars
     surrounding context.
  4. PREPRINT-ONLY (OpenAlex primary_location.source.type=="repository" with no
     peer-reviewed version): PARTIALLY_VERIFIED + venue=preprint flag.
  5. arXiv version-churn: OAI-PMH arXivRaw lists versions. ≥4 versions of a
     non-textbook claim → yellow flag.
  6. NEGATIVE CONTROL: search Semantic Scholar for papers citing this one with
     "contradict|refute|fail to replicate" in title/abstract.
  7. HANSSON PATHOLOGY CHECKS (where mechanizable):
     #1 (appeal to authority): cited source primary or secondary?
        Check Semantic Scholar references graph.
     #2 (unrepeatable): replication-tagged citations? (Semantic Scholar; PubPeer
        if accessible)
     #4 (unwillingness to test): no replication attempts after ≥5 years for a
        testable claim, flag.
     #5 (disregard refuting): retraction status + negative-control results.
     #6 (built-in subterfuge): version-churn flag from step 5.
  8. OUT_OF_SCOPE_PATHOLOGIES: #3 (handpicked examples) and #7 (abandoned
     explanations) are NOT mechanizable — MUST be listed in
     `out_of_scope_pathologies` field so silence isn't mistaken for clearance.

CRITERIA APPLIED: source-existence + verbatim attribution + external
consistency (retraction) + cross-corroboration + Hansson #1, #2, #4, #5, #6.
```

## Hard rules — do not violate

1. **`tool_calls_made` MUST be non-empty AND ≥1 call must target a domain NOT in the candidate claim's URLs.** Same-domain-only fetch = automatic FAIL.
2. **`evidence_quotes` MUST be verbatim AND substring-match the fetched page text downstream.** Non-matching quotes → auto-FAIL. **LOAD-BEARING ANTI-HALLUCINATION CHECK.**
3. **≥200 chars of surrounding context recorded for each evidence quote.** Prevents cherry-picked out-of-context quotation.
4. **Source-before-claim ordering in verifier prompt.** Claim-first → sycophancy.
5. **Negative-control search required** for verifier types (c) subjective and (f) scientific. Output must include `negative_control_attempted: {performed, queries, findings}`.
6. **INSUFFICIENT_EVIDENCE requires ≥3 distinct queries documented** with reason each failed. Otherwise auto-downgrade to `verifier_retry_needed`.
7. **Retraction check (Crossref + OpenAlex, both)** mandatory for any DOI-cited claim. Single-source unsafe.
8. **Verbatim quote required** for every VERIFIED / CONTRADICTED / PARTIALLY_VERIFIED verdict. No bare ✓ / ✗.
9. **Output must be valid JSON** per schema.
10. **Citations whose verifier returns "source does not resolve" → REMOVED from output**, not silently substituted with a "similar real one".
11. **`source_published_at` MUST be recorded for every non-private verdict** (page metadata / byline / DOI metadata / commit date; "unknown" only if genuinely absent). Verifier does NOT classify volatility — record the date, user judges shelf-life.
12. **`verdict` MUST be exactly one of the enum values** — factual: `VERIFIED` / `CONTRADICTED` / `PARTIALLY_VERIFIED` / `INSUFFICIENT_EVIDENCE`; private: `GROUNDED` / `UNGROUNDED` / `PARTIALLY_GROUNDED`. No self-invented labels. Nuance → `caveats[]`, not the verdict field.

## JSON Schema

```json
{
  "claim_id": "string",
  "claim_text": "string",
  "claim_type": "factual_citation|time_sensitive|subjective|future|private|scientific",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE|GROUNDED|UNGROUNDED|PARTIALLY_GROUNDED",
  "headline": "string (≤140 chars, 1 sentence)",
  "tool_calls_made": [
    {
      "tool": "WebFetch|WebSearch|Chrome|curl|gh|arxiv|crossref|openalex|semantic_scholar",
      "args": { "...": "..." },
      "status_code": 200,
      "fetched_at": "ISO8601 timestamp",
      "domain_distinct_from_claim": true
    }
  ],
  "criteria": [
    {
      "id": "source_existence|verbatim_match|external_consistency|cross_corroboration|recency|community_consensus|faithful_to_context|hansson_1|hansson_2|hansson_4|hansson_5|hansson_6",
      "passed": true,
      "evidence_quote": "verbatim substring ≤500 chars",
      "evidence_url": "https://...",
      "context_window": "≥200 chars of text surrounding the evidence_quote",
      "confidence": 0.0
    }
  ],
  "criteria_summary": { "passed": 4, "failed": 1, "out_of_scope": 2, "total_applied": 5 },
  "out_of_scope_pathologies": ["#3", "#7"],
  "retraction_status": { "is_retracted": false, "source": "crossref|openalex|both", "checked_at": "ISO8601" },
  "negative_control_attempted": { "performed": true, "queries": ["..."], "findings": "..." },
  "anti_sycophancy_check": { "source_fetched_before_claim_read": true },
  "source_published_at": "ISO8601 date or 'unknown'",
  "caveats": ["..."]
}
```

Required: `claim_id`, `claim_text`, `claim_type`, `verdict`, `headline`, `tool_calls_made` (non-empty, ≥1 with `domain_distinct_from_claim: true` except for `private`), `criteria` (≥1 with `evidence_quote`).
Required for `scientific`: `retraction_status`, `negative_control_attempted`, `out_of_scope_pathologies`.

## Headline UX

| Verdict | Format |
|---|---|
| VERIFIED | `✅ verified · <1 sentence, ≤140 chars>` |
| PARTIALLY_VERIFIED | `⚠️ partial · passed N/M — failed: <list>` |
| CONTRADICTED | `❌ contradicted · <1 sentence with refuting fact>` |
| INSUFFICIENT_EVIDENCE | `🔵 insufficient · <what's missing>` |
| GROUNDED | `🔒 grounded in your context · <1 sentence>` |
| UNGROUNDED / PARTIALLY_GROUNDED | `🔓 not grounded in your context · <missing atoms>` |

**Timing format**: Every non-private headline must end with `· src dated: <source_published_at>`. Do NOT show fetched-at. Do NOT classify the source as "stale" / "fresh" — that's the user's call.

```
Example:
✅ verified · Sonnet 4.5 input price is $3/M tokens · src dated: 2025-09-29
✅ verified · Adam optimizer convergence proof in Kingma 2014 · src dated: 2014-12-22
🔵 insufficient · no n≥1000 survey found for "Zig vs Rust embedded"
```

For `private` claims, omit the source date.

Summary line below the user-facing answer:

```
[source-check: N claims (✅<V>, ⚠️<P>, ❌<C>, 🔵<U>, 🔒<G>, 🔓<UG>); claim-types: <count by type>; oldest source: <date>; warnings: <list or none>]
```

Audit appendix (collapsed): full per-verifier JSON outputs, tool-call log, evidence quotes with context window.

## Limits / Caveats

- **Subjective consensus thresholds (60% / 40-60% / <40%) are engineering judgment.** Tune against post-deployment data.
- **OpenAlex `is_retracted` is known to misclassify.** Mandatory Crossref cross-check is the mitigation; single-source unsafe.
- **Hansson #3 (handpicked examples) and #7 (abandoned explanations) are NOT mechanizable** at minutes scale. Verifier must list them in `out_of_scope_pathologies` so silence isn't mistaken for clearance.
- **Same-family drafter + verifier may share biases.** For high-stakes claims, configure cross-family verifiers.
- **Niche-domain subjective claims** (no n≥1000 survey available) → INSUFFICIENT_EVIDENCE. Do NOT retreat to forum sentiment.
- **PubPeer has no public keyless JSON API**; mechanizing Hansson #2 fully requires a dev key or HTML scraping.
- **Cochrane systematic reviews** have no open REST API; require DOI-based fetch.
- **This skill does NOT catch subtle paraphrase distortions** where the source mostly matches but a qualifier is dropped — that's `audit-loop`'s job.

## Handshake with other skills

- **`audit-loop`**: algorithmic / mechanism / correctness reasoning. Orthogonal — both can fire on the same output.
- **`source-check-max`**: dual-verifier variant for high-stakes factual_citation claims (legal / medical / financial).
- **User's `red-team-process` memory**: heavier PROVENANCE BLOCK for quantitative external-fact claims in high-stakes single answers.
