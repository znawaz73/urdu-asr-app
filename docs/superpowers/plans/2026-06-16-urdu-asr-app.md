# Urdu ASR App — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `index.html` — a single-file, zero-dependency browser app that streams microphone audio to the Nemotron ASR backend and displays transcript tokens in real time.

**Architecture:** Single `index.html` with inline CSS and JS. JS is organized into sections via comments: CONFIG → STATE → UI → READY POLL → WEBSOCKET → AUDIO → EVENTS. No build step, no framework, no CDN.

**Tech Stack:** HTML5, CSS3, Vanilla JS — Web Audio API (AudioWorklet + ScriptProcessor fallback), WebSocket API (native browser), OfflineAudioContext for resampling to 16 kHz.

---

## File Structure

| File | Action | Responsibility |
|---|---|---|
| `index.html` | Create | Everything: HTML shell, dark-theme CSS, all JS sections |
| `README.md` | Create | How to start Docker + open the app |

---

### Task 1: HTML shell + CSS

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create index.html with full HTML structure and CSS**

Create `/home/zubair/work/Urdu-transcription/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Urdu ASR — Realtime Transcription</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      background: #0f1117;
      color: #cbd5e1;
      font-family: system-ui, -apple-system, sans-serif;
      font-size: 15px;
      line-height: 1.6;
      min-height: 100vh;
      display: flex;
      align-items: flex-start;
      justify-content: center;
      padding: 32px 16px;
    }

    .container {
      width: 100%;
      max-width: 760px;
      display: flex;
      flex-direction: column;
      gap: 16px;
    }

    header h1 {
      font-size: 22px;
      font-weight: 600;
      color: #f1f5f9;
    }
    header .subtitle {
      font-size: 13px;
      color: #64748b;
      margin-top: 2px;
    }

    .status-bar {
      display: flex;
      align-items: center;
      gap: 12px;
      background: #0a0d14;
      border: 1px solid #1e2433;
      border-radius: 8px;
      padding: 10px 14px;
    }
    .status-pill {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 13px;
    }
    .dot {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      flex-shrink: 0;
    }
    .dot--grey  { background: #64748b; }
    .dot--amber { background: #fcd34d; }
    .dot--green { background: #22c55e; }
    .dot--red   { background: #f87171; }
    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50%       { opacity: 0.4; }
    }
    .dot--pulse { animation: pulse 1.2s ease-in-out infinite; }

    .lang-label {
      font-size: 12px;
      color: #64748b;
      margin-left: auto;
    }

    .controls {
      display: flex;
      gap: 10px;
      align-items: center;
      flex-wrap: wrap;
    }

    button {
      font: inherit;
      font-size: 14px;
      cursor: pointer;
      border: none;
      border-radius: 6px;
      padding: 9px 18px;
      transition: opacity 0.15s;
    }
    button:disabled {
      opacity: 0.45;
      cursor: not-allowed;
    }

    #start-btn {
      background: #16a34a;
      color: #fff;
      font-weight: 500;
    }
    #start-btn.recording {
      background: #dc2626;
      display: flex;
      align-items: center;
      gap: 8px;
    }
    #start-btn.recording::before {
      content: '';
      width: 8px;
      height: 8px;
      border-radius: 50%;
      background: #fff;
      animation: pulse 1.2s ease-in-out infinite;
      flex-shrink: 0;
    }
    #start-btn.stopping {
      background: #374151;
      color: #9ca3af;
    }

    #lang-select {
      font: inherit;
      font-size: 13px;
      background: #0a0d14;
      color: #cbd5e1;
      border: 1px solid #1e2433;
      border-radius: 6px;
      padding: 9px 12px;
      cursor: pointer;
    }

    #clear-btn {
      background: #1e2433;
      color: #94a3b8;
    }
    #clear-btn:hover:not(:disabled) { background: #263047; }

    .error-banner {
      background: #1c0a0a;
      border: 1px solid #7f1d1d;
      border-radius: 8px;
      padding: 12px 14px;
      font-size: 13px;
      color: #fca5a5;
      display: flex;
      align-items: center;
      gap: 12px;
    }
    .error-banner.hidden { display: none; }

    .retry-btn {
      background: #7f1d1d;
      color: #fca5a5;
      font-size: 12px;
      padding: 5px 12px;
      margin-left: auto;
      flex-shrink: 0;
    }
    .retry-btn:hover:not(:disabled) { background: #991b1b; }
    .retry-btn.hidden { display: none; }

    .history {
      background: #0a0d14;
      border: 1px solid #1e2433;
      border-radius: 8px 8px 0 0;
      padding: 16px;
      min-height: 200px;
      max-height: 400px;
      overflow-y: auto;
      font-size: 15px;
      line-height: 1.8;
      white-space: pre-wrap;
      word-break: break-word;
    }
    .history .placeholder {
      color: #334155;
      font-style: italic;
    }

    .live {
      background: #0d1220;
      border: 1px solid #1e2433;
      border-top: none;
      border-radius: 0 0 8px 8px;
      padding: 12px 16px;
      min-height: 44px;
      font-size: 15px;
      color: #fcd34d;
      white-space: pre-wrap;
      word-break: break-word;
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Urdu ASR</h1>
      <p class="subtitle">Realtime speech transcription — Nemotron 3.5</p>
    </header>

    <div class="status-bar">
      <span class="status-pill">
        <span id="status-dot" class="dot dot--grey"></span>
        <span id="status-text">Loading model…</span>
      </span>
      <span id="lang-label" class="lang-label"></span>
    </div>

    <div class="controls">
      <button id="start-btn" disabled>Start Recording</button>
      <select id="lang-select">
        <option value="auto">auto (detect)</option>
        <option value="en">English</option>
      </select>
      <button id="clear-btn">Clear</button>
    </div>

    <div id="error-banner" class="error-banner hidden">
      <span id="error-msg"></span>
      <button id="retry-btn" class="retry-btn hidden">Retry</button>
    </div>

    <div id="history" class="history">
      <span class="placeholder">Transcript will appear here…</span>
    </div>

    <div id="live" class="live"></div>
  </div>

  <script>
    // JS added in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify layout**

```bash
python3 -m http.server 8080 --directory /home/zubair/work/Urdu-transcription
```

Open `http://localhost:8080` in Chrome. Expected:
- Dark background `#0f1117`, centred content, max-width ~760 px
- Header: "Urdu ASR" + subtitle in muted grey
- Status bar: grey dot + "Loading model…" text
- Controls: greyed-out "Start Recording" (green tint), language select, "Clear" button
- No error banner (hidden)
- History zone: light grey italic placeholder text "Transcript will appear here…"
- Amber-tinted live zone below history, visually joined (no gap)

- [ ] **Step 3: Commit**

```bash
cd /home/zubair/work/Urdu-transcription
git add index.html
git commit -m "feat: add HTML shell and dark-theme CSS"
```

---

### Task 2: CONFIG, STATE, UI refs, UI functions, Clear button

**Files:**
- Modify: `index.html` — replace `// JS added in subsequent tasks` with the full script

- [ ] **Step 1: Add CONFIG constants and STATE variables**

Replace `// JS added in subsequent tasks` with:

```js
// ── CONFIG ────────────────────────────────────────────────────────────────────
const BASE_URL      = 'http://localhost:8000';
const WS_URL        = BASE_URL.replace('http', 'ws') + '/v1/transcriptions/stream';
const CHUNK_SAMPLES = 8960;
const CHUNK_MS      = 560;
const POLL_INTERVAL = 2000;
const POLL_TIMEOUT  = 60000;

// ── STATE ─────────────────────────────────────────────────────────────────────
let ws          = null;
let audioCtx    = null;
let micStream   = null;
let workletNode = null;
let procNode    = null;
let sampleBuf   = new Float32Array(CHUNK_SAMPLES);
let sampleCount = 0;
let isRecording = false;
let isStopping  = false;
let pollTimer   = null;
let pollElapsed = 0;
let blobUrl     = null;

// ── UI REFS ───────────────────────────────────────────────────────────────────
const $dot     = document.getElementById('status-dot');
const $txt     = document.getElementById('status-text');
const $lang    = document.getElementById('lang-label');
const $sel     = document.getElementById('lang-select');
const $start   = document.getElementById('start-btn');
const $clear   = document.getElementById('clear-btn');
const $history = document.getElementById('history');
const $live    = document.getElementById('live');
const $errBan  = document.getElementById('error-banner');
const $errMsg  = document.getElementById('error-msg');
const $retry   = document.getElementById('retry-btn');

// ── UI FUNCTIONS ──────────────────────────────────────────────────────────────
function setStatus(state) {
  // state: 'loading' | 'connecting' | 'connected' | 'disconnected' | 'error'
  const map = {
    loading:      ['dot--grey',              'Loading model…'],
    connecting:   ['dot--amber dot--pulse',  'Connecting…'],
    connected:    ['dot--green',             'Connected'],
    disconnected: ['dot--grey',              'Disconnected'],
    error:        ['dot--red',               'Error'],
  };
  const [cls, label] = map[state] || map.disconnected;
  $dot.className = 'dot ' + cls;
  $txt.textContent = label;
}

function setLang(name) {
  $lang.textContent = name ? 'Detected: ' + name : '';
}

function appendHistory(text) {
  const ph = $history.querySelector('.placeholder');
  if (ph) ph.remove();
  $history.appendChild(document.createTextNode(text + '\n'));
  $history.scrollTop = $history.scrollHeight;
}

function appendLive(token) {
  $live.textContent += token;
}

function clearLive() {
  $live.textContent = '';
}

function showError(msg, withRetry) {
  $errMsg.textContent = msg;
  $errBan.classList.remove('hidden');
  $retry.classList.toggle('hidden', !withRetry);
}

function clearError() {
  $errBan.classList.add('hidden');
  $errMsg.textContent = '';
  $retry.classList.add('hidden');
}

function clearTranscript() {
  $history.innerHTML = '<span class="placeholder">Transcript will appear here…</span>';
  clearLive();
}

// ── EVENTS: Clear ─────────────────────────────────────────────────────────────
$clear.addEventListener('click', clearTranscript);
```

- [ ] **Step 2: Verify in browser console**

Open `http://localhost:8080`. Open DevTools console and run:

```js
appendHistory('Test line 1');
appendHistory('Test line 2');
appendLive(' partial token');
```

Expected: history shows both lines; live zone shows " partial token" in amber.

Click "Clear" — both zones reset to the italic placeholder. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add CONFIG, STATE, UI functions, and Clear button"
```

---

### Task 3: Ready Poller

**Files:**
- Modify: `index.html` — add READY POLL section + DOMContentLoaded init

- [ ] **Step 1: Add polling functions after the Clear event listener**

Append to the `<script>` block (after `$clear.addEventListener('click', clearTranscript);`):

```js
// ── READY POLL ────────────────────────────────────────────────────────────────
function startPolling() {
  pollElapsed = 0;
  clearError();
  setStatus('loading');
  $start.disabled = true;
  pollTimer = setInterval(async () => {
    pollElapsed += POLL_INTERVAL;
    try {
      const res = await fetch(BASE_URL + '/ready');
      if (res.ok) {
        const data = await res.json().catch(() => ({}));
        if (data.chunk_samples && data.chunk_samples !== CHUNK_SAMPLES) {
          console.warn('chunk_samples mismatch — server:', data.chunk_samples, 'client:', CHUNK_SAMPLES);
        }
        if (data.sample_rate && data.sample_rate !== 16000) {
          console.warn('sample_rate mismatch — server:', data.sample_rate, 'client: 16000');
        }
        stopPolling();
        setStatus('disconnected');
        $start.disabled = false;
        return;
      }
    } catch (_) {
      // Backend not reachable yet — keep polling
    }
    if (pollElapsed >= POLL_TIMEOUT) {
      stopPolling();
      setStatus('error');
      showError('Backend not responding after 60 s. Is docker compose up running?', true);
    }
  }, POLL_INTERVAL);
}

function stopPolling() {
  clearInterval(pollTimer);
  pollTimer = null;
}

// ── INIT ──────────────────────────────────────────────────────────────────────
document.addEventListener('DOMContentLoaded', () => {
  if (!window.AudioContext && !window.webkitAudioContext) {
    showError('Your browser does not support Web Audio. Please use Chrome 120+ or Firefox 120+.', false);
    return;
  }
  startPolling();
});

$retry.addEventListener('click', () => {
  clearError();
  startPolling();
});
```

- [ ] **Step 2: Test — backend DOWN**

Ensure Docker is stopped. Reload `http://localhost:8080`.

Expected:
- Grey dot + "Loading model…" immediately
- "Start Recording" disabled
- After ~60 s: red dot + "Error" + banner "Backend not responding after 60 s. Is docker compose up running?" + "Retry" button
- Click Retry: banner clears, "Loading model…" resumes

- [ ] **Step 3: Test — backend UP**

Start Docker: `docker compose up` in `/home/zubair/work/nemotron-speech-docker`. Wait for model to load.
Reload `http://localhost:8080`.

Expected:
- "Loading model…" for ≤2 s, then grey dot + "Disconnected"
- "Start Recording" button becomes enabled (green)
- No error banner

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add ready poller with 60 s timeout and retry"
```

---

### Task 4: WebSocket session handler

**Files:**
- Modify: `index.html` — add WEBSOCKET section + EVENTS: Start/Stop

- [ ] **Step 1: Add resetButton + WebSocket functions**

Append to the `<script>` block (after `$retry.addEventListener('click', ...)`):

```js
// ── WEBSOCKET ─────────────────────────────────────────────────────────────────
function resetButton() {
  isRecording = false;
  isStopping  = false;
  $start.disabled = false;
  $start.textContent = 'Start Recording';
  $start.className = '';
  $sel.disabled = false;
}

function openWs() {
  setStatus('connecting');
  clearError();
  ws = new WebSocket(WS_URL);
  ws.binaryType = 'arraybuffer';

  ws.onopen = () => sendConfig();

  ws.onmessage = (e) => {
    let msg;
    try { msg = JSON.parse(e.data); } catch (_) { return; }
    handleMessage(msg);
  };

  ws.onclose = () => {
    if (isStopping) return; // expected close — handled after final
    handleDisconnect('Connection lost.');
  };

  ws.onerror = () => {
    handleDisconnect('Backend not running. Check that docker compose up is running.');
  };
}

function sendConfig() {
  ws.send(JSON.stringify({
    language:    $sel.value,
    sample_rate: 16000,
    chunk_ms:    CHUNK_MS,
    use_vad:     false,
    format:      'f32le',
  }));
}

function handleMessage(msg) {
  switch (msg.event) {
    case 'ready':
      setStatus('connected');
      setLang(msg.language_name || '');
      startAudio();
      break;

    case 'partial':
      if (msg.text) appendLive(msg.text);
      break;

    case 'final': {
      const text = $live.textContent.trim();
      appendHistory(text || '(no speech detected)');
      clearLive();
      closeWs();
      stopAudio();
      resetButton();
      setStatus('disconnected');
      break;
    }

    case 'error':
      showError(msg.message || 'Transcription error.', false);
      stopAudio();
      closeWs();
      resetButton();
      setStatus('disconnected');
      break;
  }
}

function handleDisconnect(msg) {
  stopAudio();
  closeWs();
  resetButton();
  setStatus('disconnected');
  showError(msg, true);
}

function closeWs() {
  isStopping = false;
  if (!ws) return;
  ws.onclose = null;
  ws.onerror = null;
  if (ws.readyState === WebSocket.OPEN || ws.readyState === WebSocket.CONNECTING) {
    ws.close();
  }
  ws = null;
}

// ── EVENTS: Start/Stop ────────────────────────────────────────────────────────
$start.addEventListener('click', () => {
  if (!isRecording && !isStopping) {
    isRecording = true;
    $start.textContent = 'Stop Recording';
    $start.className = 'recording';
    $sel.disabled = true;
    clearError();
    setLang('');
    openWs();
  } else if (isRecording && !isStopping) {
    isStopping  = true;
    isRecording = false;
    $start.textContent = 'Stopping…';
    $start.className = 'stopping';
    $start.disabled = true;
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ event: 'end' }));
      // Do NOT close WS here — wait for final event
    } else {
      handleDisconnect('Connection lost before stop.');
    }
  }
});
```

- [ ] **Step 2: Add startAudio/stopAudio stubs for testing this task in isolation**

Append (these will be replaced in Task 5):

```js
// ── AUDIO (stubs — replaced in Task 5) ───────────────────────────────────────
function startAudio() { console.log('[stub] startAudio called'); }
function stopAudio()  { console.log('[stub] stopAudio called'); }
```

- [ ] **Step 3: Test — WebSocket connect with backend running**

With Docker running, reload `http://localhost:8080`. Wait for "Start Recording" to enable.
Click "Start Recording".

Expected:
- Dot turns amber + "Connecting…"
- Within 1 s: dot turns green + "Connected"
- Console logs `[stub] startAudio called`
- Button shows "Stop Recording" (red with pulsing dot)
- Language label: e.g. "Detected: English"

Click "Stop Recording":
- Button shows "Stopping…" (grey, disabled)
- Network tab → WS → see outgoing `{"event":"end"}` frame
- Server sends `final` → "(no speech detected)" appears in history
- Button resets to "Start Recording" (green, enabled)
- Status → "Disconnected"

- [ ] **Step 4: Test — WebSocket refused (backend DOWN)**

Stop Docker. Wait for "Start Recording" to re-enable (from poller). Click Start.

Expected:
- Error banner: "Backend not running. Check that docker compose up is running." + Retry
- Button resets to "Start Recording"

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add WebSocket session handler and Start/Stop button"
```

---

### Task 5: Audio pipeline

**Files:**
- Modify: `index.html` — replace the two audio stubs with real implementations

- [ ] **Step 1: Replace stubs with full audio section**

Remove the `// ── AUDIO (stubs…)` block and the two stub functions. Replace with:

```js
// ── AUDIO ─────────────────────────────────────────────────────────────────────
function buildWorkletBlobUrl() {
  const src = `
class PCMProcessor extends AudioWorkletProcessor {
  process(inputs) {
    const ch = inputs[0]?.[0];
    if (ch) this.port.postMessage(ch.slice());
    return true;
  }
}
registerProcessor('pcm-processor', PCMProcessor);
`;
  return URL.createObjectURL(new Blob([src], { type: 'application/javascript' }));
}

async function resample(samples, fromRate) {
  const targetLen = Math.round(samples.length * 16000 / fromRate);
  const offCtx = new OfflineAudioContext(1, targetLen, 16000);
  const buf = offCtx.createBuffer(1, samples.length, fromRate);
  buf.copyToChannel(samples, 0);
  const src = offCtx.createBufferSource();
  src.buffer = buf;
  src.connect(offCtx.destination);
  src.start();
  const rendered = await offCtx.startRendering();
  return rendered.getChannelData(0);
}

function pushSamples(samples) {
  let offset = 0;
  while (offset < samples.length) {
    const room = CHUNK_SAMPLES - sampleCount;
    const take = Math.min(room, samples.length - offset);
    sampleBuf.set(samples.subarray(offset, offset + take), sampleCount);
    sampleCount += take;
    offset += take;
    if (sampleCount === CHUNK_SAMPLES) {
      sendChunk();
      sampleBuf   = new Float32Array(CHUNK_SAMPLES);
      sampleCount = 0;
    }
  }
}

function sendChunk() {
  if (!ws || ws.readyState !== WebSocket.OPEN) return;
  ws.send(new Float32Array(sampleBuf).buffer);
}

async function startAudio() {
  sampleBuf   = new Float32Array(CHUNK_SAMPLES);
  sampleCount = 0;

  try {
    micStream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
  } catch (_) {
    showError('Microphone access denied. Please allow mic access and retry.', false);
    handleDisconnect('Mic denied.');
    return;
  }

  const ActxClass = window.AudioContext || window.webkitAudioContext;
  audioCtx = new ActxClass();
  const micSource  = audioCtx.createMediaStreamSource(micStream);
  const nativeRate = audioCtx.sampleRate;
  const needsResample = nativeRate !== 16000;

  if (window.AudioWorkletNode) {
    try {
      if (!blobUrl) blobUrl = buildWorkletBlobUrl();
      await audioCtx.audioWorklet.addModule(blobUrl);
      workletNode = new AudioWorkletNode(audioCtx, 'pcm-processor');
      workletNode.port.onmessage = async (e) => {
        let s = e.data;
        if (needsResample) s = await resample(s, nativeRate);
        pushSamples(s);
      };
      micSource.connect(workletNode);
      return;
    } catch (err) {
      console.warn('AudioWorklet failed, falling back to ScriptProcessorNode:', err);
    }
  } else {
    console.warn('AudioWorkletNode unavailable, using ScriptProcessorNode');
  }

  // ScriptProcessorNode fallback
  procNode = audioCtx.createScriptProcessor(4096, 1, 1);
  procNode.onaudioprocess = async (e) => {
    let s = e.inputBuffer.getChannelData(0).slice();
    if (needsResample) s = await resample(s, nativeRate);
    pushSamples(s);
  };
  micSource.connect(procNode);
  procNode.connect(audioCtx.destination); // ScriptProcessor requires a destination node
}

function stopAudio() {
  if (workletNode) { workletNode.disconnect(); workletNode = null; }
  if (procNode)    { procNode.disconnect();    procNode    = null; }
  if (audioCtx)   { audioCtx.close();         audioCtx    = null; }
  if (micStream)  { micStream.getTracks().forEach(t => t.stop()); micStream = null; }
}
```

- [ ] **Step 2: Test — golden path with real audio**

With Docker running, reload `http://localhost:8080`. Wait for "Start Recording".
Click Start. Speak for 5–10 seconds in any language.

Expected:
- Amber partial tokens appear in the live zone as you speak (token by token)
- Click "Stop Recording" → button shows "Stopping…"
- Live zone content absorbed into history, live zone clears
- Button resets to "Start Recording", status → "Disconnected"
- Browser tab: no mic indicator (mic released)

In DevTools Network → WS: verify binary frames arriving, each 35840 bytes (8960 × 4).

- [ ] **Step 3: Test — mic denied**

Open `http://localhost:8080` in a new incognito tab (to reset mic permission).
Wait for "Start Recording". Click Start. Deny mic permission in the browser prompt.

Expected:
- Error banner: "Microphone access denied. Please allow mic access and retry."
- Button resets to "Start Recording"
- No unhandled console errors

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add audio pipeline with AudioWorklet, resampling, and 8960-sample buffering"
```

---

### Task 6: End-to-end verification

**Files:**
- Modify: `index.html` — fix any issues found during testing

- [ ] **Step 1: Test transcript accumulation**

With Docker running:
1. Click Start, speak "Hello world", click Stop — history shows the transcript
2. Click Start again, speak "Second sentence", click Stop — history shows both, stacked
3. Click Clear — both zones reset to placeholder

- [ ] **Step 2: Test empty transcript**

Click Start, wait 3 s without speaking, click Stop.

Expected: history shows "(no speech detected)"

- [ ] **Step 3: Test mid-session disconnect**

Click Start, speak a few words. In another terminal:

```bash
docker compose stop
```

Expected:
- Red dot + "Disconnected" status
- Error banner: "Connection lost." + Retry button
- Mic stops (no mic indicator in browser tab)
- Button resets to "Start Recording"

Click Retry → error clears, "Loading model…" polling resumes.

- [ ] **Step 4: Test in Firefox 120+**

Open `http://localhost:8080` in Firefox. Run the golden path (speak, stop, verify history).

Expected: identical behaviour. Check Firefox DevTools console for any AudioWorklet warnings.

- [ ] **Step 5: Commit any fixes**

```bash
git add index.html
git commit -m "fix: edge case and cross-browser fixes from manual testing"
```

---

### Task 7: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README**

Create `/home/zubair/work/Urdu-transcription/README.md`:

````markdown
# Urdu ASR — Realtime Transcription

Browser app for evaluating the Nemotron 3.5 ASR model on Urdu speech. Streams
microphone audio over WebSocket to a locally running Docker container and displays
transcript tokens in real time. No install, no build step — open `index.html` in
a browser.

## Prerequisites

- Docker + Docker Compose
- Chrome 120+ or Firefox 120+

## 1. Start the backend

```bash
cd /path/to/nemotron-speech-docker
docker compose up
```

Wait until the logs show the model is loaded. The app polls `/ready` automatically
and enables the Start button when the model is ready.

## 2. Open the app

**Option A — direct file open:**
Double-click `index.html`, or drag it into a browser tab.

**Option B — local server (required if AudioWorklet is blocked by `file://` origin):**
```bash
python3 -m http.server 8080 --directory /path/to/Urdu-transcription
```
Then open `http://localhost:8080`.

## Usage

1. Wait for the status indicator to show **Disconnected** (model loaded)
2. Click **Start Recording** and speak
3. Transcript tokens appear in real time in the amber live line below the history
4. Click **Stop Recording** — text is committed to the history zone
5. Repeat to accumulate sessions; **Clear** resets the display

## Changing the backend URL

Edit the top of the `<script>` block in `index.html`:

```js
const BASE_URL = 'http://localhost:8000';
```
````

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with setup and usage instructions"
```

---

## Self-Review

### Spec coverage

| Requirement | Task |
|---|---|
| Single `index.html`, no build, no CDN | 1 |
| Dark theme (`#0f1117` bg, amber live, green connected) | 1 |
| Header + subtitle | 1 |
| Status bar: dot + text + language label | 1, 2 |
| Start/Stop toggle button + 3 states (green/red/grey) | 1, 4 |
| Language select (`auto` default + `en` option) | 1, 4 |
| Clear button resets both zones + restores placeholder | 2 |
| History zone: scrollable, settled text, persists across cycles | 1, 2 |
| Live zone: amber, appends incremental tokens | 1, 2 |
| Error banner (no `alert()`), inline | 1, 2 |
| Retry button on applicable errors | 3, 4 |
| `BASE_URL` as single config point | 2 |
| `WS_URL` derived from `BASE_URL` | 2 |
| `/ready` poller: 2 s interval, 60 s timeout | 3 |
| chunk_samples / sample_rate mismatch warning | 3 |
| Web Audio unsupported → disable Start, show message | 3 |
| WebSocket config: `language`, `sample_rate`, `chunk_ms`, `use_vad:false`, `format` | 4 |
| `ready` event → set language label, unblock audio | 4 |
| `partial` event → append to live zone (incremental) | 4 |
| `final` event → live → history, clear live, close WS, stop audio | 4 |
| `error` event → show message, stop, reset | 4 |
| Stop sequence: send `{"event":"end"}`, wait for `final`, then close | 4 |
| Empty transcript → "(no speech detected)" placeholder | 4 |
| Mid-session WS close → stop audio, show Disconnected + Retry | 4 |
| WS connection refused → show message + Retry | 4 |
| AudioWorklet via Blob URL | 5 |
| ScriptProcessorNode fallback with `console.warn` | 5 |
| Resample via `OfflineAudioContext` if mic ≠ 16 kHz | 5 |
| Buffer exactly 8960 samples before sending | 5 |
| Never send audio while WS not `OPEN` | 5 (`sendChunk` guard) |
| Stop audio: disconnect nodes, close AudioContext, stop mic tracks | 5 |
| Mic denied → inline message, no crash | 5 |
| Transcript accumulates across Start/Stop; Clear is only reset | 6 |
| Chrome 120+ and Firefox 120+ | 6 |
| `README.md` | 7 |

All 24 user stories and all §8 error states are covered. No spec gaps.

### Placeholder scan

No TBDs, TODOs, "similar to Task N", or vague "add error handling" lines found.

### Type/name consistency

- `appendLive(token)` appends; `clearLive()` clears — called correctly in Task 4 (`partial` → appendLive, `final` → clearLive)
- `handleDisconnect(msg)` signature is `(string)` — called consistently from `ws.onclose`, `ws.onerror`, and Task 5 mic-denied path
- `isStopping` set true on Stop click, cleared in `closeWs()` — no leak between sessions
- `blobUrl` created once, reused — `buildWorkletBlobUrl()` not called on subsequent recordings
- `sampleBuf` reset to fresh `Float32Array(CHUNK_SAMPLES)` at the top of `startAudio()` and after each chunk send
