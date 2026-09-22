# VybRover

A modern reading surface for [Hermes](https://hermes-agent.nousresearch.com) sessions —
written in **vybey**, the language of the [Vyb](https://github.com/rickenator/Vyb) compiler.

Hermes' built-in chat view renders the live TUI (xterm) in the browser. That's faithful
and charming, but it is a terminal: text reflows in place, there are no roles, no
timestamps, and tool calls scroll past as raw output. VybRover reads Hermes' own
structured session API and renders a transcript instead — roles, timestamps, markdown
responses, and collapsible tool calls.

It also **continues** a conversation. Read a transcript, then pick up where the
discussion left off, with the reply streaming into a composer pane beside it.

## Why it's a separate program

VybRover does **not** patch Hermes' frontend. It talks to Hermes' REST API
(`GET /api/sessions`, `/messages`) and serves its own UI, so a `hermes update` can never
clobber it, and nothing in the Hermes install is modified.

## New chat, and continuing a discussion

Reading needs only a `GET`; writing needs a **WebSocket**, on any of three paths:

- **＋ New chat** — `session.create`: a fresh conversation, no parent.
- **⑂ Continue** — resume the open transcript, then `session.branch` it: a new session carrying the
  parent's history, leaving the original untouched.
- **Resume** — take a session as-is (`/api/attach`), when no fork is wanted.

Hermes exposes no HTTP route that submits a turn — across its API the write verbs (`prompt.submit`,
`session.steer`, `session.redirect`, `session.interrupt`) exist only as JSON-RPC methods over
`/api/ws`. Hermes' own Chat tab opens that socket for the same reason. VybRover holds it directly,
using Vyb's `stdlib/websocket` (an RFC 6455 client), and streams the reply back to the page over its
own `/api/events` poll.

**Why Continue forks.** Hermes refuses writes into a session another window holds (`4090
SESSION_NOT_OWNED`), so continuing a transcript resumes it and then branches it. The child gets its
own lease; the original is never written to. That is why the button says *continue*, not *reopen*:
the transcript you were reading stays as it was.

One writer at a time: attaching releases the previous lease, since the composer is a single
conversation. `GET /api/state` reports what the slot holds, so a page reload reattaches instead of
falsely showing "not attached", and **✕** releases it deliberately.

Keyboard: **n** new chat, **Enter** sends (Shift+Enter is a newline).

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
| `VYBROVER_CWD` | `.` | Working directory for sessions VybRover creates |

## Routes

| Route | Purpose |
|---|---|
| `/` | The transcript UI (`index.html`) |
| `/healthz` | Liveness: `{"ok":true}` |
| `POST /api/new` | Start a fresh conversation (`session.create`, optional `{"title": "..."}`) |
| `POST /api/fork` | Resume a session, branch it, and attach the writer (`{"session_id": "..."}`) |
| `POST /api/attach` | Resume and attach without branching |
| `POST /api/send` | Submit a turn on the attached writer (`{"text": "..."}`) |
| `GET /api/events` | Drain the writer's queued events as JSON |
| `GET /api/state` | What the writer slot holds: `{"live": bool, "stored_id": ...}` |
| `POST /api/detach` | Release the writer lease |
| `/api/*` | Authenticated pass-through to the Hermes API |

## What this exercises in vybey

This is a real workload, not a toy:

- **`http` module as a genuine server** — `http_listen` / `http_accept` /
  `http_request_path` / `http_send_all` / `http_close`, rather than a harness.
- **`websocket` module as a real client** — the RFC 6455 handshake, masked frames, and a
  streaming reader, against Hermes' live control plane.
- **`aspect` + `bind`** — an `Emittable` aspect with binds for `String` and `Vec<String>`,
  so route bodies compose uniformly.
- **Fallibility discipline** — every transport op is `T?` (absent-on-failure) and each is
  forced to be handled via `else`; no integer sentinels are smuggled as data.
- **`our<T>` + `mutex`** — the writer slot is shared ref-counted state mutated safely
  from the HTTP worker threads.
- **`network` sockets directly** for the upstream round-trip, because Hermes' API needs a
  request header (`http_get_full` takes no headers).

## Two Hermes quirks worth knowing

Both cost real debugging time; neither is documented upstream:

- **`http_read_head` discards the body.** The stdlib helper returns only the head
  (`buf.substring(0, idx + 4)`) and drops the rest of the chunk it read. A `POST` body
  commonly arrives in the same TCP segment, and once dropped the bytes are gone — a later
  body read then waits forever. VybRover reads the head itself, keeping the overflow.
- **`session.branch` rejects `omit_messages`** while `session.resume` accepts it. Without
  it, resume embeds the whole transcript (500 KB+ on a long session) and first-match
  scraping can grab a transcript message's own `session_id`. Branch, by contrast, always
  returns the child's full transcript, so the reader needs a large frame budget rather
  than a small one.

## API parameters (Hermes)

Hermes' API validates its query parameters strictly — worth knowing:

- `GET /api/sessions` accepts `order = created | recent`, and `limit <= 100`
- `GET /api/sessions/{id}/messages` accepts `order = oldest | latest`

Sending `asc`/`recent` to `/messages` or `limit=200` returns a `detail` error envelope.

## Status

Working end to end: session list, per-session transcript, collapsible tool calls,
authenticated API proxy, **and the writer** — start a new chat, continue a discussion by forking it,
submit turns, and stream the reply into the composer pane. Verified against a live Hermes instance:

- **＋ New chat** creating a fresh session in ~0.05 s, then a turn returning `NEW-CHAT-OK`
- a fork of a stored session and of a branch child, each ~0.6 s
- a submitted turn streaming `reply → start → thinking → answer → done`, with the final
  answer rendered in the pane (`msg bot: Hermes UI-NEWCHAT-OK`)
- a page reload reattaching to the live writer via `/api/state`, and **✕** releasing it

Not built: steering a turn mid-flight (`session.steer` / `session.interrupt`), attaching
to a session another window currently holds (Hermes refuses it; fork is the sanctioned
path), and more than one composer at a time.
