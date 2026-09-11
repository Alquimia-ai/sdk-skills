# GenUI — Generative UI

GenUI lets the agent render real, interactive UI in chat — forms, cards, charts, dashboards — instead of describing it in prose. The agent emits a declarative **surface** through a single `render_ui` client tool; the SDK renders it with React components and sends the user's input back so the agent can continue.

**UI is data, never code.** The agent never emits HTML, CSS, or JS. It composes only from an authorized **catalog** of component names, and each name maps to a React component you control.

> Need components the default catalog doesn't have? Read `features/genui-custom-components.md` after this file — it's a five-step recipe for adding any component you want. This file covers the default catalog; that one covers building your own.

---

## 1. The loop

```
user asks something
      │
      ▼
POST /event/infer ──► agent decides UI beats prose, calls render_ui
      │                (the surface IS the tool-call arguments)
      ▼
GET /event/stream (SSE) ──► ClientToolExecution frame carries the surface
      │                     INFERENCE PAUSES here
      ▼
useAlquimia exposes it as genui.surfaces[]
      │
      ├─ display-only surface  → SDK auto-completes { shown: true }
      └─ interactive surface   → rendered; waits for the user
      │
      ▼
POST /event/tool-completion  { action, surfaceId, componentId, data }
      │
      ▼
agent resumes ──► final answer streams back
```

Two things follow from "inference pauses":

- An interactive surface **parks the conversation** until a result is posted. The user must always have an escape hatch — see §5.
- The loop keys off the `ClientToolExecution` stream frame, not off who registered the tool. Client-injected and agent-spec tools behave identically.

---

## 2. Turn it on

Add `genui` to the `useAlquimia` config. That single option registers the `render_ui` tool, injects the prompt clause that teaches the agent when to use UI, and runs the whole surface lifecycle.

```tsx
import { useAlquimia } from '@alquimia-ai/tools/hooks';
import { coreCatalog } from '@alquimia-ai/tools/genui';

const alquimia = useAlquimia({
  assistantId,
  adapter,
  genui: {
    catalog: coreCatalog,   // optional — defaults to coreCatalog
    allow: 'all',           // optional — 'all' | string[] of component names
    source: 'client',       // optional — 'client' (default) | 'agent'
  },
});
```

| Option | Meaning |
|---|---|
| `catalog` | The `CatalogManifest` the agent composes against. Defaults to `coreCatalog` (~43 components). |
| `allow` | Restrict the agent to a subset of catalog components. `'all'` or an array of names. Defaults to all. |
| `source` | Who tells the agent it can draw: `'client'` (default) — the SDK injects the `render_ui` tool + prompt clause on every request, so the agent needs no configuration. `'agent'` — the tool is defined on the agent spec in Studio, so the SDK injects nothing. See `genui-custom-components.md` §7. |

Narrowing `allow` is the cheapest quality lever available. A smaller surface area means the model picks better components and hallucinates less:

```tsx
genui: { allow: ['Card', 'Stack', 'Text', 'TextField', 'Select', 'Button'] }
```

### What the hook returns

When `genui` is set, the hook's return includes a `genui` object (it is `undefined` otherwise):

```typescript
alquimia.genui = {
  surfaces: Array<{
    controlId: string;        // the tool-execution id this surface answers
    surface: A2uiSurface;     // the declarative tree
    afterCount: number;       // message index it is anchored after
    submitted?: boolean;      // already answered — freeze it
    dismissedWith?: string;   // the user answered in words instead
  }>,
  onAgentAction: (controlId: string) => (action: UIAction) => void,
  awaitingUser: boolean,      // a surface is parked waiting on the user
  dismiss: (userMessage?: string) => void,
}
```

---

## 3. Render it — the batteries-included path

`AssistantChat` is a thin composition of the existing `AssistantMessageArea` + `AssistantInput` organisms. It interleaves surfaces into the message flow at the right position, freezes submitted surfaces, and handles the dismissal escape hatch. It does **not** reimplement message rendering, so markdown, streaming, errors, and timestamps all still work.

```tsx
import { AssistantChat } from '@alquimia-ai/ui/components/genui';

<AssistantChat
  alquimia={alquimia}
  conversationId={conversationId}
  placeholder="Ask the agent to build something…"
  suggestions={['Book a meeting', 'Show me last quarter']}
/>
```

| Prop | Purpose |
|---|---|
| `alquimia` | The full `useAlquimia({ genui })` return value. |
| `registry` | Catalog name → React component. Defaults to `coreUiRegistry`. Swap it to use your own components. |
| `conversationId` | Passed through to `handleSubmit`. Optional if the SDK already holds one. |
| `suggestions` | Empty-state prompt chips. |
| `thinkIndicator` | Override the typing indicator. Defaults to animated dots. |

**Requires `@alquimia-ai/ui`.** With custom UI (no Alquimia UI library), use `A2uiRenderer` directly and supply your own registry — see §4 and `genui-custom-components.md`.

---

## 4. Render it — composing yourself

To control the chat shell, render surfaces yourself with `A2uiRenderer`:

```tsx
import { A2uiRenderer, coreUiRegistry } from '@alquimia-ai/ui/components/genui';

{alquimia.genui?.surfaces.map((item) => (
  <div key={item.controlId} className={item.submitted ? 'opacity-70 pointer-events-none' : ''}>
    <A2uiRenderer
      surface={item.surface}
      registry={coreUiRegistry}
      onAgentAction={alquimia.genui.onAgentAction(item.controlId)}
      onLocalAction={(a) => console.log('local action', a)}
    />
  </div>
))}
```

| Prop | Purpose |
|---|---|
| `surface` | The `A2uiSurface` to render. |
| `registry` | `Record<string, React.ComponentType<A2uiNodeProps>>`. |
| `onAgentAction` | Fired for `wantResponse: true` actions — must complete the tool. Use `genui.onAgentAction(controlId)`. |
| `onLocalAction` | Fired for actions without `wantResponse` — purely client-side, no round trip. |

Two rules if you compose yourself:

1. **Freeze submitted surfaces** (`item.submitted`). Otherwise the user can re-submit a surface whose tool call is already closed.
2. **Anchor surfaces by `afterCount`** so they appear in the right place in the message order, not all at the bottom.

---

## 5. Display vs. interactive — and the escape hatch

A surface with no `wantResponse: true` action is **display-only**. The SDK auto-completes it with `{ shown: true }` immediately, and the agent keeps talking. Nothing is required of you.

A surface with an agent-bound action is **interactive**: the inference stays parked until a result is posted. `genui.awaitingUser` is `true` while this is the case.

**The escape hatch is not optional.** If the surface is the only way to answer and the user wants none of the offered options, they are stuck and the conversation never resumes. `AssistantChat` handles this already: while `awaitingUser` is true, typing a message calls `genui.dismiss(text)` instead of sending a normal message. That posts a dismissal result carrying the user's actual words, so the agent can respond to them.

If you compose the shell yourself, you must wire this:

```tsx
const onSubmit = async (e) => {
  e.preventDefault();
  if (!input.trim()) return;
  if (alquimia.genui?.awaitingUser) {
    alquimia.genui.dismiss(input);   // unpark with the user's words
    return;
  }
  await alquimia.handleSubmit(e, undefined, conversationId);
};
```

Note the related input-disabling trap: while a surface is parked, `isLoading` stays `true`. Disable the input on `isLoading && !awaitingUser`, never on `isLoading` alone — the user is not being made to wait, they are being asked.

The dismissal arrives at the agent as action `userDismissedSurface` (exported as `DISMISS_ACTION`) with a `userMessage` field.

---

## 6. What the agent gets back

Agent-bound actions post this `result` to `/event/tool-completion`:

```typescript
interface SurfaceResult {
  action: string;                     // the semantic action name, e.g. "submitBooking"
  surfaceId: string;                  // which surface it came from
  componentId: string;                // WHICH element fired it
  data: Record<string, unknown>;      // the surface's collected data model
  userMessage?: string;               // only on dismissal
}
```

`componentId` matters more than it looks. When several elements share one action name — a grid of cards that all emit `selectItem` — it is the only thing telling the agent *which* one the user clicked. Without it the agent has to ask "which one did you pick?", which reads as a broken assistant.

---

## 7. The default catalog

`coreCatalog` (id `https://alquimia.ai/catalogs/core/v1/catalog.json`) ships ~43 components:

| Group | Components |
|---|---|
| **Layout** | `Stack` `Grid` `Row` `Column` `Divider` `Spacer` `Card` `Accordion` `Tabs` |
| **Content** | `Text` `Heading` `Markdown` `Image` `Icon` `Badge` `Avatar` `Alert` `Stat` `KeyValue` `List` `Quote` |
| **Inputs** | `TextField` `TextArea` `NumberField` `Select` `MultiSelect` `Combobox` `Checkbox` `CheckboxGroup` `Radio` `Switch` `Slider` `DatePicker` `DateRange` `FileUpload` `Rating` |
| **Data** | `Table` `Chart` `Progress` `Timeline` |
| **Actions** | `Button` `ButtonGroup` `Link` |

Every component also accepts the shared **style vocabulary** — semantic intent, never CSS:

```typescript
tone:     'default' | 'muted' | 'primary' | 'success' | 'warning' | 'danger'
size:     'sm' | 'md' | 'lg'
emphasis: 'low' | 'medium' | 'high'
align:    'start' | 'center' | 'end'
density:  'compact' | 'comfortable'
```

The renderer maps these to your design tokens, so the agent can express "this is a warning, shown prominently" without ever knowing your brand colors.

---

## 8. Surface anatomy

A surface is a **flat adjacency list**, not a nested tree — the agent emits an array of components that reference each other by id. The root is the component with `id: "root"`.

```jsonc
{
  "surfaceId": "book-1",
  "catalogId": "https://alquimia.ai/catalogs/core/v1/catalog.json",
  "components": [
    { "id": "root", "component": "Card", "title": "Book a demo", "child": "form" },
    { "id": "form", "component": "Stack", "gap": "md", "children": ["name", "when", "go"] },
    { "id": "name", "component": "TextField", "label": "Your name", "value": { "path": "/name" } },
    { "id": "when", "component": "DatePicker", "label": "Date",     "value": { "path": "/date" } },
    { "id": "go",   "component": "Button", "text": "Book",
      "action": { "name": "submitBooking", "wantResponse": true } }
  ],
  "dataModel": { "name": "", "date": null }
}
```

Key points:

- **Containers** use `children: string[]`; single-child components (Button, Card) use `child: string`.
- **Binding** is a `DynamicString`: any prop value may be a literal *or* `{ path: "/json/pointer" }` referencing `dataModel`. Bound inputs get a working `onChange` that writes back to the model; that model is what ships to the agent as `data`.
- **Actions are semantic names, never URLs.** `assertSafeActionName` rejects anything scheme- or protocol-relative.
- `wantResponse: true` → agent-bound (round trip). Omitted → local (client-side only).
- `catalogId` is stamped on every surface by the SDK if the agent omits it.

---

## 9. Failure behavior

GenUI degrades instead of crashing — a model will eventually emit something wrong, and a blank chat is a worse outcome than a partial render.

| Problem | Result |
|---|---|
| Unknown component name | Renders `[unsupported component: X]` text fallback; siblings still render |
| Invalid props on a node | Warning; the surface still renders |
| No resolvable root | Fatal — surface is not rendered |
| Cyclic `children` references | Cycle guard stops the recursion |

`validateSurface(input, catalog)` returns `{ ok, errors, warnings }` — errors are fatal, warnings degrade. The hook runs this for you before a surface reaches the renderer.

---

## 10. Checklist

- [ ] `genui` passed to `useAlquimia`
- [ ] Surfaces rendered via `AssistantChat` *or* `A2uiRenderer` anchored by `afterCount`
- [ ] Submitted surfaces frozen (`item.submitted`)
- [ ] Dismissal wired — typing while `awaitingUser` calls `genui.dismiss(input)`
- [ ] Input disabled on `isLoading && !awaitingUser`, not `isLoading`
- [ ] `allow` narrowed to the components this agent actually needs
