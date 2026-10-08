# TASKS.md — TaswiQ Studio MVP (NEXT version)

> One task per Claude Code session. Prompt to use every time:
> **"Read CLAUDE.md, then do the next unchecked task in docs/TASKS.md."**
> Each task: implement → lint/typecheck/tests (read result) → commit → tick here → note in docs/PROGRESS.md.
> **USER** = the owner must do something first/alongside; Claude must not fake keys or continue past a missing prerequisite.
> Estimates are per session (~2–4 h of agent work), not measured. Timeline: private beta ≈ end of Phase 4 (~3 weeks), public launch ≈ end of T5.1 (~4–5 weeks).

## Preparation checklist (what the owner must have, and when)
| ID | What | Needed by | Notes |
|---|---|---|---|
| U0.1 | **USD payment route to providers** (card/virtual card/crypto) tested with a $25 Atlas top-up | before T0.3 | Atlas min top-up $25; accepts card, Alipay/WeChat, crypto. Confirm kie.ai methods. |
| U0.2 | Accounts: Atlas Cloud, kie.ai, Google AI Studio (Gemini key), Azure Speech (Free F0 ok to test), Supabase project (EU region), Cloudflare (DNS + Turnstile) | before T0.3 / T1.1 | Keys go to `.env.local`, never to chat. |
| U0.3 | New GitHub repo `taswiq-studio` (private until launch, then public — AGPL) | before T0.1 | |
| U0.4 | **Chargily Pay merchant account created** (start now; verification may take days) | day 1, needed by T5.1 | Ask what documents they require. |
| U0.5 | Day-0 assets: ≥ 10 real product photos (5 "hard": logo text, transparent PNG, human model, dark, low-res) + 3 sample product texts in Darija/French | before T0.3 | |
| U0.6 | A native listener (you) available for the Darija voice + copy test | T0.3 | 30 min |
| U1.1 | VPS (4 vCPU/8 GB, Ubuntu 24.04, root SSH) and DNS `studio.taswiq.store` → VPS | before T4.3 (earlier if you want staging) | |
| U2.1 | 8–10 avatar faces (generated, approved by you; M/F, ages, Algerian look) | before T2.5 | Claude can generate prompts; generation via Atlas image model, ~$0.04 each. |
| U2.2 | 3 music tracks with a commercial licence + licence text | before T2.4 | |
| U2.3 | Brand: logo, colours, Arabic font preference (default Cairo) | before T2.4 | |
| U4.1 | Terms of Use/Privacy review by a lawyer (Algerian law 18-05, 18-07; AI labelling) | before public launch | Not legal advice. |
| U4.2 | Credit→DZD price list and pack sizes | before T4.4 | Defaults in CLAUDE.md §7 |
| U4.3 | 3 sample ads for the landing page (made with the product itself) | before T4.4 | |
| U5.1 | OTP provider decision (WhatsApp Cloud API / SMS) | before T5.2 | |

## Phase 0 — Setup & Day-0 test (days 1–3)
- [ ] **T0.1 New repo** — USER U0.3 done. Create `taswiq-studio`: Next.js 16 + TS + Tailwind 4 + shadcn + `next-intl` (ar default, RTL, fr), ESLint, Vitest, AGPL-3.0 `LICENSE`, `NOTICE`, copy `CLAUDE.md`/`TASKS.md`, empty `docs/ARCHITECTURE.md` + `PROGRESS.md`, `.env.example`, GitHub Actions (lint+typecheck+test).
  - Done when: `npm run dev` shows an Arabic RTL hello page, CI green, no secrets committed.
- [ ] **T0.2 Price + model catalogue script** — `scripts/check-prices.ts` reads Atlas `GET https://api.atlascloud.ai/api/v1/models` and MuAPI `GET https://api.muapi.ai/api/v1/models` (public) and prints the models in `config/tiers.ts` with list price vs `config/prices.ts`. Create `config/prices.ts` with the 2026-10-08 values from BASE_COMPARISON §5.
  - Done when: script outputs a diff table; unknown model ids are flagged.
- [ ] **T0.3 Day-0 provider test** — USER U0.1, U0.2, U0.5 done. `scripts/day0/*`: for the 10 images call (a) Atlas Seedance 1.0 Pro Fast 480p 5 s and 720p, (b) Atlas Wan 2.2 turbo, (c) kie.ai InfiniteTalk 480p and 720p with one avatar image + one 8 s Azure TTS clip, (d) Atlas InfiniteTalk 480p; 3 repeats of the cheapest path to measure failure rate and latency. Save videos + `day0.csv` (model, provider, seconds, latency, reported cost, billed cost from dashboard, error). Check: 9:16 respected, logo/text warping, face-rejection errors, 15 s cap on kie.
  - Done when: CSV + videos exist, the owner has chosen the `standard` models, and `config/tiers.ts` is updated. Actual billed amounts compared with BASE_COMPARISON.
- [ ] **T0.4 Voice + copy test** — USER U0.6. Azure `ar-DZ` Amina/Ismael, `ar-SA`/other candidate, a Gemini TTS option, and French voices read 5 Darija + 5 French scripts; Gemini Flash-Lite vs Flash write the copy for 10 products. Produce an HTML page with players and the generated text.
  - Done when: the owner picks one male + one female voice per language and the copy model; word-boundary events confirmed for Arabic (or fallback alignment chosen); `config` updated.

## Phase 1 — Foundation (days 4–6)
- [ ] **T1.1 Supabase schema** — Migrations for all tables in CLAUDE.md §8 + RLS + buckets + seed (3 placeholder music rows, avatar rows pointing to U2.1 files when ready, templates).
  - Done when: `supabase db reset` applies cleanly; RLS tests show user A cannot read user B.
- [ ] **T1.2 Auth + shell** — Supabase email auth, Turnstile on signup, profile trigger, protected `/studio` and `/admin`, RTL layout shell with Arabic/French switcher.
  - Done when: signup/login works; admin route blocked for non-admins; Playwright smoke passes in `ar` and `fr`.
- [ ] **T1.3 Credits as SQL functions** — `reserve_credits`, `commit_credits`, `refund_credits`, `grant_credits` (idempotent on `ref_id`+reason, row-locked), `lib/credits.ts` wrapper, `/studio/credits` page (balance + ledger).
  - Done when: concurrency test (50 parallel reserves) never overdraws and never double-refunds.
- [ ] **T1.4 Provider layer** — `lib/providers/{types,registry,circuit-breaker,atlas,kie}.ts` (+ disabled stubs for muapi/fal/byteplus), timeouts, expected-cost calculation from `config/prices.ts`, error → `retryable`. Port `circuit-breaker` and `ssrf-guard` from ClipForge (NOTICE entry).
  - Done when: unit tests with mocked HTTP cover success, failure, timeout, fallback, breaker open; no key reachable from client bundle (grep test).
- [ ] **T1.5 Cost logging + budgets** — `lib/costs.ts` writes `generation_jobs` for every provider/LLM/TTS/render call; `settings` budgets; guard function used by generate flow.
  - Done when: daily budget reached ⇒ trials rejected first, then all AI, with a clear Arabic message.

## Phase 2 — Ad builder & preview (days 7–12)
- [ ] **T2.1 Products** — Upload 1–5 images with validation (size, min 600 px, blur check), create/edit product (name, price DZD, description). Port `product-ingest` shape for later import.
  - Done when: bad images give friendly messages; images stored in `product-images`.
- [ ] **T2.2 Script API** — `/api/script`: Gemini vision → strict JSON for darija/ar/fr with few-shot Algerian COD copy (livraison 58 wilayas, الدفع عند الاستلام); banned-term scan (port `ad-compliance` rules, AR/FR); length control for the audio cap.
  - Done when: 10 sample products give valid JSON in all 3 languages; invalid JSON auto-retries once.
- [ ] **T2.3 TTS API** — `/api/tts`: Azure with chosen voices, word timings, cache by hash, bucket `audio`.
  - Done when: second identical call is a cache hit; timings reach the subtitle component.
- [ ] **T2.4 Remotion ProductAd** — USER U2.2, U2.3 done. 15 s composition: hook, product images with motion, price badge, synced subtitles, CTA end card, AI-clip slot, watermark prop; local OFL fonts. Render benchmark on the dev machine and VPS spec noted.
  - Done when: RTL test string renders correctly in Player and in `renderMedia`; measured seconds/render recorded in `docs/PROGRESS.md`.
- [ ] **T2.5 Remotion UgcAd** — USER U2.1 done. 20 s composition: framed avatar window (video or still), product cut-ins, subtitles, price, CTA, watermark.
  - Done when: preview and render match; avatar window stays crisp at 480p source.
- [ ] **T2.6 Builder UI** — `/studio/ads/new` wizard and `/studio/ads/[id]` editor with live `@remotion/player` preview; editing never calls AI; clear cost display in credits (never model names).
  - Done when: all 3 ad types can be previewed end-to-end in Arabic RTL on a phone-sized viewport.

## Phase 3 — Generation & render (days 13–16)
- [ ] **T3.1 Queue + AI worker** — BullMQ `ai-jobs`; worker uses registry by tier (i2v or lipsync, splitting audio > provider cap), polls, stores to `ai-assets`, logs expected vs billed cost, commit/refund, 1 auto-retry, resume after worker restart.
  - Done when: integration test with a fake provider covers success, failure refund, retry, worker crash recovery.
- [ ] **T3.2 Render worker** — BullMQ `render-jobs`; Remotion MP4 1080×1920 (720p + watermark for trial), upload to `renders`, signed download URL, concurrency 1, queue position.
  - Done when: 3 simultaneous requests finish sequentially without OOM on the target VPS spec.
- [ ] **T3.3 Generate flow** — Generate button: balance + budget guard → reserve → enqueue; progress UI; paid regenerate (max 2) of the AI part; re-render on edit without AI.
  - Done when: editing text after generation causes no `generation_jobs` rows of kind i2v/lipsync.
- [ ] **T3.4 Library** — `/studio/library`: status, preview, download, duplicate, delete; 30-day retention job.
  - Done when: expired renders are removed and the UI explains it.

## Phase 4 — Private beta (days 17–21)
- [ ] **T4.1 Admin** — users, manual credits (ledger `admin`), jobs with expected vs billed cost, daily cost per ad type, provider/tier switches, budget settings, trial abuse view (signups per IP hash).
  - Done when: owner can top-up a user and pause trials from the UI.
- [ ] **T4.2 Safety + legal pages** — input/prompt checks, blocked terms, AI-label notice, ToS/Privacy pages (AR/FR drafts) — USER U4.1 for review before public launch.
  - Done when: blocked-term test suite passes; pages linked from signup.
- [ ] **T4.3 Docker + deploy** — USER U1.1 done. Docker Compose (app, ai-worker, render-worker, Redis, Caddy), Supabase prod project, env vars, DNS `studio.taswiq.store`, HTTPS, nightly DB backup, basic uptime check. Step-by-step commands one at a time.
  - Done when: `https://studio.taswiq.store` serves the app and a worker job completes on the VPS.
- [ ] **T4.4 Landing + pricing** — USER U4.2, U4.3 done. Arabic/French landing with 3 example ads, free-trial CTA, credit packs (manual top-up instructions for now).
  - Done when: Lighthouse mobile ≥ 80; RTL verified.
- [ ] **T4.5 Smoke test + beta** — Full run of the 3 ad types on production with a new account; invite 5–10 friendly merchants; collect cost per ad vs BASE_COMPARISON §5.4.
  - Done when: measured cost per ad type is within +30% of the estimate (otherwise open a cost-fix task).

## Phase 5 — Public launch
- [ ] **T5.1 Chargily Pay** — USER U0.4 done. Checkout + signed webhook → idempotent ledger `topup`; credit packs page; fee-aware pricing; test mode then live.
  - Done when: test payment credits the account exactly once even if the webhook is replayed.
- [ ] **T5.2 Phone OTP** — USER U5.1. OTP required before first AI generation; one trial per verified phone.
- [ ] **T5.3 TaswiQ import** — `/import?token=` HMAC-signed product import + button snippet for CodFlow store dashboards.
- [ ] **T5.4 Second provider live** — fal or MuAPI adapter enabled behind the registry with per-tier switch; price check job in CI/cron.
- [ ] **T5.5 Enable `pro`/`premium`**, more templates, 10–20 avatars, background removal (rembg) as an optional step.

## Phase 6 — Later (only when numbers justify)
- [ ] **T6.1 GPU measurement** — rent one 24 GB GPU for a day, run InfiniteTalk 480p + Wan 2.2, measure seconds of compute per output second and cost; compare with API prices; decide by the rule in BASE_COMPARISON §5.8 (monthly AI spend > ~$300 and utilisation > ~50%). Check weights licences first.
- [ ] **T6.2 Self-hosted provider adapter** — same `LipSyncProvider`/`VideoProvider` interface, behind the registry.
