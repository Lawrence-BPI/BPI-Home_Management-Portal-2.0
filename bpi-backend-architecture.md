# BPI Home Portal — Backend Architecture

**Version 1 · Apr 30, 2026**
For Buffalo Property Inspections · DFW · Single-tenant build

---

## 1. The recommendation, in one paragraph

Build the backend on **Supabase** (Postgres + Auth + Storage in one managed service), deploy the frontend to **Vercel**, do PDF analysis by sending the file directly to **Claude Sonnet 4.6 via the Anthropic API**, and route transactional email through **Resend**. This stack is chosen because it (a) requires zero server administration on your part, (b) has a credible path from $0/month at zero customers to ~$50–80/month at hundreds of inspections per month with no architecture changes, (c) lets a single contractor build it in roughly 3–5 weeks, and (d) keeps every component swappable later — Supabase is plain Postgres, Resend is plain SMTP, and the Claude API call is a 30-line function that could be replaced with OpenAI or anything else if pricing or capability changes.

Total recurring cost at production launch (10–30 inspections/month): about **$25/month base** (Supabase Pro) plus **~$2–5/month** in Claude API calls plus **~$0** for Resend (free tier covers it) and **$0** for Vercel (Hobby tier covers it). At 100+ inspections/month: roughly **$25 + $15–30 + $0 + $0 = $40–55/month**.

---

## 2. Why this stack and not others

I evaluated four realistic options. Here's the honest tradeoff table:

| Stack | Why it fits | Why it doesn't |
|---|---|---|
| **Supabase + Vercel + Claude API + Resend** ✅ | All managed. Postgres = standard, portable. Auth + storage in one place. Magic-link auth is built in. Cheap at low scale. | You're tied to Supabase's dashboard for DB admin; if they raise prices significantly, migration is a project (not a one-day swap). |
| Firebase + Vercel + Claude API | Also managed, similar price. | NoSQL data model fights against the relational shape of profiles ↔ findings ↔ photos. Auth is Google's, not magic-link-first. Vendor lock-in is stronger. |
| Self-hosted Node/Postgres on Railway or Render | Cheapest at scale, full control. | You said you can't write backend code — this requires it. Even pre-built templates need patching when things break at 2am. |
| AWS (Cognito + Lambda + RDS + S3 + SES) | Most flexible long-term. | Significantly more setup, multiple services to wire together, much harder for a non-developer to operate. Overkill for a single-tenant DFW business. |

**Why Supabase wins for your specific situation:** Auth is the hardest piece to build correctly, and Supabase's auth handles magic links, session refresh, password recovery, OAuth, MFA, and row-level security policies *out of the box*. You'd otherwise pay a developer 1–2 weeks just to build that part. Postgres-based means your data is portable — if Supabase ever disappears, the DB exports as standard SQL.

**Why Claude over OpenAI for PDF parsing:** I checked the specifics. Claude takes PDFs as a native `document` content block — no separate text-extraction step, the model handles the vision component itself, and the Citations feature can ground each extracted field (address, inspector, finding) to a specific page span in the source PDF. That last part matters specifically for an inspection-report parser: you'll want auditability for "where did this address come from in the report" when something looks wrong. OpenAI can do similar work but the workflow involves more glue code. Pricing is comparable; Claude is slightly more expensive per token ($3/$15 per million for Sonnet 4.6 vs $2.50/$10 for GPT-4o) but the difference at your volume is single-digit dollars per month.

**Why Sonnet 4.6 specifically (not Opus, not Haiku):** Opus is overkill for structured extraction from a templated form — you'd be paying 1.7x for capability you don't need. Haiku is cheap but the long-tail of weird PDF layouts (scanned reports, hand-typed cover sheets, multi-column inspector notes) is where Sonnet's quality earns its price. If your PDFs turn out to all come from one inspection software with a consistent layout, you can downgrade to Haiku later and cut LLM costs by 70%.

---

## 3. Data model

The shape of the data falls out of the lifecycle we already mapped: profiles get created first, get an inspection report attached, and have many findings.

```
auth.users                      ← Supabase manages this table
  id (uuid)
  email
  …

profiles                        ← inspection profiles
  id (uuid, primary key)
  client_email      (text, indexed)        ← the homeowner who can see this
  created_by        (uuid, fk → auth.users) ← inspector or self
  property_address  (text)
  property_city, property_state, property_zip
  property_type     (text)
  client_name       (text)
  status            (enum: 'empty' | 'parsing' | 'ready' | 'failed')
  created_at, updated_at (timestamps)

inspection_reports              ← one report per profile (for now)
  id (uuid)
  profile_id        (uuid, fk → profiles)
  pdf_storage_path  (text)        ← path in Supabase Storage
  pdf_filename      (text)
  inspector_name    (text)
  inspector_trec_license (text)
  inspection_date   (date)
  report_number     (text)
  parse_status      (enum: 'pending' | 'extracting' | 'reviewed' | 'failed')
  parse_error       (text)        ← if failed
  raw_extraction    (jsonb)       ← what the LLM returned, before review
  parsed_at         (timestamp)
  reviewed_at       (timestamp)
  reviewed_by       (uuid, fk → auth.users)

findings                        ← deficiencies parsed from the report
  id (uuid)
  report_id         (uuid, fk → inspection_reports)
  system            (enum: structural/electrical/hvac/plumbing/appliances/optional)
  section           (text)        ← e.g. "II.A Foundations"
  title             (text)
  description       (text)        ← the inspector's exact note
  plain_summary     (text)        ← AI-generated plain-English version
  severity          (enum: safety/high/standard/maintenance/diy)
  cost_low_cents    (integer)
  cost_high_cents   (integer)
  recommended_pro   (text)
  source_page       (integer)     ← citation: which PDF page
  source_quote      (text)        ← citation: the exact text used

intake_links                    ← magic links sent by inspectors
  id (uuid)
  profile_id        (uuid)
  client_email      (text)
  sent_at           (timestamp)
  consumed_at       (timestamp, nullable)
  expires_at        (timestamp)
```

**Row-level security policies** (Supabase enforces these in the database itself, so even a bug in the frontend can't leak another client's data):

- A client can `SELECT` from `profiles` only `WHERE client_email = auth.jwt() ->> 'email'`.
- An admin (member of the `admins` table or matching `@buffalopropertyinspections.com`) can `SELECT` from `profiles` without filter.
- All write operations on `findings` go through a stored procedure that checks the user owns the parent profile.
- `inspection_reports.pdf_storage_path` is only readable via a Supabase Storage signed URL that expires in 5 minutes.

This is the part that's nearly impossible to get right when rolling your own auth, and is the main reason the recommendation is not "build it from scratch on AWS."

---

## 4. The PDF → profile pipeline

This is the piece you most need to work, so let me describe it precisely.

```
┌──────────────────┐     ┌────────────────┐     ┌───────────────┐
│  Client uploads  │ ──▶ │  Supabase      │ ──▶ │  Edge function │
│  PDF in browser  │     │  Storage       │     │  triggered     │
└──────────────────┘     └────────────────┘     └───────────────┘
                                                        │
                                                        ▼
                                              ┌────────────────┐
                                              │  Calls Claude  │
                                              │  API with PDF  │
                                              │  + extraction  │
                                              │  prompt        │
                                              └────────────────┘
                                                        │
                                                        ▼
                                              ┌────────────────┐
                                              │  Returns JSON  │
                                              │  (address,     │
                                              │  findings, …)  │
                                              └────────────────┘
                                                        │
                                                        ▼
                                              ┌────────────────┐
                                              │  Stored as     │
                                              │  raw_extraction│
                                              │  pending review│
                                              └────────────────┘
                                                        │
                                                        ▼
                                              ┌────────────────┐
                                              │  Inspector or  │
                                              │  client opens  │
                                              │  review screen │
                                              └────────────────┘
                                                        │
                                                        ▼
                                              ┌────────────────┐
                                              │  Edits/confirms│
                                              │  → findings    │
                                              │  table written │
                                              └────────────────┘
```

**Critical implementation notes:**

1. **The PDF goes to Claude as a `document` block, not pre-extracted text.** Claude's vision component reads the actual PDF, including diagrams, photo captions, and tables — which a text-only extractor would miss.

2. **The extraction prompt asks for structured JSON with citations.** Each field (address, inspector name, every finding) returns with a `source_page` and `source_quote` so a reviewer can verify it. This is the genuine value of using an LLM here vs regex — it can read a scanned cover page and know "this is the property address" even when no field label says so.

3. **Nothing auto-publishes.** The LLM output goes into `raw_extraction` and the report's `parse_status` becomes `'extracting' → 'reviewed'`. The findings only land in the queryable `findings` table after the inspector or client confirms the review screen. This is the "confirm-and-edit" step you asked for.

4. **The review screen is the most important piece of UX in this whole pipeline.** Each extracted field shows side-by-side with a "this came from page X — show me" link that opens the PDF to that page. The reviewer can edit any field inline and click Confirm. Misextractions become a 5-second fix instead of a 5-minute hunt.

5. **Failures fail loud, not quiet.** If the LLM can't find an address, the field renders as an empty input with a red badge "couldn't extract — please type." The system never silently inserts placeholder data.

**Estimated cost per PDF parse:** A 40-page TREC report is roughly 60–120k input tokens (Claude treats each page as ~1500–3000 tokens including the visual rendering) plus ~3–8k output tokens for the structured JSON. At Sonnet 4.6 pricing that's:

- Input: 100k × $3/M = **$0.30**
- Output: 5k × $15/M = **$0.075**
- **Per-report cost: ~$0.35–0.50**

Even at 100 reports/month, that's **$35–50/month in LLM costs**. The Batch API (50% off, 24-hour latency) could halve this for non-urgent reports, but real-time is more useful for client UX.

---

## 5. Auth flow (with Supabase doing the work)

The current prototype simulates magic links. Supabase does it for real, with these specific differences:

| Prototype (today) | Production (Supabase) |
|---|---|
| Magic-link tokens stored in localStorage | Tokens stored server-side, signed JWT |
| "Demo: open link" button surfaces the URL | Real email lands in inbox via Resend |
| Single-device only (localStorage) | Cross-device — sign in on phone, see profiles on laptop |
| Admin = email matches `@buffalopropertyinspections.com` | Admin = membership in `admins` table OR matches pattern (configurable in DB) |
| Tokens expire after 15 min, no rotation | Tokens expire after configurable TTL, refresh tokens auto-rotate |

Supabase's magic-link flow is identical from the user's perspective: enter email → check inbox → click link → signed in. The replacement work is roughly 100 lines of frontend code: replace the `auth.requestMagicLink` and `auth.verifyMagicLink` bodies in the existing API layer with `supabase.auth.signInWithOtp()` and `supabase.auth.exchangeCodeForSession()`. The rest of the app (which already routes through `auth.*` and `profiles.*`) doesn't change.

---

## 6. Phased build plan

I'd structure this as four phases. Each phase should be a separately reviewable, deployable checkpoint — not "build the whole thing for 6 weeks then demo."

### Phase 1 — Backend skeleton (week 1)
- Supabase project provisioned, schema created, RLS policies written and tested
- Vercel project deployed with the existing HTML as a static site
- Resend account set up, sender domain verified (`team@buffalopropertyinspections.com`)
- Sign-in flow works end-to-end with real email delivery
- Existing localStorage profiles have a one-time export/import path

**Deliverable:** Working sign-in with real magic links. No PDF features yet.

### Phase 2 — PDF pipeline (week 2)
- Edge function for PDF upload + Claude API call
- Extraction prompt iterated against 5–10 real BPI inspection reports
- Review screen renders extracted fields with per-field source citations
- Confirm-and-edit writes to the `findings` table

**Deliverable:** Upload a real report, review the extracted fields, save the profile.

### Phase 3 — Inspector tools (week 3)
- Admin dashboard route lists all profiles with filtering
- "Create profile for client" form
- Send-intake-link button + copyable URL fallback
- Audit log of who edited what (low-rent version: `created_by` and `updated_by` columns)

**Deliverable:** Inspectors can onboard clients without the client ever touching the upload screen.

### Phase 4 — Polish (weeks 4–5)
- Empty-state UX (profile exists, no report yet)
- Retry/re-upload flow when extraction fails
- Mobile QA pass on a real device
- Production data migration plan (export current localStorage profiles, import via admin tool)
- Cost monitoring dashboard (Anthropic console + Supabase usage)

**Deliverable:** Ready to onboard the first paying client.

---

## 7. What you (the BPI side) need to do

Even with a developer doing the implementation work, there are decisions only you can make:

1. **Decide which sender email magic links come from.** `team@buffalopropertyinspections.com` is the obvious choice but requires DNS records (SPF, DKIM, DMARC) on the domain. Resend will give the developer the exact records to add.
2. **Provide 5–10 sample inspection reports for prompt iteration.** These should span the variety of layouts your inspectors actually produce — not 10 copies of one template. Scrub any PII before sharing.
3. **Confirm the admin email allowlist.** Just `@buffalopropertyinspections.com`, or specific people? Add or remove patterns in one config file.
4. **Decide retention policy.** When a profile is deleted, do PDFs in storage delete immediately, or stay 30 days for recovery? GDPR and Texas don't require either, but you want a written answer.
5. **Decide on disclaimer flow.** Currently the disclaimer is per-session in the prototype. In production it should be acknowledged once per user and recorded in the database with timestamp and IP for legal defensibility.

---

## 8. Cost projection (12 months)

| Month | Inspections/month | Supabase | Claude API | Resend | Vercel | **Total** |
|---|---|---|---|---|---|---|
| 1 | 0 (build phase) | $0 (free tier) | ~$2 (testing) | $0 | $0 | **~$2** |
| 2 | 5 | $25 (Pro) | $3 | $0 | $0 | **~$28** |
| 3 | 15 | $25 | $7 | $0 | $0 | **~$32** |
| 6 | 40 | $25 | $18 | $0 | $0 | **~$43** |
| 12 | 100 | $25 + maybe $5 storage overage | $40 | $0 | $0 | **~$70** |

**What changes the picture:**

- If you add photos to findings (inspector photos of issues), Supabase Storage egress costs become the next variable above ~50GB/month of client traffic. At BPI's volume, unlikely to matter.
- If you start serving 5 inspectors who each create 50 profiles/month, the LLM bill scales linearly — that's where the Batch API (50% discount, 24-hour latency for non-urgent reports) becomes worth wiring in.
- If a competitor calls your sign-up endpoint 1M times to spam-create profiles, you'd want Cloudflare Turnstile (free) on the sign-in form. Not urgent at day one.

---

## 9. Risks and unknowns

Things I'm not certain about and would want to validate during Phase 2:

- **Variance in TREC PDF layouts.** I'm assuming Sonnet 4.6 handles the long tail well. The 5–10 sample reports in Phase 2 will tell us if extraction quality is good enough or if we need a fallback to manual entry more often than expected.
- **Photo extraction.** TREC reports often have inspector photos embedded. Pulling those out as separate images linked to specific findings is doable but adds storage cost and complexity. I'd punt this to Phase 5 unless you specifically want it on day one.
- **Mobile camera upload of paper reports.** If your clients sometimes have *physical* inspection reports (not digital PDFs), they'd want to take photos and have them OCR'd. This is a different path — Claude can absolutely do it (vision model), but the prompt needs different treatment than a clean PDF. Worth validating whether this is a real scenario before building.
- **Legacy data migration.** The current prototype has localStorage profiles in browsers wherever you've been testing. We need a one-time export step in Phase 1 so nothing is lost, even though the dataset is small.

---

## 10. Decision points before starting

To kick off Phase 1, the developer (or you, if you DIY some of it) needs concrete answers to:

- Domain name for the production app? (`portal.buffalopropertyinspections.com`?)
- Sender email for magic links? (Same domain? Subdomain?)
- Admin email pattern(s)? (Just `@buffalopropertyinspections.com`, or specific addresses?)
- Should inspector accounts be created by signup-and-promote, or pre-seeded?
- 5–10 sample inspection PDFs (PII scrubbed) for the extraction prompt iteration

Everything else (schema, prompts, UI) is decided by this document.

---

## Appendix A — Specific links and references

- Claude API PDF support: https://docs.claude.com/en/docs/build-with-claude/pdf-support
- Claude API pricing: https://docs.claude.com/en/about-claude/pricing
- Supabase Auth (magic links): https://supabase.com/docs/guides/auth/auth-email-passwordless
- Supabase Row Level Security: https://supabase.com/docs/guides/auth/row-level-security
- Resend (transactional email): https://resend.com/docs
- Vercel deployment: https://vercel.com/docs

## Appendix B — Why I'm flagging this as the right path even though it's "more setup work"

You picked "cheapest credible path, even if more setup work." That's the right instinct, and Supabase is genuinely the cheapest credible path. The cheaper alternatives (truly free hosting, fully self-hosted) cross a line where setup turns into ongoing maintenance — patching Postgres, renewing TLS certificates, monitoring uptime. With Supabase you get the cheap-month-1 *and* the no-3am-pages, which the alternatives don't combine.
