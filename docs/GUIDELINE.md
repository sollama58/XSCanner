# XSCanner — Project Guideline

A brief for the agent building XSCanner. It says **what** to build and **why**, records decisions the research already supports, and leaves the remaining **how** to you. Where something is an *open question*, decide, write down why (in `docs/DECISIONS.md`), and keep going.

Background evidence, numbers and source links are in [`RESEARCH.md`](RESEARCH.md). Prices, rate limits and platform features in this space change monthly. Anything tagged **[verify]** must be checked against the official docs before you build on it.

---

## 1. Goal

Watch X (Twitter) for discussion of newly launched Solana memecoins, starting with Pump.fun. Turn that discussion into an **attention score** for each token that answers three questions:

1. **How much** attention is there, measured as independent voices rather than raw post count?
2. **How fast** is it changing, and is that change unusual for a token of this age?
3. **How real** is it: organic discovery, or bots, raids and paid shilling?

## 2. Why this is worth building (and where the gap is)

- About 1 in 4 Pump.fun coins is created *in response to* an X post, and fewer than about 2% ever graduate from the bonding curve. A coin's whole life is short and driven by social media, so the relevant time horizon is **minutes to hours**.
- Research consistently finds that **social volume predicts trading volume and volatility** better than it predicts price direction. **Lagged volume beats sentiment.** Clean volume therefore comes before clever NLP.
- Manipulation is the norm, not the edge case:
  - One 2026 study found **over 80% of memecoins with >100% returns showed artificial growth**.
  - Pump-and-dump research shows bot share rises sharply during pumps.
  - Paid KOL posts are almost never disclosed.
- Existing tools leave clear gaps:
  - LunarCrush, Santiment and Kaito are built around tickers and listed coins, and their scoring is opaque.
  - GMGN, Axiom and Photon have "X tracker" features, but those are raw feeds of posts.
- **XSCanner's niche:**
  - A transparent, **authenticity-adjusted, contract-address-keyed, minute-resolution** attention measure for fresh mints.
  - Each token is tracked from the moment it is created.
  - The score is validated against on-chain flow.

## 3. Scope

**v1 in scope**
- Find tokens being discussed on X, and find X discussion of newly created tokens (both directions, see §4).
- Resolve every reference to a canonical **mint address**.
- Compute windowed attention metrics and a composite score, with each component visible.
- Measure authenticity: how much of the attention is coordinated versus independent.
- Store history so scores can be replayed, re-scored and backtested.
- Outputs: CLI leaderboard and JSON API first. Dashboard and alerts after that.

**Out of scope**
- Trading, wallet connection, buy/sell calls. This is a measurement tool and its output is descriptive.
- Anything that **rewards users for posting**. X revoked API access for "InfoFi" reward apps in Jan 2026, and reward systems get gamed.
- Running scrapers on logged-in X accounts.

**Designed for, built later:** other Solana launchpads (Bonk.fun, Bags, Believe, Meteora DBC brands), sentiment analysis, and chains other than Solana.

## 4. Architecture

Discovery runs in **two directions**. This is the key structural idea:

```
            ┌──────────── X → chain ─────────────┐
 X source ──► Ingestor ─► Extractor ─► Resolver ─┤
                                                 ├─► Store ─► Scorer ─► Outputs
 Chain feed ─► New-mint watcher ─► Query planner ┘          ▲          (CLI/API/alerts/dashboard)
 (PumpPortal)        │                   │                  │
                     └── metadata ───────┴─► X queries ──────┘
            └──────────── chain → X ─────────────┘
                              Enricher (market + on-chain) ──► Store
```

- **X → chain:** posts arrive, contract addresses, links and cashtags are extracted, and those resolve to mints.
- **chain → X:** a new mint appears on-chain. Its metadata (the creator-supplied `twitter` field, which may be a profile or a *tweet URL*) and its mint address seed targeted X queries.

The chain → X direction **controls cost**. Data is cheap on the chain side and expensive on the X side, so spend X reads only on tokens that are showing signs of life.

### 4.1 X data access (biggest cost and risk driver)

The research found the following as of Oct 2026 **[verify all numbers]**:

| Route | Cost | Notes |
|---|---|---|
| Official X API, pay-per-use (default since Feb 2026; Basic/Pro closed to new buyers) | about **$5 per 1k posts read**, $10 per 1k users; cap of about 2–3M reads/month | Only ToS-clean route. Recent search (7 days), full-archive search, filtered stream (1 connection, 1,000 rules), and **counts endpoints billed per request rather than per post** |
| Third-party providers (TwitterAPI.io, SocialData, Apify actors) | about **$0.15–0.40 per 1k posts** | Built on scraping. Some offer rule-based webhook/websocket delivery. Supply can disappear (X shut down Nitter in Aug 2026) |
| Grok API X Search | about $5 per 1k posts plus tokens | Only useful for LLM summaries, not bulk counting |

Required design:
- **`PostSource` interface** with interchangeable implementations: `FixtureSource` (offline replay; required), `XOfficialSource`, `ThirdPartySource`.
- **`CountSource` interface.** The official **counts endpoint** returns mention-volume time series for a query such as a mint address without paying for each post. This may be the cheapest clean way to get the volume backbone, with full post content pulled only for a sample or for hot tokens. **Validate this pricing first. If it holds, it shapes the whole design.**
- A **budget governor** that tracks spend per source per day and degrades gracefully: hot tokens get full posts, warm tokens get counts only, cold tokens get nothing.
- Query building blocks (all configurable):
  - `url:pump.fun`
  - mint addresses as keywords
  - `$` cashtag operator (now available on all tiers **[verify]**)
  - `has:cashtags`, `-is:retweet`, `lang:`
  - the new `min_likes:` / `min_reposts:` operators
- **X Smart Cashtags** (launched Apr 2026; shows a chart when a contract address is pasted) means more posts now contain **raw contract addresses**. That is good news for extraction accuracy.
- **Decision for you:** official-only, third-party-only, or a hybrid. A hybrid would use official counts as a clean volume baseline and a third-party feed for post content. Record the ToS trade-off in `DECISIONS.md`.

### 4.2 Chain-side feeds

- **New mints and graduations:** PumpPortal websocket `subscribeNewToken` and `subscribeMigration` (free). Per-token trade streams cost about 0.01 SOL per 10k events **[verify]**.
- **Token metadata and state:**
  - Pump.fun's unofficial `frontend-api-v3` (`/coins/{mint}`) gives `twitter`/`website`/`telegram`, `creator`, `complete`, `usd_market_cap` and `is_currently_live`.
  - It is undocumented and can break, so wrap it and degrade gracefully.
- **Market data:**
  - DexScreener: free, no key, about 300 requests/min on pair/token endpoints.
  - Fallbacks: GeckoTerminal and Jupiter Price v3.
  - Paid upgrade path: Helius or Birdeye.
- **Paid-promotion signals:** DexScreener `token-profiles`, `token-boosts` and `orders` show who paid for visibility. Record these; they are attention that was *bought*.
- **Risk data:** RugCheck report (top-holder concentration, insiders), creator-wallet sells, same-slot bundle buys.

**Pump.fun specifics to handle:**
- The **`pump` suffix on a mint is a vanity convention, not proof** of where a token came from. Identify the launchpad by program ID and config, not by suffix (one open-source project saw about 72k false positives from suffix matching).
- Both legacy SPL tokens and **Token-2022** (`create_v2`, since Nov 2025).
- **Mayhem Mode** coins have bot trading for their first 24h. Flag them so their on-chain activity isn't mistaken for organic demand.
- **Custom Pairs** (Sep 2026): the quote asset may not be SOL. Never assume SOL when converting to USD.
- Graduation happens at about 85 SOL raised, about 800M of 1B tokens sold. Graduates go to **PumpSwap** (older ones went to Raydium).
- Record each coin's **fee mode** (creator fee, holder rewards, cashback, mayhem). It changes incentives to promote the coin.

### 4.3 Extraction

From strongest signal to weakest; record `method` and `confidence` for each:

1. **Contract addresses.** Base58, 32–44 chars, validated as a real on-chain mint. This filters out wallet addresses, transaction signatures and random strings.
2. **URLs:**
   - `pump.fun/coin/<mint>`
   - DexScreener, Birdeye, GMGN, Photon, Axiom, Bullx links
   - Expand `t.co` links
   - Keep the **`?ref=` referral codes**. They are a strong signal of paid or affiliate shilling (see §6).
3. **Cashtags.** Ambiguous, because many tokens share a ticker. Resolve only with context (§4.4).
4. **Names, hashtags, images.** Leave for later (OCR of CA screenshots is a stretch goal).

Also capture the following per post:
- `t.me` invite links. Research found most accounts sharing them were bots linked to pump groups.
- Reply/quote targets.
- Whether the post is a reply inside a thread that contains a CA. Such posts count as attention at lower weight.

### 4.4 Resolution and the "narrative" layer

- The canonical key is the **mint**.
- **Ticker → mint:**
  - Accept only if the same post or thread contains the CA, or if one candidate dominates recent CA-bearing mentions.
  - Otherwise keep the mention as `unresolved` and attach it at the narrative level.
- **Narrative clustering (a differentiator):** a single viral post or meme usually spawns many copycat mints with the same name or ticker.
  - Group them under one `narrative` (source post plus matching name or ticker plus close creation times).
  - Score attention at **both** the narrative level and the per-mint level.
  - Report each mint's **share of its narrative's attention**: which copycat is winning.

### 4.5 Storage

Start with Postgres, or SQLite while prototyping. If minute-level series grow large, consider TimescaleDB or DuckDB. Keep **raw posts** (within the source's ToS) so the scoring can be re-run.

Core entities:
- `post`
- `author`: account age, followers, a bio snapshot, and **bio-change history**. KOLs often reveal launches in their bio.
- `mention`
- `token`: launchpad, program, quote asset, fee mode, mayhem flag
- `narrative`
- `token_market_snapshot`
- `promo_event`: boosts and paid profiles
- `cluster`: coordinated groups
- `attention_snapshot`: every component value plus the score and the score version

## 5. Quantifying attention

Work in time bins (1 minute natively, aggregated to 5m/15m/1h/6h/24h). Most bins for most tokens contain 0–3 posts, so **use count-appropriate statistics, not plain z-scores**.

### 5.1 Base quantities per token per bin
- `posts`: raw post count. Kept for reference but **not used as the main signal**.
- `authors`: unique authors.
- `effective_voices`: unique authors after collapsing each coordinated cluster to about one voice (§6). **This is the core measure.**
- `reach`: author-weighted, `Σ log(1+followers)` capped per author. Optionally use follower-*quality* (who follows them) in the TweetScout/Sorsa style.
- `engagement`: log-scaled likes, reposts, replies and quotes. Weight each engagement by the engager's quality when you have it (Kaito's idea).
- `novelty`: share of authors mentioning this token for the first time.
- `concentration`: share of volume from the top-N authors or clusters.
- `kol_mentions`: mentions from a curated or learned list of influential accounts, kept separate.

### 5.2 Dynamics
- **Decayed attention:** `S ← S·e^(−λΔt) + w_new`. This can be updated incrementally. Set λ from a half-life you **estimate from the data**. No published memecoin attention half-life exists; this is an open research gap.
- **Velocity and acceleration:** the ratio of a fast to a slow EWMA (e.g. τ = 5 min vs. 60 min), MACD-style.
- **Surprise:** a Poisson (or negative-binomial) tail probability `P(X ≥ c_t | λ_baseline)`. This handles small counts properly.
- **Cohort baseline (important for new mints with no history):** compare a token against **all mints of the same age** (e.g. "at 20 minutes old, this token is in the 99th percentile of effective voices"). This solves the problem of having no baseline at all.
- **Burst state** (optional): Kleinberg burst levels, which are easy to read in a UI.
- **Virality** (stretch goal): fit a Hawkes process. Its branching ratio shows whether attention is self-sustaining (n* → 1) or fading. Its baseline term separates outside shocks (a KOL post) from people reacting to each other.

### 5.3 Composite Attention Score (0–100)

A starting point; the weights are config, not code:

```
score = 100 · percentile_vs_live_universe(
          decayed_effective_voices
        × f(velocity, surprise)
        × authenticity            # 1 − coordinated share, see §6
        × quality                 # author-weight factor, bounded
       )
```

Requirements:
- Store and expose **every component**, the **score version**, and the config hash.
- Normalize both against the live universe (comparable across quiet and busy markets) and against same-age cohorts.
- Show an **organic ratio** and **flags** next to the score (e.g. `RAID_DETECTED`, `PAID_BOOST`, `MAYHEM_MODE`, `KOL_CLUSTER`, `BUNDLED_LAUNCH`). Don't just filter suspicious activity out silently.
- Document the formula in `docs/SCORING.md`. **Transparency is the product's differentiator.** LunarCrush, Kaito and Cookie are all opaque.

## 6. Authenticity and manipulation detection

Treat this as a first-class feature, not cleanup. Botometer is effectively dead for post-2023 accounts, so build lightweight in-house methods.

**Coordination clustering** (Pacheco et al. method): build an account × behavioral-trace matrix, weight it with TF-IDF, project it to account–account cosine similarity, apply a threshold, and take connected components. Traces to use:
- Co-mentioning the same CA within minutes.
- Co-replying to the same tweet within Δt (the **raid** signature).
- Near-duplicate text: MinHash/SimHash, or embeddings after stripping CAs, tickers and emoji.
- Shared referral codes or `t.me` invites.
- Synchronized posting times; clustered account creation dates; handle patterns like name+digits.

**Detecting specific patterns:**
- **Raids.** Telegram raid bots coordinate reply and repost bursts on a target post. Signs:
  - Many short or templated replies within minutes, often LLM-paraphrased.
  - A **stable roster that shows up across different coins**.
  - The roster should count as about one voice.
- **Paid KOL shilling.** Disclosure is close to nonexistent. Infer it from:
  - Several KOLs posting the same CA within a tight window.
  - Similar phrasing.
  - Recurring co-shill cohorts across coins.
  - On-chain evidence that the KOL wallet bought *before* the post.
- **LLM bots.** Look for self-disclosure phrases ("as an AI language model") and other LLM tells. The fox8 botnet was found this way.
- **Bought visibility.** DexScreener boosts and paid profiles are attention that was paid for. Show it as its own stream.

**On-chain cross-checks** (social signals alone aren't enough):
- Bundled launch (same-slot buys).
- Top-holder concentration.
- Creator sells.
- Mayhem Mode volume.
- Wash-trading patterns.

**Lead/lag check.** Did attention come before buying, which suggests organic discovery, or did buying come first, with insiders exiting into the attention?

## 7. Feature ideas, beyond the core score

Ranked roughly by value relative to effort:

1. **Live leaderboard.** Top tokens by score, with the organic ratio, flags, age, launchpad, market cap and bonding-curve progress.
2. **"Risers" alert stream.** Fires on a cohort-percentile jump plus high authenticity. Delivered by Telegram/Discord webhook.
3. **Token timeline view.** Attention components over time overlaid on price, volume and holders, with annotations for KOL posts, boosts, graduation and creator sells.
4. **Narrative board.** Viral source post → copycat mints → each mint's share of the attention.
5. **Coordination explorer.** A graph of detected clusters and raid rosters. Accounts that recur across many coins get a reputation.
6. **Early-caller / KOL ledger.** Which accounts consistently mention tokens *early* (relative to the attention curve), and how their calls went. Optionally linked to their wallets (public KOL wallet lists from Kolscan/GMGN-style sources).
7. **Graduation-probability research.** Does early X attention predict bonding-curve graduation? No study has linked these; the project's own data could answer it. This is a natural offline research notebook.
8. **Backtest/replay mode.** Replay stored data through any score version to compare formulas.
9. **Bio-change watcher.** Watch tracked KOLs for bio changes that add a CA or a launch link.
10. **Livestream signal.** Use Pump.fun's `is_currently_live` as an extra attention input.
11. **Public methodology page.** Publishing the formula and how it was validated builds trust.

## 8. Validation (how we know the score means something)

- Build a labeled set: hand-label around 200 tokens as organic, raided or shilled, and get graduation and rug outcomes from chain data.
- Measure:
  - **Lead/lag:** cross-correlation of attention vs. on-chain unique buyers, volume and volatility.
  - **Predictive value:** graduation within N hours; rug within 1 hour (compare against on-chain-only baselines).
  - **Stability:** score rank correlation under small config changes.
- Expect attention to predict **activity and volatility** more than **price direction**. Report it that way and don't overclaim.

## 9. Milestones

Each milestone ends runnable and tested:

1. **Skeleton:** repo layout, config, logging, the `PostSource`, `CountSource` and `ChainSource` interfaces, fixture sources, CI with offline tests.
2. **Extraction:** CA, URL and cashtag extraction with a fixture corpus of real-shaped posts, including hard negatives (wallet addresses, transaction signatures, fake "pump" suffixes).
3. **Chain feed:** PumpPortal new-mint and migration watcher, metadata fetch, launchpad identification by program ID, DexScreener enrichment.
4. **Store and metrics v1:** schema, unique authors, decayed attention, velocity, cohort percentile; CLI leaderboard.
5. **Live X source plus budget governor:** after pricing checks; tiered hot/warm/cold querying.
6. **Authenticity v1:** near-duplicate detection, co-mention clustering, raid detection, effective voices, flags.
7. **Composite score and `SCORING.md`;** replay/backtest.
8. **Outputs:** JSON API, alerts, then a dashboard.
9. **Research track:** lead/lag analysis, graduation-prediction notebook, half-life estimation.

## 10. Engineering expectations

- **Language:** your choice; justify it in `DECISIONS.md`.
  - Python suits the stats and data work (pandas/polars, scipy, datasketch for MinHash).
  - TypeScript matches the official `@pump-fun/pump-sdk` and Solana web tooling.
  - A Python core using HTTP/websocket APIs is reasonable.
- Async ingestion with backpressure. Reconnect websockets with backoff. Make writes idempotent, keyed on post ID and on mint + bin.
- Secrets come from the environment; commit a `.env.example` and never commit keys.
- Tests are deterministic and offline. No live API calls in CI.
- Every external API sits behind an adapter with caching, rate limiting and a fixture double. Undocumented endpoints (the Pump.fun frontend API) are expected to break.
- Version the scoring formula, and record the version on every snapshot.
- Keep commits small and reviewable.

## 11. Open questions (decide and document)

1. X access route: official, third-party, or hybrid. What monthly budget? (Ask the owner if unclear.)
2. Does the official counts endpoint really cost per request, and at what bin granularity?
3. Should v1 cover Bonk.fun and Bags, or only Pump.fun? Bags ties fee-earners to X handles, which links tokens to accounts cheaply.
4. How long to keep raw posts, given each source's ToS?
5. Thread context: how much weight should a reply count for when it doesn't mention the token but sits under a CA post?
6. Where should the curated KOL list come from: hand-made, learned from early-caller performance, or third-party (Sorsa/TweetScout)?

## 12. Ground rules

- Follow the ToS of every data source. Don't run scrapers on logged-in X accounts. Note X's ToS damages clause and the Aug 2026 Nitter takedown.
- No trading features and no financial advice. Present output as descriptive measurement, with its uncertainty shown.
- No features that pay or reward users for posting.
- Keep only the author data the metrics need, and don't publish profiles of individuals beyond what the feature needs. The KOL ledger covers public, influential accounts only.
