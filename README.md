# Alquimia Skills

Agent skills for building applications on the [Alquimia AI](https://alquimia.ai) runtime.

These skills guide AI coding agents (Claude Code and other skill-aware tools) through wiring an application up to the Alquimia AI runtime — from SDK initialization to a working chat UI — using current, correct APIs instead of guesswork.

## Installation

```bash
npx skills add Alquimia-ai/sdk-skills
```

Or via Claude Code plugin:

```
/plugin marketplace add Alquimia-ai/sdk-skills
/plugin install alquimia-sdk@alquimia-skills
```

## Available Skills

### alquimia-sdk

Builds an application that communicates with the Alquimia AI runtime. The skill interviews you about your setup, reads only the files relevant to your answers, and then implements the integration.

**Triggers when you:**

- Want to build a chat or assistant app on the Alquimia runtime
- Ask to set up `@alquimia-ai/tools` or `@alquimia-ai/ui`
- Need a server proxy for the Alquimia SSE stream
- Add Alquimia features like TTS, STT, user tools, attachments, or a reasoning sidebar
- Want the agent to render real interactive UI in chat (**GenUI**) — with the shipped catalog or your own domain components

**What it does:**

The skill starts by asking three questions:

1. **Server setup** — Next.js App Router, SPA + separate server, or SPA with a lightweight stream proxy.
2. **UI approach** — full `@alquimia-ai/ui` component library, or your own custom components driven by `@alquimia-ai/tools`.
3. **Optional features** — Text-to-Speech, Speech-to-Text, user tools, attachments, sidebar, GenUI, worklog.

Based on your answers it covers:

- **SDK initialization** — adapter creation (`createNextJsAdapter`, `createFetchAdapter`, or a custom `AlquimiaAdapter`) and the `useAlquimia` hook.
- **Server proxy setup** — `createNextJsRouteHandlers`, `createAlquimiaProxyHandler`, or a ~30-line Hono SSE proxy, since `EventSource` can't send auth headers from the browser.
- **Chat UI** — composing `AssistantMessageArea` / `AssistantInput`, theming, and `alq--` utility classes — or wiring the hook's return values into your own components.
- **Optional features** — TTS/STT providers, structured user tools, file attachments, the reasoning/thinkings sidebar, and the agent execution trace (worklog).
- **GenUI** — generative UI: the agent answers with real interactive components (forms, cards, charts, trackers) instead of prose. Covers the `render_ui` loop, the shipped ~43-component catalog, and **authoring your own catalog and components** so the agent composes your domain — product cards, policy comparisons, shipment trackers — in your design system.

## The SDK

The Alquimia SDK has two packages:

- **`@alquimia-ai/tools`** — Core runtime: `AlquimiaSDK` class, `useAlquimia` hook, adapters, server proxy handlers, providers (TTS/STT, image gen, characterization, ratings, logging), the GenUI protocol (`/genui`), and the worklog interpreter (`/worklog`).
- **`@alquimia-ai/ui`** *(optional)* — React component library based on shadcn: `AssistantMessageArea`, `AssistantInput`, `SpeechToText`, `Whisper`, `ThinkIndicator`, `Drawer`, `AlquimiaUIProvider`, atoms, theme CSS, and the GenUI renderer (`A2uiRenderer`, `coreUiRegistry`, `AssistantChat`).

## Contributing

To add or update a skill:

1. Create or edit `skills/<skill-name>/SKILL.md` plus supporting files (`setup/`, `core/`, `features/`, `reference/`, `styles/`).
2. Register the skill in `.claude-plugin/marketplace.json`.
3. Open a PR.

## License

Apache 2.0
