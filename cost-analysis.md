# Cost Analysis — Running BuzzDaddy Yourself

This estimates what it actually costs to run this project in production, based on its real architecture (daily cron → scrape 3 platforms → GPT-4o filters posts → GPT-4o-mini writes replies → posts back to each platform). Pricing below is current as of mid-2026 (sources linked per section). Treat the dollar figures as directional, not a quote — actual cost depends heavily on how many campaigns/keywords you run and how aggressively autopilot replies.

**The short version: the dollar cost is not the main blocker. The Reddit and LinkedIn pieces of this app are not really legally/commercially viable to run at any real volume without enterprise contracts or accepting a high ban risk. Twitter/X reply-posting is also far more expensive per-action than it looks, because this app's replies almost always contain a link.**

---

## 1. Fixed monthly infrastructure costs

| Service | Why it's needed | Cost |
|---|---|---|
| **Vercel Pro** | `vercel.json` sets `maxDuration: 300` for all functions. Vercel's Hobby plan caps function duration well below that, so Pro is effectively required to run the cron jobs and scraping routes without timing out. | **$20/month** per seat |
| **Postgres (Neon or Supabase)** | Prisma's `DATABASE_URL`. Neon's free tier (100 CU-hours, 0.5GB storage) or Supabase's free tier (500MB) can work for a single low-traffic instance, but you'll outgrow it fast once cron jobs are writing batches of posts/comments daily. | **$0–25/month** small scale, **$25–69/month** once you need always-on compute (Supabase Pro is a flat $25/mo; Neon is usage-based, ~$0.106/CU-hr + $0.35/GB-mo) |
| **Resend** (email) | Magic-link login, welcome email, weekly summary. | **$0/month** — free tier covers 3,000 emails/month, which this app won't approach at small scale |
| **Domain** | For `NEXT_PUBLIC_APP_URL` / `NEXTAUTH_URL`. | **~$12–20/year** |
| **Google OAuth** | Login provider. | **$0** |

**Fixed baseline: roughly $20–45/month** before any AI or scraping usage.

---

## 2. Usage-based costs (the part that actually scales with how much you run)

### OpenAI — the AI generation layer
([openai.com/api/pricing](https://openai.com/api/pricing/))

| Call | Model | Price | Notes |
|---|---|---|---|
| `generateCampaignKeywords` | gpt-3.5-turbo | ~$0.10–0.40/1M tokens-equivalent | Runs once per campaign creation. Negligible (<$0.01/campaign). |
| `filterRelevantPostsInBatches` | **gpt-4o-2024-08-06** ($2.50/1M in, $10/1M out) | **This is the expensive one.** It serializes up to 50 full scraped posts + the full campaign object into the prompt, then asks the model to return the *entire filtered post list back out* as structured JSON. Output tokens roughly mirror input tokens because the model echoes post data back. A batch of 50 posts runs ~$0.08–0.15. | At 200 raw posts/day across platforms for one campaign (~4 batches), that's **~$0.40–0.60/day per campaign**, or **~$12–18/month per active campaign**. |
| `generatePostComment` | gpt-4o-mini ($0.15/1M in, $0.60/1M out) | Cheap — short prompt + short reply. | <$0.001/comment. Even at 1,000 comments/month, well under $1. |

**Realistic OpenAI cost: ~$15–20/month per active campaign**, dominated almost entirely by the filtering step — which is architecturally wasteful (it pays to have the LLM copy data it was just given, rather than returning indices/IDs to keep).

### Twitter / X API
([X API pricing tiers, pay-per-use since Feb 2026](https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/))

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

This is the biggest non-obvious problem. Reddit's free OAuth tier (what `snoowrap` uses here) is explicitly for **personal/non-commercial use**. Automated, scheduled, promotional commenting on behalf of a paying SaaS product is commercial use by Reddit's own definition. Reddit's actual commercial Data API tier starts at **$12,000/month for up to 50M calls**, and getting any commercial agreement requires applying and going through a use-case review — there's no self-serve checkout.

Practically, this means one of two things:
- Run it on the free personal tier and accept real risk of the Reddit account being **suspended/banned** for ToS violation (this is what most clones of this kind actually do), or
- Budget **$12,000+/month** to do it within Reddit's actual commercial terms.

There is effectively no cheap, compliant middle ground for Reddit in this app's current design.

### LinkedIn
([LinkedIn Marketing Developer Platform access](https://www.blotato.com/blog/linkedin-api-pricing)) + Apify scraping

- **Posting** replies via your own connected LinkedIn account (`Share on LinkedIn` product) is free and requires no special partnership — this part of the app is fine cost-wise, as long as it's posting as the authenticated user themselves.
- **Scraping search results** is done here via an Apify actor (`curious_coder/linkedin-post-search-scraper`) using saved session cookies, at roughly **$1–8 per 1,000 results** depending on which actor is used.
- The bigger issue: LinkedIn's User Agreement explicitly prohibits scraping, and LinkedIn has a track record of pursuing legal action against scrapers (e.g. the hiQ Labs case). Cookie-based scraping like this risks the connected LinkedIn account being flagged or banned, independent of dollar cost.

### Apify (general — Twitter + LinkedIn scraping)
([Apify pricing](https://apify.com/pricing))

- Free plan: $5/month in credits, fine for light testing.
- Pay-per-result actor pricing actually used here: Twitter scrapers run **$0.15–0.40 per 1,000 tweets**, LinkedIn post search **~$1–8 per 1,000 results**.
- At light-to-moderate scraping volume (a few thousand results/month across platforms), this lands around **$5–30/month**. Apify itself is the cheapest piece of this stack.

### SerpApi (optional — `SERP_API_KEY`)
([serpapi.com/pricing](https://serpapi.com/pricing))

Only relevant if `app/api/search` is actually used. Free tier: 250 searches/month. First paid tier: **$25/month for 1,000 searches**. Skippable if you don't need general web search on top of the three social platforms.

### Stripe
2.9% + $0.30 per transaction — only applies to revenue *you* collect from your own paying users, not a cost of operating the scraping/AI pipeline itself.

---

## 3. Putting it together — realistic monthly totals

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

These aren't hypothetical — they're the dominant practical cost of running this for real, and no amount of infrastructure optimization removes them.

## 5. Where you could cut cost without changing the product

- **Fix the GPT-4o filtering call** to return only post IDs/indices to keep, instead of echoing the full post objects back as output — this alone should cut OpenAI cost by 60–80%, since output tokens are currently paying to regenerate input data.
- **Drop the official Twitter API entirely** and rely only on the Apify Twitter scrapers already integrated (`app/actions/apify.ts`) for both search and avoid the $0.20/link-post write fee — though note Twitter's own automation rules still apply to however you actually post the reply.
- **Cap replies per day per campaign** — most of the variable cost (Twitter writes, OpenAI filtering volume) scales linearly with how many posts/replies autopilot processes per run.
- **Skip SerpApi** unless `app/api/search` is something you actually use.

---

## Sources

- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [Apify Pricing](https://apify.com/pricing)
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
