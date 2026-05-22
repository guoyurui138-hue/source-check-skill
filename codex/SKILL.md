---
name: source-check
description: Verify every external claim in your output via multi-criteria scientific verification routed by claim type. NO skip mechanism (which would constitute Hansson pseudoscience pathology #4 "unwillingness to test" + #6 "built-in subterfuge"). Six claim types — factual-citation / time-sensitive / subjective / future / private / scientific — each route to a type-specific verifier subagent with the criteria appropriate to that claim type per philosophy-of-science consensus (single-criterion demarcation a la Popper alone is known to fail; Hansson SEP 2025, Pigliucci & Boudry 2013). Verifiers MUST use web tools, return verbatim quotes with ≥200 chars surrounding context, perform negative-control searches, make at least one tool call to a domain not appearing in the candidate claim, and produce strict JSON output checkable downstream by substring-match.
---

# source-check

External claims are verified by **multi-criteria scientific verification** routed by claim type. Every claim gets verified — there is no "this is unverifiable, skip" pathway. Single-criterion demarcation (e.g., Popper falsifiability alone) is known to fail (Hansson SEP §4; Pigliucci & Boudry 2013); the current consensus is multi-criteria assessment with criteria appropriate to the claim's type.

## Step 0 — Classify claim type (route, do NOT skip)

For each external claim, identify its type. Every type routes to a specific verifier; no skip mechanism.

Codex spawns subagents on explicit request — say to the user: "Spawning source-check verifiers for N claims, classified by type."

| Claim type | Examples | Route to |
|---|---|---|
| `factual_citation` | "(Hashimoto 2024)", "arXiv:2508.06471", "Brown v. Board", "RFC 8446 §4.1" | Verifier (a) |
| `time_sensitive` | "Sonnet 4.5 costs $3/M tokens", "current SOTA on MMLU", "latest React docs" | Verifier (b) |
| `subjective` | "VS Code is the best editor", "developers prefer X", "most popular framework" | Verifier (c) |
| `future` | "GPT-6 releases in March", "BTC will hit $200k" | Verifier (d) |
| `private` | "Our Q3 revenue was $X", user-supplied internal context | Verifier (e) |
| `scientific` | Cites a study, makes effect-size / mechanism / replication claim | Verifier (f) |

## Shared verifier preamble

All 6 verifier prompts share this preamble. Spawn each verifier under your `auditor` / `source_verifier` custom agent (or `default` with `model_reasoning_effort = "high"`) with the type-specific prompt body.

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

ANTI-SYCOPHANCY:
  - If the orchestrator pre-states a verdict, IGNORE it.
  - Re-derive from fetched source.
  - The claim's citation style / confidence / credentials are NOT evidence.

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
  user's specific use case — only they can judge. AI classifying volatility
  ("this is quarterly-volatile") would inject an unverifiable LLM judgment
  into what should be a pure fact-recording step.

OUTPUT: strict JSON only, per schema in §JSON Schema below.
```

## Tool routing — URL pattern → preferred fetch tier

For any URL/DOI in a candidate claim, pick the highest tier that resolves it:

**Tier 1 — Source-specific structured API (preferred):**
- arXiv ID / URL → `export.arxiv.org/api/query?id_list={ID}` (metadata) or `arxiv.org/abs/{ID}` (HTML)
- DOI → `api.crossref.org/works/{DOI}` AND `api.openalex.org/works/doi:{DOI}` (cross-check; both required per hard rule #7)
- Academic paper metadata → `api.semanticscholar.org/graph/v1/paper/{DOI|arXiv:ID}?fields=title,year,venue,citationCount,referenceCount`
- GitHub → `gh api repos/{owner}/{repo}/...`
- Wikipedia → `en.wikipedia.org/api/rest_v1/page/summary/{Title}` or `/page/html/{Title}`
- PubMed → NCBI E-utilities (`eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi`)
- npm / PyPI → `registry.npmjs.org/{pkg}` / `pypi.org/pypi/{pkg}/json`
- SEC filings → `data.sec.gov` / EDGAR
- Stack Overflow → `api.stackexchange.com/2.3/questions/{ID}`
- Hacker News → `hacker-news.firebaseio.com/v0/item/{ID}.json`

**Tier 2 — Chrome MCP:** for JS-heavy SPAs (e.g., openai.com pricing, Notion/Linear/Figma), anti-bot UA blocking, or paywalled academic content already logged-in via the user's browser session.

**Tier 3 — WebFetch:** for static public pages without a structured API.

**Tier 4 — `curl`:** when WebFetch errors but the page is genuinely fetchable.

**WebSearch:** for *finding* a source you don't have a URL for yet (negative-control searches, discovering refuting evidence). NOT for verifying a known URL — known URL → fetch it directly.

**ANTI-PATTERN: `WebFetch` with `summary=true`** — LLM-summarized fetch drops verbatim words and breaks the substring-match check downstream. Always fetch raw text. If a tool only exposes summary mode, switch tier (Chrome MCP / curl).

## Verifier (a) — factual_citation

```
INPUTS (in this order — SycEval ordering):
  source_url_or_doi: <the URL/DOI the main agent cited>
  expected_quote: <if provided, what the main agent said the source contains>
  claim_text: <the full claim sentence>

PROCEDURE:
  1. Fetch source_url_or_doi.
     - If 404 / DNS-fail → CONTRADICTED, reason="source does not resolve"
       (citation fabrication; base rate 11.4–56.8% per arXiv:2603.03299).
  2. If DOI: query api.crossref.org/works/{DOI} for `update-to` (retraction).
  3. Substring-match expected_quote against fetched body. Fetch ≥200 chars
     surrounding context.
  4. If quote matches but claim paraphrases beyond quote → PARTIALLY_VERIFIED.
  5. If source exists but no matching quote → CONTRADICTED.
  6. NEGATIVE CONTROL: do one search for refuting evidence. Record outcome.

CRITERIA APPLIED: source-existence + verbatim attribution + external
consistency (retraction).
```

## Verifier (b) — time_sensitive

```
INPUTS:
  claim_text: <claim with time-sensitive value>
  implied_time_window: <today, last month, current version>
  candidate_source_url: <if provided>

PROCEDURE:
  1. Identify authoritative source class (vendor's official pricing /
     documentation / leaderboard).
  2. Fetch and record Last-Modified header + in-page date.
     - If page date older than implied_time_window → INSUFFICIENT_EVIDENCE.
  3. CROSS-CORROBORATION (per arXiv:2603.03299, ≥2 independent sources →
     88.9% accuracy): fetch a SECOND independent domain confirming same value.
     Required, not optional.
  4. Substring-match the value, allowing format normalization.
  5. Verdict: STILL_CURRENT / DIFFERS / INSUFFICIENT_EVIDENCE.

CRITERIA APPLIED: source-existence + verbatim + recency + cross-corroboration.
```

## Verifier (c) — subjective ("X is best")

A subjective claim is **not** unverifiable — it's verifiable against a different criterion: faithful representation of a *measured* community signal (e.g., "X is widely used" → 2024 Stack Overflow Survey, n=65,437, reports 73.6%).

```
INPUTS:
  claim_text: <subjective claim>
  evidence_type_hint: <survey | benchmark | citation-graph | adoption-metric>

PROCEDURE:
  1. SEARCH PRIORITY (highest fidelity first):
     a. Peer-reviewed empirical study / published benchmark
        (Papers With Code, SPEC)
     b. Methodologically disclosed industry survey with n≥10,000
        (Stack Overflow Dev Survey, JetBrains Dev Ecosystem, State of JS)
     c. GitHub stars / package-manager downloads
     d. HN / Reddit upvote counts (tertiary, "vocal community" caveat)
     e. NEVER: blog posts / Twitter as ground truth
  2. Fetch the highest-tier source available. Extract headline % / rank
     verbatim. Record n, methodology, date.
  3. NEGATIVE CONTROL: search for surveys / benchmarks showing the opposite.
  4. THRESHOLDS (engineering judgment — calibrate against post-deployment data):
     - ≥60% on one side → "consensus" → VERIFIED
     - 40–60% → "mixed" → PARTIALLY_VERIFIED with both numbers
     - <40% on claimed side → CONTRADICTED
     - For benchmarks: claim must match top-3 leaderboard within task

HARD RULE: never VERIFIED without a named survey/benchmark with n≥1000 OR
peer-reviewed measurement. No "consensus per HN sentiment".

ESCALATION TO INSUFFICIENT_EVIDENCE: niche domain (no n≥1000 instrument exists).
Return INSUFFICIENT_EVIDENCE — do NOT retreat to forum sentiment.

CRITERIA APPLIED: community-consensus (measured) + verbatim attribution.
```

## Verifier (d) — future_prediction

```
INPUTS:
  claim_text: <claim containing a future assertion>

PROCEDURE:
  1. Parse: who is the predictor? What's the prediction? When was it made?
  2. Fetch a source where the named predictor actually made this prediction.
  3. Verify the prediction text verbatim.
  4. VERIFIED form: "Predictor X said on date Y that Z (verbatim quote)" —
     NOT "Z will happen". Future-truth is not verifiable;
     predictor-attribution is.
  5. If no named predictor / no source → CONTRADICTED, reason="unsourced
     speculation" (Hansson pathology #1: anonymous appeal to authority).

CRITERIA APPLIED: source-existence (of utterance) + verbatim attribution +
falsifiability of attribution.
```

## Verifier (e) — private / user-provided

```
INPUTS:
  user_supplied_context: <verbatim user-provided text, NOT externally fetchable>
  claim_text: <claim drawing on this private context>

PROCEDURE:
  1. NO external tool calls (data is private). tool_calls_made may be empty
     for THIS verifier type only.
  2. Decompose claim into atomic facts (FActScore-style, Min et al. 2023,
     arXiv:2305.14251).
  3. Substring/entailment-match each atomic fact against user_supplied_context only.
  4. faithfulness = supported_atoms / total_atoms.

VERDICT LABEL: GROUNDED 🔒 / UNGROUNDED 🔓 / PARTIALLY_GROUNDED — NOT
"VERIFIED" (no external truth check; verifier checks faithfulness to provided
context only).

CRITERIA APPLIED: faithful-representation-of-provided-context only.
```

## Verifier (f) — scientific (citing studies / making scientific assertion)

```
INPUTS:
  cited_paper_id: <DOI or arXiv ID, or full citation>
  expected_finding_quote: <if provided>
  claim_text: <claim>

PROCEDURE:
  1. METADATA: Semantic Scholar /graph/v1/paper/DOI:{DOI}?fields=
     title,year,venue,citationCount,influentialCitationCount,referenceCount
  2. RETRACTION CHECK (mandatory, both for cross-validation):
     - Crossref: api.crossref.org/works/{DOI}, `update-to` array
     - OpenAlex: api.openalex.org/works/doi:{DOI}, `is_retracted`
     - Cross-check both — OpenAlex `is_retracted` had ~2300 misclassifications
       Dec 2023 / Mar 2024 (arXiv:2403.13339)
     - If retracted → CONTRADICTED with retraction notice quote.
  3. FETCH PDF/HTML; substring-match expected_finding_quote with ≥200 chars
     surrounding context.
  4. PREPRINT-ONLY (no peer-reviewed version): PARTIALLY_VERIFIED + flag.
  5. arXiv version-churn: OAI-PMH arXivRaw. ≥4 versions = Hansson #6 yellow flag.
  6. NEGATIVE CONTROL: search Semantic Scholar for citing papers with
     "contradict|refute|fail to replicate" in title/abstract.
  7. HANSSON PATHOLOGY CHECKS:
     #1 (authority): is cited source primary or secondary? Check Semantic
        Scholar references graph.
     #2 (unrepeatable): replication-tagged citations? PubPeer threads if accessible.
     #4 (unwillingness to test): no replication after 5+ years for testable claim?
     #5 (disregard refuting): retraction + negative-control results.
     #6 (built-in subterfuge): version-churn flag.
  8. OUT_OF_SCOPE_PATHOLOGIES: #3 ("handpicked examples") + #7
     ("explanations abandoned") are NOT mechanizable at minutes scale by
     an AI verifier. MUST be explicitly listed in `out_of_scope_pathologies`.

CRITERIA APPLIED: source-existence + verbatim + external consistency
(retraction) + cross-corroboration + Hansson #1, #2, #4, #5, #6.
```

## Hard rules — do not violate

1. **`tool_calls_made` MUST be non-empty AND ≥1 call must target a domain NOT in claim's URLs.** Same-domain-only fetch = automatic FAIL.
2. **`evidence_quotes` MUST be verbatim AND substring-match the fetched page text downstream.** Non-matching → auto-FAIL. **LOAD-BEARING ANTI-HALLUCINATION CHECK.**
3. **≥200 chars surrounding context recorded** for each evidence quote (Hansson SEP §5.1: cherry-picked quotation is a documented pseudoscience tactic).
4. **Source-before-claim ordering** in verifier prompt. SycEval: claim-first = +5pp sycophancy.
5. **Negative-control search required** for verifier types (c) subjective + (f) scientific. Output includes `negative_control_attempted: {performed, queries, findings}`.
6. **INSUFFICIENT_EVIDENCE requires ≥3 distinct queries documented** with reason each failed; otherwise auto-downgrade to `verifier_retry_needed`.
7. **Retraction check (Crossref + OpenAlex, both)** mandatory for any DOI-cited claim.
8. **Verbatim quote required** for every VERIFIED / CONTRADICTED / PARTIALLY_VERIFIED verdict.
9. **Output must be valid JSON** per schema.
10. **Citations resolving to "source does not resolve" → REMOVED**, not substituted.
11. **`source_published_at` MUST be recorded for every non-private verdict** (from page metadata / byline / DOI metadata / commit date; "unknown" only if genuinely absent). `fetched_at` is mechanically recorded in `tool_calls_made`. Verifier does NOT classify volatility — recording dates is a fact, classifying "shelf life" is an LLM judgment the user should make themselves.
12. **`verdict` MUST be exactly one of the enum values** — factual types: `VERIFIED` / `CONTRADICTED` / `PARTIALLY_VERIFIED` / `INSUFFICIENT_EVIDENCE`; private type: `GROUNDED` / `UNGROUNDED` / `PARTIALLY_GROUNDED`. No self-invented labels (e.g., `verified_with_caveat`, `MISLEADING_WITHOUT_CONTEXT`) — non-enum verdicts silently fall through downstream consumers (source-check-max combination table, headline UX mapping). Any nuance / qualification → `caveats[]`, not the verdict field.

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
      "context_window": "≥200 chars surrounding text",
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

| Verdict | Headline format |
|---|---|
| VERIFIED | `✅ verified · <1 sentence, ≤140 chars>` |
| PARTIALLY_VERIFIED | `⚠️ partial · passed N/M — failed: <list>` |
| CONTRADICTED | `❌ contradicted · <1 sentence with refuting fact>` |
| INSUFFICIENT_EVIDENCE | `🔵 insufficient · <what's missing>` |
| GROUNDED | `🔒 grounded in your context · <1 sentence>` |
| UNGROUNDED / PARTIALLY_GROUNDED | `🔓 not grounded in your context · <missing atoms>` |

**Timing — show the source's date, let the user judge staleness.** Every non-private headline must end with `· src dated: <source_published_at>`. We do NOT show the fetched-at time (it's always ≈ "now" and adds no signal), and we do NOT classify the source as "stale" or "fresh" (that depends on the user's use case — a 2019 numerical-methods result is fresh; a 2019 LLM benchmark is ancient). User sees the source's date, decides themselves.

```
Example:
✅ verified · Sonnet 4.5 input price is $3/M tokens · src dated: 2025-09-29
✅ verified · Adam optimizer convergence proof in Kingma 2014 · src dated: 2014-12-22
🔵 insufficient · no n≥1000 survey found for "Zig vs Rust embedded"
```

For `private` (GROUNDED / UNGROUNDED) claims, omit the source date — dates come from user's own context.

Below user-facing answer:
```
[source-check: N claims (✅<V>, ⚠️<P>, ❌<C>, 🔵<U>, 🔒<G>, 🔓<UG>); types: <count>; oldest source: <date>; warnings: <list or none>]
```

The `oldest source` field lets the user spot at a glance whether anything rests on aged data — without the skill deciding for them what "aged" means.

Audit appendix (collapsed): full per-verifier JSON outputs, tool-call log, evidence + context.

## Optional custom agent

Define a dedicated verifier at `~/.codex/agents/source-verifier.toml`. **Cross-family setup highly recommended for high-stakes** — pin a different model family than the drafter via the `model` field. Cross-family reduces preference leakage from 28-37% to ±1.5% (Preference Leakage, ICLR 2026).

```toml
name = "source_verifier"
description = "Multi-criteria source-check verifier. MUST use web tools."
model = "gpt-5.5"  # pin different from drafter for cross-family
model_reasoning_effort = "high"
sandbox_mode = "read-only"
developer_instructions = """
You verify external claims against live web sources using claim-type-specific
criteria. MUST use web tools. Return strict JSON with verbatim quotes,
≥200 chars context, tool_calls_made with domain_distinct_from_claim flag.
"""
```

## Limits / Caveats (honest)

- **Subjective consensus thresholds (60%/40-60%/<40%) are engineering judgment**, not literature-supported. Calibrate against post-deployment data.
- **OpenAlex `is_retracted` had ~2300 misclassifications** late 2023 / early 2024 (arXiv:2403.13339). Mandatory Crossref cross-check is the mitigation.
- **Hansson #3 + #7 are NOT mechanizable** at minutes scale. Explicit `out_of_scope_pathologies` field prevents silence-as-clearance.
- **Same-family drafter + verifier may share biases** (SycEval 78.5% sycophancy persistence). Cross-family verifier setup recommended for high-stakes.
- **Niche subjective claims** (no n≥1000 survey) → INSUFFICIENT_EVIDENCE is honest verdict.
- **Future predictions never VERIFIED as "Y will happen"** — only as "X predicted Y on date Z".
- **Private data uses GROUNDED / UNGROUNDED**, not VERIFIED.
- **PubPeer has no public keyless JSON API**; Cochrane API is auth-gated. Both partially limit Hansson mechanization.
- **Does NOT catch subtle paraphrase distortions** where source mostly matches but qualifier dropped (see `audit-loop` synthesis-from-fetched-source structural limit).

## Handshake with other skills

- **`audit-loop`**: algorithmic / mechanism / correctness reasoning. Orthogonal.
- **`source-check-max`**: experimental dual-verifier; not yet validated. Use source-check for production.
- **User's `red-team-process` memory**: heavier PROVENANCE BLOCK for quantitative external-fact high-stakes single answers.
