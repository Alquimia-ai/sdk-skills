# SDK Initialization & Hook Setup

## 1. Create an adapter

Pick the adapter matching your server setup (see `setup/backend-routes.md` for full details):

```typescript
// Next.js App Router
import { createNextJsAdapter } from '@alquimia-ai/tools/adapters/next';
const adapter = createNextJsAdapter();

// SPA + separate server
import { createFetchAdapter } from '@alquimia-ai/tools/adapters/fetch';
const adapter = createFetchAdapter({ baseUrl: 'https://my-api.example.com' });

// SPA with stream proxy — see setup/backend-routes.md Mode 3 for full setup
// Uses a custom adapter: infer/blob upload go direct, stream goes through a local proxy
import type { AlquimiaAdapter } from '@alquimia-ai/tools/adapters';
const adapter: AlquimiaAdapter = {
  resolveInferUrl(assistantId) { return `${BACKEND}/event/infer/${assistantId}`; },
  resolveStreamUrl(streamId) { return `${PROXY}/event/stream/${streamId}`; },
  resolveBlobUploadUrl() { return `${BACKEND}/context/blob/upload`; },
  getHeaders() { return { Authorization: `Bearer ${API_KEY}` }; },
};
```

---

## 2. useAlquimia hook

The hook creates the `AlquimiaSDK` internally via `useMemo`. Pass a config object — not an SDK instance.

```typescript
import { useAlquimia } from '@alquimia-ai/tools/hooks';

const alquimia = useAlquimia({
  assistantId: 'your-assistant-id',  // required
  adapter,                            // required — from step 1
  providers: {                        // all optional
    whisper: whisperProvider,
    stableDiffusion: sdProvider,
    characterization: charProvider,
    ratings: ratingsProvider,
    logger: loggerProvider,
  },
  options: {                          // all optional
    enforceCharacterization: false,
    userId: user?.email,
    extraInstructions: { key: 'value' }, // Record<string, string> — runtime prompt overrides (renamed from extraData in v2.1)
  },
});
```

### What the hook returns

```typescript
const {
  // SDK instance (for advanced use)
  sdk,                   // AlquimiaSDK — call .withConversationId(), .withTools(), etc.

  // Message state
  messages,              // AlquimiaMessage[] — full conversation
  input,                 // string — current input text
  isMessageLoading,      // boolean — request in flight
  isMessageStreaming,    // boolean — SSE stream open
  streamingMessageId,    // string | null — ID of message being streamed
  hasThinkings,          // boolean — backend sends reasoning steps

  // Actions
  handleSubmit,          // (event, traceParentId?, sessionId?, additionalInfo?) => Promise<void>
  handleSystemMessage,   // (text, { sessionId?, messageType?, traceParentId? }) => Promise<void>
  handleInputChange,     // (event: ChangeEvent<HTMLTextAreaElement>) => void
  handleReplaceInput,    // (text: string) => void — used by STT
  handleLoadingCancel,   // () => void — cancel in-flight request
  populateMessages,      // (messages: AlquimiaMessage[]) => void — load history

  // Attachments
  attachments,           // File[]
  addAttachments,        // (files: File[]) => void
  removeAttachment,      // (index: number) => void
  clearAttachments,      // () => void

  // Audio
  isAudioRecording,      // boolean
  setIsAudioRecording,   // Dispatch<SetStateAction<boolean>>
} = alquimia;
```

### Advanced SDK methods

The hook exposes `alquimia.sdk` for chainable setters:

```typescript
alquimia.sdk.withConversationId(conversationId);
alquimia.sdk.withUserId(userId);
alquimia.sdk.withExtraInstructions({ key: 'value' }); // Record<string, string>
alquimia.sdk.withTools(toolSchemas);                  // wraps into evaluation_strategy.tool_schemas (native)
alquimia.sdk.withWhisperProvider(provider);
alquimia.sdk.withStableDiffusionProvider(provider);
alquimia.sdk.withAnalyzeCharacterizationProvider(provider);
alquimia.sdk.withRatingsProvider(provider);
alquimia.sdk.withLoggerProvider(provider);
```

> Removed in v2.1: `withForceProfile()` (the runtime no longer accepts a
> `force_profile` field) and `withExtraData()` (renamed to
> `withExtraInstructions()` with a stricter `Record<string, string>`
> type). The hook also no longer returns `evaluationStrategy` — the
> runtime stopped reporting it.

---

## 3. Conversation ID

Before sending messages you need a conversation ID. Pick one pattern for your stack.

### A. SPA / pure client (Vite, CRA, no App Router server)

There is **no** Next.js server action here: generate and persist a UUID in the browser.

```typescript
import { useState } from 'react';

const [conversationId, setConversationId] = useState<string>(() => {
  const stored = localStorage.getItem('alquimia-session');
  if (stored) return stored;
  const id = crypto.randomUUID();
  localStorage.setItem('alquimia-session', id);
  return id;
});

const resetConversation = () => {
  const id = crypto.randomUUID();
  localStorage.setItem('alquimia-session', id);
  setConversationId(id);
};
```

You can call `alquimia.sdk.withConversationId(conversationId)` on every render (as in the examples) or in a `useEffect` tied to `conversationId`.

### B. Next.js App Router — `initConversation` server action (cookies)

The SDK’s `initConversation(reset?, topicId?)` is implemented as **`"use server"`** and uses `cookies()` from `next/headers`. It **must run on the server**, so from client components you invoke it from **`useEffect`** (after mount), a form action, or an event handler — **not** during SSR as the source of truth for the id.

Import from `@alquimia-ai/tools/next` (the implementation matches `session.action.ts`: one id vs a per-topic map).

| `topicId` | Cookie name | Meaning |
|-----------|-------------|---------|
| omitted / `undefined` | `alquimia-session` | A single conversation id for the whole app (until `reset`). |
| provided | `alquimia-sessions` | JSON map of `topicId → uuid` (e.g. one thread per assistant, per doc, or per route). |

`reset === true` forces a new UUID (and updates the cookie / map entry).

```typescript
'use client';

import { useEffect, useState } from 'react';
import { initConversation } from '@alquimia-ai/tools/next';

const [conversationId, setConversationId] = useState<string | null>(null);

// Server action: run after mount so cookies are read/written on the server.
useEffect(() => {
  let cancelled = false;
  (async () => {
    const id = await initConversation(
      false,           // reset — use true for “new chat”
      undefined,       // optional topicId — pass string to use alquimia-sessions map
    );
    if (!cancelled) setConversationId(id);
  })();
  return () => {
    cancelled = true;
  };
}, [/* topicId if you pass it — include in deps so topic changes re-init */]);

useEffect(() => {
  if (conversationId) alquimia.sdk.withConversationId(conversationId);
}, [conversationId, alquimia.sdk]);
```

Until `conversationId` is non-null, disable send / show loading (same as waiting on any async init).

### C. Framework-agnostic server storage

`initConversation(storage, reset?, topicId?)` from `@alquimia-ai/tools/actions` still accepts an injectable `SessionStorage` — useful when you are **not** using the built-in Next cookie action but want the same **topic map vs single-session** behavior wired to Redis, your own cookies, etc.

```typescript
import { initConversation } from '@alquimia-ai/tools/actions';
import type { SessionStorage } from '@alquimia-ai/tools/actions';

const storage: SessionStorage = {
  get(key) { return cookies().get(key)?.value; },
  set(key, value) { cookies().set(key, value); },
};

await initConversation(storage, true, topicId);
```

---

Set the id on the SDK along with a user ID so both are included with every request:

```typescript
alquimia.sdk.withConversationId(conversationId); // skip until defined when using pattern B
alquimia.sdk.withUserId('user'); // replace with actual user identifier (e.g. user.email, user.id)
```

**Always call `withUserId`** — the backend requires it. Use a real user identifier when available, or `'user'` as a fallback for anonymous/dev usage.

---

## 4. Sending messages

```typescript
// User submit — event from form onSubmit
const handleSendMessage = async (event?: React.FormEvent<HTMLFormElement>) => {
  await alquimia.handleSubmit(
    event ?? (new Event('submit') as any),
    undefined,          // traceParentId (APM, optional)
    conversationId,     // required
  );
};

// Automated / kickoff message (no user input)
await alquimia.handleSystemMessage('Hi, how can I help you?', {
  sessionId: conversationId,
  messageType: 'system',
});
```

---

## 5. Loading conversation history

```typescript
const loadHistory = async () => {
  const res = await fetch(`/api/conversations/${conversationId}/messages`);
  const storedMessages = await res.json();
  alquimia.populateMessages(storedMessages);
};

useEffect(() => {
  if (conversationId) loadHistory();
}, [conversationId]);
```

---

## 6. Filtering messages for display

System messages should not appear in the chat UI:

```typescript
const filteredMessages = alquimia.messages.filter((m) => m.role !== 'system');
```

---

## 7. Wiring to custom UI components

When not using `@alquimia-ai/ui`, wire the hook return values to your own components:

```tsx
function MyChat() {
  const alquimia = useAlquimia({ assistantId, adapter });
  const [conversationId] = useState(() => crypto.randomUUID());

  alquimia.sdk.withConversationId(conversationId);
  alquimia.sdk.withUserId('user');

  const filteredMessages = alquimia.messages.filter((m) => m.role !== 'system');

  const handleSend = async (e?: React.FormEvent) => {
    await alquimia.handleSubmit(
      e ?? (new Event('submit') as any),
      undefined,
      conversationId,
    );
  };

  return (
    <div>
      {/* Your message list */}
      {filteredMessages.map((msg) => (
        <div key={msg.id}>
          <strong>{msg.role}:</strong> {msg.content}
        </div>
      ))}

      {/* Loading indicator */}
      {alquimia.isMessageLoading && <div>Loading...</div>}

      {/* Your input */}
      <form onSubmit={handleSend}>
        <textarea
          value={alquimia.input}
          onChange={alquimia.handleInputChange}
        />
        <button
          type="submit"
          disabled={alquimia.isMessageStreaming || !alquimia.input.trim()}
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

Key values for custom UI:
- **`messages`** — render your message list from this array
- **`input` + `handleInputChange`** — controlled textarea state
- **`handleSubmit`** — call on form submit
- **`isMessageLoading`** — show a loading indicator
- **`isMessageStreaming`** — disable send button during stream
- **`streamingMessageId`** — highlight the message being streamed
- **`handleLoadingCancel`** — wire to a cancel/stop button
- **`attachments` / `addAttachments` / `removeAttachment`** — file upload state

---

## 8. Error handling

```typescript
useEffect(() => {
  const lastErrorIndex = alquimia.messages.reduce(
    (last, msg, i) => (msg.error_code ? i : last),
    -1,
  );
  if (lastErrorIndex !== -1 && alquimia.messages.length === lastErrorIndex + 1) {
    console.error('Message error:', alquimia.messages[lastErrorIndex].error_code);
  }
}, [alquimia.messages]);
```
