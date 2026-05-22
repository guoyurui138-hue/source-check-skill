# source-check-max (plugin)

Dual-verifier extension to [source-check](../source-check/README.md) for high-stakes factual citations.

## What it does

For each factual citation in your draft, source-check-max spawns **two independent verifiers in parallel**:

- **V1** fetches the URL the agent gave (standard source-check Verifier (a))
- **V2** receives only `author + year + title` — *not* the URL — and must independently rediscover the source via search, then pass a metadata gate (author/year/title match) before content verification

The two outputs are combined via a fixed 8-state headline table:
`STRONG_VERIFIED` / `SUSPICIOUS_BOTH_PASS` / `CONFLICT_REVIEW` / `LIKELY_WRONG` / `WEAK_VERIFIED` / `LIKELY_FABRICATED` / `UNVERIFIABLE` / `CONFLICT_REVIEW`.

Catches two failure modes the single verifier can't:
1. **Wrong-URL citations** — V2 surfaces the correct URL when V1's URL points elsewhere
2. **Fully fabricated papers** — V2 fails the metadata gate (no real source exists)

Validated separately with n=13 paired test (13/13 predictions matched, including all 3 wrong-URL cases and all 6 fabrication subtypes).

## When to use

- Legal precedent / case-law citations
- Medical literature / drug references
- Financial regulation / contract references
- Scientific claims driving material decisions

For everyday work, use [`source-check`](../source-check/) (single verifier, ~2× cheaper).

## Install in Cowork

**Requires** the `source-check` plugin installed first.

```bash
/plugin marketplace add Mercer8964/source-check-skill
/plugin install source-check@source-check-skill
/plugin install source-check-max@source-check-skill
```

Or drag both `.plugin` files into the Cowork chat.

## License

MIT.
