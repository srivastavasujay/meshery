---
title: HTTP Error Response Contract
description: How Meshery Server responds to error conditions over HTTP, and how clients should parse those responses.
categories: [contributing]
---

# HTTP Error Response Contract

Every non-2xx HTTP response from Meshery Server carries a JSON body with
`Content-Type: application/json; charset=utf-8`. Clients should parse the body
as JSON before surfacing errors to users.

## Shape

```json
{
  "error": "Human-readable short description",
  "code": "meshery-server-1033",
  "severity": "ERROR",
  "probable_cause": ["Connection to the remote provider timed out."],
  "suggested_remediation": ["Verify that Meshery Cloud is reachable."],
  "long_description": ["Full technical details suitable for logs."]
}
```

### Fields

| Field | Required | Notes |
|-------|----------|-------|
| `error` | yes | User-facing message. For MeshKit errors, this is the ShortDescription. |
| `code` | when available | MeshKit error code (e.g. `meshery-server-1033`). Stable across releases. Use for telemetry, i18n lookup, and programmatic handling. |
| `severity` | when available | One of `EMERGENCY`, `ALERT`, `CRITICAL`, `FATAL`, `ERROR`. |
| `probable_cause` | optional | Array of strings. |
| `suggested_remediation` | optional | Array of strings. Surface to users when present. |
| `long_description` | optional | Array of strings. Suitable for developer logs; may contain stack-style detail. |

Fields marked "when available" are omitted (via `omitempty`) for errors that
originated outside the MeshKit error catalog.

## Client contract

- Do not rely on plain-text error bodies — they are always JSON.
- When `code` is present, prefer it over string matching on `error`.
- When `suggested_remediation` is non-empty, surface it alongside `error`.
- When the body is not valid JSON, treat the response as a bug and report
  the offending endpoint; do not attempt a text fallback.

## Producing errors in handlers

Use `writeMeshkitError(w, err, status)` in `server/handlers/utils.go`:

```go
if err != nil {
    h.log.Error(ErrGetResult(err))
    writeMeshkitError(w, ErrGetResult(err), http.StatusNotFound)
    return
}
```

For bare-string errors without a MeshKit code, use `writeJSONError(w, msg, status)`.
Every bare-string error is a candidate for promotion to a MeshKit error —
prefer adding a code when fixing an adjacent bug.

Do not use `http.Error` in handlers or provider code. It writes
`Content-Type: text/plain` and strips MeshKit metadata, which crashes
RTK Query's default baseQuery on the UI.

Legitimate exceptions (enforced by `.github/.golangci.yml` allowlist):
- SSE stream handlers (`Content-Type: text/event-stream`)
- Kubernetes healthz probes (plain text is the probe contract)
- Binary/tar/YAML downloads
- HTTP redirects (no body)
