# CLAUDE.md — TaswiQ Studio (NEXT version, replaces CLAUDE.md after the owner approves)

> Read automatically by Claude Code at the start of every session. Single source of truth.
> Do NOT ask the user to re-explain anything written here. Evidence and prices: `docs/BASE_COMPARISON.md`.
> This file goes into the **new repo `taswiq-studio`** (see §3). Copy it there as `CLAUDE.md`, and `TASKS.next.md` as `docs/TASKS.md`.

## 0. How to work with me (mandatory)
- Reply to the user in **Arabic**, very concise and practical. Code, comments, commit messages: English.
- Work on **one task from `docs/TASKS.md` per session**. Do not start the next task unless asked.
- Before coding a task: read only the files needed. Do not re-scan the repo; use `docs/ARCHITECTURE.md`.
- When the user must run something manually (accounts, keys, deploy): give **one command at a time** + one line on what it verifies, then wait. Tasks marked **USER** in TASKS need the user first; do not fake keys or skip them.
- **Never** run destructive commands (rm -rf, drop table, force push, deleting migrations) without explicit permission.
- Run tests/lint/typecheck and **read the result before committing** (separate steps). Stage specific files only. Then small commit, tick the task, append 2–4 lines to `docs/PROGRESS.md`.
- If a decision is not covered here, choose the simplest option that keeps the provider layer replaceable and record it in `docs/PROGRESS.md`.
- Never put API keys in client code. Never commit `.env*` files. Never paste keys into chat; the user puts them in `.env.local` / VPS env.
- Prices change: provider prices live only in `config/tiers.ts` and `config/prices.ts`, each with `source` and `checked_at`. Do not hard-code prices elsewhere.

## 1. Product
**TaswiQ Studio** (`studio.taswiq.store`): self-service platform where Algerian e-commerce merchants turn a product image into a ready-to-publish vertical ad video (9:16).

Owner: Aimen. Parent business: TaswiQ (taswiq.store) builds COD stores on CodFlow; each merchant store is a separate deployment.

### Ad types (all three in the MVP)
1. **Product Ad** — product images + animated template + headline + price in DZD + CTA + voice-over + subtitles. No AI video. (~$0.023)
2. **Product Ad + AI shot** — same with one AI-generated 3–5 s clip (image-to-video from the product image). (~$0.08)
3. **UGC Ad** — a TaswiQ avatar (AI-generated face) talks about the product (lip-sync 8–10 s) + product shots + price + subtitles + CTA. (~$0.20)

### Languages
Algerian Darija (Arabic script), Modern Standard Arabic, French. UI Arabic (RTL) first, French second.

## 2. Non-negotiable principles
1. **AI only where it adds value.** AI video limited to short clips (≤ 5 s product shot, ≤ 10 s UGC speech). Everything else is deterministic rendering (Remotion + FFmpeg).
2. **Editing never re-triggers AI.** Text, price, CTA, colours, music, voice, template changes reuse existing AI assets.
3. **Preview before generation.** Free in-browser preview (Remotion Player, still placeholders for AI parts). AI starts only after explicit approval, with credits reserved first.
4. **Replaceable providers.** All AI calls go through `lib/providers` behind one interface. MVP enables **Atlas Cloud + kie.ai**; fal, MuAPI, BytePlus and a self-hosted GPU must be addable without touching UI/business logic.
5. **Every generation is cost-logged** (provider, model, billed seconds, expected cost, billed cost, status, retry).
6. **Merchants never see model names**, only quality tiers.
7. **Content safety**: input checks, blocked terms (AR/FR), AI-label notice, avatars are our own generated faces only (no real-person likeness), Terms of Use.
8. **Cost guardrails**: global daily AI budget (admin switch) auto-pauses free trials first, then all AI; provider circuit breaker; per-job timeout.
9. **Licence hygiene**: only copy code with a clear licence (see §3). Keep `NOTICE` current.

## 3. Base project (changed)
- **New repo `taswiq-studio`, licence AGPL-3.0, TypeScript only.** No fork of Open-Generative-AI (JS, no auth, key in browser). The old fork `TaswiQ-Generative-AI` is kept as a read-only reference (MuAPI model list) and archived after launch.
- Port (copy with header + `NOTICE` entry, AGPL-3.0 from `xixihhhh/clipforge`): `ssrf-guard`, `circuit-breaker`, `ad-compliance` rules (translated to AR/FR), `product-ingest`, `subtitle-export`, hook patterns, presenter realism prompt constraint.
- Re-implement from ideas only (no code copying, no licence file upstream): Atlas Marketing Studio credit pricing formula, `PaymentProvider` interface, task resume.
- Do **not** copy from: one-daihuo, daihuo-jianshou mirrors, 三千AI (proprietary), anything without a licence file.
- Remotion: free for individuals/companies up to 3 people; a 4th team member (or an employee/investor-owned entity) requires a paid licence (Automators: $0.01/render, $100/month minimum). Record in `docs/PROGRESS.md` when the team grows; fallback is FFmpeg compose.

## 4. Stack
| Layer | Choice |
|---|---|
| App | Next.js 16 (App Router), TypeScript, Tailwind 4, shadcn/ui, `next-intl` (ar default RTL, fr) |
| Auth + DB + Storage | Supabase (Postgres, RLS, Storage). Credits via SQL functions (RPC), not client transactions |
| AI shot (i2v) | **Atlas Cloud** `bytedance/seedance-v1-pro-fast/image-to-video` 480p (standard) |
| UGC lip-sync | **kie.ai** InfiniteTalk 480p (standard); fallback Atlas `atlascloud/infinitetalk` |
| Script writing | Gemini Flash-Lite (vision) → strict JSON (model id in config; upgrade to Flash if Darija test fails) |
| Voice | Azure Speech TTS `ar-DZ` (male+female), `fr-FR` (male+female), word boundaries → subtitles; alternative voices decided at Day-0 |
| Preview | `@remotion/player` |
| Final render | Remotion + FFmpeg in a Docker worker, concurrency 1 (raise after measurement) |
| Queue | Redis + BullMQ (`ai-jobs`, `render-jobs`) |
| Payments | Manual top-up first → Chargily Pay V2 (CIB/EDAHABIA) webhook, idempotent on checkout id |
| Anti-abuse | Cloudflare Turnstile at signup, daily trial budget cap, one trial per account; phone OTP post-launch |
| Hosting | VPS 4 vCPU/8 GB, Docker Compose, Caddy, DNS on Cloudflare |

## 5. Target directory layout
```
app/
  [locale]/(marketing)/    landing + pricing (Arabic/French)
  [locale]/(auth)/         login / signup (Supabase + Turnstile)
  [locale]/studio/
    ads/new/               wizard
    ads/[id]/              editor + preview + generate + download
    library/  credits/
  [locale]/admin/          users, manual credits, jobs, costs, budget switch, tier switch
  api/
    ads/ ads/[id]/generate ads/[id]/render
    script/ tts/ upload/
    payments/chargily/     checkout + webhook
    taswiq/import
lib/
  providers/
    types.ts registry.ts circuit-breaker.ts
    atlas.ts kie.ts        (MVP)   muapi.ts fal.ts byteplus.ts (stubs, disabled)
  credits.ts costs.ts abuse.ts
  supabase/ tts/azure.ts llm/gemini.ts compliance/ ssrf-guard.ts
remotion/
  compositions/ProductAd.tsx UgcAd.tsx
  components/              PriceBadge, Headline, Subtitles (RTL-safe), CTA, Watermark, AvatarFrame
  Root.tsx
worker/ ai-worker.ts render-worker.ts Dockerfile
config/ tiers.ts prices.ts
supabase/migrations/
docs/ TASKS.md ARCHITECTURE.md PROGRESS.md BASE_COMPARISON.md
NOTICE LICENSE (AGPL-3.0)
```

## 6. Provider interface (contract)
```ts
export type JobStatus = 'queued' | 'running' | 'succeeded' | 'failed';

export interface ImageToVideoInput {
  imageUrl: string; prompt: string; durationSec: number;
  resolution: '480p' | '720p' | '768p' | '1080p'; aspect: '9:16';
}
export interface LipSyncInput {
  avatarImageUrl: string; audioUrl: string; audioSec: number; resolution: '480p' | '720p';
}
export interface ProviderJob { providerJobId: string; expectedCostUsd: number; }
export interface ProviderResult {
  status: JobStatus; videoUrl?: string; error?: string; retryable?: boolean;
  billedSeconds?: number; costUsd?: number;
}
export interface VideoProvider {
  name: string;
  imageToVideo(model: string, input: ImageToVideoInput): Promise<ProviderJob>;
  getResult(providerJobId: string): Promise<ProviderResult>;
}
export interface LipSyncProvider {
  name: string; maxAudioSec: number;          // kie.ai InfiniteTalk = 15
  lipSync(model: string, input: LipSyncInput): Promise<ProviderJob>;
  getResult(providerJobId: string): Promise<ProviderResult>;
}
```
UI/business logic call `registry.ts` with a **tier** (and ad type), never a provider. Registry order per tier: primary → fallback, with circuit breaker and per-job timeout (`i2v` 5 min, `lipsync` 8 min).

## 7. Quality tiers (`config/tiers.ts`) — prices checked 2026-10-08 (list prices, no promos)
| Tier | AI shot (5 s) | UGC lip-sync (10 s) | AI cost |
|---|---|---|---|
| `standard` (free trial, MVP) | Atlas Seedance 1.0 Pro Fast 480p ($0.010/s) | kie.ai InfiniteTalk 480p ($0.015/s) → fallback Atlas ($0.03/s) | $0.05 / $0.15 |
| `pro` (disabled) | Atlas Wan 2.2 ($0.03/s) or Seedance Pro Fast 720p | kie.ai InfiniteTalk 720p ($0.06/s) | $0.15 / $0.60 |
| `premium` (disabled) | Atlas Kling v2.6 Pro ($0.07/s) / MiniMax H3 Max Turbo ($0.025/s) | Atlas Kling v2.6 Std Avatar ($0.056/s) | $0.35 / $0.56 |
Credit cost per action is in the same file; initial placeholder: Product 2, +AI shot 6, UGC 12 credits (owner sets credit→DZD). Credits are integers. Retry allowance 15% is included in margins, not charged to merchants.

## 8. Data model (Supabase, RLS by `user_id`)
- `profiles` (id, full_name, phone, phone_verified, store_url, is_admin, free_trial_used, signup_ip_hash, created_at)
- `credit_ledger` (id, user_id, delta int, reason enum[topup, reserve, commit, refund, free_trial, admin], ref_id unique per reason, created_at) — balance = SUM(delta); reserve/commit/refund implemented as Postgres functions
- `products` (id, user_id, name, price_dzd, description, image_paths text[], source enum[upload, taswiq], created_at)
- `avatars` (id, name, gender, age_group, image_path, voice_id, language, active) — TaswiQ-owned generated faces, read-only for users
- `ad_projects` (id, user_id, product_id, type enum[product, product_ai, ugc], template_id, language enum[darija, ar, fr], tier, headline, script jsonb, price_text, cta_text, colors jsonb, music_id, voice_id, avatar_id, status enum[draft, generating, ready_for_render, rendering, done, failed], ai_asset_path, audio_path, output_path, watermark bool, regen_count int, created_at, updated_at)
- `generation_jobs` (id, ad_project_id, user_id, kind enum[i2v, lipsync, tts, script, render], provider, model, provider_job_id, status, billed_seconds, expected_cost_usd, cost_usd, retry_count, error, created_at, finished_at)
- `payments` (id, user_id, provider 'chargily' | 'manual', checkout_id unique, amount_dzd, credits, status, raw jsonb, created_at)
- `settings` (key, value jsonb): `daily_ai_budget_usd`, `daily_trial_budget_usd`, `tiers_enabled`, `providers_enabled`
Storage buckets: `product-images`, `ai-assets`, `audio`, `renders`, `avatars` (public read); others private/signed URLs. Retention: AI assets and renders 30 days unless saved.

## 9. Core flows
**Ad builder**
1. Upload product images (min 600 px short side, blur/size check) or import from TaswiQ. No background removal in MVP.
2. Choose type, language, template, voice (or avatar for UGC).
3. `POST /api/script` → Gemini (image + product data) → JSON `{hook, headline, script, cta, subtitles_text}` in the chosen language; merchant can edit everything. Script length validated against the audio cap (UGC speech ≤ 10 s).
4. `POST /api/tts` → Azure audio + word timings, cached by hash(text+voice).
5. Live preview with Remotion Player (AI parts as animated still / avatar image).
6. "Generate" → balance + budget guard → **reserve** credits → enqueue `ai-jobs` (skipped for `product`).
7. AI worker → registry (primary → fallback) → store asset → log expected vs billed cost → **commit** credits (refund on failure, 1 auto-retry; merchant may regenerate the AI part up to 2 times, each paid).
8. Enqueue `render-jobs` → MP4 1080×1920 (720p + watermark for free trial) → upload → `done`.
9. Later edits of text/price/CTA/music/voice → re-render only (no AI, no AI credits; render free in MVP).
**AI shot fidelity**: the product is always shown by deterministic layers (logo, price badge, original product image); the AI clip is mood/context with a low-motion prompt. UGC avatar sits in a framed window (not full-screen) because 480p is soft.

**Free trial** (changed): on signup after Turnstile + email verification, grant 1 AI generation at `standard`, watermarked; UGC trial capped at 6 s of avatar speech; up to 3 free watermarked Product Ads (no AI). Trials stop automatically when `daily_trial_budget_usd` is reached. Phone OTP before AI generation is added post-launch (T5.2).

**Payments**: MVP: admin adds credits manually (BaridiMob/CCP receipts via WhatsApp). Then Chargily: create checkout → redirect → signed webhook → verify signature → ledger `topup` (idempotent on checkout id). Price packs are net of Chargily fees (1.5–4.5% by plan; refunds 2%).

**TaswiQ integration (after MVP)**: store dashboard button opens `studio.taswiq.store/import?token=...`; HMAC-signed `{store_url, product:{name, price, images, description}, exp}`.

## 10. Rendering rules
- Compositions are pure React; all text from props; `dir="rtl"`; OFL Arabic font (Cairo/Tajawal) bundled locally (no runtime Google Fonts fetch in the renderer).
- Mixed Arabic/Latin/digits must render correctly (test: `السعر 2,900 دج - Livraison 58 wilayas`) in both Player and final render.
- Durations: Product Ad 15 s, UGC 20 s. Fixed slots for AI clip / avatar window.
- Output: H.264 MP4, AAC, 30 fps, 1080×1920. Music: only tracks with an explicit commercial licence, license text stored in `public/music/LICENSES.md`.

## 11. Environment variables and network domains
```
NEXT_PUBLIC_SUPABASE_URL=  NEXT_PUBLIC_SUPABASE_ANON_KEY=  SUPABASE_SERVICE_ROLE_KEY=
ATLASCLOUD_API_KEY=   KIE_API_KEY=   GEMINI_API_KEY=
AZURE_SPEECH_KEY=  AZURE_SPEECH_REGION=
REDIS_URL=  APP_URL=
NEXT_PUBLIC_TURNSTILE_SITE_KEY=  TURNSTILE_SECRET_KEY=
CHARGILY_SECRET_KEY=  CHARGILY_WEBHOOK_SECRET=
TASWIQ_IMPORT_HMAC_SECRET=
```
Allow-list for Claude Code sessions (network): `github.com`, `api.github.com`, `codeload.github.com`, `raw.githubusercontent.com`, `registry.npmjs.org`, `registry-1.docker.io`, `ghcr.io`, `api.atlascloud.ai`, `static.atlascloud.ai`, `api.kie.ai`, `kie.ai`, `docs.kie.ai`, `generativelanguage.googleapis.com`, `*.tts.speech.microsoft.com`, `*.cognitiveservices.azure.com`, `*.supabase.co`, `challenges.cloudflare.com`, `pay.chargily.net` (verify exact host in T5.1), `dev.chargily.com`, `www.remotion.dev`, `fonts.googleapis.com` / `fonts.gstatic.com` (only to download fonts once), `api.muapi.ai`, `fal.ai`, `queue.fal.run` (optional price checks). If a domain is blocked, tell the user exactly which one.

## 12. Out of scope for MVP (do not build)
Pro/premium tiers enabled, more than two providers live, background removal, music library beyond 3 tracks, custom avatars from users, voice cloning of users, timeline editor, team accounts, public API, self-hosted GPU, mobile app, analytics dashboards, Douyin/TikTok scraping or "viral replication".

## 13. Definition of done (MVP)
A new merchant can sign up (Turnstile + email), get the free trial, build each of the 3 ad types with a free preview in Arabic RTL (and French), generate, and download an MP4 (watermarked for trial); admin can add credits manually, see jobs with expected vs billed cost, set daily budgets and enable/disable tiers/providers; unit economics visible per ad type; app runs on the VPS behind `studio.taswiq.store`. Chargily is part of public launch (T5.1), not the private beta.
