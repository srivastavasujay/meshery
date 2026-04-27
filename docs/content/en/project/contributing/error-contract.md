---
title: HTTP Error Response Contract
description: How Meshery Server responds to error conditions over HTTP, and how clients should parse those responses.
categories: [contributing]
---

# HTTP Error Response Contract

Every non-2xx HTTP response from Meshery Server carries a JSON body with
`Content-Type: application/json; charset=utf-8`. Clients should parse the body
as JSON before surfacing errors to users.

> **Migration status:** the JSON contract is enforced for all server endpoints
> in `server/handlers/` and `server/models/*provider*.go`. The `forbidigo` lint
> rule was added in advisory mode in PR
> [#18919](https://github.com/meshery/meshery/pull/18919) (CI currently runs
> with `--new-from-rev=origin/master`, so newly introduced `http.Error` calls
> in handlers/models block merge while pre-existing usage is surfaced
> non-blockingly); PR
> [#18927](https://github.com/meshery/meshery/pull/18927) flips it to fully
> blocking. Legitimate exceptions are SSE stream handlers
> (`events_streamer.go`, `load_test_handler.go`), Kubernetes healthz probes,
> the static-asset UI handler, binary/tar/YAML downloads, and HTTP redirects.

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

### Streaming JSON responses

When a handler writes a JSON body via `json.NewEncoder(w).Encode(...)`,
encoding into the `ResponseWriter` directly commits headers + status before
the encoder errors. If the encode fails partway, you cannot emit a fresh
error response — the wire already has a partial JSON envelope and a 200 OK
status. **Buffer first, write once:**

```go
var buf bytes.Buffer
if err := json.NewEncoder(&buf).Encode(payload); err != nil {
    // No headers committed yet — safe to emit a fresh error response.
    writeMeshkitError(w, models.ErrEncoding(err, "<object name>"), http.StatusInternalServerError)
    return
}
w.Header().Set("Content-Type", "application/json; charset=utf-8")
if _, err := w.Write(buf.Bytes()); err != nil {
    // Client likely disconnected; headers already committed, can't emit
    // a new error. Log and move on.
    h.log.Warn(fmt.Sprintf("write response: %v", err))
}
```

The provider layer (`server/models/remote_provider.go`,
`server/models/default_local_provider.go`) follows this pattern — see commit
`ed1ce9f25c` for reference call sites.

Legitimate exceptions (enforced by a `forbidigo` allowlist in `.github/.golangci.yml`; the lint guard was added in advisory mode in PR [#18919](https://github.com/meshery/meshery/pull/18919) and is being flipped to fully blocking in PR [#18927](https://github.com/meshery/meshery/pull/18927)):
- SSE stream handlers, e.g. `events_streamer.go` and `load_test_handler.go` (`Content-Type: text/event-stream`)
- Kubernetes healthz probes, e.g. `k8s_healthz_handler.go` (plain text is the probe contract)
- Static-asset UI handler (`ui_handler.go`) — asset-resolution errors, not API surface
- Binary/tar/YAML downloads
- HTTP redirects (no body)
