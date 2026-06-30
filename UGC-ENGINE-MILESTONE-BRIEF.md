# Milestone brief — UGC Factory Engine (astridr v25.0)

> Paste-ready intake for `/gsd-new-milestone` in `astridr-repo`. Closes SEED-005 (the UGC-Factory
> port). Source design: [`UGC-PLATFORM-DESIGN.md`](./UGC-PLATFORM-DESIGN.md) +
> [`ASTRIDR-PORT-DESIGN.md`](./ASTRIDR-PORT-DESIGN.md) (Track A validation verdict, 2026-06-30).
> Mirror of `astridr-repo/.planning/seeds/SEED-005-engine-milestone-brief.md`. Engine-only this
> milestone; CodePulse + Airtable surfaces are separate later milestones.

**Milestone name:** UGC Factory Engine
**Version:** v25.0 (phases continue from 160 → **161+**)
**Owners:** engine/config → Zeta · Hildr persona → Beta

## Goal (one paragraph)
Build the **headless UGC ad-generation engine** in astridr — the production core that turns a brief
(product, offer, topic, persona, format) into a finished short-form video ad via the Higgsfield REST
API, with a resumable state machine, granular regeneration, multi-tenant isolation, and a per-tenant
cost gate. This is Hildr's port pipeline (closes SEED-005) **and** the shared core that CodePulse and
Airtable will later front. Engine-only this milestone; UI surfaces are separate milestones.

## Why now
Track A (the `/ugc-factory` Claude Code skill) is **validated** (2026-06-30): the visual engine ships,
but dialogue/audio is the real design problem and the skill is stateless. The Higgsfield headless-auth
risk is **retired** — REST API + static key confirmed. Design is fully written
(`UGC-PLATFORM-DESIGN.md`, `ASTRIDR-PORT-DESIGN.md`). Nothing blocks the engine build.

## Scope — IN
- Headless engine: brief → cast Elements → beat sheet (HITL) → keyframes → clips → VO → stitch.
- Higgsfield **REST API** integration (`api.higgsfield.ai/v1`, static `HIGGSFIELD_API_KEY`).
- Supabase store + job queue (`ugc_runs`/`ugc_clips`/`ugc_assets`/`ugc_elements`/`ugc_tenants`).
- Resumable state machine + **granular regen** (clip / VO / cast / stitch).
- Multi-tenancy (RLS), **per-tenant credit budget + cost gate**, content safety on client briefs.
- Northbound: **CLI + HTTP/in-process API** (so surfaces can attach) + optional MCP wrapper for agents.
- Hildr wrapper (in-process) — closes the original port.

## Scope — OUT (explicit boundaries)
- **CodePulse UI** → its own milestone (codepulse repo).
- **Airtable Interfaces + n8n bridge** → its own milestone.
- Dedicated lip-sync/talking-head model → deferred; this milestone is **VO-led with sync-hiding shot
  selection** (per the validated finding).
- Music bed / virality scoring → optional, not required to ship.

## Requirements
| ID | Requirement | Acceptance |
|----|-------------|------------|
| **ENG-01** | Higgsfield REST client (auth, generate, poll, cancel) | Real image+video generated headlessly via `api.higgsfield.ai/v1` with `HIGGSFIELD_API_KEY`; no MCP, no CLI subprocess |
| **ENG-02** | Pipeline port (cast→beat sheet→keyframes→clips→stitch) | One end-to-end ad produced from a brief JSON via `ugc make` |
| **ENG-03** | VO-led dialogue (scripted TTS, never Seedance native) | Spoken CTA words are exactly the script; native TTS never used for dialogue |
| **ENG-04** | Supabase store + Storage | All runs/assets/elements persisted; binaries in Storage, signed URLs out |
| **REGEN-01** | Resumable state machine (idempotent stages) | Killing mid-run and re-running resumes at first incomplete stage |
| **REGEN-02** | Granular regen | `regen(clip=N)`, `regen(stage=vo/cast/stitch)` re-roll only that piece + re-stitch |
| **TENANT-01** | Multi-tenant isolation (RLS) | Tenant A cannot read tenant B's runs/assets/elements |
| **TENANT-02** | Per-tenant credit budget + cost gate | Run blocks at HITL if it would exceed remaining budget; estimate + live balance recorded |
| **TENANT-03** | Content safety on client briefs/assets | Disallowed briefs rejected pre-render, logged |
| **API-01** | Northbound CLI + HTTP/worker (Supabase claim loop) | A `queued` row in Supabase is claimed and driven to `done`; CLI + HTTP both trigger runs |
| **HILDR-01** | Hildr wrapper (in-process; optional MCP wrapper) | Hildr produces an ad by calling the engine; SEED-005 closed |

## Proposed phases (161–165)
| Phase | Name | Requirements |
|-------|------|--------------|
| **161** | Engine core: REST client + pipeline port + Supabase + CLI (VO-led) | ENG-01..04 |
| **162** | Resumable state machine + granular regen | REGEN-01, REGEN-02 |
| **163** | Multi-tenancy + cost gate + content safety | TENANT-01..03 |
| **164** | Northbound HTTP/worker API + host + observability (surface-ready) | API-01 |
| **165** | Hildr wrapper — closes SEED-005 | HILDR-01 |

## Decisions already locked (don't re-litigate in questioning)
- **Higgsfield via REST API + static key**, not the OAuth MCP, not a CLI subprocess (UGC-PLATFORM-DESIGN §8.5 + north/south).
- **Dialogue is VO-led**; shot-selection hides loose lip-sync; native Seedance TTS banned for dialogue.
- **Supabase is the single spine** (store + queue); engine talks only to Supabase + Higgsfield + Anthropic.
- **GPT Image 2** for branded product text; **Elements-first** character/product lock; element reuse keyed by brand.
- **No `budget_tokens`/extended-thinking** params on Opus 4.8 anywhere (astridr standing rule).

## Constraints / non-negotiables
- Async, structlog, writes via `engine/atomic_io.py` (astridr conventions).
- `HIGGSFIELD_API_KEY` as a `SecretRef(required=False)` — missing key must not trip bootstrap fail-fast.
- Behavioral verification (SEED-003): every success criterion exercises the real path + observes real output.
- Rebuild proof for any container-affecting change (`docker compose --profile prod up --build -d`).

## Milestone success criteria
A non-technical brief (JSON or CLI) produces a **finished, on-script, multi-tenant-isolated UGC ad**
end-to-end via the REST engine, is **resumable and re-rollable per-clip**, **blocks on budget**, and
**Hildr can produce one by calling the engine** — with CodePulse/Airtable able to attach later via the
HTTP API.

## Plan-time open items (non-blocking)
- Confirm REST API exposes **Elements / audio_references / `seedance_2_0`** (shapes differ from MCP).
- Confirm REST draws **Ultra credits vs a separate API pool** (feeds the cost gate).
- Lead with **ENG-01** (REST integration) — it's the only real unknown left.

## References
- [`UGC-PLATFORM-DESIGN.md`](./UGC-PLATFORM-DESIGN.md) — full platform design (engine, surfaces, data model, §8.5 auth verdict).
- [`ASTRIDR-PORT-DESIGN.md`](./ASTRIDR-PORT-DESIGN.md) — port seed + Track A validation.
- `astridr-repo/.planning/seeds/SEED-005-ugc-factory-port.md` — the dormant seed this milestone activates.
