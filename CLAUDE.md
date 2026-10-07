# CLAUDE.md — TaswiQ Studio

> This file is read automatically by Claude Code at the start of every session.
> It is the single source of truth. Do NOT ask the user to re-explain anything written here.

## 0. How to work with me (mandatory)
- Reply to the user in **Arabic**, very concise and practical. Code, comments, commit messages: English.
- Work on **one task from `docs/TASKS.md` per session**. Do not start the next task unless asked.
- Before coding a task: read only the files needed. Do not re-scan the whole repo every session; use `docs/ARCHITECTURE.md`.
- When the user must run something manually (accounts, keys, deploy): give **one command at a time** + one line on what it verifies, then wait.
- **Never** run destructive commands (rm -rf, drop table, force push, deleting migrations) without explicit permission.
- After finishing a task: run lint/typecheck, make a small commit, tick the task in `docs/TASKS.md`, and append 2–4 lines to `docs/PROGRESS.md` (what was done, decisions, anything pending).
- If a decision is not covered here, choose the simplest option that keeps the provider layer replaceable, and record it in `docs/PROGRESS.md`.
- Never put API keys in client code. Never commit `.env*` files.

## 1. Product
**TaswiQ Studio** (`studio.taswiq.store`): a self-service platform where Algerian e-commerce merchants turn a product image into a ready-to-publish vertical ad video (9:16). Think "Higgsfield specialised for Algerian store ads".

Owner: Aimen. Parent business: TaswiQ (taswiq.store) builds COD stores on CodFlow; each merchant store is a separate deployment.

### Ad types (all three in the MVP)
1. **Product Ad** — product images + animated template + headline + price in DZD + CTA + voice-over + subtitles. No AI video.
2. **Product Ad + AI shot** — same as (1) with one AI-generated 3–5 s clip (image-to-video from the product image) inserted in the template.
3. **UGC Ad** — a TaswiQ avatar talks about the product (lip-sync 8–10 s) + product shots + price + subtitles + CTA.

### Languages
Algerian Darija (written in Arabic script), Modern Standard Arabic, French. UI in Arabic (RTL) first, French second.

## 2. Non-negotiable principles
1. **AI only where it adds value.** AI video is limited to short clips (3–5 s product shot, 8–10 s UGC speech). Everything else is deterministic rendering (Remotion + FFmpeg).
2. **Editing never re-triggers AI.** Changing text, price, CTA, colours, music, voice, template must reuse existing AI assets.
3. **Preview before generation.** The merchant builds the ad and sees a free in-browser preview (Remotion Player, static image placeholders for AI parts). AI generation is only triggered after explicit approval, and credits are reserved first.
4. **Replaceable providers.** All AI calls go through `lib/providers` behind one interface. MVP implements MuAPI only; fal, Atlas Cloud, kie.ai, and a future self-hosted GPU must be addable without touching UI or business logic.
5. **Every generation is cost-logged** (provider, model, seconds, cost_usd, status, retry). Unit economics are measured from day one.
6. **Merchants never see model names**, only quality tiers.
7. **Content safety.** The upstream project has no content filters. We add prompt/input checks and Terms of Use. Generated assets are reviewed by status flags (see admin).

## 3. Base project
- Fork of `github.com/Anil-matcha/Open-Generative-AI` (Next.js, MIT). Keep the MIT licence notice.
- We keep its UI pieces where useful, but **move all MuAPI calls server-side** and add: auth, credits, ad builder, templates, avatars, render pipeline, payments.
- First task maps the real structure into `docs/ARCHITECTURE.md`. Adapt the paths below to the real repo layout and record any change there.

## 4. Stack
| Layer | Choice |
|---|---|
| App | Next.js (from the fork), TypeScript |
| Auth + DB + Storage | Supabase (Postgres, RLS, Storage buckets) |
| AI video / lip-sync | MuAPI (MVP). Later: fal, Atlas Cloud / kie.ai, self-hosted GPU |
| Script writing | Gemini Flash-Lite (vision) → strict JSON |
| Voice | Azure Speech TTS: `ar-DZ` (male + female), `fr-FR` (male + female); word-boundary timings for subtitles |
| Preview | `@remotion/player` in the browser |
| Final render | Remotion renderer + FFmpeg in a Docker worker on the VPS |
| Queue | Redis + BullMQ (`ai-jobs`, `render-jobs`) |
| Payments | Chargily Pay V2 (CIB / EDAHABIA), webhook → credits. MVP week 1: manual top-up |
| Hosting | VPS 4 vCPU / 8 GB, Docker Compose, Caddy, DNS on Cloudflare |

## 5. Target directory layout (adapt to fork)
```
app/
  (marketing)/             landing + pricing (Arabic)
  (auth)/                  login / signup (Supabase, phone verification later)
  studio/
    ads/new/               ad builder wizard
    ads/[id]/              editor + preview + generate + download
    library/               merchant's ads
    credits/               balance, top-up
  admin/                   minimal: users, manual credit top-up, jobs, costs
  api/
    ads/                   CRUD ad_projects
    ads/[id]/generate      reserve credits → enqueue ai job
    ads/[id]/render        enqueue render job
    script/                Gemini script generation
    tts/                   Azure TTS (cached)
    payments/chargily/     checkout + webhook
    taswiq/import          signed product import from TaswiQ stores
lib/
  providers/
    types.ts               VideoProvider, LipSyncProvider interfaces
    muapi.ts               MVP implementation
    registry.ts            tier → provider+model mapping (from config)
  credits.ts               reserve / commit / refund (transactional)
  costs.ts                 cost logging
  supabase/                server + browser clients
  tts/azure.ts
  llm/gemini.ts
remotion/
  compositions/ProductAd.tsx
  compositions/UgcAd.tsx
  components/              PriceBadge, Headline, Subtitles (RTL-safe), CTA, Watermark
  Root.tsx
worker/
  ai-worker.ts             runs AI jobs, polls provider, stores result, logs cost
  render-worker.ts         renders Remotion → MP4, uploads to storage
  Dockerfile
config/
  tiers.ts                 quality tiers, models, credit prices (single place to edit)
supabase/migrations/
docs/ CLAUDE.md companions: TASKS.md, ARCHITECTURE.md, PROGRESS.md
```

## 6. Provider interface (contract)
```ts
export type JobStatus = 'queued' | 'running' | 'succeeded' | 'failed';

export interface ImageToVideoInput {
  imageUrl: string; prompt: string; durationSec: number;
  resolution: '480p' | '720p' | '768p' | '1080p'; aspect: '9:16';
}
export interface LipSyncInput {
  avatarImageUrl: string; audioUrl: string; resolution: '480p' | '720p';
}
export interface ProviderJob { providerJobId: string; }
export interface ProviderResult {
  status: JobStatus; videoUrl?: string; error?: string;
  billedSeconds?: number; costUsd?: number;
}
export interface VideoProvider {
  name: string;
  imageToVideo(model: string, input: ImageToVideoInput): Promise<ProviderJob>;
  getResult(providerJobId: string): Promise<ProviderResult>;
}
export interface LipSyncProvider {
  name: string;
  lipSync(model: string, input: LipSyncInput): Promise<ProviderJob>;
  getResult(providerJobId: string): Promise<ProviderResult>;
}
```
UI and business logic only call `registry.ts` with a **tier**, never a provider directly.

## 7. Quality tiers (`config/tiers.ts`) — prices are placeholders until Day-0 test
| Tier (shown to merchant) | AI shot model | UGC lip-sync model |
|---|---|---|
| `standard` (free trial) | Seedance 1.0 Pro Fast 480p | InfiniteTalk 480p |
| `pro` | Wan 2.2 | InfiniteTalk 720p |
| `premium` | MiniMax H3 Max Turbo | Kling v2.6 Avatar |
MVP enables **`standard` only**; other tiers exist in config but are disabled.
Credit cost per action is defined in the same file. Credits are integers; 1 credit maps to a DZD price set by the owner.

## 8. Data model (Supabase, all tables with RLS by `user_id`)
- `profiles` (id = auth.users.id, full_name, phone, store_url, is_admin, free_trial_used bool, created_at)
- `credit_ledger` (id, user_id, delta int, reason enum[topup, reserve, commit, refund, free_trial, admin], ref_id, created_at) — balance = SUM(delta)
- `products` (id, user_id, name, price_dzd, description, image_paths text[], source enum[upload, taswiq], created_at)
- `avatars` (id, name, gender, age_group, image_path, voice_id, active) — owned by TaswiQ, read-only for users
- `ad_projects` (id, user_id, product_id, type enum[product, product_ai, ugc], template_id, language enum[darija, ar, fr], tier, headline, script jsonb, price_text, cta_text, colors jsonb, music_id, voice_id, avatar_id, status enum[draft, generating, ready_for_render, rendering, done, failed], ai_asset_path, audio_path, output_path, watermark bool, created_at, updated_at)
- `generation_jobs` (id, ad_project_id, user_id, kind enum[i2v, lipsync, tts, script, render], provider, model, provider_job_id, status, billed_seconds, cost_usd, retry_count, error, created_at, finished_at)
- `payments` (id, user_id, provider 'chargily', checkout_id, amount_dzd, credits, status, raw jsonb, created_at)

Storage buckets: `product-images`, `ai-assets`, `audio`, `renders`, `avatars` (public read), others private with signed URLs.

## 9. Core flows
**Ad builder**
1. Upload product images (or import from TaswiQ) → background removal is NOT in MVP.
2. Choose type, language, template, voice (or avatar for UGC).
3. `POST /api/script` → Gemini reads the image + product data → JSON `{hook, headline, script, cta, subtitles_text}` in the chosen language. Merchant can edit all fields.
4. `POST /api/tts` → Azure audio + word timings. Cache by hash(text+voice).
5. Live preview with Remotion Player (AI parts shown as animated still / avatar image).
6. "Generate" → check balance → **reserve** credits → enqueue `ai-jobs` (skipped for type `product`).
7. AI worker → provider → store asset in `ai-assets` → log cost → **commit** credits (refund on failure, max 1 auto-retry).
8. Enqueue `render-jobs` → render MP4 1080×1920 (720p + watermark for free trial) → upload → status `done`.
9. Any later edit of text/price/CTA/music/voice → re-render only (no AI, no AI credits; render is free in MVP).

**Free trial**: on signup, `free_trial_used=false`. One generation of any type at `standard` tier, watermarked, granted via ledger `free_trial`. Unlimited previews. Later: phone OTP to prevent multi-accounts.

**Payments**: MVP week 1 → admin adds credits manually (BaridiMob/CCP receipts via WhatsApp). Then Chargily: create checkout → redirect → signed webhook → verify signature → ledger `topup` (idempotent on checkout_id).

**TaswiQ integration (after MVP)**: store dashboard button opens `studio.taswiq.store/import?token=...`; token is HMAC-signed `{store_url, product: {name, price, images, description}, exp}`.

## 10. Rendering rules
- Compositions are pure React; all text from props; RTL via `dir="rtl"` and an Arabic font with commercial licence (e.g. Cairo / Tajawal, OFL).
- Arabic + Latin + digits mixed text must render correctly (test case: `السعر 2,900 دج - Livraison 58 wilayas`).
- Durations: Product Ad 15 s, UGC 20 s. Fixed layout slots for the AI clip / avatar.
- Output: H.264 MP4, AAC, 30 fps, 1080×1920.

## 11. Environment variables (`.env.example`, never commit real values)
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
MUAPI_API_KEY=
GEMINI_API_KEY=
AZURE_SPEECH_KEY=
AZURE_SPEECH_REGION=
REDIS_URL=
CHARGILY_SECRET_KEY=
CHARGILY_WEBHOOK_SECRET=
TASWIQ_IMPORT_HMAC_SECRET=
APP_URL=
```

## 12. Out of scope for MVP (do not build)
Multiple providers live, pro/premium tiers enabled, background removal, music library beyond 3 tracks, custom avatars from users, voice cloning of users, timeline editor, team accounts, public API, self-hosted GPU, mobile app, analytics dashboards.

## 13. Definition of done (MVP)
A new merchant can sign up, get 1 free generation, build each of the 3 ad types with a free preview, generate, download a watermarked MP4; an admin can add credits manually; every AI call appears in `generation_jobs` with cost; app runs on the VPS behind `studio.taswiq.store`.
