# Installation & Setup

## Path A: Full Alquimia UI (`@alquimia-ai/tools` + `@alquimia-ai/ui`)

### Dependencies

```bash
npm install @alquimia-ai/tools @alquimia-ai/ui
npm install tailwindcss tailwindcss-animate
# Peer deps (usually auto-installed as transitive deps)
npm install framer-motion lucide-react
```

Neither package requires `next` or `next-themes` as a peer dependency.

### tailwind.config.js

Use the UI package as a **preset** — one line replaces all color/radius/animation token config:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  presets: [require('@alquimia-ai/ui/tailwind.config')],
  content: [
    './pages/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './app/**/*.{ts,tsx}',
    './src/**/*.{ts,tsx}',
  ],
  // your own theme extensions go here
};
```

The preset already includes the path to the UI package dist files in `content`, all color/radius/animation mappings, and the `tailwindcss-animate` plugin.

### Global CSS — Theme files

Import one of the built-in theme stylesheets in your global CSS or layout entry:

```ts
import '@alquimia-ai/ui/styles/themes/base.css';           // neutral light + dark
import '@alquimia-ai/ui/styles/themes/base-alquimia.css';  // alquimia brand
import '@alquimia-ai/ui/styles/themes/base-nordic.css';    // nordic theme
import '@alquimia-ai/ui/styles/themes/base-primary.css';   // primary color theme
```

Pick one. Each theme defines the CSS variables (`--background`, `--foreground`, `--primary`, etc.) that all components reference via Tailwind.

**Alternative:** Define your own CSS variables instead of importing a theme file. The components will use them automatically:

```css
@layer base {
  :root {
    --primary: 223 82% 60%;
    --primary-foreground: 0 0% 98%;
    --background: 0 0% 95%;
    --foreground: 0 0% 3.9%;
    --card: 0 0% 100%;
    --card-foreground: 0 0% 3.9%;
    --muted: 0 0% 96.1%;
    --muted-foreground: 0 0% 45.1%;
    --accent: 0 0% 100%;
    --accent-foreground: 223 82% 60%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 0 0% 98%;
    --border: 0 0% 85%;
    --input: 0 0% 89.8%;
    --ring: 221 83% 53%;
    --radius: 0.75rem;
  }
}
```

### Component-level styling

The SDK components use `alq--` prefixed CSS classes. Copy the contents of `styles/chat.css` (included in this skill) into your global stylesheet for the default look. See `styles/style-guide.md` for the full class reference.

### AlquimiaUIProvider

Wrap your app with `AlquimiaUIProvider` for dark mode support. Replaces `next-themes` — no Next.js dependency:

```tsx
import { AlquimiaUIProvider } from '@alquimia-ai/ui';

// In your layout or app root:
<AlquimiaUIProvider defaultMode="system"> {/* 'light' | 'dark' | 'system' */}
  {children}
</AlquimiaUIProvider>
```

If you already use `next-themes` for other parts of your app, keep it — `AlquimiaUIProvider` is independent.

---

## Path B: Tools-only (`@alquimia-ai/tools` — custom UI)

### Dependencies

```bash
npm install @alquimia-ai/tools
```

No Tailwind config, no theme files, no CSS from the SDK. You build your own UI components and wire them to the `useAlquimia` hook return values.

### Environment Variables

Same for both paths:

```bash
# Server-side (never expose to browser)
ASSISTANT_BASEURL=https://your-backend.com
ALQUIMIA_ASSISTANT_API_KEY=your-secret-key

# Client-side (prefix with NEXT_PUBLIC_ for Next.js, VITE_ for Vite)
# Only needed for optional features:
NEXT_PUBLIC_ELEVENLABS_API_KEY=     # TTS/STT
NEXT_PUBLIC_ELEVENLABS_VOICE_ID=
NEXT_PUBLIC_ELEVENLABS_BASEURL=
```
