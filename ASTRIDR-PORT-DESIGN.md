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

## Three deliverables

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
the Higgsfield tools (`generate_image`, `generate_video`, `show_reference_elements`,
`media_upload`, `media_confirm`, `job_status`, `show_generations`, `models_explore`) + the ffmpeg
media tool, granted to **Hildr**.

### 3. Ported pipeline — playbook/prompt for Hildr
Convert `skill/tasks/run-pipeline.md` + the 5 `skill/frameworks/*` + 15 `skill/styles/*/SKILL.md`
into an Ástríðr-native playbook the creative agent loads. Adaptations:
- **Interview → param-driven single-shot.** Brief (product, offer, topic, persona, length,
  ratio/platform) arrives in one request instead of multi-turn `AskUserQuestion`.
- **One HITL gate.** Keep exactly one approval gate at the **beat sheet**, before any paid
  generation, using Ástríðr's existing HITL/approval-gate mechanism.
- **Carry invariants verbatim:** the never-blank-script rule (`params.prompt` is the script),
  Elements-first conditioning, element-lock across clips, ratio discipline, descriptive filenames.
- **Output:** same per-ad folder layout under `higgsfield-generations/` inside the agent's workspace.

## Guardrails / error handling
- Higgsfield MCP unreachable → graceful fail, logged (notebooklm pattern); other servers unaffected.
- Cost gate = beat-sheet approval before any generation.
- No blocking I/O (async); file writes through `engine/atomic_io.py`; structlog only.

## Prereqs (verified)
- node/npx ✓, ffmpeg in Dockerfile ✓, mcp-servers.yaml supports streamable-http ✓.
- Higgsfield account with Elements + Seedance 2.0 access (paid) — shared with Track A.

## Execution
GSD phase in `astridr-repo` (repo is GSD-gated). Pre-flight: CWD = astridr-repo, CONTEXT seeded
from this doc, UI-SPEC not required (no new UI surface). Likely a small new phase or milestone
addition. config/ schema owned by Zeta; Hildr persona/agent owned by Beta.
