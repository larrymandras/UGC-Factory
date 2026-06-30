# UGC Factory Platform — Design (one engine, four surfaces)

> Companion to [`ASTRIDR-PORT-DESIGN.md`](./ASTRIDR-PORT-DESIGN.md) (the Track B port seed) and the
> 2026-06-30 Track A validation. This doc widens the port into a **multi-surface platform**: the same
> headless engine drives a CLI, the Hildr agent, a CodePulse daily-driver UI, and an Airtable
> client-intake surface. Author: Larry. Status: **approved design seed**, pre-GSD.

## Decisions locked (2026-06-30)
- **One engine, four surfaces.** Build the headless pipeline **once** in `astridr-repo` (it *is*
  Hildr's port deliverable #3). CLI, Hildr, CodePulse, and Airtable all front the same engine.
- **CodePulse is the daily-driver surface** (Larry). Airtable is the **external-client** surface.
- **Supabase is the single spine** — store + job queue. The engine talks **only** to Supabase.
- **External clients are in scope** → multi-tenant: per-brand isolation, auth, per-client credit caps,
  content safety on client-submitted briefs/assets.
- **Trigger = poll/claim from Supabase + n8n bridges Airtable.** Engine claims `queued` runs from
  Supabase; CodePulse writes Supabase directly (realtime); n8n syncs Airtable⇄Supabase.

## Thesis: don't build a UGC app — build the engine and put faces on it
The `/ugc-factory` skill can't be called headlessly (it's an interactive Claude Code agent). Anything
that "runs from Airtable" needs a headless engine behind it — and that engine is **already** the
Ástríðr port (Hildr deliverable #3). So the Airtable app is not separate work; it's a thin surface on
an engine we're building anyway. Build the engine once; the surfaces are cheap.

```
   ┌────────── SURFACES (control plane) ──────────┐
   │                                              │
   │  CodePulse (TS/React, Supabase-native)  ◄──► │   Larry's daily driver: brief, approve,
   │     realtime run board + review              │   regenerate, observe live renders
   │                                              │
   │  Airtable Interfaces  ◄── n8n ──►            │   External clients: intake form, review,
   │     forms + galleries + buttons              │   regenerate; n8n syncs to/from Supabase
   │                                              │
   │  CLI (`ugc make …`)                          │   Larry local / power use / batch
   │  Hildr (in-process)                          │   Ástríðr agent invocation
   └───────────────────┬──────────────────────────┘
                       │  (all write intent rows / read status)
            ┌──────────▼───────────┐
            │   SUPABASE (spine)   │  ugc_runs (job queue + state), ugc_clips, ugc_assets,
            │   Postgres + Storage │  ugc_elements, ugc_tenants; RLS per tenant; Storage buckets;
            │   + Realtime         │  signed URLs out
            └──────────┬───────────┘
                       │  claim `queued`, write progress (idempotent, resumable)
            ┌──────────▼───────────────────────────────────────────────┐
            │  ugc-engine  (Python module in astridr-repo)              │
            │  brief → cast Elements → beat sheet (HITL) → keyframes →  │ ──► Higgsfield MCP
            │  clips (Seedance) → VO (scripted TTS) → stitch (ffmpeg)   │ ──► Anthropic SDK (brain)
            │  state machine · idempotent stages · granular regen       │ ──► ffmpeg
            └──────────────────────────────────────────────────────────┘
```

---

## 1. The engine (`ugc-engine`) — the heart, and Hildr's core

A resumable, idempotent **state machine**. One `ugc_runs` row per ad; every stage writes its output
as its own row so any single piece can be re-rolled without redoing the rest.

### Pipeline stages
```
brief ─► cast ─► beat_sheet ─►(HITL: approve + cost gate)─► keyframes ─► clips ─► vo ─► stitch ─► done
         │                                                   │           │       │      │
         └ character/product/env Elements (reuse if branded) per-clip   per-beat  scripted  ffmpeg
                                                              anchor    Seedance  TTS+mux   concat
```

- **Idempotent stages.** Each stage checks "is my output already present and current?" before doing
  work. Re-running a run resumes at the first incomplete stage. (Mirrors astridr's resumable patterns.)
- **Granular regeneration** (first-class — the "regenerate" button):
  - `regen(run, clip=N)` → re-roll only clip N's keyframe + Seedance clip, then re-stitch.
  - `regen(run, stage=vo)` → re-do only the voiceover + re-mux.
  - `regen(run, stage=cast, slot=character)` → recast the creator, propagate to keyframes downstream.
  - `regen(run, stage=stitch)` → just re-concat (e.g., after reordering beats).
- **The creative brain = Claude (Anthropic SDK)** writes the beat sheet, per-clip prompts, VO script,
  and style routing. Guardrails (see §6) live in its system prompt.
- **Higgsfield via astridr's MCP client.** Reuses the `mcp-servers.yaml` `higgsfield` entry from the
  port design (auth header is the known open item to resolve first). ffmpeg is already in astridr's
  Dockerfile.

### Engine interfaces (one core, three entry points)
| Entry point | Use |
|---|---|
| **CLI** `ugc make --brief brief.json` / `ugc regen <run> --clip 3` | Larry local, batch, power use |
| **In-process API** (`engine.run_goal(...)`) | Hildr / Ástríðr calls it directly |
| **Supabase claim loop** (worker) | Surfaces enqueue; worker claims `status=queued` and drives the run |

The worker model means **no inbound networking** for the engine: it reaches out to Supabase, claims
jobs, writes progress. Run it on Larry's laptop *or* hosted — same code, one config flag.

---

## 2. Multi-tenancy (external clients run it too)

- **Tenant = brand/client.** `ugc_tenants` row; every run/asset/element carries `tenant_id`.
  Supabase **RLS** isolates tenant data; Storage paths namespaced `{tenant_id}/{run_id}/…`.
- **Auth.** CodePulse = Larry's existing auth (owner). Airtable clients authenticate at the Airtable
  layer; n8n carries a service token to Supabase and stamps `tenant_id` per base/client. No client
  ever touches Supabase or Higgsfield creds directly.
- **Cost governance (hard requirement for outsiders).** Per-tenant **credit budget** + the F2
  pre-generation **cost gate**: engine estimates image+clip+VO credits from the beat sheet, reads live
  `balance`, and **blocks at the HITL gate** if the run would exceed the tenant's remaining budget.
  No client can silently burn your Higgsfield Ultra credits.
- **Content safety.** Client-submitted briefs/assets pass a moderation check (Claude) before any paid
  render — reject disallowed products/claims; log rejections. Protects your account + brand.

---

## 3. Data model (Supabase = source of truth)

| Table | Key fields |
|---|---|
| `ugc_tenants` | id, name, credit_budget, credits_spent, status |
| `ugc_runs` | id, tenant_id, **status** (queued/casting/awaiting_approval/rendering/review/done/failed), brief(jsonb), style_route, cost_estimate, final_asset_id, virality_score, claimed_by, claimed_at |
| `ugc_clips` | id, run_id, beat, prompt, keyframe_asset_id, clip_asset_id, status (enables per-clip regen) |
| `ugc_assets` | id, tenant_id, run_id, role (input/character/product/env/keyframe/clip/vo/final), storage_path, higgsfield_id, source(uploaded/generated) |
| `ugc_elements` | id, tenant_id, brand, role, higgsfield_element_id → **stateful reuse the skill lacks** (recurring spokesperson, no re-cast) |

- **Storage buckets:** `ugc-inputs`, `ugc-assets`, `ugc-final` (signed URLs out; RLS per tenant).
- **Job queue = the `ugc_runs.status` column** + `claimed_by`/`claimed_at` (claim with a conditional
  update; stale-claim reaper). No extra queue infra.
- **Airtable** mirrors run/asset rows as **links + thumbnails only** (signed Supabase URLs) — never
  binaries (respects Airtable's expiring-attachment + plan-cap limits, per the port doc's decision).

---

## 4. Surfaces

### CodePulse — daily driver (Larry)
Supabase-native + **realtime** (same pattern as the v21 swarm view):
- **Run board** — live Kanban by `status`; renders stream in as they finish (realtime subscription).
- **New Ad** — brief form; uploads go straight to Supabase Storage; writes a `queued` run.
- **Review** — embedded final video + per-clip thumbnails; **Regenerate this clip / VO / recast**
  buttons (write a regen intent the engine claims).
- **Approval gate** — beat sheet + cost estimate + budget surfaced; **Approve** flips status.
- **Element library** — reusable spokespeople/products per brand.
Reads need no engine API (Supabase realtime); writes are just row inserts the engine claims.

### Airtable Interfaces — external clients
- **Intake form** (product, offer, topic, persona, length, ratio, platform + attach photos).
- **Run board** (Kanban), **Review gallery** (Supabase video link + thumbnails), **Regenerate** button.
- **Approve beat sheet** checkbox (HITL).
- n8n bridges: Airtable row → Supabase run (download attachments immediately — they expire); Supabase
  status/outputs → Airtable fields.

### CLI & Hildr
- **CLI** for Larry: `ugc make`, `ugc regen`, `ugc list`, `ugc cost <run>` — batch + scripting.
- **Hildr** calls the engine in-process; the Ástríðr playbook becomes a thin wrapper that builds a
  brief and calls `engine.run_goal()`.

---

## 5. Trigger architecture (both poll-claim and n8n)
- **Engine ⇄ Supabase only.** It claims `ugc_runs.status='queued'` (conditional update → `claimed`),
  drives the run, writes progress. Poll interval ~10–15s, or Supabase Realtime/`LISTEN` for instant.
- **CodePulse → Supabase** directly (it's Supabase-backed). Instant, no bridge.
- **Airtable → n8n → Supabase** (and back). n8n owns: attachment download, tenant stamping, retries,
  logging, status mirroring. Keeps the Airtable edge out of the engine.

---

## 6. Creative guardrails (baked into the engine, from the validated findings)
- **Dialogue is VO-led, never Seedance native TTS** (2026-06-30: native garbled "bottle"→"bossel"/
  "bottler"; no native path gives correct words *and* tight lip-sync). Engine generates a scripted
  `text2speech_v2` VO and muxes it.
- **Shot-selection rule for speaking beats** — over B-roll / product / hands / non-face-on framing;
  tight face-on close-ups only for non-speaking beats (hides loose lip-sync). Face-on dialogue ⇒
  flag a dedicated lip-sync model as an open dependency.
- **Never-blank-script, Elements-first, element-lock, ratio discipline** — carried verbatim.
- **Cost gate (F2)** at the HITL boundary, now also enforcing per-tenant budget.
- **GPT Image 2 for branded product text** (validated: rendered the label cleanly).
- **Element reuse** keyed by brand → recurring spokesperson without re-casting (the stateful win).

---

## 7. Security & secrets
- Higgsfield + Anthropic + Supabase service creds live **only** in the engine env (astridr `.env`,
  bootstrap-resolved). No surface holds them. Clients never see them.
- Per-tenant RLS on every table + Storage path; signed, expiring URLs out.
- Content moderation on client briefs/assets before paid render.
- Higgsfield auth = **OAuth, not a static key** — see §8.5 (the Phase-0 critical-path item).

## 8.5 Higgsfield headless auth — investigation verdict (2026-06-30)
Investigated astridr's MCP layer + how Claude Code authenticates Higgsfield. Findings:

- **Streamable-HTTP transport is already supported and battle-tested** in astridr —
  `integrations/mcp.py::_connect_streamable_http` uses the official `mcp` SDK
  (`streamablehttp_client` + `ClientSession`); the `notebooklm` server runs on it in prod. ✓
- **But astridr's streamable-HTTP path passes NO auth.** `MCPServerConfig` (mcp.py) and
  `MCPServerEntry` (engine/config.py) have **no `headers`/auth field**, and `_connect_streamable_http`
  calls `streamablehttp_client(url=..., timeout=..., sse_read_timeout=...)` with **no `headers=`**.
  `notebooklm` needed none (unauthenticated, internal Docker network). So remote *authenticated* http
  MCPs are currently unsupported.
- **Higgsfield specifically uses OAuth, not a static bearer token.** Evidence: in Claude Code's MCP
  config, the other remote http MCPs (github / supabase / n8n) each carry a static
  `Authorization: Bearer …` header, but **Higgsfield has only `type:http` + `url`, no header** —
  yet it clearly authenticates (it knows the account's credits/generations). Claude Code has an
  MCP-OAuth subsystem (`tengu_mcp_local_oauth_blocked_hosts`) and a `.credentials.json` store; the
  Higgsfield tokens live there, obtained via the browser OAuth flow at install (the affiliate link).

**Implication — two difficulty tiers, depending on what Higgsfield offers for server-to-server use:**
1. **If Higgsfield exposes a static API key / personal access token / client-credentials grant**
   (server-to-server) → **SMALL fix**: add `headers: dict[str,str] | None` to `MCPServerConfig` +
   `MCPServerEntry`, resolve a `SecretRef` (`required=False`, so a missing token doesn't trip
   bootstrap fail-fast) into `Authorization: Bearer …`, and pass `headers=` to `streamablehttp_client`
   (the SDK accepts it). Then `mcp-servers.yaml` gets a `higgsfield` entry with a `headers` block —
   identical shape to how github/supabase/n8n are wired in Claude Code.
2. **If Higgsfield is OAuth-only (no static key)** → **MEDIUM**: implement an MCP-OAuth client in
   astridr (authorization-code consent once, persist + refresh the token via the credential store),
   passing the SDK's `auth=` provider to `streamablehttp_client`. Heavier, but a reusable capability
   (other OAuth MCPs benefit). **Alternative:** check for a **Higgsfield REST API with an API key**
   and bypass the MCP entirely for the headless engine (the MCP is a convenience layer; a REST key
   would be the cleanest server-to-server path).

**Next action (gates Phase 0):** confirm with Higgsfield whether a static API key / PAT /
client-credentials grant (or a REST API) exists. That single answer picks tier 1 vs tier 2 and
unblocks the engine build. Until then, treat tier 2 (OAuth) as the planning assumption.

---

## 8. Build roadmap (mapped to GSD, across repos)
| Phase | Repo | Deliverable |
|---|---|---|
| **0. Engine MVP** | astridr | Headless pipeline (port the skill, VO-led, Supabase store, CLI). Resolve http-MCP auth first. |
| **1. State machine + granular regen** | astridr | Idempotent stages, resumable, `regen(clip/vo/cast/stitch)` |
| **2. Multi-tenancy + cost gate** | astridr/supabase | tenants, RLS, per-tenant budgets, content safety |
| **3. CodePulse surface** | codepulse | Realtime run board, new-ad, review, regen buttons, element library (Larry's daily driver) |
| **4. Airtable + n8n bridge** | airtable/n8n | Client intake Interface, n8n Airtable⇄Supabase sync, approval mapping |
| **5. Host + harden** | astridr | Host the worker, observability, rate/cost limits, client onboarding |
| **6. Hildr wrapper** | astridr | Ástríðr playbook calls the engine in-process (closes the original port) |

Phases 0–2 are the engine (also the Hildr port). Phase 3 is the CodePulse daily-driver. Phase 4 opens
it to clients. This is **a milestone in astridr + a milestone in codepulse**, sharing one Supabase spine.

## 9. Open questions / risks
- **Higgsfield headless auth — investigated (§8.5).** Verdict: transport ready; astridr's http-MCP
  path has no auth field; **Higgsfield uses OAuth, not a static key.** Open sub-question that picks
  the difficulty tier: does Higgsfield offer a static API key / client-credentials / REST API for
  server-to-server use? Confirm before Phase 0.
- **Lip-sync ceiling** — if face-on dialogue is ever required, budget a dedicated lip-sync model; until then, VO-led + shot-selection.
- **Credit blast radius** — external clients on your Ultra plan; per-tenant budgets + cost gate are mandatory, not optional.
- **Airtable attachment expiry** — n8n must download on arrival; never store the Airtable URL.
- **Render latency** — 15s clips take 1–3 min; surfaces must be async/realtime (no blocking UI).

## 10. References
- [`ASTRIDR-PORT-DESIGN.md`](./ASTRIDR-PORT-DESIGN.md) — the port seed + Track A validation verdict.
- `skill/` — upstream pipeline logic to port (run-pipeline.md, 5 frameworks, 15 style guides).
- Validated test run: `higgsfield-generations/UGC-lume-glow-2026-06-30/` (assets gitignored).
