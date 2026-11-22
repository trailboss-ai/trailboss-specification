# trailboss serve

`trailboss serve` runs a local HTTP server for frontends. Each endpoint answers exactly what one command prints with `--json`, byte for byte (the trailing newline included), so the schemas in [../json/](../json/) describe the bodies, and a frontend can move between running the CLI and asking the server without changing how it reads the answers.

Every endpoint is a `GET`, and no request types into an agent, starts one or changes your settings. One endpoint, [`/events`](#the-event-stream), streams what `trailboss watch` reports; while a client follows it, the server watches as `trailboss watch` does, with the same effects (see there).

## Starting it

```sh
trailboss serve [--port 7420] [--interval 10]
```

- It listens on `127.0.0.1` only, on `--port` (`0` picks a free port).
- It indexes new transcript data when it starts and every `--interval` seconds after, so answers stay current; CLI commands can run while it does.
- With `--json` it prints one line once it's listening ([`serve` schema](../json/serve.schema.json)): the `url`, the `port`, the `token_file` and, on the start that made the token, `open_url`. Without `--json` it prints the same as text.

### The token

Every request needs a token, reads included. It's made on the first start and kept in a file beside Trailboss's index (`token_file`), readable by its owner only; later starts use the same one. The first start prints it once, as a URL with the token in its fragment: `http://127.0.0.1:7420/#token=<token>`. Browsers never send a fragment to a server, so the token doesn't end up in any request log.

A frontend that starts the server itself reads the token from `token_file`. One that finds a server already running needs the token from the user, or from that file.

## Requests

Each request needs:

- **`Host`**: `127.0.0.1:<port>` or `localhost:<port>`, with the server's port. Anything else is refused, which stops a page on another name from reaching the server through DNS rebinding.
- **`Authorization: Bearer <token>`**.
- **No foreign `Origin`.** A request with an `Origin` header is answered only when it's the server's own, `http://127.0.0.1:<port>` or `http://localhost:<port>`. Requests from programs, which send no `Origin`, are fine.

The server sends no CORS headers, so a page from anywhere else can't read its answers.

The checks run in this order: `Host`, `Origin`, token, method, endpoint. Request bodies are never read.

### Query options

An endpoint's options are its command's long flag names:

- A flag that takes a value: `name=value` (`/sessions?limit=5&agent=claude,codex`). A flag that takes several values takes them comma-separated, as on the command line.
- A switch: `name`, `name=1` or `name=true`; `name=0` or `name=false` leaves it off (`/live/board?all`).
- Values are percent-encoded; `+` is a space.
- An option an endpoint doesn't list below is refused with `400`, never ignored.

## Endpoints

| Endpoint | Answers what this prints | Query options | Schema |
|---|---|---|---|
| `GET /version` | `trailboss --json version` | | [version](../json/version.schema.json) |
| `GET /doctor` | `trailboss --json doctor` | | [doctor](../json/doctor.schema.json) |
| `GET /events` | `trailboss --json events`, as a stream of [server-sent events](#the-event-stream) | | [events](../json/events.schema.json) |
| `GET /live` | `trailboss --json live` | | [live](../json/live.schema.json) |
| `GET /live/board` | `trailboss --json live --board` | `all` | [live-board](../json/live-board.schema.json) |
| `GET /sessions` | `trailboss --json sessions` | `limit`, `agent`, `under`, `filter` | [sessions](../json/sessions.schema.json) |
| `GET /search` | `trailboss --json search <q>...` | `q` (the words, required; repeat it for several), `limit`, `all` | [search](../json/search.schema.json) |
| `GET /thread/<AGENT:ID>` | `trailboss --json thread <AGENT:ID>` | `tree` | [thread](../json/thread.schema.json), with `tree`: [thread-tree](../json/thread-tree.schema.json) |
| `GET /recap` | `trailboss --json recap [<session>]` | `session`, `since` | [recap](../json/recap.schema.json) |
| `GET /usage` | `trailboss --json usage` | `by`, `since` | [usage](../json/usage.schema.json) |
| `GET /quota` | `trailboss --json quota` | `provider` | [quota](../json/quota.schema.json) |
| `GET /plans` | `trailboss --json plans` | | [plans](../json/plans.schema.json) |
| `GET /insights` | `trailboss --json insights` | `period`, `year`, `provider`, `model`, `day`, `no-cache`, `monday` | [insights](../json/insights.schema.json) |
| `GET /stats` | `trailboss --json stats` | `days`, `prs` | [stats](../json/stats.schema.json) |
| `GET /task` | `trailboss --json task` | | [task](../json/task.schema.json) |
| `GET /queue` | `trailboss --json queue` | | [queue](../json/queue.schema.json) |

The session in `/thread/<AGENT:ID>` is percent-encoded like any path segment (`/thread/claude%3As-1` or `/thread/claude:s-1`).

## The event stream

`GET /events` answers `200` with `Content-Type: text/event-stream` and keeps the connection open: a [server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html) stream, which a browser's `EventSource` reads (sending the token needs a client that can set the `Authorization` header). It takes no query options. It needs the same `Host`, `Origin` and token as every other request.

### Events

The server looks at your agents every `--interval` seconds, as `trailboss watch` does, and sends what each look finds:

- One event per item of `trailboss --json events`' `events` array: the event's name is the item's `kind` (`needs_you`, `turn_finished`, `finished`, `ended`, `same_folder`, `stuck`, `spend_alert`; more may be added, so skip names you don't know), and its `data` is the item, as one line of JSON.
- Then a `live` event, whose `data` is that look's rows: the `live` array of `trailboss --json events`. A client redraws from it and never needs to ask `/live` again.

The [events schema](../json/events.schema.json) describes both. Lines starting with `:` are comments that keep a quiet connection open; ignore them.

```text
id: 1795320000000-41
event: needs_you
data: {"kind":"needs_you","key":"herdr:t1","where":"t1 work · app","agent":"claude",...}

id: 1795320000000-42
event: live
data: [{"where":"t1 work · app","agent":"claude","status":"blocked",...}]

```

A client that connects gets the latest `live` event straight away when there is one; with no client before it, the server looks at once and sends that look's `live` event.

### Ids and reconnecting

Every event has an `id`. Ids are opaque strings: compare them for equality only. A client that reconnects with the last id it got, in a `Last-Event-ID` header (`EventSource` does this on its own), gets the events it missed, then the latest `live` event. When the server no longer has all of them, or the id isn't one this server gave (after a restart, say), it sends the latest `live` event alone, which is all a client needs to redraw.

### What following does

The server looks only while at least one client follows `/events`, once per interval however many follow. Each look is the same look `trailboss watch` and `trailboss events` make, with the same effects, so while a client follows, the server also does what `watch` does at each look: types messages queued with `send --queue` when their agent's turn ends, starts queued tasks and automatic reviewers that are due, records spend alerts and saves automatic checkpoints. Each of these happens once, whether the server, `watch` or `events` saw the change. For the same reason, an event goes to whichever of them looked first: when `trailboss watch` runs beside a followed server, each change is reported by one of the two, not both.

With no client following, the server makes no look and none of this happens.

## Responses

- **`200`**: the command's `--json` output, `Content-Type: application/json`, `Cache-Control: no-store`. `/doctor` answers `200` even when a check fails, with `ok: false` (where the command exits 1).
- **Errors** have the body `{"error": "<message>"}`:

| Status | When |
|---|---|
| `400` | An option the endpoint doesn't take, a value that can't be read, or one the command refuses. |
| `401` | No token, or the wrong one (with `WWW-Authenticate: Bearer`). |
| `403` | A `Host` other than the server's, or a foreign `Origin`. |
| `404` | No such endpoint. |
| `405` | A method other than `GET` (with `Allow: GET`). |
| `503` | `/events` only: too many clients already follow it. |
| `500` | The command failed; `error` is the message the CLI prints for it. |

## Versioning

The protocol follows [VERSIONING.md](../VERSIONING.md): endpoints and query options are added, never renamed or removed without raising `api`, and the bodies change only as their commands' `--json` does. `GET /version` returns the `api`.
