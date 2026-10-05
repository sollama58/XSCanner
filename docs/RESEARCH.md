# XSCanner — Research Notes (Oct 2026)

Supporting evidence for [`GUIDELINE.md`](GUIDELINE.md).

**How this was compiled:** web search on 2026-10-05. Many primary documentation sites (docs.x.com, DexScreener, PumpPortal, Birdeye, Helius, arXiv) could not be fetched directly, so a lot of the figures come from search-result summaries and secondary blogs. Some of those blogs come from vendors that compete with the official X API. Items marked **[verify]** are uncertain or conflicting, and should be checked against primary docs before you depend on them.

---

## 1. Getting data from X

### Official X API
- **Pricing timeline:**
  - **Oct 2025:** pay-per-use pilot.
  - **6 Feb 2026:** pay-per-use becomes the default.
    - The Free tier closed to new signups.
    - Basic ($200/mo) and Pro ($5k/mo) closed to new buyers.
    - Legacy plans were force-migrated during Jun–Sep 2026.
  - **Enterprise:** starts around $42k/mo (blog figure).
  - Sources: [launch post](https://devcommunity.x.com/t/announcing-the-launch-of-x-api-pay-per-use-pricing/256476), [TechCrunch](https://techcrunch.com/2025/10/21/x-is-testing-a-pay-per-use-pricing-model-for-its-api), [docs pricing](https://docs.x.com/x-api/getting-started/pricing)
- **Pay-per-use rates:**
  - $0.005 per post read ($5 per 1k).
  - $0.010 per user read.
  - The same resource read again in the same UTC day is billed once.
  - Failed requests are free.
  - Monthly cap of about **2M** (docs snippet) or **3M** (2026 blogs) post reads **[verify]**.
  - Spend can earn up to 20% back in xAI credits.
- **Counts endpoints:**
  - `/2/tweets/counts/recent` and `/counts/all` are available on pay-per-use.
  - Reportedly about $0.005–0.010 **per request**, not per post **[verify]**.
  - If confirmed, this is a cheap way to get a mention-volume time series. [Counts docs](https://docs.x.com/x-api/posts/counts/introduction)
- **Filtered stream on pay-per-use:** 1 connection, 1,000 rules, 1,024 characters per rule. Each delivered post is probably billed as a read **[verify]**. [Docs](https://docs.x.com/x-api/posts/filtered-stream/introduction)
- **Rate limits:** recent search allows 450 requests per 15 min per app. [Docs](https://docs.x.com/x-api/fundamentals/rate-limits)
- **Operators:**
  - A 2026 changelog moved search to a new index and opened up several operators to all tiers: `$` cashtag, `has:cashtags`, `bio:`, `bio_name:`, geo operators, and new `min_likes:` / `min_replies:` / `min_reposts:` **[verify]**.
  - Before that, `$` was limited to Enterprise/Academic.
  - Sources: [changelog](https://docs.x.com/changelog), [operators](https://docs.x.com/x-api/posts/search/integrate/operators)

### Third-party providers (unofficial; scraping-backed)

| Provider | Price per 1k posts | Real-time | Notes |
|---|---|---|---|
| [TwitterAPI.io](https://twitterapi.io/) | about $0.15 | Rule-based webhook/websocket | No monthly fee; vendor claims about 500ms latency |
| [SocialData.tools](https://docs.socialdata.tools/getting-started/pricing/) | about $0.20 | Search/user monitors via webhook | Failed requests are free |
| [Apify actors](https://apify.com/apidojo/tweet-scraper) | about $0.14–0.40 | Batch only | Break when X changes its frontend |
| [Sorsa (ex-TweetScout)](https://sorsa.io/) | $49 per 10k requests | — | Account authority scores, based on who follows the account |
| [Xpoz](https://www.xpoz.ai/) | very cheap tiers | — | **[verify]** claims look implausible |

### X platform context
- **Smart Cashtags** (Apr 2026, iOS US/CA): pasting a Solana contract address shows a live chart (Jupiter prices) and a feed of posts about the token. There is no API for it. Expect more posts to carry raw contract addresses.
  - Sources: [bitcoin.com](https://news.bitcoin.com/x-launches-interactive-cashtags-with-real-time-stock-and-crypto-data-for-us-and-canada-iphone-users/), [99Bitcoins via TradingView](https://www.tradingview.com/news/99Bitcoins:3d2d285ac094b:0-x-money-adds-live-crypto-cashtags-how-this-changes-coin-discovery-for-retail/)
- **InfoFi ban (15 Jan 2026):** X revoked API access for apps that reward users for posting. Kaito sunset Yaps and Cookie ended Snaps.
  - Sources: [CoinDesk](https://www.coindesk.com/business/2026/01/15/kaito-to-sunset-yaps-as-x-cracks-down-on-infofi-apps-token-falls-17), [Bankless](https://www.bankless.com/read/news/twitter-x-bans-reward-apps-to-cut-spam-killing-infofi)
- **Grok API X Search:** about $5 per 1k posts plus tokens. [Docs](https://docs.x.ai/developers/tools/x-search)

### Legal / ToS
- Since Nov 2024, X's ToS ban scraping and set **liquidated damages of $15k per 1M posts accessed in 24h**. [CNBC](https://www.cnbc.com/2024/11/22/why-x-new-terms-of-service-driving-some-users-to-leave-elon-musk-platform.html)
- **X v. Bright Data:** X's contract claims were dismissed in 2024. The case was then stayed in 2025 after a settlement, so it set no final precedent. [Goldman blog](https://blog.ericgoldman.org/archives/2025/01/catching-up-on-the-heavyweight-scraping-battle-between-x-and-bright-data-guest-blog-post.htm)
- **Nitter / XCancel** went offline after a cease-and-desist on 24 Aug 2026. [The Register](https://www.theregister.com/legal/2026/08/26/nitter-no-more-x-sends-in-the-lawyers-to-shut-down-open-source-project/5292548)
- **Takeaway:** the official API is the only clean route. Third-party vendors take on the scraping risk, but their data supply can disappear without warning.

---

## 2. Pump.fun and the Solana launchpad landscape

### Pump.fun mechanics
- **Mint addresses:** the `pump` suffix is a vanity grind, not a protocol rule. Identify the launchpad by **program ID and config keys** instead.
  - [j.tools](https://j.tools/en/blog/pump-suffix-solana-token-address); suffix matching gave about 72k false positives in [opentrench PR #19](https://github.com/frogeth/opentrench/pull/19).
- **Bonding curve:** 1B supply, about 800M sold on the curve, virtual reserves of 30 SOL and about 1.073B tokens. Graduation happens at about **85 SOL** raised.
  - The graduation rate is typically **below 2%** (1.15% weekly in Sep 2026).
  - Graduates go to **PumpSwap** (since Mar 2025); older ones went to Raydium.
  - Sources: [curve math](https://github.com/nirholas/pump-fun-sdk/blob/main/docs/bonding-curve-math.md), [Cryptopolitan](https://www.cryptopolitan.com/pump-fun-graduating-tokens-break-to-1-15-of-new-launches/), [blocmates](https://www.blocmates.com/news-posts/pump-fun-introduces-pumpswap-a-new-dex-for-graduated-token-listings)
- **Program changes since Nov 2025:**
  - `create_v2` uses Token-2022.
  - **Mayhem Mode:** an AI agent trades the coin randomly for its first 24h.
  - `buy_v2` / `sell_v2` instructions.
  - Reserve fields were renamed to `*_quote_reserves`, and can be negative since a 30 Sep update.
  - Sources: [pump-public-docs](https://github.com/pump-fun/pump-public-docs), [Chainstack](https://chainstack.com/trading-bot-update-full-mayhem-mode-support-for-pump-fun/)
- **Custom Pairs (Sep 2026):** about 93 quote assets, including tokenized stocks and metals. Don't assume SOL is the quote asset. [FXStreet](https://www.fxstreet.com/cryptocurrencies/news/pumpfun-launches-custom-pairs-for-tokenized-stocks-and-real-world-assets-on-solana-202609100558)
- **Fee modes over time** (record the mode for each coin):
  - Project Ascend dynamic creator fees (Sep 2025).
  - Fee sharing across up to 10 wallets (Jan 2026).
  - Cashback (Feb 2026).
  - **Holder Rewards** (12 Sep 2026), which replaced cashback on standard launches.
  - Sources: [CMC](https://coinmarketcap.com/academy/article/pump-token-project-ascend-launches-10x-creator-rewards), [TradingView](https://www.tradingview.com/news/coinmarketcal:df0f05658094b:0-pump-fun-holder-rewards-launch-replaces-cashback-mode-12-sep-2026/)
- **Livestreams:** they returned in 2025 after a 2024 shutdown. The `is_currently_live` field is exposed. [crypto.news](https://crypto.news/pump-fun-reopens-livestreams-to-5-of-users-after-moderation-overhaul/)
- **Acquisitions:** Pump.fun bought Kolscan (KOL wallet PnL tracker) in Jul 2025 and the Padre terminal in Oct 2025. [CoinDesk](https://www.coindesk.com/markets/2025/07/11/pumpfun-acquires-wallet-tracker-kolscan-to-expand-onchain-trading-tools)

### Pump.fun data access
- **Unofficial frontend API:** `https://frontend-api-v3.pump.fun`. The `/coins/{mint}` endpoint returns `twitter`, `website`, `telegram`, `creator`, `complete`, `usd_market_cap` and `is_currently_live`.
  - Many endpoints want a JWT and an `Origin` header.
  - There is no stability guarantee.
  - Endpoint catalogue: [BankkRoll/pumpfun-apis](https://github.com/BankkRoll/pumpfun-apis)
- **Token metadata:** the IPFS JSON has `name`, `symbol`, `description`, `image`, `twitter`, `telegram` and `website`.
  - The `twitter` field is creator-supplied free text, so impersonation is common.
  - It may hold a profile URL or a tweet URL.
- **PumpPortal websocket** (`wss://pumpportal.fun/api/data`), covering Pump.fun and PumpSwap:
  - `subscribeNewToken` and `subscribeMigration` are free.
  - Token and account trade streams cost about 0.01 SOL per 10k events **[verify]**.
  - Source: [pumpportal.fun](https://pumpportal.fun/)
- **Official SDKs:** `@pump-fun/pump-sdk` (TS) and `pump-rust-client`.

### Other launchpads (for later expansion)
- **Bonk.fun / LetsBonk:** runs on Raydium LaunchLab. It briefly overtook Pump.fun in mid-2025. Pump.fun has since regained about 70–90% of launches.
  - Sources: [Meme Insider](https://meme-insider.com/en/article/launchlabs-metrics-bonk-fun-raydium-surge/), [CMC](https://coinmarketcap.com/academy/article/pumpfun-reclaims-90percent-market-share-in-solana-launchpad-war)
- **Believe:** launches a coin from a tweet that tags @launchcoin, on Meteora DBC. Whether this still works in 2026 is unknown **[verify]**. It is the most X-native launch path. [CoinGecko](https://www.coingecko.com/learn/what-is-believe-token-launchpad)
- **Bags.fm:** fee sharing is tied to **social handles**, including X. It has a public API, which makes it a cheap source of token ↔ X-account links. [Docs](https://docs.bags.fm/changelog/changelog)
- **Heaven, Moonshot, Boop, daos.fun, trends.fun:** smaller or uncertain status.
  - Meteora DBC brands can be told apart by config key (see opentrench PR #19).
  - Meteora DBC program: `dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN`.

### Market and risk data

| Source | Free tier | Use |
|---|---|---|
| DexScreener `api.dexscreener.com` | No key; about 300 req/min for pairs/tokens, 60 req/min for profiles/boosts/orders | Price, liquidity, socials, **paid boosts/profiles** |
| GeckoTerminal | About 10–30 calls/min | Fallback prices / OHLCV |
| Jupiter Price API v3 | 60 req/min | Prices |
| Helius | 1M credits/mo; Developer plan $49 | RPC, DAS, enhanced websockets / webhooks |
| Birdeye | Small free tier; Starter $99 | Rich token analytics |
| Bitquery | Trial only | GraphQL streams for Pump.fun, PumpSwap, Bonk, Bags, DBC |
| RugCheck `api.rugcheck.xyz/v1/tokens/{mint}/report` | Free | Top-holder concentration, insider flags, risk score |
| Kolscan / GMGN / Solana Tracker | Varies | KOL wallet labels and PnL |

Sources: [DexScreener API reference (mirror)](https://github.com/openSVM/dexscreener-mcp-server/blob/main/docs/api-reference.md), [Birdeye pricing](https://birdeye.so/data-api/pricing), [Bitquery Pump.fun docs](https://docs.bitquery.io/docs/blockchain/Solana/Pumpfun/Pump-Fun-API/), [GMGN skills](https://github.com/GMGNAI/gmgn-skills)

---

## 3. Research on attention and manipulation

### Does attention predict returns?
- **Liu & Tsyvinski (RFS):** a one-standard-deviation rise in Twitter post count goes with about +2.5% BTC return one week later. Attention measures are among the main crypto-specific predictors. [NBER w24877](https://www.nber.org/system/files/working_papers/w24877/w24877.pdf)
- **Shen, Urquhart & Wang (2019):** tweet count Granger-causes **trading volume and realized volatility**. The effect on returns is weaker. [PDF](https://centaur.reading.ac.uk/80420/1/Twitter.Bitcoin.pdf)
- **Systematic reviews:** lagged tweet volume generally beats sentiment polarity. [MDPI review](https://www.mdpi.com/2227-7072/13/2/87)
- **Ardia & Bluteau (2024), >1,100 pump events:** X posts spike in the minutes before manipulated moves, and Twitter-reliant investors sell late. [GERAD](https://www.gerad.ca/en/papers/G-2023-69), [event data](https://github.com/ArdiaD/PumpDump)

### Pump-and-dumps and memecoins
- **Xu & Livshits (USENIX Sec 2019):** you can predict the target coin of a Telegram pump before it happens. [PDF](https://www.usenix.org/system/files/sec19-xu-jiahua_0.pdf)
- **La Morgia et al. (2020/ACM TOIT):** real-time detection of pumps within seconds, using rolling-window trade features. [PDF](https://massimolamorgia.com/assets/pdf/Pump_Dump__ICCCN__2020.pdf)
- **Mirtaheri et al. (2019):** the bot share of crypto tweets rises during pumps. [arXiv](https://arxiv.org/pdf/1902.03110)
- **Nizzoli et al. (2020):**
  - More than 56% of accounts sharing Telegram/Discord invites were bots or later suspended.
  - 93% of bot-shared invites led to pump channels.
  - [Semantic Scholar](https://www.semanticscholar.org/paper/313affc91ad6a4ba2b0d1edf2c205bade6168ac3)
- **"Meme Coin Factories" (2026), about 15M Pump.fun coins:**
  - **About 23.5% of coins were created after an X or Truth Social post.**
  - It identifies five classes of manipulation, including social-media manipulation and copycat coins.
  - [arXiv 2609.10246](https://arxiv.org/abs/2609.10246) **[verify]**
- **"A Midsummer Meme's Dream" (USENIX Sec 2026):** **82.89%** of memecoins with >100% returns showed artificial growth. [arXiv 2507.01963](https://arxiv.org/abs/2507.01963)
- **"Catching the Rug" (2026):** the first 5 minutes of trading predict whether a token rugs within 1 hour. [arXiv 2608.20271](https://arxiv.org/abs/2608.20271)
- **"The Memecoin Phenomenon: Solana":** Pump.fun accounted for up to 71% of Solana mints in Q4 2024. [arXiv 2512.11850](https://arxiv.org/abs/2512.11850)
- **Gaps:** no published estimate of how fast memecoin attention decays (half-life), and no study linking X attention to Pump.fun graduation.

### Bots and coordination
- **Botometer:** the free API ended in 2023, and Botometer X covers accounts only up to May 2023. It is effectively unusable for memecoin shill accounts. [Botometer](https://x.com/Botometer/status/1655414250863566848)
- **fox8 botnet (Yang & Menczer 2024):** 1,140 ChatGPT-driven crypto bots, found through the phrase "as an AI language model". [arXiv](https://arxiv.org/pdf/2307.16336), [code](https://github.com/osome-iu/AIBot_fox8)
- **Pacheco et al. (ICWSM 2021):** a general coordination-detection method using account × trace TF-IDF and cosine similarity, applied to co-retweets, hashtag sequences, images and timing. [arXiv](https://arxiv.org/html/2001.05658v2)
- **Telegram raid bots** (RaidSharks, D.RaidBot) organize like/reply quotas on a target post, increasingly with AI-written replies. [RaidSharks docs](https://docs.raidsharksbot.com/)
- **Paid KOLs:** $1.5k–$60k per post. **Fewer than 5 of about 160** paid KOLs disclosed the posts as ads. [CoinDesk](https://www.coindesk.com/business/2024/05/09/inside-cryptos-kol-economy-influencer-investors-get-special-treatment-in-token-deals)

---

## 4. Existing products (how they measure attention)

| Product | Metric | How it works | Lesson / gap |
|---|---|---|---|
| LunarCrush | Galaxy Score, AltRank, Social Dominance | Blends price, social and sentiment scores. AltRank is a *relative* rank across all coins | Ranking against the cross-section is good. Weights are opaque. Built for listed coins and tickers |
| Santiment | Social Volume, Social Dominance, Weighted Sentiment | Counts documents (not mentions). Dominance is a share of the top-100. Weighted sentiment is roughly a z-score of volume × sentiment | Extreme dominance works as a **contrarian** signal |
| Kaito | Mindshare (Yaps discontinued) | Posts weighted by reach and LLM-judged quality. Engagement from high-reputation accounts counts more. Originality checked with embeddings | Rewarding posting got gamed and then banned by X |
| Sorsa / TweetScout | Account score 0–1000 | Based only on *who follows* the account (KOLs, VCs, projects) | A cheap prior that resists fake accounts. Static |
| Cookie3 | Mindshare, smart engagement | Engagement from a curated set of "smart" accounts | Also hit by the InfoFi ban |
| Moni | Moni Score | Smart followers, smart mentions and momentum | Account-risk flags |
| GMGN / Axiom / Photon | X Tracker | Live feed of about 2000+ KOLs' posts and **bio changes**, alongside wallet labels | A feed, not a measurement. Linking posts to wallets is the clever part |

Sources: [LunarCrush support](https://lunarcrush.com/support), [Santiment academy](https://academy.santiment.net/metrics/social-dominance/), [Kaito overview](https://transak.com/blog/what-is-kaito-ai), [TweetScout](https://tweetscout.io/), [Moni](https://getmoni.io/discover), [GMGN X Tracker](https://docs.gmgn.ai/index/x-tracker)

---

## 5. Statistical toolbox

Notation: `c_t` is the count in bin `t`. Prefer counting **effective voices** over raw posts.

- **Robust z-score:** `(c_t − median) / (1.4826·MAD)`. Unreliable at low counts.
- **Poisson surprise:** `p = 1 − F_Pois(c_t − 1; λ̂)`, with λ̂ from an EWMA baseline.
  - Use a negative binomial if counts are overdispersed.
  - Or apply the Anscombe transform `2√(c+3/8)` before z-scoring.
- **EWMA:**
  - `μ_t = αc_t + (1−α)μ_{t−1}`, with `α = 1 − e^(−Δt/τ)`; half-life is `τ·ln2`.
  - The ratio of a fast to a slow EWMA gives velocity.
  - CUSUM detects change points.
- **Kleinberg bursts:** a hidden-state model over gaps between posts, with an increasing-state cost of `γ·ln n`, decoded with Viterbi. Gives discrete burst levels. [Implementation](https://github.com/nmarinsek/burst_detection)
- **Hawkes process:**
  - `λ(t) = μ + Σ m_i φ(t − t_i)`, with marks `m_i ∝ followers^γ`.
  - Branching ratio `n* = E[m]∫φ`; `n* → 1` means viral.
  - μ captures attention coming from outside (exogenous); the sum captures attention generated by reactions (endogenous).
  - Papers: [SEISMIC](https://arxiv.org/pdf/1506.02594), [HIP](https://arxiv.org/pdf/1602.06033), [Evently](https://arxiv.org/pdf/2006.06167)
- **Time-decay ranking:**
  - Hacker News: `(P−1)/(T+2)^1.8`.
  - Reddit hot: `log10(votes) + t/45000`.
  - Exponential: `S ← S·e^(−λΔt) + w`.
- **Normalization:**
  - Percentile against the live universe (as AltRank does).
  - Percentile against same-age cohorts, for brand-new mints.
  - Share-of-voice (dominance).
