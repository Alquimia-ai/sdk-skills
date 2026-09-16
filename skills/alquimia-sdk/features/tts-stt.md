# Text-to-Speech & Speech-to-Text (client-side)

Both features use a `WhisperProvider` passed to the hook via the `providers.whisper` config.
The audio pipeline runs **in the browser**, against whatever vendor you configure — the agent
is not involved and nothing about it appears in the worklog.

> If the agent itself should transcribe the user and speak its reply, that is a different
> feature: `features/audio-inference.md`. Pick this file when the app owns the audio, that one
> when the agent does.

---

## Provider Setup

### ElevenLabs (most common)

```typescript
import { ElevenLabsWhisperProvider } from '@alquimia-ai/tools/providers';

const whisperProvider = new ElevenLabsWhisperProvider({
  apiKey: process.env.NEXT_PUBLIC_ELEVENLABS_API_KEY!,
  voiceId: process.env.NEXT_PUBLIC_ELEVENLABS_VOICE_ID!,
  baseURL: process.env.NEXT_PUBLIC_ELEVENLABS_BASEURL!,
  model: 'eleven_multilingual_v2',  // optional, this is the default
  language: 'es',                   // optional, this is the default
  voiceSettings: {                  // optional
    stability: 0.7,
    similarity_boost: 0.3,
    style: 0.2,
  },
});
```

### Orpheus

```typescript
import { OrpheusWhisperProvider } from '@alquimia-ai/tools/providers';

const whisperProvider = new OrpheusWhisperProvider({
  apiKey: process.env.NEXT_PUBLIC_ORPHEUS_API_KEY!,
  modelId: 'your-model-id',
  voiceSettings: {
    voice: 'tara',
    temperature: 0.6,
    top_p: 0.95,
  },
});
```

### Alquimia (self-hosted)

```typescript
import { AlquimiaWhisperProvider } from '@alquimia-ai/tools/providers';

const whisperProvider = new AlquimiaWhisperProvider({
  baseURL: process.env.NEXT_PUBLIC_WHISPER_BASEURL!,
  ttsRoute: '/tts',
  sttRoute: '/stt',
});
```

### Pass to useAlquimia

```typescript
const alquimia = useAlquimia({
  assistantId,
  adapter,
  providers: {
    whisper: whisperProvider,
  },
});

// alquimia.sdk.textToSpeech is now available
```

---

## Text-to-Speech

TTS plays audio for any assistant message. Add it as a custom message action.

### With Alquimia UI components

```tsx
import { Speaker } from 'lucide-react';
import { Whisper } from '@alquimia-ai/ui/components/organisms';
import type { Message } from 'ai';

const messageActions = [
  // ... other actions (Copy, etc.)
  {
    label: 'text to speech',
    icon: <Speaker className="h-4 w-4" />,
    onClick: async () => {},
    custom: true,
    component: ({ message }: { message?: Message }) => (
      <Whisper
        message={message!}
        isMessageStreaming={alquimia.isMessageStreaming}
        textToSpeech={alquimia.sdk.textToSpeech}
        className="alq--action-whisper"
      />
    ),
  },
];

// Pass to AssistantMessageArea:
// <AssistantMessageArea ... actions={messageActions} />
```

### With custom UI

Call `alquimia.sdk.textToSpeech` directly — it returns a `TTSResult`:

```typescript
const result = await alquimia.sdk.textToSpeech(messageContent);
// result: { type: 'blob', data: Blob } | { type: 'url', data: string } | { type: 'error', message: string }

if (result.type === 'blob') {
  const audio = new Audio(URL.createObjectURL(result.data));
  audio.play();
} else if (result.type === 'url') {
  const audio = new Audio(result.data);
  audio.play();
}
```

### TTS streaming state

If you want to block input during TTS playback:

```tsx
const [isTextStreaming, setIsTextStreaming] = useState(false);
const isStreaming = alquimia.isMessageStreaming || alquimia.isAudioRecording || isTextStreaming;

// Pass to AssistantMessageArea (Full UI):
// handleIsTextStreaming={(v) => setIsTextStreaming(v)}
```

---

## Speech-to-Text

STT requires a `speechToText` function — a backend call that accepts base64 audio and returns a transcript string. Implement it as a server action (Next.js) or API route call.

### Server Action (Next.js)

```typescript
// actions/speech.ts
'use server';
export async function speechToText(audioBase64: string): Promise<string> {
  const provider = new ElevenLabsWhisperProvider({ /* server-side config */ });
  return provider.speechToText(audioBase64);
}
```

### API route call (any framework)

```typescript
async function speechToText(audioBase64: string): Promise<string> {
  const res = await fetch('/api/speech-to-text', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ audio: audioBase64 }),
  });
  const { transcript } = await res.json();
  return transcript;
}
```

### SpeechToText Component (Full Alquimia UI)

```tsx
import { SpeechToText } from '@alquimia-ai/ui/components/organisms';
import { Mic } from 'lucide-react';

const isStreaming = alquimia.isMessageStreaming || alquimia.isAudioRecording;

const RecordingAudioIcon = (
  <div className="flex items-center gap-2">
    <p className="text-sm text-muted-foreground recording-text">Recording...</p>
    <div className="bg-destructive w-8 h-8 rounded-full flex items-center justify-center">
      <div className="w-3 h-3 bg-white" />
    </div>
  </div>
);

const IdleAudioIcon = (
  <div className={`${isStreaming ? 'bg-muted' : 'bg-primary'} w-8 h-8 rounded-full flex items-center justify-center`}>
    <Mic className="text-primary-foreground w-5 h-5" strokeWidth={1.5} />
  </div>
);

const sttComponent = (
  <SpeechToText
    speechToText={speechToText}
    RecordAudioIcon={RecordingAudioIcon}
    IdleAudioIcon={IdleAudioIcon}
    handleReplaceInput={alquimia.handleReplaceInput}
    setIsAudioRecording={alquimia.setIsAudioRecording}
    onSendAudio={() => handleSendMessage()}
    className={`alq--speech-to-text ${alquimia.input.length > 0 ? 'hidden' : ''}`}
  />
);

// Pass to AssistantInput:
// <AssistantInput ... speechToTextComponent={sttComponent} />
```

### Custom UI — STT wiring

Call `speechToText(base64)` from your own recording component, then use `alquimia.handleReplaceInput(transcript)` to set the input text.

---

## Environment Variables (client-side)

```bash
NEXT_PUBLIC_ELEVENLABS_API_KEY=
NEXT_PUBLIC_ELEVENLABS_VOICE_ID=
NEXT_PUBLIC_ELEVENLABS_BASEURL=

NEXT_PUBLIC_ORPHEUS_API_KEY=
NEXT_PUBLIC_WHISPER_BASEURL=   # for AlquimiaWhisperProvider
```

For Vite/SPA, replace `NEXT_PUBLIC_` with `VITE_` and access via `import.meta.env.VITE_*`.
