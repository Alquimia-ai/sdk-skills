# Chat Component Composition

> This file is only relevant when using **Full Alquimia UI** (`@alquimia-ai/ui`). If the user chose Custom UI, skip this file — see `core/sdk-init.md` section 7 instead.

For **SPA (no Next server)**, drop `initConversation` from `@alquimia-ai/tools/next` and use the localStorage conversation id pattern in `core/sdk-init.md` §3A instead.

## Full Page Chat Layout

```tsx
"use client"; // Next.js only — remove for SPA
import { useRef, useState, useEffect } from "react";
import { useAlquimia } from "@alquimia-ai/tools/hooks";
import { initConversation } from "@alquimia-ai/tools/next";
import { AssistantMessageArea, AssistantInput } from "@alquimia-ai/ui/components/organisms";
import { ThinkIndicator, Loader } from "@alquimia-ai/ui/components/atoms";

const thoughts = [
  "Analyzing your request...",
  "Processing information...",
  "Formulating response...",
];

export function ChatComponent({ assistantId }: { assistantId: string }) {
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const scrollableRef = useRef<HTMLDivElement>(null);

  // Adapter — see setup/backend-routes.md
  const adapter = /* createNextJsAdapter() or createFetchAdapter({ baseUrl }) */;

  // Hook setup — see core/sdk-init.md
  const alquimia = useAlquimia({
    assistantId,
    adapter,
    // providers: { ... },  // optional — see features/tts-stt.md
    // options: { ... },     // optional
  });

  // Conversation ID — Next.js: cookie-backed server action; must run after mount (see core/sdk-init.md §3B).
  // Second arg = topicId: pass `assistantId` for a separate thread per assistant (`alquimia-sessions` map), or omit for one site-wide id (`alquimia-session`).
  const [conversationId, setConversationId] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      const id = await initConversation(false, assistantId);
      if (!cancelled) setConversationId(id);
    })();
    return () => {
      cancelled = true;
    };
  }, [assistantId]);

  useEffect(() => {
    if (!conversationId) return;
    alquimia.sdk.withConversationId(conversationId);
  }, [conversationId, alquimia.sdk]);

  alquimia.sdk.withUserId("user"); // replace with actual user identifier

  const isStreaming = alquimia.isMessageStreaming || alquimia.isAudioRecording;
  const filteredMessages = alquimia.messages.filter((m) => m.role !== "system");

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [alquimia.messages]);

  const handleSendMessage = async (event?: React.FormEvent<HTMLFormElement>) => {
    if (!conversationId) return;
    await alquimia.handleSubmit(
      event ?? (new Event("submit") as any),
      undefined,
      conversationId,
    );
  };

  return (
    <div className="relative flex-1 flex flex-col h-full overflow-hidden">
      <div className="flex-1 flex flex-col h-full">

        {/* Scrollable message area */}
        <div ref={scrollableRef} className="flex-1 overflow-y-auto p-4 chat-message-area">
          {alquimia.isMessageLoading && filteredMessages.length === 0 ? (
            <div className="flex items-center justify-center h-full">
              <Loader size="large" />
            </div>
          ) : filteredMessages.length === 0 ? (
            <div className="flex items-end justify-center h-full">
              <p className="text-muted-foreground font-bold text-center animate-fade-in">
                Hi, what can I help you with?
              </p>
            </div>
          ) : (
            <AssistantMessageArea
              messages={filteredMessages}
              messagesEndRef={messagesEndRef}
              isLoading={alquimia.isMessageLoading}
              isMessageStreaming={isStreaming}
              streamingMessageId={alquimia.streamingMessageId}
              className="space-y-4 px-10 alq--prose"
              actions={messageActions}
              thinkIndicator={<ThinkIndicator thoughts={thoughts} interval={2500} />}
              toolFactory={toolFactory}        // optional — see features/tools.md
              showDetailedErrors={false}
            />
          )}
        </div>

        {/* Input area */}
        <div className="p-0 pt-2 pb-0 relative">
          <AssistantInput
            sendMessageFunc={handleSendMessage}
            isButtonDisabled={isStreaming || !conversationId}
            input={alquimia.input}
            handleInputChange={alquimia.handleInputChange}
            isMessageStreaming={isStreaming}
            className="alq--assistant-input"
            placeholders={["Ask me anything...", "Type your message..."]}
            speechToTextComponent={sttComponent}     // optional — features/tts-stt.md
            userToolboxComponent={toolboxComponent}  // optional — features/tools.md
            onFileDrop={(files) => alquimia.addAttachments(files)} // optional — features/attachments.md
            attachmentsSlot={attachmentsSlot}        // optional — features/attachments.md
          />
        </div>

      </div>
    </div>
  );
}
```

---

## AssistantMessageArea Props

```typescript
<AssistantMessageArea
  messages={filteredMessages}        // AlquimiaMessage[] — REQUIRED, already filtered
  messagesEndRef={messagesEndRef}    // React.RefObject<HTMLDivElement> — REQUIRED
  isLoading={alquimia.isMessageLoading}  // boolean
  isMessageStreaming={isStreaming}    // boolean
  streamingMessageId={alquimia.streamingMessageId} // string | null
  className="space-y-4 px-10 alq--prose" // alq--prose applies markdown styles
  actions={messageActions}           // MessageAction[] — copy, TTS, etc.
  thinkIndicator={<ThinkIndicator thoughts={thoughts} interval={2500} />}
  toolFactory={toolFactory}          // renders tool call events in messages
  showDetailedErrors={false}         // show raw error_code/error_detail
  handleIsTextStreaming={(v) => setIsTextStreaming(v)} // for TTS streaming state
/>
```

---

## AssistantInput Props

```typescript
<AssistantInput
  sendMessageFunc={handleSendMessage}     // (event?) => Promise<void> — REQUIRED
  input={alquimia.input}                  // string — REQUIRED
  handleInputChange={alquimia.handleInputChange} // REQUIRED
  isButtonDisabled={isStreaming || !conversationId} // REQUIRED
  isMessageStreaming={isStreaming}         // REQUIRED
  className="alq--assistant-input"        // apply for proper styling
  placeholders={["First placeholder", "Second placeholder"]} // optional cycling text
  speechToTextComponent={sttComponent}    // optional React node
  userToolboxComponent={toolboxComponent} // optional React node
  onFileDrop={(files) => alquimia.addAttachments(files)} // optional
  attachmentsSlot={attachmentsSlot}       // optional React node (shown above textarea)
/>
```

---

## Message Actions

Actions appear on hover below each message. Use `custom: true` + `component` for components that need the full message object (like TTS).

```tsx
import { Copy } from "lucide-react";
import type { Message } from "ai";

const messageActions = [
  {
    label: "Copy",
    icon: <Copy className="h-4 w-4" />,
    onClick: async (message?: Message) => {
      const text = message?.content?.replace(/\*\*/g, "") ?? "";
      await navigator.clipboard.writeText(text);
    },
  },
  // Add TTS action here — see features/tts-stt.md
];
```

---

## Drawer (Modal) Alternative

Use when you want chat in a modal instead of a full page:

```tsx
import {
  Drawer, DrawerContent, DrawerHeader, DrawerTitle,
  DrawerFooter, DrawerTrigger
} from "@alquimia-ai/ui/components/atoms";
import { Button } from "@alquimia-ai/ui/components/atoms";

<Drawer open={isOpen} onOpenChange={setIsOpen}>
  <DrawerTrigger asChild>
    <Button className="alq--button-primary">Open Chat</Button>
  </DrawerTrigger>
  <DrawerContent className="h-[80%]">
    <DrawerHeader className="border-b border-border">
      <DrawerTitle>Assistant</DrawerTitle>
    </DrawerHeader>
    <AssistantMessageArea {...messageAreaProps} />
    <DrawerFooter className="py-6">
      <AssistantInput {...inputProps} />
    </DrawerFooter>
  </DrawerContent>
</Drawer>
```
