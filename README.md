# claude-better-voice-mode

A hands-free, interruptible voice interface for Claude that runs in the browser.

> **Status:** planning. This README is the plan; no code yet.

## Goals

| Feature | What it means for the user |
|---|---|
| **Interruptible (barge-in)** | Start talking while Claude is speaking and it stops right away, listens, and answers the new thing. |
| **Hands-free** | No push-to-talk. The app always listens, works out when you've finished a sentence, and replies. |
| **Voice choice** | Pick from several natural voices and set the speaking speed. The choice is remembered. |
| **Transcript + memory** | Every conversation is saved as text you can scroll, search and copy. Claude remembers key facts across sessions. |

### Non-goals (for v1)

- Native mobile apps (the web app should still work well in mobile browsers)
- Multi-user accounts and sharing
- Phone calls / telephony

## How it works

```
 Browser                                         Server (Node)
┌──────────────────────────────────────┐        ┌──────────────────────────────┐
│ Mic ─► VAD (speech start/end)        │        │                              │
│          │                           │  WS    │  STT stream ─► final text    │
│          ├─► audio chunks ───────────┼───────►│                 │            │
│          │                           │        │                 ▼            │
│          └─► "speech started" ───────┼───────►│  Claude API (streaming)      │
│               (barge-in signal)      │        │                 │ sentences   │
│                                      │        │                 ▼            │
│ Speaker ◄── audio queue ◄────────────┼◄───────│  TTS stream (chosen voice)   │
│                                      │        │                              │
│ Transcript UI ◄── text events ───────┼◄───────│  Store: transcripts + memory │
└──────────────────────────────────────┘        └──────────────────────────────┘
```

The API keys stay on the server and the browser never sees them.

### 1. Hands-free listening
- **Voice activity detection (VAD)** runs in the browser (Silero VAD via `@ricky0123/vad-web`) to detect when speech starts and ends.
- **End of turn** fires after about 700 ms of silence. That gap can be changed in settings, so slow talkers aren't cut off.
- **Speech-to-text** uses a streaming provider such as Deepgram, which returns partial text live. A Whisper-based option can come later.
- Echo cancellation (`getUserMedia({ echoCancellation: true })`) keeps Claude's own voice from triggering the mic.

### 2. Interruptions (barge-in)
When VAD detects speech **while Claude is talking**:
1. Stop audio playback immediately and clear the audio queue.
2. Cancel the in-flight Claude stream and TTS request.
3. Save only the part of the reply that was **actually spoken** to history, marked as `[interrupted]`, so Claude knows what you heard.
4. Start a new turn with what you say next.

To avoid false triggers (a cough, background noise), barge-in only fires after about 250 ms of continuous speech.

### 3. Low-latency replies
- Stream Claude's response and split it into sentences as they arrive.
- Send each sentence to TTS as soon as it's complete, so speaking starts after the first sentence instead of after the whole reply.
- The system prompt asks for spoken-style answers: short, no markdown, no bullet lists, and no code blocks read aloud. Code goes to the transcript only.
- Target: **under 1.5 s** from when you stop talking to when you hear the first word.

### 4. Voice choice
- Use a TTS provider with a voice catalog, such as ElevenLabs or OpenAI TTS. The browser's built-in `speechSynthesis` is a free fallback.
- The settings panel offers a voice picker with a preview button and a speed slider (0.75×–1.5×).

### 5. Transcript + memory
- **Transcript:** every turn is saved with a timestamp and an interrupted flag. It can be searched and exported as Markdown.
- **Memory:** when a session ends, Claude summarizes it into a short list of durable facts and preferences, such as "prefers short answers" or "working on the ezpass project".
- Those facts go into the system prompt of future sessions. A **Memory** screen lets you view, edit or delete any fact.
- Storage starts with SQLite on the server and can move to Supabase later.

## Tech stack

| Layer | Choice |
|---|---|
| Frontend | React + Vite + TypeScript |
| Backend | Node + TypeScript, WebSocket (`ws`) |
| LLM | Claude API via `@anthropic-ai/sdk`, streaming; model is configurable |
| VAD | `@ricky0123/vad-web` (Silero) |
| STT | Deepgram streaming (pluggable interface) |
| TTS | ElevenLabs or OpenAI TTS (pluggable), browser fallback |
| Storage | SQLite (`better-sqlite3`) |

STT and TTS each sit behind a small interface so providers can be swapped without touching the rest of the app.

## Milestones

1. **Skeleton:** Vite app, Node WebSocket server, `.env` handling, mic permission.
2. **Talk loop:** a push-to-talk button, then STT, Claude and TTS, to prove the pipeline end to end.
3. **Hands-free:** add VAD and end-of-turn detection, and remove the button.
4. **Barge-in:** cancel playback and streams, save partial replies, and tune false-trigger handling.
5. **Streaming speech:** sentence-chunked TTS and a latency check against the 1.5 s target.
6. **Voices:** a settings panel with voice picker, preview and speed.
7. **Transcript:** a saved, searchable conversation view with Markdown export.
8. **Memory:** end-of-session summaries, injection into the system prompt, and an editable Memory screen.
9. **Polish:** mobile browser testing, error states (mic denied, network drop), and a visual indicator for listening, thinking and speaking.

## Configuration (planned `.env`)

```
ANTHROPIC_API_KEY=
CLAUDE_MODEL=
DEEPGRAM_API_KEY=
TTS_PROVIDER=elevenlabs   # elevenlabs | openai | browser
ELEVENLABS_API_KEY=
OPENAI_API_KEY=
```

## Open questions

- Which TTS provider do you prefer? This choice affects cost and voice quality the most.
- Should it run only locally on your computer, or be deployed somewhere so you can use it from your phone?
- Should memory be shared with the `ezpass` project or kept separate?
