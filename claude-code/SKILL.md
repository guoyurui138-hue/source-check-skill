---
name: source-check
description: Verify every external claim in your output via multi-criteria scientific verification routed by claim type. NO skip mechanism (which would constitute Hansson pseudoscience pathology #4 "unwillingness to test" + #6 "built-in subterfuge"). Six claim types — factual-citation / time-sensitive / subjective / future / private / scientific — each route to a type-specific verifier subagent with the criteria appropriate to that claim type per philosophy-of-science consensus (single-criterion demarcation a la Popper alone is known to fail; Hansson SEP 2025, Pigliucci & Boudry 2013). Verifiers MUST use web tools, return verbatim quotes with ≥200 chars surrounding context, perform negative-control searches, make at least one tool call to a domain not appearing in the candidate claim, and produce strict JSON output checkable downstream by substring-match.
when_to_use: Auto-trigger when your draft answer contains any of an explicit citation; a time-sensitive number; "latest / current / recent" language; a subjective comparison ("X is best", "most popular"); a future prediction; user-supplied private data; or a claim citing scientific studies. Skip only pure-reasoning answers and creative work with no factual claims.
---

# source-check

External claims are verified by **multi-criteria scientific verification** routed by claim type. Every claim gets verified — there is no "this is unverifiable, skip" pathway. Single-criterion demarcation (e.g., Popper falsifiability alone) is known to fail (Hansson SEP §4; Pigliucci & Boudry 2013); the current consensus is multi-criteria assessment with criteria appropriate to the claim's type.

## Step 0 — Classify claim type (route, do NOT skip)

For each external claim in the draft, identify its type. Every type routes to a specific verifier prompt below. No skip mechanism — claims that **appear** unverifiable (subjective, future, private) get **different criteria**, not skip.

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

ORDER OF OPERATIONS (per SycEval, arXiv:2502.08177 — claim-first framing
produces +5pp sycophancy; show source first):
  1. Fetch sources FIRST.
  2. Read fetched text.
  3. Only THEN read the candidate claim.

TOOL USE — MANDATORY:
  - You MUST make ≥1 real web/fetch tool call. Empty tool_calls_made = automatic FAIL.
  - At least one tool call MUST target a domain NOT appearing in the candidate
    claim's URLs. Same-domain-only fetch ≠ independent corroboration ≠ verification.

ANTI-CRITICGPT (McAleese et al. 2024, arXiv:2407.00215):
  - Do NOT invent issues to seem rigorous.
  - Report only problems substantiated by a fetched verbatim quote.
  - Hallucinated objections penalized equally with sycophantic agreement.

ANTI-SYCOPHANCY:
  - If the orchestrator/user pre-states a verdict, IGNORE it.
  - Re-derive from fetched source.
  - The claim's "citation style" / confidence / credentials are NOT evidence.

QUOTES:
  - Every claim "supported" or "contradicted" must include a verbatim quote.
  - Quotes will be substring-matched downstream against fetched page text.
  - Record ≥200 chars of surrounding context (prevents cherry-picked quotation —
    a documented pseudoscience tactic per Hansson SEP §5.1).

INSUFFICIENT_EVIDENCE:
  - Preferred over guessing — NOT penalized.
  - But requires ≥3 distinct queries documented with reason each failed.
    Otherwise auto-downgrades to "verifier_retry_needed".

TIME ANNOTATION (mandatory — record the source's publication date):
  - source_published_at: when the source itself was published / dated
    (from <meta> tag, byline, paper publication date, commit date, or "unknown")
  - fetched_at is mechanically recorded inside each tool_calls_made entry —
    it always equals "approximately now" and is not user-facing in the headline.
  RULE: record the source's date as a fact; do NOT classify "shelf life".
  A verified fact may still be stale, but whether it matters depends on the
  user's specific use case — only they can judge. We give them the source's
  publication date; they decide. AI classifying volatility ("this is
  quarterly-volatile") would inject an unverifiable LLM judgment into what
  should be a pure fact-recording step.

OUTPUT: strict JSON only, per schema in §JSON Schema below.
```

## Tool routing — URL pattern → preferred fetch tier

For any URL/DOI in a candidate claim, pick the highest tier that resolves it:

**Tier 1 — Source-specific structured API (preferred):**
- arXiv ID / URL → `export.arxiv.org/api/query?id_list={ID}` (metadata) or `arxiv.org/abs/{ID}` (full HTML)
- DOI → `api.crossref.org/works/{DOI}` AND `api.openalex.org/works/doi:{DOI}` (cross-check for retraction; both required per hard rule #7)
- Academic paper metadata → `api.semanticscholar.org/graph/v1/paper/{DOI|arXiv:ID}?fields=title,year,venue,citationCount,referenceCount`
- GitHub URLs (repos / files / issues / PRs) → `gh api repos/{owner}/{repo}/...`
- Wikipedia → `en.wikipedia.org/api/rest_v1/page/summary/{Title}` (summary) or `/page/html/{Title}` (full HTML)
- PubMed (`pubmed.ncbi.nlm.nih.gov/{PMID}`) → NCBI E-utilities (`eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi`)
- npm / PyPI → `registry.npmjs.org/{pkg}` / `pypi.org/pypi/{pkg}/json`
- SEC filings → `data.sec.gov` / EDGAR API
- Stack Overflow → `api.stackexchange.com/2.3/questions/{ID}`
- Hacker News → `hacker-news.firebaseio.com/v0/item/{ID}.json`

**Tier 2 — Chrome MCP:** for JS-heavy SPAs (e.g., openai.com pricing pages, Notion/Linear/Figma docs), sites with anti-bot UA blocking, or paywalled academic content already logged-in via the user's browser session.

**Tier 3 — WebFetch:** for static public pages without a structured API (most news sites, vendor docs, personal blogs).

**Tier 4 — `curl`:** when WebFetch errors out but the page is genuinely fetchable (rare; usually means UA spoofing or specific headers needed).

**WebSearch:** use for *finding* a source you don't have a URL for yet (the negative-control searches, discovering refuting evidence). NOT for verifying a known URL — known URL → fetch it directly.

**ANTI-PATTERN — `WebFetch` with `summary=true`** (LLM-summarized fetch): drops verbatim words, breaking the load-bearing substring-match check downstream. Always fetch raw text. If a tool only exposes summary mode, switch tier (Chrome MCP / curl).

## Verifier (a) — factual_citation

```
INPUTS (in this order — SycEval ordering):
  source_url_or_doi: <the URL/DOI the main agent cited>
  expected_quote: <if provided, what the main agent said the source contains>
  claim_text: <the full claim sentence>

PROCEDURE:
  1. Fetch source_url_or_doi via WebFetch / Chrome MCP / curl.
     - If 404 / DNS-fail / page-doesn't-exist → CONTRADICTED,
       reason="source does not resolve" (citation fabrication; base rate
       11.4–56.8% per arXiv:2603.03299).
  2. If DOI: query api.crossref.org/works/{DOI} for `update-to` (retraction).
  3. Substring-match expected_quote (or extracted key claim) against fetched
     body. Fetch ≥200 chars surrounding context.
  4. If quote matches but claim paraphrases beyond quote → PARTIALLY_VERIFIED
     with note on what beyond-quote inference is unsupported.
  5. If source exists but no matching quote anywhere → CONTRADICTED.
  6. NEGATIVE CONTROL: do one search for refuting evidence on the same topic
     (e.g., other primary sources that contradict). Record outcome.

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
  3. CROSS-CORROBORATION (per arXiv:2603.03299, ≥2 independent sources →
     88.9% accuracy): fetch a SECOND independent domain confirming the same
     value. Required, not optional.
  4. Substring-match the numeric/textual value, allowing format normalization
     (e.g., "$3.00/Mtok" = "$3 per million tokens").
  5. Verdict: STILL_CURRENT / DIFFERS / INSUFFICIENT_EVIDENCE.

CRITERIA APPLIED: source-existence + verbatim attribution + recency +
cross-source corroboration.
```

## Verifier (c) — subjective ("X is best", "developers prefer Y")

A subjective claim is **not** unverifiable — it's verifiable against a different criterion: faithful representation of a *measured* community signal (e.g., "X is widely used" → 2024 Stack Overflow Survey, n=65,437, reports 73.6%). Skipping it would be Hansson pathology #4 (unwillingness to test).

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
  4. THRESHOLDS (engineering judgment — calibrate against post-deployment data):
     - ≥60% of respondents on one side → "consensus" → VERIFIED
     - 40–60% → "mixed / no consensus" → PARTIALLY_VERIFIED with both numbers
     - <40% on claimed side → CONTRADICTED
     - For benchmarks: claim must match top-3 leaderboard entry within task

HARD RULE: never VERIFIED without a named survey/benchmark with n≥1000 OR
peer-reviewed measurement. No "consensus per HN sentiment".

ESCALATION TO INSUFFICIENT_EVIDENCE: when the subjective claim is in a niche
domain where no n≥1000 instrument exists ("Zig is better than Rust for
embedded"). Return INSUFFICIENT_EVIDENCE explicitly — do NOT retreat to forum
sentiment.

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
     "unsourced speculation" (Hansson pathology #1: anonymous appeal to authority).

CRITERIA APPLIED: source-existence (of the prediction utterance) + verbatim
attribution + falsifiability-of-prediction-attribution (Popperian: who said it
and when is checkable; whether it comes true is for a later check).
```

## Verifier (e) — private / user-provided

```
INPUTS:
  user_supplied_context: <verbatim user-provided text, NOT externally fetchable>
  claim_text: <claim drawing on this private context>

PROCEDURE:
  1. NO external tool calls (data is private — externally checking would either
     fail or violate scope). tool_calls_made may be empty for THIS verifier type only.
  2. Decompose claim into atomic facts (FActScore-style, Min et al. 2023,
     arXiv:2305.14251).
  3. Substring-match / entailment-match each atomic fact against
     user_supplied_context only.
  4. faithfulness = supported_atoms / total_atoms.

VERDICT LABEL: GROUNDED 🔒 / UNGROUNDED 🔓 / PARTIALLY_GROUNDED — NOT
"VERIFIED" (no external truth check possible; this verifier checks faithfulness
to provided context only, not correspondence to external reality).

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
     - Cross-check both — OpenAlex `is_retracted` had ~2300 documented
       misclassifications in Dec 2023 / Mar 2024 (arXiv:2403.13339)
     - If retracted → CONTRADICTED with retraction notice quote
  3. FETCH PDF/HTML; substring-match expected_finding_quote with ≥200 chars
     surrounding context.
  4. PREPRINT-ONLY (OpenAlex primary_location.source.type=="repository" with no
     peer-reviewed version): PARTIALLY_VERIFIED + venue=preprint flag.
  5. arXiv version-churn (Hansson pathology #6 "built-in subterfuge"): OAI-PMH
     arXivRaw lists versions. ≥4 versions of a non-textbook claim → yellow flag.
  6. NEGATIVE CONTROL: search Semantic Scholar for papers citing this one with
     "contradict|refute|fail to replicate" in title/abstract. Record findings.
  7. HANSSON PATHOLOGY CHECKS (where mechanizable):
     #1 (appeal to authority): is the cited source the *primary* paper or a
        secondary citation? Check Semantic Scholar references graph.
     #2 (unrepeatable): are there replication-tagged citations? (Semantic
        Scholar; PubPeer if accessible)
     #4 (unwillingness to test): citation graph signal — if no replication
        attempts after ≥5 years for a testable claim, flag.
     #5 (disregard refuting): retraction status + negative-control results.
     #6 (built-in subterfuge): version-churn flag from step 5.
  8. OUT_OF_SCOPE_PATHOLOGIES: #3 ("handpicked examples") and #7
     ("explanations abandoned") are NOT mechanizable for a minutes-scale AI
     verifier — they require domain expertise and longitudinal evidence.
     MUST be explicitly listed in output's `out_of_scope_pathologies` field
     so downstream consumers don't mistake silence for clearance.

CRITERIA APPLIED: source-existence + verbatim attribution + external
consistency (retraction) + cross-corroboration + Hansson #1, #2, #4, #5, #6.
```

## Hard rules — do not violate

1. **`tool_calls_made` MUST be non-empty AND ≥1 call must target a domain NOT in the candidate claim's URLs.** Same-domain-only fetch = automatic FAIL (closes the loophole where a verifier fetches only the URL the drafter cited and rubber-stamps it).
2. **`evidence_quotes` MUST be verbatim AND substring-match the fetched page text downstream.** Main agent's downstream check is mechanical (programmatic substring search). Non-matching quotes → auto-FAIL. **THIS IS THE LOAD-BEARING ANTI-HALLUCINATION CHECK.**
3. **≥200 chars of surrounding context recorded for each evidence quote.** Prevents cherry-picked out-of-context quotation (Hansson SEP §5.1).
4. **Source-before-claim ordering in verifier prompt.** SycEval (arXiv:2502.08177): claim-first ordering produces +5pp sycophancy rate.
5. **Negative-control search required** for verifier types (c) subjective and (f) scientific. The output must include `negative_control_attempted: {performed, queries, findings}`.
6. **INSUFFICIENT_EVIDENCE requires ≥3 distinct queries documented** with reason each failed. Otherwise auto-downgrade to `verifier_retry_needed` (closes the "lazy INSUFFICIENT_EVIDENCE" loophole that would be Hansson pathology #4 at the verifier level).
7. **Retraction check (Crossref + OpenAlex, both)** mandatory for any DOI-cited claim. Cross-check because OpenAlex single source has documented misclassifications (arXiv:2403.13339).
8. **Verbatim quote required** for every VERIFIED / CONTRADICTED / PARTIALLY_VERIFIED verdict. No bare ✓ / ✗.
9. **Output must be valid JSON** per schema. Malformed → failed verification.
10. **Citations whose verifier returns "source does not resolve" → REMOVED from output**, not silently substituted with a "similar real one" (that's still fabrication, just dressed).
11. **`source_published_at` MUST be recorded for every non-private verdict** (extracted from page metadata / byline / DOI metadata / commit date; "unknown" only if genuinely absent). `fetched_at` is mechanically recorded in `tool_calls_made`. The verifier does NOT classify volatility — recording dates is a fact, classifying "shelf life" is an LLM judgment the user should make themselves.
12. **`verdict` MUST be exactly one of the enum values** — factual types: `VERIFIED` / `CONTRADICTED` / `PARTIALLY_VERIFIED` / `INSUFFICIENT_EVIDENCE`; private type: `GROUNDED` / `UNGROUNDED` / `PARTIALLY_GROUNDED`. No self-invented labels (e.g., `verified_with_caveat`, `MISLEADING_WITHOUT_CONTEXT`) — non-enum verdicts silently fall through downstream consumers (source-check-max combination table, headline UX mapping). Any nuance / qualification → `caveats[]`, not the verdict field.

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
  "criteria_summary": {
    "passed": 4,
    "failed": 1,
    "out_of_scope": 2,
    "total_applied": 5
  },
  "out_of_scope_pathologies": ["#3", "#7"],
  "retraction_status": {
    "is_retracted": false,
    "source": "crossref|openalex|both",
    "checked_at": "ISO8601"
  },
  "negative_control_attempted": {
    "performed": true,
    "queries": ["..."],
    "findings": "..."
  },
  "anti_sycophancy_check": {
    "source_fetched_before_claim_read": true
  },
  "source_published_at": "ISO8601 date or 'unknown'",
  "caveats": ["..."]
}
```

Required fields: `claim_id`, `claim_text`, `claim_type`, `verdict`, `headline`, `tool_calls_made` (non-empty, ≥1 with `domain_distinct_from_claim: true` except for `private` type), `criteria` (≥1 with `evidence_quote`).

Required for `scientific`: `retraction_status`, `negative_control_attempted`, `out_of_scope_pathologies`.

## Headline UX

Main agent renders one line per claim into the user-facing output:

| Verdict | Headline emoji + format |
|---|---|
| VERIFIED (all applicable criteria passed) | `✅ verified · <1 sentence summary, ≤140 chars>` |
| PARTIALLY_VERIFIED (N/M criteria, with failed list) | `⚠️ partial · passed N/M — failed: <list>` |
| CONTRADICTED | `❌ contradicted · <1 sentence with the refuting fact>` |
| INSUFFICIENT_EVIDENCE | `🔵 insufficient · <what's missing, in 1 sentence>` |
| GROUNDED (private) | `🔒 grounded in your context · <1 sentence>` |
| UNGROUNDED / PARTIALLY_GROUNDED (private) | `🔓 not grounded in your context · <which atoms missing>` |

**Timing — show the source's date, let the user judge staleness.** Every non-private headline must end with `· src dated: <source_published_at>`. We do NOT show the fetched-at time (it's always ≈ "now" and adds no signal), and we do NOT classify the source as "stale" or "fresh" (that depends on the user's use case — a 2019 numerical-methods result is fresh; a 2019 LLM benchmark is ancient). User sees the source's date, decides themselves.

```
Example:
✅ verified · Sonnet 4.5 input price is $3/M tokens · src dated: 2025-09-29
✅ verified · Adam optimizer convergence proof in Kingma 2014 · src dated: 2014-12-22
🔵 insufficient · no n≥1000 survey found for "Zig vs Rust embedded"
```

For `private` (GROUNDED / UNGROUNDED) claims, omit the source date — the dates come from the user's own context.

Below the user-facing answer, summary line:

```
[source-check: N claims (✅<V>, ⚠️<P>, ❌<C>, 🔵<U>, 🔒<G>, 🔓<UG>); claim-types: <count by type>; oldest source: <date>; warnings: <list or none>]
```

The `oldest source` field lets the user spot at a glance whether anything in the answer rests on potentially-aged data — without the skill having to decide for them what "aged" means.

Audit appendix (collapsed by default — user expands only for ⚠️ ❌ 🔵 🔓 claims, or when they want to scrutinize timing): full per-verifier JSON outputs, tool-call log, evidence quotes with context window.

## Limits / Caveats (honest)

- **Subjective consensus thresholds (60% / 40-60% / <40%) are engineering judgment.** No philosophy-of-science literature directly supports these specific numbers. Treat as initial calibration points; tune against post-deployment ground truth.
- **OpenAlex `is_retracted` field is known to have ~2300 misclassifications late 2023 / early 2024** (arXiv:2403.13339). The mandatory Crossref cross-check is the mitigation; single-source retraction check is unsafe.
- **Hansson #3 (handpicked examples) and #7 (abandoned explanations) are NOT mechanizable** at minutes scale by an AI verifier. They require domain expertise and longitudinal programme tracking (per Lakatos MSRP). Verifier output must explicitly list them in `out_of_scope_pathologies` so downstream consumers do not mistake silence for clearance.
- **Same-family drafter + verifier may share biases.** SycEval (arXiv:2502.08177) found 78.5% sycophancy persistence across model families; whether spawning a verifier from a *different* model family meaningfully reduces drafter-correlated bias is an open empirical question. For high-stakes claims, configure cross-family verifiers where the platform supports it.
- **Niche-domain subjective claims** (no n≥1000 survey available) → INSUFFICIENT_EVIDENCE is the honest verdict. Retreating to forum sentiment would be Hansson pathology #1 (appeal to authority masquerading as evidence).
- **Future predictions are never VERIFIED as "Y will happen"** — only as "X predicted Y on date Z". Predictor-attribution is verifiable; predicted-future-state is not.
- **Private / user-provided data uses GROUNDED / UNGROUNDED labels**, not VERIFIED — there is no external ground truth to verify against, only faithfulness to the supplied context.
- **PubPeer has no public keyless JSON API**; mechanizing Hansson #2 fully requires either a PubPeer Foundation dev key or HTML scraping.
- **Cochrane systematic reviews** (for clinical evidence) have no open REST API; require DOI-based fetch.
- **This skill catches fabricated existence + misattribution + factual drift since training.** It does NOT catch subtle paraphrase distortions where the source mostly matches but a qualifier is dropped (see `audit-loop`'s synthesis-from-fetched-source structural limit).

## Handshake with other skills

- **`audit-loop`**: handles algorithmic / mechanism / correctness reasoning. source-check handles external-claim grounding. Orthogonal — both can fire on the same output.
- **`source-check-max`** (experimental, dual-verifier with V1=URL-given + V2=URL-rediscovered): design not yet validated. Prefer source-check (this skill) for production use until source-check-max is empirically validated.
- **User's `red-team-process` memory**: heavier PROVENANCE BLOCK protocol for quantitative external-fact claims in high-stakes single answers. source-check is the lighter always-on enforcement.
