# source-check

**Catches the two ways AI agents lie most often: fabricated citations and silently stale info.**

For [Claude Code](#claude-code), [Codex](#codex-codex-cli), and [OpenClaw](#openclaw). MIT licensed. Validated with a [pre-registered n=13 test](experiments/v8-validation.md) — 13/13 cases handled correctly.

---

## What it looks like

**Without source-check** — agent confidently writes:

> Hashimoto et al. (2024) proved diffusion-model convergence in *Foundations of Score-Based Generative Modeling*, JACM 71(4), DOI 10.1145/3676932.3676933.

This is fabricated. The DOI doesn't resolve. JACM 71(4) contains no such paper. No matching paper exists in any indexer.

**With source-check** — a verifier subagent spawns, fetches live sources before reading the claim, and tags the output:

> ❌ **contradicted** · source does not resolve (DOI returns 404 on doi.org and Crossref; JACM 71(4) ToC excludes this title; Google Scholar / Semantic Scholar return 0 results for title+author+year) · src dated: unknown

The verifier cited a verbatim Crossref response, made tool calls to three domains distinct from the claim's URL, and ran a negative-control search — all mechanically checkable downstream. It also catches retracted papers (Wakefield 1998), wrong author attribution (e.g., "Lewis et al. introduced GPT-3" — that was Brown et al.), stale "latest" claims, and unsourced future predictions. See the [validation experiment](experiments/v8-validation.md) for the full set.

---

## Why this exists

AI citation fabrication is empirically severe — GPT-3.5 ~55%, GPT-4 ~18% ([Walters & Wilder 2023](https://www.nature.com/articles/s41598-023-41032-5)); 11.4–56.8% across 10 deployed LLMs over 69,557 citation instances ([arXiv:2603.03299](https://arxiv.org/abs/2603.03299)). The fake citations pattern like real ones — plausible authors, journals, DOIs — so a reader can't tell without checking.

The parallel failure: time-sensitive claims answered from training memory without any signal to the user that the answer is recall-based.

Both share one root cause — no live external grounding. This skill enforces grounding with type-appropriate criteria, refusing to skip claims as "unverifiable" (which would itself be a [known pseudoscience pathology](https://plato.stanford.edu/entries/pseudo-science/) — "unwillingness to test").

---

## Install

The skill auto-triggers when your draft contains an external claim. No command to remember.

### Claude Code

```bash
mkdir -p ~/.claude/skills/source-check
cp claude-code/SKILL.md ~/.claude/skills/source-check/SKILL.md
```

### Codex (codex CLI)

```bash
mkdir -p ~/.agents/skills/source-check
cp codex/SKILL.md ~/.agents/skills/source-check/SKILL.md
```

Codex announces `"Spawning source-check verifiers for N claims"` when triggered. For high-stakes work, define a `source_verifier` agent in `~/.codex/agents/` pinned to a different model family than your drafter — cross-family verification reduces preference leakage from 28–37% to ±1.5% ([Preference Leakage, ICLR 2026](https://arxiv.org/abs/2502.01534)).

### OpenClaw

```bash
mkdir -p ~/.openclaw/skills/source-check
cp openclaw/SKILL.md ~/.openclaw/skills/source-check/SKILL.md
```

Uses `sessions_spawn` with `context: "isolated"` for verifier independence. Cross-family setup is easiest here via `agents.list[].subagents.model`.

---

## How it works

```
claim ─► Step 0 classifier (route, no skip) ─► 1 of 6 type-specific verifiers:

  factual_citation → fetch URL · retraction check · verbatim quote substring-match
  time_sensitive   → fetch authoritative · cross-corroborate ≥2 independent domains
  subjective       → measured signal (n≥1000 survey / peer-reviewed benchmark)
  future           → verify predictor-attribution (NOT predicted-future-state)
  private          → faithfulness to user's supplied context (GROUNDED / UNGROUNDED)
  scientific       → Crossref + OpenAlex retraction cross-check · Hansson pathology checks
                     ↓
        strict JSON output (verdict from enum · verbatim quote · ≥200 chars context
                            · ≥1 tool call to domain ≠ claim's URL)
                     ↓
                you see:  ✅ verified  ⚠️ partial  ❌ contradicted  🔵 insufficient
                          🔒 grounded  🔓 ungrounded   + source date
```

**Load-bearing design choices** (the things that make this not-just-another-LLM-grading-itself):

- **Source-before-claim ordering** — verifier reads the source *before* the agent's claim, blocking claim-first sycophancy (+5pp per [SycEval](https://arxiv.org/abs/2502.08177)).
- **Different-domain tool call required** — verifier must hit a domain not in the claim's URLs; closes the "fetch only the cited URL and rubber-stamp it" loophole.
- **Verbatim quote with substring-match** — every supported/contradicted finding needs a quote that *mechanically substring-matches* the fetched page. Fabricated quotes fail this check by construction. **This is the load-bearing anti-hallucination guarantee.**
- **Retraction cross-check** — DOI claims hit both Crossref `update-to` and OpenAlex `is_retracted` (single-source unsafe — [OpenAlex had ~2300 misclassifications](https://arxiv.org/abs/2403.13339)).
- **Source date shown, staleness not classified** — verifier records when the source was published; *you* judge if that's stale. A 2019 numerical-methods result is fresh; a 2019 LLM benchmark is ancient — only you know your use case.
- **No skip mechanism** — even "subjective" / "future" / "private" claims get verified, just against *different criteria*.

---

## Evidence it works

[Full experiment + raw verifier outputs.](experiments/v8-validation.md) Headline:

| Test | Result |
|---|---|
| Citation fabrication caught (fake DOI · fake arXiv · wrong attribution · retracted paper) | **4/4** ✅ |
| Stale "latest" claim caught | **1/1** ✅ |
| Real-and-correct citations correctly VERIFIED (false-positive control) | **3/3** ✅ |
| Subjective / future / private claims routed off VERIFIED | **3/3** ✅ |
| Schema-enum compliance after one-line patch | **3/3** ✅ |
| Different-domain tool call (anti-rubber-stamp) | **12/12** non-private cases |

**Why the experiment is credible** (not just "AI passed its own tests"):

1. **Pre-registered predictions** — all 13 verdicts written down *before* any verifier ran. No post-hoc fitting.
2. **Ground truth independent of the skill** — we fabricated the fake DOIs ourselves; Wakefield 1998 has been a public retraction since 2010; Vaswani 2017 is verifiable.
3. **Verifiers blind to predictions and ground truth** — each one saw only the claim and (if given) the URL.
4. **Positive AND negative controls** — both "should VERIFY" and "should CONTRADICT" cases, to catch false positives AND false negatives.
5. **Failure surfaced and patched in-loop** — 3 verifiers self-invented off-schema verdict labels; one new hard rule pinned verdict to the enum; re-test was 3/3 enum-compliant.

---

## Limitations

- Does NOT catch subtle paraphrase distortions where source mostly matches but a qualifier is dropped. That's `audit-loop`'s job (separate skill, can fire alongside).
- Subjective thresholds (60% / 40–60% / <40%) are engineering judgment — calibrate against your post-deployment data.
- Same-family drafter + verifier may share biases ([SycEval](https://arxiv.org/abs/2502.08177): 78.5% sycophancy persistence). For high-stakes, configure cross-family verifiers.
- Niche subjective claims (no n≥1000 survey) return INSUFFICIENT_EVIDENCE, not forum-sentiment fallback.
- Future predictions are never VERIFIED as "Y will happen" — only as "X predicted Y on date Z".
- Hansson #3 (handpicked examples) and #7 (abandoned explanations) are not mechanizable at minutes scale — explicitly listed in `out_of_scope_pathologies` so silence isn't mistaken for clearance.
- PubPeer has no public keyless JSON API; Cochrane is auth-gated — both partially limit replication signals.
- n=13 validation is small — real-world use will surface failure modes that didn't appear here.

There's a separate `source-check-max` variant (not in this repo) for legal / medical / financial citations where dual-verifier cost is justified.

---

## License & contributing

MIT — use, modify, redistribute freely.

If you hit a failure mode in the wild, the highest-value bug report is: the specific claim + source URL + the verifier's JSON output. Open an issue.
