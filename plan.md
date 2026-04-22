# Plan: Centralized /v1 Root Router

The current approach scatters the version in 25+ places — the robust pattern centralizes it once.

## What's wrong now

Every router is registered as `app.include_router(x, prefix="/v1")`. The version string is repeated 25+ times. Adding v2 later means touching every callsite again.

## Target architecture

`main.py` ends up with one line for versioned API:

```python
app.include_router(v1_router)
```

Future v2 is just:

```python
app.include_router(v2_router)
```

## Phase 1 — Consolidate prefix (low-risk, no file moves)

**Step 1.1** — Create `mcpgateway/api/__init__.py` (empty namespace package)

**Step 1.2** — Create `mcpgateway/api/v1/__init__.py` with a `build_v1_router(settings)` factory function that:

- Creates `v1_router = APIRouter(prefix="/v1")`
- Unconditionally includes the always-on inline routers from `main.py` (`protocol_router`, `tool_router`, `resource_router`, `prompt_router`, `gateway_router`, `root_router`, `server_router`, `metrics_router`, `tag_router`, `export_import_router`)
- Conditionally includes feature-flagged routers (`a2a_router`, `observability_router`, `log_search_router`, `toolops_router`, `cancellation_router`, `tool_plugin_bindings_router`, etc.) based on `settings.*`
- Handles the auth cluster: `email_auth_router` at sub-prefix `/auth/email`, `auth_router`, `sso_router`, `teams_router` at `/teams`, `tokens_router`, `rbac_router`
- Handles the LLM cluster: `llm_config_router` at `/llm`, `llm_admin_router` at `/admin/llm`, `llmchat_router`; `llm_proxy_router` stays on `app` directly (its prefix is `settings.llm_api_prefix`, not necessarily `/v1`)
- Returns the assembled `v1_router`

**Step 1.3** — Update `main.py` router assembly block (~L11743–11970):

- Remove all `app.include_router(x, prefix="/v1")` calls
- Replace with: `v1_router = build_v1_router(settings)` then `app.include_router(v1_router)`
- The inline router definitions (`protocol_router = APIRouter(prefix="/protocol")` etc.) stay in `main.py` for now — they're passed into the factory
- Unversioned routers stay on `app` directly: `well_known_router`, `oauth_router`, `utility_router`, `version_router`, `server_well_known_router`
- The `llm_proxy_router` stays on `app` directly since its prefix is runtime-configured

**Step 1.4** — Strip the now-redundant `/v1` from the compound includes:

- `prefix="/v1/auth/email"` → sub-prefix `"/auth/email"` (since `v1_router` already provides `/v1`)
- `prefix="/v1/llm"` → sub-prefix `"/llm"`
- `prefix="/v1/admin/llm"` → sub-prefix `"/admin/llm"`
- `prefix="/v1/admin/runtime"` → sub-prefix `"/admin/runtime"`
- `prefix="/v1/teams"` → sub-prefix `"/teams"`

## Phase 2 — Extract route modules (deferred, independent of Phase 1)

Move the massive inline route handlers out of `main.py` into `mcpgateway/api/v1/` modules:

```
mcpgateway/api/v1/
├── __init__.py       ← build_v1_router() (Phase 1)
├── protocol.py
├── tools.py
├── resources.py
├── prompts.py
├── gateways.py
├── roots.py
├── servers.py
├── metrics.py
├── tags.py
├── export_import.py
└── a2a.py
```

Each module exports its `router = APIRouter(prefix="/tools")`. `__init__.py` imports and includes them. `main.py` shrinks dramatically.

This phase can be done file-by-file without touching the others — a safe, incremental migration.

## What does NOT change

| Item | Stays where |
|------|------------|
| `well_known_router` | `app.include_router(well_known_router)` — RFC, no version |
| `server_well_known_router` | `app.include_router(..., prefix="/servers")` — RFC, no version |
| `oauth_router` | `app.include_router(oauth_router)` — `/oauth` stays unversioned |
| `version_router` | `app.include_router(version_router)` — `/version` endpoint |
| `utility_router` | `app.include_router(utility_router)` — `/_internal/*` must stay at root |
| `llm_proxy_router` | `app.include_router(llm_proxy_router, prefix=settings.llm_api_prefix)` — prefix is runtime config |
| Middleware stack | Completely unchanged |
| All test paths | Already updated to `/v1/*` — no regression |

## Scope and risk

| Phase | Files changed | Risk |
|-------|--------------|------|
| Phase 1 | `main.py` (router assembly block only), 2 new `api/` files | Low — pure structural refactor, no handler logic moves |
| Phase 2 | One file per router group, `main.py` shrinks | Medium — large moves, but each is independently testable |
