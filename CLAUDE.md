# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this project is

**Ombre Brain** is a long-term *emotional* memory system for Claude, exposed over
MCP. Memories are tagged on Russell's valence/arousal circumplex, stored as
Obsidian-compatible Markdown files, decayed with a modified Ebbinghaus forgetting
curve, and retrieved through a dual-channel (keyword + vector) search.

The project is **bilingual (Chinese-first)**. User-facing docs, code comments, log
messages, prompts, and the test dataset are predominantly Chinese; identifiers and
config keys are English. When editing, match the surrounding language — keep
Chinese comments Chinese, and write new user-facing strings in both where the file
already does so.

There are two distinct "Claude-facing" documents — do not confuse them:
- **`CLAUDE.md`** (this file): instructions for an assistant *editing the codebase*.
- **`CLAUDE_PROMPT.md`**: the runtime usage guide for Claude *using the memory
  system* (goes in a system prompt). If you change tool behavior, semantics, or the
  conversation-start sequence, update `CLAUDE_PROMPT.md` too.

## Architecture

```
Claude ──MCP── server.py ──┬── bucket_manager.py   CRUD + dual-channel search + wikilinks
                           ├── dehydrator.py        LLM compression + auto-tagging + merge/split
                           ├── decay_engine.py      forgetting curve + auto-archive/resolve
                           └── embedding_engine.py  Gemini embeddings + SQLite + cosine search
                                       │
                                    utils.py        config/env loading, logging, IDs, path safety, token est.
```

- `server.py` — single MCP entry point (FastMCP). Registers the 6 tools, the FastAPI
  Dashboard + `/api/*` endpoints, auth, hooks (`/breath-hook`, `/dream-hook`), and the
  webhook pusher. Largest file (~1900 lines); `if __name__ == "__main__"` at the
  bottom dispatches stdio vs. HTTP transport.
- **Storage is the filesystem**: each memory ("bucket") is one `.md` file with YAML
  frontmatter under `buckets_dir`. Subdirectories: `permanent/`, `dynamic/<domain>/`,
  `archive/`, `feel/`. There is no relational DB for memories.
- **Embeddings live separately** in `embeddings.db` (SQLite, 3072-dim). They are
  derived data — deletable and regenerable via `backfill_embeddings.py`.
- **Dehydration cache** is `dehydration_cache.db` (SQLite). `*.db` files are
  git-ignored.

Authoritative deep-dive: **`INTERNALS.md`** (module table, every hardcoded constant,
full `config.yaml` key table, design-decision log). Read it before changing scoring,
decay, or search behavior. **`BEHAVIOR_SPEC.md`** is the behavioral spec (scenarios,
the degradation-behavior table). Keep both in sync with code changes.

## The 6 MCP tools (in `server.py`)

| Tool | Purpose |
|------|---------|
| `breath` | Surface (no args) or search (with `query`) memories; dual-channel keyword+vector; supports `domain`/`valence`/`arousal`/`importance_min` filters. `domain="feel"` reads feel buckets. |
| `hold`   | Store one memory (auto-tag → embed → merge similar). `feel=True` stores the model's own reflection into `feel/`. |
| `grow`   | Diary digest: LLM splits a long passage into 2–6 buckets, each stored via the `hold` path. |
| `trace`  | Edit metadata / content, mark `resolved`/`pinned`/`digested`, or `delete`. |
| `pulse`  | System status + bucket listing. |
| `dream`  | Conversation-start self-reflection: recent buckets + crystallization hints. |

Bucket types: `dynamic` (decays), `permanent` (pinned/protected, importance=10, never
decays), `feel` (model reflections — excluded from normal surfacing/decay/merge),
`archived` (decayed out).

## Conventions to preserve

- **No silent degradation of dehydration.** If the LLM API is unavailable, `hold`/`grow`
  raise an explicit `RuntimeError` — they must NOT fall back to truncation or local
  tagging (mis-classified memories are worse than an error). This is a deliberate design
  decision; see `INTERNALS.md` §5.4 and the degradation table in `BEHAVIOR_SPEC.md`.
  Vector search, by contrast, *does* degrade gracefully to fuzzy matching.
- **User-supplied `valence`/`arousal` win over LLM analysis** in `hold` (incl. `0.0`);
  don't let `analyze()` overwrite explicit values (regression B-09).
- **`feel` buckets keep `domain=[]`** — never backfill them with `["未分类"]` (B-10).
- **`resolved` never deletes or auto-archives** a bucket; it down-weights (×0.05, or
  ×0.02 if also digested) so keyword search can still surface it (B-01). Only `decay`
  archiving and explicit `delete=True` remove buckets.
- **Hardcoded magic numbers are catalogued** in `INTERNALS.md` §3. If you change a
  scoring/decay/threshold constant, update that table and the relevant test.
- **Config precedence: env var > `config.yaml` > hardcoded default.** All env vars are
  read in `utils.py` and injected into the config dict. Add new env vars there and
  document them in `ENV_VARS.md` and `INTERNALS.md` §1.
- **Never persist `OMBRE_API_KEY` to `config.yaml`**; keys come from env. The `/api/config`
  endpoint masks keys.
- **Path safety**: all bucket file access goes through `safe_path()` (path-traversal
  guard). Don't bypass it.
- Bucket IDs are 12-char UUID hex; bucket names are clamped to 80 chars (`utils.py`).

## Configuration

- Copy `config.example.yaml` → `config.yaml` (git-ignored). It is the source of truth
  for default values and is heavily commented (bilingual).
- Key env vars (full list in `ENV_VARS.md`): `OMBRE_API_KEY`, `OMBRE_BASE_URL`,
  `OMBRE_TRANSPORT` (`stdio`/`sse`/`streamable-http`), `OMBRE_PORT` (default 8000),
  `OMBRE_BUCKETS_DIR`, `OMBRE_DASHBOARD_PASSWORD`, `OMBRE_HOOK_URL`/`OMBRE_HOOK_SKIP`,
  and the `OMBRE_DEHYDRATION_*` / `OMBRE_EMBEDDING_*` model overrides.
- Default dehydration provider in config is DeepSeek; the recommended free setup is
  Google AI Studio (`gemini-2.5-flash-lite` for dehydration, `gemini-embedding-001`
  for embeddings). Any OpenAI-compatible endpoint works via `base_url` + `model`.

## Running locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp config.example.yaml config.yaml
export OMBRE_API_KEY="your-key"        # needed for hold/grow/dream and embeddings

# Local stdio (Claude Desktop):
python server.py
# Remote HTTP (Claude.ai / tunnel / Dashboard):
OMBRE_TRANSPORT=streamable-http python server.py   # serves :8000, Dashboard at /dashboard, MCP at /mcp
```

Requires Python 3.11+ (Docker image and CI use 3.12). Dependencies: `mcp`, `openai`
(any OpenAI-compatible client), `rapidfuzz`, `python-frontmatter`, `pyyaml`, `jieba`
(Chinese segmentation), `httpx`, `numpy`, `scikit-learn`.

## Testing

CI (`.github/workflows/tests.yml`, Python 3.12) runs exactly:

```bash
pip install pytest pytest-asyncio
python -m pytest tests/test_scoring.py tests/test_feel_flow.py -v --asyncio-mode=auto   # no API key needed
python -m pytest tests/test_llm_quality.py -v --asyncio-mode=auto                        # only if OMBRE_API_KEY is set
```

Committed test suite (`tests/`):
- `conftest.py` — shared fixtures. `test_config` builds an isolated `tmp_path` bucket
  dir with **spec-correct** scoring weights; `buggy_config` documents the pre-fix
  weights for regression contrast; `mock_dehydrator`/`mock_embedding_engine` avoid all
  network calls.
- `dataset.py` — 50 fixed buckets covering all types/quadrants/states.
- `test_scoring.py` — pure-local decay & search scoring regression (no LLM).
- `test_feel_flow.py` — end-to-end feel lifecycle (no LLM).
- `test_llm_quality.py` — LLM auto-tagging baseline; **skipped unless `OMBRE_API_KEY`** is set.

> **Note:** `README.md`/`INTERNALS.md` describe a richer `tests/{unit,integration,regression}/`
> layout (scenarios 01–11, regression B-01…B-10). Those directories are **git-ignored**
> (see `.gitignore`) and are NOT in the repo — the committed suite above is the actual
> state. The B-01…B-10 fixes are real and live in the code; only their dedicated test
> dirs are excluded. Don't assume those paths exist when running tests.

> All tests run against `tmp_path` and never touch real memory data — keep that
> invariant when adding tests.

## Deployment surfaces (don't break their assumptions)

- **Dockerfile** (`python:3.12-slim`): defaults `OMBRE_TRANSPORT=streamable-http`,
  `OMBRE_BUCKETS_DIR=/app/buckets`, exposes 8000, `VOLUME [/app/buckets]`.
- **`docker-compose.yml`**: source build + Cloudflare Tunnel; host vault via
  `OMBRE_HOST_VAULT_DIR` → `/data`, port `18001:8000`.
- **`docker-compose.user.yml`**: prebuilt Docker Hub image (`p0luz/ombre-brain`) for end users.
- **`render.yaml`** (Render, persistent disk at `/opt/render/project/src/buckets`) and
  **`zbpack.json`** (Zeabur, builds via Dockerfile, volume at `/app/buckets`).
- HTTP mode auto-enables CORS and a keepalive ping; the SessionStart hook
  (`.claude/settings.json` → `.claude/hooks/session_breath.py`) auto-calls `breath` and
  only works in HTTP mode (silently exits under stdio).

## Working in this repo

- Develop on the assigned feature branch; commit with clear messages; push with
  `git push -u origin <branch>`. Do not open a PR unless explicitly asked.
- When you change behavior, update the docs that mirror it: `INTERNALS.md` (constants,
  config keys, module table, design decisions), `BEHAVIOR_SPEC.md` (scenarios &
  degradation), `README.md` (user-facing), `ENV_VARS.md` (env vars), and
  `CLAUDE_PROMPT.md` (runtime tool guidance).
- Utility/CLI scripts (each standalone): `backfill_embeddings.py`, `write_memory.py`,
  `check_buckets.py`, `check_icloud_conflicts.py`, `import_memory.py` (history-import
  engine used by the Dashboard Import tab), `migrate_to_domains.py`,
  `reclassify_domains.py`, `reclassify_api.py`.
- The Dashboard is a single static `dashboard.html` served by `server.py`; `/api/*`
  endpoints are password-protected (`/health`, `/*-hook`, `/mcp*` are public).
