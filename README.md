# VybRover

A modern reading surface for [Hermes](https://hermes-agent.nousresearch.com) sessions —
written in **vybey**, the language of the [Vyb](https://github.com/rickenator/Vyb) compiler.

Hermes' built-in chat view renders the live TUI (xterm) in the browser. That's faithful
and charming, but it is a terminal: text reflows in place, there are no roles, no
timestamps, and tool calls scroll past as raw output. VybRover reads Hermes' own
structured session API and renders a transcript instead — roles, timestamps, markdown
responses, and collapsible tool calls.

## Why it's a separate program

VybRover does **not** patch Hermes' frontend. It talks to Hermes' REST API
(`GET /api/sessions`, `/messages`) and serves its own UI, so a `hermes update` can never
clobber it, and nothing in the Hermes install is modified.

## Requirements

- Hermes running with the dashboard/serve endpoint reachable (default `127.0.0.1:9119`)
- A Hermes dashboard session token (`X-Hermes-Session-Token`)
- Vyb compiler + stdlib (`VYB_STDLIB`)

## Running

```sh
export VYB_STDLIB=/path/to/Vyb/stdlib
export HERMES_TRANSCRIPT_TOKEN="$(cat ~/.hermes/.dashboard-token)"
export HERMES_BASE=127.0.0.1 HERMES_PORT=9119
export VYBROVER_STATIC_DIR=/path/to/VybRover VYBROVER_PORT=9210

/path/to/Vyb/build/vyb vybrover.vyb
```

Then open <http://127.0.0.1:9210>.

Production note: the token belongs in an environment variable or a `0600` file, never on
a command line or in a transcript.

## Environment variables

| Variable | Default | Meaning |
|---|---|---|
| `HERMES_TRANSCRIPT_TOKEN` | (none) | Dashboard session token sent as `X-Hermes-Session-Token` |
| `HERMES_BASE` | `127.0.0.1` | Hermes API host |
| `HERMES_PORT` | `9119` | Hermes API port |
| `VYBROVER_STATIC_DIR` | `.` | Directory holding `index.html` |
| `VYBROVER_PORT` | `9210` | Port VybRover listens on |

## Routes

| Route | Purpose |
|---|---|
| `/` | The transcript UI (`index.html`) |
| `/healthz` | Liveness: `{"ok":true}` |
| `/api/*` | Authenticated pass-through to the Hermes API |

## What this exercises in vybey

This is a real workload, not a toy:

- **`http` module as a genuine server** — `http_listen` / `http_accept` / `http_read_head`
  / `http_request_path` / `http_send_all` / `http_close`, rather than a harness.
- **`aspect` + `bind`** — an `Emittable` aspect with binds for `String` and `Vec<String>`,
  so route bodies compose uniformly.
- **Fallibility discipline** — every transport op is `T?` (absent-on-failure) and each is
  forced to be handled via `else`; no integer sentinels are smuggled as data.
- **`network` sockets directly** for the upstream round-trip, because Hermes' API needs a
  request header (`http_get_full` takes no headers).

## API parameters (Hermes)

Hermes' API validates its query parameters strictly — worth knowing:

- `GET /api/sessions` accepts `order = created | recent`, and `limit <= 100`
- `GET /api/sessions/{id}/messages` accepts `order = oldest | latest`

Sending `asc`/`recent` to `/messages` or `limit=200` returns a `detail` error envelope.

## Status

Working end to end: session list, per-session transcript, collapsible tool calls,
authenticated API proxy. Verified against a live Hermes instance.
