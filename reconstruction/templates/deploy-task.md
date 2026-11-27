# Kanban task template -- Deployment (coolify)

> Copy this whole block into `kanban_create` (assignee: `coolify`).
> The `DEPLOY-AUTH:` token is **mandatory** and must be the first line of the
> brief. A brief without it is treated as `DEPLOY-AUTH: none`.

```
DEPLOY-AUTH: <production|staging|dry-run|none>

# Task: Deploy <project-slug> to <target-host/domain>

## Handoff from rea
- Project slug:      <slug>
- Source repo:       <path>
- Source revision:   <git sha / commit>
- Built artifact:    <image tag | tarball path | build command + output path>
- Target host:       <server uuid>
- Target domain:     <fqdn>
- Coolify project:   <project uuid, or "create new">
- Coolify app:       <application uuid, or "create new">
- Env vars / secrets:<names + where they come from (never inline values)>
- Health endpoint:   <path that must return ok>
- Reverse-engineering scope: see attached evidence bundle <evidence_id>

## Prior verification already done by rea
- [ ] Frontend built and served locally
- [ ] Backend endpoints implemented and exercised locally
- [ ] API contract diff vs. original: <result>
- [ ] Secrets/endpoints extracted: <result>
- [ ] Known gaps accepted by rea: <list>

## What coolify must do
1. Read `reconstruction/pipeline.md` Phase 8 and the deployment authorisation gate.
2. Provision / configure the application per the handoff above.
3. Deploy and run the domain gate:
   - HTTP 200 on <target domain>/
   - <health endpoint> returns the expected body
   - TLS certificate valid for <target domain>
4. Post the deployment identifiers (application uuid, project uuid, domain) in a
   kanban comment.
5. Delete every resource you created unless the brief asked to keep them.

## Exit criteria
- [ ] Domain gate passes on the deployed domain (not localhost)
- [ ] Deployment identifiers posted as a kanban comment
- [ ] Task closed with a status comment

## Notes for the worker
- If `DEPLOY-AUTH` is `dry-run` or `none`: plan and report only. Do NOT call
  `mcp__coolify_server__deploy`, `manage_application`, `manage_dns`, or
  `manage_project` in a way that creates or mutates live resources. If the
  gate cannot be satisfied, close the task as `blocked` naming the missing
  token -- blocking is a success.
- If `DEPLOY-AUTH` is `production`: post a pre-deploy comment naming the
  target domain, the source revision, and the exact Coolify operation, and
  wait for `DEPLOY-CONFIRMED: yes` on this task before executing.
```
