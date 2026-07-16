# RupeeReceipt

Photograph a receipt (or forward it on WhatsApp), get an itemized expense entry with GST split and a Tally-ready CSV export — no manual data entry.

## The problem

Indian freelancers and small businesses collect paper/digital receipts all month and then spend hours re-typing them into a spreadsheet or Tally before filing. RupeeReceipt reads the receipt directly and produces the itemized, GST-split entry.

## Target buyer

Indian freelancers and small businesses who file their own books or hand a CSV to their accountant/Tally operator monthly.

## Pricing hypothesis

Rs299/month subscription, capped monthly receipt quota (well above typical single-freelancer volume), manual UPI renewal at launch.

Status: planned — not yet built (50-SaaS challenge #29)

## Stack

Cloudflare Worker (TypeScript, ESM) for API + static frontend, single synchronous vision LLM call per receipt (no VPS needed). Supabase for auth/Postgres/RLS and private receipt-image storage. WhatsApp ingest via Meta Cloud API webhook as a second entry point alongside direct web upload.

## How to continue this build

Nothing is built yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered, TDD task list, and execute tasks in order. `CLAUDE.md` points at the private reference implementation for the shared Worker+Supabase+credits conventions this product reuses.

## Risks / constraints

- **Receipt reading uses LLM vision natively — no OCR library.** This locks the extraction call to **`claude-sonnet-4-6` (Anthropic) specifically**, not the usual provider-switchable layer: DeepSeek's `deepseek-chat` is text-only and cannot read an image receipt, so it is not a valid fallback for this step.
- **WhatsApp ingest via Meta Cloud API is the v2/business tier and needs real webhook plumbing** (signature verification, phone-number-to-account linking, media download via the Graph API) plus Meta Business/app verification — **this specifically blocks the WhatsApp channel at launch**, not the product as a whole. Web upload can and should ship independently of WhatsApp being approved.
- **GST split is computed by the LLM in the prompt but never trusted blindly** — server-side validation re-checks that item totals reconcile with the receipt total and that GST amounts match standard slabs (0/5/12/18/28%) before a receipt is marked clean; mismatches are flagged, not silently accepted.
- This is bookkeeping assistance, not tax or accounting advice — the user (or their accountant) is responsible for final filing accuracy.
