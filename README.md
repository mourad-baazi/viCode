# CodeScopeAI

> An AI-powered code review tool that plugs into your GitHub repos, catches bugs before they hit production, and actually explains what's wrong — not just that something *is* wrong.

---

## What is this?

CodeScopeAI is a full-stack app I built that automatically reviews pull requests using Claude AI. You connect your GitHub repos, and every time someone opens a PR, the app analyzes the code for security issues, bad patterns, performance problems, and general code smells. Then it posts a detailed review comment directly on the PR.

Think of it like having a senior dev who never sleeps looking at every PR you push.

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=black)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=node.js&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=flat&logo=supabase&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat&logo=express&logoColor=white)

---

## Why I built this

I got tired of waiting hours (or days) for code reviews on my own projects. And when I was learning, I wished there was something that could tell me *why* my code was bad, not just reject it. So I built the tool I wanted to exist.

The goal was pretty straightforward:
- **For students**: get instant feedback on your code without bugging someone
- **For teams**: catch the obvious stuff automatically so reviewers can focus on architecture and design decisions

---

## What it actually does

### The core review pipeline

When a PR lands, here's what happens under the hood:

1. **GitHub webhook fires** → Express server receives the event
2. **File filtering** → skips lock files, build artifacts, node_modules (the stuff nobody reviews anyway)
3. **Smart prioritization** → auth/payment/security files get analyzed first
4. **Claude AI analysis** → sends the diff to Claude with a structured prompt, gets back categorized issues
5. **Security scan** → separate pass looking for SQL injection, XSS, hardcoded secrets, insecure crypto
6. **Supply chain check** → flags suspicious new dependencies (typosquatting, unmaintained packages, low download counts)
7. **Performance analysis** → catches nested loops in hot paths, N+1 queries, memory leaks
8. **Auto-fix generation** → for straightforward issues, generates a unified diff patch and opens a fix PR
9. **Comment posted** → everything gets formatted as a markdown comment on the PR with a risk score

The whole thing runs in ~10-15 seconds for a typical PR.

### Dashboard

The frontend is a full dashboard where users can:
- See all their connected repos and toggle monitoring on/off
- Browse past reviews with detailed breakdowns
- Track analytics (PRs reviewed, issues caught, time saved)
- Manage their subscription plan
- Go through a guided onboarding flow (connect GitHub → pick repos → done)

### Cost controls

Since Claude API calls cost money, I built in some guardrails:
- **1000 line limit** per PR (tells the dev to split it up if it's too big)
- **Token estimation** before calling the API — truncates intelligently if needed
- **Response caching** with SHA256 content hashing — same code = free review
- **Tiered rate limits** — starter gets 10 reviews/day, pro gets 50, enterprise is unlimited
- **Model fallback chain** — tries Claude 4.5 Haiku first (cheapest), falls back to 3.5 Haiku

---

## Tech stack

| Layer | Tech | Why |
|-------|------|-----|
| **Frontend** | React 18, TypeScript, Vite | Fast dev experience, type safety |
| **Styling** | Tailwind CSS, Radix UI, Framer Motion | Consistent design system, accessible primitives, smooth animations |
| **Routing** | Wouter | Tiny bundle, does what I need |
| **State** | TanStack Query | Server state management, caching, auto-refetching |
| **Backend** | Express.js, TypeScript | Simple, flexible, tons of middleware ecosystem |
| **Database** | Supabase (Postgres) | Auth out of the box, RLS policies, realtime |
| **AI** | Anthropic Claude API | Best code understanding I tested, structured output |
| **GitHub** | Octokit, GitHub Apps | Webhooks, installation tokens, repo access |
| **Auth** | Supabase Auth + JWT | GitHub OAuth flow, session management |
| **Security** | Helmet, express-rate-limit, HMAC webhook verification | Standard hardening |

---

## Project structure

```
├── client/
│   ├── src/
│   │   ├── components/       # Reusable UI (dashboard layout, hero, pricing, etc.)
│   │   ├── contexts/         # Auth context provider
│   │   ├── hooks/            # Custom React hooks
│   │   ├── lib/              # Query client setup, utilities
│   │   └── pages/            # Route-level pages (17 pages total)
│   └── index.html
├── server/
│   ├── routes/
│   │   ├── webhook.js        # GitHub webhook handler (~970 lines, the main brain)
│   │   ├── github.js         # GitHub API integration routes
│   │   ├── reviews.js        # Review CRUD endpoints
│   │   ├── repos.js          # Repository management
│   │   ├── stats.js          # Analytics endpoints
│   │   └── plans.js          # Subscription/billing logic
│   ├── services/
│   │   ├── analyzer.js       # Claude AI integration, prompt engineering
│   │   ├── autofix.js        # Diff patch generation + auto-fix PRs
│   │   ├── security.js       # Vulnerability detection, supply chain analysis
│   │   ├── github.js         # Octokit helpers (file fetching, PR comments)
│   │   └── supabase.js       # Database operations
│   └── utils/                # Webhook verification, prompt templates
├── shared/
│   └── schema.ts             # Shared types (Drizzle + Zod)
├── database_setup.sql        # Main schema (subscriptions, reviews, repos, analytics)
└── github_integration_tables.sql  # GitHub installations, repos, onboarding
```

---

## Database design

I went with Supabase Postgres and designed around Row Level Security — every table has RLS policies so users can only see their own data, even if the API has a bug.

**Main tables:**
- `subscriptions` — plan info, Stripe IDs, billing period
- `reviews` — every PR review with full analysis JSON
- `repositories` — connected repos, monitoring status
- `user_analytics` — aggregated stats per user
- `github_installations` — GitHub App install data (account, permissions, events)
- `github_repositories` — repos selected for monitoring (with webhook tracking)
- `user_onboarding` — step-by-step onboarding progress

There are also database triggers that automatically create analytics entries and free subscriptions when new users sign up, so the onboarding is zero-friction.

---

## Some things I'm proud of

**The auto-fix pipeline.** When the analyzer finds a fixable issue (like a missing input validation or a deprecated function call), it generates a unified diff patch, applies it via the GitHub API, and opens a fix PR linked back to the original. Getting the diff parsing right was a pain.

**Webhook reliability.** GitHub webhooks are fire-and-forget — if your server crashes mid-analysis, you lose the event. I made sure the endpoint always returns 200 (so GitHub doesn't retry and spam you), logs the failure to the database, and posts an error comment on the PR so the developer knows something went wrong.

**The file content fallback chain.** When generating auto-fixes, you need the actual file content, not just the diff. But getting that content is surprisingly tricky. I built a 4-step fallback: GitHub Contents API → Blob SHA API → raw_url fetch → reconstruct from diff. One of those always works.

**Cost management.** Every decision in the pipeline has cost implications. Caching alone probably saved 60%+ of API calls. The token estimator prevents wasted requests, and the model fallback chain means we always use the cheapest model that works.

---

## What I'd do next

- [ ] Inline PR review comments (line-level, not just a big comment block)
- [ ] GitLab and Bitbucket support
- [ ] Review history diffing (compare how code quality changes over time)
- [ ] Team dashboards with aggregated metrics
- [ ] Stripe integration for actual payment processing
- [ ] Webhook retry queue with dead letter handling


