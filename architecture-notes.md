# Architecture Notes — ipzs-test-automation

## What This Project Does

End-to-end **OpenID4VCI** (OpenID for Verifiable Credential Issuance) protocol test automation for IPZS (Istituto Poligrafico e Zecca dello Stato). It tests the full credential issuance lifecycle including SD-JWT-VC (Selective Disclosure JWT Verifiable Credentials) structural conformance.

---

## Runtime Environment

- **Node-RED** at `http://localhost:1880`, port configurable via `PORT` env var
- **`settings.js`** is the only real configuration file — it defines globals available to all Function nodes via `functionGlobalContext`:
  - `global.get("ciCommon")` — `lib/ciCommon.js` (custom test utilities)
  - `global.get("crypto")` — Node.js built-in crypto
  - `global.get("jose")` — JOSE v6.1.3 (JWT/JWK, async)
- `functionExternalModules: true` — Function nodes may also `require()` npm packages directly
- `flowFilePretty: true` — `flows.json` is 4-space indented; version-controlled
- Projects mode: **manual commit** (changes must be deployed then committed via Git panel)

---

## Source File Layout

```
~/.node-red/
├── settings.js                      # Runtime config (port, globals, project mode)
├── lib/ciCommon.js                  # Shared test utility — loaded into all Function nodes
└── projects/ipzs-test-automation/
    ├── flows.json                   # ~10k lines — all flows, subflows, nodes
    ├── flows_cred.json              # Encrypted credentials (do not edit)
    ├── package.json                 # Project metadata
    └── .github/workflows/
        ├── claude.yml               # Claude Code Action (triggered by @claude mentions)
        └── claude-code-review.yml   # Auto PR review via claude-code-action@v1
```

---

## Flow Tabs (inside flows.json)

| Tab label | Tab ID | Role |
|-----------|--------|------|
| Issuance flow | `f21af705a29aab2f` | Interactive protocol walkthrough (not CI) |
| **Assert Flow CI** | `beb74de4721fafb3` | All assertion Function nodes + HTTP responses |
| **Generation Flow CI** | `1111a98978c668d9` | HTTP trigger → Set nodes → link-out routing |
| Multi test flow | `4c1844392e92d60a` | Orchestrates multiple test IDs in sequence |
| Well know mocked server | `fd0b9d33192f4cec` | Mock HTTP server for `.well-known` metadata endpoints |
| GUI | `10dbb78a9d60e878` | Browser dashboard |

---

## Subflows (reusable protocol steps)

Each subflow implements one protocol step. They are used in the Issuance flow tab and the mock server:

| Subflow | Purpose |
|---------|---------|
| `PAR Request` | Pushed Authorization Request (OAuth 2.0 PAR) |
| `Nonce Request` | Nonce endpoint call |
| `Token Request` | Token endpoint call |
| `Credential Request` | Credential endpoint call |
| `Notification Request` | Notification endpoint call |
| `Credential Issuer` | Mock issuer HTTP handlers (`/par`, `/auth`, `/token`, `/credential`, etc.) |
| `HTTP Request + Mock Fallback` | HTTP node that falls back to mock data when real endpoints fail |
| `Decode JWT not signed` | Base64url-decodes a JWT payload without signature verification |

---

## CI Test Architecture (the core pattern)

All CI tests follow the same two-tab pipeline:

```
[Generation Flow CI]                        [Assert Flow CI]
HTTP In (/test/CIXX)
  → function: initTest, set testId
  → subflow: protocol step(s)
  → Post-<step> switch                       link-in
       ├─ eq "CI_0XX" → Set CI_0XX node  →─→ assert switch (checkall)
       ├─ eq "CI_0YY" → Set CI_0YY node  →─→  ├─ contains "CI_0XX" → Assert CI_0XX
       └─ else → (pass-through)           →─→  └─ else → link-out (next assert group)
                                                     ↓
                                              Change: msg.payload.test = msg.test
                                              HTTP Response (application/json)
```

### Key routing nodes

- **Post-step switches** in Generation Flow CI route `msg.testId` to the correct Set node. Multiple switch nodes exist, one per protocol step (PAR, Auth, Nonce, Token, Credential, Notification). The switch for Post Credential has outputs for CI_120–CI_130 (SD-JWT tests).
- **Set nodes** inject `msg.subTestCase` and `msg.innerSplit = { index, _id, total }` before sending to the link-out.
- **link-out → link-in pairs** cross tab boundaries (Generation → Assert).
- **Assert switches** use `checkall: "true"` so a single message can trigger multiple assert nodes if the testId contains multiple substrings.
- **`else` outputs** chain to the next assert group via link-out.

### `innerSplit` pattern

Tests with multiple variants (e.g., one OK case and one KO case) set `total: 2` and split into two messages. A buffering function in the Assert tab collects all messages with the same `_id` before sending the combined response.

Single-variant tests (CI_120–CI_130) set `total: 1` — no buffering needed.

```js
const splittedId = Math.random();
msg.innerSplit = { index: 0, _id: splittedId, total: 1 };
msg.subTestCase = '120verify';
node.send(msg);
return;
```

---

## `msg` Object Conventions

```js
msg.test = {
  ok: true,         // false as soon as any assertion fails
  asserts: [],      // [{ name, ok, details }]
  evidence: []      // collected protocol evidence
}

msg.ci = {
  allowedAlgs: [],
  forbiddenAlgs: []
}

// Per-step status:
msg.ci_metadata        // { status, usedMock }
msg.oauth_as_metadata  // { status, usedMock }
msg.par                // { status, usedMock, ... }

msg.testId             // e.g. "CI_050"
msg.subTestCase        // e.g. "050ok" or "120verify"
msg.innerSplit         // { index, _id, total }
```

### HTTP Response format

Default compact response: `{ ok, testId, parEndpoint, usedMockCiMetadata, usedMockOauthMetadata, failed? }`

Add `?verbose=true` to get the full `msg.test` report in `response.report`.

---

## ciCommon.js — Shared Utility

Located at `lib/ciCommon.js`, exported as a factory `(global) => { ... }`. Loaded once at Node-RED start.

```js
const ci = global.get("ciCommon");

ci.initTest(msg)                         // init msg.test and msg.ci
ci.addAssert(test, name, ok, details)    // append assertion; sets test.ok=false on failure
ci.buildResponse(msg)                    // populate msg.payload (compact or verbose)
ci.jwkThumbprint(jwk)                    // SHA-256 JWK thumbprint → base64url
ci.getPayloadObject(msg)                 // parse msg.payload to object (handles JSON string)
ci.commonRequestUriAssert(msg)           // assert request_uri present in PAR response
ci.commonAlgAssert(msg)                  // assert alg in oauth client attestation cnf.jwk
ci.mockTracking(msg)                     // record metadata/PAR mock-usage assertions
ci.base64url(buf)                        // Buffer → base64url string
```

---

## Mock Server (Well know mocked server tab)

Hosts static HTTP endpoints for `.well-known/openid-credential-issuer` and `.well-known/oauth-authorization-server`. Used when real upstream endpoints are unavailable. The `HTTP Request + Mock Fallback` subflow tries the real endpoint first; on failure it calls the mock and sets `msg.*.usedMock = true`.

---

## GitHub Actions

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `claude.yml` | `@claude` mention in issue/PR | Runs `claude-code-action@v1` to fulfill the instruction |
| `claude-code-review.yml` | PR opened/updated | Auto code review via `code-review` plugin |

Both require `ANTHROPIC_API_KEY` in repository secrets.

---

## Test Case Ranges

| Range | Protocol step | Notes |
|-------|--------------|-------|
| CI_001–CI_049 | PAR, metadata | Earlier tests |
| CI_050–CI_063 | PAR, Auth response | Reference tests (used as doc examples) |
| CI_064–CI_119 | Nonce, Token, Credential | Mid-range tests |
| CI_120–CI_132 | Credential (SD-JWT-VC) | Structural SD-JWT verification |

CI_999 is a catch-all/smoke test ID visible in the assert switches.

---

## Adding a New CI Test (SD-JWT range, CI_120+)

Use the Claude skill `/ipzs-create-assert` for guided step-by-step creation.

Manual procedure for the **SD-JWT credential group**:

| What | Node ID | Tab |
|------|---------|-----|
| Post-Credential switch | `2f0f133fb016e61a` | Generation Flow CI |
| Gen group | `5cf99e9328eb55e5` | Generation Flow CI |
| Link-out Post Credential | `fd669d8614d0c9eb` | Generation Flow CI |
| Assert switch | `assert_sdjwt_switch_001` | Assert Flow CI |
| Assert group | `assert_sdjwt_group_001` | Assert Flow CI |
| Assert change node | `assert_sdjwt_change_001` | Assert Flow CI |

Steps for CI_XXX:
1. Add `eq "CI_XXX"` rule + output wire in both switches; increment `outputs` by 1.
2. Add `gen_sdjwt_set_ciXXX_001` to gen group `nodes`; add `assert_sdjwt_ciXXX_001` to assert group `nodes` (before `assert_sdjwt_change_001`).
3. Insert Set node (wires to `fd669d8614d0c9eb`) after the last Set node in the file.
4. Insert Assert node (wires to `assert_sdjwt_change_001`) before `assert_sdjwt_change_001` in the file.
5. Deploy (Ctrl+Shift+D) then test: `POST http://localhost:1880/credential-issuer/tests/CI_XXX?verbose=true`
6. Commit via Git panel.
