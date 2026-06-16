# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Teraani Widget is a **pure static HTML/CSS/JavaScript** chatbot widget for Teraani Vistos, a Brazilian visa consulting service (US B1/B2, DS-160, Canadian visas). All content is in Brazilian Portuguese (pt-BR).

There is no build system, package manager, bundler, or transpilation step. Files are served as-is.

## Development

**Serve locally** (any static file server works):
```bash
python3 -m http.server 8080
# or
npx serve .
```

There are no tests, no linters, and no CI configuration.

## File Map

| File | Purpose |
|---|---|
| `index.html` | Main landing page + full embedded chat widget (~962 lines) |
| `widget-embed.html` | Standalone embeddable widget for third-party sites via `<iframe>` |
| `pacotes-teraani.html` | Service pricing/packages page (3-tier, prices in BRL) |
| `botao-teraani.html` | Minimal button component that links to the main widget |

Each file is **self-contained**: HTML, CSS (in `<style>`), and JavaScript (in `<script>`) are all inline. There are no external JS files or shared stylesheets.

## Architecture

### Chat Widget Logic (index.html and widget-embed.html)

The `CONFIG` object at the top of each `<script>` block is the single customization point:

```js
const CONFIG = {
  webhookUrl: 'http://129.148.30.27:5678/webhook/teraani-chat', // n8n webhook
  useFallback: true,   // set false once webhook is stable in production
  welcomeMessage: '...',
  quickReplies: ['Visto americano B1/B2', 'DS-160 formulário', ...],
};
```

Runtime state is kept in module-level variables: `sessionId`, `isOpen`, `isTyping`, `messageHistory`. There is no state management library.

**Message flow:**
1. User types → `sendMessage()` → `sendUserMessage(text)`
2. `sendUserMessage` calls `addUserMessage()` (DOM), shows typing indicator, then calls the n8n webhook via `fetch()`
3. Webhook response fields checked in order: `reply`, `message`, `text`, `output`
4. On failure or when `useFallback: true`, `getDemoReply()` returns a hardcoded Portuguese response
5. `addBotMessage(text, quickReplies?)` renders the reply and optional quick-reply buttons

**Key functions:** `toggleChat()`, `openChat()`, `showWelcome()`, `addBotMessage()`, `addUserMessage()`, `sendUserMessage()`, `formatText()` (basic markdown: `**bold**`, `\n` → `<br>`).

### Design System

All files share the same CSS custom properties in `:root`:

```css
--navy: #0d1b2a        /* primary background */
--navy-mid: #1a2e45    /* secondary background */
--gold: #c9a84c        /* brand accent / CTAs */
--gold-light: #e8c97a  /* hover states */
--cream: #f5f0e8       /* light background variant */
--gray: #8a9bb0        /* muted text */
--radius: 18px
--shadow: 0 20px 60px rgba(13,27,42,0.25)
```

Typography: **Playfair Display** (headings, loaded from Google Fonts) + **DM Sans** (body).

### Backend Integration

- **Backend:** n8n (self-hosted workflow automation) on Oracle Cloud VM
- **Webhook:** `http://129.148.30.27:5678/webhook/teraani-chat`
- **Payload sent:** `{ message: string, sessionId: string, history: array }`
- **Expected response:** JSON with one of: `reply`, `message`, `text`, or `output`
- The fallback demo (`getDemoReply()`) is Portuguese and keyword-based; it should be removed (`useFallback: false`) once n8n is stable

### Embedding

`widget-embed.html` is designed to be embedded via `<iframe>` on external pages. It has `background: transparent` on `html, body` so it blends with the host page. `botao-teraani.html` provides the triggering button that opens the iframe/widget.
