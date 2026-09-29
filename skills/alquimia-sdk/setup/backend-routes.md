# Backend Configuration

## How the SDK makes requests

The SDK uses an **adapter** to resolve the endpoint URLs, then calls them from the browser:

| Call | Method | URL resolved by adapter | Needed for |
|------|--------|------------------------|------------|
| Chat infer | POST (axios) | `adapter.resolveInferUrl(assistantId)` | always |
| Stream | GET SSE (EventSource) | `adapter.resolveStreamUrl(streamId)` | always |
| Blob upload | POST (axios) | `adapter.resolveBlobUploadUrl()` | attachments, audio input |
| Tool completion | POST (axios) | `adapter.resolveToolCompletionUrl()` | client tools, **GenUI** |
| Tool approval | POST (axios) | `adapter.resolveToolApprovalUrl()` | tool calls that need human approval (tools ≥ 2.8.0) |

**If you are implementing GenUI or client tools, the tool-completion route is not optional.**
The agent parks the task waiting for the browser's answer; without that route the surface
renders, the user submits, and the run stalls. Both built-in adapters resolve the URL by
default — it is the *server* route that has to exist.

**The same holds for tool approval.** If the agent can hit a tool the runtime classifies as
needing approval (`destructive` operations always do), the run parks on a
`HumanApprovalRequired` until `POST /event/tool-approval` answers it. See
`features/tool-approval.md`.

The SDK **never adds an `Authorization` header** itself. In proxied mode (Mode 1 & 2), your server injects the API key.

**Critical limitation:** The **stream** endpoint uses `EventSource` (SSE), which **cannot send custom headers**. The `getHeaders()` adapter option works for infer (axios POST) and blob upload (axios POST), but is ignored for streaming. If your Alquimia backend (or a gateway/ingress in front of it) requires an `Authorization` header on all requests, the stream will fail with `Not authenticated` even with `getHeaders` configured. A server-side proxy is the only workaround — see Mode 2 or the lightweight proxy in Mode 3.

> **v2.1 note:** Attachments now post to `/context/blob/upload` (no
> stream_id/attachment_id in the path). The SDK attaches kebab-case
> identity headers (`assistant-id`, `session-id`, `user-id`,
> `task-id`) automatically; the proxy forwards them.

---

## Mode 1: Next.js App Router

Use `createNextJsRouteHandlers()` — each route file is ~5 lines.

### Client-side adapter

```typescript
import { createNextJsAdapter } from '@alquimia-ai/tools/adapters/next';

const adapter = createNextJsAdapter({
  inferRoute: '/api/chat',                     // default
  streamRoute: '/api/stream',                  // default
  blobUploadRoute: '/api/blob/upload',         // default
  toolCompletionRoute: '/api/tool-completion', // default
  toolApprovalRoute: '/api/tool-approval',     // default
});
```

All options are optional — defaults match the route paths below.

### Route handlers

```typescript
// app/api/chat/[...path]/route.ts
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const POST = handlers.handleInfer;
```

```typescript
// app/api/stream/[...path]/route.ts
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const GET = handlers.handleStream;
```

```typescript
// app/api/blob/upload/route.ts  (single endpoint, no [...path])
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const POST = handlers.handleBlobUpload;
```

```typescript
// app/api/tool-completion/route.ts  (client tools + GenUI; also no [...path])
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const POST = handlers.handleToolCompletion;
```

```typescript
// app/api/tool-approval/route.ts  (human approval of tool calls; no [...path])
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const POST = handlers.handleToolApproval;
```

None of the last three takes a `[...path]` segment: identity travels in headers, and the
pending tool call or approval is correlated by `control_id` in the body.

`createNextJsRouteHandlers` wraps the framework-agnostic `createAlquimiaProxyHandler` with Next.js App Router context param extraction. Optional config: `inferRoute` (default `'event/infer'`), `streamRoute` (default `'event/stream'`), `blobUploadRoute` (default `'context/blob/upload'`).

### Environment variables

```bash
# .env.local
ASSISTANT_BASEURL=https://your-backend.com
ALQUIMIA_ASSISTANT_API_KEY=your-secret-key
```

---

## Mode 2: Any server (Express / Hono / Fastify / Bun / Deno)

Use `createAlquimiaProxyHandler()` — framework-agnostic, built on the Web Fetch API (`Request`/`Response`).

### Client-side adapter

```typescript
import { createFetchAdapter } from '@alquimia-ai/tools/adapters/fetch';

const adapter = createFetchAdapter({
  baseUrl: 'https://my-api.example.com', // your server's URL — required
  inferPath: '/chat',                     // default
  streamPath: '/stream',                  // default
  blobUploadPath: '/blob/upload',         // default
  toolCompletionPath: '/tool-completion', // default
  toolApprovalPath: '/tool-approval',     // default
});
```

### Server-side proxy handler

```typescript
import { createAlquimiaProxyHandler } from '@alquimia-ai/tools/proxy';

const handler = createAlquimiaProxyHandler({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
  // Optional overrides (defaults match the Alquimia backend):
  inferRoute: 'event/infer',
  streamRoute: 'event/stream',
  blobUploadRoute: 'context/blob/upload',
  toolCompletionRoute: 'event/tool-completion',
  toolApprovalRoute: 'event/tool-approval',
});

// handler.handleInfer(request, pathSuffix)  → Response
// handler.handleStream(request, streamId)   → Response
// handler.handleBlobUpload(request)         → Response
// handler.handleToolCompletion(request)     → Response
// handler.handleToolApproval(request)       → Response
```

Tool-completion and tool-approval pass the runtime's status and body through untouched: the
approval 409s differ only in their `detail`, and the SDK reads it to tell them apart.

The handler accepts Web `Request` objects and returns Web `Response` objects — wire it into your framework's routing.

### Express example

```typescript
import express from 'express';
import { createAlquimiaProxyHandler } from '@alquimia-ai/tools/proxy';

const app = express();
const handler = createAlquimiaProxyHandler({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

app.post('/chat/*', async (req, res) => {
  const pathSuffix = req.params[0];
  const webReq = new Request(`${req.protocol}://${req.get('host')}${req.originalUrl}`, {
    method: 'POST',
    headers: Object.fromEntries(Object.entries(req.headers).filter(([, v]) => v != null) as [string, string][]),
    body: JSON.stringify(req.body),
  });
  const webRes = await handler.handleInfer(webReq, pathSuffix);
  res.status(webRes.status).json(await webRes.json());
});

app.get('/stream/:streamId', async (req, res) => {
  const webReq = new Request(`${req.protocol}://${req.get('host')}${req.originalUrl}`, {
    headers: Object.fromEntries(Object.entries(req.headers).filter(([, v]) => v != null) as [string, string][]),
  });
  const webRes = await handler.handleStream(webReq, req.params.streamId);
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  const reader = webRes.body!.getReader();
  const pump = async () => {
    const { done, value } = await reader.read();
    if (done) { res.end(); return; }
    res.write(value);
    pump();
  };
  pump();
});

app.post('/blob/upload', async (req, res) => {
  const webReq = new Request(`${req.protocol}://${req.get('host')}${req.originalUrl}`, {
    method: 'POST',
    headers: Object.fromEntries(Object.entries(req.headers).filter(([, v]) => v != null) as [string, string][]),
    body: req,
    duplex: 'half',
  } as RequestInit);
  const webRes = await handler.handleBlobUpload(webReq);
  res.status(webRes.status).json(await webRes.json());
});
```

### Hono example

```typescript
import { Hono } from 'hono';
import { createAlquimiaProxyHandler } from '@alquimia-ai/tools/proxy';

const app = new Hono();
const handler = createAlquimiaProxyHandler({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

// NOTE: Hono v4's `c.req.param('*')` does NOT return the wildcard capture — it
// yields an empty string, so the assistant/stream id is dropped and the backend
// answers 404. Derive the suffix from `c.req.path` instead (it excludes the
// query string, which the handler re-attaches from the request's searchParams).
app.post('/chat/*', (c) =>
  handler.handleInfer(c.req.raw, c.req.path.replace(/^\/chat\//, '')),
);
app.get('/stream/*', (c) =>
  handler.handleStream(c.req.raw, c.req.path.replace(/^\/stream\//, '')),
);
app.post('/blob/upload', (c) => handler.handleBlobUpload(c.req.raw));
app.post('/tool-completion', (c) => handler.handleToolCompletion(c.req.raw));
app.post('/tool-approval', (c) => handler.handleToolApproval(c.req.raw));
```

> Named params (`/stream/:streamId`) work fine in Hono v4 — only the `*`
> wildcard is broken. The path-derive form above is used consistently and also
> handles ids that contain slashes.

### Environment variables

```bash
ASSISTANT_BASEURL=https://your-backend.com
ALQUIMIA_ASSISTANT_API_KEY=your-secret-key
```

---

## Mode 3: SPA with lightweight stream proxy

Pure direct mode (no server at all) **does not work** when the Alquimia backend or its gateway requires auth headers — `EventSource` (SSE) cannot send custom headers, so the stream endpoint will always fail with `Not authenticated`.

The recommended approach for SPA apps: use `createFetchAdapter` with a small local proxy that only handles the stream. The infer and blob-upload endpoints use axios (which can send headers), so they can call the backend directly.

### Client-side adapter

```typescript
import { createFetchAdapter } from '@alquimia-ai/tools/adapters/fetch';

const adapter = createFetchAdapter({
  baseUrl: import.meta.env.VITE_ASSISTANT_BASEURL,
  getHeaders: () => ({
    Authorization: `Bearer ${import.meta.env.VITE_ALQUIMIA_API_KEY}`,
  }),
  inferPath: '/event/infer',
  streamPath: '/event/stream',
  blobUploadPath: '/context/blob/upload',
  toolCompletionPath: '/event/tool-completion',
  toolApprovalPath: '/event/tool-approval',
});
```

If the backend requires auth on the stream endpoint, override `streamPath` to point at your local proxy instead:

```typescript
const adapter = createFetchAdapter({
  baseUrl: import.meta.env.VITE_ASSISTANT_BASEURL,
  getHeaders: () => ({
    Authorization: `Bearer ${import.meta.env.VITE_ALQUIMIA_API_KEY}`,
  }),
  inferPath: '/event/infer',
  blobUploadPath: '/context/blob/upload',
});

// Override stream URL resolution to use the local proxy
// Or use a custom adapter (see below)
```

### Lightweight Hono stream proxy (~30 lines)

Runs alongside Vite on a different port. Only proxies the SSE stream — infer and attachments go direct.

```typescript
// server/proxy.ts
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { serve } from '@hono/node-server';

const app = new Hono();
app.use('/*', cors());

const BACKEND = process.env.ASSISTANT_BASEURL!;
const API_KEY = process.env.ALQUIMIA_API_KEY!;

app.get('/event/stream/:streamId', async (c) => {
  const streamId = c.req.param('streamId');
  const upstream = await fetch(`${BACKEND}/event/stream/${streamId}`, {
    headers: { Authorization: `Bearer ${API_KEY}` },
  });

  c.header('Content-Type', 'text/event-stream');
  c.header('Cache-Control', 'no-cache');
  c.header('Connection', 'keep-alive');
  return new Response(upstream.body, {
    status: upstream.status,
    headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache' },
  });
});

serve({ fetch: app.fetch, port: 3001 });
console.log('Stream proxy running on http://localhost:3001');
```

```bash
npm install hono @hono/node-server
```

### Custom adapter for mixed direct + proxy

Point infer/blob upload at the backend directly, stream at the local proxy:

```typescript
import type { AlquimiaAdapter } from '@alquimia-ai/tools/adapters';

const BACKEND = import.meta.env.VITE_ASSISTANT_BASEURL;
const PROXY = 'http://localhost:3001';
const API_KEY = import.meta.env.VITE_ALQUIMIA_API_KEY;

const adapter: AlquimiaAdapter = {
  resolveInferUrl(assistantId) {
    return `${BACKEND}/event/infer/${assistantId}`;
  },
  resolveStreamUrl(streamId) {
    return `${PROXY}/event/stream/${streamId}`;
  },
  resolveBlobUploadUrl() {
    return `${BACKEND}/context/blob/upload`;
  },
  resolveToolCompletionUrl() {
    return `${BACKEND}/event/tool-completion`;
  },
  resolveToolApprovalUrl() {
    return `${BACKEND}/event/tool-approval`;
  },
  getHeaders() {
    return { Authorization: `Bearer ${API_KEY}` };
  },
};
```

### Running both Vite + proxy

Add a script to your `package.json`:

```json
{
  "scripts": {
    "dev": "concurrently \"vite\" \"tsx server/proxy.ts\"",
    "dev:vite": "vite",
    "dev:proxy": "tsx server/proxy.ts"
  }
}
```

```bash
npm install -D concurrently tsx
```

> **Gotcha:** the proxy reads `ASSISTANT_BASEURL` / `ALQUIMIA_ASSISTANT_API_KEY`
> only at startup (via `import 'dotenv/config'`). After editing `.env`, **restart
> `npm run dev`** — a proxy left running from before the edit keeps serving the
> old config (and a stale one holding port 3001 will make a fresh `tsx` exit with
> `EADDRINUSE`, so you may not notice it's still up). Use `tsx watch server/proxy.ts`
> if you want the proxy to auto-restart on changes.

**If your backend has no auth gateway** (e.g. internal/dev environments), you can skip the proxy entirely and use `createFetchAdapter` with direct paths. The `getHeaders` will work for infer and blob upload; the stream will connect without auth.

---

## Custom adapter

If neither built-in adapter fits, implement the `AlquimiaAdapter` interface:

```typescript
import type { AlquimiaAdapter } from '@alquimia-ai/tools/adapters';

const adapter: AlquimiaAdapter = {
  resolveInferUrl(assistantId) { return `https://.../${assistantId}`; },
  resolveStreamUrl(streamId)   { return `https://.../${streamId}`; },
  resolveBlobUploadUrl()       { return `https://.../context/blob/upload`; },
  resolveToolCompletionUrl()   { return `https://.../event/tool-completion`; }, // client tools, GenUI
  resolveToolApprovalUrl()     { return `https://.../event/tool-approval`; },   // human approval
  getHeaders() { return { Authorization: 'Bearer ...' }; },
};
```
