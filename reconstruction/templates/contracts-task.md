## Task: Contract Extraction -- <project>

**Phase**: 4 (Contract Extraction)
**Depends on**: the coverage task id

### Views to run over the retained evidence

1. `inspect_analysis_view` -> `api-surface`
2. `inspect_analysis_view` -> `environment-manifest`
3. `inspect_analysis_view` -> `environment-state`

### Deliverables

- `contracts/<project>/api-surface.json`
- `contracts/<project>/environment-manifest.json`
- `contracts/<project>/environment-state.json`
- `contracts/<project>/schemas.ts` -- Zod schemas generated from the captured
  body shapes

### Exit gate

Every collection has a path template, an entity identity field, and a
request/response shape recorded.
