# Building a paid NYC business directory, in public

This repo explains how I built a paid local business directory, later named BizSpotNY. It covers what I used, why I chose it, and how data moves through the system. It describes the system rather than shipping the product code, so it's a write-up of system thinking, not a clone-and-run template.

- 🎥 **Demo (7 min):** https://youtu.be/PIgH6LWY_Bg
- 📝 [What vibe coding can't teach you: why fundamentals still matter](https://robkleiman.net/writing/so-heres-what-vibe-coding-cant-teach-you-why-fundamentals-still-matter)
- 📝 [Learning a production stack by shipping a paid directory](https://robkleiman.net/writing/learning-a-production-stack-by-shipping-a-paid-directory)
- 🔀 **[DATA-FLOW.md](DATA-FLOW.md):** every path through the system, with diagrams

## The goal

I wanted to learn a production stack by shipping a real system: one with accounts, payments, admin controls, and the edge cases that come with handling money. A tutorial skips those parts. The result took about three months and 230 commits, built with Claude Code as a collaborator.

## What I used, and why

| Layer           | Choice                                                            | Why                                                                                                                                                                                                                |
| --------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Web framework   | **Astro** (server-rendered) with small **Preact** islands         | Pages are mostly HTML and fast by default. JavaScript ships only where something is interactive, like search filters, the mobile menu, and the sticky search bar. API routes live next to the pages that use them. |
| Database        | **Supabase Postgres**                                             | Real SQL, migrations in git, and **row-level security**, so access rules are enforced by the database itself rather than by every API route remembering to check.                                                  |
| Accounts        | **Supabase Auth** (cookie sessions)                               | Lives in the same system as the data, so "who is this user" and "what can they see" use one identity.                                                                                                              |
| Files           | **Supabase Storage**                                              | Logos and photos, with the same row-level rules: anyone can read, and only the owner can change.                                                                                                                   |
| Background jobs | **Supabase Edge Functions + pg_cron**                             | Scheduled work, like reminder emails for unfinished listings, runs next to the data without a separate job server.                                                                                                 |
| Payments        | **Stripe Checkout, subscriptions, webhooks**                      | Stripe hosts the card form, so card data never touches my server. Subscriptions handle renewals, promo codes, cancellations, and refunds.                                                                          |
| Email           | **Brevo**                                                         | Transactional email (listing live, upgrade confirmed, reminders) through a simple API.                                                                                                                             |
| Validation      | **Zod** + **DOMPurify**                                           | Every input is checked against a schema, and every piece of text is sanitized, before it reaches the database or the page.                                                                                         |
| Styling         | **Tailwind CSS**                                                  | Fast iteration on UI without a growing pile of custom CSS.                                                                                                                                                         |
| Hosting         | **Vercel**                                                        | Deploys from git, previews every branch, and has a first-party Astro adapter.                                                                                                                                      |
| Tests           | **Vitest** (unit, API, integration) + **Playwright** (end to end) | API tests catch contract breaks, integration tests check the security rules against a real database, and end-to-end tests walk the submit-and-pay flow.                                                            |
| AI tooling      | **Claude Code** + MCP servers                                     | Claude Code had read access to Stripe and Supabase data through MCP. Write operations stayed under my control.                                                                                                     |

## Design principles that held up

1. **Security lives in the database.** Row-level security decides who can read or change each listing. An API bug can't leak data the database refuses to return.
2. **Money writes happen in one place.** Only the Stripe webhook, verified by its signature, marks a listing as paid. The browser's "success" page never changes billing state.
3. **Every webhook is idempotent.** Stripe retries events, so each event ID is recorded after it's handled and skipped if it arrives again. An event is recorded only when the update succeeds, so a failed update gets retried.
4. **Side effects can't break payments.** If a confirmation email fails, the payment still counts, and the failure is logged.
5. **One source of truth.** Listing statuses (`draft → pending → approved / rejected`) and plan features are each defined once, and every page, API route, and test reads from that one definition.
6. **Lazy initialization.** Clients that need runtime credentials are created on first use, not at import, so the build doesn't fail when credentials only exist in production.
7. **Feature gating by plan.** Paid features check a single plan config, so they can be turned on, off, or moved between tiers without hunting through components.

## What I learned

- **AI accelerated well-defined work.** Migrations, test scaffolding, and CSS cleanup went roughly 3–4x faster.
- **AI struggled with ambiguous bugs.** It kept fixing symptoms instead of the root cause. The bugs that mattered all took fundamentals to fix:
  - a routing collision between two dynamic routes that caused 404s only in production
  - stale client-side navigation state
  - duplicate Stripe webhook processing
  - a build that broke because credentials only existed at runtime
- **Shipping something that handles money is the fastest way to learn a stack.** The edge cases (retries, refunds, cancellations, failed renewals) teach more than the happy path.

> The AI accelerates what you already understand. It obscures what you don't, and that's where production bugs hide.

## What's not here

This repo covers architecture and lessons. The product code, data sourcing, pricing, and growth strategy stay private.

© Rob Kleiman. Shared for learning; please link back if you reference it.
