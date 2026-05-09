# Payment Gateway Integration Plan — SHEH TRAVEL (InfinPay, B2G)

## Context

The client wants to add a payment capability to the SHEH TRAVEL website so that **government agency customers** (the dominant customer segment) can pay using their **Government Procurement Cards (GPC / corporate Visa/Mastercard)**. The chosen gateway is **InfinPay**. Documentation for InfinPay has not been provided yet, so this plan is structured so that all gateway-specific work is isolated to a single adapter file — once the InfinPay docs arrive, only that adapter and a few env vars need to be filled in.

**Chosen flow (per client confirmation):**
1. Government customer enquires via the existing contact form.
2. SHEH staff reviews the enquiry, prepares a custom quote inside Sanity Studio, marks it "ready to pay", and Sanity emits the public payment URL.
3. SHEH emails that link to the customer.
4. Customer opens the link, sees the quote summary, clicks "Pay with Corporate Card", and is redirected to InfinPay.
5. InfinPay processes the card, redirects the customer back, and posts a server-to-server callback to our webhook.
6. We verify the callback signature, mark the booking paid in Sanity, and email a payment receipt + PDF tax invoice to both the customer and the SHEH inbox.

**Out of scope for this phase:** self-service checkout, MyInvois (LHDN e-invoicing) API integration, refunds via dashboard, multi-currency. PDF tax invoices are generated locally and attached to email; MyInvois will be a Phase 2 follow-up.

---

## High-Level Architecture

```
[Customer enquiry] -> /api/contact (existing) -> SHEH inbox
        |
[SHEH staff] -> Sanity Studio -> creates `booking` doc (status=quoted)
        |                           |-> generates payToken (random 32-byte)
        |                           |-> emails customer the link to /pay/{payToken}
        v
[Customer opens] /pay/{payToken}
        | (server fetches booking by payToken; shows summary + Pay button)
        v
POST /api/payment/initiate { bookingId }
        | (server: re-reads price from Sanity, calls InfinPay createPayment adapter)
        v
Redirect 302 -> InfinPay hosted checkout
        v
[Customer pays on InfinPay page with corporate card]
        |
        |--(server-to-server)--> POST /api/payment/callback   <-- TRUSTED
        |                          (verify signature, idempotent update,
        |                           re-check amount, mark booking paid,
        |                           generate PDF invoice, send emails)
        |
        +--(browser redirect)--> GET /payment/success?ref=...  <-- DISPLAY ONLY
                                 GET /payment/failed?ref=...
```

The browser redirect is **display-only** — the source of truth for "paid" is the server-to-server callback after signature verification.

---

## Files to Be Modified or Created (Complete List)

### Sanity schemas (modify + create)

- **MODIFY** [sanity/schemas/site-settings.ts](sanity/schemas/site-settings.ts) — add company tax/registration fields (see §1).
- **MODIFY** [sanity/schemas/travel-package.ts](sanity/schemas/travel-package.ts) — add `priceMyr` (number) field alongside the existing display `price` string. Keeps the marketing copy intact while giving payments a real number.
- **CREATE** `sanity/schemas/booking.ts` — the booking/order document (see §1).
- **MODIFY** `sanity/schema.ts` (or wherever schemas are registered) — register the new `booking` schema.

### Environment / config

- **MODIFY** [.env.example](.env.example) — add InfinPay, Sanity write token, signing-secret, and base-URL keys.
- **MODIFY** [.env.local](.env.local) — same, with real values (and **rotate the currently-exposed Resend key first**).
- **MODIFY** [next.config.mjs](next.config.mjs) — extend CSP `connect-src` and `form-action` to allow InfinPay endpoints once their domain is known.

### Library code

- **MODIFY** [lib/sanity.client.ts](lib/sanity.client.ts) — export a separate `writeClient` that uses `SANITY_API_TOKEN` (server-only).
- **MODIFY** [lib/sanity.queries.ts](lib/sanity.queries.ts) — add `getBookingByPayToken`, `getBookingById`, `markBookingPaid`, `appendBookingEvent`.
- **CREATE** `lib/payments/infinpay.ts` — **the only file with InfinPay-specific code.** Exports `createPayment(booking)`, `verifyCallback(rawBody, headers)`, `parseCallback(rawBody)`. All other code calls this through a stable interface — this is where the InfinPay docs will be plugged in.
- **CREATE** `lib/payments/types.ts` — shared types: `PaymentSession`, `CallbackResult`, `PaymentStatus`.
- **CREATE** `lib/payments/idempotency.ts` — small helper to deduplicate webhook deliveries by `(bookingId, gatewayTxnId)`.
- **CREATE** `lib/invoice/generate-pdf.ts` — generates the tax-invoice PDF using `@react-pdf/renderer` (or `pdfkit`). Pulls company info from `getSiteSettings()`.
- **CREATE** `lib/invoice/invoice-number.ts` — generates a sequential invoice number per year (`INV-2026-0001`), stored as a Sanity counter doc.
- **CREATE** `lib/email/send-quote.ts`, `lib/email/send-receipt.ts` — Resend helpers, modeled on the existing pattern in [app/api/contact/route.ts](app/api/contact/route.ts).
- **CREATE** `lib/rate-limit.ts` — extract the in-memory limiter currently in the contact route into a reusable helper. (Future upgrade: swap to Upstash Redis when traffic grows — noted as a follow-up, not in this phase.)

### API routes (all server-only)

- **CREATE** `app/api/payment/initiate/route.ts` — POST. Re-reads booking from Sanity, recomputes amount, calls `createPayment()` adapter, returns redirect URL.
- **CREATE** `app/api/payment/callback/route.ts` — POST. **Critical**. Reads raw body, verifies signature via `verifyCallback()`, performs idempotent update on the booking doc, fires receipt email + invoice PDF on first success, returns the exact response InfinPay expects.
- **CREATE** `app/api/payment/status/route.ts` — GET. Polled by the success page to show the latest authoritative status (because the customer may land on `/payment/success` before the callback has finished).
- **CREATE** `app/api/quotes/[payToken]/route.ts` — GET. Public-but-unguessable read of the quote summary for the customer-facing payment page.

### Frontend pages

- **CREATE** `app/pay/[payToken]/page.tsx` — server component. Fetches the booking, shows quote summary (package, customer, agency reference / PO number, line items, subtotal, SST, total), and a single "Pay with Corporate Card" button. Refuses to render if status is not `quoted`.
- **CREATE** `app/pay/[payToken]/pay-button.tsx` — client component. POSTs to `/api/payment/initiate`, then `window.location` to the returned redirect URL.
- **CREATE** `app/payment/success/page.tsx` — polls `/api/payment/status` for up to ~30s, shows confirmation + "invoice on its way to your email."
- **CREATE** `app/payment/failed/page.tsx` — shows failure reason, "try again" link, and SHEH's contact details.

### No nav changes

These pages are **not** added to the public navigation — they are reached only via the unique payment link emailed to the customer. (Confirms B2G flow.)

---

## §1 — Sanity Schema Changes (Detail)

### `site-settings.ts` — additions

Add a new `business` group with:

| Field | Type | Required | Notes |
|---|---|---|---|
| `companyLegalName` | string | yes | Goes on the tax invoice header. |
| `registrationNumber` | string | yes | SSM company number (e.g. `202301012345 (1234567-A)`). |
| `sstNumber` | string | optional | SST registration number — only populate if registered. |
| `sstRate` | number | optional | Default `6` (percent). If empty, invoice shows no SST line. |
| `invoiceAddress` | text | yes | Full mailing/legal address used on invoices (may differ from contact-page address). |
| `invoiceFooterNote` | text | optional | e.g. payment terms, thank-you note, bank ref for offline payments. |

### `travel-package.ts` — additions

| Field | Type | Notes |
|---|---|---|
| `priceMyr` | number | The real numeric price used in checkout. Existing `price` (string) stays for display. |
| `priceIsFrom` | boolean | If true, label as "From RM X" — and force the booking schema's amount to be set manually by staff. |

### `booking.ts` — new schema (the order document)

| Field | Type | Notes |
|---|---|---|
| `bookingRef` | string | Human-readable, e.g. `SHT-2026-0001`. Generated on create. |
| `payToken` | string | 32-byte hex, unguessable. The customer's payment URL is `/pay/{payToken}`. |
| `status` | string | enum: `draft`, `quoted`, `paying`, `paid`, `failed`, `refunded`, `cancelled`. |
| `customer` | object | `{ agencyName, contactName, contactEmail, contactPhone, poNumber }` — `poNumber` is the government Local Order / Purchase Order ref. |
| `package` | reference | Optional link to a `travelPackage` doc. |
| `customItinerary` | text | Free-form for custom B2G itineraries that don't match a standard package. |
| `lineItems` | array | `[{ description, quantity, unitPriceMyr, lineTotalMyr }]`. Used by invoice PDF. |
| `subtotalMyr` | number | Computed but stored. |
| `sstAmountMyr` | number | Computed but stored. |
| `totalMyr` | number | Computed but stored. **This is the only amount sent to the gateway.** |
| `currency` | string | Default `MYR`. |
| `payment` | object | `{ gateway: 'infinpay', gatewayTxnId, gatewayRef, paidAt, last4, cardBrand, callbackPayload }` — populated by webhook. |
| `invoice` | object | `{ invoiceNumber, issuedAt, pdfAssetRef }` — populated after first successful payment. |
| `events` | array | Append-only audit log: `[{ at, type, note, actor }]` — every state change written here. |
| `createdAt`, `updatedAt` | datetime | |

**Indexes / uniqueness:** `payToken`, `bookingRef`, and `payment.gatewayTxnId` must be unique. Enforce in schema validation rules and at the query layer (always look up by these fields with `.unique()`).

---

## §2 — Environment Variables

Add to `.env.local` (and document in `.env.example` with placeholders):

```
# Sanity write access (server-only, NEVER prefix NEXT_PUBLIC_)
SANITY_API_TOKEN=<editor-or-write-scoped-token>

# Public site base URL (used to build callback / return URLs)
NEXT_PUBLIC_SITE_URL=https://www.shehtravel.com

# InfinPay (TODO: confirm exact names from their docs)
INFINPAY_MERCHANT_ID=
INFINPAY_API_KEY=
INFINPAY_SIGNING_SECRET=
INFINPAY_BASE_URL=                 # likely https://api.infinpay.com or sandbox equivalent
INFINPAY_ENVIRONMENT=sandbox       # sandbox | production
INFINPAY_RETURN_URL=https://www.shehtravel.com/payment/success
INFINPAY_CANCEL_URL=https://www.shehtravel.com/payment/failed
INFINPAY_CALLBACK_URL=https://www.shehtravel.com/api/payment/callback

# Existing
RESEND_API_KEY=<rotated key>
ENQUIRY_FALLBACK_EMAIL=info@shehtravel.com
```

**Pre-flight:** Rotate the currently-exposed Resend key in [.env.local](.env.local) before any of this ships.

---

## §3 — Backend API Detail

### `app/api/payment/initiate/route.ts` (POST)

1. Apply `lib/rate-limit.ts` — 10 req/min per IP.
2. Parse `{ payToken }` from body.
3. `getBookingByPayToken(payToken)` from Sanity. 404 if missing, 409 if status ≠ `quoted`.
4. Recompute `subtotal/SST/total` server-side from `lineItems` and `sstRate`. **Never trust the stored `totalMyr` blindly — recompute and compare.** If mismatch, refuse and log.
5. Call `infinpay.createPayment({ bookingRef, totalMyr, customer, returnUrl, cancelUrl, callbackUrl })`.
6. Update booking: status → `paying`, append event.
7. Return `{ redirectUrl }`.

### `app/api/payment/callback/route.ts` (POST) — **the security-critical one**

1. Read **raw body** (do not `await req.json()` first — most signature schemes hash the raw bytes).
2. Call `infinpay.verifyCallback(rawBody, request.headers)`. Reject 400 on failure. Log every failed verification.
3. Call `infinpay.parseCallback(rawBody)` → `{ bookingRef, gatewayTxnId, status, amountMyr, last4, cardBrand, raw }`.
4. Look up booking by `bookingRef`. 404 if missing.
5. **Idempotency check:** if `payment.gatewayTxnId` already equals this `gatewayTxnId` AND status is already terminal, return 200 immediately with no side effects.
6. **Amount cross-check:** `amountMyr` from callback must equal booking `totalMyr`. If mismatch, mark booking `failed` with event note, alert SHEH inbox, return 200 (don't let InfinPay retry forever).
7. On success status:
   - Generate invoice number via `lib/invoice/invoice-number.ts`.
   - Generate PDF via `lib/invoice/generate-pdf.ts`.
   - Upload PDF to Sanity assets, store the asset reference in `invoice.pdfAssetRef`.
   - Update booking: status → `paid`, populate `payment.*` and `invoice.*`, append event.
   - Send receipt email (with PDF) to `customer.contactEmail`.
   - Send notification email to `settings.enquiryEmail`.
8. On failure status: status → `failed`, append event, notify SHEH.
9. Return the response InfinPay expects (likely `200 OK` with a specific body — confirm from docs).

### `app/api/payment/status/route.ts` (GET)

Takes `?ref=SHT-2026-0001`. Returns `{ status }` only. Used by the success page to poll. No PII.

### `app/api/quotes/[payToken]/route.ts` (GET)

Returns the minimum fields needed to render the payment page: `bookingRef`, `customer.agencyName`, `customer.poNumber`, `lineItems`, `subtotalMyr`, `sstAmountMyr`, `totalMyr`, `status`. Never returns `payment.callbackPayload`, internal events, or other bookings.

---

## §4 — Frontend Pages Detail

### `app/pay/[payToken]/page.tsx`

Server-rendered. Fetches the booking, renders:

- Header: SHEH logo + "Payment for [bookingRef]"
- Customer block: agency name, PO number, contact (read-only)
- Itinerary / package summary
- Line-item table: description, qty, unit price (RM), line total
- Subtotal, SST (X%), **Total (RM)**
- "Pay with Corporate Card (Visa / Mastercard)" button (the `pay-button.tsx` client component)
- Footer: SHEH contact info + a note that they will be redirected to InfinPay's secure page

If `status !== 'quoted'`:
- `paying` → show "Payment in progress, please complete on the gateway"
- `paid` → show "Already paid — invoice [X] sent on [date]"
- `failed` → show "Previous attempt failed — contact SHEH to issue a new link"

### `app/payment/success/page.tsx`

Reads `?ref=` from URL. Polls `/api/payment/status?ref=…` every 2s, up to 30s. Shows:
- Spinner while polling
- ✅ "Payment received. Invoice [INV-YYYY-NNNN] has been emailed to [email]." once status hits `paid`.
- "Still processing — we'll email you the moment it confirms." if 30s elapses without `paid`.

### `app/payment/failed/page.tsx`

Static. Shows reason (from query param), retry instructions, SHEH WhatsApp / phone.

---

## §5 — Email & Invoice

### Quote email (sent by SHEH from Sanity)

Phase 1: Sanity Studio shows the payment URL on the booking doc. SHEH staff copy-paste it into a manual email (or, optionally, a "Send quote" button on the booking doc that triggers `lib/email/send-quote.ts`). The email contains: greeting, summary, total, link to `/pay/{payToken}`, validity period (default 14 days — store `quoteExpiresAt` on booking).

### Receipt email (sent automatically after callback success)

- From: `accounts@shehtravel.com` (must be verified domain in Resend — not `onboarding@resend.dev`).
- To: `customer.contactEmail`. CC: `settings.enquiryEmail`.
- Subject: `Payment Receipt — [bookingRef] — RM [total]`.
- Body: HTML modeled on the existing contact-route template ([app/api/contact/route.ts:130-174](app/api/contact/route.ts#L130-L174)).
- **Attachment**: the PDF tax invoice.

### Tax invoice PDF (`lib/invoice/generate-pdf.ts`)

Layout requirements (LHDN tax invoice essentials):
- Heading "TAX INVOICE"
- Invoice number, issue date
- SHEH legal name, address, registration number, SST number (from `site-settings`)
- Bill-to: agency name, address, PO number
- Line items with description, qty, unit price, line total
- Subtotal, SST amount and rate, **Total (RM)**
- Payment method: "Paid via InfinPay — corporate card ending [last4]" + gateway txn ID
- Footer note from `invoiceFooterNote`

Library: **`@react-pdf/renderer`** — same React mental model as the rest of the app, plays nicely with Next.js.

---

## §6 — InfinPay Adapter (`lib/payments/infinpay.ts`) — Plug-In Points

This file is intentionally the only place InfinPay-specific knowledge lives. When the docs arrive, fill in:

```ts
// TODO from InfinPay docs:
//   - exact endpoint for "create payment" (POST URL)
//   - request payload field names (merchant ID, ref, amount, currency, callback URL, return URL, customer fields)
//   - amount unit (RM ringgit vs sen)  ← critical, off-by-100 is a common bug
//   - signature scheme: hash algorithm (SHA256/SHA512/HMAC), which fields are signed, in what order
//   - the header/field name they use to send the signature on callbacks
//   - the success status string they send (often "00", "1", "SUCCESS", "CAPTURED")
//   - the response body they expect from our callback (often "OK", "RECEIVED" — gateway-specific)
//   - whether they require Level 2/3 card data fields and how to send them
//   - sandbox vs production base URLs and test card numbers
export async function createPayment(b: Booking): Promise<{ redirectUrl: string; gatewayRef: string }> { ... }
export function verifyCallback(rawBody: string, headers: Headers): boolean { ... }
export function parseCallback(rawBody: string): CallbackResult { ... }
```

Until InfinPay docs are received, the rest of the code can be built and tested with a **mock adapter** (`lib/payments/mock.ts`) selected by `INFINPAY_ENVIRONMENT=mock`. This unblocks all surrounding work.

---

## §7 — Security Hardening (must-haves before going live)

1. **Rotate the leaked Resend key** in [.env.local](.env.local) — do this first.
2. Resend `from:` must be a verified SHEH domain — not `onboarding@resend.dev` ([app/api/contact/route.ts:126](app/api/contact/route.ts#L126)).
3. Turn off `eslint.ignoreDuringBuilds` and `typescript.ignoreBuildErrors` in [next.config.mjs:3-8](next.config.mjs#L3-L8) and fix what surfaces.
4. CSP `script-src` currently has `'unsafe-inline' 'unsafe-eval'` ([next.config.mjs:51](next.config.mjs#L51)) — keep them only for the payment pages if InfinPay's redirect requires them; tighten on all other routes.
5. Extend CSP `connect-src` and `form-action` to include the InfinPay domain once known.
6. **Webhook signature verification** is non-negotiable. Reject any callback without a verified signature.
7. **Idempotency** on the callback — webhook will be retried, sometimes for the same event. Dedupe on `gatewayTxnId`.
8. **Server-side amount recomputation** in `/initiate` and **amount cross-check** in `/callback`. Never trust amounts from the browser or the gateway without verifying against the stored booking.
9. `payToken` must be `crypto.randomBytes(32).toString('hex')` — not predictable IDs.
10. `SANITY_API_TOKEN` is server-only; never `NEXT_PUBLIC_`. Use it only in `lib/sanity.client.ts`'s `writeClient`, called only from API routes.
11. Add an `origin` header check on `/api/payment/initiate` to reject cross-site POSTs.
12. Log every payment event (booking ref, IP, gateway txn ID, status) — keep at least 12 months for audit.
13. Add Cloudflare in front of Vercel for WAF + DDoS + bot management before go-live.
14. Confirm with InfinPay that they **accept government corporate cards** and support **Level 2/3 card data** if required by the GPC program. If yes, capture `poNumber` and tax fields and pass them in the `createPayment` payload.

---

## §8 — Order of Execution (suggested PR sequence)

1. **PR 1 — Foundations (no gateway needed):** rotate Resend key; add Sanity schema changes (`booking`, `site-settings` fields, `travel-package.priceMyr`); register schema; add `writeClient` and new queries; populate site-settings business fields in Studio.
2. **PR 2 — Adapter shell + mock:** create `lib/payments/types.ts`, `lib/payments/infinpay.ts` (with TODOs), `lib/payments/mock.ts`. Wire env vars.
3. **PR 3 — Customer-facing payment page:** `/pay/[payToken]` page + button + `/api/quotes/[payToken]` route. End-to-end with mock adapter.
4. **PR 4 — Initiate + callback routes** using mock adapter; idempotency + amount cross-check + event log; success/failed pages; status polling.
5. **PR 5 — Invoice PDF + receipt email:** `lib/invoice/*`, `lib/email/*`. Verify Resend domain.
6. **PR 6 — InfinPay adapter implementation** (when docs arrive). Sandbox testing with InfinPay's test cards.
7. **PR 7 — Security hardening:** CSP tightening, origin checks, ignoreBuildErrors removal, Cloudflare in front.
8. **PR 8 — Production cutover:** flip `INFINPAY_ENVIRONMENT=production`, real keys, smoke test with one real low-value transaction, then enable for all customers.

---

## Verification Plan

**Local / sandbox testing**

- [ ] `pnpm dev`. Open Sanity Studio. Create a booking with a small `totalMyr` (e.g. 1.00).
- [ ] Copy the generated `/pay/{payToken}` URL. Open in an incognito window.
- [ ] Confirm the payment page renders the right summary (no other booking's data leaks).
- [ ] Click "Pay" — with mock adapter, confirm redirect to a fake gateway page; with InfinPay sandbox, complete payment with their test card.
- [ ] Confirm `/api/payment/callback` is hit, signature verifies, booking moves to `paid`.
- [ ] Confirm receipt email arrives with PDF invoice; PDF has correct SSM, SST, line items, total.
- [ ] Open the same `/pay/{payToken}` URL again — confirm "already paid" state.
- [ ] Hit the callback endpoint twice with the same payload — confirm idempotency (no double receipt).
- [ ] Tamper with the signature header — confirm the callback returns 400 and booking stays unchanged.
- [ ] Tamper with the amount in the callback payload — confirm booking is marked `failed` and SHEH is alerted.
- [ ] Try `/pay/<random-32-bytes>` — confirm 404, no information leaked.
- [ ] Run `pnpm build` with `ignoreBuildErrors: false` — must pass.

**Production go-live**

- [ ] Domain verified in Resend; receipt email passes SPF/DKIM.
- [ ] InfinPay production credentials in Vercel environment, NOT in git.
- [ ] One real low-value booking (RM 1) end-to-end with the client present. Verify settlement appears in SHEH bank account on T+1/T+2.
- [ ] Refund test (manual via InfinPay portal) — verify our system reflects it (Phase 2 may automate).
- [ ] Cloudflare in front of Vercel, WAF rules tuned for travel.

---

## Things Blocked Until InfinPay Docs Arrive

These cannot be finalized without the docs — flagged here so the PR backlog is visible:

1. Exact request/response shape of InfinPay's `createPayment` endpoint.
2. Amount unit (ringgit vs. sen).
3. Signature algorithm and which fields are signed.
4. Header name carrying the callback signature.
5. Success-status string convention (`"00"`, `"SUCCESS"`, etc.).
6. Required response body for our callback handler.
7. Whether Level 2/3 corporate-card data is supported and how to pass it.
8. Sandbox base URL and test card numbers.
9. Production base URL and the InfinPay domain to whitelist in CSP.
10. Whether they support the merchant being a travel agency (MCC 4722, often high-risk) without imposing rolling reserves.

When the docs arrive, all the above slots into `lib/payments/infinpay.ts` only — no other file should need changes.
