# Import Paths & Environment Variables

## @alquimia-ai/tools

### SDK & Hook

```typescript
import { AlquimiaSDK } from '@alquimia-ai/tools/sdk';
import { useAlquimia, useRatings } from '@alquimia-ai/tools/hooks';
```

### Adapters

```typescript
import type { AlquimiaAdapter } from '@alquimia-ai/tools/adapters';
import { createNextJsAdapter } from '@alquimia-ai/tools/adapters/next';
import { createFetchAdapter } from '@alquimia-ai/tools/adapters/fetch';
```

### Server Proxy

```typescript
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';
import { createAlquimiaProxyHandler } from '@alquimia-ai/tools/proxy';
```

### Server Actions (framework-agnostic)

```typescript
import {
  createResource,
  readResource,
  updateResource,
  deleteResource,
} from '@alquimia-ai/tools/actions';          // generic CRUD helpers
import { handleApmRequest } from '@alquimia-ai/tools/actions';  // APM telemetry proxy
import { initConversation } from '@alquimia-ai/tools/actions';  // session management (SessionStorage — §C in sdk-init.md)
import type { SessionStorage } from '@alquimia-ai/tools/actions';
```

### Providers

```typescript
import {
  ElevenLabsWhisperProvider,
  OrpheusWhisperProvider,
  AlquimiaWhisperProvider,
  OpenAIAnalyzeCharProvider,
  OpenAIStableDiffusionProvider,
  StabilityProvider,
  ElasticLoggerProvider,
} from '@alquimia-ai/tools/providers';
```

### Utilities

```typescript
import {
  getTopicSessionId,
  getCookies,
  formatTimeWithUnit,
  createMessageId,
  serializeAxiosError,
  generateHeaders,
} from '@alquimia-ai/tools/utils';
```

### GenUI (protocol)

```typescript
import {
  coreCatalog,              // the default ~43-component catalog
  CORE_CATALOG_ID,
  coreCatalogJson,          // the published, versioned A2UI catalog artifact
  emitA2uiCatalog,          // authoring catalog -> versioned catalog.json
  buildRenderUiSchema,      // catalog -> render_ui tool schema
  buildGenuiClause,         // catalog -> prompt clause
  validateSurface,          // surface -> { ok, errors, warnings }
  validateProps,
  resolveComponent,
  buildSurfaceResult,
  buildDismissResult,
  classifyAction,
  assertSafeActionName,
  DISMISS_ACTION,
  getPointer,
  setPointer,
  resolveDynamic,
  isPathRef,
  styleSchemaShape,         // shared style vocabulary (spread into your schemas)
  TONES, SIZES, EMPHASES, ALIGNS, DENSITIES,
} from '@alquimia-ai/tools/genui';

import type {
  CatalogManifest,
  UIComponentDefinition,
  A2uiSurface,
  A2uiComponent,
  SurfaceAction,
  SurfaceResult,
  UIAction,
  UIActionKind,
  DynamicValue,
  StyleProps,
  ValidationResult,
  ToolExecutionResponse,
} from '@alquimia-ai/tools/genui';
```

### Worklog (execution trace)

```typescript
import {
  useWorklog,
  foldWorklog,
  reduceWorklog,
  frameToRecord,
  initialWorklogState,
  EVENT_REGISTRY,
  resolveInterpreter,
} from '@alquimia-ai/tools/worklog';

import type {
  WorklogState,
  WorklogNode,
  WorklogRecord,
  WorklogSummary,
  NodeKind,
  NodeStatus,
  RunStatus,
} from '@alquimia-ai/tools/worklog';
```

### Legacy re-exports (Next.js only)

`@alquimia-ai/tools/next` also re-exports these deprecated helpers for backward compatibility:

```typescript
import { handleChatRequest } from '@alquimia-ai/tools/next';  // @deprecated → use createNextJsRouteHandlers().handleInfer
import { handleStreamRequest } from '@alquimia-ai/tools/next'; // @deprecated → use createNextJsRouteHandlers().handleStream
import { initConversation } from '@alquimia-ai/tools/next';    // Next only: "use server" + cookies() — call from client via useEffect
```

### Types

```typescript
import type {
  AlquimiaMessage,
  AlquimiaSDKOptions,
  ThinkingsInferenceResponse,
  AlquimiaEventData,
  AssistantInferenceResponse,
  ToolEvent,
  Tooler,
  ToolSummary,
  ToolOutput,
  AttachmentPayload,
  AIMessageChunk,
  TTSResult,
  WhisperProvider,
  RatingData,
} from '@alquimia-ai/tools/types';
```

---

## @alquimia-ai/ui

### Providers & Theme

```typescript
import { AlquimiaUIProvider, useAlquimiaTheme } from '@alquimia-ai/ui/providers';
```

### GenUI (renderer)

```typescript
import {
  A2uiRenderer,        // renders one surface against a registry
  coreUiRegistry,      // catalog name -> shadcn component (the default binding)
  AssistantChat,       // batteries-included GenUI chat shell
} from '@alquimia-ai/ui/components/genui';

import type {
  A2uiRendererProps,
  A2uiNodeProps,       // the props every registry component receives
  AssistantChatProps,
  GenuiChatController,
} from '@alquimia-ai/ui/components/genui';
```

### Organisms (main chat components)

```typescript
import {
  AssistantMessageArea,
  AssistantInput,
  SpeechToText,
  Whisper,
  RatingDialog,
} from '@alquimia-ai/ui/components/organisms';
```

### Atoms (primitive UI)

```typescript
import {
  Button,
  Input,
  Textarea,
  Loader,
  ThinkIndicator,
  Badge,
  Skeleton,
  ScrollArea,
  Tabs, TabsList, TabsTrigger, TabsContent,
  Dialog, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter,
  Drawer, DrawerContent, DrawerHeader, DrawerTitle, DrawerDescription, DrawerFooter, DrawerTrigger, DrawerClose,
  Alert, AlertTitle, AlertDescription,
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
  Popover, PopoverContent, PopoverTrigger,
  Toast, ToastAction, ToastProvider, ToastViewport,
  Label,
  Separator,
  Switch,
  Checkbox,
  Avatar, AvatarImage, AvatarFallback,
  Typography,
  RichText,
} from '@alquimia-ai/ui/components/atoms';
```

### Molecules

```typescript
import {
  Sidebar,
  CallOut, CallOutDate, CallOutResponse, CallOutActions,
  RatingStars, RatingThumbs, RatingComment,
  AssistantButton, AssistantSuggestions,
  SonnerToaster,
} from '@alquimia-ai/ui/components/molecules';
```

### Hooks

```typescript
import { useToast } from '@alquimia-ai/ui/components/hooks';
```

### Theme CSS (import one)

```typescript
import '@alquimia-ai/ui/styles/themes/base.css';
import '@alquimia-ai/ui/styles/themes/base-alquimia.css';
import '@alquimia-ai/ui/styles/themes/base-nordic.css';
import '@alquimia-ai/ui/styles/themes/base-primary.css';
```

### Tailwind Preset

```javascript
// tailwind.config.js
module.exports = {
  presets: [require('@alquimia-ai/ui/tailwind.config')],
  content: ['./src/**/*.{ts,tsx}'],
};
```

### Utility

```typescript
import { cn } from '@alquimia-ai/ui/lib/utils';
```

---

## Package Export Maps

### @alquimia-ai/tools

| Import path | What you get |
|---|---|
| `@alquimia-ai/tools` | Main index (sdk, hooks, providers, types, utils, actions) |
| `@alquimia-ai/tools/sdk` | `AlquimiaSDK` class |
| `@alquimia-ai/tools/hooks` | `useAlquimia`, `useRatings` |
| `@alquimia-ai/tools/types` | TypeScript types |
| `@alquimia-ai/tools/providers` | All provider classes |
| `@alquimia-ai/tools/adapters` | `AlquimiaAdapter` interface, `AlquimiaSDKOptions` |
| `@alquimia-ai/tools/adapters/next` | `createNextJsAdapter()` |
| `@alquimia-ai/tools/adapters/fetch` | `createFetchAdapter()` |
| `@alquimia-ai/tools/next` | `createNextJsRouteHandlers()` + deprecated `handleChatRequest`/`handleStreamRequest` + `initConversation` (Next.js) |
| `@alquimia-ai/tools/proxy` | `createAlquimiaProxyHandler()` |
| `@alquimia-ai/tools/actions` | Framework-agnostic server actions (CRUD helpers, APM proxy, session w/ `SessionStorage` interface) |

### @alquimia-ai/ui

| Import path | What you get |
|---|---|
| `@alquimia-ai/ui` | Everything (components + providers + types) |
| `@alquimia-ai/ui/providers` | `AlquimiaUIProvider`, `useAlquimiaTheme`, `ThemeContext` |
| `@alquimia-ai/ui/components/atoms` | Atom components |
| `@alquimia-ai/ui/components/molecules` | Molecule components |
| `@alquimia-ai/ui/components/organisms` | Organism components |
| `@alquimia-ai/ui/components/templates` | Template components |
| `@alquimia-ai/ui/components/hooks` | UI hooks |
| `@alquimia-ai/ui/types` | TypeScript types |
| `@alquimia-ai/ui/lib/utils` | `cn()` utility |
| `@alquimia-ai/ui/tailwind.config` | Tailwind preset |
| `@alquimia-ai/ui/styles/themes/*` | CSS theme files |

---

## Environment Variables

### Server-side (never expose to browser)

| Variable | Purpose |
|----------|---------|
| `ASSISTANT_BASEURL` | Backend inference engine base URL |
| `ALQUIMIA_ASSISTANT_API_KEY` | API key for the backend |

### Client-side

Prefix with `NEXT_PUBLIC_` (Next.js) or `VITE_` (Vite) as appropriate.

| Variable | Purpose |
|----------|---------|
| `*_ELEVENLABS_API_KEY` | ElevenLabs TTS/STT API key |
| `*_ELEVENLABS_VOICE_ID` | ElevenLabs voice ID |
| `*_ELEVENLABS_BASEURL` | ElevenLabs base URL |
| `*_ORPHEUS_API_KEY` | Orpheus TTS API key |
| `*_WHISPER_BASEURL` | Alquimia Whisper provider base URL |
| `*_DEBUG_ERROR_MESSAGES` | Set to `"1"` to show raw error codes in UI |
| `*_KICKOFF_MESSAGE` | Automated first message text |

---

## TypeScript Types Quick Reference

```typescript
type AlquimiaMessage = Message & {
  error_code?: string;
  error_detail?: string;
  stream_id?: string;       // preserved as alias for task_id internally
  task_id?: string;         // canonical name in the new runtime
  loading?: boolean;
  tooler?: Tooler[];
  thinkings?: ThinkingsInferenceResponse[];
  additionalInfo?: string;
  attachments?: AttachmentPayload[];
};

type ToolBoxItemData = {
  id: string;
  name: string;
  data: any;
};

type TTSResult =
  | { type: 'blob'; data: Blob }
  | { type: 'url'; data: string }
  | { type: 'error'; message: string };

type Tooler = {
  control_id: string;
  tool_summary?: { name: string; parameters: Record<string, any> };
  tool_output?: { result: any; status: string };
};

interface AlquimiaAdapter {
  resolveInferUrl(assistantId: string): string;
  resolveStreamUrl(streamId: string): string;
  resolveBlobUploadUrl(): string;   // v2.1 — replaced resolveAttachmentUrl(streamId, attachmentId)
  getHeaders?(): Record<string, string>;
}

interface SessionStorage {
  get(key: string): string | undefined | null | Promise<string | undefined | null>;
  set(key: string, value: string): void | Promise<void>;
}
```
