# Lambda Lang + ASAP Protocol Integration

This document describes how Lambda Lang is used within [ASAP Protocol](https://github.com/adriannoes/asap-protocol) for semantic compression of JSON-RPC 2.0 payloads.

## Overview

| Component | Purpose |
|-----------|---------|
| **ASAP Protocol** | Transport layer — JSON-RPC 2.0 over HTTP/WebSocket, task orchestration |
| **Lambda Lang** | Compression layer — semantic atom substitution for payload size reduction |

Together they provide compressed agent-to-agent communication:

```
┌─────────────────────────────────────────┐
│  Lambda Lang (semantic compression)     │  ← "How to compress"
│  §Jrpc§ §Mthd§ §Treq§ §Hb§              │
├─────────────────────────────────────────┤
│  ASAP Protocol (transport layer)        │  ← "How to communicate"
│  JSON-RPC 2.0 / HTTP / WebSocket        │
└─────────────────────────────────────────┘
```

## Content-Type Negotiation

ASAP uses standard HTTP content negotiation to enable Lambda compression. The mechanism is fully opt-in — when not requested, everything stays JSON.

### Request flow

```
Agent A                                          Agent B
  │                                                │
  │──── POST /asap ─────────────────────────────▶  │
  │     Accept: application/vnd.asap+lambda        │
  │                                                │
  │  ◀──── 200 OK ───────────────────────────────  │
  │        Content-Type: application/vnd.asap+lambda
  │        Body: λ1:{§Jrpc§:§V2§, §Mthd§:§Am§, …}  │
  │                                                │
```

### Content-Type

```
application/vnd.asap+lambda
```

### Version prefix

All encoded payloads start with `λ1:` for forward compatibility. Future codec versions can use `λ2:`, etc.

### Fallback behavior

- If the server cannot encode with Lambda, it falls back to `application/json` silently
- Error responses are always returned as JSON regardless of `Accept` header
- Wildcard `Accept: */*` does NOT trigger Lambda encoding (conservative negotiation)

## Atom Mapping

The codec substitutes common JSON-RPC and ASAP keys with short Lambda atoms wrapped in `§…§` delimiters (chosen because `§` cannot appear in valid JSON).

### JSON-RPC standard keys

| JSON key | Lambda atom |
|----------|-------------|
| `"jsonrpc"` | `§Jrpc§` |
| `"method"` | `§Mthd§` |
| `"params"` | `§Prms§` |
| `"result"` | `§Rslt§` |
| `"error"` | `§Er§` |
| `"id"` | `§Id§` |
| `"code"` | `§Cd§` |
| `"message"` | `§Msg§` |
| `"data"` | `§Dt§` |

### ASAP envelope keys

| JSON key | Lambda atom |
|----------|-------------|
| `"envelope"` | `§Env§` |
| `"sender"` | `§Snd§` |
| `"recipient"` | `§Rcp§` |
| `"payload"` | `§Pld§` |
| `"payload_type"` | `§Pt§` |
| `"task_id"` | `§Ta§` |
| `"status"` | `§St§` |
| `"trace_id"` | `§Tr§` |
| `"timestamp"` | `§Ts§` |
| `"version"` | `§Vr§` |

### ASAP values

| JSON value | Lambda atom |
|------------|-------------|
| `"asap.message"` | `§Am§` |
| `"2.0"` | `§V2§` |
| `"task.request"` | `§Treq§` |
| `"task.response"` | `§Tres§` |
| `"task.status"` | `§Tst§` |
| `"heartbeat"` | `§Hb§` |
| `"success"` | `§Ok§` |
| `"pending"` | `§Pn§` |
| `"running"` | `§Rn§` |
| `"completed"` | `§Cp§` |
| `"failed"` | `§Fl§` |
| `"cancelled"` | `§Cx§` |

Total: **35 atom mappings** covering the full JSON-RPC + ASAP envelope vocabulary.

## Compression Example

### Original JSON (235 chars)

```json
{"jsonrpc":"2.0","method":"asap.message","id":"abc-123","params":{"envelope":{"sender":"agent-a","recipient":"agent-b","payload_type":"task.request","status":"pending","trace_id":"tr-456"}}}
```

### Lambda-encoded (170 chars)

```
λ1:{§Jrpc§:§V2§,§Mthd§:§Am§,§Id§:"abc-123",§Prms§:{§Env§:{§Snd§:"agent-a",§Rcp§:"agent-b",§Pt§:§Treq§,§St§:§Pn§,§Tr§:"tr-456"}}}
```

**Compression ratio: ~28% reduction** on this typical ASAP envelope. Savings increase with larger payloads containing more substitutable keys.

## Implementation Details

### Codec architecture

- **Self-contained**: No dependency on the `lambda-lang` Python package — all atom mappings are inline
- **Single-pass regex**: Pre-compiled `re.compile()` pattern for C-level substitution speed (no Python `for` loop)
- **String-based API**: `encode(json_str) → str` and `decode(encoded) → str` — accepts pre-serialized JSON to leverage Pydantic v2's Rust core (`model_dump_json()`)
- **100% fidelity**: `decode(encode(s)) == s` for any valid JSON string

### Event loop safety

Both server-side encoding and client-side decoding are offloaded via `asyncio.to_thread()` to prevent blocking FastAPI's event loop on large payloads.

### Source code

- **Codec**: [`src/asap/transport/codecs/lambda_codec.py`](https://github.com/adriannoes/asap-protocol/blob/main/src/asap/transport/codecs/lambda_codec.py)
- **Server negotiation**: [`src/asap/transport/server.py`](https://github.com/adriannoes/asap-protocol/blob/main/src/asap/transport/server.py)
- **Client negotiation**: [`src/asap/transport/client.py`](https://github.com/adriannoes/asap-protocol/blob/main/src/asap/transport/client.py)
- **Unit tests**: [`tests/transport/unit/test_lambda_codec.py`](https://github.com/adriannoes/asap-protocol/blob/main/tests/transport/unit/test_lambda_codec.py) (27 tests)
- **Integration tests**: [`tests/transport/integration/test_lambda_negotiation.py`](https://github.com/adriannoes/asap-protocol/blob/main/tests/transport/integration/test_lambda_negotiation.py) (7 tests)

## Resources

- [ASAP Protocol Repository](https://github.com/adriannoes/asap-protocol)
- [Lambda Lang Codec Source](https://github.com/adriannoes/asap-protocol/blob/main/src/asap/transport/codecs/lambda_codec.py)
- [PR #71 — Original integration](https://github.com/adriannoes/asap-protocol/pull/71)
- [Issue #52 — Original proposal](https://github.com/adriannoes/asap-protocol/issues/52)
- [Lambda Lang Core Spec](../spec/v0.1-core.md)

---

*Integration designed by [@adriannoes](https://github.com/adriannoes) — ASAP Protocol maintainer*
