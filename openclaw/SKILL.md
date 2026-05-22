---
name: source-check
description: Verify every external claim in your output via multi-criteria scientific verification routed by claim type. NO skip mechanism (which would constitute Hansson pseudoscience pathology #4 "unwillingness to test" + #6 "built-in subterfuge"). Six claim types — factual-citation / time-sensitive / subjective / future / private / scientific — each route to a type-specific verifier subagent (spawned via sessions_spawn with context: isolated) with the criteria appropriate to that claim type per philosophy-of-science consensus (single-criterion demarcation a la Popper alone is known to fail; Hansson SEP 2025, Pigliucci & Boudry 2013). Verifiers MUST use web tools, return verbatim quotes with ≥200 chars surrounding context, perform negative-control searches, make at least one tool call to a domain not appearing in the candidate claim, and produce strict JSON output checkable downstream by substring-match.
metadata:
  openclaw:
    requires:
      tools:
        - sessions_spawn
        - sessions_yield
---

# source-check

External claims are verified by multi-criteria verification routed by claim type. Every claim gets verified — no "skip" pathway. Claims that *appear* unverifiable (subjective, future, private) get *different criteria*, not skip.

## Step 0 — Classify claim type (route, do NOT skip)

For each external claim, identify its type. Every type routes to a verifier.

| Claim type | Examples | Route to |
|---|---|---|
| `factual_citation` | "(Hashimoto 2024)", "arXiv:2508.06471", case number, RFC | Verifier (a) |
| `time_sensitive` | "$3/M tokens", "current SOTA", "latest React" | Verifier (b) |
| `subjective` | "VS Code is best", "developers prefer X" | Verifier (c) |
| `future` | "GPT-6 in March", "BTC hits $200k" | Verifier (d) |
| `private` | "Our Q3 revenue was $X" (user-supplied) | Verifier (e) |
| `scientific` | Cites study, effect-size / mechanism claim | Verifier (f) |

## Shared verifier preamble

Spawn each verifier via `sessions_spawn` with `context: "isolated"`, then `sessions_yield` to wait (no polling).

```
sessions_spawn({
  task: <preamble + type-specific verifier body>,
  taskName: "src_check_<type>_<N>",
  context: "isolated",
  runTimeoutSeconds: 600
})
sessions_yield()
```

Task body preamble:
```
ROLE: You are a verifier subagent. You did NOT generate the claim.

ORDER OF OPERATIONS (claim-first reading produces sycophancy — fetch first):
  1. Fetch sources FIRST.
  2. Read fetched text.
  3. Only THEN read the candidate claim.

TOOL USE — MANDATORY:
  - ≥1 real web/fetch tool call. Empty tool_calls_made = automatic FAIL.
  - At least one tool call MUST target a domain NOT in candidate claim's URLs.

ANTI-CRITICGPT:
  - Do NOT invent issues to seem rigorous.

ANTI-SYCOPHANCY:
  - If orchestrator pre-states a verdict, IGNORE it.
  - Re-derive from fetched source.

QUOTES:
  - Verbatim, substring-matchable downstream.
  - ≥200 chars surrounding context (prevents cherry-picked quotation).

INSUFFICIENT_EVIDENCE:
  - Preferred over guessing — NOT penalized.
  - But requires ≥3 distinct queries with reason each failed.
    Otherwise auto-downgrades to verifier_retry_needed.

TIME ANNOTATION (mandatory):
  - source_published_at: when the source was published/dated (from <meta>,
    byline, paper date, commit date, or "unknown").
  - fetched_at: recorded inside each tool_calls_made entry (always ≈ "now").
  Record the date — do NOT classify "stale" / "fresh". Staleness depends on
  user's use case, not AI judgment.

OUTPUT: strict JSON only.
```

## Verifier (a) — factual_citation

```
INPUTS:
  source_url_or_doi
  expected_quote (if provided)
  claim_text

PROCEDURE:
  1. Fetch source. 404/DNS-fail → CONTRADICTED, reason="source does not resolve".
  2. If DOI: api.crossref.org/works/{DOI} for `update-to` (retraction).
  3. Substring-match expected_quote with ≥200 chars context.
  4. Quote matches but claim paraphrases beyond → PARTIALLY_VERIFIED.
  5. Source exists, no matching quote → CONTRADICTED.
  6. NEGATIVE CONTROL: search for refuting evidence.

CRITERIA: source-existence + verbatim + external consistency (retraction).
```

## Verifier (b) — time_sensitive

```
INPUTS:
  claim_text, implied_time_window, candidate_source_url

PROCEDURE:
  1. Identify authoritative source (vendor's official pricing / docs / leaderboard).
  2. Fetch; record Last-Modified + in-page date. Outdated → INSUFFICIENT_EVIDENCE.
  3. CROSS-CORROBORATION: fetch SECOND independent domain confirming. Required.
  4. Substring-match value, allow format normalization.

CRITERIA: source-existence + verbatim + recency + cross-corroboration.
```

## Verifier (c) — subjective ("X is best")

A subjective claim is **not** unverifiable — verifiable against a *measured* community signal (named survey n≥1000, peer-reviewed benchmark, or adoption metric).

```
INPUTS:
  claim_text, evidence_type_hint

PROCEDURE:
  1. SEARCH PRIORITY:
     a. Peer-reviewed / published benchmark (Papers With Code, SPEC)
     b. Industry survey n≥10K (Stack Overflow Survey, JetBrains, State of JS)
     c. GitHub stars / package downloads
     d. HN / Reddit (tertiary, "vocal community" caveat)
     e. NEVER blog posts / Twitter
  2. Fetch highest tier. Extract headline % verbatim. Record n, methodology, date.
  3. NEGATIVE CONTROL: search opposite.
  4. THRESHOLDS:
     - ≥60% one side → consensus → VERIFIED
     - 40–60% → mixed → PARTIALLY_VERIFIED with both numbers
     - <40% on claimed side → CONTRADICTED

HARD RULE: never VERIFIED without n≥1000 named survey OR peer-reviewed measurement.

ESCALATION: niche domain (no n≥1000) → INSUFFICIENT_EVIDENCE. Do NOT retreat
to forum sentiment.

CRITERIA: community-consensus (measured) + verbatim.
```

## Verifier (d) — future_prediction

```
INPUTS:
  claim_text

PROCEDURE:
  1. Parse: who predicted, what, when.
  2. Fetch source where named predictor made the prediction.
  3. Verify prediction text verbatim.
  4. VERIFIED form: "X predicted Y on date Z (verbatim)" — NOT "Y will happen".
  5. No named predictor / no source → CONTRADICTED, reason="unsourced speculation".

CRITERIA: source-existence (utterance) + verbatim + falsifiable attribution.
```

## Verifier (e) — private / user-provided

```
INPUTS:
  user_supplied_context (verbatim user-provided)
  claim_text

PROCEDURE:
  1. NO external tool calls (private data). tool_calls_made may be empty
     for THIS type only.
  2. Decompose into atomic facts.
  3. Substring/entailment-match each atom against user_supplied_context only.
  4. faithfulness = supported_atoms / total_atoms.

VERDICT: GROUNDED 🔒 / UNGROUNDED 🔓 / PARTIALLY_GROUNDED — NOT VERIFIED
(no external truth; verifier checks faithfulness to context only).

CRITERIA: faithful-representation-of-provided-context.
```

## Verifier (f) — scientific

```
INPUTS:
  cited_paper_id (DOI or arXiv ID)
  expected_finding_quote
  claim_text

PROCEDURE:
  1. METADATA: Semantic Scholar /graph/v1/paper/DOI:{DOI}?fields=
     title,year,venue,citationCount,influentialCitationCount,referenceCount
  2. RETRACTION CHECK (mandatory, both):
     - Crossref: api.crossref.org/works/{DOI}, `update-to` array
     - OpenAlex: api.openalex.org/works/doi:{DOI}, `is_retracted`
     - Cross-check both — OpenAlex alone has documented misclassifications;
       single-source unsafe.
     - Retracted → CONTRADICTED with retraction quote.
  3. FETCH PDF/HTML; substring-match expected_finding_quote with ≥200 chars context.
  4. Preprint-only → PARTIALLY_VERIFIED + flag.
  5. arXiv version-churn: OAI-PMH arXivRaw. ≥4 versions = yellow flag.
  6. NEGATIVE CONTROL: search Semantic Scholar citing papers with
     "contradict|refute|fail to replicate" in title/abstract.
  7. HANSSON CHECKS:
     #1 (authority): primary vs secondary source (Semantic Scholar refs graph)
     #2 (unrepeatable): replication-tagged citations
     #4 (unwillingness to test): no replications after 5+ years
     #5 (disregard refuting): retraction + negative-control
     #6 (built-in subterfuge): version-churn flag
  8. OUT_OF_SCOPE_PATHOLOGIES: #3 (handpicked) + #7 (abandoned) NOT
     mechanizable — MUST be listed in `out_of_scope_pathologies` so silence
     isn't mistaken for clearance.

CRITERIA: source-existence + verbatim + external consistency (retraction) +
cross-corroboration + Hansson #1, #2, #4, #5, #6.
```

## Hard rules — do not violate

1. **`tool_calls_made` non-empty AND ≥1 call to domain NOT in claim's URLs.** Same-domain-only = auto-FAIL.
2. **`evidence_quotes` verbatim + substring-match fetched page downstream.** Non-match = auto-FAIL. **LOAD-BEARING.**
3. **≥200 chars surrounding context** for each evidence quote.
4. **Source-before-claim ordering.** Claim-first → sycophancy.
5. **Negative-control search required** for verifier types (c) + (f).
6. **INSUFFICIENT_EVIDENCE requires ≥3 documented queries** with reason each failed; else auto-downgrade to verifier_retry_needed.
7. **Retraction check (Crossref + OpenAlex, both)** mandatory for DOI claims. Single-source unsafe.
8. **Verbatim quote required** for VERIFIED / CONTRADICTED / PARTIALLY_VERIFIED.
9. **Valid JSON output** per schema.
10. **"Source does not resolve" → REMOVED**, not substituted.
11. **`context: "isolated"` only** — never `fork` (would leak draft).
12. **No polling for spawn completion** — `sessions_yield`.
13. **`source_published_at` recorded for every non-private verdict** (page metadata / byline / DOI / commit date; "unknown" only if genuinely absent). Verifier does NOT classify volatility — record the date, user judges shelf-life.
14. **`verdict` MUST be exactly one of the enum values** — factual: `VERIFIED` / `CONTRADICTED` / `PARTIALLY_VERIFIED` / `INSUFFICIENT_EVIDENCE`; private: `GROUNDED` / `UNGROUNDED` / `PARTIALLY_GROUNDED`. No self-invented labels. Nuance → `caveats[]`, not the verdict field.

## JSON Schema

```json
{
  "claim_id": "string",
  "claim_text": "string",
  "claim_type": "factual_citation|time_sensitive|subjective|future|private|scientific",
  "verdict": "VERIFIED|CONTRADICTED|PARTIALLY_VERIFIED|INSUFFICIENT_EVIDENCE|GROUNDED|UNGROUNDED|PARTIALLY_GROUNDED",
  "headline": "string (≤140 chars)",
  "tool_calls_made": [
    { "tool": "...", "args": {...}, "status_code": 200,
      "fetched_at": "ISO8601", "domain_distinct_from_claim": true }
  ],
  "criteria": [
    { "id": "source_existence|verbatim_match|external_consistency|cross_corroboration|recency|community_consensus|faithful_to_context|hansson_1|hansson_2|hansson_4|hansson_5|hansson_6",
      "passed": true,
      "evidence_quote": "verbatim ≤500 chars",
      "evidence_url": "https://...",
      "context_window": "≥200 chars",
      "confidence": 0.0 }
  ],
  "criteria_summary": { "passed": 4, "failed": 1, "out_of_scope": 2, "total_applied": 5 },
  "out_of_scope_pathologies": ["#3", "#7"],
  "retraction_status": { "is_retracted": false, "source": "crossref|openalex|both", "checked_at": "ISO8601" },
  "negative_control_attempted": { "performed": true, "queries": [...], "findings": "..." },
  "anti_sycophancy_check": { "source_fetched_before_claim_read": true },
  "source_published_at": "ISO8601 date or 'unknown'",
  "caveats": [...]
}
```

## Headline UX

| Verdict | Format |
|---|---|
| VERIFIED | `✅ verified · <1 sentence, ≤140 chars>` |
| PARTIALLY_VERIFIED | `⚠️ partial · passed N/M — failed: <list>` |
| CONTRADICTED | `❌ contradicted · <refuting fact>` |
| INSUFFICIENT_EVIDENCE | `🔵 insufficient · <what's missing>` |
| GROUNDED | `🔒 grounded in your context · <summary>` |
| UNGROUNDED | `🔓 not grounded in your context · <missing atoms>` |

**Timing format**: Every non-private headline must end with `· src dated: <source_published_at>`. Do NOT show fetched-at. Do NOT classify the source as "stale" / "fresh" — that's the user's call.

```
Example:
✅ verified · Sonnet 4.5 input price is $3/M tokens · src dated: 2025-09-29
✅ verified · Adam convergence proof in Kingma 2014 · src dated: 2014-12-22
🔵 insufficient · no n≥1000 survey for "Zig vs Rust embedded"
```

For `private` claims, omit the source date.

Summary line:
```
[source-check: N claims (✅<V>, ⚠️<P>, ❌<C>, 🔵<U>, 🔒<G>, 🔓<UG>); types: <count>; oldest source: <date>; warnings: <list>]
```

## Cross-family setup (highly recommended)

OpenClaw is the best platform for cross-family verification — `agents.list[].subagents.model` can pin to any configured provider. For high-stakes source-check, configure two verifier agents pinned to different model families (e.g., one Anthropic, one OpenAI). Highest-leverage configuration improvement available.

## Limits / Caveats

- **Subjective thresholds (60%/40-60%/<40%) are engineering judgment.** Calibrate against post-deployment.
- **OpenAlex `is_retracted` is known to misclassify.** Crossref cross-check is mandatory.
- **Hansson #3 + #7 NOT mechanizable** — `out_of_scope_pathologies` field prevents silence-as-clearance.
- **Same-family drafter + verifier may share biases.** Cross-family setup is OpenClaw's best lever.
- **Niche subjective (no n≥1000 survey)** → INSUFFICIENT_EVIDENCE.
- **PubPeer/Cochrane APIs** partially limit Hansson mechanization.
- **Does NOT catch subtle paraphrase distortions** — that's `audit-loop`'s job.

## Handshake with other skills

- **`audit-loop`**: algorithmic / mechanism reasoning. Orthogonal.
- **`source-check-max`**: dual-verifier variant for high-stakes factual_citation claims (legal / medical / financial).
- **User's `red-team-process` memory**: heavier PROVENANCE BLOCK for high-stakes single answers.
