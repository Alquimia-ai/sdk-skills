# Worklog — Agent Execution Trace

The worklog turns the runtime's granular `bus_mode` event stream into a normalized execution tree: which steps ran, in what order, nested how deeply, with what status. Use it for "show your work" panels, debugging views, and audit UIs.

This is distinct from the reasoning **sidebar** (`features/sidebar.md`), which shows the agent's thinkings. The worklog shows *execution*.

---

## 1. Opt in via the hook

```tsx
const alquimia = useAlquimia({
  assistantId,
  adapter,
  options: { worklog: true },
});

const { worklog } = alquimia;   // WorklogState | undefined
```

Off by default. When enabled, the hook accumulates stream frames into `worklog` and resets it at the start of each run.

---

## 2. The shape

```typescript
interface WorklogState {
  taskId: string | null;
  status: 'idle' | 'running' | 'success' | 'error';
  answer: string | null;
  nodes: WorklogNode[];               // the execution tree
  index: Record<string, number>;
  raw: WorklogRecord[];               // every ingested record
}

interface WorklogNode {
  id: string;
  kind: NodeKind;
  eventClass: string;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'skipped';
  title: string;
  depth: number;
  parentId?: string;
  children: WorklogNode[];
  command?: unknown;                  // the request side
  response?: unknown;                 // the response side (merged by control_id)
  error?: string;
  startedAt?: string;
  endedAt?: string;
  raw: WorklogRecord[];
}
```

Command and response frames sharing a `control_id` are merged into one node, so a step appears once with both sides attached rather than twice.

`NodeKind` is the semantic bucket a step falls into — use it to pick an icon, a colour, or to
filter the tree:

| `kind` | Event classes |
|---|---|
| `answer` | `AssistantInference(+Response)` — the run envelope |
| `safeguard` | `ShieldInference(+Response)`, `ShieldBlockedResponse` |
| `reasoning` | `ResponseInference(+Response)` |
| `tool` | `ServerToolExecution`, `BuiltinToolExecution`, `ClientToolExecution`, `UnknownToolExecution`, `ToolSchema`, `HumanApprovalRequired`, and their responses |
| `a2a` | `A2AInference`, `AgentDiscovery(+Response)` |
| `memory` | `ContextPersistence`, `ContextFlush(+Response)` |
| `knowledge` | `KnowledgeRetrieval(+Response)` — RAG lookups |
| `speech` | `SpeechTranscription(+Response)`, `SpeechSynthesis(+Response)` — see `features/audio-inference.md` |
| `unknown` | anything unregistered |

A `HumanApprovalRequired` node is titled `Approval required: <tool>` and stays `pending` until
someone answers it. The worklog only *shows* it — to answer it, see `features/tool-approval.md`
(and note approvals are tracked by the hook even with the worklog off).

An event class the registry does not know **does not error** — it renders under `unknown` with
the raw class name. So a trace that suddenly shows `unknown` nodes usually means the runtime
is newer than the SDK, not that something failed.

Rendering it is an ordinary recursive walk:

```tsx
function WorklogTree({ nodes }: { nodes: WorklogNode[] }) {
  return (
    <ul>
      {nodes.map((n) => (
        <li key={n.id} style={{ marginLeft: n.depth * 12 }} data-status={n.status}>
          {n.title} <em>{n.status}</em>
          {n.children.length ? <WorklogTree nodes={n.children} /> : null}
        </li>
      ))}
    </ul>
  );
}

{alquimia.worklog ? <WorklogTree nodes={alquimia.worklog.nodes} /> : null}
```

---

## 3. Standalone use

To build the tree outside `useAlquimia` — a history viewer over `GET /worklog/{task_id}/events`, say:

```typescript
import { foldWorklog, useWorklog, frameToRecord } from '@alquimia-ai/tools/worklog';

// Historical records → final state, in one pass:
const state = foldWorklog(records, taskId);

// Or accumulate live:
const { worklog, ingest, ingestMany, reset } = useWorklog(taskId);
ingest(frameToRecord(rawSseFrame));
```

| Export | Purpose |
|---|---|
| `foldWorklog(records, taskId?)` | Fold a complete record array into final state |
| `reduceWorklog(state, record)` | Single-record reducer |
| `useWorklog(taskId?)` | Live accumulator hook — `{ worklog, ingest, ingestMany, reset }` |
| `frameToRecord(frame)` | Normalize a raw SSE frame into a `WorklogRecord` |
| `EVENT_REGISTRY` / `resolveInterpreter(eventClass)` | How an event class maps to tree behavior — extend for custom event classes |

Runtime v0.5.0+ records carry `entry_hash` / `previous_hash` for a tamper-evident chain; they are optional on SSE frames. `GET /worklog/{task_id}/verify` walks that chain server-side and returns a `WorklogVerificationResult` (typed in `@alquimia-ai/tools/worklog`); it is audit surface, so call it from your backend rather than the browser.
