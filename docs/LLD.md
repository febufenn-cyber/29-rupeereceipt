# RupeeReceipt — Low-Level Design

Adapts the shipped contract-reviewer's Worker+Supabase+credits architecture. Two differences: extraction is vision-only (Anthropic-locked, not provider-switchable), and there are two ingest paths (web upload, WhatsApp).

## Architecture

```
Browser (static frontend, Workers Assets)          WhatsApp (Meta Cloud API)
  │ POST /api/receipts (multipart image)             │ POST /api/whatsapp/webhook
  ▼                                                   ▼
Cloudflare Worker (TypeScript, ESM)
  ├─ /config.js
  ├─ /api/receipts          (POST) → JWT, spend_credit, vision extraction (sync),
  │                                   GST validation, store
  ├─ /api/receipts          (GET)  → list caller's receipts
  ├─ /api/receipts/:id      (GET)  → single receipt, RLS-scoped
  ├─ /api/export/csv        (GET)  → Tally-ready CSV for a given month, JWT
  ├─ /api/whatsapp/webhook  (GET)  → Meta webhook verification (hub.challenge)
  ├─ /api/whatsapp/webhook  (POST) → Meta signature verify, phone→user lookup,
  │                                   download media, spend_credit, same
  │                                   extraction path as web upload
  └─ /api/me                (GET)  → profile + credits remaining this cycle
  │
  ├─ Supabase (plain fetch, service-role key): GoTrue JWT verify, PostgREST
  │     for receipts/profiles, private Storage bucket for receipt images
  │
  └─ Anthropic claude-sonnet-4-6 (vision) — the ONLY extraction provider
```

Request flow (web): user uploads a receipt photo/PDF page → Worker verifies JWT → `spend_credit(user_id)` → image sent natively to `claude-sonnet-4-6` with the extraction prompt → strict JSON parsed → server-side GST reconciliation → receipt row `status='done'` (or `'done_with_warnings'` if reconciliation flags a mismatch); any extraction failure → `refund_credit`, `status='failed'`.

Request flow (WhatsApp): Meta POSTs the incoming message webhook → Worker verifies the Meta signature → resolves the sender's phone number to a `profiles.whatsapp_phone` row → downloads the image via the Graph API using the page access token → same spend/extract/validate/store path as web upload → optionally replies with a short WhatsApp confirmation message.

Because extraction is one vision call and validation is pure arithmetic — no chunking, no subprocess — this is a **synchronous** route, same reasoning as the reference's default sync path; no VPS/async pattern is needed for v1.

## Data model

```sql
create table public.profiles (
  user_id uuid primary key references auth.users(id) on delete cascade,
  credits int not null default 5,             -- free-trial receipts
  plan_renews_at timestamptz not null default (now() + interval '30 days'),
  whatsapp_phone text unique,                  -- E.164, nullable until linked
  created_at timestamptz not null default now()
);
alter table public.profiles enable row level security;
create policy "read own profile" on public.profiles for select using (auth.uid() = user_id);

create table public.receipts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  source text not null check (source in ('web','whatsapp')),
  storage_path text not null,
  status text not null default 'processing'
    check (status in ('processing','done','done_with_warnings','failed')),
  vendor text,
  receipt_date date,
  items jsonb,
  gst_breakup jsonb,      -- {cgst, sgst, igst}
  grand_total numeric,
  validation_warnings jsonb,
  model text,
  error text,
  created_at timestamptz not null default now()
);
alter table public.receipts enable row level security;
create policy "read own receipts" on public.receipts for select using (auth.uid() = user_id);
create index receipts_user_created on public.receipts (user_id, created_at desc);

-- Reused verbatim from the reference (flat -1 spend per receipt; monthly quota
-- reset is a separate cron, not a new RPC):
create or replace function public.spend_credit(p_user uuid) returns boolean
language plpgsql security definer set search_path = public as $$
begin
  update profiles set credits = credits - 1 where user_id = p_user and credits > 0;
  return found;
end $$;

create or replace function public.refund_credit(p_user uuid) returns void
language plpgsql security definer set search_path = public as $$
begin
  update profiles set credits = credits + 1 where user_id = p_user;
end $$;
revoke execute on function public.spend_credit(uuid) from public, anon, authenticated;
revoke execute on function public.refund_credit(uuid) from public, anon, authenticated;
grant execute on function public.spend_credit(uuid), public.refund_credit(uuid) to service_role;

insert into storage.buckets (id, name, public) values ('receipts', 'receipts', false)
on conflict (id) do nothing;
```

State machine: `processing → done | done_with_warnings | failed`. A monthly cron resets `credits` to the plan quota (e.g. 300) and advances `plan_renews_at` for accounts past their renewal date — mirrors the reference's stale-sweep cron mechanism but resets rather than refunds.

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/receipts` | POST | JWT | Spend credit, vision-extract, validate, store | 401 no JWT; 402 no credits; 500 extraction error → refund + `status=failed` |
| `/api/receipts` | GET | JWT | List caller's receipts | 401 no JWT |
| `/api/receipts/:id` | GET | JWT | Single receipt, RLS-scoped | 401 no JWT; 404 not found/not owner |
| `/api/export/csv` | GET | JWT | Tally-ready CSV for `?month=YYYY-MM` | 401 no JWT; empty CSV (not an error) if no receipts that month |
| `/api/whatsapp/webhook` | GET | Meta verify token | Echoes `hub.challenge` | 403 bad verify token |
| `/api/whatsapp/webhook` | POST | Meta signature | Same pipeline as `/api/receipts` | 401 bad signature; 404 unrecognized phone number (message ignored, logged) |
| `/api/me` | GET | JWT | Profile + credits + `plan_renews_at` | 401 no JWT |

## LLM strategy

**Provider: `claude-sonnet-4-6` only — not provider-switchable for this step.** Receipt extraction reads an image natively; `deepseek-chat` is text-only in this fleet's usage (it consumes unpdf-extracted text elsewhere), so it cannot serve as a fallback for the vision call. If a cheaper/faster path is ever needed, that would mean adding a *different* vision-capable provider, not swapping in DeepSeek.

Prompt strategy: system prompt instructs the model to read the receipt image and extract vendor, date, line items (description, quantity, unit price, taxable value), GST rate and CGST/SGST/IGST breakup per item where shown, payment method, and GSTIN if visible — flagging any field it can't read as `null` rather than guessing. Output forced to strict JSON, no raw quotes inside string values (same rule as the reference), with a repair-fallback retry on malformed JSON:

```json
{
  "vendor": "string|null",
  "receipt_date": "YYYY-MM-DD|null",
  "gstin": "string|null",
  "items": [{ "description":"string","quantity":number,"unit_price":number,
              "taxable_value":number,"gst_rate":number,
              "cgst":number,"sgst":number,"igst":number,"line_total":number }],
  "subtotal": number, "total_cgst": number, "total_sgst": number,
  "total_igst": number, "grand_total": number, "payment_method": "string|null"
}
```

Server-side GST validation (not LLM-trusted): reconcile `sum(line_total) ≈ grand_total` within a rounding tolerance, confirm each `gst_rate` is a standard slab (0/5/12/18/28%), and confirm `cgst ≈ sgst` for intra-state-looking entries. Any failed check sets `status='done_with_warnings'` and populates `validation_warnings` — the extraction is still returned, just flagged for the user to double-check.

Cost per operation: a single receipt image (~1.5k input tokens incl. image, ~500 output tokens) on `claude-sonnet-4-6` is roughly Rs2-3 at current API pricing. At Rs299/month for a 300-receipt quota, worst-case cost is well under the subscription price.

## Frontend pages

`index.html` (pitch + pricing), `app.html` (upload + receipt list + monthly export button), `receipt.html` (single receipt detail, warnings shown inline), `settings.html` (WhatsApp phone linking), `pricing.html`.

## Error handling / credits flow

Credit is spent before the vision call (after auth, before extraction) and refunded on any extraction failure (API error, malformed JSON that survives the repair fallback). A validation-warning result is not a failure — it still consumes the credit, since extraction succeeded; only a hard failure to produce usable JSON triggers a refund. No async sweep is needed (synchronous request/response); the monthly quota-reset cron is the only scheduled job.

## Integrations and launch gates

**Web upload has no external launch gate** beyond the standard Supabase/Cloudflare setup — it can ship first. **WhatsApp ingest is gated on Meta**: a WhatsApp Business Platform app, Business verification, a registered phone number, and webhook URL verification are all required before the channel works, and Meta's review/verification timeline is outside this project's control — treat WhatsApp as a fast-follow, not a v1 blocker. Razorpay subscription billing is a post-KYC gate; manual UPI renewal ships first.

## Security notes

RLS restricts `receipts`/`profiles` reads to the owner; all writes go through the Worker's service-role key. `spend_credit`/`refund_credit` are SECURITY DEFINER, revoked from `anon`/`authenticated`. The WhatsApp webhook validates Meta's `X-Hub-Signature-256` on every POST before processing; the phone-to-user mapping requires the user to have explicitly linked their number in-app (no auto-linking from an inbound message alone). Receipt images live in a private Storage bucket; the frontend never gets a public URL, only the Worker-mediated extracted data. Uploaded file type/size is validated before it's sent to the LLM.

## Out of scope for v1

Automatic recurring Razorpay billing (manual UPI renewal only); multi-currency receipts; expense categorization/tagging beyond raw line items; direct Tally XML voucher import (CSV only); receipt editing after extraction (re-upload to correct, no inline edit UI); team/multi-user accounts; OCR fallback for non-LLM-readable images (blurry/rotated receipts are flagged low-confidence, not auto-corrected).
