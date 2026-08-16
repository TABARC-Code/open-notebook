---
name: sse-endpoints
description: Add or review a Server-Sent Events streaming endpoint end-to-end (FastAPI backend, Next.js proxy route, frontend consumer). Use when a feature needs live progress or token-by-token updates pushed to the UI — e.g. the Live Front-End Updates roadmap item — or when reviewing/debugging an existing SSE endpoint (stalled stream, buffering, dropped events). Dewey: 005.14.
metadata:
  author: TABARC-Code
  dewey_decimal_code: '005.14'
---

# SSE Endpoints

Open Notebook already has two working SSE endpoints — source chat
(`api/routers/source_chat.py`) and search/ask
(`api/routers/search.py`) — and a shared frontend proxy for them
(`frontend/src/app/api/_sse-proxy.ts`). This skill documents that existing
pattern so a new streaming endpoint (Live Front-End Updates, or anything
else that needs to push progress rather than poll for it) matches it
instead of reinventing a slightly different one.

There are three layers. All three need to agree, or the stream stalls or
buffers somewhere in the middle without an obvious error.

## Layer 1 — FastAPI: the event generator

An `async def` generator that `yield`s pre-formatted SSE lines, not a
route handler itself:

```python
async def stream_x_response(...) -> AsyncGenerator[str, None]:
    try:
        event = {"type": "some_event", "content": ...}
        yield f"data: {json.dumps(event)}\n\n"
        # ... more yields as work progresses ...
        yield f"data: {json.dumps({'type': 'complete'})}\n\n"
    except Exception as e:
        error_event = {"type": "error", "message": str(e)}
        yield f"data: {json.dumps(error_event)}\n\n"
```

Conventions both existing endpoints follow, worth keeping:

- Every event is a JSON object with a `type` key (`user_message`,
  `ai_message`, `context_indicators`, `complete`, `error`, ...) — the
  frontend switches on this, so an untyped or inconsistently-shaped event
  is a silent frontend bug, not a backend error.
- Always emit a terminal `{"type": "complete"}` event. The frontend needs
  a signal to stop listening; relying on the connection simply closing is
  indistinguishable from a network drop.
- Catch exceptions **inside the generator** and yield an `error` event
  rather than letting the exception propagate — by the time you're mid-
  stream, the HTTP status code is already sent as 200; an uncaught
  exception here doesn't turn into a clean 500, it truncates the stream.

**The one that actually bites**: if anything in the generator calls a
**synchronous, blocking** function (a sync LangGraph `.invoke()`, a sync
DB driver call, anything that isn't natively `async`), wrap it in
`asyncio.to_thread(...)`. Without that, the blocking call freezes the
whole event loop — not just this request. Every already-yielded SSE event
sitting in the buffer can't flush, and every *other* concurrent request
stalls until the blocking call returns. This is exactly the shape of bug
that passes a single-user manual test and falls over under any real
concurrency; see the comment in `source_chat.py` around the
`asyncio.to_thread` call for the fuller explanation, and add a
concurrency probe to the release skill's test matrix if you add a new one
of these.

## Layer 2 — FastAPI: the route

```python
return StreamingResponse(
    stream_x_response(...),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "Connection": "keep-alive",
        "X-Accel-Buffering": "no",
    },
)
```

`X-Accel-Buffering: no` matters specifically if this ever sits behind
nginx (it does, in the release image's reverse-proxy setup) — without it
nginx buffers the whole response before forwarding, which defeats
streaming entirely while looking identical to a working endpoint in local
dev where nginx isn't in the path.

## Layer 3 — Next.js: the proxy route

Don't write a new proxy from scratch — use `sseProxy()` from
`frontend/src/app/api/_sse-proxy.ts`:

```typescript
import { sseProxy } from '../../_sse-proxy'

export async function POST(req: NextRequest) {
  return sseProxy(req, '/api/your-new-endpoint')
}
```

It forwards the request to the FastAPI backend with
`Accept: text/event-stream`, passes the auth header through if present,
and re-emits the same three headers on the response back to the browser.
If you hand-roll this instead of reusing it, you will eventually drop one
of `Cache-Control: no-transform` or `X-Accel-Buffering: no` and get an
endpoint that streams in dev and buffers in the deployed image — this
class of bug is exactly why the release skill's probe library says to
verify SSE endpoints "stream progressively (first byte ≪ total time via
`curl -N -w`)" rather than trusting that a 200 response means streaming
worked.

## Layer 4 — Frontend: consuming it

`frontend/src/lib/api/source-chat.ts` reads the response with `fetch()` +
the response body's `ReadableStream`, not the browser `EventSource` API —
`EventSource` only supports GET, and these endpoints are POST (the
message body needs to go somewhere). Follow that pattern, not
`EventSource`, for the same reason.

## Testing

- `curl -N -w '%{time_starttransfer}\n' <endpoint>` — the `-N` disables
  curl's own buffering; `time_starttransfer` should be much smaller than
  total request time. If they're close together, something upstream
  (nginx, a missing header, an un-awaited blocking call) is buffering the
  whole response before sending it.
- Add the endpoint to the release skill's smoke-e2e agent
  (`.claude/agents/smoke-e2e.md` / `.codex/agents/smoke-e2e.toml`) and its
  probe library (`.agents/skills/release/test-matrix.md`) if it's part of
  a feature going into a release — SSE endpoints are exactly the kind of
  thing that passes a curl-once check and still buffers in production
  behind nginx.
