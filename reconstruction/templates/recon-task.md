## Task: Reconnaissance -- <project>

**Phase**: 1 (Reconnaissance)
**Mode**: PASSIVE ONLY. No writes to the target.

### Target

- URL / target id: <url or target_id>
- Kind: web | electron | bundle | har
- Authorization confirmed: yes (see scope.md)

### Steps

1. `list_browser_targets` (or `list_electron_targets`) -> select target_id
2. `inspect_web_page` -> DOM, AX tree, network requests, scripts, storage
3. `capture_web_screenshot` -> visual baseline
4. `analyze_web_bundle` -> routes, endpoint candidates, vendor fingerprints
5. `analyze_javascript_application` on exported scripts if the frontend is a
   shipped bundle

### Deliverables

- `evidence/<project>/recon/` with the inspection output and screenshot
- `evidence/<project>/manifest.json` updated with the evidence ids
- Route inventory and endpoint candidate list written to
  `evidence/<project>/routes.md`

### Exit gate

Route inventory and endpoint candidate list exist and are non-empty.
