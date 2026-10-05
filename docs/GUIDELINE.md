# XSCanner — Project Guideline

A brief for the agent building XSCanner. It describes **what** to build and **why**, and leaves most **how** decisions to you. Where something is marked *open question*, decide, write down why, and keep going. Don't block on it.

---

## 1. One-line goal

Watch X (Twitter) for posts about Pump.fun tokens (Solana), and turn the raw chatter into a **number that measures attention**: how much, how fast it is growing, and how much of it is real.

## 2. Why

New Pump.fun tokens live or die on social attention, and that attention moves faster than on-chain data shows it. A tool that answers *"which tokens are people suddenly talking about, and is it organic?"* is useful for research, alerting, and dashboards.

## 3. Scope

**In scope (v1)**
- Ingest posts from X that mention tokens.
- Detect and normalize token references to one canonical ID (the Solana mint address).
- Compute attention metrics over rolling time windows.
- Rank tokens by attention and surface "risers".
- Store history so trends can be replayed and backtested.
- A simple output surface: CLI and/or JSON API first, dashboard later.

**Out of scope (v1)**
- Trading, wallet connections, or buy/sell signals. This is a measurement tool, not financial advice.
- Chains other than Solana / launchpads other than Pump.fun. Keep the design open to adding them later.
- Sentiment analysis beyond something simple (it's a stretch goal; see §6).

## 4. How the system works

```
 X source ──► Ingestor ──► Extractor ──► Resolver ──► Store ──► Scorer ──► Output
 (API/stream)  (raw posts)   (token refs)  (mint addr)  (DB)     (metrics)  (CLI/API/alerts)
```

### 4.1 Ingestion: getting posts from X
- **Preferred:** the official X API (filtered stream or recent search). Access tiers, rate limits, and pricing change often. Check the current terms before you design around them, and keep the source behind an interface (`PostSource`) so it can be swapped.
- **Also build:** a `FixturePostSource` that replays saved posts from disk. It is needed for tests and for development without API credits.
- **Avoid:** scraping that breaks X's Terms of Service. If a third-party data provider is used, it plugs in as another `PostSource`.
- **Query seeds:** `pump.fun`, `pump.fun/coin/`, contract-address patterns, `$` cashtags, plus known CT (crypto Twitter) keywords. Make these configurable.

### 4.2 Extraction: finding token references in a post
Detect, from strongest signal to weakest:
1. **Mint addresses.** Base58 strings of 32–44 chars. Pump.fun mints usually end in `pump`, which is a strong hint.
2. **URLs** to `pump.fun/coin/<mint>`, DexScreener, Birdeye, GMGN, Photon, etc. Expand t.co links.
3. **Cashtags** (`$TICKER`). These are ambiguous: many tokens share a ticker.
4. Bare names / hashtags. Weakest signal; leave for later.

Every extraction keeps its **confidence** and **method**, so that weak matches can be weighted down later.

### 4.3 Resolution: one token, one ID
- The canonical key is the **mint address**.
- Resolve and enrich it from public sources (Pump.fun data, DexScreener, a Solana RPC): name, symbol, creation time, bonding-curve / migration status, market cap, liquidity.
- **Ticker collisions:** map a `$TICKER` to a mint only when the context makes it clear (the same post or thread contains the CA, or one candidate dominates recent mentions). Otherwise keep it as "unresolved".
- Cache aggressively. Market data is context for the attention score, not part of it.

### 4.4 Storage
- Start simple: SQLite or Postgres. Store raw posts so metrics can be recomputed when formulas change.
- Core entities (a suggested starting point):
  - `post` (id, author_id, text, created_at, metrics snapshot, raw JSON)
  - `author` (id, handle, followers, account age, verified flag, bot-score)
  - `token` (mint, symbol, name, created_at, chain metadata)
  - `mention` (post_id, mint, method, confidence)
  - `attention_snapshot` (mint, window, timestamp, metric values)

## 5. Quantifying attention (the core of the project)

Compute per token, per rolling window (e.g. 5m / 15m / 1h / 6h / 24h):

| Metric | What it captures |
|---|---|
| **Mention count** | Raw volume |
| **Unique authors** | Breadth. More robust than raw count against spam |
| **Velocity** | Mentions per minute in the current window |
| **Acceleration** | Velocity now vs. the previous window (is it taking off?) |
| **Engagement-weighted reach** | Sum of likes/reposts/replies/views, log-scaled |
| **Author-weighted reach** | Mentions weighted by author influence (followers, log-scaled, with a cap so one whale doesn't dominate) |
| **Novelty** | Share of authors mentioning this token for the first time |
| **Concentration** | How much volume comes from the top N authors (high = likely shilling/raid) |

**Composite Attention Score.** Combine the metrics into one 0–100 score. Requirements:
- **Transparent.** Store the component values next to the score and expose them.
- **Config-driven weights.** No magic numbers buried in code.
- **Normalized** against the population of tokens tracked at the same time (e.g. percentile or z-score), so scores can be compared across quiet and busy market periods.
- **Discounted for manipulation:** weigh down low-quality authors, duplicate or near-duplicate text, coordinated bursts, and high concentration.

**Quality / anti-manipulation signals (important).** Crypto Twitter is full of bots and paid raids. At minimum:
- Near-duplicate text detection (e.g. MinHash / shingling).
- Author heuristics: account age, follower/following ratio, posting rate, default avatar.
- Burst detection: many new accounts posting the same CA within seconds.
- Report an **"organic ratio"** next to the score instead of silently filtering.

## 6. Stretch goals (after the MVP)
- Lightweight sentiment / stance (hype vs. warning such as "rug", "scam").
- Correlate attention with on-chain data (holders, volume, price) and measure lead/lag. Does attention come before price?
- Alerts (Telegram/Discord webhook) when a token crosses score or acceleration thresholds.
- Web dashboard: leaderboard, per-token timeline, author breakdown.
- Influencer tracking: which accounts consistently mention tokens early.

## 7. Suggested milestones

1. **Skeleton.** Repo layout, config, logging, `PostSource` interface + fixture source, tests running.
2. **Extraction.** Mint/URL/cashtag extractor with unit tests on real-world-shaped examples (include tricky negatives such as random base58 strings and wallet addresses).
3. **Resolution + storage.** Mint lookup, caching, DB schema, ingesting fixtures end to end.
4. **Metrics v1.** Windowed counts, unique authors, velocity, acceleration. CLI prints a leaderboard.
5. **Live source.** Connect the real X source behind the same interface. Respect rate limits; back off on errors.
6. **Composite score + quality signals.** Score, organic ratio, near-dupe detection.
7. **Output.** JSON API and/or simple dashboard; optional alerts.

Each milestone should be runnable and tested before moving on.

## 8. Engineering expectations
- **Language:** your choice. Python (fast iteration, good data tooling) or TypeScript (good Solana / web ecosystem) are both reasonable. Pick one, justify it briefly in the README, and stay consistent.
- Secrets (API keys, RPC URLs) come from environment variables / `.env`, never committed. Provide `.env.example`.
- Deterministic tests that run offline against fixtures. No live API calls in CI.
- Keep the pipeline stages decoupled so each can be tested and replaced on its own.
- Document the scoring formula in `docs/SCORING.md` and keep it in sync with the code.
- Prefer small, reviewable commits.

## 9. Open questions (decide and document)
- Which X data access tier/provider is realistic for the budget? This decides polling vs. streaming and how much data we get.
- How should we handle tokens before they have a mint (pre-launch hype)? Track them as unresolved tickers, or ignore them?
- What retention period do we need for raw posts, given storage and X's ToS?
- Should "attention" include posts *about* a token that don't name it directly (replies in a thread under a CA post)? Probably yes, through thread context, but at lower weight.

## 10. Ground rules
- Follow X's Developer Agreement and rate limits. No ToS-violating scraping.
- No trading or financial-advice features. Output is descriptive data.
- Don't store more personal data about authors than the metrics need.
