# Design Spec — Urdu ASR Realtime Transcription App

## Overview

A single `index.html` file — no build step, no dependencies, no framework. Opens by
double-clicking or via `python -m http.server`. Captures microphone audio, resamples to
16 kHz, and streams to the Nemotron backend over WebSocket. Transcript tokens appear in
real time as the model produces them.

---

## Architecture

Single file. All HTML, CSS, and JS live in `index.html`. JS is split into logical
sections via comments — not separate files. No module system.

```
index.html
  └─ <style>         — dark theme, layout
  └─ <body>          — static shell: header, status bar, controls, transcript zones
  └─ <script>
       ├─ CONFIG     — const BASE_URL, CHUNK_SAMPLES, CHUNK_MS, POLL_INTERVAL, POLL_TIMEOUT
       ├─ STATE      — wsState, isRecording, pollTimer, audioCtx, ws, partialText
       ├─ UI         — DOM refs + update functions (setStatus, setLang, appendHistory, setLive, showError)
       ├─ READY POLL — pollReady(), startPolling(), stopPolling()
       ├─ WEBSOCKET  — openWs(), sendConfig(), handleMessage(), closeWs()
       ├─ AUDIO      — startAudio(), buildWorkletBlobUrl(), sendChunk(), stopAudio()
       └─ EVENTS     — startBtn click, clearBtn click
```

---

## Components

### CONFIG constants

```js
const BASE_URL       = 'http://localhost:8000';
const CHUNK_SAMPLES  = 8960;   // 560 ms at 16 kHz
const CHUNK_MS       = 560;
const POLL_INTERVAL  = 2000;   // ms between /ready polls
const POLL_TIMEOUT   = 60000;  // ms before giving up
```

These are the only place backend coordinates appear. Changing the host/port means
editing one block.

### Ready Poller

Polls `GET /ready` every 2 s on page load. While polling: Start button disabled, status
shows "Loading model…". On success: enable Start, update status to "Ready". After 60 s
with no success: stop polling, show error banner with manual Retry button. Retry resets
the timer and starts polling again.

Also validates that `chunk_samples` and `sample_rate` from the `/ready` response match
the client's hardcoded constants; logs a console warning if they diverge.

### WebSocket session

**Open:** create `WebSocket(BASE_URL.replace('http','ws') + '/v1/transcriptions/stream')`.
Update status indicator to "Connecting".

**Config message (first text frame):** uses the current value of the language selector
(defaults to `"auto"`):
```json
{"language":"<langSelect.value>","sample_rate":16000,"chunk_ms":560,"use_vad":false,"format":"f32le"}
```

**`ready` event:** extract `language_name`, display in detected-language label. Status →
"Connected". Unblock audio pipeline.

**`partial` event:** append `event.text` to the live zone (incremental tokens).

**`final` event:** move live zone content into history zone; clear live zone. Stop audio
pipeline. Close WebSocket.

**`error` event:** show error banner with `event.message`. Stop audio. Close WebSocket.

**Close / disconnect:** stop audio, status → "Disconnected", show Retry option.
Retry re-runs the ready poller from scratch (does not auto-reconnect WebSocket).
The user must click Start again to begin a new session after reconnecting.

Do not send audio frames while `ws.readyState !== WebSocket.OPEN`.

**Stop sequence:** send `{"event":"end"}` as text, then wait — do NOT close the socket.
The server flushes its buffer and sends `final`; the handler above closes everything.

### Audio pipeline

**Startup:**
1. `navigator.mediaDevices.getUserMedia({audio:true, video:false})`
2. Create `AudioContext` — note its `sampleRate`
3. If `AudioWorkletNode` is available: build processor blob URL, call
   `audioCtx.audioWorklet.addModule(blobUrl)`, create node. Otherwise fall back to
   `ScriptProcessorNode` (bufferSize 4096) with a `console.warn`.
4. Connect: mic source → worklet/processor node → (no destination; node is a tap)

**Resampling:** if `audioCtx.sampleRate !== 16000`, resample each raw buffer through
`OfflineAudioContext` before pushing into the sample buffer.

**Buffering:** maintain a `Float32Array` ring buffer. When it reaches `CHUNK_SAMPLES`
(8960) samples, convert to `ArrayBuffer` (raw bytes of Float32LE) and send as a binary
WebSocket frame. Never pad or truncate — accumulate until exactly 8960.

**AudioWorklet processor (inline blob):**
```js
class PCMProcessor extends AudioWorkletProcessor {
  process(inputs) {
    const ch = inputs[0]?.[0];
    if (ch) this.port.postMessage(ch.slice());
    return true;
  }
}
registerProcessor('pcm-processor', PCMProcessor);
```

**Shutdown:** disconnect nodes, close `AudioContext`, release mic stream tracks. Always
runs on Stop, WebSocket error, or WebSocket close.

### UI shell

Dark theme (`#0f1117` background). Layout sections:

| Zone | Purpose |
|---|---|
| Header | App title + subtitle |
| Status bar | Connection pill (●) + detected language label |
| Controls | Start/Stop toggle · language `<select>` (default `auto`) · Clear button |
| History zone | Scrollable `<div>`, settled text appends here, persists across Start/Stop |
| Live zone | Single line, amber colour, shows current partial; clears on `final` |
| Error banner | Conditional, shown inline below live zone; never uses `alert()` |

**Start/Stop button states:**

| State | Label | Style |
|---|---|---|
| Idle / ready | "Start Recording" | Green |
| Recording | "Stop Recording" | Red + pulse dot |
| Waiting for final | "Stopping…" | Disabled grey |

**Connection status pill colours:**

| State | Colour |
|---|---|
| Disconnected / page load | Grey |
| Connecting | Amber |
| Connected | Green |
| Error / disconnected mid-session | Red |

### Error handling

| Error | Behaviour |
|---|---|
| Mic permission denied | Show inline banner: "Microphone access denied. Please allow mic access and retry." |
| Web Audio not supported | Show banner: "Your browser does not support Web Audio. Please use Chrome 120+ or Firefox 120+." Disable Start permanently. |
| WebSocket connection refused | Show banner: "Backend not running. Check that `docker compose up` is running." + Retry |
| `/ready` timeout (60 s) | Show banner: "Backend not responding after 60 s." + Retry |
| WebSocket closes mid-session | Stop audio. Status → Disconnected. Banner: "Connection lost." + Retry |
| `error` event from server | Show `event.message` verbatim in banner |
| Empty `partial` text | Skip — do not append blank strings to live zone |
| Empty transcript at session end | Show placeholder in history: "(no speech detected)" |

---

## Data Flow

```
Mic → AudioWorklet (PCMProcessor)
         │  Float32 samples (port.postMessage)
         ▼
    sample buffer (Float32Array)
         │  every 8960 samples
         ▼
    ArrayBuffer (raw f32le bytes)
         │  ws.send(binary)
         ▼
    WebSocket → backend
         │  {"event":"partial","text":"..."}
         ▼
    live zone (append tokens)
         │  {"event":"final","text":"..."}
         ▼
    history zone (absorb) + live zone (clear) + WS close
```

---

## Visual Design

Dark theme matching the approved mockup:
- Background: `#0f1117` / `#0a0d14` (alternating surfaces)
- Borders: `#1e2433`
- Body text: `#cbd5e1`
- Muted / labels: `#64748b`
- Live partial: `#fcd34d` (amber) on `#0d1220`
- Connected: `#22c55e` / `#86efac`
- Error: `#fca5a5` on `#1c0a0a`
- Font: system-ui stack, no external fonts

---

## Testing

Manual browser testing — no automated test framework.

**Golden path (Chrome + Firefox):**
1. Start Docker → open `index.html` → wait for "Ready" status
2. Click Start → speak → confirm amber partial tokens appear token-by-token
3. Click Stop → confirm live zone absorbed into history, live zone clears
4. Click Start again → confirm new text appends below previous session
5. Click Clear → both zones empty

**Error injection:**
- Deny mic → inline banner, no crash
- Open before Docker → 60 s polling, then error + Retry
- Stop Docker mid-session → Disconnected state, Retry offered
- Use unsupported browser → graceful message, Start disabled

---

## Out of Scope

- RTL layout or Nastaliq/Naskh fonts (see ADR-0001)
- `ur` language code (not in backend's language map — see ADR-0001)
- Client-side transliteration
- Copy/download/word-count features
- Automated tests
- Node.js proxy layer
- Multi-file split (unless file exceeds 800 lines)
