# moon_router

Chi-style HTTP routing for moon's `std.http_server` (plan 57): URL patterns
with params and wildcards, method dispatch with correct 404/405 semantics,
nested groups, onion middleware, CORS, timeout classes, panic-free error
envelopes, and structured request logging via `std.log`.

Plus what a request meets on the way through (plan 137), because a service
otherwise writes each of them again: request-body binding, typed query reads,
async middleware, a pluggable error-body shape, and file routes that stream
their bytes without leaving the router.

```moon
use moon_router.router
use moon_router.router.{Router, Ctx}
use moon_router.envelope
use moon_router.middleware

fn build() -> Router {
  let r = router.new()
  r.use_mw(middleware.logging())
  r.use_mw(middleware.cors(middleware.permissive()))

  r.get("videos", |ctx| envelope.success("[]"))
  r.get("videos/{id}/stream", |ctx| {
    // Streaming routes respond through the raw request (plan 55) and
    // return the sentinel so the serve loop skips the ordinary respond.
    let served = std.http_server.serve_file(ctx.req, media_path(ctx.param("id")), "video/mp4")
    router.responded()
  })

  let api = r.group("api/v1")
  api.get("health", |ctx| std.http_server.text(200, "ok"))
  api.delete("import/downloads", |ctx| envelope.success("{}"))
  api.delete("import/downloads/{jobId}", |ctx| envelope.success("{}"))

  // Timeout classes per group: a short default, minutes for batch work,
  // none for streaming. Enforced by dispatch_async.
  let batch = r.group("import").with_timeout(120000)
  batch.post("batch", |ctx| envelope.success("{}"))
  r
}

async fn main() {
  let r = build()
  let server = std.http_server.listen("0.0.0.0", 8080)
  while true {
    let req = std.http_server.accept_stream(server)
    let response = await router.dispatch_async(r, req)
    if !router.is_responded(response) {
      let _sent = std.http_server.respond(req, response)
    }
  }
}
```

## Matching semantics (chi-compatible)

- Segments: literals, `{name}` params, and a trailing `{rest...}` wildcard
  capturing the remaining path (`""` for an empty tail).
- Literal segments beat params at the same depth, with backtracking — a
  literal `DELETE /import/downloads` registered beside
  `DELETE /import/downloads/{jobId}` resolves correctly in both directions.
- A matched path with the wrong method is **405** with a complete `Allow`
  header; anything else is **404**. Both carry the JSON error envelope.
- Params are percent-decoded; query strings are ignored for matching.
- A `HEAD` with no `HEAD` route of its own runs the `GET` one, which is what a
  HEAD is (RFC 9110) and what Go's `ServeMux`, axum, Express, Werkzeug and
  Django all do. `std.http_server` drops the body of a HEAD response at the
  connection, so a handler writes one response either way.
- `Router.find(method, path)` exposes the matcher for tests.
- `router.path_of(target)` is the path a target names: the query dropped, and a
  trailing slash with it. For anything that reads a target *back* — a log line,
  a metric label — since matching does not need it.

## Middleware

`fn(Ctx, next) -> Response` closures; first `use_mw`'d runs outermost. The
chain active on a group at registration time is captured per route. A
method-miss (405 and OPTIONS preflight) still runs the node's chain, which is
how `middleware.cors` answers preflights. In package:

- `middleware.logging()` — one `std.log` info record per request.
- `middleware.cors(options)` — preflight 204 + `Access-Control-*` headers;
  `middleware.permissive()` for the common defaults.

Every route that talks to a database is registered with `*_async`, and
`use_mw_async` feeds a chain of its own — a sync onion cannot wrap a leaf that
awaits — so the two above never fire for one. The async twins are what such a
service actually reaches:

```moon
r.use_mw_async(middleware.logging_async(LoggingOptions {
  // Paths logged only when they fail: a UI polling a job list emits thousands
  // of lines an hour that bury everything else.
  quiet: |path| path == "/api/health",
  // Zero or one, the package's idiom for an optional function. Empty is
  // `std.log`; a service that renders readable lines for a person passes its
  // own sink and gets a `RequestRecord{method, path, status, ms}`.
  sink: [],
}))
r.use_mw_async(middleware.cors_async(CorsOptions {
  allow_origin: "*",
  allow_methods: "GET, POST, PUT, DELETE, OPTIONS",
  allow_headers: "Accept, Authorization, Content-Type, Range",
  // Without `Content-Range` here a cross-origin player cannot seek.
  expose_headers: "Content-Length, Content-Range, Accept-Ranges",
  allow_credentials: true,
  max_age: 300,
  // Answer OPTIONS from these rules alone, ahead of matching: a preflight to a
  // path no route claims must not be a 404, or the browser refuses the real
  // request that would have followed. Off by default, which is chi's behaviour.
  preflight_unmatched: true,
}))
```

`middleware.with_cors(response, options)` attaches the same headers to a
response that never reached the chain — which is every fallback, by
construction.

Recovery and timeouts need the scheduler, so they are properties of
`router.dispatch_async` (accept-loop servers): a failed handler becomes a
500 envelope with the detail logged (never leaked to the client), and a route
whose group has `with_timeout(ms)` returns a 503 envelope on expiry (the
handler task is not preempted, mirroring chi). The synchronous
`r.handler()` / `r.dispatch(req)` path — pluggable into
`std.http_server.serve` — runs the same routes without those two.

## Envelope

`envelope.success(json)`, `envelope.ok_status(status, json)`, and
`envelope.app_error(status, code, message, field)` produce one `AppError`
shape: `{"error":{"message":…,"code":…,"field":…,"status_code":…}}`.

A handler usually holds a `Json` rather than text, so `envelope.json(status,
value)`, `envelope.ok_json(value)`, `envelope.no_content()` and
`envelope.from_json_result(result)` take the value and set the content type.

`envelope.status_for(name)` / `code_for(name)` are the taxonomy: an error's
*name* is its wire code, its status is looked up from it, and anything
unclassified reports as a 500 `INTERNAL_ERROR` so a `SqlError` cannot describe
the server's internals to a client. A validation error names its field through
`field_message(msg, field)`, which hides it behind U+0001 in the message —
moon has no user-constructible error payload — and `split_field` takes it out
again.

### A body shape the service defines

A port has to match the bytes of the thing it replaced, and those are rarely
this package's nested shape:

```moon
// Go's flat AppError, encoded the way Go encodes it.
envelope.use_error_body(|status, code, message, field| my_flat_body(status, code, message, field))
```

Set once at boot; `envelope.flat_error_body` is a stock flat shape and
`clear_error_body()` restores the default. Process-wide rather than a `Router`
field on purpose: a body shape belongs to a service's wire contract, and a
handler calling `from_error` has no router in scope to ask.

## Binding a request body

`moon_router.binding` is the layer a handler would otherwise write per field —
Hono's validator, Go's `ShouldBindJSON`:

```moon
let body = binding.json_body(ctx.req.body)?
let title = binding.require_non_empty(binding.string_field(body, "title")?, "title")?
let zoom = binding.optional_number_field(body, "zoomLevel")?
let tags = binding.int_array_field(body, "tagIds")?
```

Three answers, the same for every field: an absent field is the caller's
fallback (`""`/`[]`/`none`), `Json.Null` reads as absent, and a present field of
the wrong type is a 400. Every failure is
`err.VALIDATION_ERROR(envelope.field_message(msg, field))`, so the field name
comes back out at whatever error body the application installed — which is what
lets binding be package-level while the wire shape stays the application's.

`binding.strict_int(text)` is an optional sign and digits and nothing else:
`" 12"` and `"12abc"` are not twelve, and reading them as twelve is how a typo
in a URL silently operates on the wrong row. `require_strict_int` is the same
refusal as a 400 naming its field.

## Typed query reads

```moon
let limit = ctx.query_int_clamped("limit", 20, 1, 100)  // parse, default, clamp
let page = ctx.query_int("page", 1)
let force = ctx.query_bool("force", false)              // a bare `?force` is true
```

Unparseable falls back rather than failing, which is what a hand-rolled version
always chooses: a bad `?limit=` should still render the page. A route that wants
a bad value *refused* asks `binding.require_strict_int` instead. `fallback` is
clamped too, so a default outside the range cannot escape through the
missing-key path.

## File routes

A `Response` carries its whole body, and a multi-gigabyte video cannot be one.
`Router.get_stream` registers a route whose handler answers on the socket
itself:

```moon
r.get_stream("videos/{id}/stream", |ctx| files.serve(ctx, path_for(ctx.param("id")), "video/mp4"))
```

`files.serve` writes the file through `std.http_server.serve_file` — `HEAD`,
`Accept-Ranges`, one `Range`, `If-Range`, 206 slices and 416 — and returns
`router.responded()`. A missing file renders through the application's
not-found rather than `serve_file`'s plain-text 404, so a client cannot tell
file routes apart by the shape of a miss. Also in the module:
`files.content_type_for(path)` (an extension table), `files.serve_path` (the
two together), `files.is_read_method`, `files.header(req, name)` and
`files.range_start(req, size)`.

The point is not the helpers. It is that a streaming route is a *route*: it
appears in `Router.surface`, so a contract test sees the whole API rather than
the JSON half plus a hand-kept list of the rest.

## Fallbacks

The 404, 405, 500 and 503 above are this package's envelope, which is the right
answer for a service that has no other opinion. A port has one: it has to
answer what the thing it replaced answered, byte for byte, or clients can tell
the difference. `use_fallbacks` hands those four decisions back to the
application.

```moon
r.use_fallbacks(Fallbacks {
  not_found: |_ctx| std.http_server.text(404, "404 Not Found"),
  // A wrong verb on a known path answered as a miss, because the framework
  // being replaced did not distinguish one.
  method_not_allowed: |_ctx, _allow| std.http_server.text(404, "404 Not Found"),
  timed_out: |_ctx, _ms| my_error_body("SERVICE_UNAVAILABLE", "request timed out"),
})
```

It applies to the whole shared route storage rather than to one group — a 404
has no group to belong to. A handler that hits a checked programming bug ends
the process, as every Moon program does, so there is no failure fallback.

## Serving

`moon_router.serve` is the accept loop the router feeds, in a module of its
own so a consumer that wants the loop and not the trie pays for nothing:

```moon
use moon_router.serve

async fn serve_one(r: Router, req: Request) -> int {
  let response = await router.dispatch_async(r, req)
  if !router.is_responded(response) {
    let _sent = std.http_server.respond(req, response)
  }
  0
}

let drained = await serve.serve_with_shutdown(
  listener,
  |req| serve_one(app, req),
  || stop_everything_else(),
  30000,   // how long the drain may take; 0 keeps DEFAULT_DRAIN_MS
  60000,   // keep-alive idle timeout; 0 keeps std.http_server's 60 s
)
```

`std.http_server.serve` is right for a service whose handlers take a whole
request and return a whole response. A service that streams is not that
service, so the loop is `accept_stream` and the handler is handed the raw
request — which is also what makes `router.responded()` work end to end.

The order inside `serve_with_shutdown` is the module: accept in a spawned task,
wait for a stop signal (`SIGINT`/`SIGTERM`), **cancel the acceptor before
closing the listener** — a parked `accept` cannot notice a socket closed
underneath it — then run the caller's drain against a deadline. A second stop
signal abandons the drain. That pending signal wait keeps the default hybrid
clock paced, so the deadline also applies when work is blocked in native I/O.
It answers whether the drain finished in time; on false, the application can
exit non-zero. Cooperative cancellation cannot forcibly interrupt a native
call, so the application still owns its process-exit policy. An explicitly
pinned virtual clock retains its virtual-time behavior.
`accept_loop` and `wait_for_stop` are the
halves, for a service that wants a different arrangement.

## Testing without a socket

`router.detached_request(method, target, headers, body)` builds a `Request` that
never came off a connection, so `dispatch` and `dispatch_async` can be driven
from a unit test:

```moon
let answer = await router.dispatch_async(app, router.detached_request("GET", "/api/videos?tag=a", "", ""))
assert_eq(answer.status, 200)
```

Routing is the half of a service worth testing hardest and the half that needs
a live connection least: which route a path picks, what the params decode to,
what the middleware adds on the way out, and what answers a request nothing
matched are all settled before a byte moves.

The body is whatever is passed in — there is no socket to drain one from, and
`router.is_connected(req)` is false, which is what anything reaching for the
connection should check. Routes that stream (`serve_file`, `respond_stream`)
do need a real one, so they stay in a live-server suite.

## Pages

`pages.templates(dir)` is a `std.template` environment over a directory of
HTML templates, escaped by context; `pages.page(env, name, value)` renders one
as a `text/html` response, or a 500 response naming the template error:

```moon
use moon_router.pages

struct Home { user: string }

let site = pages.templates("templates")
api.get("", |ctx| pages.page(site, "home.html", Home { user: "Ada" }))
```
