---
name: magnific-content-factory
description: >
  A 5-stage AI content pipeline that runs a full UGC ad campaign on top of Magnific (engine:
  the Magnific MCP Claude connector at mcp.magnific.com — OAuth, no API keys), focused on
  viral UGC content split evenly across 5 formats: UGC Entertainment (challenges), Street
  Interview, Unboxing, Product Review, ASMR. Button-driven UX — clicks, no typed commands.
  Stage 1: confirm the product + Magnific session folder, research trends in the product's
  niche on Instagram, TikTok, YouTube, and turn them into scripted ad ideas mapped to the 5
  formats. Stage 2: produce one HTML video content plan; user sets the count, the skill
  divides evenly across the 5 formats, every row carries a spoken script. Stage 3: generate
  videos one format batch at a time through Magnific's video_generate (Seedance 2.0 by
  default), with a mandatory dialogue-approval gate and credit-estimate gate before each
  batch, then auto-generate the image asset pack via images_generate (Nano Banana 2 default).
  Stage 4: schedule to Meta Ads via Meta MCP. Stage 5: render a cost comparison — Magnific
  credits spent (account_balance delta + creations history) vs traditional production cost.
  Trigger on a product image + content request, or phrases like 'create a Magnific campaign',
  'build a Magnific content plan', 'make UGC ads with Magnific', 'generate a batch of
  Magnific ads', 'run the Magnific content pipeline', 'schedule Magnific ads to Meta'.
user-invocable: true
metadata:
  tags: [magnific, content-factory, campaign, pipeline, orchestration, ugc, ad-campaign, seedance, content-plan, batch-generation]
  version: 2.0.0
  parent: arcads-content-factory
---

# Magnific Content Factory — Campaign Orchestration Pipeline

A 5-stage pipeline that runs a full ad campaign on top of **Magnific**:
**Research → Plan → Generate → Publish → Report.** This is the Magnific twin of the
Higgsfield/Arcads Content Factories — same workflow, same UGC-first 5-format mix, same
button-driven UX — but every video and image is generated through the **Magnific MCP
connector** (`https://mcp.magnific.com`). There is no HTTP-skill fallback: Magnific's MCP is
OAuth-based (no API keys), so the connector IS the engine. If it isn't connected, the skill
helps the user connect it before generating.

**Content focus is UGC-first.** Every campaign defaults to a heavy share of UGC-family
content — talking-head testimonials, honest reviews, sidewalk interviews, premium unbox
reveals, ASMR-style close-ups. This pipeline is **script-forward**: most videos carry an
actual spoken line the AI person delivers, and that script is approved before anything
generates.

**UX is button-driven.** Every clarifying question MUST be asked via AskUserQuestion with 2–4
concrete option buttons. Never ask a free-form "type your answer" question for navigation,
confirmation, or routing. Free-form typing is reserved ONLY for content the user must
originate (a product URL paste, a custom campaign name, or an edited script line) — and even
then, always offer a smart default the user can accept with one click. The user should drive
the entire pipeline by clicking option buttons.

---

## OPERATING PRINCIPLES — Magnific-specific (read first)

These principles are what make this skill a *Magnific* pipeline. They must always hold.

1. **The engine is the Magnific MCP connector** (`https://mcp.magnific.com`, streamable HTTP,
   OAuth 2.0 — the user signs in with their Magnific account once; no API keys, nothing to
   rotate). All generation, uploads, folders, history, and credit checks run through its
   tools. **Resolve the live tool set at runtime via ToolSearch** — Magnific MCP tools load
   per-session (typically prefixed `mcp__magnific__*` or `Magnific:*`), and the server's
   `tools/list` is the source of truth. The documented stable tool names to map by capability:
   - **Video:** `video_generate` · `video_models_list` / `video_models_show`
   - **Image:** `images_generate` · `images_upscale` · `images_crop` / `images_resize` ·
     `images_remove_background` · `images_models_list` / `images_models_show`
   - **Uploads:** `creations_request_upload` → `creations_upload` → `creations_finalize_upload`
   - **Polling/results:** `creations_wait` · `creation_status` · `creations_get` ·
     `creations_show` · `creations_search`
   - **Organization:** `folders_create` / `folders_list` / `folders_get` · `creations_move` ·
     `spaces_list` / `spaces_view`
   - **Credits/spend:** `account_balance` · `project_report`
   - **Consistent identity (premium):** `custom_references_create` / `custom_references_list`
     (Soul characters/styles)
   - **Audio (optional VO):** `audio_tts` · `audio_voices_list`

2. **If the connector is not connected, do not improvise.** Run
   `search_mcp_registry(["magnific","image generation","video generation"])` →
   `suggest_connectors`; if it's not in the directory, walk the user through the custom
   connector path: Settings → Connectors → **Add custom connector** → Name `Magnific`, URL
   `https://mcp.magnific.com` → complete the browser OAuth sign-in. Any paid Magnific plan
   works; MCP actions always consume credits (even for models "unlimited" in-app). Research
   stages (1–2) can run without the connector; Stage 3 cannot.

3. **Credit cost is ALWAYS an estimate — show it and gate on it before every batch.** Source
   the estimate in this order: (a) `account_balance` before/after a small probe batch or from
   prior batches this session (actual credits observed); (b) `video_models_show` /
   `images_models_show` per-model pricing metadata if exposed; (c) `project_report` history;
   (d) ask the user and remember for the session. Always call `account_balance` before a
   batch, show current balance + labeled estimate, cite the source, and never generate until
   the user confirms.

4. **The dialogue-approval gate is MANDATORY for any talking video.** For every format that
   speaks (UGC Entertainment, Street Interview, Product Review, and any Unboxing with
   voiceover), extract the spoken lines into a numbered block, time them against the clip
   duration, and get an explicit `yes` BEFORE the credit gate and BEFORE generation. Skip the
   gate only for silent flows (pure ASMR with no speech, silent product-hero b-roll).

**User-facing language rule (HARD).** The user is not a developer. Do NOT narrate technical
mechanics — no tool names, no upload/finalize plumbing, no "polling the creation," no
model-enum internals, no folder-ID caching. All of that runs silently. Send ONE clear stage
banner when each stage starts, in plain language:

| Stage | Banner |
|---|---|
| 1 | **🔍 Stage 1: Research & ideas — starting now.** I'm scanning what's trending this week in your product's niche across Instagram, TikTok, and YouTube, then turning the trends into 15+ scripted UGC ad ideas for your brand. |
| 2 | **🗂️ Stage 2: Content plan — starting now.** I'm building your full video content plan as a polished HTML document, with every ad mapped, dated, scripted, and ready to generate. |
| 3 | **🎬 Stage 3: Generating videos — starting now.** I'm producing your videos through Magnific, one format at a time. I'll show you the script and the cost estimate before each batch fires, so you stay in control. |
| 3 (image step) | **🖼️ Image asset pack — starting now.** I'm generating your social posts, hero banners, and product stills through Magnific. |
| 4 | **📅 Stage 4: Scheduling to Meta Ads — starting now.** I'm setting up your ad campaigns and scheduling everything across the calendar you approved. |
| 5 | **💰 Stage 5: Cost report — starting now.** I'm compiling the savings report — what you actually spent in Magnific credits versus what this volume would cost the traditional way. |

Between stages, send a brief "Stage [N] done — [deliverable]" line, then the next banner if
the user approved continuation. The user sees a clean narrative arc, not a tool log.

**Internal mechanics — keep silent.** Anywhere this skill says "upload the product photo,"
"create the session folder," "poll with creations_wait," "move the creation into the folder" —
those are AGENT-FACING. Execute them; do not narrate them. If the user asks "how does it work
under the hood," THEN explain.

---

## ONBOARDING — Always run this first (SINGLE-SHOT, NO PAUSES)

> ⚠️ CRITICAL: Ask EVERY onboarding question — A, B, C, AND D — in ONE single AskUserQuestion
> message at the start of the session. Do NOT ask them sequentially. The user answers
> everything in one reply, then the pipeline runs.
>
> If the user already provided a product image or URL in their first message, you still ask
> A/B/C in one message — but skip D, since the product is already in hand.

**Step A — Check Magnific is connected** (button question)
Options: "Yes — Magnific is connected" · "Not sure — check it for me" · "Skip — research only"
(If "Not sure," silently scan the tool list for Magnific MCP tools; if absent, run the
connector-install flow from Operating Principle 2 before Stage 3.)

**Step B — Confirm starting stage** (button question)
- "Stage 1 — Full pipeline (needs a product image)"
- "Stage 2 — Build content plan (I have a brief)"
- "Stage 3 — Generate now (I have a plan)"
- "Stage 4 — Schedule to Meta Ads (content is ready)"

If they provide a product image with no other context, default to Stage 1.

**Step C — Set video volume** (user-driven — ANY number; auto-divides across the 5 formats)
Buttons: "50 videos" · "100 videos (Recommended)" · "150 videos" · "200 videos" · *(Other →
type any number)*. Store as `[VIDEO_COUNT]`. Image pack count scales from this (Stage 3 Step 5).

**Compute the per-format split silently — DO NOT announce it at this step.** The instant
`[VIDEO_COUNT]` is locked, compute `floor(VIDEO_COUNT / 5)` per format internally. Do not show
the breakdown yet — it surfaces naturally inside the Stage 1 Viral Content Brief, framed as a
consequence of what the trend research showed, not as a config rule.

**Default video split (5 UGC formats, even):** `per_format = floor(VIDEO_COUNT / 5)`. If
`VIDEO_COUNT` doesn't divide evenly by 5, distribute the remainder one-per-format starting
from format 1.

| Total | Entertainment | Street Interview | Unboxing | Product Review | ASMR |
|---|---|---|---|---|---|
| **50** | 10 | 10 | 10 | 10 | 10 |
| **100** (Recommended) | 20 | 20 | 20 | 20 | 20 |
| **150** | 30 | 30 | 30 | 30 | 30 |
| **200** | 40 | 40 | 40 | 40 | 40 |
| **17** (custom, rem 2) | 4 | 4 | 3 | 3 | 3 |

**Step D — Get the product (image or URL) IN THE SAME MESSAGE**
> "Attach your product image to this message OR drop a URL — that's all I need to start."

If a product image is already attached, skip D. If the user picks Stage 3 or 4 in Step B, swap
D with: "Drop your existing content plan file or paste it here."

**Single-message rule:** Send the AskUserQuestion (A + B + C as buttons) and the product-attach
prompt in the SAME message. Once the user clicks and attaches the product, proceed straight to
the chosen stage with NO extra confirmation. Never ask A/B/C/D separately.

---

## MAGNIFIC MCP GROUND TRUTH

Always honor these. The authoritative source is the connector's live `tools/list` and the
per-model metadata from `video_models_list` / `video_models_show` and `images_models_list` /
`images_models_show` — query them at session start. The snapshot below is what the pipeline
relies on.

### Primary video path
All video models go through **`video_generate`** with a model selection. Reference media (the
product photo) is uploaded once via the upload triple
(`creations_request_upload` → `creations_upload` → `creations_finalize_upload`) and the
resulting creation is passed as the image/reference input. Magnific also supports **auto
mode** (server picks the model) — this pipeline pins the model explicitly instead.

### Video model capability (snapshot — re-read `video_models_list` for live catalog + pricing)

| Model | Speech? | Image-to-video | Best for |
|---|---|---|---|
| **Seedance 2.0** (DEFAULT) | ✅ (dialogue embedded in prompt, audio on) | ✅ | **The UGC workhorse — talking testimonials, reviews, interviews, ASMR** |
| Google Veo 3.1 | ✅ | ✅ | Ultra-real cinematic talking head |
| Kling 3.0 Omni | limited | ✅ | Silent product motion / b-roll |
| Others (catalog updated daily) | varies | varies | Check `video_models_show` before proposing |

**Default for ALL 5 formats: Seedance 2.0.** It does speech + audio + product-grounded
image-to-video + 9:16 in one call. Only deviate on explicit user request — and before routing
anything to a non-default model, confirm its capabilities and price via `video_models_show`.

### Seedance 2.0 defaults (the default engine)
- Aspect ratio: **9:16** for UGC/social (default), 16:9 for landscape.
- Duration: keep every clip **≤15s**; longer ideas split into a 2-clip sequence.
- Audio on for any talking or ASMR clip; embed dialogue in the prompt as
  `Dialogue: "..."` or `She says: "..."`.
- Reference: the uploaded product photo as the image/reference input. Auto-upscale via
  `images_upscale` first if the longest side < 1024px.
- Credits are consumed per generation via the user's Magnific plan — tighten flagged prompts
  rather than blind-retrying failures.

### Script length → duration (auto-select, ~2.5 words/sec, round up)
| Script length | Seedance duration |
|---|---|
| 1–8 words | 4–5s |
| 9–15 words | 6–8s |
| 16–25 words | 9–12s |
| 26–35 words | 13–15s |
| **36+ words** | **Too long** — split into a 2-clip sequence |

### Magnific-signature path — Soul characters (consistent AI people)
Magnific's standout capability for UGC is **custom references**: train a consistent face,
product, or style once via `custom_references_create` and reuse it across every shot. Offer
this as a **premium option for Product Review and Street Interview** when the user wants the
same recognizable "creator" across many videos instead of prompt-described people. Training
costs credits — gate it like any batch. Optional add-on: generate a voiced narration track
with `audio_tts` (pick a voice via `audio_voices_list`) for layering in edit.

### Image generation
**`images_generate`**. Default model **Nano Banana 2** (alternatives per catalog: Seedream 5,
Google Imagen 4, Recraft V4, GPT 2 — confirm via `images_models_list`). Aspect ratios 1:1 /
16:9 / 9:16. Product photo passed as reference. `images_generate_svg` / `images_to_svg` exist
for vector needs but are out of scope for the default pack.

### Session organization (always)
At the start of any generating session: `folders_list` → reuse or `folders_create` the dated
folder **"Magnific Content Factory - YYYY-MM-DD"**; after each generation completes, use
`creations_move` to file it there. Every asset is findable under that folder (and browsable
later via `creations_search`).

### What this pipeline must NOT propose
- ❌ A single Seedance clip longer than 15s (split into a sequence).
- ❌ Reliable multi-character coordinated dialogue with consistent identities across cuts in
  one clip (use a Soul character or split into separate clips).
- ❌ On-screen rendered text/captions (add in edit — see the no-text rule in Stage 3).
- ❌ Model capabilities you haven't confirmed via `video_models_show` for non-default models.

### Spend tracking (silent, every session)
Call `account_balance` at session start and after each batch; the delta is the actual spend.
Record per-batch: format, model, clip count, durations, balance-before, balance-after.
`project_report` and `creations_search` (filtered to the session folder/date) back this up.
Stage 5 reads these numbers for the real spend.

---

## 5 UGC FORMAT DEFINITIONS → MAGNIFIC MODEL MAPPING

Every campaign distributes evenly across these 5 formats. Each maps to a Magnific model
config. When generating idea cards (Stage 1) and rendering (Stage 3), vary the **concept
seed** within each format so no two videos are the same concept.

> Default model is **Seedance 2.0** for all five. The columns below tune audio, speech, and
> the dialogue gate per format.

### Format 1 — UGC Entertainment
- **Vibe:** challenge / dare / entertainment-first. The product is the punchline.
- **Model:** Seedance 2.0 · audio on · 9:16 · product photo as reference
- **Speech:** yes (short punchy lines) → **dialogue gate applies**
- **Escalation:** if a skit genuinely needs >15s, split into a "(1/2)"+"(2/2)" sequence, or
  check `video_models_show` for a longer-duration model before proposing it
- **Concept seeds:** blind taste "guess which one"; "I'll give you $ if you try it" street dare;
  "will it pour?" absurd pour; product-flying-into-frame deadpan reaction; failed-dare recovery.

### Format 2 — Street Interview
- **Vibe:** sidewalk stranger interviews, "real people," high-trust.
- **Model:** Seedance 2.0 · audio on · 9:16 · two-person framing (interviewer + stranger)
- **Speech:** yes (both voices) → **dialogue gate applies**
- **Premium option:** Soul character (custom reference) for a consistent recurring interviewer
- **Concept seeds:** "what's your favorite [niche] right now?" → pulls product from bag; "sing
  for the product"; "rate this out of 10"; "try this on a hot day" first-sip face; blind
  opinion → brand reveal.

### Format 3 — Unboxing
- **Vibe:** premium reveal. Hands, packaging, the moment of discovery.
- **Model:** Seedance 2.0 (i2v from product photo) · audio on (crinkle/reveal sound)
- **Speech:** minimal or none. If voiceover → dialogue gate; if pure ASMR reveal → skip gate.
- **Concept seeds:** trio variant reveal in pastel paper; single-bottle slow ribbon-pull;
  subscription-box drop with brand note; premium gift-set unbox; hangtag macro series.

### Format 4 — Product Review
- **Vibe:** honest talking-head. Product in hand, ingredients read aloud, ranking.
- **Model:** Seedance 2.0 · audio on · 9:16 (the 9-layer UGC formula)
- **Speech:** yes (the core testimonial) → **dialogue gate applies**
- **Premium option:** **Soul character review series** — train one consistent creator via
  `custom_references_create` and have "them" front every review. This is the most
  Magnific-native format; offer it here first.
- **Concept seeds:** two-ingredient test (read label, raise eyebrow, sip); "cold side of the
  fridge" ranking; side-by-side vs generic competitor; "I tried this for 7 days" diary review;
  final flavor ranking lined up on camera.

### Format 5 — ASMR
- **Vibe:** sound-led close-ups. No talking, audible product handling.
- **Model:** Seedance 2.0 (i2v) · audio on · **no dialogue → skip the gate** · 9:16
- **Setting:** intimate, low-noise (kitchen / bathroom / bedroom). Never street/gym.
- **Concept seeds:** macro cap-unscrew + glug pour into iced glass; condensation-bead slide then
  open; spoon-clink + ice-drop; bottle-on-marble tap-and-rotate; gentle two-bottle clink, no music.

> Generation note: Formats 1, 2, 4 are talking Seedance clips (dialogue gate). Format 3 is a
> Seedance i2v reveal (gate only if voiced). Format 5 is a silent-speech Seedance i2v with
> audio on for ambient sound (no gate). All five default to Seedance 2.0.

---

## STAGE 1 — Trend Research & Viral Idea Generation

> ⚠️ MANDATORY. All research from live web searches. Every idea validated against live
> Magnific capability (≤15s Seedance, 9:16, product-grounded i2v, scripted speech). UGC-first.

### Step 0 — Confirm Magnific is ready + session folder *(internal — silent)*
> Send the Stage 1 banner BEFORE this. Do not narrate.
1. Confirm Magnific MCP tools are present in the tool list (ToolSearch if needed). If absent
   and the user chose a generating stage, run the connector-install flow before Stage 3.
2. Call `account_balance` — cache the opening balance for Stage 5.
3. Create/reuse the dated session folder via `folders_list` / `folders_create`; cache its ID.
4. `video_models_list` + `images_models_list` — confirm Seedance 2.0 and Nano Banana 2 are
   live and note any pricing metadata.

### Step 1 — Identify the product & niche *(auto-detect — DO NOT ask the user to confirm)*
Auto-derive everything silently from the product image and/or URL. No clarifying card.
1. **From the image:** category, variants/SKUs, packaging palette, demographic cues.
2. **From the URL (if given):** `web_fetch` → product name, official category, brand voice, claims.
3. **Niche keyword** in plain English (e.g. "natural fruit juice", "skincare serum").
4. **Target market:** default "Global / English-speaking" unless packaging/page says otherwise.
5. **Primary goal:** default "Mixed (awareness + conversion)" unless the user stated otherwise.
6. **Model routing:** Seedance 2.0 for all five formats by default (see mapping above).

**User-facing output — one short status line, NOT a question.** Frame the format choice as an
observation of what's viral right now, not a system rule. Do NOT mention "the 5 formats" or
"even split" as an enumeration:
> "Got it — looks like a [niche] with [variants]. I'll target a [market] audience and lean
> into what's moving in this category right now — challenge-style clips, sidewalk stranger
> interviews, premium unboxings, honest scripted reviews, and audio-first ASMR pours."

Then proceed straight to Step 2. **No AskUserQuestion here.** If the user wants to change niche/
market/goal/model, they'll say so in chat — apply it and restate the summary once.

### Step 2 — Run mandatory trend research *(internal — execute silently; no query enumeration)*
Replace `[niche]` and `[current month year]`:
1. `[niche] TikTok trending videos this week [current month year]`
2. `viral [niche] content Instagram Reels [current month year]`
3. `[niche] YouTube Shorts trending [current month year]`
4. `[niche] brand content going viral [current month year]`
5. `top [niche] ads performing Meta [current month year]`
6. `[niche] UGC content trend [current month year]`
7. `[niche] hooks that stop the scroll [current month year]`
8. `[niche] competitor brands social media strategy [current month year]`

Extract per result: format, hook patterns, visual style, brands, engagement.

### Step 3 — Fetch at least 2 source pages (in parallel)
`web_fetch` the 2 most useful URLs. Pull specific hook lines and creative patterns — these
become the **spoken scripts** for the seed ideas.

### Step 4 — Synthesize the Viral Content Brief (UGC-first, script-forward)
Translate every viral trend into a producible Magnific idea. **Reveal the format split
naturally inside the brief** in a "Recommended Content Mix" section that names per-format
counts as a consequence of the trends ("based on what's winning this week, here's the mix") —
never as a config rule. This is the only place the numeric breakdown appears before generation.

For every idea, REQUIRED fields:

```
N. **[Title]**
- Format: [1–5 from the 5 UGC Format Definitions]
- Model: Seedance 2.0 (default) | Veo 3.1 (cinematic) | Soul character (premium consistent creator)
- Duration: [4–15s, set by script length]
- Aspect ratio: 9:16 (default) | 16:9
- Audio: [on/off]
- Persona/actor: [short description, or "Soul character" if custom reference flow]
- Scene prompt: [≤2 sentences, respects 15s + product-grounded i2v bounds]
- Spoken script: [the EXACT words the AI person says — or "(silent — ASMR)"]
- Social post caption: [used when uploading to TikTok/IG — NEVER rendered on-screen in the video]
- Inspired by: [specific trend/competitor from research]
- Why viral now: [specific reason tied to research]
```

### Producibility self-check (before adding any idea)
1. **Duration ≤15s?** If no, split into a 2-clip sequence.
2. **Model routing matches the format?** (Talking → Seedance; consistent creator → Soul
   character; silent product motion → Kling 3.0 Omni.)
3. **Product-grounded shot?** The model must support image-to-video with the product photo —
   confirmed for Seedance 2.0; check `video_models_show` for anything else.
4. **Script fits the duration?** Word count ÷ 2.5 ≤ chosen seconds.
5. **Forbidden patterns?** on-screen text, multi-character cross-cut dialogue, unconfirmed
   model capabilities.
6. **UGC-first ratio honored?** ~All five formats are UGC-family by design.

### Brief structure
Trends table · Competitor table · Hook patterns (verbal) · Format momentum (producible) ·
**Recommended Content Mix** (per-format counts) · **15+ seed ideas** with spoken scripts.

### Step 5 — Approval (button-driven)
Present the brief, then AskUserQuestion:
> "Brief is UGC-first and producible in Magnific. What next?"
> - "Looks good — proceed to Stage 2 (Recommended)"
> - "Add more UGC ideas first"
> - "Swap or re-script some ideas"
> - "Adjust the mix ratios"

Only on click does the skill proceed.

---

## STAGE 2 — Video Content Plan

### Goal
One HTML deliverable: the **Video Content Plan** — `[VIDEO_COUNT]` entries, every row mapped to
a model, a duration, and a **spoken script**. Image assets are auto-generated at the end of
Stage 3.

### Steps

**1. Confirm campaign details (single AskUserQuestion — all buttons, smart defaults)**
- Campaign name → "Use auto: [Brand] Campaign [current month year]" / "Different name"
- Date range → "Next 30 days (Recommended)" / "Next 60 days" / "Next 90 days" / "Custom"
- Variants → multiSelect buttons listing every variant detected on the product

Do NOT ask the user to "confirm the format breakdown" — it was revealed in the Stage 1 brief.
Goal and brand colors are auto-derived; only ask if the user raises them.

**2. Generate the Video Content Plan HTML — 5-format even split**
- `[VIDEO_COUNT]` videos distributed `floor(N/5)` per format, remainder from format 1.
- Every row: #, Date, **Format (1–5)**, **Model**, **Duration**, Aspect Ratio, Audio (on/off),
  Persona/actor, Scene prompt, **Spoken script (or "silent — ASMR")**, **Social post caption
  (upload metadata only — NEVER rendered in the video)**, Goal.
- Group rows by **format bucket** in this order: (1) UGC Entertainment → (2) Street Interview →
  (3) Unboxing → (4) Product Review → (5) ASMR. This grouping powers Stage 3's per-batch gates.
- Within each format, vary the concept seed so no two videos are the same.
- Distribute dates evenly across the window; interleave formats day-to-day.
- Multi-clip sequences (a >15s idea split) listed as "(1/2)" and "(2/2)".

**3. Save the video plan**
`/mnt/user-data/outputs/[brand]-magnific-video-plan.html`. Present the plan and ask for
feedback via button before Stage 3.

---

## STAGE 3 — Generate Content via Magnific

> ⚠️ CRITICAL: Two gates before EACH format batch — the **dialogue-approval gate** (talking
> formats) and the **credit-estimate gate**. Never run a batch without both confirmations.

### Goal
Generate the videos through the Magnific MCP, batch-by-batch, with permission + dialogue +
cost gates.

### Steps

**1. Prepare the product reference + session folder** *(internal — silent)*
- Upload the product photo via the upload triple (`creations_request_upload` →
  `creations_upload` → `creations_finalize_upload`); cache the creation ID for use as the
  reference in every call. If the longest side < 1024px, `images_upscale` it first.
- Ensure the dated session folder exists; cache its ID. A friendly "Getting your product
  ready…" line is fine; no tool names.
- Call `account_balance`; cache balance-before for this stage.

**2. Per-batch processing order (REQUIRED — one format at a time)**

| Order | Format | Model / config | Gate(s) |
|---|---|---|---|
| 1 | **UGC Entertainment** | Seedance 2.0, audio on, 9:16 | dialogue + credit |
| 2 | **Street Interview** | Seedance 2.0, audio on, two-person | dialogue + credit |
| 3 | **Unboxing** | Seedance 2.0 i2v, audio on | dialogue (if voiced) + credit |
| 4 | **Product Review** | Seedance 2.0 (or Soul character) | dialogue + credit |
| 5 | **ASMR** | Seedance 2.0 i2v, audio on, no speech | credit only (no dialogue) |

Each batch is `floor(VIDEO_COUNT / 5)` videos (remainder from format 1).

**3. Dialogue-approval gate (talking formats only)** — BEFORE the credit gate.
Extract every spoken line in the batch into a numbered block with beat labels, mark silent
beats, count words vs duration, and get explicit `yes`:
```
📝 Dialogue script (please confirm before I generate)
  1. [HOOK]    "..."
  2. [DEMO]    (silent beat — no dialogue)
  3. [VERDICT] "..."
Total spoken words: ~28 | Target duration: 15s | Fits at natural pace: ✅
Approve this dialogue? (yes / edit / rewrite)
```
Never assume approval from earlier gates. Skip only for ASMR (format 5) and any silent clip.

**4. Credit-estimate gate** — BEFORE generating. Call `account_balance` (show the current
balance), pull the per-model estimate from observed spend this session, model metadata, or
`project_report`, and gate via AskUserQuestion:
> "Ready to generate the **[N] [format]** videos? (Seedance 2.0, 9:16, audio [on/off]).
> Current balance: **[B] credits**. Estimated cost: **~[X] credits** ([cite source]).
> ⚠️ Estimate only — Magnific charges per generation based on model and resolution."
> - **"Yes — generate all [N]"**
> - **"Start with [3] for a quality check first (Recommended)"**
> - **"Skip this batch for now"**
> - **"Change settings before generating"**

Wait for the click. ONLY THEN fire generation.

**5. Generate the batch (Magnific MCP)** *(internal — silent)*
For each row: build the prompt from the scene + product + style cues, embed the approved
`Dialogue: "..."` for talking clips, set the model (default Seedance 2.0), duration (from
script length), aspect ratio (9:16), audio flag, and the cached product reference. Call
`video_generate` once per video (one call per requested variation — no fan-out inside one
call), then poll each via `creations_wait` / `creation_status` until finished or failed. On
completion, `creations_move` the asset into the session folder and `creations_show` a batch
preview inline where the client supports it. Record model, count, and the `account_balance`
delta for Stage 5.

**Prompt template — NO on-screen text, EVER:**
```
[Scene prompt]. Product: [name], [color/packaging/label detail from the image].
[For talking clips:] Dialogue: "[approved script line]".
Style cues: [per format — "authentic handheld iPhone feel, natural daylight" for UGC;
            "intimate ASMR close-up, no music, audible product handling" for ASMR].
Negative: no text overlay, no captions, no subtitles, no on-screen text, no watermarks,
no lower-third, no graphic typography. Clean image only.
```
> ⚠️ The "Social post caption" field is upload metadata only. NEVER put it in the generation
> prompt as an overlay. The video must contain zero rendered text — captions are added in edit.

After each batch: show results, then AskUserQuestion ("Generate next batch" / "Re-do this one" /
"Pause here"). No typing to advance.

**6. Image asset pack — auto-generated via `images_generate` (last step)**
After all video batches are approved, fire ONE credit gate, then generate the pack.
**Pack count = `floor(VIDEO_COUNT / 5)`.** Breakdown: **40% Social · 20% Hero · 20% With-People ·
20% Without-People.**

| Video count | Pack total | Social | Hero | With-people | Without-people |
|---|---:|---:|---:|---:|---:|
| 50 | 10 | 4 | 2 | 2 | 2 |
| **100** | **20** | **8** | **4** | **4** | **4** |
| 150 | 30 | 12 | 6 | 6 | 6 |
| 200 | 40 | 16 | 8 | 8 | 8 |

Allocation: `social=floor(total×0.4)` · `hero=floor(total×0.2)` · `with_people=floor(total×0.2)`
· `without_people=total − the rest`.

> Gate: "Videos done. Ready to generate the image asset pack — [N] images via Magnific (Nano
> Banana 2)?" → "Yes — generate all [N] (Recommended)" / "Skip with-people" / "Skip
> without-people" / "Skip image pack entirely".

Generate via `images_generate`, model Nano Banana 2, product photo as reference, aspect ratio
1:1 (social) / 16:9 (hero) / mixed (people). Batch image generation is supported — use it
where the tool contract allows. Run a **mandatory image QA**: inspect each output for bad
hands/faces/label distortion; regenerate with a corrected prompt up to 2 retries. File
everything into the session folder (`creations_move`), save local copies to
`/mnt/user-data/outputs/[brand]-asset-pack/` with descriptive names
(`social-01-watermelon-kitchen.png`, `hero-02-lifestyle.png`, etc.). Show files, then offer:
"All set — proceed to Stage 4" / "Re-do specific assets" / "Pause here".

### Standalone image-pack mode (skip the videos)
When the user asks for "just the image pack" / "image assets only": confirm the connector,
upload the product photo, auto-detect details silently, pick pack size via a single
AskUserQuestion ("10 / 20 (Recommended) / 30 / 40"), run the same 40/20/20/20 generation via
`images_generate`, save to the asset-pack folder, and end (no Stage 4/5 unless asked).

**7. Failure handling**
If a generation fails (moderation flag, server error, rate limit), record the failed row IDs,
tighten any flagged prompt (don't blind-retry — each attempt consumes credits), and offer
"Retry / Skip / Pause" buttons for the subset.

---

## STAGE 4 — Schedule & Publish to Meta Ads

Unchanged from the sibling pipelines — Magnific only changes how the creatives were made, not
how they're scheduled.

**1a. Meta MCP connection check — FIRST question.**
> "Quick check before we schedule — is your Meta MCP connected to Meta Ads?"
> - "Yes — Meta MCP is connected (Recommended)"
> - "Not connected — help me install it now"
> - "Skip live scheduling — give me an exportable calendar instead"

- **Connected →** proceed to 1b.
- **Not connected →** `search_mcp_registry(["meta ads","facebook ads","meta marketing"])` →
  `suggest_connectors` so an install card renders; include the Settings → Connectors fallback
  and the docs link; re-ask 1a after install.
- **Skip →** export `[brand]-content-calendar.csv` (Date · Time · Format · Model · Video file ·
  Image file · Social post caption · Goal · Notes) to outputs; skip to Stage 5.

**1b. Campaign details (only if connected — single AskUserQuestion, all buttons)**
Objective ("Awareness"/"Traffic"/"Conversions"/"Mixed") · Budget tier · Date range ("Match the
plan dates (Recommended)" / "Next 30 days" / "Custom"). Ad Account auto-detected; only ask if
multiple.

**2. Content calendar review (button approval)** — "Schedule looks good?" → "Yes — schedule
everything" / "Yes — week 1 only" / "Adjust dates first".

**3. Create campaigns via Meta Ads MCP** — campaign (objective) → ad sets (targeting via
AskUserQuestion buttons) → upload the Magnific videos/images as creatives (download from their
creation URLs first if the Meta MCP needs local files) → schedule per plan dates.

**4. Confirm scheduling** — weekly summary table. "Continue to Stage 5?" → "Yes — render the
cost report (Recommended)" / "Pause a campaign" / "Generate more content" / "Skip Stage 5".

---

## STAGE 5 — Cost Comparison Report

### Goal
Compare **actual Magnific credit spend** for this campaign against the **traditional
production cost** of the same volume. Output savings ratio + time savings, save HTML, present.

### Steps

**1. Pull live spend from Magnific.** Source in this order: (a) the session's
`account_balance` deltas per batch (opening balance − closing balance = total spend; per-batch
deltas were recorded during Stage 3); (b) `project_report` for a usage overview; (c)
`creations_search` filtered to the session folder/date to count assets per model and format.
Sum credits per model and per asset type (videos by format, images). Convert credits → USD
only if the user's plan rate is known (ask once via buttons if they want USD; otherwise
present credits-only and note it's plan-dependent). Label every number an estimate.

**2. Apply the traditional production cost model (2026 industry-average midpoints, low–mid–high).**

| Asset type | Low | Mid | High |
|---|---:|---:|---:|
| UGC creator video (TikTok/Reels) | 250 | 750 | 1,500 |
| Product Review video | 300 | 900 | 2,000 |
| Street Interview / man-on-street video | 400 | 1,000 | 2,200 |
| Unboxing video | 300 | 800 | 1,500 |
| ASMR product video | 300 | 850 | 1,800 |
| Social media post (1:1 lifestyle still) | 100 | 250 | 500 |
| Hero banner (16:9 cinematic) | 1,000 | 2,500 | 5,000 |
| Product photoshoot WITH people | 500 | 1,500 | 3,000 |
| Product photoshoot WITHOUT people | 200 | 700 | 1,500 |

Time-savings benchmark: 100 mixed Magnific videos ≈ 1–4 hrs render vs 4–12 weeks traditional;
20-image pack ≈ minutes vs 1–3 weeks photographer + retouch; scheduling ≈ minutes via Meta MCP.

**3. Compute savings.**
`traditional_{low,mid,high} = Σ(count × rate)` · `magnific_usd = total_credits × user_rate` (if
known) · `savings_pct_mid = 1 − magnific_usd / traditional_mid` (cap 99.99%) · time savings in
weeks.

**4. Render the HTML report** → `/mnt/user-data/outputs/[brand]-cost-comparison.html`.
Sections: (1) hero number card ("[Brand] delivered for **$X** instead of **$Y–$Z** — saved
**N%** and **W weeks**"); (2) volume summary; (3) Magnific spend breakdown (credits per
format/model, USD if rate known, **labeled estimate from balance deltas + creation history**);
(4) traditional cost breakdown (low/mid/high); (5) side-by-side bars (pure HTML/CSS); (6) time
savings panel; (7) methodology footer (traditional = 2026 industry-average estimates, not a
quote; Magnific USD = credits × user plan rate; credits measured as the account-balance delta
across the session, so they are the ground truth).

**5. Present (button confirm)** — "Cost report ready. What next?" → "Done — close the pipeline
(Recommended)" / "Email this report to my team" / "Adjust the traditional-cost rate card and
re-render" / "Run the pipeline again for another product".

---

## General Guidelines

- **Engine = the Magnific MCP connector, always.** Resolve live tool names via ToolSearch at
  session start; the server's `tools/list` is authoritative. Never hard-code prefixes; map by
  capability using the stable names in Operating Principle 1. If the connector is missing, run
  the install flow — never simulate generation, never fall back to scraping or an API key.
- **Button-driven rule (HARD):** every clarifying question is an AskUserQuestion with 2–4 button
  options and a smart default. Only product upload, URL paste, custom name, and edited script
  lines may be typed.
- **No-pause rule:** bundle every clarifying question into a single AskUserQuestion call.
- **5-format split (HARD):** distribute evenly across UGC Entertainment, Street Interview,
  Unboxing, Product Review, ASMR — `floor(VIDEO_COUNT / 5)` each, remainder from format 1.
- **Default model = Seedance 2.0** for all five formats; default image model = Nano Banana 2.
  Only deviate on explicit request (Veo 3.1 for cinematic realism, Kling 3.0 Omni for silent
  b-roll, Soul characters for consistent creators) — and confirm capability + price via the
  model catalog tools first.
- **Dialogue gate (HARD):** approve spoken scripts before the credit gate and before generation
  for every talking format. Skip only for ASMR/silent clips.
- **Credit gate (HARD):** call `account_balance`, show the balance and a labeled estimate (cite
  the source), and wait for confirmation before EVERY batch and the image pack. Remember: MCP
  actions always consume credits, even on plans with "unlimited" in-app generations.
- **Producibility rule:** every idea producible within Magnific bounds (≤15s Seedance, 9:16,
  product-grounded i2v on a confirmed-capable model) or split into a sequence.
- **No on-screen text:** generated videos carry zero rendered text; captions are added in edit.
  The plan's caption field is upload metadata only.
- **Track every batch's balance delta** — Stage 5's spend numbers come from `account_balance`
  deltas backed by `project_report` and `creations_search`.
- **File everything:** every finished creation is moved into the dated session folder so the
  campaign is browsable in Magnific afterwards.
- **Visual identity consistency:** same product reference image, palette, and tone throughout;
  rotate personas within a batch for feed diversity — or pin one Soul character when the user
  wants a recurring creator.
- **Failure handling:** tighten flagged prompts (every attempt costs credits — no blind
  retries); record failed IDs; offer "Retry / Skip / Pause" buttons.

---

## Source acknowledgment

Workflow mirrors the `higgsfield-content-factory` / `arcads-content-factory` pipelines
(Research → Plan → Generate → Publish → Report, UGC-first 5-format even split, button-driven
UX). The generation engine is swapped to the **Magnific MCP Claude connector**
(`https://mcp.magnific.com`, OAuth, streamable HTTP). Ground truth for tools, models, and
pricing is the connector's live `tools/list` plus `video_models_list` / `images_models_list`;
the snapshot in this skill reflects the Magnific MCP docs as of July 2026.
