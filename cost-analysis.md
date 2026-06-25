# Cost Analysis — Running BuzzDaddy Yourself

This estimates what it actually costs to run this project in production, based on its real architecture (daily cron → scrape 3 platforms → AI filters posts → AI writes replies → posts back to each platform). Pricing below is current as of mid-2026 (sources linked per section). Treat the dollar figures as directional, not a quote — actual cost depends heavily on how many campaigns/keywords you run and how aggressively autopilot replies.

**The short version: the dollar cost is not the main blocker. The Reddit and LinkedIn pieces of this app are not really legally/commercially viable to run at any real volume without enterprise contracts or accepting a high ban risk. Twitter/X reply-posting is also far more expensive per-action than it looks, because this app's replies almost always contain a link. Sections 6 and 7 below cover what changes if you swap OpenAI → Gemini and Apify → Bright Data.**

---

## 1. Fixed monthly infrastructure costs

| Service | Why we need it (justification) | Cost |
|---|---|---|
| **Vercel Pro** | `vercel.json` sets `maxDuration: 300` for every function. The cron routes do sequential multi-platform scraping + batched AI calls per campaign, which routinely runs past Vercel Hobby's duration cap. Without Pro, the daily autopilot cron jobs would simply time out mid-run. | **$20/month** per seat |
| **Postgres (Neon or Supabase)** | Required by Prisma (`DATABASE_URL`) to persist users, campaigns, keywords, scraped posts, and comments — this is the system of record; the app does not function without it. Free tier (Neon 100 CU-hr/0.5GB, Supabase 500MB) is enough to start, but cron jobs writing batches of posts/comments daily will outgrow it. | **$0–25/month** small scale, **$25–69/month** at always-on scale |
| **Resend** (email) | Powers `lib/auth.ts`'s magic-link sign-in email, the welcome email, and the weekly summary — i.e. core auth flow, not optional. | **$0/month** — free 3,000 emails/month covers this easily at small scale |
| **Domain** | Needed for `NEXT_PUBLIC_APP_URL` / `NEXTAUTH_URL` — OAuth providers and the cron jobs' self-calls require a stable public URL. | **~$12–20/year** |
| **Google OAuth** | Login provider configured in `lib/auth.ts`. No usage-based cost from Google for standard sign-in volume. | **$0** |

**Fixed baseline: roughly $20–45/month** before any AI or scraping usage. None of this is avoidable — it's the minimum to have the app online and able to log in/store data at all.

---

## 2. Usage-based costs — current implementation (OpenAI + official APIs + Apify)

### OpenAI — the AI generation layer
([openai.com/api/pricing](https://openai.com/api/pricing/))

| Call | Why we need it | Model | Price | Notes |
|---|---|---|---|---|
| `generateCampaignKeywords` | Turns a campaign's description into search keywords — without this, you'd have to hand-write keywords for every campaign. | gpt-3.5-turbo | ~$0.10–0.40/1M tokens-equivalent | Runs once per campaign creation. Negligible (<$0.01/campaign). |
| `filterRelevantPostsInBatches` | Raw keyword search returns a lot of noise (off-topic posts); this call is what keeps the bot from replying to completely irrelevant posts, which would be an obvious giveaway it's a bot. | **gpt-4o-2024-08-06** ($2.50/1M in, $10/1M out) | **This is the expensive one.** It serializes up to 50 full scraped posts + the full campaign object into the prompt, then asks the model to return the *entire filtered post list back out* as structured JSON. Output tokens roughly mirror input tokens because the model echoes post data back. A batch of 50 posts runs ~$0.08–0.15. | At 200 raw posts/day across platforms for one campaign (~4 batches), that's **~$0.40–0.60/day per campaign**, or **~$12–18/month per active campaign**. |
| `generatePostComment` | This is the actual product — generates the human-sounding promotional reply for each relevant post. | gpt-4o-mini ($0.15/1M in, $0.60/1M out) | Cheap — short prompt + short reply. | <$0.001/comment. Even at 1,000 comments/month, well under $1. |

**Realistic OpenAI cost: ~$15–20/month per active campaign**, dominated almost entirely by the filtering step — which is architecturally wasteful (it pays to have the LLM copy data it was just given, rather than returning indices/IDs to keep).

### Twitter / X API
([X API pricing tiers, pay-per-use since Feb 2026](https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/))

Why we need it: it's the only first-party way to search recent tweets by keyword and to post a reply as a specific authenticated account — without it, there's no way to actually act on Twitter/X.

New developers can no longer buy the old $100/mo Basic tier — it's now **pay-per-use**:
- Reads (search): **$0.005/tweet read**, capped at 2M reads/month
- Writes (posting a reply) **without** a link: $0.015/post
- Writes (posting a reply) **with** a link: **$0.20/post**

This matters a lot here: the entire point of this app's generated comments is to "plug the campaign with a link... woven in seamlessly." That means **most autopilot replies will be billed at the $0.20/post rate, not $0.015.**

- Searching 10 keywords/campaign once a day (~10 tweets returned each by default) ≈ 100 reads/day ≈ $0.50/day ≈ **~$15/month** per campaign just to search.
- Posting 20 replies/day with links ≈ $4/day ≈ **~$120/month** per campaign for replies alone.

**A single actively-posting campaign on Twitter/X realistically costs $50–150+/month**, almost entirely from the per-link-post fee.

### Reddit API
([Reddit Data API commercial pricing](https://octolens.com/blog/reddit-api-pricing))

Why we (theoretically) need it: posting a comment on Reddit requires authenticating as a real account and going through Reddit's API — there's no way to post without it.

This is the biggest non-obvious problem. Reddit's free OAuth tier (what `snoowrap` uses here) is explicitly for **personal/non-commercial use**. Automated, scheduled, promotional commenting on behalf of a paying SaaS product is commercial use by Reddit's own definition. Reddit's actual commercial Data API tier starts at **$12,000/month for up to 50M calls**, and getting any commercial agreement requires applying and going through a use-case review — there's no self-serve checkout.

Practically, this means one of two things:
- Run it on the free personal tier and accept real risk of the Reddit account being **suspended/banned** for ToS violation (this is what most clones of this kind actually do), or
- Budget **$12,000+/month** to do it within Reddit's actual commercial terms.

There is effectively no cheap, compliant middle ground for Reddit in this app's current design.

### LinkedIn
([LinkedIn Marketing Developer Platform access](https://www.blotato.com/blog/linkedin-api-pricing)) + Apify scraping

Why we need each piece: posting requires LinkedIn's own API (`Share on LinkedIn`); finding posts to reply to requires either LinkedIn's search (gated behind partner access) or scraping, which is why this app uses Apify with stored session cookies instead.

- **Posting** replies via your own connected LinkedIn account (`Share on LinkedIn` product) is free and requires no special partnership — this part of the app is fine cost-wise, as long as it's posting as the authenticated user themselves.
- **Scraping search results** is done here via an Apify actor (`curious_coder/linkedin-post-search-scraper`) using saved session cookies, at roughly **$1–8 per 1,000 results** depending on which actor is used.
- The bigger issue: LinkedIn's User Agreement explicitly prohibits scraping, and LinkedIn has a track record of pursuing legal action against scrapers (e.g. the hiQ Labs case). Cookie-based scraping like this risks the connected LinkedIn account being flagged or banned, independent of dollar cost.

### Apify (general — Twitter + LinkedIn scraping)
([Apify pricing](https://apify.com/pricing))

Why we need it: it's the workaround for not having (or not wanting to pay for) official search access — Apify runs hosted scraper "Actors" that do the actual browsing/scraping for you.

- Free plan: $5/month in credits, fine for light testing.
- Pay-per-result actor pricing actually used here: Twitter scrapers run **$0.15–0.40 per 1,000 tweets**, LinkedIn post search **~$1–8 per 1,000 results**.
- At light-to-moderate scraping volume (a few thousand results/month across platforms), this lands around **$5–30/month**. Apify itself is the cheapest piece of this stack.

### SerpApi (optional — `SERP_API_KEY`)
([serpapi.com/pricing](https://serpapi.com/pricing))

Why we'd need it: only relevant if `app/api/search` (general web search, not one of the three social platforms) is actually exercised in your usage of the app.

Free tier: 250 searches/month. First paid tier: **$25/month for 1,000 searches**. Skippable if you don't need general web search on top of the three social platforms.

### Stripe
Why we need it: it's how *you* charge *your own* customers if you're running this as a paid SaaS, not a cost of operating the scraping/AI pipeline itself. 2.9% + $0.30 per transaction — only applies to revenue you actually collect.

---

## 3. Putting it together — realistic monthly totals (current implementation)

| Scenario | Campaigns | What's running | Estimated monthly cost |
|---|---|---|---|
| **Testing / personal, no real posting** | 1 | Hosting + DB + light OpenAI filtering, autopilot off or posting a handful of replies manually, Reddit/LinkedIn on free/risky tier | **~$25–60/month** |
| **Small business, autopilot on, 1 platform (Twitter only, with link-replies)** | 1–2 | Fixed infra + OpenAI filtering + Twitter pay-per-use writes | **~$120–250/month** |
| **Small business, all 3 platforms, autopilot on, moderate volume** | 3–5 | Fixed infra + OpenAI filtering (×N campaigns) + Twitter writes (dominant cost) + Apify + accepted Reddit/LinkedIn ban risk | **~$400–900/month** |
| **"Doing it properly" — Reddit on an actual commercial agreement** | any | Everything above **plus** a real Reddit Data API commercial contract | **+$12,000/month minimum**, making this infeasible outside a funded company |

The realistic number for someone running this as a side project, accepting that Reddit/LinkedIn run on personal-tier credentials at ban risk rather than paying for commercial access, is **roughly $100–900/month**, with **Twitter's per-post-with-link fee ($0.20) as the single largest and most volume-sensitive line item**, followed by GPT-4o filtering cost (which scales per campaign, not per reply).

---

## 4. The non-dollar costs (arguably the real cost)

- **Reddit**: no compliant low-cost path exists. Either pay $12K+/month or run on a personal-use API key in violation of Reddit's terms.
- **LinkedIn**: scraping search results violates LinkedIn's User Agreement; the connected account risks suspension regardless of how little you pay Apify.
- **Twitter/X**: posting automated, link-containing replies at scale is the kind of behavior X's spam/platform-manipulation policies target, independent of whether you're paying for the API — account suspension is a real risk even if you're within budget.
- **All three platforms**: the entire product is built around making AI-generated promotional replies look organic ("must not seem like an ad"). That's the core value proposition, but it's also exactly the behavior most platforms' anti-spam/inauthentic-engagement policies exist to catch.

These aren't hypothetical — they're the dominant practical cost of running this for real, and no amount of infrastructure optimization removes them. **Swapping AI providers or scraping vendors (sections 6–7) does not change anything in this section** — those risks live entirely on the posting/scraping side, not the AI side.

---

## 6. Case 2 — Replacing OpenAI with Gemini (using your own existing key)

([Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing), [Gemini free tier rate limits](https://ai.google.dev/gemini-api/docs/rate-limits))

This swap is mechanically simple: the app already calls the Vercel AI SDK's `generateObject()` (see `app/actions/ai.ts`) with an OpenAI provider; the AI SDK has a Gemini provider (`@ai-sdk/google`) with the same `generateObject` interface, so the three call sites (`generateCampaignKeywords`, `filterRelevantPostsInBatches`, `generatePostComment`) would swap providers/models without changing the surrounding logic.

### What it costs

| Tier | Limits | Cost |
|---|---|---|
| **Free tier (Gemini 2.5 Flash / Flash-Lite)** | Flash: ~10–15 RPM, 250 RPD. Flash-Lite: ~15–30 RPM, 1,000 RPD. (Gemini 2.5 Pro free tier is far more restricted: 5 RPM / 50 RPD, and as of April 2026 Pro models are no longer on the free tier at all.) | **$0** |
| **Paid Gemini 2.5 Flash** | No daily cap, pay per token | $0.30/1M input, $2.50/1M output — **~8x cheaper input, ~4x cheaper output** than GPT-4o |

### Does the free tier actually cover this app's usage?

It depends entirely on call volume, and the filtering step is the one to watch:

- **Keyword generation**: maybe 1–5 calls/month (once per campaign created/edited). Trivially within free tier.
- **Comment generation**: 1 call per relevant post per day. At, say, 20–50 comments/day across all your campaigns, that's well within Flash's 250 RPD or Flash-Lite's 1,000 RPD.
- **Post filtering**: this is the one that can blow the free tier. It's called once per **batch of 50 raw posts**, not per reply. At 200 raw posts/day for one campaign (~4 batches), that's only 4 calls/day — fine. But across 5+ campaigns each pulling a few hundred posts/day from 3 platforms, you can realistically hit 20–40+ filtering calls/day, which is still under Flash's 250 RPD ceiling for a single project, but will bump into the **per-minute** limit (10–15 RPM) if the cron job fires all batches back-to-back rather than spacing them out — the current code's batch loop in the cron route does call them sequentially with no delay, so this needs a small throttle, not a redesign.

**Practical read**: for 1–5 campaigns at moderate volume, Gemini's free tier plausibly covers 100% of this app's AI calls — i.e. **$0/month for the entire AI layer**, down from ~$15–20/month per active campaign on OpenAI. Past roughly 5+ campaigns or aggressive batch sizes, you'd spill into Gemini 2.5 Flash's paid tier, which is still **dramatically cheaper than GPT-4o** (roughly 4–8x cheaper per token) and still much cheaper than staying on OpenAI even after exceeding free quota.

### Caveat carried over from the OpenAI section
The filtering call's design flaw (paying to echo full post objects back as output) still applies on Gemini — it just costs nothing while you're inside the free tier, and costs proportionally less once you're not. Fixing the prompt to return IDs instead of full objects is still worth doing; it's what determines how far the free tier actually stretches.

### Updated total with Gemini swapped in

| Scenario | Current (OpenAI) AI cost | With Gemini (free tier) | With Gemini (past free tier) |
|---|---|---|---|
| 1 campaign, moderate volume | ~$15–20/month | **$0/month** | ~$3–5/month |
| 3–5 campaigns, moderate volume | ~$45–100/month | **$0/month** (likely, watch RPM) | ~$10–25/month |

This removes the second-largest cost driver in the whole stack (after Twitter writes) essentially to zero for small-scale use, since you already hold the key and Google isn't billing you for it.

---

## 7. Case 3 — Replacing Apify (and informal scraping) with Bright Data

([Bright Data Web Scraper API pricing](https://brightdata.com/pricing/web-scraper), [Bright Data Twitter/X Scraper](https://docs.brightdata.com/datasets/scrapers/twitter/introduction))

Why you'd consider this: Bright Data offers purpose-built, maintained scrapers for Twitter/X, LinkedIn, and Reddit specifically (rather than Apify's marketplace of third-party community actors of varying reliability/upkeep), with proxy infrastructure designed to avoid blocks, and a "pay only for successful requests" model.

### Pricing

| Platform | Bright Data pay-as-you-go | Bright Data with monthly commitment | Apify equivalent (current) |
|---|---|---|---|
| Twitter/X | $1.50/1,000 records (or $0.75/1,000 with a promo code, first 3 months) | $499/month Scale plan = 384,000 records included, $1.30/1,000 after | $0.15–0.40/1,000 tweets |
| LinkedIn | $1.50/1,000 records | $0.75–0.98/1,000 records on a committed plan | $1–8/1,000 results |
| Reddit | Bright Data also has a dedicated Reddit scraper (similar ~$1.5/1,000 pay-as-you-go pricing) | — | Not currently used (this app calls Reddit's own API directly via `snoowrap`, not a scraper) |

**Net effect on cost**: for Twitter, Bright Data is actually **more expensive per record than the Apify actors currently wired up** ($1.50/1k vs. $0.15–0.40/1k) — Apify wins here because the specific community actors this app uses are unusually cheap. For LinkedIn, Bright Data is roughly **comparable to, or cheaper than, the higher end of Apify's range**, and likely more reliable since it doesn't depend on a third party maintaining session-cookie compatibility.

### What it actually changes (the real reason to consider it)

The dollar swing is small; the operational difference is bigger:
- **No session cookies to maintain** for LinkedIn — Apify's `curious_coder` actor depends on you feeding it a logged-in LinkedIn session cookie, which expires and needs refreshing. Bright Data's dedicated scrapers handle target-site auth/anti-bot themselves.
- **Better reliability/uptime** — Bright Data publishes a 99.99% uptime SLA; community Apify actors can break silently when the target site changes its markup, with no SLA.
- **Does not remove the ToS risk.** This is the important part: Bright Data is still *scraping* LinkedIn and Twitter on your behalf — it doesn't grant you any official permission from those platforms, and LinkedIn's anti-scraping legal stance applies regardless of which vendor's infrastructure does the scraping. Switching scraping vendors changes reliability and minor cost, not the legal exposure already covered in Section 4.
- **Doesn't touch Reddit's core problem at all** — this app posts to Reddit via the official API (`snoowrap`), not a scraper, so Bright Data is irrelevant to the Reddit commercial-tier issue.

### Updated total with Bright Data swapped in for scraping

| Cost line | Current (Apify) | With Bright Data |
|---|---|---|
| Twitter scraping | ~$0.15–0.40/1k tweets (cheaper) | ~$0.75–1.50/1k records (more expensive, but no cookie maintenance) |
| LinkedIn scraping | ~$1–8/1k results | ~$0.75–1.50/1k records (likely cheaper at the high end, more reliable) |

**Net**: roughly a wash to slightly more expensive on raw dollars for a small operation, traded for materially less scraper-maintenance overhead. Not a cost-cutting move by itself — it's a reliability upgrade that costs about the same or a bit more.

---

## 8. Combined scenario — Gemini (free) + Bright Data + current Twitter/Reddit setup

| Scenario | Monthly cost |
|---|---|
| Fixed infra (Section 1) | ~$20–45 |
| AI (Gemini, free tier) | **$0** |
| Scraping (Bright Data, light volume) | ~$5–40 |
| Twitter/X writes (unchanged — link-post fee still applies) | ~$50–150+ per actively-posting campaign |
| Reddit (unchanged — still either free/risky or $12K/month commercial) | $0 (risky) or $12,000+ |
| **Total, realistic small-scale run** | **~$75–235/month**, down from ~$100–900/month, almost entirely because the AI layer drops to $0 |

The biggest lever by far is the Gemini swap — it removes a cost that scaled per-campaign and per-batch. Twitter's per-link-post fee remains the largest *remaining* line item and isn't something either of these swaps touches; reducing it requires either posting fewer/less aggressively, or accepting it as the actual price of running organic-looking promotion at any real volume on X.

---

## Sources

- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [Gemini Developer API Pricing](https://ai.google.dev/gemini-api/docs/pricing)
- [Gemini API Rate Limits](https://ai.google.dev/gemini-api/docs/rate-limits)
- [Apify Pricing](https://apify.com/pricing)
- [Bright Data Web Scraper API Pricing](https://brightdata.com/pricing/web-scraper)
- [Bright Data X (Twitter) Scraper API Docs](https://docs.brightdata.com/datasets/scrapers/twitter/introduction)
- [Twitter/X API Pricing 2026 — pay-per-use breakdown](https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/)
- [Reddit API Pricing 2026](https://octolens.com/blog/reddit-api-pricing)
- [Vercel Cron Jobs — Usage & Pricing](https://vercel.com/docs/cron-jobs/usage-and-pricing)
- [Vercel Pricing](https://vercel.com/pricing)
- [LinkedIn API Pricing Guide 2026](https://www.blotato.com/blog/linkedin-api-pricing)
- [Resend Pricing](https://resend.com/pricing)
- [Neon vs Supabase Pricing 2026](https://www.closefuture.io/blogs/neon-vs-supabase)
- [SerpApi Pricing](https://serpapi.com/pricing)
- [Apify curious_coder LinkedIn Post Search Scraper](https://apify.com/curious_coder/linkedin-post-search-scraper)
- [Apify apidojo Tweet Scraper](https://apify.com/apidojo/tweet-scraper)
