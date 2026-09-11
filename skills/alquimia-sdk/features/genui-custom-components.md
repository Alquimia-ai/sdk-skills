# GenUI — Building Your Own Components

Read `features/genui.md` first. This file covers making the agent compose **your** components — whatever they are, in your design system.

This is the main reason GenUI is catalog-based. The agent does not know what any of your concepts are — it knows the component names *you* authorized and the props *you* declared. Whatever you can name and give a schema to, the agent can compose. §4 is the recipe; repeat it once per component.

---

## 1. Pick a level

| Level | You want | Cost |
|---|---|---|
| **1 — Swap the registry** | The default catalog, rendered with your components | Minutes. Keeps ~43 components working. |
| **2 — Author a catalog** | Components you define, named and shaped by you | The real work. Full control of what the agent may emit. |
| **3 — Move the tool to the agent spec** | The tool + clause defined in Studio, not injected by the client | Config, not code. Composes with 1 or 2. |

Levels 1 and 2 combine: most catalogs reuse the core layout components and add their own on top.

---

## 2. Level 1 — our catalog, your components

The catalog is a contract of *names and props*. The registry decides what actually renders. To put the default catalog on your design system, override the entries you care about:

```tsx
import { useAlquimia } from '@alquimia-ai/tools/hooks';
import { coreCatalog } from '@alquimia-ai/tools/genui';
import { AssistantChat, coreUiRegistry, type A2uiNodeProps } from '@alquimia-ai/ui/components/genui';

const myRegistry: Record<string, React.ComponentType<A2uiNodeProps>> = {
  ...coreUiRegistry,        // start from the shipped shadcn binding
  Button: MyBrandButton,    // override only what differs
  Card: MyBrandCard,
};

const alquimia = useAlquimia({ assistantId, adapter, genui: { catalog: coreCatalog } });

<AssistantChat alquimia={alquimia} registry={myRegistry} conversationId={id} />;
```

You can also build a registry from scratch — with a fully custom UI (no `@alquimia-ai/ui`), supply every key yourself and render with `A2uiRenderer`.

### The component contract

Every component in a registry receives `A2uiNodeProps`:

```typescript
interface A2uiNodeProps {
  node: A2uiComponent;                    // the raw node: id, component, and all props
  value?: unknown;                        // resolved value (binding already dereferenced)
  onChange?: (value: unknown) => void;    // present only when the node is bound to the data model
  onAction?: (action: SurfaceAction) => void;  // present only when the node declares an action
  children?: React.ReactNode;             // already-rendered children
}
```

Reading it:

```tsx
function MyBrandButton({ node, onAction, children }: A2uiNodeProps) {
  return (
    <button
      className={toneToClass(node.tone as string)}
      onClick={() => node.action && onAction?.(node.action)}
    >
      {(node.text as string) ?? children}
    </button>
  );
}

function MyTextField({ node, value, onChange }: A2uiNodeProps) {
  return (
    <label>
      {node.label as string}
      <input
        value={(value as string) ?? ''}
        placeholder={node.placeholder as string}
        onChange={(e) => onChange?.(e.target.value)}
      />
    </label>
  );
}
```

Four things to respect:

1. **Presentational props come off `node`** — they are whatever your schema declared.
2. **`value` is already resolved.** The renderer dereferences `{ path: "/ptr" }` against the data model before you see it. Never resolve pointers yourself.
3. **`onChange` writes back to the data model** — that model is exactly what the agent receives as `data`. An input without `onChange` is unbound; its value will not reach the agent.
4. **`onAction` is only present when the node declares an action.** Fire it with `node.action`; don't invent action names in the component.

---

## 3. Level 2 — author your own catalog

A catalog is a `CatalogManifest`: an id plus component name → zod prop schema.

```typescript
import { z } from 'zod';
import { styleSchemaShape, type CatalogManifest } from '@alquimia-ai/tools/genui';

export const storeCatalog: CatalogManifest = {
  id: 'https://acme.example/catalogs/store/v1/catalog.json',   // versioned URI
  components: {
    ItemCard: {
      name: 'ItemCard',
      schema: z.object({
        ...styleSchemaShape,                      // inherit tone/size/emphasis/align/density
        itemId: z.string(),
        title: z.string(),
        subtitle: z.string().optional(),
        state: z.enum(['active', 'pending', 'blocked']),
      }).passthrough(),
      allowedActions: ['selectItem'],
    },
  },
};
```

| Field | Purpose |
|---|---|
| `id` | Versioned URI. The version lives in the path and is stamped on every surface. |
| `components[name].schema` | Zod schema for **presentational props only**. `id`, `component`, `children`, `child`, and `action` are structural — never declare them. |
| `components[name].allowedActions` | Semantic action names this component may emit. |

Spread `styleSchemaShape` into each schema so your components inherit the shared style vocabulary. Use `.passthrough()` so an extra prop from the model degrades to a warning rather than failing the surface.

### Designing components an agent can actually use

This is where catalogs succeed or fail. You are not designing for a developer reading docs — you are designing for a model that sees only component names and a JSON Schema.

**Make props semantic, never presentational.** The model must express *intent*; your renderer owns appearance.

```typescript
// BAD — the model picks your styling, and picks it badly
z.object({ color: z.string(), className: z.string(), width: z.number() })

// GOOD — the model expresses meaning; the renderer maps it to tokens
z.object({ tone: z.enum(['default','success','warning','danger']), emphasis: z.enum(['low','medium','high']) })
```

**Name components after what they represent, not how they look.** A name that states the concept tells the model *when* to reach for it; `BoxWithImageAndText` does not. The name is the single strongest hint the model gets.

**Constrain with enums wherever a value is finite.** An enum is a guarantee; a free string is a hallucination surface. `state: z.enum(['active','pending','blocked'])` can only ever render something you designed. `state: z.string()` will one day render `"URGENT!!!"`.

**Prefer one rich component over five primitives.** If the model must assemble the same tile from `Card` + `Image` + `Text` + `Badge` + `Button` every time, it will do it inconsistently and sometimes wrongly. One purpose-built component renders correctly by construction — and your design system stays enforced.

**Require the identifiers you need back.** If you cannot act on a selection without its id, make that prop required. The model reliably fills required props; optional ones it often skips.

**Keep schemas shallow.** Flat props compose reliably. Deeply nested object props are where models drift — prefer a list of child components over an array-of-objects prop when the items are visual.

**Write the description into the shape.** Enum values, required fields, and precise names carry more weight than prose, because they end up in the JSON Schema the model is constrained by.

---
## 4. The recipe — adding one component

Five steps, the same for every component. Repeat once per component until the catalog covers what your agent needs to show.

The examples below use placeholder names. Substitute whatever your agent actually has to render.

### Step 1 — name it after the thing it shows

The name is the strongest signal the model gets about *when* to reach for it. Name the concept; the layout is your renderer's business.

```
GOOD:  <Concept>Card   <Concept>Picker   <Concept>Tracker   <Concept>Summary
BAD:   BigBox          Panel2            CustomWidget       FlexRowWithIcon
```

### Step 2 — declare the props the model must supply

Presentational props only. `id`, `component`, `children`, `child`, and `action` are structural — the renderer owns them, never declare them.

```typescript
import { z } from 'zod';
import { styleSchemaShape, type UIComponentDefinition } from '@alquimia-ai/tools/genui';

const ItemCard: UIComponentDefinition = {
  name: 'ItemCard',
  schema: z.object({
    ...styleSchemaShape,                               // tone/size/emphasis/align/density
    itemId: z.string(),                                // required — you need it back
    title: z.string(),
    subtitle: z.string().optional(),
    state: z.enum(['active', 'pending', 'blocked']),   // enum, not free string
  }).passthrough(),
  allowedActions: ['selectItem'],
};
```

Ask three questions about every prop:

| Question | If yes |
|---|---|
| Is the set of valid values finite? | `z.enum([...])` — never `z.string()` |
| Do I need this value back to act on it? | Make it **required** |
| Is it about appearance rather than meaning? | Delete it — `tone`/`emphasis` already carry intent |

### Step 3 — implement the React component

It receives `A2uiNodeProps`. Read your props off `node`; use `value`/`onChange` if it collects input; use `onAction` if the user can act on it.

```tsx
function ItemCardView({ node, onAction }: A2uiNodeProps) {
  const state = node.state as string;

  return (
    <article data-state={state} className="rounded-xl border p-4">
      <h3>{node.title as string}</h3>
      {node.subtitle ? <p>{node.subtitle as string}</p> : null}
      <button
        disabled={state === 'blocked'}
        onClick={() => node.action && onAction?.(node.action)}
      >
        Choose
      </button>
    </article>
  );
}
```

Three archetypes cover nearly everything you will ever author:

| Archetype | Uses | Behavior |
|---|---|---|
| **Display** | `node` | Renders data. No action, no binding — the SDK auto-completes the surface. |
| **Input** | `node` + `value` + `onChange` | Collects a value into the data model. That model reaches the agent as `data`. |
| **Actionable** | `node` + `onAction` | Fires `onAction(node.action)`. With `wantResponse`, it completes the tool call. |

A component can be several at once — a card that shows data *and* has a button is display + actionable.

The input archetype is the one people get wrong, so here it is explicitly. Without `onChange`, the value never leaves the browser:

```tsx
function OptionPickerView({ node, value, onChange }: A2uiNodeProps) {
  const options = node.options as Array<{ value: string; label: string; available?: boolean }>;

  return (
    <fieldset>
      <legend>{node.label as string}</legend>
      {options.map((o) => (
        <button
          key={o.value}
          disabled={o.available === false}
          aria-pressed={value === o.value}
          onClick={() => onChange?.(o.value)}     // writes into the data model
        >{o.label}</button>
      ))}
    </fieldset>
  );
}
```

### Step 4 — register both sides

The catalog entry and the registry key must use the **same name**, or the component renders as a text fallback.

```typescript
// catalog
export const myCatalog: CatalogManifest = {
  id: 'https://acme.example/catalogs/mine/v1/catalog.json',
  components: { ItemCard, OptionPicker },
};

// registry
export const myRegistry = {
  ...coreUiRegistry,                // keep the core components rendering
  ItemCard: ItemCardView,
  OptionPicker: OptionPickerView,
};
```

Then catalog to the hook, registry to the renderer:

```tsx
const alquimia = useAlquimia({ assistantId, adapter, genui: { catalog: myCatalog } });
<AssistantChat alquimia={alquimia} registry={myRegistry} conversationId={id} />;
```

### Step 5 — check it round-trips

Ask the agent for something that should produce the component, then confirm:

| Symptom | Cause |
|---|---|
| `[unsupported component: X]` | Catalog name and registry key differ |
| Renders, but typing changes nothing | Input component is missing `onChange` |
| Acting on it does nothing | The action lacks `wantResponse: true` |
| Agent doesn't know what you picked | Missing a required identifier prop |
| Surface rejected entirely | No component with `id: "root"` — check `validateSurface` errors |

---

## 5. Assembling the whole catalog

Two things make a catalog pleasant to maintain.

**Use a helper** so every component inherits the style vocabulary and `.passthrough()` without repetition:

```typescript
import { z } from 'zod';
import { coreCatalog, styleSchemaShape, type CatalogManifest } from '@alquimia-ai/tools/genui';

const def = (name: string, own: z.ZodRawShape, allowedActions?: string[]) => ({
  name,
  schema: z.object({ ...styleSchemaShape, ...own }).passthrough(),
  allowedActions,
});
```

**Reuse core layout.** There is no reason to reinvent a Stack — pull the primitives you want from `coreCatalog` and add only what is yours:

```typescript
export const myCatalog: CatalogManifest = {
  id: 'https://acme.example/catalogs/mine/v1/catalog.json',
  components: {
    // borrowed from the default catalog
    Stack: coreCatalog.components.Stack,
    Grid: coreCatalog.components.Grid,
    Text: coreCatalog.components.Text,

    // yours
    ItemCard: def('ItemCard', {
      itemId: z.string(),
      title: z.string(),
      state: z.enum(['active', 'pending', 'blocked']),
    }, ['selectItem']),

    OptionPicker: def('OptionPicker', {
      label: z.string(),
      options: z.array(z.object({
        value: z.string(),
        label: z.string(),
        available: z.boolean().optional(),
      })).min(1),
    }, ['selectOption']),

    StatusTracker: def('StatusTracker', {
      referenceId: z.string(),
      status: z.enum(['received', 'processing', 'ready', 'closed']),
      eta: z.string().optional(),
    }),
  },
};
```

Borrowing layout from `coreCatalog` means the corresponding `coreUiRegistry` entries already render them — spread `coreUiRegistry` into your registry and only implement your own components.

A catalog does not need to be large. A handful of well-named components with tight enums produces better output than forty loose ones, because every name you add is another choice the model can get wrong.

## 6. Publish and version the catalog

The catalog is a contract, so it needs a published, versioned artifact. Compile the authoring catalog with `emitA2uiCatalog` — never hand-write the JSON, or it will drift from what the SDK validates against.

```typescript
import { writeFileSync } from 'node:fs';
import { emitA2uiCatalog } from '@alquimia-ai/tools/genui';
import { storeCatalog } from './store-catalog';

writeFileSync(
  'store-v1.catalog.json',
  JSON.stringify(emitA2uiCatalog(storeCatalog, { catalogId: storeCatalog.id }), null, 2),
);
```

Everything derives from that one artifact:

```
store-v1.catalog.json
        ├──► render_ui tool schema   (buildRenderUiSchema)  → what the model may emit
        ├──► prompt clause           (buildGenuiClause)     → when/how to use UI
        ├──► client validators       (validateSurface)      → reject/degrade bad output
        └──► renderer keys           (your registry)        → names you must implement
```

**Versioning rule:** the version lives in the id path (`/store/v1/`). Bump it for any breaking change — a removed component, a renamed prop, a narrowed enum. Every surface carries its `catalogId`, so older agents keep composing against the old version while unknown components degrade to a text fallback rather than crashing.

Additive changes (a new optional prop, a new component) do not need a bump.

---

## 7. Level 3 — `source: 'client'` vs `source: 'agent'`

This setting answers one question: **who tells the agent that it can draw UI?**

For the agent to emit a surface, two things must reach it on every turn:

1. the **`render_ui` tool schema** — the constrained shape it may emit (derived from your catalog)
2. the **prompt clause** — when to use UI instead of prose, and how the result comes back

They can come from the browser, or from the agent's own configuration. That is the whole difference.

### `source: 'client'` — the default

The SDK injects both on every request. Concretely, `useAlquimia` calls `withTools([buildRenderUiSchema(catalog, allow)])` and adds the clause to `extra_instructions`, and they ride out on the infer body as `evaluation_strategy`.

```tsx
genui: { catalog: myCatalog }     // source defaults to 'client'
```

- The agent needs **zero configuration** — a plain agent gains GenUI purely from the frontend.
- Change your catalog, redeploy the frontend, and the agent composes the new components on the next message. No registry change.
- Each app can hand the same agent a different catalog.

This is what you want unless you have a specific reason otherwise.

### `source: 'agent'`

The `render_ui` tool and the clause are already defined on the **agent spec** in the registry (configured in Studio). The SDK then injects nothing:

```tsx
genui: { catalog: myCatalog, source: 'agent' }
```

You still pass `catalog` — the SDK needs it to validate and render incoming surfaces. It just stops *sending* it.

**Why the switch has to exist:** `evaluation_strategy` **replaces** the spec's tools rather than merging with them. If the SDK kept injecting its own while the spec already defined `render_ui`, the client's version would silently clobber the server-side one. `source: 'agent'` is what makes the SDK stay quiet. (This invariant is pinned by `genui-source.test.ts`: no registered client tools ⇒ no `evaluation_strategy` on the wire.)

Use it when the UI capability belongs to the agent itself — governed centrally, identical across every client, versioned with the agent rather than with the frontend.

### Either way, the loop is identical

The render/complete lifecycle keys off the `ClientToolExecution` stream frame, not off who registered the tool. Nothing in your components or catalog changes between the two.

| | `source: 'client'` (default) | `source: 'agent'` |
|---|---|---|
| Tool schema + clause come from | the SDK, every request | the agent spec |
| `evaluation_strategy` on the wire | yes | no |
| Agent needs configuring | no | yes (Studio) |
| Change the catalog by | redeploying the frontend | updating the agent spec |
| Catalog still passed to the hook | yes | yes — for validation + rendering |

> **Status:** `source: 'agent'` is wired and unit-tested but not yet verified end-to-end against a live agent spec. Prefer `source: 'client'` unless you specifically need the tool defined server-side.

## 8. Rules that never change

- The catalog is **abstract** — semantic props (`tone`, `emphasis`, `availability`), never CSS, colors, or HTML.
- The renderer maps to **your** design system. Appearance is never the agent's decision.
- **UI is data, not code.** The agent emits a component tree, never markup or script.
- **Actions are semantic names, never URLs.** `assertSafeActionName` rejects scheme- and protocol-relative strings.
- The **`catalogId`** — a versioned URI — rides on every surface.
- Unknown components **degrade** to a text fallback. Never crash the chat on model output.

---

## 9. Checklist

- [ ] Component named after the thing it shows, not its layout
- [ ] Props semantic and enum-constrained wherever the value is finite
- [ ] `styleSchemaShape` spread into every schema
- [ ] `.passthrough()` on every schema so extra props warn instead of failing
- [ ] Identifiers you need back marked required, not optional
- [ ] `allowedActions` declared per component
- [ ] Registry key for **every** catalog component name — identical spelling, or it falls back to text
- [ ] Inputs call `onChange` (or their value never reaches the agent)
- [ ] Actions fire `onAction(node.action)` — no invented names
- [ ] Catalog artifact emitted with `emitA2uiCatalog`, id versioned
