# Web Application 1:1 Reconstruction Checklist (REA-based)

## A. The reconstructable targets

| Target Kind                              | Input Required                                                   | What REA Produces                                                                                                                     |
| ---------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Live web application**                 | Accessible URL + credentials via env vars (never in chat)        | AgentEnv bundle (`agent-env/`) that serves recorded reads + writes via replay/state                                                   |
| **HAR / SAZ / mitmproxy / qlog capture** | Local capture file path                                          | AgentEnv bundle via `inspect_web_network_capture` → `export_environment_bundle` (or `rea environment-build <capture.har> <dir>`)      |
| **Electron app (ASAR)**                  | `app.asar` path or extracted tree                                | Static JS analysis via `analyze_javascript_application` (route/endpoint candidates, source recovery via `recover_javascript_sources`) |
| **Electron app (live)**                  | Loopback CDP endpoint + `target_id` from `list_electron_targets` | Passive capture (`analyze_web_bundle`, `inspect_web_page`), scenario capture (`capture_browser_scenario` with connect mode)           |
| **Browser page (live, user-owned)**      | Loopback CDP endpoint + `target_id` from `list_browser_targets`  | Passive inspection (`inspect_web_page`, `capture_web_screenshot`), scenario capture (`capture_browser_scenario`)                      |
| **Shipped SPA bundle**                   | Local JS files or extracted web bundle                           | Static analysis via `analyze_web_bundle` (route inventory, endpoint candidates, vendor fingerprints)                                  |

**Key constraint**: REA does **not** start, seed, or reset applications. The target instance must be disposable or pre-seeded by the user.

---

## B. The AgentEnv bundle lifecycle (ordered steps)

### 1. Reconnaissance (passive, no writes)

| Tool                                             | Consumes                                       | Produces                                                                                                                                                  |
| ------------------------------------------------ | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list_browser_targets` / `list_electron_targets` | CDP endpoint                                   | Available targets with `target_id`                                                                                                                        |
| `inspect_web_page`                               | `cdp_endpoint`, `target_id`, `allowed_origins` | DOM, accessibility tree, network requests (no bodies by default), script sources (opt-in), storage keys (opt-in)                                          |
| `capture_web_screenshot`                         | `cdp_endpoint`, `target_id`                    | Content-addressed PNG artifact (viewport only)                                                                                                            |
| `analyze_web_bundle`                             | `cdp_endpoint`, `target_id`, `allowed_origins` | Static chunk graph, route candidates (`javascript-routes` view), endpoint inferences (`web-bundle:endpoints` view), vendor fingerprints, WebMCP inventory |

### 2. Scenario recording (agent-performed actions with full capture)

| Tool                       | Consumes                                                                            | Produces                                                                                                                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `capture_browser_scenario` | `browser` (launch/connect), `start_url`, `actions[]`, `secrets[]`, `capture` config | **Complete Evidence** (`evidence_id`) with: network (request/response bodies, headers), DOM/accessibility per step, events, storage, screenshots. Actions: `fill`, `click`, `press`, `wait_for`, `request`, `goto`, `wait_for_timeout`. `request` steps with `save_response` capture IDs from UI-triggered writes. |

**Scenario structure (from environments.md):**

- Sign in via UI (`fill` username, `fill` password as secret, `press` Enter, `wait_for` signed-in element)
- Read each collection **twice** before any write (captures auto-changing fields)
- Read each listed object by ID (up to 10/collection)
- Write via UI: `fill` forms, `click` submit buttons, `save_response` on create to capture new ID
- After each write: read object by ID → read list (create 1, read 1, list, create 2, read 2, list)
- `wait_for_timeout` 1100ms between creates and before first edit
- Update both objects with varied fields; toggle actions twice; delete both and verify 404/deleted
- 60–160 requests total; every step has `timeout_ms` and `expected_status`

### 3. Coverage check

| Tool                    | Consumes                                                                                                          | Produces                                                                                                                        |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `inspect_analysis_view` | `source: {kind: "retained-evidence", evidence_id}`, `view: {kind: "environment-coverage", template_rules: [...]}` | Read tools served (from state/replay), write tools served, **withheld writes** with `reason` + `hint` (see reasons table below) |

**Withheld write reasons & remediation:**

| Reason                | Action                                                                    |
| --------------------- | ------------------------------------------------------------------------- |
| `no-state-target`     | Check path template; record missing reads; route may not support mutation |
| `no-observation`      | Re-record: read entity by ID after write; read first for update/delete    |
| `unconfirmed`         | Re-record: read entity + list after write                                 |
| `incomplete-entities` | Re-record: read entity through every list view                            |
| `unexplained-change`  | Re-record **once** with second, different write call                      |
| `no-call`             | Re-record once with a write that succeeds                                 |
| `side-effect`         | **Stop**: app changes other entities                                      |
| `contradicted`        | **Stop**: later read disagrees with modelled effect                       |
| `body-unusable`       | Re-record UI write once with `request` steps; else stop                   |

Max **2 re-recordings** per withheld write (1 for application-behaviour reasons).

### 4. Export bundle

| Tool                        | Consumes                                                                                                                                                                                                          | Produces                                                                                                                                                                                 |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `export_environment_bundle` | `source: {kind: "retained-evidence", evidence_id}` OR `{kind: "evidence-file", path}`, `output_directory` (abs, new), `name` (required for localhost/IP), optional `template_rules`, `package_tarball`, `dry_run` | `agent-env/` folder with: recorded responses, local browser path, derived read tools (state/replay), served write tools, coverage report, `README` with `agent-env run <folder>` command |

**Also available**: `export_web_scripts` — exports JS bytes from a capture to `analysis_input` for `analyze_javascript_application`.

---

## C. Backend reality (what REA can/cannot recover)

### What REA **CAN** recover from a web target

| Artifact                                | Tool / Method                                                                                                                  | Notes                                                                                                                       |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| **API endpoints** (observed)            | `capture_browser_scenario` (network capture), `inspect_web_network_capture` (HAR/SAZ), `analyze_web_bundle` (static inference) | Full request/response bodies, headers, query params when `capture.network.request_body/response_body/header_values: true`   |
| **Path templates**                      | `inspect_analysis_view` with `view: {kind: "api-surface"}` on HAR Evidence                                                     | Groups by method, origin, path template; format matches are **inferences**, not handler identity                            |
| **Auth flows** (as observed)            | Scenario capture: login `request`/`fill` steps, token headers from cookies/localStorage                                        | Secrets (passwords, tokens) are **redacted** via declared `secrets[]`; CSRF tokens, bearer tokens captured as header values |
| **Data shapes** (request/response JSON) | Captured bodies in scenario/HAR                                                                                                | Value-free shapes retained; actual values only in recordings                                                                |
| **Entity identity fields**              | `environment-state` view on HAR/capture                                                                                        | Infers identity fields from reads before first write                                                                        |
| **WebSocket shapes**                    | `analyze_web_bundle` with `include_websocket_shapes: true`                                                                     | Message shapes only, not full protocols                                                                                     |

### What REA **CANNOT** recover (agent must implement independently)

| Gap                                                                       | Why                                      | Agent Responsibility                                       |
| ------------------------------------------------------------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| **Server-side business logic**                                            | Only observed I/O; no server code access | Re-implement validation, transformations, workflows        |
| **Database schema / queries**                                             | Never observed; only API shapes          | Design schema from API contracts; write migrations         |
| **Auth implementation** (password hashing, token issuance, session store) | Only client-side token usage observed    | Implement JWT/session logic, password hashing, OAuth flows |
| **Authorization rules** (RBAC, ownership)                                 | Only observed allow/deny outcomes        | Infer from test matrix; implement policy engine            |
| **Background jobs / async processing**                                    | Not visible in frontend capture          | Design queue/worker system; implement webhooks             |
| **File storage / upload handling**                                        | Only multipart requests observed         | Implement storage backend (S3, local, etc.)                |
| **Rate limiting, quotas, throttling**                                     | May appear as 429 but logic hidden       | Implement token bucket / sliding window                    |
| **Server-side rendering / SEO**                                           | Only hydrated DOM observed               | Implement SSR if needed (Next.js, etc.)                    |
| **Real-time / push** (WebSocket server, SSE)                              | Only client frames captured              | Implement server endpoint + connection mgmt                |
| **Email / notification sending**                                          | Never observed from frontend             | Integrate SendGrid, etc.                                   |
| **Search indexing**                                                       | Only search API observed                 | Implement Elasticsearch/Meilisearch/Postgres FTS           |
| **Audit logging / admin APIs**                                            | Not exercised in typical user flow       | Build admin panel + audit trail                            |

**Honest limit**: The AgentEnv bundle **serves reads from recorded state/replay** and **serves writes only when a later read confirms the effect**. It does **not** execute server logic. The reconstructed backend must implement the actual semantics.

---

## D. Verification primitives (exact tool names)

| Comparison                                    | Tool                                                             | Consumes                                                                                                                                             | What It Compares                                                                                                                                                                                 |
| --------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Web capture vs. web capture**               | `compare_web_captures`                                           | `before.inspection` + `after.inspection` (passive `inspect_web_page` results) OR `before_scenario` + `after_scenario` (scenario `normalized_result`) | DOM, accessibility, URL, history, storage, events, HTTP requests (ordered), WebSocket streams. Aligns scenario steps by exact `step_id`. Optional `normalization` rules for literal replacement. |
| **Screenshot vs. screenshot**                 | `compare_web_screenshots`                                        | `before` + `after` PNG artifacts (sha256, bytes, base64)                                                                                             | Exact changed-pixel ratios, per-channel deltas. No OCR.                                                                                                                                          |
| **Network capture (HAR/SAZ) vs. API surface** | `inspect_analysis_view` + `view: {kind: "api-surface"}`          | Retained `inspect_web_network_capture` Evidence                                                                                                      | Groups request lines by method, origin, path template; observed query **names** (not values); format matches are inferences                                                                      |
| **Network capture vs. Environment manifest**  | `inspect_analysis_view` + `view: {kind: "environment-manifest"}` | Retained HAR Evidence                                                                                                                                | Projects primary origin requests → named tools with path templates, parameters, observed query names, value-free body shapes, record pointers                                                    |
| **Network capture vs. Environment state**     | `inspect_analysis_view` + `view: {kind: "environment-state"}`    | Retained HAR Evidence + `template_rules`                                                                                                             | Entity types + identity fields, entities with recorded values, documents, per-tool render plans; write plans with served/withheld status                                                         |
| **Static JS bundle vs. runtime**              | `reconcile_javascript_runtime`                                   | Static `analyze_javascript_application` Evidence + passive browser/Electron Evidence                                                                 | **Inference only** — bundle observed at runtime ≠ every module executed. Path/digest disagreement = mismatch.                                                                                    |
| **Two app versions (static)**                 | `compare_application_versions`                                   | Two `analyze_javascript_application` Evidence IDs                                                                                                    | Digest, source-map, structural fingerprint, non-module semantic matches. Module ordinals/minified names **not** persistent identity.                                                             |
| **Two exported callable shapes**              | `compare_javascript_export_shapes`                               | Two analysis Evidence IDs + explicit module paths + export names                                                                                     | Property inventories, presence changes (`added`/`removed`/`unknown`). Static inference only.                                                                                                     |
| **Native binaries**                           | `compare_artifacts` / `compare_functions`                        | Two `inspect_artifact` / `analyze_function` Evidence IDs                                                                                             | Inventory by occurrence path, content, metadata, graph relations; function-level pseudocode/assembly comparison.                                                                                 |

---

## E. Build/serve shape (how the reconstructed app runs)

### AgentEnv bundle structure (what `export_environment_bundle` writes)

```
agent-env/
├── .agentenv/
│   ├── config.toml          # Model endpoints, roles (judge, default)
│   └── config.toml.example
├── data/
│   ├── reset                # POST /agentenv/data/reset — restores seed state
│   ├── add                  # POST /agentenv/data/add — adds entity
│   ├── get                  # POST /agentenv/data/get — reads entity
│   └── <tool-name>.json     # Per-tool read/write implementations
├── scripts/                 # Exported JS (from export_web_scripts) if any
├── capture/                 # Original capture Evidence (redacted)
├── README.md                # Run: `agent-env run <folder>`
└── package.json             # rea-agents dependency (bundled version)
```

### How the reconstructed app is served

| Mode                               | What Runs                                                                                                                                                                 | How API is Stubs/Implemented                                                                                                                                                                                                                                                                                                      |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AgentEnv (replay/state)**        | `rea environment-serve <bundle>/envs/<name>/manifest.json --evidence <bundle>/envs/<name>/evidence.json --host 127.0.0.1 --port 18765` starts an MCP server at **`/mcp`** | **Read tools**: answered from recorded state (derived at startup) or replayed from capture.<br>**Write tools**: served only if a later read confirmed effect; applies effect to in-memory state.<br>**Unconfirmed writes**: not exposed as tools at all (`tools/list` omits them; a call returns `-32602 Tool <name> not found`). |
| **Dev server (for frontend work)** | Standard Vite/Webpack/Next dev server pointing at reconstructed frontend                                                                                                  | **API proxy**: `vite.config.ts` proxy `/api` → `http://127.0.0.1:<port>/mcp` OR custom Express/Fastify stub implementing the same contracts                                                                                                                                                                                       |
| **Static bundle**                  | `npm run build` → `dist/` served by Nginx/Caddy                                                                                                                           | **API**: Separate backend process (see below) or serverless functions matching AgentEnv contracts                                                                                                                                                                                                                                 |

> **CORRECTED 2026-11-20 (verified on a real bundle).** The endpoint is
> **`/mcp`**, not `/agentenv`; `/` and `/agentenv` both return
> `404 No route for GET ...`. It is streamable HTTP: `POST /mcp` with
> `Accept: application/json, text/event-stream`, and every request after
> `initialize` must carry the `Mcp-Session-Id` header from `initialize`.
> Tool calls return `structuredContent` = `{status, content_type, body, replay, state}`.
> `agent-env run <folder>` is the container path and needs Docker — see below.

### Backend implementation shape (agent builds this)

```
reconstructed-backend/
├── src/
│   ├── routes/              # One file per path template from api-surface
│   │   ├── folders.ts       # GET/POST /api/folders, GET/PATCH/DELETE /api/folders/:id
│   │   └── ...
│   ├── models/              # DB schema (Prisma/Drizzle/SQL) inferred from entity shapes
│   ├── auth/                # JWT/session, password hash, OAuth — IMPLEMENT FROM SCRATCH
│   ├── middleware/          # Auth guard, rate limit, validation (Zod from captured shapes)
│   ├── services/            # Business logic (re-implemented from observed I/O patterns)
│   └── index.ts             # Fastify/Express/Hono app + /agentenv compatibility layer
├── prisma/schema.prisma     # Or drizzle/schema.ts
├── package.json
└── Dockerfile
```

### Contract compatibility (AgentEnv ↔ Custom Backend)

| AgentEnv Tool  | Custom Backend Must Implement                                                         |
| -------------- | ------------------------------------------------------------------------------------- |
| `data/reset`   | `POST /agentenv/data/reset` — truncate + seed from captured initial reads             |
| `data/get`     | `POST /agentenv/data/get` — `{type, id}` → entity                                     |
| `data/add`     | `POST /agentenv/data/add` — `{type, data}` → `{id, ...entity}`                        |
| `<read-tool>`  | `GET /api/<collection>` + `GET /api/<collection>/:id` (match path templates)          |
| `<write-tool>` | `POST/PATCH/DELETE /api/<collection>[:id]` with **identical request/response shapes** |

**Critical**: The custom backend must accept **exact same JSON shapes** (field names, types, nesting) as captured. Use Zod schemas generated from `api-surface` view for validation.

### Running the full stack locally (VERIFIED 2026-11-20)

```bash
# Terminal 1: AgentEnv (replay backend for verification) -- NATIVE, no Docker
rea environment-serve ./agent-env-myapp/envs/<name>/manifest.json \
                      --evidence ./agent-env-myapp/envs/<name>/evidence.json \
                      --host 127.0.0.1 --port 18765

# Terminal 2: Custom backend (implements real logic)
cd reconstructed-backend && npm run dev  # --port 3001

# Terminal 3: Frontend dev server (proxies /api to custom backend)
cd reconstructed-frontend && npm run dev  # --port 5173, proxy /api → localhost:3001

# Verification: compare_web_captures between live capture and AgentEnv replay
# or run the bundle's own verifier:
# python3 ./agent-env-myapp/artifacts/check_<name>/verify.py http://127.0.0.1:18765/mcp
```

**Docker note**: `agent-env run <folder>` is the container path and needs a Docker
daemon. This host cannot run Docker (AppArmor blocks unprivileged user
namespaces). The native `rea environment-serve` runs the exact same logic
without Docker and is the correct path here.

---

## Quick Reference: Core Tool Names (exact)

| Category                | Tools                                                                                                                                                                                         |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Discovery**           | `list_browser_targets`, `list_electron_targets`, `inspect_electron_fuses`                                                                                                                     |
| **Passive capture**     | `inspect_web_page`, `capture_web_screenshot`, `analyze_web_bundle`, `inspect_web_network_capture`                                                                                             |
| **Active capture**      | `capture_browser_scenario` (launch/connect, actions, secrets, network capture)                                                                                                                |
| **Analysis**            | `inspect_analysis_view` (views: `environment-coverage`, `api-surface`, `environment-manifest`, `environment-state`, `javascript-routes`, `web-bundle:endpoints`, `browser-scenario`)          |
| **Export**              | `export_environment_bundle`, `export_web_scripts`, `export_evidence_bundle`                                                                                                                   |
| **Verification**        | `compare_web_captures`, `compare_web_screenshots`, `observe_web_execution`, `reconcile_javascript_runtime`                                                                                    |
| **JS static**           | `analyze_javascript_application`, `recover_javascript_sources`, `trace_application_feature`, `compare_application_versions`, `compare_javascript_export_shapes`                               |
| **Runtime**             | `observe_web_execution` (coverage window), `inspect_web_event_listeners` (listener sources)                                                                                                   |
| **Desktop (if needed)** | `start_build_computer`, `act_on_build_computer`, `start_build_capture`, `mark_build_checkpoint`, `stop_build_capture`, `export_build_environment`, `check_build_task`, `close_build_computer` |

---

## Minimal viable reconstruction workflow (checklist)

- [ ] User confirms target is disposable/seeded; credentials via env vars
- [ ] `list_browser_targets` → pick `target_id` (or `list_electron_targets`)
- [ ] `inspect_web_page` + `capture_web_screenshot` → map login form, list/edit pages, API calls
- [ ] `analyze_web_bundle` → route candidates, endpoint inferences, vendor fingerprints
- [ ] Design `capture_browser_scenario` with: secrets, sign-in, 2× reads per collection, 2× creates/reads/lists per write type, updates, toggles, deletes, 1100ms waits
- [ ] Run scenario → verify all steps `status: "completed"`
- [ ] `inspect_analysis_view` + `environment-coverage` → check served/withheld writes
- [ ] Re-record withheld writes (max 2× each, only for recording gaps)
- [ ] `export_environment_bundle` → `agent-env/` folder
- [ ] `agent-env run ./agent-env` → verify reads/writes work
- [ ] `export_web_scripts` → frontend JS for `analyze_javascript_application` if needed
- [ ] Build custom backend from `api-surface` + `environment-state` contracts
- [ ] Proxy frontend dev server to custom backend
- [ ] `compare_web_captures` (live vs. AgentEnv replay) → verify parity
- [ ] Document gaps: every withheld write, every backend logic piece re-implemented

---

_Generated from REA skill references: environments.md, javascript-applications.md, runtime-observation.md, evidence-workflows.md, desktop-build.md, SKILL.md_
