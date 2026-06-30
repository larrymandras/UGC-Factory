# Ástríðr Port — UGC-Factory (Track B design)

> Approved design seed for a GSD phase in `astridr-repo`. Source material = this repo's
> `skill/` (run-pipeline.md + 5 frameworks + 15 style skills). Track A (Claude Code install)
> is already done; this doc covers only the Ástríðr port.

## Goal
Give Ástríðr the same UGC-ad capability the Claude Code `/ugc-factory` skill provides, owned by
the **Hildr** agent. Ástríðr cannot execute a Claude-Code skill (it uses `AskUserQuestion`,
slash commands, `@frameworks/...` loads, interview HITL), so this is a faithful *port*, not a copy.

## Shared engine (already resolved)
Both agents drive the **same remote Higgsfield MCP**: `https://mcp.higgsfield.ai/mcp`
(streamable-http, hosted — like the supabase/github MCPs). Ástríðr-in-Docker reaches the external
HTTPS endpoint fine (no `host.docker.internal` problem; unlike the notebooklm sidecar this is
truly remote). ffmpeg is already in `astridr-repo/Dockerfile`, so in-container stitching works.

## Four deliverables

> **2026-06-20 — folded in a stack-fit skill review (Codex-validated).** The upstream skill is
> generic, portable, and deliberately *stateless*; deliverables #2 and #4 and the guardrails below
> close gaps that matter for Larry's stack specifically (cost-consciousness, asset reuse, review +
> resumable operation). Findings F1–F6 referenced inline.

### 1. MCP wiring — `config/mcp-servers.yaml`
Add a streamable-http entry mirroring the `notebooklm` shape:
```yaml
  - name: higgsfield
    transport: streamable-http
    url: https://mcp.higgsfield.ai/mcp
    # auth: confirm the http MCP config model (engine/config.py) supports an auth header;
    # token via SecretRef required=False (optional ref must not abort bootstrap fail-fast).
```
- **Open item:** the current `mcp-servers.yaml` examples show `env:` (stdio) and bare `url:`
  (notebooklm is unauthenticated on the internal net). Higgsfield is an *authenticated remote* MCP.
  Verify how Ástríðr passes a bearer/OAuth token to an http MCP server; extend the Pydantic config
  model if needed. This is the main unknown to resolve in discuss/plan.
- **Secret lifecycle:** token injected via `.env` is baked at container **create** — use
  `docker compose --profile prod up -d astridr` (recreate), NOT `restart` (stale value).

### 2. Tool-kit — `config/classification.yaml`
Register a `ugc-factory` kit (kit defs live in classification.yaml, **not** tool-kits.yaml) grouping
the Higgsfield tools + the ffmpeg media tool, granted to **Hildr**. The upstream skill's
`allowed-tools` list is **incomplete** — carry the full set, not the original's:
- **Core (already in skill):** `generate_image`, `generate_video`, `show_reference_elements`,
  `media_upload`, `media_confirm`, `job_status`, `show_generations`, `models_explore`.
- **Add `media_import_url`** (F3) — the skill documents the web-URL product path
  (`character-creation.md`) but omits the tool from `allowed-tools`; the port must include it.
- **Add `balance` + `show_plans_and_credits`** (F2) — needed for the pre-generation cost estimate
  (see Guardrails); the skill has no cost tool at all.
- **Add `generate_audio` + `text2speech_v2` + `list_voices`** (F4 — re-scoped after the 2026-06-30
  test, see Track A validation). The pipeline produces a **silent** ad, and Seedance's *native* audio
  (`generate_audio:true` on `generate_video`) **mangles spoken words** ("bottle" → "bossel"/"bottler"
  across two takes). Dialogue must come from a **scripted TTS track** (`text2speech_v2`, e.g.
  ElevenLabs voice via `list_voices`), never Seedance native TTS. (Larry also has the `acestep` music
  MCP for a music bed.) **Lip-sync caveat:** no Higgsfield-native tool lip-syncs a scripted VO to
  face-on dialogue — see the validation verdict below; this drives a shot-selection rule in #3.
- **Add `virality_predictor`** (F5) — optional closing quality pass on the final cut.

### 3. Ported pipeline — playbook/prompt for Hildr
Convert `skill/tasks/run-pipeline.md` + the 5 `skill/frameworks/*` + 15 `skill/styles/*/SKILL.md`
into an Ástríðr-native playbook the creative agent loads. Adaptations:
- **Interview → param-driven single-shot.** Brief (product, offer, topic, persona, length,
  ratio/platform) arrives in one request instead of multi-turn `AskUserQuestion`.
- **One HITL gate.** Keep exactly one approval gate at the **beat sheet**, before any paid
  generation, using Ástríðr's existing HITL/approval-gate mechanism.
- **Carry invariants verbatim:** the never-blank-script rule (`params.prompt` is the script),
  Elements-first conditioning, element-lock across clips, ratio discipline, descriptive filenames.
- **Condense the 15 style guides (F6).** Each `skill/styles/*/SKILL.md` is ~900 lines of SEO-toned
  marketing prose (e.g. `07-ecommerce-ad` = 928 lines, "Seedance 2.0" ×56). Distil each to its
  agent-actionable core — hook patterns, camera/motion vocabulary, the category playbook table —
  before loading into the Hildr playbook. Raw, they are a heavy per-invocation token cost.
- **Dialogue is VO-led, not lip-sync-led (F4 — hardened by the 2026-06-30 test).** For any ad with
  spoken lines: generate a **scripted TTS VO** (`text2speech_v2`) of the exact script, then mux it
  with ffmpeg. Do **not** rely on Seedance native audio (garbles words) and do **not** promise tight
  lip-sync (Seedance `audio_references` conditions timing/ambience, not phonemes — lips stay loose).
  **Shot-selection rule:** author speaking beats over B-roll, product shots, hands/application, or
  creator framings that are *not* tight face-on talking (over-shoulder, looking away, walking),
  so loose sync is never exposed; reserve tight face-on close-ups for *non-speaking* beats. If
  face-on dialogue is a hard requirement, that needs a **dedicated lip-sync/talking-head model
  outside Seedance** — call it out as an open dependency, do not assume Seedance covers it.
- **Optional music + virality steps (F5).** Optionally lay a music bed (`acestep`) under the VO and
  run `virality_predictor` on the final cut. Both gated/optional — they must not block a valid ad.
- **Output:** same per-ad folder layout under `higgsfield-generations/` inside the agent's workspace,
  with assets + state also persisted to the registry in deliverable #4.

### 4. Asset registry + persistence — Supabase (F1)
The upstream skill is **stateless by design** (`character-creation.md`: "fresh every run, never a
forever-avatar") — every run re-casts a new creator/product and persists nothing, so any repeat
brand re-pays generation. The port makes the pipeline **stateful** so assets can be stored for
review and operation can resume.

**Decision: Supabase (Postgres + Storage), not Airtable.** Rationale (decided 2026-06-20):
- **Binaries → Supabase Storage.** A UGC ad is multiple Seedance clips + keyframes + a stitched
  final (tens–hundreds of MB/ad). Airtable attachments are plan-capped and serve via **expiring
  URLs** — not a media backend. Supabase Storage (S3-backed, signed URLs, RLS) is built for this and
  sits next to the metadata. The in-container `higgsfield-generations/` folder stays the working
  area; Storage is the durable home + out-of-Docker review access.
- **Operational state → Supabase Postgres + realtime.** Resumable pipeline state (brief, beat sheet,
  job IDs, clip statuses, retries, the registered element IDs) wants a transactional store with
  realtime status — and Ástríðr is **already Postgres/Supabase-backed**, so this keeps one source of
  truth. Airtable as an autonomous agent's system-of-record is awkward (5 req/s-per-base limits,
  eventual consistency, hard SaaS dependency in the state path).
- **Reuse, not re-pay.** Persisting `{character_element_id, product_element_id, environment_element_id,
  brief, brand_kit, ratio, style_route}` keyed by brand lets repeat ads skip re-casting — and gives a
  **recurring brand spokesperson without Soul training** (the only path the skill currently offers).

**Schema sketch (refine in plan):** `ugc_ads` (run row: brief, status, style_route, final asset
ref, virality score), `ugc_elements` (reusable element IDs keyed by brand), `ugc_assets` (per-asset
Storage path + role: character/keyframe/clip/broll/final). Writes go through `engine/atomic_io.py`;
DB access via Ástríðr's existing Supabase client.

**Review surface (optional, follow-on — not core port).** Single source of truth stays Supabase;
build the review board in React/Vercel or as a **CodePulse** panel (CodePulse already owns the
Agentic-OS front-end + a v21 swarm-observability handoff). Only if a non-technical reviewer wants
the Airtable grid specifically: mirror **lightweight review rows** (signed-URL/thumbnail + status +
brand) into Airtable as a disposable *view* — never the store, never the binaries.

## Guardrails / error handling
- Higgsfield MCP unreachable → graceful fail, logged (notebooklm pattern); other servers unaffected.
- **Cost gate with estimate (F2).** Upstream has only a qualitative beat-sheet approval. The port
  adds a **pre-generation cost estimate** at that same gate: derive image+clip counts from the beat
  sheet, price them, read live `balance`, and surface "≈ X credits vs balance Y — proceed?" before
  any paid call. One gate, now quantified.
- No blocking I/O (async); file writes through `engine/atomic_io.py`; structlog only.
- Supabase write failure must not lose a rendered asset: persist the local file first, then upsert
  the registry row; reconcile on retry (idempotent on run/asset key).

## Prereqs (verified)
- node/npx ✓, ffmpeg in Dockerfile ✓, mcp-servers.yaml supports streamable-http ✓.
- Higgsfield account with Elements + Seedance 2.0 access (paid) — shared with Track A.
- Supabase project (Larry's stack default) for the asset registry + Storage bucket — confirm
  Ástríðr's existing Supabase client/creds are reachable from the prod container.

## Execution
GSD phase in `astridr-repo` (repo is GSD-gated). Pre-flight: CWD = astridr-repo, CONTEXT seeded
from this doc. Now scoped as **four deliverables** (was three) — closer to a small milestone than a
single phase; consider splitting the Supabase registry (#4) from the pipeline port (#3) at plan time.
**UI-SPEC:** still not required for the core port (Hildr is headless/agent-driven). The optional
review board *is* a UI surface but is explicitly follow-on (likely a CodePulse handoff phase, like
the v21 swarm view) — spec it only if/when that board is built, not for the core port.
config/ schema owned by Zeta; Hildr persona/agent owned by Beta.

## Track A validation — live test run (2026-06-30)

Ran the upstream `/ugc-factory` skill end-to-end before committing to the port, to satisfy the
SEED-005 gate ("dormant, gated on Track A proving good output"). One full ad: generic skincare
serum ("LUMÉ"), GPT Image 2 + Seedance 2.0, 9:16, 15s / 1 clip, keyframe-pinned, Elements-first.
Cost: a clean ad ≈ **76 credits**; the full iterative session (3 video re-renders chasing the audio
bug) ≈ **212 credits** on the Ultra plan.

### Gate verdict: VISUALS PASS, AUDIO/DIALOGUE FAILS
- **Visual pipeline is excellent and worth porting verbatim.** Character + product **Element lock
  held across every beat**, including a tight application close-up — same creator, same bottle, label
  ("LUMÉ / Vitamin C Brightening Serum") legible in the product shot, the keyframe, *and* the moving
  video. Per-beat motion (push-in hook → raise/rack-focus reveal → serum-application demo → settle
  CTA) executed as scripted. Reads as authentic UGC. **GPT Image 2 is the right call for branded
  product text** (it rendered the label cleanly; this is why the model routing matters).
- **Audio/dialogue is the blocking gap.** Three approaches were tested, all fail for face-on spoken
  dialogue:
  1. **Seedance native audio** (`generate_audio:true`): perfect lip-sync, but **garbles the words** —
     "your first bottle" came out **"bossel"** then **"bottler"** on two separate renders. Non-
     deterministic and **uncorrectable in-language**: `voice_change` keeps the (wrong) words,
     `dubbing` re-transcribes the garbled audio and only translates. Unusable for any spoken CTA.
  2. **Scripted TTS VO muxed post-hoc** (ElevenLabs via `text2speech_v2`, ffmpeg overlay): words
     correct, but **zero lip-sync** — mouth was animated for Seedance's own audio. Larry's words:
     "a disaster… a mess."
  3. **Scripted VO as Seedance `audio_references`, then mux** (the "proper" VO-first method): words
     correct, visual intact, but **lip-sync still loose / out of sync**. Seedance ingested the audio
     as an `_sfx`/ambient reference, not phoneme-level lip-sync. Larry confirmed: "still loose… way
     out of sync."

**Conclusion:** there is **no Higgsfield-native path that delivers BOTH correct words AND tight
lip-sync** for face-on dialogue. Pick one: correct-words-loose-lips (scripted VO) or
right-lips-wrong-words (Seedance native). For an ad you actually ship, correct words win — so the
port goes **VO-led**, and **shot selection must hide the loose sync** (speak over B-roll / product /
hands / non-face-on framing; reserve tight face-on close-ups for non-speaking beats). True face-on
talking-head dialogue requires a **dedicated lip-sync model outside Seedance** — an open dependency,
not something the current toolchain covers. This is folded into deliverables #2 and #3 (F4) above.

### Secondary findings (smaller, fold into plan)
- **Style routing for beauty is off.** `beauty/cosmetics → 13-fashion-lookbook` gave apparel/runway
  craft for a serum demo. Worked only because it was adapted by hand. Revisit the routing map (a
  `07-ecommerce-ad` or dedicated beauty route fits better) when condensing the style guides (F6).
- **The "3–4 angle-set" character step is fragile.** Independently generating angles risks casting
  *different people* and muddying the Element. The test used **one strong front portrait** instead
  (fine for front-facing ads). Port should prescribe **reference-chained angles** (generate front,
  derive other angles *from it*) or explicitly accept single-portrait Elements for front-on ads.
- **Image quality defaults to `low`/1k.** Acceptable, but the port should set quality **explicitly
  per asset** (the product/label shot was bumped to `high` to keep the text crisp).
- **F2 (cost tool) validated as real.** There is no native cost surface; spend had to be eyeballed
  via `balance`. The pre-generation estimate at the beat-sheet gate is a genuine, confirmed value-add.
- **F4 silent-ad framing was too soft.** The original "produces a silent ad" understates it — the
  real problem is that the *only* lip-synced audio path produces *wrong words*. Re-scoped above.

### Net recommendation
Port is justified — the visual engine is strong and the Element-lock is the whole value. But the
plan must treat **dialogue/audio as a first-class design problem, not optional polish**, and either
(a) commit to VO-led ads with sync-hiding shot selection, or (b) budget a separate lip-sync model.
Decide this at discuss/plan time; it changes the playbook's shot vocabulary and the tool-kit.
