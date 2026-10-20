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

Requires **`@alquimia-ai/tools` ≥ 2.10.0** (inline audio; `handleSubmit`'s audio argument exists from 2.9.0, but before 2.10.0 it only took an uploaded blob, which does not transcribe).

Audio goes **inline** in the inference request. Pass the recording itself — a `File`, or the
`Blob` that `MediaRecorder` produces — and the SDK base64-encodes it into `input_audio`. There
is no upload step.

**Blob upload is not for audio.** `uploadAttachment` / `POST /context/blob/upload` is for file
attachments only. Never upload a recording and pass the result as `inputAudio`: the runtime's
speech-to-text reads the bytes from `input_audio.data_base64` and never looks a blob up by id,
so an uploaded reference transcribes to nothing. The SDK rejects one with an error.

Through the hook, pass the audio as `handleSubmit`'s fifth argument. This is the path to use:
the user's turn enters `messages`, the stream opens, and the worklog resets, exactly as for a
typed message.

```typescript
await alquimia.handleSubmit(event, traceParentId, sessionId, undefined, {
  inputAudio: recording,          // File | Blob
});
```

With an empty input box that is an audio-only turn. If the user also typed something, the text
is sent as `query` alongside the audio.

Calling the SDK directly works too, but skips everything the hook owns (message list, stream,
worklog) — only do it outside a chat UI:

```typescript
await alquimia.sdk.sendMessage(undefined, { inputAudio: recording });     // audio only
await alquimia.sdk.sendMessage('in Spanish, please', { inputAudio: recording }); // audio + text
```

A turn with neither a query nor audio throws before any request is made.

On the wire the SDK sends the runtime's `Blob` shape with the bytes in it. `blob_id` is omitted
(the runtime generates one) and `filename` is free-form:

```json
{
  "input_audio": {
    "filename": "turn.webm",
    "content_size": 12345,
    "content_type": "audio/webm;codecs=opus",
    "data_base64": "<base64>"
  }
}
```

If you build the request yourself, `toInputAudio(file)` from `@alquimia-ai/tools/sdk` produces
exactly that object (type `InputAudio`).

### Size limit

The inference body travels through Kafka as a single event (10 MiB by default), base64 adds a
third, and the conversation and agent config ride in the same event. The SDK therefore caps a
turn at **`MAX_INPUT_AUDIO_BYTES` (7 MB)** of raw audio and throws before sending anything
larger. That is minutes of compressed voice; if you hit it, record shorter turns rather than
raising it.

### Recording in the browser

From **`@alquimia-ai/tools` ≥ 2.10.0**, `sendAudio` sends a voice turn without a form event.
It uses the same message list, stream and worklog as `handleSubmit`:

```typescript
const chunks: Blob[] = [];
const recorder = new MediaRecorder(await navigator.mediaDevices.getUserMedia({ audio: true }));

recorder.ondataavailable = (e) => chunks.push(e.data);
recorder.onstop = async () => {
  const recording = new Blob(chunks, { type: recorder.mimeType });
  await alquimia.sendAudio(recording);        // options?: { sessionId?, traceParentId?, additionalInfo?, outputAudio? }
};
```

With the Full Alquimia UI (**`@alquimia-ai/ui` ≥ 2.4.0**), `SpeechToText` does the recording.
Give it `onAudioRecorded` instead of `speechToText`, and the recording goes to the agent
rather than to a browser-side provider:

```tsx
<SpeechToText
  RecordAudioIcon={<MicOff />}
  IdleAudioIcon={<Mic />}
  onAudioRecorded={(audio) => alquimia.sendAudio(audio)}
  handleReplaceInput={alquimia.handleReplaceInput}
  setIsAudioRecording={alquimia.setIsAudioRecording}
/>
```

A bare `Blob` has no name, so the SDK calls it `turn.<subtype>` (`turn.webm`); wrap it in a
`File` if you want a specific name. Check which container formats the agent's STT adapter
accepts before settling on one.

---

## 2. Playing the spoken reply

Requires **runtime ≥ 0.6.0** and **`@alquimia-ai/tools` ≥ 2.10.0**. Older runtimes do
synthesize the reply, but they do not stream it to the browser.

When the agent speaks, the final stream frame carries the TTS inline. `useAlquimia` puts it on
the assistant message as `message.audio` (`OutputAudio`: `{ dataBase64, contentType, filename? }`),
next to the text. Text replies have no `audio`.

With the Full Alquimia UI (**`@alquimia-ai/ui` ≥ 2.4.0**), `AssistantMessageArea` renders a
player on those messages on its own. Add `autoPlayAudio` to play each reply as it arrives:

```tsx
<AssistantMessageArea messages={alquimia.messages} autoPlayAudio /* …other props */ />
```

A history restored on mount never auto-plays. Browsers may still block autoplay until the user
has interacted with the page, which is normally the case after they recorded a turn.

With your own components, build the `src` with `outputAudioUrl`:

```tsx
import { outputAudioUrl } from '@alquimia-ai/tools/sdk';

{message.audio && <audio controls autoPlay src={outputAudioUrl(message.audio)} />}
```

When the agent answers in audio depends on its `reply_in` policy (runtime config, not
something the frontend sends): `always` (the default) speaks every reply, `mirror` speaks
only when the user spoke, and `text` never speaks.

## 3. Turning the spoken reply off

For an audio-enabled agent the runtime defaults `output_audio` to **`true`**. A frontend that
cannot play audio should turn it off, which also keeps the base64 audio out of the stream:

```typescript
await alquimia.sdk.sendMessage('hello', { outputAudio: false });
```

The SDK only sends the field when you set it, so leaving it alone preserves the runtime's own
default. Setting `false` makes the runtime drop `audio` from the agent's output modalities for
that turn only; the agent's stored config is untouched.

---

## 4. What it looks like in the worklog

With `options: { worklog: true }`, the speech steps appear as their own nodes under the
`speech` kind — `SpeechTranscription` on the way in, `SpeechSynthesis` on the way out. See
`features/worklog.md`.

---

## 5. Route requirement

None beyond the inference route. Audio rides the infer request, so it does **not** need the
blob-upload route — that route is only for file attachments.
