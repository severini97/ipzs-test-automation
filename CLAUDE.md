# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

IPZS Test Automation is an end-to-end test suite for the OpenID4VCI (OpenID for Verifiable Credential Issuance) protocol, built as a Node-RED project for IPZS (Istituto Poligrafico e Zecca dello Stato). It verifies the full credential issuance flow (PAR → Auth → Nonce → Token → Credential → Notification) including SD-JWT-VC structural conformance (CI_120–CI_130).

## Tech Stack

- **Node-RED** (flow-based runtime, port 1880)
- **JavaScript** (Function nodes inside flows.json)
- **jose** v6.1.3 — JWT/JWK operations (`global.get("jose")`)
- **crypto** — Node.js built-in (`global.get("crypto")`)
- **ciCommon** — custom test utilities at `~/.node-red/lib/ciCommon.js` (`global.get("ciCommon")`)

## Key Directories

```
~/.node-red/
├── settings.js              # Runtime config, globalContext, project mode
├── lib/ciCommon.js          # Shared assertion utilities for all Function nodes
└── projects/ipzs-test-automation/
    ├── flows.json           # All flows (~10k lines) — the source of truth
    ├── flows_cred.json      # Encrypted credentials (do not edit)
    └── .github/workflows/   # Claude Code Action + auto PR review
```

## Running Tests

Start Node-RED:
```
node-red
```

Trigger a CI test via HTTP (editor at http://localhost:1880):
```
GET http://localhost:1880/test/CI_050
GET http://localhost:1880/test/CI_120
GET http://localhost:1880/test/CI_050?verbose=true   # full assertion report
```

Deploy flow changes: **Ctrl+Shift+D** in the editor, then commit via the Git panel.

See `architecture-notes.md` for the full flow architecture and instructions for adding new CI tests.
