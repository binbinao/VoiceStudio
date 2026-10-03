# Repository Guidelines

Binding for every AI agent (Claude, Codex, Cursor, review bots, …). `CLAUDE.md` is the full constitution; this is the operating contract. When they conflict, `CLAUDE.md` wins.

## Token economy (owner directive, 2026-07-20; tightened 2026-07-28)
- **Default to the shortest response that fully answers.** Outlines and tables over prose; no preamble, no recap of what you just did, no re-explaining a fix the diff already shows. Applies to every response, not just status updates.
- Lead with the outcome. No narration, no restating diffs, no filler praise, no plans you're about to execute anyway.
- Status updates: one line. Final reports: only what changes the reader's next action.
- Don't re-derive what CI, linters, or review bots already computed — read their output first (`gh pr checks`, bot comments via `gh api .../pulls/N/comments`).
- Mechanical rules live in deterministic tests, never in agent effort: changelog style (`tests/test_changelog_style.py`), locale parity (`tests/test_locale_parity.py`), version lockstep (`tests/test_app_version.py`), CJK (`tests/test_no_hardcoded_cjk.py`).
- Run targeted tests while iterating; full suites only before landing.
- Tests and CI simulate CI honestly: `HF_HUB_OFFLINE=1` + empty `HF_HUB_CACHE` — a populated dev cache masks real failures.

## Project Overview

VoiceStudio (project package name `omnivoice`, v0.5.x) is an open-source, fully-local ElevenLabs alternative: voice cloning, voice design, video dubbing, and real-time dictation across 646 languages. Runs entirely on the user's machine (CUDA/MPS/ROCm/CPU auto-detect); no API keys, no accounts, no cloud dependencies. Core value: **a first-run that actually works** — engine compatibility with existing installs, cross-platform parity, and local-first guarantees are non-negotiable.

## Architecture & Data Flow

Three cooperating processes plus an optional remote GPU worker:

1. **Electron main** (`electron/src/main/index.ts`) — window/tray/updater/repair; spawns and supervises the Python backend (`BackendSupervisor` in `electron/src/main/backend.ts`).
2. **FastAPI backend** (`backend/main.py` → `app`, uvicorn on `127.0.0.1:3900`) — ~44 routers (`backend/api/routers/`), static audio mounts, SPA at `/`, MCP server at `/mcp`.
3. **Electron renderer** (`electron/src/renderer/`) — React SPA calling the backend over `/api`.
4. **Rust helper** (`native/desktop-bridge/`) — dictation capture, global shortcuts, watch folders; JSON-over-stdin RPC spawned by Electron.
5. **Remote worker** (`backend/worker/`, gRPC) — a GPU box renders for another machine, reusing the SAME engine code (`worker/executor.py` → backend `services/` path).

**TTS request flow**: renderer `use-generate` → `lib/api/generate.ts` → `apiFetch('/generate')` (`lib/api/client.ts`, `API_BASE='/api'`) → `POST /generate` (`backend/api/routers/generation.py`) → engine resolution (`services/tts_backend.get_backend_class`, env `OMNIVOICE_TTS_BACKEND` > persisted pref > `'omnivoice'`) → **local**: `services/model_manager.get_model()` on the GPU pool → `backend.generate(...)` → `_finalize_generation` → **remote**: `services/gpu_gateway._routing_decision()` → worker gRPC → `worker/executor.py` → artifact upload back.

**Load-bearing invariants**:
- **Deferred startup**: `backend/main.py` platform bootstrap (top ~120 lines: `--supervise` dispatch, Windows patches, `ensure_omnivoice_importable`) MUST run before torch/FastAPI import; routers register in `_phase_a_build_inner`/`_phase_a_finalize` AFTER socket bind so `/health` answers during the multi-second torch import. Never reorder, never import heavy modules at router module scope.
- **Watermark chokepoint**: EVERY audio producer (generation, openai_compat, tts_stream, dub_generate, batch, audiobook, voice_convert, worker/executor, …) routes through `mark_synthetic`/`mark_synthetic_async` (`backend/services/watermark.py`). Use the async variant (sync blocks the GPU pool); remote workers mark `force=True`; it never raises. A new audio-producing route without it is a provenance gap (#1169).
- **GPU pool is a `ThreadPoolExecutor`, not asyncio** (`services/model_manager.py::run_on_gpu_pool_guarded`): serializes GPU work; MPS pins it to 1 worker (can't kill a blocking thread); dedicated separate pools for watermark and model load. New GPU work goes through it, not `asyncio.to_thread`.
- **Loopback-only by default**: `api/dependencies.require_loopback` gates admin routes unless `OMNIVOICE_SERVER_MODE=1` (Docker). Treat every query/path/form param as hostile; FS destinations are authorized in Electron main, never via HTTP params.
- **Electron IPC**: every `ipcMain.handle` (`electron/src/main/ipc.ts`) opens with `assertTrustedMainFrame(event, owner)`; preload exposes a typed bridge via `contextBridge`, never raw `ipcRenderer`.

## Key Directories

| Path | Purpose |
|---|---|
| `backend/` | FastAPI server: `main.py` (entry, load-bearing boot order), `api/routers/` (~44 routers), `core/` (config/db/auth/errors), `services/` (~90 modules: tts_backend, model_manager, watermark, gpu_gateway, …), `engines/` (subprocess-isolated TTS sidecars), `worker/` (remote GPU worker, gRPC), `speech_client/` (CLI bridge), `mcp_shim/`, `migrations/versions/` (alembic), `plugins/` (empty; SDK in `services/plugin_sdk.py`) |
| `omnivoice/` | The native TTS model package: `models/omnivoice.py` (`OmniVoice` HF model), `cli/` (`omnivoice-infer`, `train`, `dub`), `training/`, `utils/`, `eval/`. PEP-562 lazy exports — importing `omnivoice.utils.*` must NOT pull torch |
| `electron/src/` | `main/` (Electron main, IPC, backend supervisor, updater), `preload/` (contextBridge), `renderer/` (React SPA: TanStack Router hash history, TanStack Query + Zustand, `i18n/locales/` — the 21-locale catalog the app loads), `shared/` (shared-library slice, NOT a runnable app, imported via `@shared/*`) |
| `native/desktop-bridge/` | Rust helper (dictation capture, shortcuts, watch folders), built by cargo, bundled via electron-builder afterPack |
| `tests/` | Main pytest session (~558 files) incl. mechanical-rule suites; `smoke/`, `evals/`, `frontend/` (node:test) |
| `backend/tests/` | Isolated pytest session (routers on bare FastAPI apps) — never reintroduce module-level `sys.modules` stubs |
| `electron/tests/` | node:test units (`*.test.mjs`), smoke scripts, packaging contracts |
| `scripts/` | Dev/build/release helpers (`dev-backend.mjs`, `setup.py`, installers, `check-docs-drift.py`, …) |
| `deploy/` | Dockerfile (CUDA default; ROCm via `BASE_IMAGE`/`GPU_FLAVOR`), docker-compose (cpu/gpu/rocm/worker profiles), `install-worker/` (Cloudflare Worker serving voicestudio.sh/install) |
| `docs/` | `STRUCTURE.md` (authoritative layout — new top-level dirs need a PR updating it), `RELEASING.md` (release checklist), `features.yaml` (canonical feature inventory, drives docs-drift CI), `adr/`, `agents/`, `specs/`, `install/`, `engines/`, `dubbing/` |
| `bin/` | Prebuilt `omnivoice-tts` sidecars per platform |

Three test homes are deliberate, not drift: `tests/` (main pytest), `backend/tests/` (isolated session), `electron/tests/` (bun/vitest). Tests mirror source paths where a mirror exists.

## Development Commands

Setup: `bun install` → `bun run setup:api` (= `uv sync && uv run python scripts/setup.py`).

```bash
# Dev (desktop, Electron):      bun run dev          # = bun run --cwd electron dev
# Dev (web, full stack):        bun run dev:web      # api on :3900 + web on :3901
# Dev (API only):               bun run dev:api      # = bun scripts/dev-backend.mjs
# Build (desktop):              bun run build
# Build (web):                  bun run build:web
# Distribute:                   bun run dist          # root script already passes --publish never
# Typecheck:                    bun run typecheck    # tsc --noEmit, tsconfig.node.json + tsconfig.web.json
# Lint / format:                bun run lint | bun run format     # vp lint / vp fmt (oxlint + oxfmt)
# Electron full contract:       bun run check:electron  # typecheck && test && build && build:web && packaging-contract.mjs
# Packaged smoke:               bun run smoke-test

# Backend tests (CI shape):
HF_HUB_OFFLINE=1 uv run --no-sync pytest tests/ -q --tb=short
HF_HUB_OFFLINE=1 uv run --no-sync pytest backend/tests/ -q --tb=short
HF_HUB_OFFLINE=1 uv run --no-sync pytest tests/test_locale_parity.py -q   # single file
# Frontend unit tests:          node --test tests/frontend/*.test.mjs
# Electron suite:               bun run --cwd electron test  # locale:check && vp test --run (×2 configs)
# Native helper:                cargo test --locked --manifest-path native/desktop-bridge/Cargo.toml
```

Runtime: Python 3.11 (uv, `uv.lock`), Bun 1.4.2 (`packageManager`, repo-root `bun.lock`), Node 22 in CI, Rust stable for the native helper. Ports: backend 3900, dev web 3901.

## Code Conventions & Common Patterns

- **Formatting**: oxfmt `printWidth 100`, single quotes (`electron/.oxfmtrc.json`); oxlint correctness=error (`electron/.oxlintrc.json`). No ruff/black/mypy config for Python — follow existing style.
- **Naming**: `snake_case` Python, kebab-case JS/TS files; tests `test_*.py` mirroring source paths; issue-tagged files `test_*_<NNNN>.py`.
- **Error handling**: `backend/core/failure.py::classify` maps exceptions to codes + repair hints; `core/public_errors.py` shapes HTTP responses; `TTSInputError` → 400; remote failures → 503 with `Retry-After` and `X-OmniVoice-Retryable` headers. Errors should tell the user what to do.
- **Async**: FastAPI async routers; blocking work offloaded via `run_on_gpu_pool_guarded` (GPU), dedicated pools, or `asyncio.to_thread`; `_model_lock` serializes cold loads (loop-bound — `get_model` detects pool-worker callers to avoid cross-loop deadlock, #1417).
- **Dependency injection**: minimal — `Depends` mostly for auth (`require_admin`, `require_loopback`); heavy services are module-level singletons (model_manager, tts_backend registry).
- **Config**: flat module constants from env (`backend/core/config.py`); user prefs in SQLite (`core/prefs.py`); no pydantic-settings.
- **State management (renderer)**: TanStack Query for server state, Zustand for UI state, small `useSyncExternalStore` stores; hash history routing (custom `app://` protocol); all API calls via `lib/api/client.ts:apiFetch`.
- **Engines**: in-process backends subclass `TTSBackend` (`services/tts_backend.py::_REGISTRY`); subprocess sidecars (`backend/engines/<name>/main.py`) must be stdlib-only at import, must NOT import `backend.*` (isolated venvs), speak length-prefixed JSON over stdio (`services/subprocess_backend.py`), and emit `{op:'ready'}` within 30s.
- **DB**: SQLite + alembic; migrations hand-written (`target_metadata=None`), schema source of truth `core/db.py::_BASE_SCHEMA`; `ensure_schema` runs `upgrade head` on startup with pre-migration backup. Never break `omnivoice_data/` backward compat.
- **Logging**: `omnivoice.*` logger namespaces; `core/logging_filter.py` scrubs HF tokens — never log secrets or absolute home paths.
- **Shared selects**: use `electron/src/shared/components/SearchableSelect.jsx` for new/redesigned selects; reuse `VoiceSelector`; no native `<select>`; localized `ariaLabel`; `menuPortal` in scrolling containers.

## Important Files

- `backend/api/routers/generation.py` — `/generate`, `_finalize_generation`, audio serving.
- `backend/services/tts_backend.py` — engine registry + `TTSBackend` ABC.
- `backend/services/model_manager.py` — model cache + GPU pool + VRAM eviction.
- `backend/services/watermark.py` — `mark_synthetic` chokepoint.
- `electron/src/main/ipc.ts` — all privileged IPC (trusted-frame asserts).
- `electron/src/renderer/src/i18n/locales/*.json` — the 21-locale catalog the app loads.
- `docs/STRUCTURE.md` — authoritative directory map; update it in the same PR when layout changes.
- `docs/features.yaml` — canonical feature inventory; engine ids must match `backend/services/tts_backend.py` registries (checked by `scripts/check-docs-drift.py`).
- `pyproject.toml` / root `package.json` — dependency manifests; root `package.json` version is canonical.

## Runtime/Tooling Preferences

- **Bun, not npm/yarn**: `packageManager: bun@1.4.2`; Docker runs `bun install --frozen-lockfile`. Any JS dependency change MUST regenerate repo-root `bun.lock` and confirm the frozen install passes.
- **uv, not pip**: `uv sync` (retry wrapper: `bash scripts/uv-sync-retry.sh`). torch pinned via `[tool.uv] constraint-dependencies` (torch==2.8.0 family; mirrored in `deploy/torch-constraints.txt` — `uv pip install` ignores project constraints).
- TypeScript 7 (tsgo) via the Vite+ `vp` CLI for lint/fmt/test; electron-vite + electron-builder for packaging; shadcn via `bun run --cwd electron ui:add`.
- Prefer what's already pinned; check `uv tree` before adding Python dependencies.

## Testing & QA

- **Frameworks**: pytest 9 (root `tests/` + isolated `backend/tests/`), node:test (`tests/frontend/`, `electron/tests/*.test.mjs`), vitest 4 via `vp test` (jsdom, two vite configs), Playwright renderer smokes in CI. No custom pytest markers — gating uses `skipif`/`xfail` (e.g. `OMNIVOICE_SMOKE=1`, `PROBE_E2E=1`).
- **Hermeticity** (both conftests): `OMNIVOICE_MODEL=test` sentinel (never resolve a real checkpoint), `OMNIVOICE_DATA_DIR`=mkdtemp, `OMNIVOICE_PRELOAD_WATERMARK=0`, `OMNIVOICE_DISABLE_GPU_INVENTORY=1`; CI adds `HF_HUB_OFFLINE=1` + empty `HF_HUB_CACHE`. Autouse fixtures snapshot/restore persisted settings; ASR tests opt in via `asr_model_installed`.
- **Mechanical-rule suites enforce repo policy** — these are the contract, keep them passing: `tests/test_changelog_style.py`, `tests/test_locale_parity.py` (21-locale parity + ratchet), `tests/test_app_version.py` (version lockstep), `tests/test_no_hardcoded_cjk.py` (CJK allowlist).
- **Coverage**: pytest-cov available but no CI coverage gate.
- **CI gates** (`.github/workflows/ci.yml`): `test` job = pytest ×2 → `bun run test:frontend` → `bun run check:electron` → Playwright smokes; `smoke-matrix` (mac/win/linux), `installer`, `linux-native-package`. Merge gate: "Tests (backend + frontend)" green + MERGEABLE. Other gating checks: `commit-identity`, CLA, gitleaks. LLM-judge evals (`evals.yml`) are intentionally non-gating.

## Change Rules (binding)

1. **Root-cause the class, not the instance**; fail-before/pass-after regression test; smallest correct, recurrence-proof change.
2. **Cross-platform parity is behavioral**: default-mode features work identically on macOS/Windows/Linux; platform-only features go behind explicit opt-in (Settings/env/CLI). Divergent default = P0. Don't fix a parity finding by disabling a working optimization everywhere — performance may vary by host.
3. **Local-first**: no new required network calls; HF downloads gated on installed-ness or explicit user action; all synthetic audio through `mark_synthetic`. Sanctioned calls are listed in `.github/CONTRIBUTING.md` (Quality gates) — additions need owner approval.
4. **i18n**: every user-facing string via `t('...')` keys present in ALL 21 `electron/src/renderer/src/i18n/locales/*.json` with real translations; never hardcode CJK (use unicode escapes where functionally required, e.g. `omnivoice/utils/voice_design.py`).
5. **Docs-sync in the same PR**: README, `.github/CONTRIBUTING|SECURITY|SUPPORT.md`, LICENSE, `docs/**` — stale docs are bugs.
6. **Changelog**: Unreleased entries are quiet one-liners ending `(#N)` (+ `— thanks @user!` for community work) under a short `**Highlights**` list; tagged releases lead with the biggest user-visible change (see `tests/test_changelog_style.py`, `docs/RELEASING.md`).
7. **Versioning**: root `package.json` is the single source of truth; mirrors `pyproject.toml` + `backend/core/version.py::_FALLBACK_VERSION` bump in lockstep (guarded by `tests/test_app_version.py`). Never bump without the owner asking.
8. **Attribution**: commits, PR descriptions and comments carry only the submitter's git identity. Never credit an agent (no `Co-authored-by:`, "Generated with …", session links) and never add other names/emails. Enforced by `commit-identity`.
9. **Keep main green**: validate the full active CI matrix + `deploy/Dockerfile`, not just the checks you ran; watch `main`'s post-merge runs to green; red main = drop everything and fix.
10. **Issues**: absorb or decline — never defer to a future version. Check the open-PR queue before implementing community-reported fixes.

## Merge protocol (hard rules)

1. Never merge without review. Harvest CodeRabbit + Greptile comments first; never merge with an unread Critical/P1.
2. Never accept a PR as-is: fix findings ON the PR branch pre-merge (maintainer commits fine; credit contributors in the CHANGELOG). No merge-then-fix, no comment-and-walk-away.
3. Merge current `main` into stale branches before judging their CI — PR-green under an old workflow ≠ main-green.
4. Gate: "Tests (backend + frontend)" green + MERGEABLE.

## Agent skills

- Pinned skills under `.agents/skills/` (locked by `skills-lock.json`): **vite** (Vite config/HMR/builds/Vitest), **fastapi-python** (FastAPI/Pydantic patterns). Repository rules override generic skill guidance.
- Published skills: `skills/voicestudio/` (audio workflows via the REST API), `skills/voicestudio-maintainer/` (triage/review/CI/release prep).
- Issue tracker: GitHub Issues on `debpalash/VoiceStudio` via `gh` CLI (`docs/agents/issue-tracker.md`).
- Triage labels: five canonical roles (`docs/agents/triage-labels.md`).
- Domain docs: `docs/adr/` + root `CONTEXT.md` (lazily created; skip when absent).
