# The 10-Phase Reconstruction Pipeline

Each phase has an **exit gate**. Do not advance on hope -- advance on evidence.
Record the gate result in the kanban task comment before creating the next task.

| Phase                      | Owner                                  | Deliverable                                                               | Exit Gate                                                                              |
| -------------------------- | -------------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 0. Scope & Authorization   | rea + human                            | Written scope, target authorization, disposable-instance confirmation     | User confirmed authorization AND target is disposable/seeding is possible              |
| 1. Reconnaissance          | rea                                    | Passive DOM/AX/network/screenshot + static bundle analysis                | `inspect_web_page` + `analyze_web_bundle` returned route + endpoint inventories        |
| 2. Scenario Capture        | rea                                    | Full evidence bundle (network bodies, DOM per step, storage, screenshots) | Every scenario step `status: "completed"`; read+write coverage present                 |
| 3. Coverage Reconciliation | rea                                    | `environment-coverage` view; withheld-write remediation                   | All writes served, or each withheld write has a documented reason                      |
| 4. Contract Extraction     | rea                                    | `api-surface`, `environment-manifest`, `environment-state`                | Path templates, entity identity fields, and body shapes extracted for every collection |
| 5. Environment Export      | rea                                    | `agent-env/` bundle                                                       | `agent-env run <folder>` serves reads; coverage report attached                        |
| 6. Backend Implementation  | rea                                    | Working backend implementing the captured contracts                       | Contract tests pass against `agent-env`; server logic re-implemented (not replayed)    |
| 7. Frontend Reconstruction | rea                                    | 1:1 UI + interaction parity                                               | `compare_web_captures` + `compare_web_screenshots` show parity or explained deltas     |
| 8. Deployment              | coolify                                | Live deployment on the target domain                                      | Domain gate: HTTP 200 on domain + health endpoint returns ok                           |
| 9. Deployed-Domain QA      | coolify (lead) + rea (contract review) | QA report vs. original behavior                                           | Every journey in the QA matrix passes or has a signed-off waiver                       |
| 10. Sign-off               | human                                  | Acceptance report                                                         | User accepts; kanban board shows all tasks done                                        |

## Phase 0 -- Scope and Authorization (BLOCKING)

Reverse engineering a target you are not authorized to analyze is out of scope.
Before any capture:

1. Confirm the user owns the target, has written authorization, or is doing
   security research under a scoped engagement.
2. Confirm the target instance is **disposable** or that you can seed/reset it.
   REA does not start, seed, or reset applications. If the target is a live
   production system with real user data, STOP and tell the user.
3. Confirm credentials will be supplied via environment variables or the vault
   -- never pasted in chat.
4. Write the scope to the kanban task body and create the child tasks.

If any of these are unresolved, ask the user. Do not proceed on assumption.

## Phase 1 -- Reconnaissance (PASSIVE ONLY)

Passive means no writes to the target. Establish the shape before touching anything.

```
list_browser_targets                  -> pick target_id
inspect_web_page                      -> DOM, AX tree, network requests, scripts, storage
capture_web_screenshot                -> visual baseline PNG
analyze_web_bundle                    -> route candidates, endpoint inferences, vendor fingerprints
```

For an Electron target use `list_electron_targets` / `inspect_electron_page`
instead. For a shipped bundle with no live instance use `analyze_web_bundle`
with local files, plus `analyze_javascript_application` on extracted sources.

**Exit gate:** you can name every route and every API endpoint candidate.

## Phase 2 -- Scenario Capture

Design one `capture_browser_scenario` per user journey. The scenario structure
that produces usable state is non-negotiable:

- Sign in through the UI; declare the password as a `secrets[]` entry so it is
  redacted in the evidence.
- Read each collection **twice** before any write (catches auto-changing fields).
- Read each listed object by ID (up to ~10 per collection).
- Write through the UI (`fill` + `click` submit), and add a `request` step with
  `save_response` on create so the new ID is captured.
- After each write: read the object by ID, then read the list.
  Create 1 -> read 1 -> list -> create 2 -> read 2 -> list.
- `wait_for_timeout` 1100ms between creates and before the first edit.
- Update both objects with varied fields. Toggle actions twice. Delete both and
  verify the 404/deleted state.
- Every step carries `timeout_ms` and `expected_status`.

Target 60-160 requests per scenario. Every step must report
`status: "completed"`.

**Exit gate:** evidence bundle exists, all steps completed, network bodies captured.

## Phase 3 -- Coverage Reconciliation

```
inspect_analysis_view
  source: { kind: "retained-evidence", evidence_id: "<id>" }
  view:   { kind: "environment-coverage", template_rules: [...] }
```

Withheld-write reasons and what each one demands:

| Reason                | Action                                                                        |
| --------------------- | ----------------------------------------------------------------------------- |
| `no-state-target`     | Check the path template; record missing reads; route may not support mutation |
| `no-observation`      | Re-record: read entity by ID after write, read before update/delete           |
| `unconfirmed`         | Re-record: read entity + list after write                                     |
| `incomplete-entities` | Re-record: read entity through every list view                                |
| `unexplained-change`  | Re-record **once** with a second, different write call                        |
| `no-call`             | Re-record once with a write that succeeds                                     |
| `side-effect`         | **STOP** -- the app changes other entities                                    |
| `contradicted`        | **STOP** -- a later read disagrees with the modelled effect                   |
| `body-unusable`       | Re-record the UI write once with `request` steps; else stop                   |

Maximum **2 re-recordings** per withheld write, and only **1** when the reason
is about application behaviour (`side-effect`, `contradicted`).

**Exit gate:** every write is served, or every withheld write has a documented
reason and an explicit decision (implement from scratch / leave unsupported).

## Phase 4 -- Contract Extraction

Run three views over the retained evidence:

- `api-surface` -- method, origin, path template, observed query **names**,
  value-free body shapes. Path format matches are **inferences**, not handler
  identity.
- `environment-manifest` -- primary-origin requests projected to named tools
  with path templates and record pointers.
- `environment-state` -- entity types, identity fields, entities with recorded
  values, per-tool render plans, write plans with served/withheld status.

**Exit gate:** every collection has a path template, an entity identity field,
and a request/response shape recorded.

## Phase 5 -- Environment Export

```
export_environment_bundle
  source: { kind: "retained-evidence", evidence_id: "<id>" }
  output_directory: "<abs path, must not already exist>"
  name: "<required for localhost/IP targets>"
  template_rules: [...]
  package_tarball: false
```

Produces `agent-env/` with `.agentenv/config.toml`, `data/<tool>.json`,
`data/reset`, `data/add`, `data/get`, `capture/`, optional `scripts/`, a
coverage report, and a README whose run command is `agent-env run <folder>`.

Also export the frontend sources when you need them:

```
export_web_scripts   -> JS bytes into analysis_input for analyze_javascript_application
```

**Exit gate:** the bundle's read tools answer with the captured bodies.

### Phase 5 exit gate -- how to actually serve the bundle (VERIFIED 2026-11-20)

`agent-env` (the `scaleapi/agentenv-framework` package, `bin: agent-env`) is a
**separate install** -- it is not a `rea` subcommand:

```
uv tool install --tool-dir /workspace/rea/toolchain/uv-tools agentenv-framework
export PATH="/workspace/rea/toolchain/local/bin:$PATH"
export UV_TOOL_DIR=/workspace/rea/toolchain/uv-tools
```

`agent-env run <folder>` needs a **Docker daemon**. This host has **no Docker
and cannot get one**: AppArmor sets
`kernel.apparmor_restrict_unprivileged_userns=1`, so `unshare --user` fails
with `Operation not permitted` and rootless Docker cannot start. Do not spend
time on it.

**Serve the bundle natively instead.** The bundle's Dockerfile runs exactly one
command, and that command runs on this host without Docker:

```
rea environment-serve <bundle>/envs/<name>/manifest.json \
                      --evidence <bundle>/envs/<name>/evidence.json \
                      --host 127.0.0.1 --port 18765
```

Verified behaviour of that server:

- The MCP endpoint is **`/mcp`** (NOT `/` and NOT `/agentenv`; those 404).
- It is **streamable HTTP**: `POST /mcp` with
  `Accept: application/json, text/event-stream`, and every response after
  `initialize` must carry the `Mcp-Session-Id` header returned by `initialize`.
- Tool calls return `structuredContent` = `{status, content_type, body, replay, state}`.
- `replay.exact_match: true` means the answer came from the retained record.
- `state.source: "state"` means the answer came from state derived at startup;
  `state.view` is `list` or `item`.
- The bundle ships its own verifier at
  `artifacts/check_<name>/verify.py`; run it against the live URL and require
  exit 0.

**HAR import pitfall (VERIFIED):** `readRecordedResponseBody` accepts only
`encoding` absent (producer-decoded text) or `"base64"`. Any other declared
encoding -- including `"utf-8"` -- makes the body opaque, and every response
then replays with `body: null` and `state.reason: "body-unusable"`. When
importing a HAR, **omit `encoding` entirely**. Also omit `postData` on
bodyless requests: `postData: null` fails HAR schema validation.

**Write-tool guardrail (VERIFIED):** a write whose effect no later read
confirmed is not merely refused at call time -- it is **not exposed as a tool
at all** (`tools/list` omits it; calling it returns
`-32602 Tool <name> not found`). This is the correct behaviour; do not work
around it. Capture a confirming read-after-write to make a write servable.

## Phase 6 -- Backend Implementation (THE PART REA CANNOT DO FOR YOU)

The AgentEnv bundle **replays recorded reads and serves only confirmed writes**.
It does not execute server logic. The real backend must be implemented from the
extracted contracts.

What you must build yourself:

- Server-side business logic, validation, transformations, workflows
- Database schema and migrations (inferred from entity shapes, not observed)
- Auth implementation: password hashing, token issuance, session store, OAuth
- Authorization rules (RBAC, ownership) inferred from observed allow/deny
- Background jobs, queues, webhooks
- File storage and upload handling
- Rate limiting, quotas, throttling
- SSR / SEO if the original had it
- WebSocket server or SSE endpoint
- Email / notification sending
- Search indexing
- Audit logging and admin APIs

Contract compatibility rules (the custom backend must speak the same shapes):

| AgentEnv tool  | Custom backend must implement                                                         |
| -------------- | ------------------------------------------------------------------------------------- |
| `data/reset`   | `POST /agentenv/data/reset` -- truncate + seed from captured initial reads            |
| `data/get`     | `POST /agentenv/data/get` -- `{type, id}` -> entity                                   |
| `data/add`     | `POST /agentenv/data/add` -- `{type, data}` -> `{id, ...entity}`                      |
| `<read-tool>`  | `GET /api/<collection>` and `GET /api/<collection>/:id` matching the path template    |
| `<write-tool>` | `POST/PATCH/DELETE /api/<collection>[:id]` with **identical** request/response shapes |

Validate with Zod schemas generated from the `api-surface` body shapes.

**Exit gate:** contract tests pass against `agent-env`, and the same tests pass
against the custom backend with identical payloads.

## Phase 7 -- Frontend Reconstruction

Target structure:

```
reconstructed-frontend/   Vite/Next app, proxying /api -> backend
reconstructed-backend/    Fastify/Express/Hono + Prisma/Drizzle + Dockerfile
```

Verification tools:

| Comparison                                                            | Tool                               |
| --------------------------------------------------------------------- | ---------------------------------- |
| Live original vs. reconstructed (DOM, AX, URL, storage, requests, WS) | `compare_web_captures`             |
| Screenshot pixel diff                                                 | `compare_web_screenshots`          |
| Static bundle vs. runtime                                             | `reconcile_javascript_runtime`     |
| Two app versions                                                      | `compare_application_versions`     |
| Two exported callable shapes                                          | `compare_javascript_export_shapes` |

`compare_web_captures` takes `before.inspection` + `after.inspection` (passive
`inspect_web_page` results) or `before_scenario` + `after_scenario` (scenario
`normalized_result`). Scenario steps are aligned by exact `step_id`.

**Exit gate:** capture parity, or every remaining delta is explained and
accepted in writing.

## Phase 8 -- Deployment (OWNED BY COOLIFY)

Create a kanban task assigned to `coolify` using the handoff contract in
`handoff.md`. Coolify owns discovery, configuration, deployment, domain gate,
cleanup, and audit. Do not deploy from the `rea` profile.

**Exit gate:** the domain gate passes -- HTTP 200 on the configured hostname and
the health endpoint returns the expected body.

### 8.1 Deployment authorisation gate (MANDATORY)

Deployment is a **production side effect**. It must never happen on inference.
Before any `mcp__coolify_server__deploy`, `manage_application`, `manage_dns`,
or `manage_project` call that creates or mutates live resources, the Coolify
worker must find an explicit authorisation in the task brief. The brief MUST
contain one of these literal tokens:

| Token                     | Meaning                                       |
| ------------------------- | --------------------------------------------- |
| `DEPLOY-AUTH: production` | Deploy to the live domain.                    |
| `DEPLOY-AUTH: staging`    | Deploy to a staging host only.                |
| `DEPLOY-AUTH: dry-run`    | **No live deployment.** Plan and report only. |
| `DEPLOY-AUTH: none`       | No deployment permitted at all.               |

Rules, in order of precedence:

1. **No token -> no deployment.** If the brief lacks a `DEPLOY-AUTH:` token, the
   worker must NOT deploy. It may plan, discover, and report a dry-run plan.
2. **Absence of a token is not permission.** A brief saying "no real
   deployment", "just a smoke test", "dry run", or "don't deploy" is a
   **negative** instruction and is equivalent to `DEPLOY-AUTH: none` -- even if
   it also contains words like "deploy", "test the deployment", or "make it
   live". Prose never overrides the token.
3. **`dry-run` / `none` forbids every live-resource mutation.** No
   `deploy`, no `manage_application`, no `manage_dns`, no `manage_project`
   creation, no resource that becomes publicly reachable.
4. **`production` requires a companion confirmation token.** A brief with
   `DEPLOY-AUTH: production` but no `DEPLOY-CONFIRMED: yes` is NOT enough.
   The worker must post a pre-deploy kanban comment naming the target domain,
   the source revision, and the exact Coolify operation, then close the task
   as `blocked` awaiting confirmation. Only a brief carrying **both**
   `DEPLOY-AUTH: production` **and** `DEPLOY-CONFIRMED: yes` authorises the
   deployment to execute.
5. **A smoke test is never a production deploy.** If the brief says
   "smoke test", the worker must use a throwaway host, or `DEPLOY-AUTH:
   dry-run`, and must delete every resource it created when the task closes.
6. **Cleanup is mandatory.** Any task that created live resources records their
   identifiers (application uuid, project uuid, domain) in a `kanban_comment`
   before closing, and deletes them unless the brief asked to keep them.

A worker that cannot satisfy the gate must finish the task as `blocked` with a
comment explaining exactly which token is missing. Blocking is a success; an
unauthorised production deploy is a failure.

## Phase 9 -- Deployed-Domain QA

Coolify leads QA against the deployed domain using the matrix in `qa-matrix.md`.
REA reviews the API contract results. Run the QA against the **deployed domain**,
not against localhost -- localhost does not prove DNS, TLS, proxy, or
certificate behaviour.

**Exit gate:** every journey in the matrix passes or carries a signed-off waiver.

## Phase 10 -- Sign-off

Report: what was reconstructed, what was verified, what was deliberately left
out (withheld writes, unimplemented server behaviour), and the live URL.

---

## Artifact Layout Per Project

```
/workspace/rea/reconstruction/<project-slug>/
├── scope.md              # Phase 0 authorization + scope
├── evidence/             # REA evidence bundles + manifests
│   └── manifest.json     # evidence_id -> phase -> view mapping
├── agent-env/            # exported bundle
├── contracts/            # api-surface, environment-state, schemas
├── reconstructed-frontend/
├── reconstructed-backend/
├── qa/                   # QA reports, screenshots, diffs
└── handoff/              # kanban task ids, deployment uuid, domain
```

## Non-Negotiables

- No capture before authorization and disposable-instance confirmation.
- No secrets in chat, in task bodies, in code, or in logs.
- No blind retries on a failed deploy without a cited fix.
- No "done" without a passing exit gate.
- Never mark a kanban task complete while its children are open.
