# GenUI — Building Your Own Components

Read `features/genui.md` first. This file covers making the agent compose **your** components: your design system, your domain, your vocabulary.

This is the main reason GenUI is catalog-based. The agent does not know what a product, a policy, a shipment, or a lab result is — it knows the component names *you* authorized and the props *you* declared. Swap the catalog and the same agent composes a different domain, unchanged.

---

## 1. Pick a level

| Level | You want | Cost |
|---|---|---|
| **1 — Swap the registry** | The default catalog, rendered with your components | Minutes. Keeps ~43 components working. |
| **2 — Author a catalog** | Your own domain components, your own vocabulary | The real work. Full control of what the agent may emit. |
| **3 — Move the tool to the agent spec** | The tool + clause defined in Studio, not injected by the client | Config, not code. Composes with 1 or 2. |

Levels 1 and 2 combine: a domain catalog usually starts by reusing core layout components and adding domain ones on top.

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
    ProductCard: {
      name: 'ProductCard',
      schema: z.object({
        ...styleSchemaShape,                      // inherit tone/size/emphasis/align/density
        sku: z.string(),
        title: z.string(),
        price: z.number(),
        currency: z.string().length(3).optional(),
        imageUrl: z.string().optional(),
        badge: z.enum(['new', 'sale', 'low-stock']).optional(),
      }).passthrough(),
      allowedActions: ['selectProduct', 'addToCart'],
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

**Name components after domain concepts, not layouts.** `ProductCard` and `OrderTracker` tell the model *when* to use them. `BoxWithImageAndText` does not. The name is the single strongest hint the model gets.

**Constrain with enums wherever a value is finite.** An enum is a guarantee; a free string is a hallucination surface. `badge: z.enum(['new','sale','low-stock'])` can only ever render something you designed. `badge: z.string()` will one day render `"BEST DEAL!!!"`.

**Prefer one rich component over five primitives.** If the model must assemble a product tile from `Card` + `Image` + `Text` + `Badge` + `Button` every time, it will do it inconsistently and sometimes wrongly. `ProductCard` renders correctly by construction — and your design system stays enforced.

**Require the identifiers you need back.** If you cannot act on a selection without a `sku`, make `sku` required. The model fills required props; optional ones it often skips.

**Keep schemas shallow.** Flat props compose reliably. Deeply nested object props are where models drift — prefer a list of child components over an array-of-objects prop when the items are visual.

**Write the description into the shape.** Enum values, required fields, and precise names carry more weight than prose, because they end up in the JSON Schema the model is constrained by.

---

## 4. Worked example — an ecommerce catalog

The pattern generalizes; ecommerce is just a concrete instance. Here the agent can browse, recommend, collect a choice, and report order status — without inventing any markup.

```typescript
// store-catalog.ts
import { z } from 'zod';
import { coreCatalog, styleSchemaShape, type CatalogManifest } from '@alquimia-ai/tools/genui';

const def = (name: string, own: z.ZodRawShape, allowedActions?: string[]) => ({
  name,
  schema: z.object({ ...styleSchemaShape, ...own }).passthrough(),
  allowedActions,
});

const money = z.object({ amount: z.number(), currency: z.string().length(3) });

export const storeCatalog: CatalogManifest = {
  id: 'https://acme.example/catalogs/store/v1/catalog.json',
  components: {
    // Reuse core layout — no reason to reinvent a Stack.
    Stack: coreCatalog.components.Stack,
    Grid: coreCatalog.components.Grid,
    Text: coreCatalog.components.Text,
    Heading: coreCatalog.components.Heading,

    // ---- domain ----
    ProductCard: def('ProductCard', {
      sku: z.string(),
      title: z.string(),
      price: money,
      imageUrl: z.string().optional(),
      rating: z.number().min(0).max(5).optional(),
      badge: z.enum(['new', 'sale', 'low-stock', 'bestseller']).optional(),
      availability: z.enum(['in-stock', 'backorder', 'out-of-stock']).optional(),
    }, ['selectProduct', 'addToCart']),

    ProductGrid: def('ProductGrid', {
      columns: z.number().int().min(1).max(4).optional(),
    }),

    VariantPicker: def('VariantPicker', {
      label: z.string(),
      options: z.array(z.object({
        value: z.string(),
        label: z.string(),
        available: z.boolean().optional(),
      })).min(1),
    }, ['selectVariant']),

    CartSummary: def('CartSummary', {
      lines: z.array(z.object({ sku: z.string(), title: z.string(), qty: z.number().int(), total: money })),
      subtotal: money,
      shipping: money.optional(),
      total: money,
    }, ['checkout', 'editCart']),

    OrderTracker: def('OrderTracker', {
      orderId: z.string(),
      status: z.enum(['placed', 'packed', 'shipped', 'out-for-delivery', 'delivered']),
      eta: z.string().optional(),
      carrier: z.string().optional(),
    }, ['trackShipment']),
  },
};
```

Note what the schemas do: `availability` and `status` are enums, so the agent can never invent a state your UI has no design for. `sku` is required on `ProductCard`, so a selection always identifies a real product. `price` is a structured `money` object, so currency can't go missing.

### The matching registry

```tsx
// store-registry.tsx
import type { A2uiNodeProps } from '@alquimia-ai/ui/components/genui';
import { coreUiRegistry } from '@alquimia-ai/ui/components/genui';

function ProductCard({ node, onAction }: A2uiNodeProps) {
  const price = node.price as { amount: number; currency: string };
  const badge = node.badge as string | undefined;

  return (
    <article className="rounded-xl border p-4">
      {node.imageUrl ? <img src={node.imageUrl as string} alt={node.title as string} /> : null}
      {badge ? <span data-badge={badge}>{badge}</span> : null}
      <h3>{node.title as string}</h3>
      <p>{new Intl.NumberFormat(undefined, { style: 'currency', currency: price.currency })
            .format(price.amount)}</p>
      <button
        disabled={node.availability === 'out-of-stock'}
        onClick={() => node.action && onAction?.(node.action)}
      >
        {node.availability === 'out-of-stock' ? 'Out of stock' : 'Choose'}
      </button>
    </article>
  );
}

function ProductGrid({ node, children }: A2uiNodeProps) {
  return (
    <div style={{ display: 'grid', gridTemplateColumns: `repeat(${(node.columns as number) ?? 3}, 1fr)`, gap: 16 }}>
      {children}
    </div>
  );
}

function VariantPicker({ node, value, onChange }: A2uiNodeProps) {
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

// CartSummary and OrderTracker follow the same shape: read props off `node`,
// call onAction?.(node.action) for anything the agent should hear about.

export const storeRegistry: Record<string, React.ComponentType<A2uiNodeProps>> = {
  ...coreUiRegistry,          // keeps Stack/Grid/Text/Heading rendering
  ProductCard,
  ProductGrid,
  VariantPicker,
  CartSummary,
  OrderTracker,
};
```

`VariantPicker` calls `onChange`, so the chosen variant lands in the data model and ships to the agent. `ProductCard` calls `onAction`, so clicking it completes the tool call and names the component that fired — which is how the agent knows *which* product was picked out of twelve.

### Wire it up

```tsx
const alquimia = useAlquimia({
  assistantId,
  adapter,
  genui: { catalog: storeCatalog },
});

<AssistantChat alquimia={alquimia} registry={storeRegistry} conversationId={id} />;
```

That is the whole integration. The hook derives the `render_ui` tool schema, the prompt clause, and the validators from `storeCatalog`, so the agent now composes product UI and nothing else.

### The same pattern in other verticals

| Vertical | Domain components worth authoring |
|---|---|
| **Banking** | `AccountCard` `TransactionList` `TransferForm` `PaymentConfirm` `CardControls` |
| **Insurance** | `PolicyCard` `CoverageComparison` `ClaimForm` `ClaimTracker` `QuoteBreakdown` |
| **Logistics** | `ShipmentTracker` `RouteMap` `SlotPicker` `ManifestTable` `ExceptionAlert` |
| **Healthcare** | `AppointmentSlots` `MedicationList` `SymptomChecklist` `LabResultPanel` `TriageBanner` |
| **Travel** | `FlightOption` `HotelCard` `ItineraryTimeline` `SeatMap` `FareBreakdown` |
| **HR / Internal** | `EmployeeCard` `TimeOffRequest` `ApprovalQueue` `OrgChart` `PayslipSummary` |

The recurring shape is the same everywhere: **one card component per domain object, one picker per decision, one tracker per process, one form per action.** If you can name the nouns and the decisions in your domain, you can write the catalog.

---

## 5. Publish and version the catalog

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

## 6. Level 3 — move the tool onto the agent spec

By default (`source: 'client'`) the SDK injects the `render_ui` tool and the prompt clause on every request. If the tool and clause instead live on the **agent spec** in the registry (configured in Studio), set `source: 'agent'` — the SDK then injects nothing and sends no `evaluation_strategy`, which would otherwise replace the spec's tool:

```tsx
const alquimia = useAlquimia({
  assistantId,
  adapter,
  genui: { catalog: storeCatalog, source: 'agent' },
});
```

You still pass the catalog — the SDK needs it to validate and render incoming surfaces. The render/complete loop is identical either way, because it keys off the `ClientToolExecution` stream frame rather than who registered the tool.

> **Status:** `source: 'agent'` is wired and unit-tested but has not yet been verified end-to-end against a live agent spec. Prefer `source: 'client'` unless you specifically need the tool defined server-side.

---

## 7. Rules that never change

- The catalog is **abstract** — semantic props (`tone`, `emphasis`, `availability`), never CSS, colors, or HTML.
- The renderer maps to **your** design system. Appearance is never the agent's decision.
- **UI is data, not code.** The agent emits a component tree, never markup or script.
- **Actions are semantic names, never URLs.** `assertSafeActionName` rejects scheme- and protocol-relative strings.
- The **`catalogId`** — a versioned URI — rides on every surface.
- Unknown components **degrade** to a text fallback. Never crash the chat on model output.

---

## 8. Checklist

- [ ] Components named after domain concepts, not layouts
- [ ] Props semantic and enum-constrained wherever the value is finite
- [ ] `styleSchemaShape` spread into every schema
- [ ] `.passthrough()` on every schema so extra props warn instead of failing
- [ ] Identifiers you need back (`sku`, `orderId`, …) marked required
- [ ] `allowedActions` declared per component
- [ ] Registry key for **every** catalog component name
- [ ] Inputs call `onChange` (or their value never reaches the agent)
- [ ] Actions fire `onAction(node.action)` — no invented names
- [ ] Catalog artifact emitted with `emitA2uiCatalog`, id versioned
