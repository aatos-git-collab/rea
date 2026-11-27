DEPLOY-AUTH: staging

# Task: Deploy mysports.ai clone to staging — 4 new commits incl. the stale-data fix

## Handoff from rea

- Project slug: mysports.ai
- Source repo: aatos-git-collab/mysports.ai
- Implementation branch: `master` (HEAD a3cc0a1)
- Deployment branch: `coolify-dev` — **already fast-forwarded to a3cc0a1** and pushed
- Coolify app: `eresncvxi2sfzm9nwczrxkxb` (`mysports-staging`)
- Target host: `paperclip` (172.104.88.222)
- Target domain: https://mysports.team.nexeraa.io (gate must land on this, NOT `go.nexeraa.io`)
- Coolify project: `coygvxmhnyochejvjhikbygf` / environment `kxudt2eec2mikggzf4vnngsy`
- Env vars: none added. Existing set is intact (verified key names only, no plaintext
  retrieved): DATABASE_URL, DATABASE_POOL_MAX, MIGRATE_ON_START,
  BETTER_AUTH_SECRET, BETTER_AUTH_URL, BETTER_AUTH_TRUSTED_ORIGINS,
  BETTER_AUTH_USE_SECURE_COOKIES, API_ORIGIN, PORT.
- Health endpoint: /api/health -> 200 `{"status":"ok","database":{"configured":true,"reachable":true}}`

## Why this deploy is needed

The live site runs `eaf41a7` (deployment `wofndaymuojt41n1nkcqt8gi`, finished
2026-11-22T02:58:35Z). Four commits have landed since and NONE are deployed:

1. `7826aab` — 3-tier role model (super admin / admin / user), DB-backed sign-in
   settings, i18n catalogues
2. `30134ec` — **live fixtures instead of a stale snapshot** — this is the fix for the
   2-day-stale odds/scoreboard the live site is serving right now
3. `6a2bd22` — peak-championship calendar
4. `a3cc0a1` — plan-gate cleanup (dead code removed)

## Measured evidence of the defect to be fixed

```
GET /api/odds/feed     -> {"updated_at":"2026-11-20T12:50:41.857Z", ...}
GET /api/scoreboard    -> {"date":null,"league":"mlb","games":[{"start":"2026-07-04T15:05:00Z",...}]}
```

Today is 2026-11-22, so the odds feed is two days stale. After this deploy the
`updated_at` must advance past `2026-11-20` (ideally 2026-11-22) and the scoreboard
must carry current fixtures, not the baked snapshot.

## What coolify must do

1. Trigger a deployment of `eresncvxi2sfzm9nwczrxkxb` (branch `coolify-dev`).
   A fast-forward to a3cc0a1 is already on origin; no git work is required from you.
2. Do NOT change domains, DNS, env vars, or app config. The domain
   `https://mysports.team.nexeraa.io` is already correct and converged — the previous
   flip-flop was caused by repeated deploys, and it is now fixed. One clean deploy only.
3. Run the domain gate:
   - `GET https://mysports.team.nexeraa.io/` -> 301 -> `/tw/`
   - `GET https://mysports.team.nexeraa.io/tw/` -> 200
   - `GET https://mysports.team.nexeraa.io/api/health` -> 200 with `database.reachable: true`
   - TLS valid for `mysports.team.nexeraa.io`
4. Assert the data fix (the actual point of this deploy):
   - `GET /api/odds/feed` -> `updated_at` must be 2026-11-21 or LATER
   - `GET /api/scoreboard` -> must not be limited to the old snapshot dates
5. Assert `/api/config/auth` -> `{"otp":true,"password":true,"google":false}`
   (google stays off unless the super admin enabled it in app settings — no code
   change or redeploy is needed to turn it on, so do not deploy again for that.)
6. Post the deployment uuid + measured status codes/response times as a kanban comment.

## Exit criteria

- [ ] Domain gate passes on `mysports.team.nexeraa.io` (not localhost)
- [ ] Odds/scoreboard data confirmed current (not 2026-11-20)
- [ ] Deployment uuid + measured evidence posted as a comment
- [ ] Task closed with a status comment

## Notes for the worker

- `DEPLOY-AUTH` is `staging`: deploy to the staging app above. Do not create or mutate
  any other app, database, service, or DNS record. Do not delete anything.
- If a gate cannot be satisfied, close the task as `blocked` naming the failing check
  — blocking with evidence is a success.
- Never print secret values. `get_env_values` returns plaintext; use `list_env_keys`.
- Do NOT merge or touch `master`; rea owns that branch.
- History note: this app previously flip-flopped between 200 and 503 because two
  agents issued competing deploys. Issue exactly ONE deploy and let it converge.
