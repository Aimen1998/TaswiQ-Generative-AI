# TASKS.md — TaswiQ Studio MVP

> One task per Claude Code session. Prompt to use every time:
> **"Read CLAUDE.md, then do the next unchecked task in docs/TASKS.md."**
> Each task: implement → lint/typecheck → commit → tick here → note in docs/PROGRESS.md.

## Phase 0 — Setup & Day-0 test (Day 1)
- [ ] **T0.1 Repo map** — Install deps, run the fork locally. Write `docs/ARCHITECTURE.md`: real folder structure, where MuAPI is called (client/server), UI components worth reusing, build/run commands. Adapt CLAUDE.md §5 paths if needed.
  - Done when: app runs on localhost and ARCHITECTURE.md exists.
- [ ] **T0.2 Day-0 model test script** — `scripts/day0-test.ts`: for 5 images in `scripts/samples/`, call MuAPI image-to-video (Seedance Pro Fast 480p, Wan 2.2, H3 Max Turbo) and lip-sync (InfiniteTalk 480p with one avatar + one TTS audio). Save outputs + a CSV (model, seconds, latency, cost reported).
  - Done when: CSV + videos exist; user picks the `standard` models. Update `config/tiers.ts` values.

## Phase 1 — Foundation (Days 2–3)
- [ ] **T1.1 Supabase schema** — Migrations for all tables in CLAUDE.md §8 + RLS + storage buckets. Seed: 3 music tracks, 5 avatars placeholders, templates.
- [ ] **T1.2 Auth** — Supabase email login (phone OTP later), `profiles` auto-create trigger, protected `/studio` and `/admin` routes, Arabic RTL layout shell.
- [ ] **T1.3 Server-side providers** — Create `lib/providers/{types,muapi,registry}.ts`. Remove/disable any client-side MuAPI key usage from the fork. All AI calls via API routes/workers only.
- [ ] **T1.4 Credits** — `lib/credits.ts` (balance, reserve, commit, refund; transactional, idempotent by ref_id) + free-trial grant on signup + `/studio/credits` page showing balance and ledger.
- [ ] **T1.5 Cost logging** — `lib/costs.ts` writes `generation_jobs` for every provider/LLM/TTS call.

## Phase 2 — Ad builder & preview (Days 4–6)
- [ ] **T2.1 Products** — Upload 1–5 images to `product-images`, create/edit product (name, price DZD, description).
- [ ] **T2.2 Script API** — `/api/script` with Gemini Flash-Lite vision → strict JSON (hook, headline, script, cta, subtitles_text) for darija/ar/fr. Few-shot examples of Algerian COD ad copy (livraison 58 wilayas, paiement à la livraison / الدفع عند الاستلام).
- [ ] **T2.3 TTS API** — `/api/tts` Azure ar-DZ + fr-FR (1 male, 1 female each), word-boundary timings, cache by hash, store in `audio`.
- [ ] **T2.4 Remotion ProductAd** — Composition 15 s: hook text, product images with motion, price badge DZD, subtitles synced to timings, CTA end card, optional AI-clip slot, watermark prop. RTL test string from CLAUDE.md §10 must render correctly.
- [ ] **T2.5 Remotion UgcAd** — Composition 20 s: avatar slot (video or still), product cut-ins, subtitles, price, CTA, watermark.
- [ ] **T2.6 Builder UI** — `/studio/ads/new` wizard (type → product → language/voice/avatar → script edit) and `/studio/ads/[id]` editor with live `@remotion/player` preview. Editing never calls AI.

## Phase 3 — Generation & render (Days 7–8)
- [ ] **T3.1 Queue + AI worker** — BullMQ `ai-jobs`; worker calls registry by tier (i2v or lipsync), polls result, saves to `ai-assets`, logs cost, commits or refunds credits, 1 auto-retry.
- [ ] **T3.2 Render worker** — BullMQ `render-jobs`; Remotion render to MP4 1080×1920 (720p + watermark for free trial), upload to `renders`, set status `done`, signed download URL.
- [ ] **T3.3 Generate flow** — "Generate" button: balance check → reserve → enqueue; progress states in UI; re-render on edit without AI.
- [ ] **T3.4 Library** — `/studio/library` list of ads with status, preview, download, duplicate.

## Phase 4 — Launch (Days 9–10)
- [ ] **T4.1 Admin** — `/admin`: users, add credits manually (ledger `admin`), jobs list with cost, total cost per day.
- [ ] **T4.2 Content safety + legal** — Input/prompt checks (no faces of real people uploaded as avatars, blocked words list), Terms of Use + AI-content notice pages (Arabic/French).
- [ ] **T4.3 Docker + deploy** — Docker Compose (app, workers, Redis, Caddy). Deploy to VPS, DNS `studio.taswiq.store` via Cloudflare, HTTPS, env vars. Step-by-step commands for the user, one at a time.
- [ ] **T4.4 Landing + pricing** — Arabic landing page with 3 example ads, free-trial CTA, credit packs.
- [ ] **T4.5 Smoke test** — Full run of the 3 ad types on production with a new account. Fix blockers only.

## Phase 5 — Right after launch
- [ ] **T5.1 Chargily Pay** — Checkout + signed webhook → idempotent ledger `topup`; credits packs page.
- [ ] **T5.2 Phone OTP** for free trial abuse protection.
- [ ] **T5.3 TaswiQ import** — `/import?token=` HMAC-signed product import + button snippet for store dashboards.
- [ ] **T5.4 Second provider** (fal or Atlas/kie) in registry + per-tier switch in config.
- [ ] **T5.5 Enable `pro`/`premium` tiers**, more templates, 10–20 avatars.
