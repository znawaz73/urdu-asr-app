# Urdu ASR — Realtime Transcription App

A lightweight browser-based speech-to-text app for Urdu audio, powered by the Nemotron ASR model running in a local Docker container.

## Prerequisites

- **Docker & Docker Compose** — to run the Nemotron backend
- **Chrome 120+** or **Firefox 120+** — the app uses Web Audio API and WebSocket
- **Microphone access** — the app will request permission when you click Start Recording

## Start the Backend

The app requires a running Nemotron ASR server. Navigate to the `nemotron-speech-docker` directory and start the container:

```bash
cd nemotron-speech-docker
docker compose up
```

Wait until you see output indicating the server is ready. You should see something like:
```
[INFO] uvicorn.server: Application startup complete
```

The app expects the server to be running at `http://localhost:8000`. Once the backend is ready, the app's status indicator will switch from **Loading model…** to **Disconnected** (you can open the app now).

## Open the App

You have two options:

### Option 1: Double-click the file
Open `index.html` in your browser by double-clicking it. The app will run directly from your filesystem.

### Option 2: Serve with HTTP
```bash
python3 -m http.server 8080
```
Then open your browser to `http://localhost:8080` and click `index.html`.

**Note:** Double-clicking works fine. The HTTP server option is useful if you run into CORS or filesystem-related issues.

## Basic Usage

1. **Check the status** — Once the page loads, wait for the status to show **Disconnected** (it will start at **Loading model…**).
2. **Start Recording** — Click the **Start** button to begin capturing audio from your microphone.
3. **Speak** — The app streams audio to the backend in real time. You should see the detected language label appear (e.g., "Detected: Urdu").
4. **Watch the transcript** — Partial results appear on a live line as the model processes speech. Completed sentences are moved to the history above.
5. **Stop Recording** — Click **Stop Recording** to end the session. The model will process any remaining buffered audio.
6. **Clear** — Click **Clear** to reset the transcript display. The next recording will start fresh.

## Change the Backend URL

If you need to connect to a backend at a different address, edit `index.html` and find this line near the top of the `<script>` block:

```javascript
const BASE_URL = 'http://localhost:8000';
```

Replace `localhost:8000` with your server's address (e.g., `192.168.1.100:8000`) and save. The app will use the new URL on reload.

## Troubleshooting

- **Status shows "Disconnected" and stays that way:** The backend is not running or not reachable. Check that Docker is running and `docker compose up` shows startup completed.
- **Mic permission denied:** The browser blocked microphone access. Check your system audio permissions and browser settings.
- **No text appears while speaking:** The backend may not be ready. Refresh the page and wait for the status to show **Disconnected** (grey dot).
- **Text appears in wrong script (Devanagari instead of Urdu):** This is the model's output as-is. The Nemotron model sometimes detects the language differently depending on the speech characteristics.

## File Structure

```
.
├── index.html       # The complete app — HTML + CSS + JavaScript (no build needed)
├── README.md        # This file
└── CLAUDE.md        # Project development guidelines
```

The app is a single HTML file. No npm, no build step, no external dependencies.
