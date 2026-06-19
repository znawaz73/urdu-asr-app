# CLAUDE.md — Urdu Realtime ASR Browser App

> Project-specific instructions for Claude Code. Inherits all global operating rules
> (PRD-first, anti-sycophancy, reversibility protocol). This file adds context,
> constraints, and conventions specific to this project.

---

## 1. Project Context

**What this is:** A browser-based, realtime speech transcription app for Urdu audio.
The backend is already running — a Dockerised FastAPI + ONNX Runtime server wrapping
`onnx-community/nemotron-3.5-asr-streaming-0.6b-onnx-int4`.

**Owner:** Zubair Nawaz — Senior AI Consultant, Tkxel, Lahore, Pakistan.

**Why this exists:** Evaluate Nemotron's Urdu ASR quality in a realistic streaming
scenario, and produce a lightweight app that can be demoed or extended for client work.

---

## 2. Architecture

```
Browser (Mic)
  └─ Web Audio API → ScriptProcessorNode / AudioWorkletNode
       └─ Float32 PCM chunks (mono, 16 kHz, little-endian)
            └─ WebSocket → ws://localhost:8000/v1/transcriptions/stream
                 └─ Nemotron Docker container
                      └─ Transcript tokens → streamed back to UI
```

**Key backend facts (do not re-invent or re-derive):**

| Property | Value |
|---|---|
| WebSocket endpoint | `ws://localhost:8000/v1/transcriptions/stream` |
| HTTP transcription | `POST http://localhost:8000/v1/transcriptions` |
| Health check | `GET http://localhost:8000/health` |
| Ready check | `GET http://localhost:8000/ready` |
| Required audio format | Mono, Float32, little-endian PCM |
| Model sample rate | 16000 Hz |
| Default chunk size | 8960 samples (~560 ms) |
| Default language | `auto` (`ur` is not supported by the model) |
| Supported language list | `GET http://localhost:8000/v1/languages` |
| WS config message | First message must be JSON: `{"language":"auto","sample_rate":16000,"chunk_ms":560,"use_vad":false,"format":"f32le"}` — `use_vad` is always `false` |
| WS events (server→client) | `ready` · `partial` (incremental tokens, append) · `final` (complete text) · `error` |
| WS end session | Client sends `{"event":"end"}` → server flushes buffer → sends `final` |

The backend is **not your concern** — do not modify Docker files, `config.yaml`,
or server-side code unless explicitly asked.

---

## 3. Tech Stack — Frontend

- **Pure HTML + CSS + Vanilla JS** — single `index.html` file, no build step, no framework.
  The app must open by double-clicking the file or serving with `python -m http.server`.
- **Web Audio API** for microphone capture and resampling to 16 kHz.
- **WebSocket API** (native browser) for streaming to the backend.
- **No npm, no webpack, no React, no TypeScript** unless explicitly asked to introduce them.
- **No external CDN dependencies** unless they are truly unavoidable and pinned to a
  specific version URL.

Rationale: keep it zero-dependency so it runs anywhere without setup friction.

---

## 4. PRD-First Rule (project-specific application)

Before writing any code, produce a mini-PRD covering:

1. **What the user interaction looks like** — button states, UI flow, error states
2. **Audio pipeline design** — how mic → 16 kHz float32 → WebSocket chunks will work
3. **Urdu text rendering** — RTL layout, font choice, display direction
4. **Failure modes** — mic denied, WebSocket drop, model not ready, unsupported browser
5. **Open questions** — anything that needs a decision before building

Wait for explicit "go" before writing code.

---

## 5. Language and Script Requirements

- **`ur` is not in the backend's `LANG_TO_ID` map.** Sending `language: "ur"` will
  return `{"event": "error"}`. Do not attempt it.
- **Supported language selector options:** `auto` (default) and `en` (English). `en` is
  in the backend's `LANG_TO_ID` map and is safe to send. Do not add other language codes
  without verifying them against `LANG_TO_ID` first.
- **Default language: `auto`** — the model detects the language from speech.
- **Display the model's raw output as-is.** Do not enforce RTL layout, do not apply
  Nastaliq fonts. If the model produces Devanagari for Urdu speech, display Devanagari.
  If it produces Arabic script, display Arabic script. Do not post-process or transliterate.
- Show the detected language returned in the `ready` event (`language_name` field) as a
  label in the UI, so the user knows what the model is treating the input as.
- Standard system font stack is fine — no special Urdu font required.

---

## 6. Audio Pipeline Rules

- **Resample in the browser** if the user's mic is not at 16 kHz. Use `OfflineAudioContext`
  for resampling — do not send wrong sample-rate audio to the server.
- **Chunk size:** send chunks of exactly 8960 samples (matching `chunk_samples` from the
  server). Buffer if necessary before sending.
- **Use `AudioWorkletNode` via Blob URL** — encode the processor as a string, create
  a `Blob`, and pass `URL.createObjectURL(blob)` to `audioContext.audioWorklet.addModule()`.
  Fall back to `ScriptProcessorNode` only if `AudioWorkletNode` is unavailable, with a
  console warning.
- Do not send audio while the WebSocket is in `CONNECTING` or `CLOSED` state.
- **Stop sequence:** send `{"event": "end"}` as text, wait for `{"event": "final"}`,
  then close the WebSocket and stop the mic. Do not hard-close before receiving `final` —
  buffered audio at the end of an utterance will be lost.
- On WebSocket close or error, stop the mic stream and update UI state — never leave
  the mic open silently.

---

## 7. UI/UX Constraints

- **Minimal, functional UI.** Not a design showcase — a working demo tool.
- Required controls:
  - Start / Stop recording button (single toggle, clear state label)
  - Language selector (default: `auto`)
  - Connection status indicator (connected / connecting / disconnected)
  - Transcript display area: two zones — a scrollable history of settled text above,
    and a single live line below showing the current partial (clears on each new Start)
  - Clear transcript button
- Optional nice-to-have (only if explicitly requested):
  - Copy transcript to clipboard
  - Download transcript as `.txt`
  - RTF meter or word count
- Do not add animations, gradients, or decorative elements unless asked.

---

## 8. Error Handling — Mandatory Coverage

Every code path must handle these explicitly (not with generic `catch` and silence):

| Error | Expected behaviour |
|---|---|
| Mic permission denied | Show clear message, do not crash |
| Browser doesn't support WebAudio | Alert and graceful exit |
| WebSocket connection refused | Show "Backend not running" message, retry button |
| Backend `/ready` returns not-ready | Poll every 2 s; after 60 s show "Backend not responding" + manual Retry button |
| WebSocket closes mid-session | Stop recording, show disconnected state, offer reconnect |
| Empty transcript returned | Show placeholder, not blank space |

---

## 9. File Structure

```
urdu-asr-app/
├── index.html          # everything — HTML + CSS + JS in one file
├── CLAUDE.md           # this file
└── README.md           # how to run (auto-generate at end)
```

Do not split into multiple files unless the single-file approach becomes genuinely
unmanageable (>800 lines). If splitting becomes necessary, ask first.

---

## 10. Testing Approach

Before marking any feature complete:

1. Test with the backend running (`docker compose up`) at `localhost:8000`
2. Test mic permission grant AND deny flows
3. Test with actual Urdu speech — not just English
4. Check RTL rendering in both Chrome and Firefox
5. Verify WebSocket reconnect behaviour by stopping and restarting Docker mid-session

Do not claim "done" based on code inspection alone.

---

## 11. What Not To Do

- Do not modify any files in the `nemotron-speech-docker` repo.
- Do not add a Node.js/Express proxy layer — connect directly to the Docker container.
- Do not use `alert()` for error handling — use in-page UI messages.
- `const BASE_URL = 'http://localhost:8000'` at the top of the JS is the configuration
  point. Do not add a visible URL input — this is a single-developer tool.
- Transcript accumulates across Start/Stop cycles within a session. Clicking Start again
  appends new text below the previous session. Only the "Clear" button resets the display.
- Do not stream raw microphone audio without resampling — the server rejects wrong rates.
- Do not use deprecated `getUserMedia` on `navigator` directly — use
  `navigator.mediaDevices.getUserMedia`.

---

## 12. Definition of Done

The app is done when:

- [ ] Mic opens, audio streams to WebSocket at correct format and chunk size
- [ ] Transcript appears in real time as speech is detected
- [ ] Detected language label (`language_name` from `ready` event) is shown in the UI
- [ ] All error states in §8 are covered
- [ ] Works in Chrome 120+ and Firefox 120+ without modification
- [ ] Single `index.html` file, opens without a build step
- [ ] README.md explains how to run the backend and open the app

---

## 13. General Coding Principles

Behavioral guidelines that apply across all sessions. These complement the project-specific rules above; where they conflict, the project-specific rules (§1–§12) take precedence.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

### Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

This extends §4 (PRD-First Rule): surface open questions before writing a single line.

### Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked. (See §7 — optional features only on explicit request.)
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios. (§8 already enumerates the real ones.)
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

### Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

Every changed line should trace directly to the user's request.

### Goal-Driven Execution

**Define success criteria. Verify before declaring done.**

Transform tasks into verifiable goals. Because this project has no automated test framework (§3 — pure HTML/JS, no build step), "verify" means manual browser testing per §10:

- "Add validation" → define the invalid input, test it in the browser manually
- "Fix a bug" → reproduce it first, then confirm it's gone after the fix
- "Refactor X" → confirm all §12 acceptance criteria still pass in the browser

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria allow independent iteration. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
