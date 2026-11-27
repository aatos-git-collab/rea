DEPLOY-AUTH: staging

# Deployment Brief — mysports.ai clone (`coolify-dev` branch)

You are the `coolify` profile agent: Chief Deployment Architect & Infrastructure
Engineer. This is a **staging** deployment. Own every step; report facts with
status codes and timings.

## Context

`rea` reconstructed `https://mysports.ai` as a Next.js 14 App Router clone in a
private repo. `rea` owns the application code. **You own deployment, domain
wiring, env wiring, deployment debugging, and deployed-domain browser QA.**
If you find an application bug, do NOT patch application code — comment the
evidence on this task and report it back to `rea`.

## Source revision

- **Repository**: `https://github.com/aatos-git-collab/mysports.ai.git` (private)
- **Branch**: `coolify-dev`
- **Commit**: `0319c128c9050eb96d51e8eaadc2f31a684aadfd`
- **GitHub App**: `ot7vc8yo7j8gafurnc2udjlw` (owner `aatos-git-collab`) —
  already installed on the team and already able to read this repo and its
  branches. Use the `github_app` source type. Do not request a deploy key.
- **Build**: Dockerfile at the repository root (`/Dockerfile`).

## Branch discipline (important)

- `coolify-dev` is the **deployment branch**. Commit every deployment fix you
  make here, and push it to `origin/coolify-dev`.
- **Never commit to `master`.** `rea` is landing application work on `master`
  concurrently, in parallel with you. Do not merge `master` into
  `coolify-dev` yourself — `rea` does that, and it will tell you when.
- If you need a newer `master` commit in your build, comment on this task
  asking `rea` to merge `master` into `coolify-dev`, then redeploy.

## Artifact to deploy

- **Type**: Dockerfile (single application), plus a managed PostgreSQL database.
- **Path**: `/Dockerfile` (repository root).
- **Exposed port**: `3000`.
- **Entrypoint**: `docker/entrypoint.sh` runs `scripts/bootstrap.ts` first when
  `MIGRATE_ON_START=1`, then `node server.js`. The bootstrap is idempotent.
  The first boot imports the engine odds archive (`data/engine/*.sql.gz`,
  ~9.5 MB compressed / ~118 MB SQL) and takes several minutes; the Dockerfile
  already sets `HEALTHCHECK --start-period=900s` for this. **Do not treat a
  long first boot as a failure** — watch the deployment log and wait.

## Service topology

| Service | Type       | Image/Build     | Port | Public domain            |
| ------- | ---------- | --------------- | ---- | ------------------------ |
| db      | postgresql | Coolify-managed | 5432 | internal only            |
| app     | dockerfile | `/Dockerfile`   | 3000 | `mysports.go.nexeraa.io` |

Suggested domain: `mysports.go.nexeraa.io` — the team already serves
`browse.go.nexeraa.io` through the same Cloudflare-managed zone, and the
existing `browser-qa` application is the pattern to copy (it lives in project
`Browser`, environment `production`, on the `paperclip` server). **Resolve the
project, environment, server and destination UUIDs live** — never hardcode
UUIDs from this brief.

## Required environment variables

| Name                             | Secret? | Value source        | Notes                                                                                                                                                                                         |
| -------------------------------- | ------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DATABASE_URL`                   | yes     | Coolify-managed DB  | Inject the provisioned database's connection URL.                                                                                                                                             |
| `DATABASE_POOL_MAX`              | no      | literal `10`        | Optional; the default is fine.                                                                                                                                                                |
| `MIGRATE_ON_START`               | no      | literal `1`         | Required for the first boot to migrate + seed.                                                                                                                                                |
| `BETTER_AUTH_SECRET`             | yes     | generate            | **REQUIRED.** >= 32 chars high entropy. The app has **no fallback** — it deliberately never reuses `DATABASE_URL` as the cookie-signing secret. Generate with `openssl rand -base64 32`.      |
| `BETTER_AUTH_URL`                | yes     | public origin       | `https://mysports.go.nexeraa.io` — the public origin the app is served from. Better Auth uses it as its base URL and trusted origin, and it decides whether the session cookie gets `Secure`. |
| `BETTER_AUTH_TRUSTED_ORIGINS`    | no      | comma separated     | Only needed if the app is reachable under more than one public URL.                                                                                                                           |
| `BETTER_AUTH_USE_SECURE_COOKIES` | no      | literal `true`      | Leave `true` for an HTTPS deployment.                                                                                                                                                         |
| `API_ORIGIN`                     | no      | leave **empty**     | Empty keeps the catch-all `/api/*` proxy from forwarding to the original site. The clone serves its own API from PostgreSQL.                                                                  |
| `PORT`                           | no      | literal `3000`      | Matches `EXPOSE 3000`.                                                                                                                                                                        |
| `NEXT_PUBLIC_API_ORIGIN`         | no      | build arg, optional | Inlined at build time; empty disables the feature that needs it.                                                                                                                              |

**Never place a secret value in a brief, a comment, or a log.** Set secrets via
`manage_env` only, and report outcomes with `list_env_keys` (names only).
Avoid `get_env_values` — it returns plaintext secrets.

## Health check

- **Endpoint**: `GET /api/health`
- **Expected status**: `200`
- **Expected body**: `{"status":"ok", ...}` — `status` must be `"ok"`.
  The body also reports `database.configured` and `database.reachable`;
  `database.reachable` should be `true` once the bootstrap has finished.
- The Dockerfile's own `HEALTHCHECK` hits the same endpoint on
  `http://127.0.0.1:3000/api/health`.

## Domain gate (verify all three, report the actual numbers)

1. `GET https://mysports.go.nexeraa.io/` -> HTTP 200 (report the response time)
2. `GET https://mysports.go.nexeraa.io/api/health` -> HTTP 200, body `status` = `"ok"`
3. TLS certificate valid and matching the hostname

Also verify these application routes return HTTP 200 through the public domain,
because they are the read endpoints the clone serves from its own database:

- `/app/msai/`
- `/api/msai/predictions/`
- `/api/msai/feed/`
- `/api/odds/feed/`
- `/api/tw-singles/`
- `/api/products/`
- `/api/odds/radar/?limit=3` (this one reads `engine_game_odds`; it is
  expected to take ~1 second and return ~2,250 events)

## Browser QA (deployed domain)

After the domain gate passes, run a real browser pass against the **deployed**
URL and report findings:

- Load `/` and confirm the locale redirect works.
- Load `/tw/`, `/hk/`, `/en/`, `/ja/`, `/es/` and confirm HTTP 200 and a
  correct `<html lang>` (`zh-Hant` / `zh-HK` / `en` / `ja` / `es`).
- Load `/app/msai/` and confirm the page renders the MSAI content.
- Confirm no Chinese strings leak into the **body** of the `/en/`, `/ja/` and
  `/es/` pages. (`<meta>` tags carry a Traditional Chinese `og:title` on the
  original too — that is faithful 1:1 behaviour, not a bug. Only body leaks
  are defects.)
- Report any console errors, failed network requests, or broken layout.

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

## Rollback

- Source: the previous successful deployment for this application.
- Expected recovery: redeploy the previous image and re-run the domain gate.
- This is a **new** application, so there may be no previous image. If the
  first deploy fails irrecoverably, report it rather than improvising.

## Acceptance criteria (the task is complete when ALL are true)

- [ ] Deployment finished with no failures in the deployment log tail
- [ ] Container status `running:healthy`
- [ ] Domain gate: all three checks pass, with the actual status codes and
      response times reported
- [ ] The application routes listed above all return HTTP 200 through the
      public domain
- [ ] Browser QA pass completed against the deployed URL, with findings
      reported
- [ ] **Login flow driven in a browser against the deployed domain**, with the
      auth endpoint, HTTP status, response body, and session outcome reported
      (see Login QA above)
- [ ] Env variables set; **no plaintext secrets** in this brief, in the task
      comments, or in the logs
- [ ] Local residue cleaned (caches, scratch files, credential copies)
- [ ] Domain re-verified after cleanup
- [ ] `list_audit_events(source=mcp)` reported
- [ ] Any deployment fix you made is committed to `coolify-dev` and pushed to
      `origin/coolify-dev` (never to `master`)

## Completion contract

Complete the task with `kanban_complete` and include this metadata:

- `metadata.deployment_uuid` — the Coolify deployment UUID
- `metadata.application_uuid` — the Coolify application UUID
- `metadata.domain` — the public domain that was gated
- `metadata.domain_gate` — `{root_status, health_status, health_body_ok, tls_valid}`
- `metadata.cleanup_performed` — `true`
- `metadata.audit_events_checked` — `true`
- `metadata.branch` — `coolify-dev`
- `metadata.login_qa` — `{endpoint, status, body, session_established}` from the
  browser-driven login test
- `metadata.commit` — the commit that was actually deployed

The `summary` must mention the domain and the health status.

## Out of scope for you

- Reverse engineering and re-implementing server logic (done by `rea`)
- Frontend fidelity work (done by `rea`)
- Fixing application-level bugs found during QA — report them on this task
- Merging `master` into `coolify-dev`, or merging `coolify-dev` into `master`
  (`rea` owns both merges)

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
