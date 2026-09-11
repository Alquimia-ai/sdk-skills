---
name: alquimia-sdk
description: Use when building an application that communicates with the Alquimia AI runtime. Covers SDK initialization with adapters, useAlquimia hook, server proxy setup, GenUI (generative UI — agents that render real interactive components in chat, with the default catalog or your own domain components), and optional features (TTS, STT, user tools, attachments, sidebar, worklog). Supports Next.js, SPA + server, or direct backend calls. Works with the full Alquimia UI library or with the user's own custom components.
---

# Alquimia SDK — Application Builder

## Step 1: Ask these questions before writing any code

Present the options as a numbered/lettered menu so the user can reply with just the numbers/letters.

---

**Question 1 — Server setup:**

> How does your app talk to the Alquimia backend?
> ```
> 1) Next.js App Router  — built-in route handlers proxy requests
> 2) SPA + separate server (Express / Hono / Fastify / Bun / etc.)
> 3) SPA (Vite / CRA / etc.) — no existing server
> ```
> Reply with the number.
>
> Option 3 uses a lightweight ~30-line Hono proxy for the SSE stream (EventSource can't send auth headers). Infer and attachments call the backend directly.

---

**Question 2 — UI approach:**

> How do you want to build the UI?
> ```
> A) Full Alquimia UI  — @alquimia-ai/ui components (chat widget, message area, input, themes)
> B) Custom UI         — only @alquimia-ai/tools for runtime communication; you build your own components
> ```
> Reply with the letter.

---

**Question 3 — Optional features:**

> Which optional features do you want? (reply with the letters, e.g. `A C`)
> ```
> A) Text-to-Speech   — ElevenLabs / Orpheus / Alquimia (which provider?)
> B) Speech-to-Text
> C) User Tools       — structured data sent with messages
> D) Attachments      — file upload / drag-and-drop
> E) Sidebar          — reasoning / thinkings panel (requires Full Alquimia UI)
> F) GenUI           — the agent renders real interactive UI in chat (forms, cards, charts)
> G) Worklog         — normalized agent execution trace ("show your work")
> ```
> Leave blank for none.
>
> If F, also ask:
> ```
> F1) Default catalog  — the agent composes from ~43 shipped components
> F2) Your components  — your own domain catalog (ProductCard, PolicyCard, ShipmentTracker, ...)
> ```

---

Wait for the user's replies before proceeding.

---

## Step 2: Read the required files for this task

All file paths are relative to the skill root.

### Always read (every implementation):
| File | Purpose |
|------|---------|
| `setup/installation.md` | npm packages, Tailwind config (if using UI), CSS/theme setup |
| `setup/backend-routes.md` | Proxy setup — read the section matching Q1 |
| `core/sdk-init.md` | Adapter creation, `useAlquimia` hook config, conversation ID, custom UI wiring |

**Q1 answer → section in `backend-routes.md`:**
- `1` → Next.js (`createNextJsRouteHandlers`)
- `2` → Any server (`createAlquimiaProxyHandler`)
- `3` → SPA with lightweight stream proxy (`createFetchAdapter` + Hono SSE proxy)

### Read only if Q2 = A (Full Alquimia UI):
| File | Purpose |
|------|---------|
| `core/chat-component.md` | `AssistantMessageArea` / `AssistantInput` composition and props |
| `styles/style-guide.md` | CSS themes, variables, and `alq--` utility classes |

### Read for selected features (Q3):
| Feature | File |
|---------|------|
| TTS or STT | `features/tts-stt.md` |
| User Tools | `features/tools.md` |
| Attachments | `features/attachments.md` |
| Sidebar | `features/sidebar.md` |
| GenUI | `features/genui.md` — always |
| GenUI with own components (F2) | `features/genui-custom-components.md` — **plus** `features/genui.md` |
| Worklog | `features/worklog.md` |

### Reference (import paths, types, env vars):
`reference/imports.md`

---

## SDK Overview

The Alquimia SDK has two packages:

- **`@alquimia-ai/tools`** — Core runtime: `AlquimiaSDK` class, `useAlquimia` hook, adapters (`createNextJsAdapter`, `createFetchAdapter`), server proxy (`createAlquimiaProxyHandler`, `createNextJsRouteHandlers`), providers (TTS/STT, image gen, characterization, ratings, logging), GenUI protocol (`/genui` — catalog, surface types, validation), execution trace (`/worklog`).
- **`@alquimia-ai/ui`** *(optional)* — React component library: `AssistantMessageArea`, `AssistantInput`, `SpeechToText`, `Whisper`, `ThinkIndicator`, `Loader`, `Drawer`, `AlquimiaUIProvider`, atoms (Button, Dialog, Tabs…), and the GenUI renderer (`/components/genui` — `A2uiRenderer`, `coreUiRegistry`, `AssistantChat`). Based on shadcn, ships with theme CSS files.

### How it connects

The SDK uses an **adapter** to resolve request URLs. Two built-in adapters:

- `createNextJsAdapter()` — resolves to relative paths (`/api/chat/...`). For Next.js apps where proxy routes live in the same app.
- `createFetchAdapter({ baseUrl })` — resolves to any HTTP server. For SPA + separate server setups.

You can also implement a custom `AlquimiaAdapter` (e.g. to split infer/stream across different servers).

**Important:** The SSE stream endpoint uses `EventSource`, which **cannot send custom headers**. If the backend or its gateway requires `Authorization` on all requests, the stream will fail in pure client mode. A server-side proxy (even a tiny one) is needed to inject the auth header on the stream. See Mode 3 in `backend-routes.md`.

The `useAlquimia` hook creates the SDK internally and returns the full chat state (messages, input, loading flags, send/cancel actions, attachments, audio recording). When the user wants custom UI, they wire these return values to their own components. When using Full Alquimia UI, they pass them to `AssistantMessageArea` and `AssistantInput`.

### GenUI in one paragraph

With `useAlquimia({ genui })`, the agent can answer with **real interactive UI** instead of prose. It calls a `render_ui` client tool whose arguments are a declarative component tree; the SDK validates that tree against a **catalog**, renders it with React components you control, and posts the user's input back so the agent continues. The agent never emits markup — only component names and props you authorized. Point the catalog at your own domain components and the same agent composes your product cards, policy comparisons, or shipment trackers. See `features/genui.md`, then `features/genui-custom-components.md`.
