## Task: Scenario Capture -- <project>

**Phase**: 2 (Scenario Capture)
**Depends on**: the reconnaissance task id

### Scenario scope

- Journeys: <list of journeys>
- Collections: <list>
- Credential source: environment variable / vault (never in this task)

### Required scenario shape

- Sign in through the UI; declare the password in `secrets[]`
- Read each collection **twice** before any write
- Read each listed object by ID (up to ~10 per collection)
- Write through the UI; add a `request` step with `save_response` on create
- After each write: read object by ID, then read list
  (create 1 -> read 1 -> list -> create 2 -> read 2 -> list)
- `wait_for_timeout` 1100ms between creates and before the first edit
- Update both objects with varied fields; toggle actions twice; delete both and
  verify the deleted state
- Every step has `timeout_ms` and `expected_status`
- Target 60-160 requests total

### Deliverables

- Evidence bundle id in `evidence/<project>/manifest.json`
- `evidence/<project>/capture/<journey>.json` (the normalized result)

### Exit gate

Every step reports `status: "completed"`. Network bodies captured.
