# TraceDB Protocol

Private staging repository for the TraceDB protocol contract.

This repository is the source of truth for `platform-contract-v0`, the HTTP
`/v1` reference docs and OpenAPI artifact, the scenario manifest, and
executable conformance behavior used by the TraceDB core and SDK repositories
during the private organization split.

`platform-contract-v0` is the cross-surface contract version for SDKs, TraceQL,
GraphQL, and direct HTTP conformance. It does not rename the HTTP API. The HTTP
routes remain versioned under `/v1`, with `docs/api/v1-http.md` and
`docs/api/v1-openapi.json` as the route-level reference.

Current contract sources:

- `docs/api/v1-http.md`
- `docs/api/v1-openapi.json`
- `docs/platform-contract-v0.md`
- `docs/platform-contract-v0.json`
- `scripts/platform_conformance.py`

Downstream repositories pin this repo with `tracedb-protocol.lock`. During the
split, core and SDK repositories may carry local mirrors or generated artifacts
for their own validation, but contract edits should originate here first and
then be copied/regenerated outward.

Local checks:

```bash
python3 -m json.tool docs/platform-contract-v0.json >/dev/null
python3 scripts/platform_conformance.py --help
python3 scripts/validate_protocol_locks.py --repo-root ..
```
