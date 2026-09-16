# Audio Inference (runtime-side)

The agent itself transcribes the user's audio and synthesizes its reply. Requires runtime
**v0.5.1+** and an agent configured with STT / TTS adapters.

**This is not the same feature as `features/tts-stt.md`.** That file covers *client-side*
providers (ElevenLabs, OpenAI, Orpheus) that run in the browser, independent of the agent.
Use this file when the **agent** owns the audio pipeline and the audio rides the inference
request itself. The two can coexist but solve different problems:

| | `features/tts-stt.md` | this file |
|---|---|---|
| Runs | in the browser, via a provider | inside the runtime, as part of the turn |
| Configured by | the app (`withWhisperProvider`) | the agent's runtime config |
| Appears in the worklog | no | yes — `SpeechTranscription` / `SpeechSynthesis` nodes |

---

## 1. Sending audio

`input_audio` is a **blob reference, not bytes**. Upload first, then reference the blob the
runtime gives back. There is no multipart path on infer — it accepts JSON only.

```typescript
// 1. upload → POST /context/blob/upload → RuntimeBlob
const blob = await alquimia.sdk.uploadAttachment(audioFile);

// 2. send the reference
await alquimia.sdk.sendMessage(undefined, { inputAudio: blob });
```

`query` is optional when audio is supplied, so the call above is an audio-only turn. Send both
to supply audio *and* text:

```typescript
await alquimia.sdk.sendMessage('in Spanish, please', { inputAudio: blob });
```

A turn with neither a query nor audio throws before any request is made.

`RuntimeBlob` is the runtime's own blob schema:

```typescript
interface RuntimeBlob {
  blob_id: string;
  filename: string;
  content_size: number;
  content_type?: string | null;
  checksum?: string | null;
  created_at?: number | null;
  expires_at?: number | null;
  data_base64?: string | null;   // populated by channel ingest, not by blob upload
}
```

### Recording in the browser

`MediaRecorder` gives you a `Blob`; wrap it in a `File` so `uploadAttachment` can post it:

```typescript
const chunks: Blob[] = [];
const recorder = new MediaRecorder(await navigator.mediaDevices.getUserMedia({ audio: true }));

recorder.ondataavailable = (e) => chunks.push(e.data);
recorder.onstop = async () => {
  const file = new File(chunks, 'turn.webm', { type: recorder.mimeType });
  const blob = await alquimia.sdk.uploadAttachment(file);
  await alquimia.sdk.sendMessage(undefined, { inputAudio: blob });
};
```

Check which container formats the agent's STT adapter accepts before settling on one.

---

## 2. Controlling the spoken reply

For an audio-enabled agent the runtime defaults `output_audio` to **`true`** — it may
synthesize speech whether or not your UI can play it. A text-only frontend should say so:

```typescript
await alquimia.sdk.sendMessage('hello', { outputAudio: false });
```

The SDK only sends the field when you set it, so leaving it alone preserves the runtime's own
default. Setting `false` makes the runtime drop `audio` from the agent's output modalities for
that turn only; the agent's stored config is untouched.

---

## 3. What it looks like in the worklog

With `options: { worklog: true }`, the speech steps appear as their own nodes under the
`speech` kind — `SpeechTranscription` on the way in, `SpeechSynthesis` on the way out. See
`features/worklog.md`.

---

## 4. Route requirement

Audio input goes through the blob-upload route, so `app/api/blob/upload/route.ts` (or its
equivalent) must exist even if the app has no file-attachment UI. See
`setup/backend-routes.md`.
