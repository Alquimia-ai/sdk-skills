# Reasoning Sidebar

> This feature requires **Full Alquimia UI** (`@alquimia-ai/ui`). If using custom UI, read `message.thinkings[]` directly from the hook's `messages` array and render with your own components.

The sidebar shows `message.thinkings[]` — live reasoning steps streamed from the backend. It's only relevant when `hasThinkings === true` from `useAlquimia`.

---

## Detection

```typescript
const alquimia = useAlquimia({ assistantId, adapter });
const { hasThinkings } = alquimia;

const filteredMessages = alquimia.messages.filter((m) => m.role !== 'system');

// Is the backend actively reasoning RIGHT NOW?
const lastMsg = filteredMessages.at(-1);
const isReasoning =
  lastMsg?.role === 'assistant' &&
  lastMsg?.loading === true &&
  Array.isArray(lastMsg?.thinkings) &&
  lastMsg.thinkings.length > 0 &&
  hasThinkings;

// Total reasoning steps across the conversation
const totalSteps = filteredMessages.reduce(
  (sum, m) => sum + (m.thinkings?.length ?? 0),
  0,
);
```

---

## Trigger Button

Place in your chat header. Shows a pulsing ring and brain icon when reasoning is active.

```tsx
import { Brain, SquareArrowLeft } from 'lucide-react';
import { motion } from 'framer-motion';
import { Button } from '@alquimia-ai/ui/components/atoms';

const [sidebarOpen, setSidebarOpen] = useState(false);

{!sidebarOpen && (
  <div className="fixed top-6 right-8 z-40">
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setSidebarOpen(true)}
      className={`relative bg-card shadow-sm hover:bg-accent transition-all duration-200 ${
        isReasoning ? 'ring-2 ring-primary/50' : ''
      }`}
    >
      <motion.div
        animate={isReasoning ? { scale: [1, 1.1, 1] } : {}}
        transition={{ duration: 1, repeat: Infinity }}
      >
        {isReasoning
          ? <Brain className="h-5 w-5 text-primary" />
          : <SquareArrowLeft className="h-5 w-5" />
        }
      </motion.div>

      {isReasoning && (
        <motion.div
          initial={{ scale: 0 }}
          animate={{ scale: 1 }}
          className="absolute -top-1 -right-1 w-3 h-3 bg-primary rounded-full"
        >
          <motion.div
            animate={{ scale: [1, 1.5, 1] }}
            transition={{ duration: 1, repeat: Infinity }}
            className="w-full h-full bg-primary rounded-full opacity-75"
          />
        </motion.div>
      )}

      {totalSteps > 0 && !isReasoning && (
        <motion.div
          initial={{ scale: 0 }}
          animate={{ scale: 1 }}
          className="absolute -top-2 -right-2 h-5 w-5 bg-primary text-primary-foreground rounded-full flex items-center justify-center text-xs font-medium"
        >
          {totalSteps}
        </motion.div>
      )}
    </Button>
  </div>
)}
```

---

## Layout with AnimatePresence

The sidebar animates in alongside the chat. Shrink the chat column when open.

```tsx
import { AnimatePresence, motion } from 'framer-motion';

<div className="flex h-full overflow-hidden">
  {/* Main chat column — shrinks when sidebar opens */}
  <div className={`flex-1 flex flex-col overflow-hidden transition-all ${sidebarOpen ? 'max-w-[65%]' : ''}`}>
    {/* ... chat content */}
  </div>

  <AnimatePresence>
    {sidebarOpen && (
      <motion.div
        initial={{ width: 0, opacity: 0 }}
        animate={{ width: '35%', opacity: 1 }}
        exit={{ width: 0, opacity: 0 }}
        transition={{ duration: 0.3, ease: 'easeInOut' }}
        className="h-full border-l bg-card overflow-hidden flex flex-col"
      >
        <ChatSidebar
          messages={filteredMessages}
          isMessageLoading={alquimia.isMessageLoading}
          hasThinkings={hasThinkings}
          onClose={() => setSidebarOpen(false)}
        />
      </motion.div>
    )}
  </AnimatePresence>
</div>
```

---

## ChatSidebar Component

```tsx
import { Tabs, TabsList, TabsTrigger, TabsContent, Button } from '@alquimia-ai/ui/components/atoms';
import { Brain, X } from 'lucide-react';

function ChatSidebar({ messages, isMessageLoading, hasThinkings, onClose }) {
  return (
    <Tabs defaultValue={hasThinkings ? 'reasoner' : 'info'} className="h-full flex flex-col">
      <div className="border-b px-4 py-3 flex items-center justify-between flex-shrink-0">
        <TabsList className="bg-muted">
          {hasThinkings && (
            <TabsTrigger value="reasoner" className="flex items-center gap-1.5">
              <Brain className="h-3.5 w-3.5" /> Reasoner
            </TabsTrigger>
          )}
          <TabsTrigger value="info">Info</TabsTrigger>
        </TabsList>
        <Button variant="ghost" size="icon" onClick={onClose}>
          <X className="h-4 w-4" />
        </Button>
      </div>

      <div className="flex-1 min-h-0">
        <TabsContent value="reasoner" className="h-full m-0">
          <ChatReasoner messages={messages} isMessageLoading={isMessageLoading} />
        </TabsContent>
        <TabsContent value="info" className="h-full m-0 p-4">
          {/* Your own content — conversation info, settings, etc. */}
        </TabsContent>
      </div>
    </Tabs>
  );
}
```

---

## ChatReasoner Component

The `ChatReasoner` renders `message.thinkings[]` with step numbering, toggle between latest / all messages, and tool execution results.

It handles two thinking formats:
1. **Modern** — `thinking.data.content.additional_kwargs.reasoning_content` + `tool_calls[]`
2. **Legacy** — JSON string in `thinking.data.content.content` (parsed to `{ thought, action, ... }`)

**Copy the full implementation from `examples/chat/chat-reasoner.tsx` in the SDK repo** — it's ~450 lines and handles all edge cases. The key props are:

```tsx
<ChatReasoner
  messages={filteredMessages}
  isMessageLoading={alquimia.isMessageLoading}
/>
```

The component reads `message.thinkings` and `message.tooler` directly from messages.

---

## Custom UI — reading thinkings data

If building your own reasoning panel, access the data from messages directly:

```typescript
alquimia.messages.forEach((msg) => {
  if (msg.thinkings?.length) {
    msg.thinkings.forEach((thinking) => {
      // Modern format:
      const reasoning = thinking.data?.content?.additional_kwargs?.reasoning_content;
      const toolCalls = thinking.data?.content?.additional_kwargs?.tool_calls;
      // Legacy format:
      const legacyContent = thinking.data?.content?.content; // JSON string
    });
  }
  if (msg.tooler?.length) {
    msg.tooler.forEach((t) => {
      // t.tool_summary?.name, t.tool_summary?.parameters
      // t.tool_output?.result, t.tool_output?.status
    });
  }
});
```
