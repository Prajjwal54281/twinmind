# TwinMind — Live Meeting Copilot

A real-time AI meeting copilot that listens to live audio, transcribes it, and continuously surfaces 3 useful suggestions based on what is being said. Clicking a suggestion streams a detailed, transcript-grounded answer into a chat panel.

**Live demo:** `https://your-netlify-url.netlify.app`  
**Assignment:** TwinMind Live Suggestions — April 2026

---

## Setup

### 1. Get a Groq API key
- Sign up at [console.groq.com](https://console.groq.com)
- Go to **API Keys** → **Create API Key**
- Copy the key (starts with `gsk_...`)

### 2. Run the app
No build step required. Open `index.html` directly in Chrome:
- Double-click the file, or
- Deploy to Netlify/Vercel and open the URL

### 3. Paste your API key
- Click **Settings** (top right)
- Paste your Groq key into the **API Key** field
- Click **Save**

The key is stored in your browser's `localStorage` only. It is never sent anywhere except directly to Groq's API as an `Authorization: Bearer` header.

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | Single `index.html` | Zero build step, deploy anywhere, easy to audit |
| UI | React 18 via ESM importmap | No bundler needed, fast to load |
| Transcription | Groq Whisper Large V3 | Best-in-class accuracy, fastest inference available |
| LLM | Groq `openai/gpt-oss-120b` | Required by assignment — 120B parameter model, strong reasoning |
| Streaming | SSE via fetch + ReadableStream | Low latency first token, native browser support |
| Storage | `localStorage` for settings only | Transcript and chat stay in React state — no persistence needed |

---

## Architecture

```
Browser
├── MediaRecorder (1s slices → chunksRef[])
│   └── Every ~30s → Blob → Whisper Large V3
│                          → transcriptChunks[]
│                          → openai/gpt-oss-120b (suggestions)
│                          → SuggestionsPanel (3 cards)
│
├── Suggestion click → openai/gpt-oss-120b (streaming detail)
│                    → ChatPanel
│
└── Free chat input → openai/gpt-oss-120b (streaming)
                    → ChatPanel
```

**Key implementation details:**
- `chunksRef` accumulates 1s audio blobs from MediaRecorder
- `transcribeAndSuggest()` flushes the buffer → Whisper → appends chunk → generates suggestions
- `refreshBusyRef` and `suggestionBusyRef` prevent re-entrant API calls
- `groqChatStream()` returns an async generator consumed with `for await`
- Three separate context windows: suggestions (1500 chars), click detail (2500 chars), chat (3000 chars)

---

## Prompt Strategy

This is the core of the product. The goal was to make suggestions feel like they came from a smart human observer who was paying close attention — not a generic AI.

### Live Suggestions Prompt

**What context I pass:**
- The last N characters of transcript (default 1500 chars — roughly 2-3 minutes of speech)
- Passed as `[timestamp] text` formatted lines so the model can see recency
- Wrapped in a system prompt that defines the 5 suggestion types and the rules for choosing between them

**How I structure it:**
The system prompt defines the types (QUESTION, ANSWER, FACTCHECK, TALKINGPOINT, CLARIFY) and then gives explicit rules for *when* to use each one:
- If someone just asked a question → surface an ANSWER
- If a dubious claim was made → surface a FACTCHECK  
- If a decision point is approaching → surface a TALKINGPOINT
- Mix types — never give 3 QUESTIONs when the conversation needs something else

The key insight is that the type selection *is* the intelligence. A model that always returns 3 QUESTIONs is useless. The prompt forces the model to read what just happened and respond to it.

**Why 1500 chars for suggestions:**
Enough to capture the last 2-3 minutes of conversation (the relevant context) without overwhelming the model with older, less relevant content. Suggestions should reflect *right now*, not the whole meeting.

**Temperature: 0.45**
Low enough to stay grounded in what was actually said, high enough to generate varied and non-obvious suggestions.

### Click Detail Prompt

**What context I pass:**
- Last 2500 chars of transcript (more than suggestions — deeper answer needs more context)
- The suggestion type, preview, and detail from the card that was clicked
- A system prompt that instructs the model to cover: what was said that makes this relevant, the key insight, caveats, and one concrete next step

**Why a separate context window:**
Expanded answers need more transcript context than suggestions. A suggestion needs to be timely (recent context). An expanded answer needs to be thorough (broader context). Merging these into one setting would force a tradeoff.

### Chat Prompt

**What context I pass:**
- Last 3000 chars of transcript as system context
- Last 16 prior chat messages for conversation continuity
- System prompt that instructs the model to ground every answer in what was actually said

**Why 3000 chars for chat:**
Free chat questions can be about anything in the meeting, not just the last 2 minutes. Broader context window gives better answers to questions like "what did we decide about X earlier?"

---

## Tradeoffs

**Single file vs. component architecture**
Chose single file per the assignment spec. The tradeoff is that the file is long (~600 lines) but everything is in one place, easy to audit, and zero build complexity. For a production codebase this would be split into components.

**No partial transcription display**
Whisper requires a complete audio blob — you can't stream partial transcription. The "Listening..." hint tells users audio is being captured. A production version might use a faster, lower-quality model for partial display and Whisper for the committed chunk.

**Context window as character count vs. token count**
Used character count for simplicity (no tokenizer in the browser). At ~4 chars/token, 1500 chars ≈ 375 tokens — well within limits and predictable for the user to reason about in Settings.

**localStorage for settings only**
Transcript and chat are intentionally not persisted. The assignment says no data persistence is needed on reload. Keeping them in React state keeps the implementation simple and avoids stale data issues across sessions.

**Auto-refresh vs. VAD (Voice Activity Detection)**
Could have used VAD to trigger transcription only when speech is detected, reducing API calls. Chose fixed 30s interval for simplicity and predictability — the evaluator can reason about when refreshes happen.

---

## Features

- 🎙️ **Mic recording** with start/stop, 1s audio slices
- 📝 **Live transcript** with timestamps, auto-scroll
- 💡 **3 suggestions per batch**, newest on top, older batches stay visible
- 🃏 **5 suggestion types**: QUESTION, ANSWER, FACTCHECK, TALKINGPOINT, CLARIFY
- ⏱️ **Countdown bar** showing time to next auto-refresh
- 🔄 **Manual refresh** button that flushes audio buffer immediately
- 💬 **Streaming chat** with suggestion-triggered and free-form messages
- 📤 **Export** full session as JSON (transcript + suggestions + chat + timestamps)
- ⚙️ **Settings modal** with all prompts and context windows editable
- 🔑 **API key** entered at runtime, stored in localStorage only

---

## Project Structure

```
index.html          — entire application (HTML + CSS + JS)
README.md           — this file
```
