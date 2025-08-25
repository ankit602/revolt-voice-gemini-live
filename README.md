# revolt-voice-gemini-live
Revolt Motors voice assistant using Gemini Live — real-time PCM mic streaming, native-audio replies, barge-in, and low-latency Node/Express backend.
# Rev — Revolt Motors Voice Assistant
**Node.js (server-to-server) + Gemini Live API + low-latency voice in/out**

A minimal, production-ready clone of the Rev voice experience using Google’s **Gemini Live** API.  
It streams the mic as **PCM 16 kHz**, handles **interruptions/barge-in**, and plays back the model’s **streamed audio** with low latency.

---

## ✨ Features

- **Server-to-server** architecture (browser → your Node server → Gemini Live WebSocket)
- **Real-time** voice: mic → PCM-16k frames over WS; model starts speaking while you’re still talking
- **Barge-in / interrupt**: user can cut off model mid-utterance and ask a new question
- **Silence detection**: auto-finalizes the turn ~700 ms after you stop speaking
- **Low-latency playback**:
  - Preferred: queue PCM into WebAudio (`AudioContext`) for seamless playback
  - Fallback: wrap raw PCM in a WAV header and play via `<audio>`
- **Scoped assistant**: system prompt keeps the bot on Revolt Motors topics
- **Text mode**: type + send = immediate response turn

---

## 📦 Project structure

```
revolt-voice/
├─ server/
│  ├─ index.js           # Node/Express + WS bridge to Gemini Live (server-to-server)
│  ├─ .env.example       # sample environment file (create your .env from this)
│  └─ package.json
└─ web/
   ├─ index.html         # Simple UI
   ├─ app.js             # PCM capture + audio playback + WS messaging
   └─ pcm-worklet.js     # AudioWorklet: 48kHz → 16kHz, Float32 → Int16 PCM
```

> If your repo layout differs, adjust paths accordingly. The server expects to serve the `web/` folder from a sibling directory of `server/`.

---

## 🧱 Requirements

- **Node 18+** (tested with Node 20/22)
- A **Google AI Studio** API key: https://aistudio.google.com/
- Chrome/Edge (AudioWorklet support)

---

## ⚙️ Setup

1) **Install dependencies**
```bash
cd server
npm install
```

2) **Create `.env`**
```ini
# server/.env
GOOGLE_API_KEY=YOUR_AI_STUDIO_KEY
# Final submission (native audio):
GEMINI_MODEL=gemini-2.5-flash-preview-native-audio-dialog
# For dev / heavy testing (fewer rate limits), you can switch to:
# GEMINI_MODEL=gemini-2.0-flash-live-001
PORT=8000
```

3) **Run the server**
```bash
node index.js
# -> Server running at http://127.0.0.1:8000  (UI at /web)
```

4) **Open the UI**
- Visit `http://127.0.0.1:8000/web`
- Click **Start Mic** and grant mic permission
- Speak → pause for ~0.7s → hear the reply  
  (or type a message and click **Send** for immediate response)

---

## 🧭 Architecture

```
[Browser UI]
  ├─ AudioWorklet: 48k → 16k downsample, Float32 → Int16 PCM
  ├─ WS: send {type:"audio", mime:"audio/pcm;rate=16000", data:"<b64>"}
  ├─ WS: send {type:"text"} / {type:"interrupt"} / {type:"end_turn"}
  └─ Playback: WebAudio queue for PCM (smooth); WAV fallback for containers

        ⇅  WebSocket  (ws://host:8000/ws)

[Node server]
  ├─ Express serves /web static UI
  ├─ WS bridge /ws: browser ⇆ Google Live WS (BidiGenerateContent)
  ├─ Sends setup:
  │     - model: models/${GEMINI_MODEL}
  │     - generationConfig.responseModalities = ["AUDIO"]
  │     - systemInstruction = { role:"system", parts:[{ text: SYSTEM_PROMPT }] }
  ├─ For audio frames: forwards as realtimeInput.audio (PCM 16k)
  ├─ Turn finalization:
  │     - 700 ms debounce after last audio chunk (auto)
  │     - or on {type:"end_turn"} from the client (manual)
  └─ Relays text deltas + audio chunks back to the browser
```

---

## 🗣️ System instruction (scope to Revolt Motors)

Defined in `server/index.js` as a **Content** object (not a plain string):

```js
const SYSTEM_PROMPT =
  "You are Rev, the Revolt Motors assistant. Only answer questions related to Revolt Motors motorcycles, charging, service, pricing, availability, test rides, specifications, financing, and official contact details. If asked unrelated questions, briefly steer the user back to Revolt topics.";

systemInstruction: {
  role: 'system',
  parts: [{ text: SYSTEM_PROMPT }]
},
```

---

## 🔌 Browser ↔ Server WS protocol

**Client → Server**
```json
{"type":"audio","mime":"audio/pcm;rate=16000","data":"<base64 pcm16>"}
{"type":"text","data":"What's the on-road price in Pune?"}
{"type":"interrupt"}
{"type":"end_turn"}   // explicit “I’m done talking; start responding”
```

**Server → Client**
```json
{"type":"text","data":"(streamed text delta)"}
{"type":"audio","mime":"audio/pcm;rate=24000","data":"<base64 pcm16>"}
{"type":"done"}
{"type":"interrupted"}
{"type":"error","data":"..."}
```

---

## 🎧 Frontend details

- **Capture / send**  
  `web/pcm-worklet.js` down-samples mic 48kHz → 16kHz and converts Float32 → Int16 PCM;  
  `web/app.js` base64-encodes bytes and sends with `mime: "audio/pcm;rate=16000"`.

- **Playback**  
  Preferred: stream PCM chunks into a WebAudio queue (`AudioContext`) for gapless, low-latency playback.  
  Fallback: if a container arrives (e.g., WAV/OGG), set `audio.src = URL.createObjectURL(blob)`.

---

## 🧠 Backend details

- WebSocket endpoint (Google):  
  `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=API_KEY`

- **Setup** (sent once per connection):
  - `model: "models/<GEMINI_MODEL>"`
  - `generationConfig.responseModalities = ["AUDIO"]`
  - `systemInstruction` **as Content** (`{ role, parts:[{text}] }`)
  - Optional: `inputAudioTranscription` / `outputAudioTranscription` for debugging

- **Audio input**: `realtimeInput.audio` with `mimeType: "audio/pcm;rate=16000"`  
- **Text input**: `clientContent.turns[{role:'user', parts:[{text}]}]` and `turnComplete: true`  
- **Interruptions**: Any new realtime input interrupts current generation; an explicit empty turn also works.

---

## 🧪 What to test

- **Voice**: Start Mic → ask in English/Hindi/Punjabi → pause → bot responds within ~1–2s
- **Barge-in**: While it’s speaking, press **Interrupt** (or just start talking) → it stops and handles the new query
- **Text**: Type `Say "connected".` → **Send** → immediate response
- **Latency**: The first audio chunk should arrive quickly after you pause

---

## 🛠️ Troubleshooting

1) **Text shows, but no audio**  
   - The model often streams **raw PCM** (e.g., `audio/pcm;rate=24000`).  
   - Ensure the client **queues PCM via WebAudio** or **wraps it in a WAV header** before playback.

2) **Error: `Invalid value at 'setup.system_instruction'`**  
   - You sent a string. Use a Content object:
     ```js
     systemInstruction: { role: 'system', parts: [{ text: SYSTEM_PROMPT }] }
     ```

3) **`setupComplete` never appears**  
   - Check `GOOGLE_API_KEY` in `.env`.  
   - Try switching `GEMINI_MODEL` to `gemini-2.0-flash-live-001` during dev.

4) **Server logs show `audio/webm;codecs=opus`**  
   - Your UI is still using `MediaRecorder` (Opus). Use the **AudioWorklet** sender provided.

5) **No reply after speaking**  
   - Ensure a turn is finalized:
     - Auto: silence debounce should log `[TURN] auto turnComplete (idle)` in Node console.
     - Manual: clicking **Stop Mic** sends `{type:'end_turn'}`.

6) **Autoplay blocked**  
   - Interact with the page once or press play the first time. WebAudio usually bypasses autoplay restrictions.

7) **Echo/feedback**  
   - Ensure the mic worklet is **not** connected to `destination` (sidetone disabled).  
   - Use headphones or echo cancellation if needed.

---

## 🧪 Benchmark checklist (for evaluation)

- Natural dialog in multiple languages  
- Fast start of model audio after user pause (~1–2s)  
- Clean handling of interruptions (barge-in)  
- Scoped answers (Revolt Motors only)

---

## 🔐 Security / Production notes

- Keep your `GOOGLE_API_KEY` **server-side** only.  
- Use HTTPS + `wss://` in production. If reverse-proxying, enable WS upgrade headers.  
- Handle rate limiting and retries; native-audio models have strict free-tier limits.

---

## 📄 Submission guide

1) **Demo video (30–60s)**  
   - Show mic conversation, interruption mid-reply, and responsiveness.  
   - Upload to Google Drive with public view permissions.

2) **Source code (GitHub)**  
   - Include this README and a short setup section in the repo root.  
   - Provide `.env.example` (no secrets committed).

---

## 🙌 Credits

- Google AI — **Gemini Live API** (BidiGenerateContent)  
- Audio pipeline: **AudioWorklet** (capture) + **WebAudio** (playback queue)
