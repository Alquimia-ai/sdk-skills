# Tool Approval — Human-in-the-Loop Tool Calls

Requires **`@alquimia-ai/tools` ≥ 2.8.0** (and **`@alquimia-ai/ui` ≥ 2.3.0** for the card).

When the runtime's authorization policy resolves a tool call to `REQUIRE_APPROVAL` — an operation classified `destructive` (always), or one with `approval_required` set — the agent does **not** run the tool. The run parks and the stream carries a `HumanApprovalRequired` instead. Nothing resumes it until someone answers `POST /event/tool-approval`.

**An app that can receive these but has no way to answer them hangs the conversation forever.** If the agent holds any tool that can be classified destructive (a Studio user can do that with one click on an MCP server), wire this in.

This is a different mechanism from client tools (`features/tools.md`, `pendingClientTool` / `completeToolExecution`): a client tool is something the *browser executes*; an approval is a *person deciding* whether the server may execute it.

---

## 1. The server route (required)

The approval POST goes through your proxy like every other leg — the API key stays server-side.

**Next.js** — one more route file, no `[...path]` segment:

```typescript
// app/api/tool-approval/route.ts
import { createNextJsRouteHandlers } from '@alquimia-ai/tools/next';

const handlers = createNextJsRouteHandlers({
  assistantBaseUrl: process.env.ASSISTANT_BASEURL!,
  apiKey: process.env.ALQUIMIA_ASSISTANT_API_KEY!,
});

export const POST = handlers.handleToolApproval;
```

**Any server:**

```typescript
app.post('/tool-approval', (c) => handler.handleToolApproval(c.req.raw));
```

Both built-in adapters already resolve the URL (`/api/tool-approval` for Next, `${baseUrl}/tool-approval` for fetch); override with `toolApprovalRoute` / `toolApprovalPath`. A custom `AlquimiaAdapter` needs `resolveToolApprovalUrl()` — without it `approveTool` resolves to `{ outcome: 'error' }` and the SDK method throws. Full detail in `setup/backend-routes.md`.

---

## 2. Full Alquimia UI — nothing else to do with `AssistantChat`

`AssistantChat` renders pending approvals in place when you pass the whole hook return:

```tsx
const alquimia = useAlquimia({ assistantId, adapter });

<AssistantChat alquimia={alquimia} approvalChannelName="WhatsApp" />
```

If you compose `AssistantMessageArea` yourself, render `ToolApprovalCard`:

```tsx
import { ToolApprovalCard } from '@alquimia-ai/ui/components/organisms';

{alquimia.toolApprovals.map((approval) => (
  <ToolApprovalCard
    key={approval.controlId}
    approval={approval}
    onDecide={alquimia.approveTool}
    channelName="WhatsApp"          // optional — named in the "answer it elsewhere" message
  />
))}
```

The card shows the tool name and its arguments, Approve / Reject (rejecting takes an optional reason, kept in the runtime's audit trail), the settled state ("Approved by you" / "Rejected elsewhere" plus the reason), and — when the gate cannot be answered from here — an explanation instead of buttons. Each `approval.afterCount` is the message count it was raised after, if you want to interleave it in the transcript like `AssistantChat` does.

---

## 3. Custom UI — the hook surface

```tsx
const { toolApprovals, pendingApprovals, approveTool } = useAlquimia({ assistantId, adapter });

{pendingApprovals.map((a) => (
  <div key={a.controlId}>
    <p>The agent wants to run <code>{a.toolName}</code></p>
    <pre>{JSON.stringify(a.arguments, null, 2)}</pre>
    {a.canAnswer ? (
      <>
        <button disabled={a.status === 'submitting'} onClick={() => approveTool(a.controlId, true)}>
          Approve
        </button>
        <button disabled={a.status === 'submitting'} onClick={() => approveTool(a.controlId, false, reason)}>
          Reject
        </button>
      </>
    ) : (
      <p>This request can't be answered here.</p>
    )}
  </div>
))}
```

Gates are tracked **always** — no `options.worklog` needed. `toolApprovals` holds every gate of the conversation (pending and settled), `pendingApprovals` only the undecided ones.

```typescript
interface ToolApproval {
  controlId: string;        // the gate's id — pass THIS to approveTool
  toolCallId: string | null;// the parked tool call's own id — NOT what approveTool takes
  toolName: string | null;
  arguments: unknown;       // what is being approved — always show it
  channelOwned: boolean;    // answered on the originating channel; never offer buttons
  status: 'pending' | 'submitting' | 'approved' | 'rejected';
  decision: { approved: boolean; reason: string | null; decidedAt: string } | null; // from the stream only
  reason: string | null;    // rejection reason (stream's, or this client's accepted one)
  decidedHere: boolean;     // this client made the accepted decision
  canAnswer: boolean;       // buttons can still work
  lastResult: ToolApprovalResult | null;
  requestedAt: string | null;
  afterCount: number;       // message count the gate was raised after
}
```

Rules the hook already enforces — don't reimplement them:

- **The stream settles a gate, not the POST.** A decision made by another operator or on the originating channel arrives as `HumanApprovalRequiredResponse` and settles the gate here too. `status` reflects an accepted local decision immediately; `decision` stays `null` until the stream confirms.
- **Idempotent per gate.** A second `approveTool` while one is in flight, or after one was accepted, returns the same promise — a double click is one POST.
- **Never throws for a refusal.** It resolves to a `ToolApprovalResult`:

| `outcome` | Runtime answer | `canAnswer` after |
|---|---|---|
| `accepted` | 200 | `false` — the gate is decided |
| `already-decided` | 409 — a decision is already recorded | `false` (someone else answered; the stream will settle it) |
| `wrong-channel` | 409 — must be answered through its original channel | `false` |
| `not-pending` | 409 — no pending approval for this control id | `false` |
| `forbidden` | 401 / 403 | `false` |
| `not-found` | 404 — task finished or expired | `false` |
| `error` | anything else (5xx, network) | `true` — retry is fine |

`lastResult.detail` carries the runtime's message verbatim.

---

## 4. Authorization — who may approve

The runtime authorizes **the caller of `/event/tool-approval`**, not the app:

- The bundled proxy sends the server's static API token, which maps to `API_TOKEN_ROLES` (default `platform-admin`) and passes. For most apps this just works.
- If your proxy forwards an **end-user token** instead, that user needs a role in `TOOL_APPROVAL_ROLES` (default `editor,operator`; platform and org admins always pass) **and** must be bound to the task's agentspace — otherwise `forbidden`.
- The SDK sends the `agentspaceid` the runtime returned on infer; the runtime checks a non-admin caller against it. Nothing to configure.

Because the decision is recorded with the approver's identity in the audit trail, think about *who* is clicking: with the static token, every approval is attributed to `api-token`. If attribution matters, forward the user's token.

---

## 5. Without the hook

```typescript
// Post a decision directly:
const result = await sdk.submitToolApproval(controlId, false, 'wrong record');
// → { outcome: 'accepted' } | { outcome: 'already-decided', status: 409, detail } | ...

// Derive gates from any worklog records (e.g. a history viewer):
import { foldApprovals, pendingApprovals, listApprovals } from '@alquimia-ai/tools/worklog';
const state = foldApprovals(records);
pendingApprovals(state);   // ApprovalRequest[] with decision === null
```

---

## Gotchas

- **`controlId` ≠ `toolCallId`.** The runtime keys pending approvals by the `HumanApprovalRequired` event's own control id. Posting the tool call's id gets `not-pending`.
- **The worklog does not record who decided** — only the decision and the rejection reason. "Approved elsewhere" is as specific as the browser can be; the approver lives in the runtime's audit records.
- **Channel-owned gates** (an MCP server with the `mcp_approval` runtime extension) are answered over the conversation's originating channel. They are flagged `channelOwned` from the request itself; the runtime would 409 an HTTP answer. Its 409 does not name the channel — pass `channelName` / `approvalChannelName` if you know it.
- **Rejection is not an error.** The agent receives the rejection (with the reason) as a failed tool result and usually replies explaining it — the stream continues either way.
- While an approval waits, the run is parked: `isLoading` stays `true`. `AssistantChat` hides the typing indicator and keeps the composer disabled; do the same in custom UI.
