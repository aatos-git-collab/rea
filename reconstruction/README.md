# REA Reconstruction Pipeline

Agent-facing workflow for turning "reverse engineer xx.com" into a deployed,
QA-verified 1:1 clone with a working frontend **and** backend.

Two Hermes profiles cooperate over a shared Kanban board:

| Profile   | Owns                                                                           |
| --------- | ------------------------------------------------------------------------------ |
| `rea`     | Recon, analysis, evidence, frontend + backend implementation, contract review. |
| `coolify` | Coolify deployment, domain gate, deployed-domain browser QA.                   |

## Files

| File                          | Purpose                                                                                                          |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `pipeline.md`                 | The 10-phase pipeline with an **exit gate** per phase. Phase 8.1 is the mandatory deployment authorisation gate. |
| `handoff.md`                  | The REA -> Coolify Kanban handoff contract + how REA writes deployment briefs.                                   |
| `qa-matrix.md`                | Deployed-domain QA matrix (run against the live domain, never localhost).                                        |
| `implementation-checklist.md` | Dense implementation checklist for reconstructing a web app (frontend + backend) 1:1 from a live instance.       |
| `templates/recon-task.md`     | Phase 1 reconnaissance task template.                                                                            |
| `templates/deploy-task.md`    | Phase 8 deployment task template (REA -> Coolify).                                                               |
| `templates/qa-task.md`        | Phase 9 deployed-domain QA task template.                                                                        |

## The deployment authorisation gate (non-negotiable)

Deployment is a production side effect. The `coolify` worker must find an
explicit token in the task brief before it may create or mutate any live
resource:

```
DEPLOY-AUTH: production    # deploy to the live domain
DEPLOY-AUTH: staging       # staging host only
DEPLOY-AUTH: dry-run       # no live deployment -- plan and report only
DEPLOY-AUTH: none          # no deployment permitted at all
```

- No token -> no deployment. Absence of a token is never permission.
- Negative prose ("no real deployment", "just a smoke test", "dry run") is
  equivalent to `DEPLOY-AUTH: none`, even if the same brief also says
  "deploy" or "make it live".
- `DEPLOY-AUTH: production` also needs `DEPLOY-CONFIRMED: yes` in the same
  brief before the worker executes.
- A worker that cannot satisfy the gate closes the task as **blocked**.
  Blocking is a success; an unauthorised production deploy is a failure.

## Operating rules

- `rea` never deploys. `coolify` never rewrites the reconstruction.
- Every phase has an exit gate; do not advance without it.
- **Serving an AgentEnv bundle on this host:** there is no Docker, and rootless
  Docker is impossible (AppArmor blocks unprivileged user namespaces). Serve
  natively with `rea environment-serve <bundle>/envs/<name>/manifest.json
  --port <p>` and speak MCP Streamable HTTP to `POST /mcp`. See
  `implementation-checklist.md` section E for the full handshake.
- **HAR body pitfall:** REA reads a HAR response body only when `encoding` is
  absent or `"base64"`. An explicit `"encoding": "utf-8"` yields a `null` body
  (`body-unusable`).
- QA runs against the **deployed domain** -- localhost does not prove DNS, TLS,
  proxy, or certificate behaviour.
- Every task that creates live resources records their identifiers
  (application uuid, project uuid, domain) in a Kanban comment and cleans up
  unless the brief asked to keep them.
