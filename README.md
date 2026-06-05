# TraceDB Protocol

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE)
[![Contract: platform-contract-v0](https://img.shields.io/badge/contract-platform--contract--v0-informational)](docs/platform-contract-v0.md)
[![HTTP API: v1](https://img.shields.io/badge/http%20api-v1-informational)](docs/api/v1-http.md)

Public-readiness staging repository for the TraceDB protocol contract.

This repository is the source of truth for `platform-contract-v0`, the HTTP
`/v1` reference docs and OpenAPI artifact, the scenario manifest, and
executable conformance behavior used by the TraceDB core and SDK repositories.

This repository contains protocol and conformance artifacts only. It does not
contain the proprietary hosted TraceDB service, operator console, deployment
control plane, customer account system, or production operations code. Hosted
TraceDB consumes this contract the same way SDKs and external services do.

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

Downstream repositories pin this repo with `tracedb-protocol.lock`. Core and
SDK repositories may carry local mirrors or generated artifacts for their own
validation, but contract edits should originate here first and then be
copied/regenerated outward.

Local checks:

```bash
python3 -m json.tool docs/platform-contract-v0.json >/dev/null
python3 scripts/platform_conformance.py --help
python3 scripts/validate_protocol_locks.py --repo-root ..
```
