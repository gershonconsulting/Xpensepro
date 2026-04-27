# RUNBOOK — XpensePro

## Section 1 — Project identity

| Field | Value |
|---|---|
| Name | **XpensePro** |
| Domain | `xpensepro.com` |
| Live URL | `https://xpensepro.com` |
| One-line purpose | Read-only Plaid-powered expense visibility for small-business owners across multiple credit cards — reports, alerts, and "what to cancel" recommendations. No transaction CRUD. |
| Site type | Next.js with `output: 'export'` + Supabase + Stripe |
| Hosting | Hostinger Business shared (static export) |
| Database | **NEW** dedicated Supabase project `xpensepro-prod` (financial data → strict isolation, do NOT reuse Gershon project) |
| Auth | Magic link + Google OAuth + LinkedIn OAuth |
| Payments | Stripe — **$29 Solo / $79 Team / $199 Pro** (monthly + annual) |
| AI | Anthropic — categorization, recurring detection, cancel-recommendation reasoning |
| Transactional email | Resend (alerts + weekly digests) |
| CDN/DNS | Cloudflare in front (new domain → safe to migrate NS day one) |
| Registrar | Tierra.net |
| Executor | **Ariel as primary** (driving browser + terminal). Charles only as fallback for OAuth consoles that block headless. |
| Repo branch | `claude/expense-tracking-app-OGkCs` |

### Architectural notes
- **Plaid webhooks land on a Supabase Edge Function**, not Hostinger. Keeps shared-hosting RAM clean.
- **No transaction CRUD** anywhere in the UI. Plaid is the source of truth. The app reads, categorizes (AI), aggregates, and alerts. Users can recategorize and tag as personal/business, but never edit amounts/dates/merchants.
- **Plaid products requested:** `transactions`, `liabilities` (for credit-card balance + APR + min-payment context).
- **Plaid environments:** sandbox first → development for testing with real Plaid sandbox banks → production after Plaid review.

---

## Section 2 — Credentials needed from Olivier

Drop each in 1Password under `xpensepro` tag. Where each goes is labeled.

| # | Credential | Where to get it | Where it goes |
|---|---|---|---|
| 1 | **Plaid `client_id`** | dashboard.plaid.com → Team Settings → Keys | GitHub Secrets (`PLAID_CLIENT_ID`) + Supabase Edge Function secrets |
| 2 | **Plaid sandbox secret** | dashboard.plaid.com → Keys → Sandbox | GitHub Secrets (`PLAID_SECRET_SANDBOX`) + Supabase |
| 3 | **Plaid development secret** | dashboard.plaid.com → Keys → Development | GitHub Secrets (`PLAID_SECRET_DEVELOPMENT`) + Supabase |
| 4 | **Plaid production secret** | After Plaid prod approval | Hold until Phase 6 |
| 5 | **Supabase Project URL** | supabase.com → new project `xpensepro-prod` → Settings → API | GitHub Secrets (`NEXT_PUBLIC_SUPABASE_URL`) + `.env.local` |
| 6 | **Supabase anon key** | Same screen | GitHub Secrets (`NEXT_PUBLIC_SUPABASE_ANON_KEY`) + `.env.local` |
| 7 | **Supabase service role key** | Same screen — Reveal | Supabase Edge Function secrets only (NEVER ship to client) |
| 8 | **Google OAuth Client ID + Secret** | console.cloud.google.com → APIs & Services → Credentials → Create OAuth 2.0 Client ID (Web). **Redirect URI to paste:** `https://<supabase-project-ref>.supabase.co/auth/v1/callback` | Supabase → Authentication → Providers → Google |
| 9 | **LinkedIn OAuth Client ID + Secret** | linkedin.com/developers → Create app → Auth tab. **Redirect URI:** `https://<supabase-project-ref>.supabase.co/auth/v1/callback`. **Scopes:** `openid profile email` | Supabase → Authentication → Providers → LinkedIn (OIDC) |
| 10 | **Stripe publishable key** | dashboard.stripe.com → Developers → API keys | GitHub Secrets (`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`) |
| 11 | **Stripe secret key** | Same — Reveal | Supabase Edge Function secrets (`STRIPE_SECRET_KEY`) |
| 12 | **Stripe webhook signing secret** | Stripe → Developers → Webhooks → Add endpoint pointing to Supabase Edge Function → Signing secret | Supabase Edge Function secrets (`STRIPE_WEBHOOK_SECRET`) |
| 13 | **Stripe Price IDs** | Stripe → Products. Create 6 prices: Solo $29/mo + $290/yr, Team $79/mo + $790/yr, Pro $199/mo + $1990/yr | GitHub Secrets (`STRIPE_PRICE_SOLO_MONTHLY`, etc., 6 total) |
| 14 | **Anthropic API key** | console.anthropic.com → API Keys. **Set monthly cap = $50** to start | Supabase Edge Function secrets (`ANTHROPIC_API_KEY`) |
| 15 | **Resend API key** | resend.com → API Keys. Verify `xpensepro.com` domain (DNS records auto-provided) | Supabase Edge Function secrets (`RESEND_API_KEY`) |

---

## Section 3 — Phase-by-phase execution (Ariel-led)

Order matters: Cloudflare zone before Tierra NS change, Hostinger before GitHub Actions push.

### Phase 1 — Cloudflare zone (Ariel)
1. cloudflare.com → Add a Site → `xpensepro.com` → Free plan
2. Cloudflare scans existing records (should be empty for a new domain). Cloudflare returns 2 nameservers — copy them.
3. Hold for Phase 2.

### Phase 2 — Tierra.net nameserver swap (Ariel)
1. tierra.net → log in → My Domains → `xpensepro.com` → Nameservers
2. Replace defaults with the 2 Cloudflare nameservers from Phase 1
3. Save. Propagation: 15 min to 4 hours.
4. Back in Cloudflare → confirm zone status flips to **Active** before continuing.

### Phase 3 — Hostinger provisioning (Ariel)
1. hpanel.hostinger.com → **Add Website** → `xpensepro.com` → choose existing Business plan
2. Hostinger asks "Where is the domain?" → "I have it elsewhere" → it provisions and gives an **A record IP** — write this down.
3. **FTP user** (dedicated per site):
   - Files → FTP Accounts → Create FTP Account
   - Username: `xpensepro_ftp` · Path: `/domains/xpensepro.com/public_html` · Password: generate, store in 1Password
4. **Email mailboxes** → Emails → `xpensepro.com`:
   - Create mailbox `info@xpensepro.com` (real inbox, password in 1Password)
   - Create 4 forwarders → all → `info@xpensepro.com`:
     - `support@xpensepro.com`
     - `sales@xpensepro.com`
     - `billing@xpensepro.com`
     - `social@xpensepro.com`
5. **SSL** → Security → SSL → install Let's Encrypt → Force HTTPS ON
6. Note the Hostinger A-record IP for Phase 4.

### Phase 4 — Cloudflare DNS records (Ariel)
Add in Cloudflare → DNS → Records (proxy = orange cloud unless noted):

**Web:**
- `A   xpensepro.com   <Hostinger IP>   Proxied`
- `CNAME   www   xpensepro.com   Proxied`

**Email (NEVER proxy these — DNS only, grey cloud):**
- `MX   xpensepro.com   mx1.hostinger.com   priority 5   DNS only`
- `MX   xpensepro.com   mx2.hostinger.com   priority 10   DNS only`
- `TXT   xpensepro.com   v=spf1 include:_spf.mail.hostinger.com ~all   DNS only`
- `CNAME   hostingermail-a._domainkey   hostingermail-a.dkim.mail.hostinger.com   DNS only`
- `CNAME   hostingermail-b._domainkey   hostingermail-b.dkim.mail.hostinger.com   DNS only`
- `CNAME   hostingermail-c._domainkey   hostingermail-c.dkim.mail.hostinger.com   DNS only`
- `TXT   _dmarc   v=DMARC1; p=quarantine; rua=mailto:info@xpensepro.com   DNS only`

**Resend (added when Resend domain is verified — Resend will provide):**
- 2× CNAME for DKIM + 1 TXT for return-path

**SSL/TLS** → Overview → set to **Full (strict)**. Never Flexible.

### Phase 5 — Supabase project + schema (Ariel)
1. supabase.com → New project → name `xpensepro-prod` → choose region closest to Olivier → strong DB password to 1Password
2. Authentication → Providers:
   - Enable Email (magic link mode, disable signup-with-password)
   - Enable Google → paste creds from Section 2 #8
   - Enable LinkedIn (OIDC) → paste creds from Section 2 #9
3. Run initial schema migration via `supabase/migrations/0001_init.sql`:
   - `profiles` (id FK auth.users, full_name, company_name, plan_tier, stripe_customer_id, created_at)
   - `plaid_items` (id, user_id, plaid_item_id, plaid_access_token (encrypted), institution_name, institution_id, status, created_at)
   - `accounts` (id, plaid_item_id, plaid_account_id, name, mask, type, subtype, current_balance, limit, created_at)
   - `transactions` (id, account_id, plaid_transaction_id, date, name, merchant_name, amount, iso_currency_code, pending, plaid_category[], ai_category, ai_subcategory, is_recurring, recurring_stream_id, user_tag (personal/business/null), created_at) — **read-only from app perspective**
   - `recurring_streams` (id, user_id, merchant_name, normalized_name, frequency, average_amount, last_date, next_predicted_date, status (active/canceled/paused), cancel_recommendation, cancel_reason, last_used_at, created_at)
   - `budgets` (id, user_id, category, monthly_limit, period_start, alert_threshold_pct, created_at)
   - `alerts` (id, user_id, kind (over_budget/unusual_spend/cancel_suggestion/new_recurring), payload jsonb, sent_at, dismissed_at, created_at)
   - `subscriptions` (id, user_id, stripe_subscription_id, stripe_price_id, plan_tier, status, current_period_end, created_at)
4. **RLS on every table** — `auth.uid() = user_id` policies. Service role bypasses.
5. Edge Functions to deploy:
   - `plaid-link-token` — creates Plaid Link token for the front-end
   - `plaid-exchange-token` — exchanges public_token for access_token, stores encrypted
   - `plaid-webhook` — receives Plaid webhooks (TRANSACTIONS:DEFAULT_UPDATE, TRANSACTIONS:RECURRING_TRANSACTIONS_UPDATE, ITEM:ERROR, etc.)
   - `plaid-sync` — pulls transactions/recurring/liabilities, writes to DB, calls Anthropic for categorization
   - `stripe-webhook` — listens to subscription lifecycle
   - `weekly-digest` (cron, Mondays 7am) — composes report + alerts via Resend
   - `cancel-recommender` (cron, daily) — scans `recurring_streams`, generates Anthropic-reasoned recommendations
6. Plaid dashboard → Webhooks → set URL to the `plaid-webhook` Edge Function URL.

### Phase 6 — GitHub repo + auto-deploy (Ariel)
1. The repo already exists at `gershonconsulting/Xpensepro`, branch `claude/expense-tracking-app-OGkCs`. Ariel scaffolds Next.js inside this branch.
2. Scaffold:
   ```
   npx create-next-app@latest . --typescript --tailwind --app --src-dir --import-alias "@/*" --eslint
   ```
   Then add `output: 'export'` to `next.config.ts`.
3. Install: `@supabase/supabase-js`, `@supabase/ssr`, `react-plaid-link`, `stripe`, `@stripe/stripe-js`, `@anthropic-ai/sdk` (Edge Functions only), `resend` (Edge Functions only), `lucide-react`, `recharts`, `shadcn/ui` components.
4. Brand tokens in `globals.css` / Tailwind config:
   - `--brand-red: #CC3333`
   - `--brand-charcoal: #2a2a2a`
   - White bg, Helvetica-style sans, red accent borders
5. **GitHub Secrets** (Settings → Secrets and variables → Actions):
   - `FTP_SERVER` = `ftp://files.000webhost…` (from Hostinger FTP details)
   - `FTP_USERNAME` = `xpensepro_ftp`
   - `FTP_PASSWORD`
   - `FTP_SERVER_DIR` = `/domains/xpensepro.com/public_html/`
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
   - `STRIPE_PRICE_SOLO_MONTHLY` … `STRIPE_PRICE_PRO_ANNUAL` (6 total)
   - `NEXT_PUBLIC_SITE_URL` = `https://xpensepro.com`
6. `.github/workflows/deploy.yml` — Node static-export workflow:
   - Trigger: `push` to `main`
   - Steps: checkout → setup-node 20 → `npm ci` → `npm run build` → FTP-Deploy-Action with `local-dir: ./out/` and `server-dir: ${{ secrets.FTP_SERVER_DIR }}`
7. Merge `claude/expense-tracking-app-OGkCs` → `main` only when ready for first deploy. Until then, develop on the feature branch.

### Phase 7 — First deploy + verification (Ariel, then Olivier eyeballs)
- [ ] `git push origin main` → GH Actions green within 2 min
- [ ] `https://xpensepro.com` loads, no cert warning
- [ ] `https://www.xpensepro.com` redirects correctly
- [ ] Send test email to `info@xpensepro.com` → arrives in Hostinger inbox
- [ ] Send test email to `support@xpensepro.com` → forwards to `info@`
- [ ] Mail-tester.com score ≥ 9/10 (proves SPF/DKIM/DMARC)
- [ ] Plaid sandbox flow: connect a sandbox bank → transactions appear in Supabase → AI categories populate within 60s
- [ ] Stripe test mode: subscribe to Solo → webhook fires → `subscriptions` row created
- [ ] Lighthouse mobile ≥ 90 on Performance + Accessibility

---

## Section 4 — Common failure modes (XpensePro-specific additions)

| Symptom | Cause | Fix |
|---|---|---|
| `ERR_TOO_MANY_REDIRECTS` | Cloudflare SSL = Flexible | Switch to **Full (strict)** |
| Email broken after NS swap | MX/SPF/DKIM not added before NS change | Add the 7 email records to Cloudflare *before* the next mail send |
| Plaid webhook silently failing | Webhook URL not registered or signature mismatch | Plaid dashboard → Webhooks → verify URL; check `plaid-webhook` Edge Function logs |
| Plaid "INVALID_INSTITUTION" in dev | Sandbox secret used with non-sandbox institution | Use sandbox-only test institutions in sandbox env |
| Stripe webhook 400 | Wrong signing secret OR Edge Function reading raw body wrong | Re-copy signing secret. Stripe SDK needs the *raw* request body; do not parse JSON before `constructEvent` |
| Supabase 401 from front-end | RLS on but no policy for `auth.uid()` | Add `select using (auth.uid() = user_id)` per table |
| Anthropic 429 / cost spike | Categorization loop hitting every transaction every sync | Cache by (merchant_name, amount-bucket); only call AI for novel merchants |
| Recurring detection misses subs | Plaid `recurring_transactions` requires 90+ days of history | Show "still learning your subscriptions — check back in 2 weeks" until threshold met |
| Action green but site unchanged | Wrong `FTP_SERVER_DIR` | Verify path in Hostinger File Manager |
| Cancel recommendation feels wrong | We don't have usage data, only spend | Frame copy as "Review this — last charged X days ago, $Y/yr" — never assert "you should cancel" |

---

## Section 5 — Hand-back

It's live. Updates = `git push`. Credentials in 1Password under tag `xpensepro`. Live Sites Inventory updated.

**Cron jobs running:**
- `plaid-sync` — every 6 hours per item
- `cancel-recommender` — daily at 2am UTC
- `weekly-digest` — Mondays at 7am Olivier-local

**Anthropic monthly cap:** $50. Bump after 100 paid users.

**Plaid prod approval to-do (parallel track):**
- Privacy policy at `/privacy`
- Terms of service at `/terms`
- Data use disclosure: "We read transactions and account balances. We never move money. We never sell data."
- Submit to Plaid for production keys.
