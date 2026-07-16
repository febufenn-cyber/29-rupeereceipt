# RupeeReceipt — Build Plan

TDD, one task per focused session. Files touched, interfaces produced, test to write first, done-criteria.

## T1 — Repo scaffold + Supabase schema

Files: `package.json`, `tsconfig.json`, `wrangler.toml`, `supabase/migrations/0001_init.sql`, `.dev.vars.example`, `.gitignore`.
Interfaces: none (schema only) — `profiles`/`receipts` tables, RLS, `spend_credit`/`refund_credit`, `receipts` private storage bucket, exactly per LLD.
Done: `supabase db push` applies cleanly; `wrangler dev` boots.

## T2 — Supabase client + receipt types (`src/supa.ts`, `src/receipt.ts`)

Files: `src/supa.ts`, `src/receipt.ts`, `test/supa.test.ts`.
Interfaces: `export interface Receipt { id; userId; source; status; items; gstBreakup; grandTotal; validationWarnings; ... }`; `insertReceipt(...)`, `getReceipt(id)`, `listReceipts(userId, opts?)`, `updateReceipt(id, patch)`, `getProfileByPhone(phone)`, `spendCredit(userId)`, `refundCredit(userId)`.
Test first: mock `fetch`; assert `getProfileByPhone` queries the right filter and `spendCredit` handles insufficient-balance.
Done: `vitest run test/supa.test.ts` green, no live network calls.

## T3 — Vision extraction provider (`src/providers/anthropic.ts`)

Files: `src/providers/anthropic.ts`, `src/prompt/system.ts`, `test/providers/anthropic.test.ts`.
Interfaces: `extractReceipt(opts: { imageBytes: ArrayBuffer; mimeType: string; apiKey: string; fetchFn?: typeof fetch }): Promise<{ report: ExtractedReceipt; model: "claude-sonnet-4-6"; inputTokens; outputTokens }>` — image sent as base64 in the Anthropic `image` content block, no other provider implements this function.
Test first: mock `fetch` with a canned JSON response; assert parsed shape; add a malformed-JSON case to exercise the repair fallback.
Done: unit tests green including the repair-fallback path.

## T4 — GST validation (`src/gst.ts`)

Files: `src/gst.ts`, `test/gst.test.ts`.
Interfaces: `validateGst(extracted: ExtractedReceipt): { warnings: string[] }` — pure function, no I/O.
Test first: table-test a clean receipt (no warnings), a total-mismatch receipt, a non-standard-GST-rate receipt, and an unbalanced CGST/SGST receipt.
Done: unit tests green for all four cases.

## T5 — Tally CSV export (`src/csv.ts`)

Files: `src/csv.ts`, `test/csv.test.ts`.
Interfaces: `receiptsToTallyCsv(receipts: Receipt[]): string` — column layout per LLD (Date, Party, Item, Qty, Rate, Taxable Value, CGST, SGST, IGST, Total); validate the exact template against a real Tally import before marking this task done.
Test first: given 2 fixture receipts, assert the CSV header row and one data row match exactly.
Done: unit test green; a manual Tally import test with the generated CSV succeeds (or the template is corrected and the test updated).

## T6 — Web upload route handlers (`src/handlers.ts`, `src/router.ts`)

Files: `src/handlers.ts`, `src/router.ts`, `test/router.test.ts`, `test/handlers.test.ts`.
Interfaces: `route(req, deps): Promise<Response|null>`; `handleCreateReceipt` (spend → extract → validate → store, refund on extraction failure), `handleListReceipts`, `handleGetReceipt`, `handleExportCsv`, `handleMe`.
Test first: mock an extraction failure after credit spend; assert refund called and `status='failed'`.
Done: `vitest run` green; every failure mode in the LLD's API table (web routes) has a test.

## T7 — WhatsApp webhook handlers (`src/handlers.ts` — `handleWhatsappVerify`, `handleWhatsappMessage`)

Files: `src/handlers.ts`, `src/providers/whatsapp.ts`, `test/handlers.whatsapp.test.ts`.
Interfaces: `verifyMetaSignature(req, appSecret): boolean`; `downloadWhatsappMedia(mediaId, accessToken, fetchFn?): Promise<ArrayBuffer>`; `handleWhatsappMessage` resolves phone → user, reuses the same extract/validate/store path as `handleCreateReceipt`.
Test first: mock a bad signature (assert 401, nothing processed) and an unrecognized phone number (assert 404-equivalent, logged, no crash).
Done: unit tests green for signature verification and phone-lookup branches; live webhook flow deferred until Meta app is provisioned (see T9).

## T8 — Worker entrypoint (`src/worker.ts`)

Files: `src/worker.ts`.
Interfaces: default export `{ fetch(req, env, ctx) }` wiring `route()` to real bindings, plus a `scheduled` handler for the monthly credit-reset cron.
Done: `wrangler dev` serves `/config.js` and rejects unauthenticated `/api/receipts` with 401.

## T9 — Frontend (`public/*.html`)

Files: `public/index.html`, `public/app.html`, `public/receipt.html`, `public/settings.html`, `public/pricing.html`.
Interfaces: none (static, supabase-js CDN + fetch to `/api/*`); `settings.html` includes the WhatsApp phone-linking form.
Done: manual click-through against `wrangler dev` completes upload → extracted receipt → CSV export.

## T10 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, production `wrangler.toml` vars, secrets via `wrangler secret put`.
Interfaces: `scripts/smoke.ts` uploads a fixture receipt image against the deployed Worker, asserts `status` is `done`/`done_with_warnings` and the CSV export endpoint returns a non-empty body.
Done: `wrangler deploy` succeeds; `npm run smoke` passes; launch checklist confirmed — web upload ships even if WhatsApp isn't yet approved by Meta (flag that channel as "coming soon" in the UI until it is), pricing page shows Rs299/mo, `ANTHROPIC_API_KEY` and `SUPABASE_SERVICE_ROLE_KEY` set via `wrangler secret put`, WhatsApp secrets (`META_APP_SECRET`, `META_ACCESS_TOKEN`, `META_VERIFY_TOKEN`) set only once the Meta app is live, no `.dev.vars` committed.
