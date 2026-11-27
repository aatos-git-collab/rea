## Task: Deployed-Domain QA -- <project>

**Phase**: 9 (Deployed-Domain QA)
**Owner**: coolify (lead) + rea (API contract review)
**Target**: `https://<domain>` -- the deployed domain, NOT localhost.

### Gates to execute

- Gate 0: Infrastructure (9 checks) -- coolify
- Gate 1: Authentication (9 checks) -- coolify, contract cross-check by rea
- Gate 2: Core Journeys -- coolify, one row per recorded REA journey
- Gate 3: Visual Parity (4 checks) -- coolify captures, rea compares
- Gate 4: API Contract (10 checks) -- rea
- Gate 5: Data Integrity (7 checks) -- coolify + rea
- Gate 6: Non-Functional (8 checks) -- coolify
- Gate 7: Regression Against Original (4 checks) -- rea

The full matrix is in `/workspace/rea/reconstruction/qa-matrix.md`.

### Deliverables

- `qa/<project>/qa-report.md` -- every check id, result, and observed value
- `qa/<project>/screenshots/` -- before/after pairs
- `qa/<project>/diffs/` -- comparison outputs

### Exit gate

Every check is PASS, or FAIL with severity and an owner, or WAIVED with a
written sign-off. Verdict recorded: SHIP / SHIP WITH KNOWN ISSUES / DO NOT SHIP.

### Rule

Never hide a FAIL. Never weaken an acceptance criterion without a written
waiver.
