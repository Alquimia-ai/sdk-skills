# Attachments

The SDK handles file attachments natively. Files are uploaded to the backend when the message is sent, not immediately on selection.

> **v2.1 note:** The hook-level API (`attachments`, `addAttachments`,
> `removeAttachment`, `clearAttachments`) is unchanged. Under the hood
> the transport moved from the removed
> `POST /event/attachment/{stream_id}/{attachment_id}` endpoint to
> `POST /context/blob/upload` (a single path-parameter-less endpoint —
> identity travels via headers, set by the SDK). See
> `setup/backend-routes.md` for the matching adapter/proxy/route updates.

---

## Hook Values

From `useAlquimia({ ... })`:

```typescript
const {
  attachments,        // File[] — currently queued files
  addAttachments,     // (files: File[]) => void
  removeAttachment,   // (index: number) => void
  clearAttachments,   // () => void
} = useAlquimia({ assistantId, adapter });
```

---

## Uploading a single file yourself

The hook uploads queued attachments automatically. To upload one directly — which is how audio
input works — call the SDK and keep what it returns:

```typescript
const blob = await alquimia.sdk.uploadAttachment(file);
// RuntimeBlob: { blob_id, filename, content_size, content_type, ... }
```

It resolves to the blob the runtime stored. That object is what `sendMessage`'s `inputAudio`
expects (see `features/audio-inference.md`); for ordinary attachments you can ignore it.

---

## AttachmentsList Component

Build this inline — it's simple enough not to need a separate file:

```tsx
import { X } from 'lucide-react';

function AttachmentsList({
  attachments,
  onRemove,
}: {
  attachments: File[];
  onRemove: (index: number) => void;
}) {
  if (attachments.length === 0) return null;

  return (
    <div className="flex gap-2 flex-wrap px-3 pt-2 pb-1">
      {attachments.map((file, i) => (
        <div
          key={`${file.name}-${i}`}
          className="flex items-center gap-1 px-2 py-1 rounded-md bg-muted text-sm text-foreground"
        >
          <span className="truncate max-w-[150px]">{file.name}</span>
          <button
            type="button"
            onClick={() => onRemove(i)}
            className="ml-1 hover:text-destructive transition-colors"
          >
            <X className="w-3 h-3" />
          </button>
        </div>
      ))}
    </div>
  );
}
```

---

## Wire Up to AssistantInput (Full Alquimia UI)

```tsx
const attachmentsSlot = (
  <AttachmentsList
    attachments={alquimia.attachments}
    onRemove={alquimia.removeAttachment}
  />
);

<AssistantInput
  // ... other props
  onFileDrop={(files) => alquimia.addAttachments(files)}
  attachmentsSlot={attachmentsSlot}
/>
```

`onFileDrop` activates the drag-and-drop zone on the textarea. The `attachmentsSlot` is rendered above the textarea inside the input container.

---

## Custom UI — attachment wiring

With your own components, use the hook values directly:

```tsx
<input
  type="file"
  multiple
  onChange={(e) => {
    const files = Array.from(e.target.files ?? []);
    alquimia.addAttachments(files);
  }}
/>

{alquimia.attachments.map((file, i) => (
  <div key={i}>
    {file.name}
    <button onClick={() => alquimia.removeAttachment(i)}>Remove</button>
  </div>
))}
```

---

## AttachFile User Tool (optional)

If you want a file picker inside the toolbox (instead of or in addition to drag-and-drop):

```tsx
export function AttachFile({
  manageToolBoxData,
  closeParentElement,
  addAttachments,
}: {
  manageToolBoxData: (data: ToolBoxItemData) => void;
  closeParentElement: () => void;
  addAttachments: (files: File[]) => void;
}) {
  const handleClick = () => {
    const input = document.createElement('input');
    input.type = 'file';
    input.multiple = true;
    input.onchange = (e) => {
      const files = Array.from((e.target as HTMLInputElement).files ?? []);
      addAttachments(files);
      closeParentElement();
    };
    input.click();
  };

  return (
    <Button variant="ghost" className="popover-element" onClick={handleClick}>
      <Paperclip className="h-4 w-4" />
      <span>Attach File</span>
    </Button>
  );
}
```

Pass `addAttachments` via `extraProps` in the Toolbox component:

```tsx
<Toolbox extraProps={{ addAttachments: alquimia.addAttachments }} />
```
