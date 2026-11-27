# REA -> Coolify Handoff Contract (Kanban)

Both profiles live on the **same** kanban board (`default`, DB at
`~/.hermes/kanban.db`). `rea` and `coolify` are both registered assignees.
The handoff is a task whose `assignee` changes from `rea` to `coolify`.

## How the handoff physically works

1. REA creates the deployment task with `kanban_create` (tool) or
   `hermes kanban create` (CLI), `--assignee coolify`.
2. The task body carries the **Deployment Brief** below.
3. The dispatcher (`hermes kanban daemon`) picks it up and spawns the
   `coolify` profile agent with that body as its context.
4. Coolify deploys, runs the domain gate, comments the results on the task,
   and completes it with `kanban_complete` (summary + metadata).
5. REA reads the completion, verifies independently, then creates the QA task.

REA must not deploy directly. Coolify must not re-implement REA work.

## Tool vs CLI

Inside a session (both profiles have the kanban toolset enabled via
`platform_toolsets.cli: ["hermes-cli", "kanban"]`):

| Tool                                                         | Purpose                                                                                                                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `kanban_create`                                              | New task. Required: `title`, `assignee`. Optional: `body`, `parents`, `priority`, `workspace_kind`, `workspace_path`, `project`, `tenant`, `idempotency_key` |
| `kanban_list`                                                | Discover work. Filters: `assignee`, `status`, `tenant`, `include_archived`, `limit` (50 default, 200 max)                                                    |
| `kanban_show`                                                | Full task with comments + events                                                                                                                             |
| `kanban_comment`                                             | Durable note. Required: `task_id`, `body`                                                                                                                    |
| `kanban_complete`                                            | Done + structured handoff. Needs `summary` or `result`; optionally `metadata`, `created_cards`, `artifacts`                                                  |
| `kanban_block`                                               | Mark blocked with a reason                                                                                                                                   |
| `kanban_unblock`                                             | Release a blocked task                                                                                                                                       |
| `kanban_link`                                                | Parent -> child dependency                                                                                                                                   |
| `kanban_heartbeat`                                           | Keep-alive on long work                                                                                                                                      |
| `kanban_request_review` / `kanban_request_changes`           | Review gates                                                                                                                                                 |
| `kanban_attach` / `kanban_attach_url` / `kanban_attachments` | Evidence and artifact links                                                                                                                                  |
| `kanban_create` with `parents`                               | Fan-out: child stays `todo` until every parent is `done`, then auto-promotes to `ready`                                                                      |

Outside a session (shell):

```bash
export HERMES_HOME=/home/hermeswebui/.hermes/profiles/rea
cd /home/hermeswebui/.hermes/hermes-agent
PY=/app/venv/bin/python

$PY -m hermes_cli.main kanban create "Deploy <app>" \
    --assignee coolify \
    --body-file /path/to/brief.md \
    --priority 10
$PY -m hermes_cli.main kanban list --assignee coolify
$PY -m hermes_cli.main kanban show <task_id>
$PY -m hermes_cli.main kanban comment <task_id> --body "..."
$PY -m hermes_cli.main kanban complete <task_id> --summary "..."
$PY -m hermes_cli.main kanban dispatch --dry-run
$PY -m hermes_cli.main kanban daemon --interval 60 --verbose
```

Use `--body-file` (not `--body`) for anything multi-line or containing
flag-like lines. Shell quoting silently corrupts inline bodies.

## The Deployment Brief

Fill this template into the task body. Every field is required unless marked
optional -- an incomplete brief produces a blocked task.

```markdown
## Deployment Brief

**Project**: <project-slug>
**Repository**: <repo url + branch + commit>
**Artifact to deploy**: <backend path | frontend path | compose file>

### Service topology

| Service | Type       | Image/Build              | Port | Public domain |
| ------- | ---------- | ------------------------ | ---- | ------------- |
| api     | dockerfile | ./reconstructed-backend  | 3000 | api.<domain>  |
| web     | dockerfile | ./reconstructed-frontend | 8080 | <domain>      |
| db      | postgresql | managed by Coolify       | 5432 | internal only |

### Required environment variables

| Name           | Secret? | Value source       | Notes                            |
| -------------- | ------- | ------------------ | -------------------------------- |
| DATABASE_URL   | yes     | Coolify-managed DB | injected from the provisioned DB |
| SESSION_SECRET | yes     | generate           | 32+ bytes                        |
| REA_PROJECT    | no      | literal            | reconstruction marker            |

Values marked secret are NEVER placed in this brief. Coolify provisions them
via `manage_env bulk_update` and the vault.

### Health check

- Endpoint: `GET /health`
- Expected status: `200`
- Expected body: `{"status":"ok"}`

### Domain gate (Coolify must verify all three)

1. `GET https://<domain>/` -> HTTP 200
2. `GET https://<domain>/health` -> HTTP 200, body `{"status":"ok"}`
3. TLS certificate valid and matching the hostname

## Login QA (deployed domain) -- MANDATORY

The app has authentication. A deployment that reports "healthy" is **not**
complete until the login flow has been driven in a real browser against the
deployed domain.

- Open the deployed login page and confirm it renders.
- **Zero inputs is the expected first state.** The email form is revealed by
  clicking the "使用 Email 登入" button. Do not report "no login form" as a
  defect before clicking it.
- Fill the email, submit, and capture the auth request: endpoint, HTTP
  status, response body.
  - `400 {"error":"not_configured"}` -> the deployed app is missing the
    runtime auth env (e.g. `AUTH_DEBUG_OTP`). Fix with `manage_env`,
    redeploy, retest. This is a **deployment defect**.
  - `400 {"error":"invalid args"}` -> the upstream original's own validation
    response (it sits behind reCAPTCHA Enterprise). This is **parity**, not a
    defect.
- Confirm the session: a session cookie is set and the browser lands on an
  authenticated page.
- Report the endpoint, status, body, and session outcome. Never report login
  as working without this browser evidence.

### Rollback

- Source: previous successful deployment for this application
- Expected recovery: redeploy previous image, re-run the domain gate

### Acceptance criteria (Coolify's task is complete when ALL are true)

- [ ] Deployment finished (no failures in the deployment log tail)
- [ ] Container status `running:healthy`
- [ ] Domain gate: all three checks above pass, with the actual status codes
      and response times reported
- [ ] Env variables set; no plaintext secrets in the brief, the task comments,
      or the logs
- [ ] **Login tested in a browser against the deployed domain**: the auth
      endpoint, HTTP status, response body, and proof that a session cookie was
      established are reported. "Deployment healthy" alone is not login
      evidence.
- [ ] Local residue cleaned (caches, scratch files, credential copies)
- [ ] Domain re-verified after cleanup
- [ ] `list_audit_events(source=mcp)` reported
- [ ] `metadata.login_qa` present in the completion summary: `{endpoint,
      status, body, session_established}`

### Out of scope for Coolify

- Reverse engineering and re-implementation of server logic (done by REA)
- Frontend fidelity work (done by REA)
- Fixing application-level bugs found during QA -- report them on the task,
  assign back to `rea`
```

## Handoff Rules

- **One deployment per task.** One app, one domain, one brief.
- **Never put a secret value in a brief or a comment.** Names and "yes/no is
  secret" only.
- **Coolify reports facts, not adjectives.** "HTTP 200 in 142ms", not "fast".
- **Coolify must not deploy an app whose brief lacks a health endpoint.** If
  the reconstructed backend has no `/health`, that is a REA gap -- send it back.
- **REA verifies independently after Coolify completes.** Never trust the
  completion summary without a fresh domain gate from the `rea` side.
- **Failures route back, not forward.** If Coolify finds an application bug,
  it comments the evidence and reassigns/creates a `rea` task. It does not
  patch application code.

## Dependency Graph

```
[rea] scope -> recon -> capture -> coverage -> contracts -> export
                                                   |
                                                   v
                                        [rea] backend + frontend
                                                   |
                                                   v
                                  [coolify] deploy (domain gate)
                                                   |
                                                   v
                                 [coolify] deployed-domain QA
                                                   |
                                                   v
                                       [rea] contract re-verify -> sign-off
```

Create the children with `parents` so each stays `todo` until its parent is
`done`, then auto-promotes to `ready`.

## Completion Contract

When REA creates the deployment task, attach a completion contract so
Coolify's completion is machine-checkable rather than prose. Use
`--completion-contract` on the CLI, or the equivalent field on
`kanban_create`.

The contract should require:

- `summary` mentioning the domain and the health status
- `metadata.deployment_uuid`
- `metadata.domain`
- `metadata.domain_gate` = `{root_status, health_status, health_body_ok, tls_valid}`
- `metadata.cleanup_performed` = true
- `metadata.audit_events_checked` = true

## Emitting Deployment Briefs (REA side)

REA creates the deployment brief that the `coolify` worker consumes. Every such
brief MUST carry exactly one authorisation token, so the worker never has to
infer permission:

| Token                     | When to use                                          |
| ------------------------- | ---------------------------------------------------- |
| `DEPLOY-AUTH: production` | The live domain must serve the clone.                |
| `DEPLOY-AUTH: staging`    | A staging host only; production must not be touched. |
| `DEPLOY-AUTH: dry-run`    | Plan and report only; **no live deployment**.        |
| `DEPLOY-AUTH: none`       | No deployment permitted at all.                      |

Brief format:

```
DEPLOY-AUTH: <production|staging|dry-run|none>
```

Rules when writing a brief:

1. **Always emit the token**, even when the answer is "no deployment".
2. **Never write a brief that says "deploy" without a token.** The worker will
   treat a token-less brief as `none` and will not deploy.
3. **Prose never overrides the token.** If the token says `dry-run` or `none`,
   the brief must not also contain "make it live", "test the deployment", or
   similar phrasing -- contradictory briefs are how an unauthorised deploy
   happens. State the token and stop.
4. **A smoke test gets `DEPLOY-AUTH: dry-run`** (or `none`) plus an explicit
   instruction to delete anything it creates.
5. **A production deploy is a two-brief handshake.** First brief carries
   `DEPLOY-AUTH: production` only; the coolify worker replies with a
   pre-deploy plan (target domain, source revision, exact operation) and
   closes the task as `blocked`. REA reviews that plan, then issues a second
   brief carrying **both** `DEPLOY-AUTH: production` **and**
   `DEPLOY-CONFIRMED: yes`. Only then does the worker deploy.

### Login QA: assert a session, not just an API response

A login is **not** proven by the OTP endpoint returning 200. Two defects
slipped through exactly that check and were only caught by driving the
browser end to end:

1. **The code step may never render.** The component can move its state to
   "enter the code" while the markup has no branch for it, so the flow is
   unreachable even though the backend is perfect.
2. **The token may never become a session.** An exchange can return a
   sign-in token that the client then drops, leaving the visitor a guest on
   a protected page.

Therefore the browser test must assert, in order:

- the login page renders and the sign-in form is revealed;
- the one-time-code request returns 200 **and** the code-entry input
  becomes visible;
- the code is submitted and the exchange returns 200;
- **a session exists** (check the app's real session store -- for this app
  a bearer token in `localStorage` under `msai.session.v1`, not a cookie);
- **the server honours it** (ask the server who the caller is; it must not
  answer as a guest);
- a protected page loads without bouncing to `/login`;
- a wrong code is rejected and creates **no** session.

Read the one-time code from the API response body, never by scraping digits
out of the page text. Measure client-rendered pages only after hydration.
Ignore third-party and aborted network calls when judging failures.

Never report a login as working without these steps; a 200 from the OTP
endpoint is not evidence.
