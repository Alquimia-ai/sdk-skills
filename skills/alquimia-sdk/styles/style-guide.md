# Style Guide

> This file is only relevant when using **Full Alquimia UI** (`@alquimia-ai/ui`).

Alquimia UI components use CSS custom properties (HSL values) and `alq--` prefixed utility classes.

Always use CSS variable tokens (`hsl(var(--primary))`) rather than hardcoded colors — this ensures theme consistency.

---

## Theme CSS Files (recommended)

Import one of the built-in themes in your global CSS or layout entry point:

```typescript
import '@alquimia-ai/ui/styles/themes/base.css';           // neutral light + dark
import '@alquimia-ai/ui/styles/themes/base-alquimia.css';  // alquimia brand
import '@alquimia-ai/ui/styles/themes/base-nordic.css';    // nordic theme
import '@alquimia-ai/ui/styles/themes/base-primary.css';   // primary color theme
```

Each theme defines all the CSS variables below for both `:root` (light) and `.dark` (dark mode). If you use a theme file, you do not need to define variables manually.

---

## CSS Variable Tokens Reference

Define these in your `:root` block **only if** you want a custom theme instead of importing a theme file:

```css
:root {
  /* Brand */
  --primary: 223 82% 60%;           /* main blue */
  --primary-foreground: 0 0% 98%;

  /* Surfaces */
  --background: 0 0% 95%;
  --foreground: 0 0% 3.9%;
  --card: 0 0% 100%;
  --card-foreground: 0 0% 3.9%;
  --muted: 0 0% 96.1%;
  --muted-foreground: 0 0% 45.1%;
  --accent: 0 0% 100%;
  --accent-foreground: 223 82% 60%;

  /* Semantic */
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 0 0% 98%;
  --border: 0 0% 85%;
  --input: 0 0% 89.8%;
  --ring: 221 83% 53%;

  /* Shape */
  --radius: 0.75rem;
}
```

---

## alq-- Class Reference

These classes should be applied via `className` on the corresponding SDK components.

### Chat Input

```css
/* Apply to <AssistantInput className="alq--assistant-input"> */
.alq--assistant-input {
  padding: 0.5rem;
  background: hsl(var(--card));
  border-radius: 1rem;
  border: 1px solid hsl(var(--border) / 0.6);
}

.alq--assistant-input textarea {
  background: hsl(var(--card));
  min-height: 60px;
  padding: 1rem 1rem 0;
  border: none;
  resize: none;
  overflow: auto;
}

.alq--assistant-input textarea::-webkit-scrollbar { width: 8px; border-radius: 8px; }
.alq--assistant-input textarea::-webkit-scrollbar-thumb { background: hsl(var(--border)); border-radius: 8px; }

.alq--assistant-actions {
  display: flex;
  width: 100%;
  padding: 0.5rem 0.5rem 0.5rem 0;
  background: hsl(var(--card));
}

/* Send button */
.alq--assistant-button-send { width: 2rem; height: 2rem; position: absolute; right: 0.375rem; bottom: 0; }
.alq--assistant-button-send svg { width: 1.25rem; height: 1.25rem; }
```

### Speech-to-Text button
```css
/* Apply to <SpeechToText className="alq--speech-to-text"> */
.alq--speech-to-text {
  position: absolute;
  right: 0.375rem;
  bottom: 0;
  z-index: 10;
}
```

### Message Bubbles

```css
/* Apply className="alq--prose" to <AssistantMessageArea> for markdown rendering */
.alq--prose { color: hsl(var(--foreground)); }
.alq--prose p { font-size: 0.875rem; line-height: 1.625; margin: 0.25rem 0; }
.alq--prose ul { list-style: disc inside; }
.alq--prose ol { list-style: decimal inside; }
.alq--prose pre { background: hsl(220 13% 18%); color: hsl(0 0% 95%); border-radius: 0.375rem; padding: 0.75rem; overflow-x: auto; }
.alq--prose code { font-family: monospace; font-size: 0.875rem; padding: 0.125rem 0.25rem; border-radius: 0.25rem; }
.alq--prose table { width: 100%; border-collapse: collapse; }
.alq--prose th { border: 1px solid hsl(var(--border)); padding: 0.5rem 1rem; font-weight: 600; }
.alq--prose td { border: 1px solid hsl(var(--border)); padding: 0.5rem 1rem; }

/* Assistant message container */
.alq--callout-message-container[data-role="assistant"] {
  font-size: 0.875rem;
  padding: 1rem;
  border-radius: 1rem;
  margin-right: 2.5rem;
  max-width: 47rem;
  background-color: hsl(var(--card) / 0.7);
  position: relative;
  overflow-wrap: break-word;
}

/* User message container */
.alq--callout-message-container[data-role="user"] {
  font-size: 0.875rem;
  padding: 1rem 1rem 2rem;
  border-radius: 1rem;
  margin-left: 2.5rem;
  background-color: hsl(var(--muted));
  position: relative;
}

/* Callout actions row */
.alq--callout-actions { display: flex; flex-direction: row; gap: 0.75rem; align-items: flex-end; height: 1.5rem; padding: 0 0.5rem; }
.alq--callout-action svg { width: 1rem; height: 1rem; color: hsl(var(--muted-foreground)); }
.alq--callout-action svg:active { transform: scale(0.85); }
.alq--callout-action[data-clicked="true"] svg { color: hsl(var(--primary)); }

/* Error message */
.alq--callout-response[data-error-code] {
  background: hsl(var(--destructive) / 0.1);
  border: 1px solid hsl(var(--destructive) / 0.3);
  color: hsl(var(--destructive));
  border-radius: 0.5rem;
  padding: 1rem;
}
```

### ThinkIndicator

```css
/* Apply className to the container wrapping ThinkIndicator */
.alq--think-indicator {
  font-size: 0.875rem;
  padding: 1rem;
  max-width: 47rem;
  border-radius: 1rem;
  background-color: hsl(var(--card) / 0.7);
  position: relative;
}

.alq--think-indicator-text {
  animation: shine 3s linear infinite;
  background: linear-gradient(90deg, hsl(var(--muted-foreground)), hsl(var(--foreground)), hsl(var(--muted-foreground)));
  background-size: 200% 100%;
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
```

### Whisper (TTS button)

```css
/* Apply className="alq--action-whisper" to <Whisper> */
.alq--action-whisper { margin-bottom: 1px; }
.alq--action-whisper button { padding: 0; margin-bottom: 0.2rem; height: 0.8rem; }
.alq--action-whisper svg { stroke-width: 1.5; }
```

### Dialogs (tool inputs, etc.)

```css
.alq--dialog-content { border-radius: 1.5rem !important; border: none; }
.alq--dialog-header { text-align: center; color: hsl(var(--foreground)); font-size: 1.25rem; }
.alq--dialog-title { text-align: center; color: hsl(var(--foreground)); padding-bottom: 0.75rem; font-size: 1.25rem; }

/* Darken the overlay */
[data-radix-dialog-overlay],
div[data-state][class*="bg-black/80"][aria-hidden="true"] {
  background-color: rgba(0, 0, 0, 0.4) !important;
}
```

### Buttons

```css
/* Primary ghost-style button */
.alq--button-primary {
  background: hsl(var(--primary) / 0.08);
  font-size: 0.875rem;
  color: hsl(var(--primary));
  border: 1px solid hsl(var(--primary) / 0.2);
  padding: 1.25rem;
  border-radius: 9999px;
}
.alq--button-primary:hover { background: hsl(var(--primary) / 0.12); }
```

### Badges (ToolboxTags)

```css
.alq--badge {
  display: inline-flex;
  align-items: center;
  padding: 0.5rem 1rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 400;
  background: transparent;
  border: 1px solid hsl(var(--primary) / 0.2);
  color: hsl(var(--muted-foreground));
}
```

### Popover (toolbox menu)

```css
.popover-element {
  color: hsl(var(--foreground));
  font-weight: 500;
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  justify-content: flex-start;
}
.popover-element:hover { background: hsl(var(--muted)); color: hsl(var(--primary)); }
.popover-element svg { color: hsl(var(--muted-foreground)); }
.popover-element:hover svg { color: hsl(var(--primary)); }
.popover-shadow { box-shadow: 0px 8px 20px 0px rgba(0, 0, 0, 0.06); }
```

### Toast

```css
.alq--toast { height: 60px; }
.alq--toast svg { color: white; }
.alq--toast-success { background: hsl(var(--primary)); color: white; border: none; }
.alq--toast-error { background: hsl(var(--destructive)); color: white; border: none; }
.alq--toast-action { border: 0; }
.alq--toast-action:hover { background: transparent !important; }
```

### Animations

```css
@keyframes shine {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
.animate-shine { animation: shine 3s linear infinite; }

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
.animate-fade-in { animation: fade-in 1s ease; }
```

---

## Ready-to-use CSS File

See `styles/chat.css` in this skill for a complete stylesheet with all `alq--` classes. Copy its contents into your own global CSS file for the default component look.

---

## How to Apply Styles

1. Import a theme file **or** define custom CSS variables (see above)
2. Copy `styles/chat.css` contents into your global CSS file for `alq--` component classes
3. Apply `className` on SDK components:

```tsx
<AssistantMessageArea className="space-y-4 px-10 alq--prose" ... />
<AssistantInput className="alq--assistant-input" ... />
<SpeechToText className="alq--speech-to-text" ... />
<Whisper className="alq--action-whisper" ... />
```
