# User Tools / Toolbox

User tools let the user pass structured context to the inference engine alongside each message — coordinates, tokens, key-value pairs, etc. They appear in a popover on the input and are stored in state until the user clears them.

---

## Architecture

```
useToolbox (state)
  └─ toolBoxData: ToolBoxItemData[]
  └─ manageToolBoxData / removeToolBoxData / clearToolBoxData

alquimia.sdk.withExtraInstructions(getToolsForExtraInstructions(toolBoxData))
  └─ sent with every message as `extra_instructions: Record<string, string>`

<Toolbox> (popover UI — Full Alquimia UI only)
  └─ ToolBoxFactory (maps tool IDs → React components)
     └─ AccessToken, ExtraData, AttachFile, MapChart, ...

<ToolboxTags> (shows active tool data as removable chips)
```

> **v2.1 note:** `extra_instructions` replaces the legacy `extra_data`
> field, and is typed as `Record<string, string>`. Non-string tool
> values must be serialized (typically `JSON.stringify`) before being
> handed to the SDK.

---

## 1. State Management with useToolbox

```typescript
// use-toolbox.tsx
import { useReducer } from 'react';

type ToolBoxItemData = { id: string; name: string; data: any };
type ToolBoxState = ToolBoxItemData[];
type ToolBoxAction =
  | { type: 'addOrUpdate'; payload: ToolBoxItemData }
  | { type: 'remove'; payload: { id: string } }
  | { type: 'clear' };

function toolBoxReducer(state: ToolBoxState, action: ToolBoxAction): ToolBoxState {
  switch (action.type) {
    case 'addOrUpdate': {
      const i = state.findIndex((t) => t.id === action.payload.id);
      return i !== -1
        ? state.map((t, idx) => (idx === i ? action.payload : t))
        : [...state, action.payload];
    }
    case 'remove': return state.filter((t) => t.id !== action.payload.id);
    case 'clear': return [];
    default: return state;
  }
}

export function useToolbox(availableToolIds: string[]) {
  const [toolBoxData, dispatch] = useReducer(toolBoxReducer, []);

  const manageToolBoxData = (data: ToolBoxItemData) =>
    dispatch({ type: 'addOrUpdate', payload: data });

  const removeToolBoxData = (id: string) =>
    dispatch({ type: 'remove', payload: { id } });

  const clearToolBoxData = () => dispatch({ type: 'clear' });

  return { toolBoxData, manageToolBoxData, removeToolBoxData, clearToolBoxData, availableTools: availableToolIds };
}
```

---

## 2. Convert Toolbox State → SDK Extra Instructions

```typescript
function getToolsForExtraInstructions(toolBoxData: ToolBoxItemData[]) {
  return toolBoxData.reduce((acc, item) => {
    // extra_instructions is Record<string, string> — stringify non-string data.
    acc[item.id] = typeof item.data === 'string' ? item.data : JSON.stringify(item.data);
    return acc;
  }, {} as Record<string, string>);
}

// In your component — update SDK with current tool data before sending:
alquimia.sdk.withExtraInstructions({ ...getToolsForExtraInstructions(toolBoxData) });
```

You can also set extra instructions via the hook config if it's static:

```typescript
const alquimia = useAlquimia({
  assistantId,
  adapter,
  options: {
    extraInstructions: { staticKey: 'staticValue' },
  },
});
```

---

## 3. ToolBoxFactory — Maps IDs to Components

```tsx
// ToolboxFactory.tsx
import { AccessToken } from './user-tools/access-token/AccessToken';
import { ExtraData } from './user-tools/extra-data/ExtraData';

class ToolBoxFactory {
  private tools: Record<string, React.ComponentType<any>> = {
    access_token: AccessToken,
    extra_data: ExtraData,
  };

  generateToolbox(
    availableTools: string[],
    manageToolBoxData: (data: ToolBoxItemData) => void,
    toolBoxData: ToolBoxItemData[],
    closeParentElement: () => void,
    extraProps?: Record<string, any>,
  ) {
    return availableTools.map((id) => {
      const Component = this.tools[id];
      if (!Component) return null;
      return (
        <Component
          key={id}
          toolId={id}
          manageToolBoxData={manageToolBoxData}
          toolBoxData={toolBoxData}
          closeParentElement={closeParentElement}
          {...extraProps}
        />
      );
    });
  }
}

export const toolboxFactory = new ToolBoxFactory();
```

---

## 4. Toolbox Popover Component (Full Alquimia UI)

```tsx
// Toolbox.tsx
import { useState } from 'react';
import { Popover, PopoverTrigger, PopoverContent } from '@alquimia-ai/ui/components/atoms';
import { Button } from '@alquimia-ai/ui/components/atoms';
import { Settings2 } from 'lucide-react';
import { toolboxFactory } from './ToolboxFactory';

interface ToolboxProps {
  availableTools: string[];
  toolBoxData: ToolBoxItemData[];
  manageToolBoxData: (data: ToolBoxItemData) => void;
  extraProps?: Record<string, any>;
}

export function Toolbox({ availableTools, manageToolBoxData, toolBoxData, extraProps }: ToolboxProps) {
  const [open, setOpen] = useState(false);

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button variant="ghost" size="sm" className="px-1 rounded-full h-[28px] text-muted-foreground hover:text-foreground hover:bg-transparent">
          <Settings2 className="h-5 w-5" />
        </Button>
      </PopoverTrigger>
      <PopoverContent align="start" side="top" className="max-w-[180px] p-0 border-none popover-shadow">
        {toolboxFactory.generateToolbox(availableTools, manageToolBoxData, toolBoxData, () => setOpen(false), extraProps)}
      </PopoverContent>
    </Popover>
  );
}
```

---

## 5. Built-in Tool Examples

### AccessToken Tool

```tsx
export function AccessToken({ toolId, manageToolBoxData, closeParentElement }) {
  const [isOpen, setIsOpen] = useState(false);
  const [token, setToken] = useState('');

  return (
    <>
      <Button variant="ghost" className="popover-element" onClick={() => setIsOpen(true)}>
        <Key className="h-4 w-4" />
        <span>Access Token</span>
      </Button>
      <Dialog open={isOpen} onOpenChange={(v) => { setIsOpen(v); if (!v) closeParentElement(); }}>
        <DialogContent className="alq--dialog-content max-w-md">
          <DialogHeader><DialogTitle className="alq--dialog-title">Access Token</DialogTitle></DialogHeader>
          <Input value={token} onChange={(e) => setToken(e.target.value)} placeholder="Enter token..." />
          <Button onClick={() => {
            manageToolBoxData({ id: toolId, name: 'access token', data: token });
            setIsOpen(false); closeParentElement();
          }}>Save</Button>
        </DialogContent>
      </Dialog>
    </>
  );
}
```

### ExtraData Tool

```tsx
export function ExtraData({ manageToolBoxData, closeParentElement }) {
  const [isOpen, setIsOpen] = useState(false);
  const [kv, setKv] = useState({ key: '', value: '' });

  return (
    <>
      <Button variant="ghost" className="popover-element" onClick={() => setIsOpen(true)}>
        <Database className="h-4 w-4" />
        <span>Extra Data</span>
      </Button>
      <Dialog open={isOpen} onOpenChange={(v) => { setIsOpen(v); if (!v) closeParentElement(); }}>
        <DialogContent className="alq--dialog-content max-w-md">
          <DialogHeader><DialogTitle className="alq--dialog-title">Extra Data</DialogTitle></DialogHeader>
          <Input placeholder="Key" value={kv.key} onChange={(e) => setKv({ ...kv, key: e.target.value })} />
          <Input placeholder="Value" value={kv.value} onChange={(e) => setKv({ ...kv, value: e.target.value })} />
          <Button
            disabled={!kv.key.trim() || !kv.value.trim()}
            onClick={() => {
              manageToolBoxData({ id: kv.key, name: 'Extra data', data: kv.value });
              setIsOpen(false); closeParentElement();
            }}
          >Save</Button>
        </DialogContent>
      </Dialog>
    </>
  );
}
```

---

## 6. ToolboxTags — Show Active Tools

```tsx
function ToolboxTags({ toolBoxData, removeToolBoxData }) {
  return (
    <div className="flex items-center gap-1 flex-wrap">
      {toolBoxData.map((item) => (
        <span key={item.id} className="alq--badge">
          {item.name}
          <button type="button" onClick={() => removeToolBoxData(item.id)} className="ml-1">
            <X className="w-3 h-3" />
          </button>
        </span>
      ))}
    </div>
  );
}
```

---

## 7. Wire Everything Up

```tsx
const { toolBoxData, manageToolBoxData, removeToolBoxData, availableTools } =
  useToolbox(['access_token', 'extra_data']);

// Update SDK with current tool data
alquimia.sdk.withExtraInstructions({ ...getToolsForExtraInstructions(toolBoxData) });

// Full Alquimia UI — toolbox popover for AssistantInput
const toolboxComponent = availableTools.length ? (
  <div className="flex gap-1 items-end">
    <Toolbox
      availableTools={availableTools}
      manageToolBoxData={manageToolBoxData}
      toolBoxData={toolBoxData}
      extraProps={{ addAttachments: alquimia.addAttachments }}
    />
    <ToolboxTags toolBoxData={toolBoxData} removeToolBoxData={removeToolBoxData} />
  </div>
) : null;

// <AssistantInput ... userToolboxComponent={toolboxComponent} />
```

### Custom UI — tool data wiring

With custom UI, skip the Toolbox/ToolboxTags components. Just manage `toolBoxData` state and call `alquimia.sdk.withExtraInstructions()` before sending. Build your own UI for collecting tool inputs.

---

## Tool Results in Messages

Tool call results arrive in `message.tooler[]` on assistant messages. `AssistantMessageArea` renders them automatically when you pass `toolFactory`. For custom rendering, build a `ToolFactory` class that maps tool names to display components, or read `message.tooler` directly:

```typescript
// Each tooler entry:
type Tooler = {
  control_id: string;
  tool_summary?: { name: string; parameters: Record<string, any> };
  tool_output?: { result: any; status: string };
};
```
