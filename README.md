# CV1QAR

Create a professional, ATS-friendly CV/resume for **1 QAR**. Country-aware structure, AI-assisted wording that never
invents facts, four ATS-safe templates, a mock payment flow ready to swap for a real gateway, and a real
selectable-text PDF export.

This is a working MVP, not a static mockup — the full wizard → AI generation → template selection → payment →
PDF download flow runs end-to-end locally with zero external accounts, using mock AI and mock payment providers.

---

## 1. Project structure

```
cv1qar/
├── app/                     # Next.js App Router
│   ├── page.tsx             # Landing page
│   ├── create/page.tsx      # Wizard (all 10 form steps + AI generation screen)
│   ├── preview/page.tsx     # CV preview, template switch, payment, download
│   ├── cv/[country]/        # SEO-friendly per-country landing pages
│   ├── privacy/, terms/     # Legal pages
│   ├── admin/                # Admin dashboard (protected)
│   └── api/                  # Server routes (generate, payment, pdf, admin, analytics)
├── components/
│   ├── wizard/               # One component per wizard step
│   ├── templates/            # On-screen HTML CV renderer (4 templates)
│   ├── pdf/                  # @react-pdf/renderer document (real PDF export)
│   └── ui/                   # Button, Field, ProgressBar, Card
├── services/                  # AIService, PaymentService, CVService, PDFService
├── country-rules/             # One file per country's CV conventions
├── database/                  # schema.sql (Postgres) + mockDb.ts (in-memory, dev)
├── lib/                       # countries list, localStorage autosave, analytics
├── types/                     # Shared TypeScript types
└── middleware.ts               # Protects /admin and /api/admin
```

## 2. Run it locally

```bash
npm install
cp .env.example .env.local   # defaults already run the app fully mocked
npm run dev
```

Open http://localhost:3000. No API keys are required to try the full flow — AI generation and payment both run
against built-in mocks until you configure real providers (see below).

Visit `/admin` (default login `admin` / `change-me`, set in `.env.local`) for the dashboard.

## 3. Environment variables

See `.env.example` for the full list. Nothing here is ever read on the client.

| Variable | Purpose |
|---|---|
| `DATABASE_URL` / `SUPABASE_*` | Postgres connection once you move off the in-memory mock DB |
| `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `USE_MOCK_AI` | AI provider for CV writing assistance |
| `PAYMENT_PROVIDER`, `PAYMENT_API_KEY`, `PAYMENT_SECRET`, `PAYMENT_WEBHOOK_SECRET` | Payment gateway for the 1 QAR charge |
| `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `ADMIN_SESSION_SECRET` | Admin dashboard login (replace with a real `admin_users` table before launch) |
| `NEXT_PUBLIC_APP_URL` | Used for metadata and building absolute URLs |

## 4. Database setup

The MVP ships with `database/mockDb.ts`, an in-memory store, so it runs with no external database. For production:

1. Provision Postgres (or Supabase).
2. `psql "$DATABASE_URL" -f database/schema.sql`
3. Replace the bodies of `services/CVService.ts`, `services/PaymentService.ts` (the `db.orders` calls), and
   `database/mockDb.ts` usages with real queries against the tables in `schema.sql`. The function signatures
   (`CVService.create`, `.get`, `.save`, `PaymentService.createPayment`, `.verifyPayment`, ...) are the contract the
   rest of the app already relies on — routes don't need to change.

## 5. AI API integration

All AI calls go through `services/AIService.ts`, called only from `app/api/generate/route.ts` — so a real API key
never reaches the browser.

1. Set `AI_API_KEY`, `AI_PROVIDER`, `AI_MODEL`, and `USE_MOCK_AI=false` in `.env.local`.
2. In `services/AIService.ts`, uncomment and adjust the `callModel()` fetch block (an Anthropic Messages API
   example is included) to match your provider.
3. Country-specific prompt guidance lives in `country-rules/<slug>.ts` and is automatically merged into the system
   prompt via `systemPromptFor()` — no other code changes needed to add a market.
4. The system prompt hard-codes the "never invent employers/titles/qualifications/achievements/dates/certifications"
   rules; keep these when customizing prompts.

## 6. Payment integration

All payment calls go through `services/PaymentService.ts`.

1. Set `PAYMENT_PROVIDER`, `PAYMENT_API_KEY`, `PAYMENT_SECRET`, `PAYMENT_WEBHOOK_SECRET` in `.env.local`.
2. In `createPayment()`, replace the mock branch with your gateway's checkout/session creation call, storing
   `providerReference` on the order.
3. In `app/api/payment/webhook/route.ts` (already wired up and provider-agnostic), verify the signature in
   `handleWebhook()` using `PAYMENT_WEBHOOK_SECRET` before flipping an order to `succeeded`.
4. **Critical:** `app/api/pdf/route.ts` only unlocks the PDF after `PaymentService.verifyPayment()` returns
   `succeeded` from the server-side order record — it never trusts a client-supplied "paid" flag. Keep this
   invariant when you swap in a real provider.
5. Once a real provider is wired up, delete/disable `app/api/payment/confirm/route.ts` — it exists only to
   simulate the mock provider's checkout completion in place of a real hosted payment page.

## 7. PDF generation

`services/PDFService.ts` uses [`@react-pdf/renderer`](https://react-pdf.org) to render `components/pdf/CvPdfDocument.tsx`
directly to a PDF buffer — this produces real selectable/searchable text (not a screenshot), matches the four
templates, and paginates automatically for longer CVs. No further setup is required; it works out of the box.

## 8. Deployment

1. Push to GitHub, import into Vercel (or any Next.js-compatible host).
2. Set all variables from `.env.example` in the host's environment settings.
3. Provision Postgres and run `database/schema.sql` before switching `CVService`/`PaymentService` off the mock DB.
4. Set `NEXT_PUBLIC_APP_URL` to your production domain.
5. Rotate `ADMIN_PASSWORD` and `ADMIN_SESSION_SECRET` away from the defaults.

## 9. Security checklist

- [x] No API keys or secrets referenced in any client component — all AI/payment calls are server routes.
- [x] `/admin` and `/api/admin/*` protected by `middleware.ts` (signed session cookie).
- [x] PDF download requires a server-verified `succeeded` payment status (`app/api/pdf/route.ts`), never a client flag.
- [x] Generated files are tracked with a private `storagePath`, never a public URL.
- [x] All optional personal fields (photo, DOB, nationality) are opt-in per country, never required.
- [x] Friendly error messages everywhere; no stack traces or internals returned to the client.
- [ ] Before launch: replace `ADMIN_USERNAME`/`ADMIN_PASSWORD` env-var auth with a real `admin_users` table + hashed
      passwords (schema already included).
- [ ] Before launch: add rate limiting to `/api/generate`, `/api/payment/*`, and `/api/pdf`.
- [ ] Before launch: add real input validation/sanitization (e.g. zod) on every API route body.
- [ ] Before launch: replace the in-memory `mockDb` with Postgres — it resets on every restart and is not
      multi-instance safe.
- [ ] Before launch: legal review of `/privacy` and `/terms`, which currently contain placeholder text.

## 10. What's mocked vs. fully functional

| Feature | Status |
|---|---|
| Landing page, wizard, all 10 steps, autosave to localStorage | ✅ Fully functional |
| Country-specific rules (16 markets) | ✅ Fully functional, data-driven, easy to extend |
| AI summary / bullet polishing / skill suggestions | ⚠️ Mock heuristic rewriter by default — swap in a real model via `AI_API_KEY` + `USE_MOCK_AI=false` |
| Template selection + live on-screen preview (4 templates) | ✅ Fully functional |
| PDF generation (selectable text, ATS-safe, paginated) | ✅ Fully functional (`@react-pdf/renderer`), no further setup needed |
| Payment flow (1 QAR) | ⚠️ Mock provider auto-confirms success by default — swap in a real gateway via `PaymentService` |
| Payment verification gating the PDF | ✅ Fully functional against whichever provider is configured |
| Database | ⚠️ In-memory mock by default — `schema.sql` ready for Postgres/Supabase |
| Admin dashboard (stats, recent orders) | ✅ Functional against the mock DB; country/template/pricing management UI is scaffolded as a future admin feature, not yet built |
| Admin auth | ⚠️ Simple env-var login — replace with hashed-password table before launch |
| Referral system | ⚠️ Database columns present (`users.referral_code`, `referred_by`); no UI/logic yet — architecture only, as scoped |
| Analytics | ✅ Functional event capture (`visitor`, `country_selected`, `cv_started`, `cv_completed`, `payment_initiated`, `payment_successful`, `download_completed`) into the mock DB; swap for a real analytics sink in production |
| Account/data deletion | ⚠️ Schema supports soft-delete (`users.deleted_at`); no self-serve UI yet |
