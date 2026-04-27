# XpensePro

Read-only Plaid-powered expense visibility for small-business owners across multiple credit cards. Reports, alerts, and "what to cancel" recommendations — no transaction CRUD.

- **Live URL:** https://xpensepro.com (provisioning)
- **Status:** Pre-scaffold — runbook complete, awaiting credentials in 1Password
- **Stack:** Next.js (static export) + Supabase + Stripe + Plaid + Anthropic, hosted on Hostinger Business
- **Pricing:** Solo $29/mo · Team $79/mo · Pro $199/mo

See [`docs/RUNBOOK_xpensepro.md`](docs/RUNBOOK_xpensepro.md) for the full architecture, credentials list, and Ariel-led execution plan.

## What it does
1. Connect multiple credit cards via Plaid Link
2. AI-categorize every charge (Anthropic)
3. Detect recurring subscriptions automatically
4. Alert when a category trends over budget
5. Surface "cancel candidates" — high-cost recurring charges with weak signal of use
6. Weekly digest email (Resend)

## What it does NOT do
- Edit, add, or delete transactions
- Move money
- Sell or share data
