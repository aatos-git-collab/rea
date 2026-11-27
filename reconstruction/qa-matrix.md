# Deployed-Domain QA Matrix

QA runs against the **live deployed domain**, not localhost. Localhost does not
exercise DNS, TLS termination, the reverse proxy, certificate renewal, or
header security policy.

The `coolify` profile leads QA. The `rea` profile owns the API-contract
comparison, because that is the part derived from reverse-engineering evidence.

## Gate 0 -- Infrastructure (Coolify)

| #   | Check               | Method                          | Pass criteria                                                                                               |
| --- | ------------------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 0.1 | Root reachable      | `GET https://<domain>/`         | HTTP 200, expected body, response time recorded                                                             |
| 0.2 | Health endpoint     | `GET https://<domain>/health`   | HTTP 200, body `{"status":"ok"}`                                                                            |
| 0.3 | TLS valid           | certificate check               | Valid chain, hostname matches, not expired, not self-signed                                                 |
| 0.4 | HTTPS redirect      | `GET http://<domain>/`          | Redirects (301/308) to HTTPS                                                                                |
| 0.5 | Container health    | Coolify app/service status      | `running:healthy`, not merely `running`                                                                     |
| 0.6 | Deployment finished | Coolify deployment history      | Last deployment `finished`, no failure in the log tail                                                      |
| 0.7 | Security headers    | response headers                | `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options` (or a documented decision to omit) |
| 0.8 | No secret leakage   | response headers + body         | No API keys, tokens, or internal hostnames in public responses                                              |
| 0.9 | Audit trail         | `list_audit_events(source=mcp)` | Coolify's writes are visible in the audit log                                                               |

## Gate 1 -- Authentication (Coolify + REA contract)

| #   | Check                         | Pass criteria                                                                                 |
| --- | ----------------------------- | --------------------------------------------------------------------------------------------- |
| 1.1 | Login works                   | Valid credentials -> authenticated session, redirect to the post-login route                  |
| 1.2 | Invalid credentials rejected  | HTTP 401/403 and a visible error, no session issued                                           |
| 1.3 | Empty credentials rejected    | Client-side and server-side validation both fire                                              |
| 1.4 | Session persists              | Auth cookie/token survives a page reload and a new tab                                        |
| 1.5 | Session expires               | Expired session redirects to login, protected routes return 401                               |
| 1.6 | Logout works                  | Session invalidated server-side; protected routes unreachable after logout                    |
| 1.7 | Protected route direct access | Unauthenticated request to a protected page -> redirect to login or 401                       |
| 1.8 | Token handling                | Token not in URL query string, not in `localStorage` if the original avoided it, never logged |
| 1.9 | Rate limiting                 | Rapid failed logins are throttled (if the original had it)                                    |

## Gate 2 -- Core Journeys (Coolify)

One row per journey recorded in the REA scenario. Every journey must be
replayed against the deployed domain.

| #   | Journey        | Steps                            | Pass criteria                                        |
| --- | -------------- | -------------------------------- | ---------------------------------------------------- |
| 2.x | <journey name> | Reproduce the REA scenario steps | Same final state, same side effects, same visible UI |

Per journey, additionally verify:

- The list view shows the same columns, order, and pagination behaviour
- The detail view renders the same fields
- The create form validates the same way (same required fields, same errors)
- The update form round-trips the same fields
- Delete removes the item and the list reflects it
- Empty states, loading states, and error states match the original

## Gate 3 -- Visual Parity (Coolify + REA)

| #   | Check                     | Tool                      | Pass criteria                                                             |
| --- | ------------------------- | ------------------------- | ------------------------------------------------------------------------- |
| 3.1 | Screenshot diff per route | `compare_web_screenshots` | Changed-pixel ratio below the agreed threshold, or every delta explained  |
| 3.2 | DOM/AX parity             | `compare_web_captures`    | Same DOM structure, same AX tree, same URL transitions, same storage keys |
| 3.3 | Network parity            | `compare_web_captures`    | Same ordered request sequence, same methods and paths                     |
| 3.4 | WebSocket parity (if any) | `compare_web_captures`    | Same message shapes and ordering                                          |

The `before` side is the original capture evidence; the `after` side is a fresh
capture of the deployed domain. Scenario captures align by exact `step_id`.

## Gate 4 -- API Contract (REA)

| #    | Check                  | Method                         | Pass criteria                                                    |
| ---- | ---------------------- | ------------------------------ | ---------------------------------------------------------------- |
| 4.1  | Path templates         | Live requests vs `api-surface` | Every documented endpoint exists at the documented template      |
| 4.2  | Methods                | Same                           | GET/POST/PATCH/DELETE match                                      |
| 4.3  | Request shapes         | Same                           | Field names, types, and nesting match the captured shape         |
| 4.4  | Response shapes        | Same                           | Field names, types, and nesting match the captured shape         |
| 4.5  | Status codes           | Same                           | Same success/error codes as the original                         |
| 4.6  | Identity fields        | Same                           | Entity identity field is present and stable                      |
| 4.7  | Query parameters       | Same                           | Same parameter **names** (values may differ)                     |
| 4.8  | Error envelope         | Same                           | Same error shape and message field                               |
| 4.9  | Unknown routes         | 404 handling                   | Same 404 shape as the original                                   |
| 4.10 | Unsupported operations | Withheld writes                | Each withheld write behaves as documented in the coverage report |

## Gate 5 -- Data Integrity (REA + Coolify)

| #   | Check                 | Pass criteria                                                            |
| --- | --------------------- | ------------------------------------------------------------------------ |
| 5.1 | Create persists       | A created entity survives a server restart                               |
| 5.2 | Update persists       | An updated field keeps its new value after reload                        |
| 5.3 | Delete persists       | A deleted entity does not reappear                                       |
| 5.4 | Referential integrity | Deleting a parent either cascades or blocks, matching the original       |
| 5.5 | Isolation             | One user's data is not visible to another user                           |
| 5.6 | Uniqueness            | Duplicate creation is rejected the same way the original rejects it      |
| 5.7 | Seed/reset            | `agent-env` reset semantics are mirrored by the real backend's seed path |

## Gate 6 -- Non-Functional (Coolify)

| #   | Check                | Pass criteria                                            |
| --- | -------------------- | -------------------------------------------------------- |
| 6.1 | Response time        | p95 under the agreed budget for the main routes          |
| 6.2 | Concurrent load      | Stable under the agreed concurrent-user count            |
| 6.3 | Cold start           | First request after deploy completes within budget       |
| 6.4 | Static assets        | Assets served with cache headers; no 404s in the console |
| 6.5 | Favicon / manifest   | Present, no 404 in the browser console                   |
| 6.6 | Browser console      | No uncaught errors on the main journeys                  |
| 6.7 | Responsive layout    | Matches the original at the agreed breakpoints           |
| 6.8 | Accessibility basics | Keyboard navigation works on the main journeys           |

## Gate 7 -- Regression Against the Original (REA)

| #   | Check                                 | Method                         | Pass criteria                                            |
| --- | ------------------------------------- | ------------------------------ | -------------------------------------------------------- |
| 7.1 | Bundle parity                         | `reconcile_javascript_runtime` | No unexplained module divergence                         |
| 7.2 | Route parity                          | `analyze_web_bundle` on both   | Same route inventory                                     |
| 7.3 | Endpoint parity                       | `analyze_web_bundle` on both   | Same endpoint candidates                                 |
| 7.4 | Version delta (if two versions exist) | `compare_application_versions` | Structural fingerprint matches within the expected delta |

## Reporting

Produce one report per project at `qa/qa-report.md`:

1. Environment: domain, deployment uuid, build commit, date
2. A table with every check id, result (PASS/FAIL/WAIVED), and the actual
   observed value (status code, response time, pixel ratio)
3. For every FAIL: the symptom, the evidence, the responsible layer
   (deployment / config / backend / frontend / original-interpretation), and
   the proposed fix
4. For every WAIVED: who signed it off and why
5. The final verdict: SHIP / SHIP WITH KNOWN ISSUES / DO NOT SHIP

Never hide a FAIL. Never weaken an acceptance criterion without a written
waiver.

## Severity

| Severity | Definition                                                   | Action                                  |
| -------- | ------------------------------------------------------------ | --------------------------------------- |
| P0       | Domain down, data loss, auth bypass                          | Block the release, roll back, fix first |
| P1       | Core journey broken, contract mismatch on a primary endpoint | Fix before sign-off                     |
| P2       | Secondary journey broken, visual delta above threshold       | Fix, or waive with owner sign-off       |
| P3       | Cosmetic delta, console warning, missing favicon             | Fix opportunistically, record it        |
