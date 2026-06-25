# Project Overview — BuzzDaddy (ReplyGuy Clone)

## What this project is

BuzzDaddy is a SaaS tool that automates organic-looking promotion ("UGC marketing") on social media. A user describes their product/startup, sets a list of keywords, and the app:

1. Scrapes recent posts on **X (Twitter)**, **LinkedIn**, and **Reddit** that match those keywords.
2. Uses an **LLM (OpenAI via the Vercel AI SDK)** to filter the scraped posts down to the ones actually relevant to the campaign.
3. Generates a casual, human-sounding comment/reply for each relevant post (using a configurable voice/tone/personality), seamlessly plugging the user's product.
4. Posts that reply back to the original platform via each platform's API.
5. Tracks everything (posts, comments, status, activity) in Postgres, and can run **fully autonomously** via daily cron jobs ("AutoPilot").

It is built on top of the open-source **"next-saas-stripe-starter"** template (Next.js + Prisma + Stripe + Auth.js boilerplate), with the social-scraping/AI/posting logic layered on top.

## Tech stack

- **Framework**: Next.js 14 (App Router), TypeScript, React 18, Tailwind CSS, shadcn/ui (Radix UI primitives), Framer Motion.
- **Auth**: Auth.js / NextAuth v4 — Google OAuth + magic-link email sign-in, JWT sessions, Prisma adapter.
- **Database**: PostgreSQL via Prisma ORM (`relationMode = "prisma"`, i.e. no native FK constraints — relations enforced at the app layer, which is what PlanetScale-style serverless Postgres needs).
- **AI**: Vercel `ai` SDK (`generateObject`) + `@ai-sdk/openai`, calling OpenAI models (`gpt-3.5-turbo`, `gpt-4o-mini`, `gpt-4o-2024-08-06`) with Zod schemas for structured output.
- **Billing**: Stripe (subscriptions, webhooks, customer portal).
- **Email**: Resend + React Email templates (magic link, welcome, weekly summary, autopilot-on notifications).
- **Scraping/Social integrations**:
  - **Apify** (`apify-client` + direct REST calls) — runs hosted scraper "Actors" for Twitter (`microworlds/twitter-scraper`, `apidojo/tweet-scraper`) and LinkedIn (`curious_coder/linkedin-post-search-scraper`, using saved LinkedIn session cookies).
  - **Twitter API v2** (`twitter-api-v2`, `twitter-api-sdk`) — direct search + posting replies.
  - **Reddit** via `snoowrap` and direct OAuth REST calls — search + posting comments, with token refresh logic.
  - **LinkedIn** via a custom `lib/linkedin-api` wrapper using LinkedIn's REST API + stored access tokens.
- **State/data fetching**: TanStack Query + TanStack Table, Zustand for client state.
- **Tooling**: ESLint, Prettier, Husky + commitlint (conventional commits), pnpm.

## Data model (Prisma — `prisma/schema.prisma`)

- `User` — core identity; linked to OAuth `Account`/`Session` (NextAuth), Stripe customer/subscription fields, and all app data.
- `Campaign` — one promotion campaign per product: name, description, businessUrl, category, plus AI persona fields (`voice`, `tone`, `personality`) and an `autoPilot` boolean toggle.
- `Keyword` — search terms tied to a campaign, used to query each platform.
- `PostPreference` — per-campaign settings: which platforms are enabled (Twitter/LinkedIn/Reddit), posting frequency.
- `Post` — a scraped social post matched to a campaign; has a `platform` enum and a `content` JSON blob, plus optional 1:1 detail tables `TwitterPost`/`LinkedInPost`/`RedditPost` for platform-specific fields.
- `Comment` — the AI-generated reply tied to a `Post`, with `status` (`PENDING`/`POSTED`/`FAILED`), the external comment ID once posted, and any error message.
- `Notification` / `ActivityLog` — user-facing activity feed (campaign created, autopilot on/off, replies sent, etc).
- Platform credentials: `TwitterToken`, `TwitterProfile`, `LinkedInToken`, `LinkedInUser`, `OAuthState`, `LinkedInState` — store OAuth tokens/state per user for posting on their behalf.

## How the pipeline actually works

### Manual flow (Playground)
The dashboard's "Playground"/"Explorer" pages let a user manually pick a platform, run a search for a campaign's keywords, review scraped posts, and trigger AI comment generation/posting on demand (`app/(project)/project/playground`, `app/(dashboard)/playground`, `app/actions/*`).

### AutoPilot flow (the real product)
Driven by two Vercel **cron jobs** defined in [vercel.json](vercel.json), both running daily at 06:00 UTC, protected by a `CRON_SECRET` bearer-token check:

1. **`/api/cron/campaign/post`** ([route.ts](app/api/cron/campaign/post/route.ts))
   - Finds all campaigns with `autoPilot = true`.
   - For each enabled platform, hits the corresponding internal API (`/api/x`, `/api/reddit`, `/api/linkedin`) to fetch fresh posts matching the campaign's keywords (last 10 days).
   - Batches results (50 at a time) through `filterRelevantPostsInBatches` (`app/actions/ai.ts`), which asks GPT-4o to keep only posts genuinely relevant to the campaign.
   - Saves the surviving posts as `Post` rows via `createAutoPilotPosts`.

2. **`/api/cron/campaign/comment`** ([route.ts](app/api/cron/campaign/comment/route.ts))
   - Finds autopilot campaigns that have posts with `PENDING` comments.
   - For each, calls `generatePostComment` (`app/actions/ai.ts`) — GPT-4o-mini generates an in-character, slang-tolerant, non-ad-sounding reply using the campaign's `voice`/`tone`/`personality`.
   - Posts the generated reply to the originating platform's API route (`/api/x`, `/api/reddit`, `/api/linkedin` — each implements its own `POST` handler to actually publish the comment).
   - Updates `Comment.status` to `POSTED` or `FAILED` (with the error message) and updates the parent `Post.status` accordingly.

### Platform integration details
- **`/api/x`**: `GET` searches Twitter API v2 recent search by query terms; `POST` replies to a tweet using a configured `twitter-api-v2` client (`lib/twitter.ts`).
- **`/api/reddit`**: `GET` searches Reddit's OAuth search API per keyword (with token auto-refresh); `POST` posts a comment via `snoowrap`, also with refresh-and-retry on 401/403.
- **`/api/linkedin`**: `GET` runs an Apify LinkedIn-scraper actor with stored session cookies; `POST` posts a comment via the custom `lib/linkedin-api` client and stored `LINKEDIN_ACCESS_TOKEN`.
- **`app/actions/apify.ts`**: thin validated wrappers around two Apify Twitter-scraper actors (used as an alternative/supplement to the native Twitter API).

### Billing
Stripe customer/subscription creation (`actions/generate-user-stripe.ts`), webhook handling (`app/api/webhooks/stripe/route.ts`), and plan config (`config/subscriptions.ts`) gate access to paid tiers (Pro/Business, monthly/yearly).

## Environment variables (`.env.example`) — what each is for

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_APP_URL` | Base URL the app uses to call its own internal API routes (e.g. cron jobs calling `/api/x`) and for absolute links. |
| `DATABASE_URL` | Postgres connection string for Prisma. |
| `NEXTAUTH_URL`, `NEXTAUTH_SECRET` | Auth.js session config — secret signs/encrypts JWTs and cookies. |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Google OAuth login provider. |
| `GITHUB_OAUTH_TOKEN` | Used by the docs/blog tooling (template leftover from the starter, not core product logic). |
| `RESEND_API_KEY` | Sends transactional email (magic-link sign-in, welcome email, weekly summary) via Resend + React Email. |
| `STRIPE_API_KEY`, `STRIPE_WEBHOOK_SECRET` | Server-side Stripe SDK auth and webhook signature verification. |
| `NEXT_PUBLIC_STRIPE_*_PLAN_ID` (Pro/Business, monthly/yearly) | Stripe Price IDs shown on the pricing page and used to start checkout for each plan. |
| `OPENAI_API_KEY` | Powers all AI generation — keyword generation, post relevance filtering, and comment/reply generation (via Vercel AI SDK + OpenAI). |
| `APIFY_TOKEN`, `APIFY_API_TOKEN` | Auth for Apify's hosted scraper actors, used for Twitter and LinkedIn post scraping. |
| `TWITTER_CLIENT_ID`, `TWITTER_APP_KEY`, `TWITTER_APP_SECRET`, `TWITTER_ACCESS_TOKEN`, `TWITTER_ACCESS_TOKEN_SECRET`, `TWITTER_BEARER_TOKEN` | Twitter/X API v2 credentials — bearer token for searching tweets, app key/secret + access tokens for posting replies on behalf of the connected account. |
| `LINKEDIN_ACCESS_TOKEN` | Used by the custom LinkedIn API client to post comments on LinkedIn posts. |
| `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`, `REDDIT_USERNAME`, `REDDIT_PASSWORD` | Reddit "script" app credentials used by `snoowrap` to authenticate and post comments. |
| `REDDIT_STATE`, `REDDIT_REDIRECT_URI`, `REDDIT_AUTHORIZATION_CODE` | Reddit OAuth authorization-code flow values (used during initial token setup). |
| `REDDIT_ACCESS_TOKEN`, `REDDIT_REFRESH_TOKEN` | Live Reddit OAuth tokens; access token is refreshed automatically using the refresh token when it expires or a request returns 401/403. |
| `SERP_API_KEY` | Search API key (declared, used for keyword/SERP-style search support — e.g. `app/api/search`). |
| `CRON_SECRET` | Shared secret that Vercel Cron must send as a Bearer token to authorize hitting the `/api/cron/*` endpoints — prevents public abuse of these expensive, write-heavy routes. |
| `ENCRYPTION_KEY` | 32-byte hex key used by `lib/encryption.ts` (AES-256-CBC) to encrypt/decrypt sensitive stored values (e.g. platform tokens) at rest. |

Note: `env.mjs` (the type-safe env loader via `@t3-oss/env-nextjs`) currently only validates a subset of these (`NEXTAUTH_*`, `GOOGLE_*`, `GITHUB_OAUTH_TOKEN`, `DATABASE_URL`, `RESEND_API_KEY`, `STRIPE_*`, `NEXT_PUBLIC_APP_URL`) — the AI/scraping/social-platform keys are read directly from `process.env` in the relevant action/route files rather than going through that schema.

## Notable implementation characteristics

- **Server Actions everywhere**: almost all business logic (`app/actions/*.ts`) is written as `"use server"` functions returning a consistent `{ type: "success" | "error", message, data }` shape, called directly from client components or from API routes — not a separate REST layer.
- **Route-group structure**: `app/(marketing)` (public landing/pricing), `app/(auth)` (login/register), `app/(dashboard)` (legacy/simple dashboard), `app/(project)` (the main per-campaign workspace — explorer, playground, keywords, preferences, activity, settings).
- **Self-calling APIs**: the cron routes don't talk to external platforms directly for *fetching* — they call the app's own `/api/x`, `/api/reddit`, `/api/linkedin` routes over HTTP, which is unusual but keeps platform-specific logic centralized in one place reusable by both manual UI actions and cron.
- **This is pre-production / prototype-quality code** in places: several `console.log` debug statements left in (including ones that print Reddit credentials to logs in `app/api/reddit/route.ts`), a hardcoded placeholder LinkedIn comment ("Great post! Thanks for sharing.") and `userUrn` in `app/api/linkedin/route.ts` rather than using the AI-generated content, and commented-out alternate implementations left in source.
