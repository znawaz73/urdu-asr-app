# PRD — Urdu Realtime ASR Browser App

## Problem Statement

Evaluating a speech recognition model's quality on Urdu audio requires a realistic
streaming scenario. Uploading audio files one at a time is too slow and does not reflect
real-world usage. The developer needs to speak naturally into a microphone and see
transcribed output appear in real time, without any setup friction.

## Solution

A single-file browser app (`index.html`) that captures microphone audio, resamples it
to 16 kHz, and streams it to the Nemotron backend over WebSocket. Transcript tokens
appear in real time as the model produces them. The app requires no install, no build
step, and no dependencies beyond a modern browser.

## User Stories

1. As a developer, I want to open `index.html` directly in a browser without any build or install step, so that I can start evaluating the model immediately.
2. As a developer, I want the app to check whether the backend is ready before I can start recording, so that I am not confused by a silent failure if the model is still loading.
3. As a developer, I want to see a "Loading model…" status while the backend is starting up, so that I know the app is working and just waiting.
4. As a developer, I want the app to stop waiting after 60 seconds and show a "Backend not responding" message with a Retry button, so that I can tell if Docker has crashed rather than staring at a spinner indefinitely.
5. As a developer, I want to click a single "Start" button to begin recording, so that the interaction is as simple as possible.
6. As a developer, I want the app to request microphone permission and show a clear inline message if I deny it, so that I understand what went wrong without the app crashing.
7. As a developer, I want the app to show a clear message if my browser does not support the Web Audio API, so that I can switch to a supported browser.
8. As a developer, I want the app to connect to the WebSocket endpoint and send a config message with `language: "auto"` before streaming any audio, so that the model correctly initialises the session.
9. As a developer, I want the app to resample my microphone audio to 16 kHz in the browser if my mic runs at a different sample rate, so that the server does not reject the audio.
10. As a developer, I want audio to be sent in chunks of exactly 8960 samples (560 ms) as the server expects, so that transcription is accurate and the buffer is not corrupted.
11. As a developer, I want to see transcript tokens appear in a live line at the bottom of the display as each `partial` event arrives, so that I can watch the model transcribe in real time.
12. As a developer, I want the live partial line to be visually distinct from the settled history above it, so that I can tell what is in-progress versus committed.
13. As a developer, I want the live partial line to be absorbed into the settled history when I click "Stop", so that the final text is preserved without duplication.
14. As a developer, I want clicking "Stop" to send `{"event": "end"}` to the server and wait for the `final` event before closing the WebSocket, so that any audio buffered at the end of my speech is not lost.
15. As a developer, I want the WebSocket to close and the mic to stop cleanly after the `final` event is received, so that I am not leaving resources open silently.
16. As a developer, I want the transcript to accumulate across multiple Start/Stop cycles within the same session, so that I can record in bursts and review a continuous output.
17. As a developer, I want a "Clear" button to reset the entire transcript display, so that I can start a fresh evaluation without reloading the page.
18. As a developer, I want to see the language the model detected (the `language_name` field from the `ready` event) displayed as a label in the UI, so that I know whether the model is treating my speech as Hindi, Arabic, or something else.
19. As a developer, I want a connection status indicator showing connected / connecting / disconnected, so that I can tell whether the WebSocket is live at a glance.
20. As a developer, I want to see a "Backend not running" message with a Retry button if the WebSocket connection is refused, so that I know to check whether Docker is up.
21. As a developer, I want the app to stop recording and show a "Disconnected" state with a Retry option if the WebSocket drops mid-session, so that I am not left with the mic open and no output.
22. As a developer, I want the transcript area to show a placeholder when no text has been produced yet, so that the empty state is clearly intentional rather than a bug.
23. As a developer, I want the app to work identically in Chrome 120+ and Firefox 120+, so that I can demo it on either browser without modification.
24. As a developer, I want to be able to change the backend URL by editing a single `const BASE_URL` at the top of the file, so that I can point the app at a different host or port without hunting through the code.

## Implementation Decisions

- **Single file**: everything — HTML, CSS, JS — lives in `index.html`. No build step, no
  npm, no framework. The file must open by double-clicking or via `python -m http.server`.
  Split only if the file exceeds 800 lines, and only with explicit approval.

- **Backend URL**: `const BASE_URL = 'http://localhost:8000'` declared at the top of the
  inline script. The WebSocket URL is derived from it (`BASE_URL.replace('http', 'ws') +
  '/v1/transcriptions/stream'`). No visible URL input in the UI.

- **Readiness polling**: on page load, poll `GET /ready` every 2 seconds. After 30 failed
  attempts (60 seconds total), stop polling and show an error with a manual Retry button.
  While polling, show "Loading model…" and disable the Start button.

- **WebSocket session lifecycle**:
  1. Open WebSocket to `ws://localhost:8000/v1/transcriptions/stream`
  2. Send JSON config as first message: `{"language":"auto","sample_rate":16000,"chunk_ms":560,"use_vad":false,"format":"f32le"}`
  3. Wait for `{"event":"ready",...}` — capture `language_name` for the UI label
  4. Begin streaming binary PCM chunks (Float32LE, 8960 samples = 35840 bytes each)
  5. On Stop: send `{"event":"end"}`, wait for `{"event":"final"}`, then close WebSocket and stop mic
  6. On error event or WebSocket close: stop mic, update UI state, offer Retry

- **Audio pipeline**:
  - Capture mic via `navigator.mediaDevices.getUserMedia({audio: true})`
  - Prefer `AudioWorkletNode`; register the processor by creating a `Blob` from the
    processor source string and passing `URL.createObjectURL(blob)` to
    `audioContext.audioWorklet.addModule()`. Fall back to `ScriptProcessorNode` with a
    console warning if `AudioWorkletNode` is unavailable.
  - If the mic sample rate differs from 16000 Hz, resample each chunk using
    `OfflineAudioContext` before buffering. Never send audio at the wrong sample rate.
  - Buffer incoming Float32 samples; emit a binary chunk to the WebSocket only when the
    buffer reaches exactly 8960 samples.
  - Do not send audio while the WebSocket is in `CONNECTING` or `CLOSED` state.

- **Transcript display — two zones**:
  - *History zone*: scrollable area of settled text. Each completed session's output
    appends here. Text accumulates across Start/Stop cycles; only "Clear" resets it.
  - *Live zone*: single line below history. Each `partial` event appends its incremental
    tokens here. On `final`, the live zone content is appended to history and the live
    zone clears. While not recording, the live zone is empty.

- **Language**: always send `language: "auto"`. The backend's language map does not
  include `ur`; attempting it triggers an error event. Display the `language_name`
  returned in the `ready` event as an informational label — do not filter or override it.
  See ADR-0001.

- **Script and layout**: no RTL enforcement, no Nastaliq/Naskh fonts. Display model
  output in a standard system font left-to-right. See ADR-0001.

- **Error display**: all errors shown as inline UI messages. No `alert()`. Each error
  state has a specific message (see §8 of CLAUDE.md). Retry buttons where applicable.

## Testing Decisions

Manual browser testing is the only seam available — the project has no test framework
and introducing one is out of scope. A good test observes external behavior (what appears
in the UI, what network traffic is sent) rather than implementation details (which
functions were called).

**Seam 1 — Golden path**
Load `index.html` in Chrome 120+ and Firefox 120+ with the backend running at
`localhost:8000`. Speak Urdu (or any language). Verify:
- Live partial tokens appear token-by-token in the live zone during speech
- Clicking Stop absorbs the live zone into history and clears the live zone
- A second Start/Stop cycle appends new text below the first session's output
- The detected language label updates after each session's `ready` event
- "Clear" empties both zones

**Seam 2 — Error injection**
Manually trigger each error state from §8 and verify the specified UI response:
- Deny mic permission → inline message, no crash
- Open `index.html` before `docker compose up` → "Backend not responding" + Retry after 60 s
- Stop Docker mid-session → "Disconnected" state, mic stops, Retry offered
- Use a browser without Web Audio API support → graceful message
- Start app while model is loading (backend up but `/ready` returns 503) → "Loading model…" polling state

## Out of Scope

- Any modification to the Nemotron backend, Docker configuration, or `config.yaml`
- Node.js/Express proxy layer
- Copy to clipboard, download as `.txt`, RTF meter, word count
- Animations, gradients, or decorative UI elements
- Automated unit or integration tests
- Multi-user support or concurrent stream handling
- Urdu script (Nastaliq/Naskh) rendering or RTL layout
- Client-side transliteration of model output to Roman/Latin script
- Any language other than `auto` in the initial config message

## Further Notes

The backend holds a global async lock — only one WebSocket stream is active at a time.
If a second client connects while a stream is live, the server returns
`{"event": "error", "message": "..."}`. The app should display this error inline and
offer a Retry, treating it the same as any other error event.

The `/ready` endpoint also returns `chunk_samples` and `sample_rate` in its response.
These can be used to confirm the client's hardcoded values (8960 and 16000) match the
server's actual configuration, and to surface a warning if they diverge.
