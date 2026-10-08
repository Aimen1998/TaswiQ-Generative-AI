# BASE_COMPARISON.md — Base project, cost study, and final architecture

> Research + planning only. No application code was changed. Date of all data: **2026-10-08**.
> Language: English (matches CLAUDE.md/TASKS.md); the owner summary was delivered in Arabic in chat.

## 0. Executive decision (one page)

| Question | Decision |
|---|---|
| Base project | **None of the candidates is a good base.** Start a **clean TypeScript repo `taswiq-studio`** (Next.js + Supabase), licence **AGPL-3.0**. Do **not** build on Open-Generative-AI. |
| Why | The only candidates with real multi-user SaaS plumbing (Atlas Marketing Studio) have **no LICENSE file**, are locked to one provider, and the reusable part is ~500 lines. The most capable one (ClipForge, AGPL) is a **single-user local app** (SQLite, files on disk, Electron), 80k LOC, China-oriented. Open-Generative-AI is JavaScript, no auth, API key in the browser. |
| Port from ClipForge (AGPL, accepted) | `ssrf-guard`, `ad-compliance` (pattern), `product-ingest`, `circuit-breaker`, `subtitle-export`, script-engine *prompt structure*, "product-lock" principle, realism prompt constraint for presenters. Copy with attribution in `NOTICE`. |
| Re-implement (do not copy) from Atlas | credits-from-cost formula, `PaymentProvider` interface (checkout vs redeem), polling/resume pattern, media-storage abstraction. Ideas are free; code has no licence. |
| Reuse from Open-Generative-AI | Only its **MuAPI endpoint/model list** as a reference (MIT). Archive the fork afterwards. |
| Providers (MVP) | **Atlas Cloud** (AI shot, fallback lip-sync, LLM option) + **kie.ai** (InfiniteTalk lip-sync, cheapest). MuAPI is **2–5× more expensive** for the same models → not the MVP default. `lib/providers` keeps all of them swappable. |
| Cost per finished ad (list prices, 15% retry allowance) | Product Ad **≈ $0.023** · Product Ad + AI shot **≈ $0.08** · UGC (10 s avatar) **≈ $0.20** (Atlas-only fallback ≈ $0.37) |
| Free trial per user | **≈ $0.13 worst case** (UGC capped at 6 s avatar), **≈ $0.10 expected** |
| Timeline (estimate, 1 task/session) | Private beta (manual top-up) **~3 weeks**; public launch with Chargily **~4–5 weeks**. The old "10 days" is not realistic. |

---

## 1. Method, scope, and what is NOT verified

* Searched GitHub (REST search + topic/keyword), Gitee/HF/Chinese keywords through web search. Gitee pages could not be fetched directly (search returned only pointers); HF was not mined for apps (only models matter for the later GPU phase).
* **Shallow-cloned (depth 1) and read code**: `xixihhhh/clipforge`, `AtlasCloudAI/atlas-marketing-studio`, `xiaolinbenben/terln-studio` (the "fork"), `AKA-liang/one-daihuo`, `witty-suckerpunch492/daihuo-jianshou`, `2961799660/sanqian_ai`, `harry0703/MoneyPrinterTurbo`, `ATH-MaaS/Pixelle-Video`, plus this repo (`Aimen1998/TaswiQ-Generative-AI`).
* **Not run**: none of the candidates was built or executed (no `npm install`, no API keys). Findings come from code reading, repo metadata, and licence files. "Code quality" is therefore a reading-based judgement, not a measured one.
* Prices: pulled from public pages/APIs on 2026-10-08 (sources in §9). Where a page was JavaScript-only and could not be read, the number is marked **unverified**.
* Lip-sync prices on MuAPI were derived from its public `estimate-cost` endpoint using a public 372-second MP3 (cost ÷ seconds).

---

## 2. Review of the current plan (CLAUDE.md + TASKS.md)

Severity: **H** = changes money/launch, **M** = causes rework, **L** = polish.

| # | Sev | Weak point / wrong assumption | Evidence | Fix in the new plan |
|---|---|---|---|---|
| 1 | H | **Base = Open-Generative-AI fork "MIT, Next.js, TypeScript".** The repo is **JavaScript** (0 `.ts` files, 151 `.js`), has **no auth**, reads the MuAPI key from `localStorage`/`window` in the browser (`src/lib/muapi.js`), mixes Vite+Electron+Next, pulls 3 git submodules (2 are other products), 116 MB. "Keep its UI pieces" saves almost nothing: generic image/video/lipsync studios ≠ ad builder. | `package.json`, `src/lib/muapi.js:7-11`, `.gitmodules` | Clean repo; T1.3 "remove client-side keys" disappears because there is nothing to remove. |
| 2 | H | **MuAPI as the only MVP provider contradicts "lowest possible cost".** Same models cost 2–5× more than Atlas/kie (table §5.3). | §5 | Atlas Cloud + kie.ai as MVP providers; MuAPI optional. |
| 3 | H | **How does the owner pay USD providers from Algeria?** Not mentioned anywhere. Atlas accepts card/crypto, min top-up **$25**; kie.ai methods **unverified**. | Atlas docs (§9) | Added as **U0 (day-0 user task)**: working USD payment route before anything else. |
| 4 | H | **Free trial can be farmed.** Email-only signup + 1 free generation of *any* type (UGC ≈ $0.20) at launch; phone OTP postponed to T5.2. | CLAUDE.md §9 | Trial limited (UGC avatar ≤ 6 s), Turnstile, one trial per phone/IP/device fingerprint where possible, **global daily trial budget cap that auto-pauses trials**, OTP after launch. |
| 5 | H | **Remotion licence not mentioned.** Free only for individuals/companies **up to 3 people**; 4+ people need a paid company licence (**Automators: $0.01/render, $100/month minimum**). Remotion is source-available, not OSI open source. | remotion.pro/license | Recorded as a risk with trigger ("4th team member or any investor/employee"). Keep Remotion; fallback = FFmpeg compose (ClipForge style). |
| 6 | M | **Darija TTS quality is unproven.** Azure `ar-DZ` voices (Amina/Ismael) read Arabic text with an Algerian accent but are MSA-oriented; written Darija ("chhal", "ماشي", French loanwords) may mispronounce. The plan has no listening test and no fallback voice. | — | Day-0 voice test judged by the owner (native), with 3 TTS candidates; fallback voices configurable per language. |
| 7 | M | **Day-0 test only on MuAPI** and only 5 images; no check of 9:16 handling, face rejection, logo/text distortion, or 480p upscale to 1080×1920. | TASKS T0.2 | Day-0 now compares Atlas/kie/MuAPI, includes 5 hard images (logo text, transparent PNG, human model, dark photo, low-res), records latency and failures. |
| 8 | M | **Quality of 480p InfiniteTalk on a 1080×1920 canvas** will look soft if full-screen. | — | UGC composition places the avatar in a framed window (≈ 60% width) + product cut-ins; 720p reserved for `pro` tier (kie $0.06/s). |
| 9 | M | **AI shot product fidelity** (i2v warps logos/packaging text) is not addressed. | ClipForge's "product lock" idea | AI shot is a *mood/context* shot of ≤ 5 s with a low-motion prompt; the product logo/price badge always comes from deterministic overlays; merchant sees the AI shot and may regenerate (capped, paid). Add regeneration policy (missing today). |
| 10 | M | **Credits: reserve/commit/refund through Supabase client can't be transactional.** PostgREST has no interactive transactions. | — | Implement as Postgres functions (RPC) with `ref_id` idempotency; unit-tested with concurrent calls. |
| 11 | M | **Render capacity on a 4 vCPU/8 GB VPS is not estimated.** Remotion uses headless Chromium (~1–3 GB per render). | — | Concurrency = 1 (2 after measurement); render test in T2.4 records seconds/video; queue with position display. (Estimate, not measured.) |
| 12 | M | **No spend guardrails**: no daily provider budget, no circuit breaker when a provider is down, no webhook/poll timeout policy. | — | `lib/providers` gets timeout, circuit breaker (port from ClipForge), daily budget switch in admin. |
| 13 | M | **Chargily**: fees (1.5%–4.5% by plan + 1.5% withdrawal beyond free payouts, refunds 2%) and merchant-account requirements are not in the plan; account verification may take days. | chargily.com/business/pay/pricing | Open the Chargily account on day 1 (U0), price packs net of fees. |
| 14 | M | **Edge-TTS is an unofficial endpoint** (used by MoneyPrinterTurbo/ClipForge for "free" voice). Using it in a paid SaaS risks a ToS block. | code reading | Use official Azure Speech for production; Edge-TTS only for local tests. |
| 15 | M | **Content/legal**: AI-generated avatars need own-generated faces (no real-person likeness), AIGC labelling for TikTok/Meta, Algerian e-commerce law 18-05 and data-protection law 18-07 — plan has only a generic T4.2. | — | T4.2 expanded; lawyer review of ToS is a user task before public launch (not legal advice). |
| 16 | L | **Music**: "3 tracks" without a licence source. | — | Use tracks with an explicit commercial licence (e.g. Pixabay Content Licence, CC0), license file stored in repo. |
| 17 | L | **Storage retention/egress** not planned (1080p MP4 ≈ 5–15 MB each). | — | Retention policy: AI assets 30 days, renders 30 days unless saved; revisit R2 when Supabase storage > 50 GB. |
| 18 | L | **Timeline**: 22 tasks in 10 days. A Claude Code session is one task; realistic is 3 weeks to beta. | — | See `TASKS.next.md`. |
| 19 | L | **Gemini Flash-Lite for Darija copywriting** unproven; cost is negligible so use the stronger Flash model if Lite fails the Day-0 copy test. | — | Model id in config; Day-0 copy test with 10 products. |
| 20 | L | **Product images quality guard** missing (blur, tiny, watermark) – cheap deterministic templates look bad on poor photos. | — | Upload validation (min 600 px short side, blur check) + friendly message. Background removal stays P1 (post-MVP). |

What is **right** in the current plan and kept: AI only for short clips, editing never re-triggers AI, preview before paid generation, provider interface + registry, cost-logging from day one, merchants never see model names, DZD credits, manual top-up first.

---

## 3. Candidate landscape

### 3.1 Everything evaluated

| Project | Licence (code) | Stars / last push | Verdict |
|---|---|---|---|
| **xixihhhh/clipforge** (canonical; ex-带货剪手) | AGPL-3.0 (LICENSE + NOTICE + package.json) | 940★ / 2026-09-23 | **Best source of ideas/modules; not a base** (single-user). |
| reedchan7/clipforge, 0xsline/clipforge, Amayzhang/clipforge | AGPL-3.0 | forks (0–1★) | Forks of the above. Ignore. |
| witty-suckerpunch492/daihuo-jianshou | **none** | 303★ / 2026-10-06, created 2026-04 | Unlicensed repackaging of the older 带货剪手 code. Do not use. |
| wangzhenchuan/daihuo-jianshou | none | 0★ | Copy. Ignore. |
| **AKA-liang/one-daihuo** | **none** | 4★ / 2026-05-26 | Early ancestor of ClipForge; Volcengine-only, Douyin scraping (legal risk), local desktop. Reject. |
| **AtlasCloudAI/atlas-marketing-studio** | **No LICENSE file** (README says "MIT") | 39★ / 2026-07-23, 1 squashed commit | Good *reference* for SaaS plumbing; not safe/useful as base. |
| xiaolinbenben/terln-studio (fork of Atlas, 12 commits ahead) | none (inherits the gap) | 0★ / 2026-10-04 | Shows how Atlas was turned into a Docker/SQLite/Alipay/admin product — **proof that Atlas needs a rewrite of auth/payments/deploy to leave Cloudflare/Stripe**. Reference only. |
| **三千AI (2961799660/sanqian_ai)** | **Proprietary "Commercial License"** (all rights reserved; paid buy-out) | 17★ / 2026-09-24 | **Not open source.** The repo holds docs/screenshots and 4 sample files only; the "电商工坊" (sp.sanqianai.cn) is a closed product. Reject. |
| **Anil-matcha/Open-Generative-AI** (this repo's upstream) | MIT | 29.8k★ / 2026-10-07 | Active, popular, but wrong shape (§3.2.4). |
| harry0703/MoneyPrinterTurbo | MIT | 129k★ / 2026-10-07 | Python/Streamlit, stock footage + LLM script + TTS + subtitles; **single-user**, no avatar; has `muapi.py`/`volcengine_seedance.py` services; no Arabic UI (12 UI languages, none AR). Useful reference for subtitle/TTS handling in Python only. |
| ATH-MaaS/Pixelle-Video | Apache-2.0 | 28.7k★ / 2026-06-14 | Topic→video via ComfyUI/RunningHub, HTML templates, Streamlit/single-user; no auth/credits. Reference for HTML templating only. |
| calesthio/OpenMontage | AGPL-3.0 | 65k★ / 2026-10-03 | "Agentic" pipelines driven by a coding agent; not a multi-tenant web product. Reject as base. |
| duixcom/Duix-Avatar (HeyGem) | custom "Other" licence (restrictions) | 15.7k★ / 2026-04-21 | Offline digital-human cloning (Windows/GPU). Licence needs reading before any commercial use; GPU phase only. |
| MeiGen-AI/InfiniteTalk | Apache-2.0 (code) | 8k★ | **Self-hosting candidate** (§6). Check weights licence before deploying. |
| Wan-Video/Wan2.2 | Apache-2.0 | 17.8k★ | Self-hosting candidate. |
| TMElyralab/MuseTalk, bytedance/LatentSync | custom / Apache-2.0 | — | Video-to-video lip-sync alternatives for GPU phase; MuseTalk licence is non-standard. |
| ClabstreamTeam/Open-Higgsfield-AI, BeatAPI/BeatDesign, clawnify/OpenStudio, opencanvasai/OpenCanvas | none / Apache-2.0 / MIT / MIT | 42–182★ | "Higgsfield alternatives" = generic generation canvases, not ad builders. Reject. |
| Linen006/AI-stream | MIT | 5★ | Python content workflow demo, mock-only. Reject. |
| supabase-community/nextjs-subscription-payments | MIT | 97★ (stale 2025-12) | Only a possible *auth/payments starter*, Stripe-centric; no value over a clean Supabase+Next setup. |
| Remotion (remotion-dev/remotion) | Remotion License (free ≤ 3 people) | 62k★ | Keep as renderer (see risk #5). |

No project found that combines: multi-user SaaS + credits + Arabic/RTL + templated deterministic video + lip-sync UGC. The market gap is real, which is why a clean build with ported modules wins.

### 3.2 Deep dives (code actually read)

#### 3.2.1 ClipForge (`xixihhhh/clipforge`, v0.9.10)

| Item | Finding |
|---|---|
| Licence | AGPL-3.0 (LICENSE, NOTICE, package.json). Bundled assets: `public/fonts/subtitle.otf` with `OFL.txt` (OFL font, OK); stock sources are API-fetched (Pexels, Pixabay, Coverr, Jamendo, Freesound, NASA…), each with their own terms. No model weights bundled (cloud APIs). |
| Activity | Very active (v0.9.x, 131 test files, CI badge, e2e smoke). Single main author. |
| Stack | Next.js 16 / React 19 / TS 5 / Tailwind 4, **SQLite (better-sqlite3 + drizzle)**, FFmpeg composition, Electron packaging, MCP server + CLI + Agent Skill. ~80k LOC in `src/`. |
| Features | Product image → selling points → script (hook patterns, 391 ad templates, script judges) → shots → TTS (Edge-TTS) → subtitles/BGM → FFmpeg compose; QC, repair single shots, batch, A/B variants, trend picking, viral replication, platform export framing. |
| Auth / credits / payments / admin / deploy | **None** (local single-user; `settings-store`, files under local paths; Electron). Adding multi-tenancy means replacing the DB layer, storage paths, task runner and all stores. |
| AI providers / lock-in | Pluggable (`providers/`: Atlas Cloud, fal, Replicate, Volcengine, SiliconFlow, Alibaba, OpenAI) — low lock-in. |
| UGC / lip-sync | "UGC" = Seedance 2.0 with in-model audio + realism prompts (`presenters.ts`); **no dedicated lip-sync/avatar pipeline** (no InfiniteTalk/OmniHuman). |
| Product fidelity | Strongest idea: "lock the original product image" so it never distorts; deterministic compose with AI only for shots. |
| Templates | Many, but Chinese-platform oriented (Douyin/Kuaishou/XHS). |
| Arabic/RTL | UI locales: zh/en only. No RTL, Arabic voices not offered. |
| Code quality (reading) | Good: typed, many pure modules, tests, SSRF guard, circuit breaker, error classification. Heavy and sprawling. |
| Effort to reach our MVP | **High (≥ 4–6 weeks)**: strip China features, replace SQLite/local FS with Postgres+object storage, add auth/credits/payments/admin/queue, RTL/Arabic UI, Remotion or RTL-safe subtitles in FFmpeg, avatar lip-sync. More than building clean. |

#### 3.2.2 Atlas Marketing Studio (`AtlasCloudAI/atlas-marketing-studio`)

| Item | Finding |
|---|---|
| Licence | **No `LICENSE` file** (GitHub reports none). README badge and a "License: MIT" line exist. Treat as *ambiguous*: ask the maintainers to add the file before any copying. ~133 MB of `public/` sample media (no licence info). |
| Activity | 1 squashed commit, last push 2026-07-23, 39★. Marketing-driven by Atlas Cloud. |
| Stack | Next.js 14, Prisma (SQLite→D1 or Neon), NextAuth (Google), Stripe, OpenNext/Cloudflare Workers or Vercel; ~10k LOC TS. |
| Features | 4 workflows: UGC product ad (nano-banana first frame → Seedance 2.0 i2v with spoken lines), reference-ad remake, AI drama, ad skit. |
| Auth / credits / payments / admin / deploy | Google login only; `User.credits` + `CreditLedger` with atomic conditional decrement (`credits.ts`, 48 lines) — **deduct up front, refund on error, no reserve/commit**; `PaymentProvider` interface (Stripe checkout or Atlas redeem code); **no admin**; Cloudflare/Vercel deploy. |
| Orchestration | **Client-driven** (browser calls step-by-step APIs and polls). A merchant on a flaky mobile connection can lose progress; no queue/worker. |
| AI providers / lock-in | Only `api.atlascloud.ai` (`src/lib/atlas.ts`); model IDs hard-coded to Seedance 2.0 (list $0.112/s at 480p → a 5 s shot ≈ $0.56). |
| UGC / lip-sync | UGC via Seedance 2.0 with speech in prompt (no avatar control, expensive); `veed/lipsync` only in the remake flow. |
| Product fidelity | Prompt-only ("nano-banana edit" first frame); no deterministic compositing. |
| Templates | None (no Remotion/FFmpeg server render; `ffmpeg.wasm` in the browser to stitch). |
| Arabic/RTL | i18n: en/zh/ja/es. No RTL. |
| Code quality (reading) | Tidy and small, Chinese comments, 2 tests. Good reference; thin. |
| Effort to reach our MVP | **Medium-high and licence-gated**: replace auth, storage, DB, payments, add queue, render, templates, Arabic. The fork `terln-studio` shows the same rewrite (Alipay, admin, Docker, SQLite) took a separate team 12 commits and still lacks templates/UGC avatar. |

What we take (as ideas, re-implemented): credits math (`price = cost × markup × margin`), `PaymentProvider` split, task persistence/resume, storage adapter, polling timeouts.

#### 3.2.3 terln-studio (Atlas fork)

Docker single container, SQLite on a volume, admin panel (domains, Alipay, providers, packages, video credit rules), Alipay login/payment with callback signature verification (`alipay-crypto.ts`). Useful as a **pattern for a local payment gateway callback + admin settings** (Chargily webhook is analogous). No licence; do not copy code.

#### 3.2.4 Open-Generative-AI (this repo)

| Item | Finding |
|---|---|
| Licence | MIT (code). No bundled model weights; "local models" are downloaded at runtime (Wan2GP etc., each with its own licence). |
| Activity | Very active (29.8k★, pushes daily). |
| Stack | **JavaScript** (no TS), Next.js + Vite + Electron hybrid, npm workspaces + 3 git submodules (`Vibe-Workflow`, `Open-Poe-AI`, `Open-AI-Design-Agent`, currently uninitialised `-` in `git submodule status`). |
| Auth/credits/payments/admin | **None.** `AuthModal.js` only stores a MuAPI key. MuAPI key lives in the browser (`localStorage('muapi_key')`). |
| Providers | MuAPI only (`src/lib/muapi.js`, 520 lines) plus local Wan2GP. High lock-in on client side. |
| UGC/lip-sync | `LipSyncStudio.js` generic tool; no ad flow, no product handling. |
| Arabic/RTL | `src/lib/i18n.js`: English UI strings; `app/zh` for Chinese. No RTL. |
| Code quality | 8 test files, all for local inference. Large, mixed-purpose. |
| Effort | The valuable part for us (a model catalogue / endpoint list) is a few KB. Everything else must be rewritten for server-side keys, auth, credits, Arabic. **Starting clean is cheaper.** |

#### 3.2.5 Others read

* **one-daihuo**: Next.js 16 + sql.js, Volcengine key in settings UI, Douyin viral-video dissection with cookie scraping (ToS/legal risk), no licence. Reject.
* **sanqian_ai**: LICENSE = proprietary commercial licence (domain-bound, paid); repository is documentation + a few update-package source files. Not open source. Reject.
* **MoneyPrinterTurbo**: great for "topic → stock footage video" in Python; outputs depend on stock clips (not product-faithful); Streamlit single-user.
* **Pixelle-Video**: Apache-2.0, ComfyUI-centric, HTML templates (`templates/1080x1920/*.html`) — conceptually similar to our Remotion templates but Python/Streamlit single-user.

### 3.3 Scorecard (0–5, higher is better for **our** MVP)

| Criterion | Weight | ClipForge | Atlas MS | OGA fork | MoneyPrinterTurbo | Pixelle | **Clean repo + ports** |
|---|---|---|---|---|---|---|---|
| Licence safety (AGPL acceptable) | 3 | 5 | 1 | 5 | 5 | 5 | **5** |
| Multi-user SaaS plumbing present | 3 | 0 | 3 | 0 | 0 | 0 | **0 → built in W1** |
| Deterministic product-faithful render | 3 | 4 | 0 | 0 | 2 | 2 | **5** (Remotion) |
| UGC lip-sync avatar | 3 | 1 | 2 | 2 | 0 | 0 | **5** (InfiniteTalk) |
| Arabic/RTL readiness | 3 | 0 | 0 | 0 | 0 | 0 | **5** (designed in) |
| Provider independence | 2 | 4 | 1 | 1 | 3 | 3 | **5** |
| Code ready to reuse in our stack | 2 | 3 | 2 | 1 | 1 | 1 | **n/a** |
| Time to MVP | 3 | 1 | 2 | 1 | 1 | 1 | **3** |
| **Weighted** (max 110) | | 47 | 30 | 28 | 32 | 32 | not scored (a plan, see below) |

(The last column is a plan, not existing software; it is shown to make the trade-off explicit. Even the best existing project, ClipForge, scores 47/110 because it has no multi-user plumbing, no UGC lip-sync and no Arabic. Scores are my judgement from code reading, weights chosen for this MVP.)

---

## 4. Recommended architecture (ONE)

```
                     studio.taswiq.store (Caddy, Cloudflare DNS)
Browser (Next.js 16, TS, next-intl ar/fr, RTL, Remotion Player preview)
   │  server actions / API routes (no provider keys in client)
   ▼
Next.js app ── Supabase (Auth, Postgres+RLS, Storage)  ← credits via SQL functions
   │                         ▲
   │ enqueue                 │ status/cost rows
   ▼                         │
Redis + BullMQ ── ai-worker ─┴─► lib/providers/registry.ts ──► Atlas Cloud (shot, fallback lip-sync, LLM)
              └─ render-worker (Remotion + FFmpeg, concurrency 1)      ──► kie.ai (InfiniteTalk)
                                                                       ──► fal / MuAPI / BytePlus (disabled adapters)
                                                                       ──► self-hosted GPU (later, same interface)
Script: Gemini Flash-Lite → JSON    Voice: Azure Speech ar-DZ / fr-FR (+ word boundaries)
Payments: manual top-up (week 1) → Chargily Pay V2 webhook (idempotent)
Anti-abuse: Turnstile + daily trial budget cap + (post-launch) phone OTP
```

### Provider per tier (list prices, ignoring promos)

| Tier | AI shot (5 s) | Lip-sync (10 s) | Cost of AI part |
|---|---|---|---|
| `standard` (MVP, free trial) | Atlas `bytedance/seedance-v1-pro-fast/image-to-video` 480p | kie.ai InfiniteTalk 480p (fallback Atlas `atlascloud/infinitetalk`) | $0.05 / $0.15 |
| `pro` (post-MVP) | Atlas Wan 2.2 (`atlascloud/wan-2.2/image-to-video`) or Seedance Pro Fast 720p | kie.ai InfiniteTalk 720p ($0.06/s) | $0.15 / $0.60 |
| `premium` (post-MVP) | Atlas Kling v2.6 Pro i2v ($0.07/s) or MiniMax H3 Max Turbo ($0.025/s) | Atlas Kling v2.6 Std avatar ($0.056/s) | $0.35 / $0.56 |

Rationale: `standard` must make the MVP margin work; the others are defined in config but disabled until Day-0 shows visible quality gains worth the price.

### Ports and rewrites

| From | What | How | Licence handling |
|---|---|---|---|
| ClipForge | `ssrf-guard.ts`, `circuit-breaker.ts`, `ad-compliance.ts` (rules → AR/FR terms), `product-ingest.ts` (TaswiQ import), `subtitle-export.ts`, hook patterns, realism constraint for avatar images | Copy file, port to our types, keep header | AGPL-3.0 → our repo AGPL-3.0, `NOTICE` lists origin |
| Atlas MS | credit pricing formula, payment interface, task resume | **Re-implement** from the ideas | No copied code |
| OGA | MuAPI model ids/endpoints | Read-only reference | MIT notice if any text is copied |
| MoneyPrinterTurbo | Arabic text normalisation helpers (`_normalize_arabic`) for subtitle matching | Re-implement in TS | MIT notice if ported |

---

## 5. Cost study

### 5.1 Method and assumptions

* All numbers: **public list prices**, USD, retrieved 2026-10-08. Promotional discounts ("actual" vs "origin" on Atlas, "90–95%" promos) are **ignored**; list ("origin") is used.
* Ad geometry: 9:16. AI shot = **5 s**; UGC avatar speech = **10 s** (trial: 6 s).
* Fixed per-ad costs (estimates, not measured): LLM script ≈ $0.003 (Gemini Flash-Lite: $0.30 in / $2.50 out per 1M tokens, one image + ~600 output tokens), TTS ≈ $0.004–0.005 (Azure neural ≈ $16 per 1M characters, ~250–300 chars), render compute ≈ $0.014–0.018 (VPS ≈ $20/month ÷ ~1,500 renders), storage/egress ≈ $0.002.
* **15% retry allowance** on the AI part (failed or rejected generations).
* DZD conversion is intentionally omitted: the owner buys USD at his own rate (see U0). Multiply by that rate.

### 5.2 Image-to-video, per second (list), by provider

| Model | Atlas Cloud | kie.ai | fal.ai | MuAPI (derived) | BytePlus | Cheapest |
|---|---|---|---|---|---|---|
| Seedance 1.0 Pro Fast (480p) | **$0.010/s** | n/a seen | ≈ $0.0097/s (token formula, $1/M tokens) | $0.020/s (5 s = $0.10) | unverified | **fal ≈ Atlas** |
| Seedance 1.0 Pro Fast (720p) | not itemised (one price shown) | — | ≈ $0.022/s | $0.030/s (5 s = $0.15) | unverified | fal |
| Seedance 1.0 Pro (480p) | $0.024/s | — | — | $0.036/s | **$0.0244/s** ($2.5/M tokens, 5 s = $0.122) | Atlas ≈ BytePlus |
| Wan 2.2 (A14B) 480p | **$0.030/s** (turbo $0.020/s) | $0.04/s (turbo) | $0.04/s (580p $0.06, 720p $0.08) | $0.060/s (5 s $0.30; 720p $0.09/s) | — | **Atlas** |
| MiniMax H3 Max Turbo | **$0.025/s** | — | $0.025/s | ≈ $0.052/s (5 s 720p = $0.26) | — | Atlas = fal |
| Kling v2.6 Pro i2v | $0.07/s | — | — | $0.18/s (5 s = $0.90) | — | Atlas |
| Kling v3.0 Std | $0.084/s | — | $0.14/s (v3) | $0.144/s ($0.72 / 5 s) | — | Atlas |
| Seedance 2.0 i2v (480p) | $0.112/s | — | — | $0.12/s (480p $0.60 / 5 s) | — | Atlas |
| Hailuo 2.3 Standard | $0.28 per 6 s clip | — | ≈ $0.27 / 768p 6 s (Hailuo-02) | $0.36 / clip | — | fal ≈ Atlas |
| Veo 3.1 Lite | $0.05/s | — | — | $0.30 (min) | — | Atlas |

### 5.3 Lip-sync / talking avatar, per second of audio/output (list)

| Model | Atlas Cloud | kie.ai | fal.ai | MuAPI (derived) | Cheapest |
|---|---|---|---|---|---|
| **InfiniteTalk 480p** | $0.030/s | **$0.015/s** (3 credits/s, ≈ $0.005/credit; **max 15 s per generation**) | **$0.20/s** (page text; 720p double) | $0.040/s | **kie.ai** |
| InfiniteTalk 720p | not itemised | **$0.060/s** | $0.40/s | $0.080/s | kie.ai |
| Kling v2/2.6 Avatar Standard | $0.056/s | — | $0.0562/s | $0.070/s | Atlas ≈ fal |
| Kling Avatar Pro | $0.112/s | — | — | $0.075 min (v2 pro; dynamic) | Atlas |
| OmniHuman 1.5 | $0.12/s | — | (page unreadable) | ≈ $0.12/s (capped 30 s) | Atlas ≈ MuAPI |
| Wan 2.2 speech-to-video | — | — | — | $0.040/s | MuAPI (only) |
| Veed Lipsync (video→video) | $0.013/s | — | $0.0067/s ($0.4/min) | $0.0087/s | fal |
| Sync Lipsync v2/v3 | $0.22 (v3, unit unclear) | — | $0.05/s ($3/min) | ≈ $0.015/s | MuAPI |

Notes: Atlas `atlascloud/infinitetalk` price shown as "$0.03 per second", resolution 480p default or 720p; whether 720p costs more is **unverified**. kie.ai's 15 s cap fits our 8–10 s UGC speech but forbids longer scripts without chunking.

### 5.4 Cost per finished ad (list, 15% retry on the AI part)

Fixed costs: Product $0.023 · UGC $0.028 (longer render).

| Ad type | Cheapest configuration | AI part | **Total** | Fallback configuration | Fallback total |
|---|---|---|---|---|---|
| **1. Product Ad** (no AI video) | Azure TTS + Remotion | $0 | **$0.023** | — | — |
| **2. Product Ad + AI shot (5 s)** | Atlas Seedance 1.0 Pro Fast 480p ($0.05) | $0.0575 | **$0.080** | Atlas Wan 2.2 turbo ($0.10) → $0.138; Wan 2.2 ($0.15) → $0.195; MuAPI SPF 480p ($0.10) → $0.138 | $0.14–0.20 |
| **3. UGC (10 s avatar)** | kie.ai InfiniteTalk 480p ($0.15) | $0.1725 | **$0.200** | Atlas InfiniteTalk ($0.30) → **$0.373**; MuAPI ($0.40) → $0.488; Kling Avatar Std ($0.56) → $0.672 | $0.37–0.67 |
| UGC `pro` (720p InfiniteTalk, kie) | $0.60 | $0.69 | $0.718 | — | — |
| For comparison: Atlas "UGC" flow in the Atlas repo (Seedance 2.0 i2v 5 s/shot × 4 shots, 480p) | $0.112 × 20 s | | ≈ $2.24 + images | | |

Selling-price guidance (owner decides): at 250 DZD per USD (assumption, **owner to set**) the costs are ≈ 6 DZD / 20 DZD / 50 DZD; credit prices of 40 / 120 / 240 DZD give 5–7× gross margin before Chargily fees (1.5–4.5%) and are far below any agency price.

### 5.5 Free trial cost per user

| Trial design | Cost per user |
|---|---|
| Original plan: 1 generation of **any** type (UGC 10 s, kie) | **$0.20** worst case; Atlas-only fallback $0.37 |
| Recommended: 1 AI generation, UGC capped to **6 s** avatar (kie $0.09 ×1.15 + $0.028) | **$0.13** worst case ($0.235 with Atlas fallback) |
| Expected mix (20% Product / 50% +AI shot / 30% UGC 10 s) | **≈ $0.105** |
| Product Ad (no AI): watermarked renders | $0.023 each (limit 3) |
| Abuse bound | Global trial budget cap, e.g. $5/day ⇒ ≤ 38 UGC trials/day (≤ 62 shot trials) |

### 5.6 Cheapest provider per model (summary)

| Model | Cheapest (list) | Price | Runner-up |
|---|---|---|---|
| Seedance 1.0 Pro Fast | Atlas ≈ fal | ~$0.010/s (480p) | MuAPI 2× |
| Wan 2.2 | Atlas | $0.03/s (turbo $0.02) | kie/fal $0.04, MuAPI $0.06 |
| MiniMax H3 Max Turbo | Atlas = fal | $0.025/s | MuAPI ~2× |
| InfiniteTalk 480p | **kie.ai** | $0.015/s | Atlas $0.03, MuAPI $0.04, fal $0.20 |
| InfiniteTalk 720p | kie.ai | $0.06/s | MuAPI $0.08 |
| Kling Avatar Std | Atlas ≈ fal | $0.056/s | MuAPI $0.07 |
| Veed Lipsync | fal | $0.0067/s | MuAPI $0.0087, Atlas $0.013 |
| TTS (Azure neural) | Azure | ≈ $16 / 1M chars | Atlas-listed TTS (Gemini 2.5 Flash TTS $0.04, MiniMax Speech 2.6 Turbo $0.06; units not shown) |

### 5.7 Platform and fixed costs (monthly, estimates)

VPS 4 vCPU/8 GB ≈ $15–25 · Supabase Pro $25 (free tier OK for beta) · domain/Cloudflare ≈ $0–2 · Redis on the VPS $0 · Remotion $0 (≤ 3 people), else ≥ $100 · Chargily Starter plan free for the first 300 transactions/300k DZD, then 1.5–4.5% per transaction. **≈ $45–55/month before AI usage.**

### 5.8 Self-hosted GPU: when does it pay? (estimate)

Break-even: `GPU $/h ÷ (video-seconds produced per hour × utilisation) < API $/s`. With kie.ai at $0.015/s, a $0.5/h GPU must produce > 33 video-seconds/hour *at 100% utilisation* just to match; real InfiniteTalk 480p speed on one 24 GB GPU and real utilisation are **unmeasured**. Rule: revisit when monthly AI spend > ~$300 **and** utilisation can exceed ~50%; measure on one rented GPU for one day first. InfiniteTalk and Wan 2.2 are Apache-2.0 (check weights terms). Expect self-hosting to help mainly for Wan/Seedance-class shots and for removing the 15 s cap, not for beating kie.ai's InfiniteTalk price.

---

## 6. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Provider price changes / promos end (Atlas shows 70–95% promo "actual" prices) | Cost drift | Use list prices in config; weekly price check script reading Atlas `/api/v1/models` and MuAPI `/api/v1/models` (both public); cost log compares billed vs expected. |
| kie.ai reliability/terms unknown (smaller vendor) | UGC outage | Registry fallback to Atlas InfiniteTalk (2× cost); circuit breaker; Day-0 reliability run (30 jobs). |
| USD funding from Algeria | Blocks all AI | U0 task; crypto top-up on Atlas; test top-up of $25 before development. |
| Darija TTS quality | Core promise | Day-0 listening test; alternative voices; MSA/Darija text style option. |
| Lip-sync/face rejection by providers | Failed jobs | Use generated avatars only; refunds on failure. |
| Remotion licence at 4+ people | +$100/month | Tracked trigger; FFmpeg fallback. |
| AGPL | Source must be offered to users of the network service | Accepted by owner; keep public repo + `NOTICE`; no proprietary add-ons in the same codebase. |
| Atlas "MIT" claim without LICENSE | Do not copy code | Re-implement; optionally ask maintainers to add LICENSE. |
| Trial abuse | Cash drain | Daily budget cap, Turnstile, OTP later. |
| Content misuse (deceptive ads, real faces) | Legal/brand | Blocked-word list, avatar-only faces, AI labelling, ToS. |
| Chargily account approval time | Launch delay | Open account on day 1; manual top-up fallback is already in plan. |
| Edge-TTS ToS | Block | Not used in production. |

---

## 7. Decisions I made without asking (autonomous decision log)

1. Docs written in English (match existing files); chat summary in Arabic.
2. Canonical ClipForge = `xixihhhh/clipforge` (parent of all forks, 940★, same history).
3. 三千AI = **not open source** (proprietary licence) → excluded from evaluation beyond a note.
4. Atlas licence treated as **absent** (README claim without LICENSE) → no code copying.
5. List ("origin") prices used; promos ignored as requested.
6. BytePlus direct pricing: only Seedance 1.0 Pro was verifiable ($2.5/M tokens); Pro Fast figure uses the same token formula as fal ($1/M) and is marked unverified.
7. Free-trial redesign and daily budget cap added (not in the original plan).
8. Phone OTP stays post-launch, replaced at launch by budget cap + Turnstile.
9. Supabase Storage for MVP (R2 later).
10. New repo instead of reusing this fork; this repo becomes a reference/archive.
11. Gemini model name per Google pricing page on 2026-10-08: `gemini-3.5-flash-lite` — config value, verify at T0.2.

---

## 8. What the owner must prepare (summary; details and timing in `docs/TASKS.next.md`)

Accounts/keys: Atlas Cloud (+$25), kie.ai, Google AI Studio key, Azure Speech key (or alternative if Algerian card is refused), Supabase project, Cloudflare (DNS + Turnstile), VPS, GitHub repo, Chargily Pay merchant account, (later) OTP provider.
Network domains for Claude Code sessions: see `CLAUDE.next.md` §11.
Assets: ≥ 10 real product photos (incl. 5 "hard" ones), 8–10 generated avatar faces approved, 3 licensed music tracks, brand logo/colors, ToS text review, 3 sample ads for the landing page, price list in DZD.

---

## 9. Sources (retrieved 2026-10-08)

* Repos: https://github.com/xixihhhh/clipforge · https://github.com/AtlasCloudAI/atlas-marketing-studio · https://github.com/xiaolinbenben/terln-studio · https://github.com/AKA-liang/one-daihuo · https://github.com/witty-suckerpunch492/daihuo-jianshou · https://github.com/2961799660/sanqian_ai · https://github.com/Anil-matcha/Open-Generative-AI · https://github.com/harry0703/MoneyPrinterTurbo · https://github.com/ATH-MaaS/Pixelle-Video · https://github.com/calesthio/OpenMontage · https://github.com/MeiGen-AI/InfiniteTalk · https://github.com/Wan-Video/Wan2.2 (metadata via GitHub API)
* Atlas Cloud model catalogue with list/actual prices: https://api.atlascloud.ai/api/v1/models · InfiniteTalk page: https://www.atlascloud.ai/models/atlascloud/infinitetalk · Wan 2.2: https://www.atlascloud.ai/models/atlascloud/wan-2.2/image-to-video · Seedance Pro Fast: https://www.atlascloud.ai/models/bytedance/seedance-v1-pro-fast/image-to-video · Billing/top-up: https://www.atlascloud.ai/docs/billing/topup
* MuAPI model list and estimator: https://api.muapi.ai/api/v1/models · `POST /api/v1/models/{model}/estimate-cost`
* fal.ai model pages: https://fal.ai/models/fal-ai/infinitalk · https://fal.ai/models/fal-ai/wan/v2.2-a14b/image-to-video · https://fal.ai/models/fal-ai/kling-video/ai-avatar/v2/standard · https://fal.ai/models/fal-ai/sync-lipsync/v2 · https://fal.ai/models/veed/lipsync · https://fal.ai/models/fal-ai/bytedance/seedance/v1/pro/fast/image-to-video · https://fal.ai/models/fal-ai/minimax/hailuo-02/standard/image-to-video
* kie.ai: https://kie.ai/infinitalk · https://kie.ai/wan/v2-2
* BytePlus: https://www.byteplus.com/en/blog/seedance-1-0-pro-guide-api-pricing
* Remotion licence: https://www.remotion.pro/license
* Chargily: https://chargily.com/business/pay/pricing · https://dev.chargily.com/pay-v2/introduction
* Gemini pricing: https://ai.google.dev/gemini-api/docs/pricing
* Azure neural TTS ≈ $16/1M characters: web search result (Azure pricing page not fetchable) — verify on https://azure.microsoft.com/pricing/details/cognitive-services/speech-services/
