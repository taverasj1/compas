# Compás — Developer Handoff
**Single-file mobile-first choreography cue trainer for Latin dance.**
Place timed move cues on a track, hear them spoken aloud just before each move. Works with YouTube or a local audio file.

**Current live URL:** https://compas-d2b35.web.app (Firebase Hosting — project `compas-d2b35`)
**Old URL:** https://magenta-phoenix-fcca34.netlify.app (Netlify — deprecated)
**Goal:** continue development via Claude Code / terminal; deploy with `firebase deploy`.

---

## Quick start
```bash
# Preview locally (MUST serve over http — not file://)
npx serve public
# or: cd public && python3 -m http.server 3000
# open http://localhost:PORT in Chrome

# Deploy to Firebase
npm install -g firebase-tools
firebase login
firebase init hosting   # public dir: public | single-page app: Y | overwrite index.html: N
firebase deploy
```
Your live URL will be `https://PROJECT_ID.web.app`.

---

## Repo layout
```
compas/
├── public/
│   ├── index.html         ← the entire app (HTML + CSS + JS, no build step)
│   ├── manifest.webmanifest ← PWA manifest (installable app)
│   ├── sw.js              ← service worker (offline shell + font cache) — bump CACHE_NAME on deploys
│   └── icons/             ← PWA icons (192, 512, 512-maskable)
├── firebase.json          ← Firebase Hosting config
├── .firebaserc            ← created by `firebase init` / `firebase use` (holds project ID)
├── .gitignore
└── CLAUDE.md              ← this file — Claude Code auto-loads it as project context
```

---

## Firebase deploy — full steps

### New Firebase project (recommended)
```bash
npm install -g firebase-tools
firebase login
# From your compas/ folder
firebase init hosting
# Prompts:
#   → Use an existing project OR create a new one
#   → Public directory: public
#   → Configure as single-page app: Y
#   → Set up automatic GitHub Action deploys: N
#   → Overwrite public/index.html: N
firebase deploy
# → Live at https://YOUR-PROJECT-ID.web.app
```

### Reuse existing Firebase project (e.g., Vital Spa) with a separate Hosting site
```bash
# Create a new Hosting site within the existing project
firebase hosting:sites:create compas-app
# Apply a named deploy target
firebase target:apply hosting compas compas-app
# Add "target": "compas" to the hosting block in firebase.json
# Deploy only this site
firebase deploy --only hosting:compas
```

---

## Android app (Google Play) — TWA packaging
The Android app is a **Trusted Web Activity**: it runs the live PWA at
`https://compas-d2b35.web.app` fullscreen in Chrome, so spoken cues
(`speechSynthesis`) work untouched. Content/feature changes ship with
`firebase deploy` and appear in the installed app immediately — only shell,
package-name, or version changes need a new `.aab` resubmitted to Play.

**Digital Asset Links** verifies the app↔domain link and removes the URL bar.
The verification file lives at `public/.well-known/assetlinks.json` and is
served at `https://compas-d2b35.web.app/.well-known/assetlinks.json`.
> `firebase.json` must NOT ignore `**/.*`, or `.well-known/` is dropped from the
> deploy. A `headers` rule pins its `Content-Type` to `application/json`.

To finish wiring (once the `.aab` is built via PWABuilder.com or Bubblewrap):
1. Set `package_name` to the TWA's Android package id.
2. Set `sha256_cert_fingerprints` to the **Play App Signing** SHA-256 (Play
   Console → Setup → App integrity), not the local upload-key fingerprint.
3. `firebase deploy`, then verify at
   `https://developers.google.com/digital-asset-links/tools/generator`.

A privacy policy (required by Play) is served at `/privacy.html`.

---

## Continuing in Claude Code
```bash
npm install -g @anthropic-ai/claude-code
cd compas
claude
```
Claude Code automatically reads `CLAUDE.md` as project context at the start of every session.
Reference: https://docs.claude.com/en/docs/claude-code/overview

---

## Code architecture
The app is a single IIFE in `public/index.html`. No framework, no build step.

### State variables (top of IIFE)
```
sourceMode      'yt' | 'file'   — which engine is active
cues            [{id, time, name}]  sorted by time
loop            {start, end, enabled}
lang            'es-US' | 'en-US'
leadInMs        ms of lead-in before cue fires
playSpeed       0.5 – 1.5
speakEnabled, countIn, keepAwake, duckVol, rate, voiceURI
```

### Unified playback layer — the key abstraction
All shared logic calls these helpers. Never call `player.*` or `audioEl.*` directly outside the engine functions.

| Function | What it does |
|---|---|
| `pbReady()` | true if a track is loaded |
| `pbTime()` | current position (seconds) |
| `pbDuration()` | total duration |
| `pbSeek(t)` | seek to t |
| `pbIsPlaying()` | true if playing |
| `pbPlay()` / `pbPause()` | play / pause |
| `pbSetRate(r)` | playback speed |
| `pbSetVol(0–100)` | volume (duck / restore) |

Each dispatches on `sourceMode`. Adding a third source = add a branch to each pb* function + a new `load*` function.

### YouTube engine
`ensureApiScript` → `onApiReady` → `loadVideo` → `buildPlayer`
Events: `onReady` / `onState` / `onError` → `syncVideo` sets `currentVideoId`.
> **Critical:** YouTube embeds require a real `https://` origin. Will not work from `file://` or in-app webviews.

### Local audio engine
`getAudioEl` → `loadFile` → `ensureAudioCtx`
Web Audio graph: `MediaElementSource → GainNode → destination`
Ducking via `gainNode.gain` — works on every platform including iPhone.
`audioEl.preservesPitch = true` keeps pitch when slowing down.
Cues persist in localStorage but audio must be re-picked each session (browser security — can't auto-reload a local file).

### Timing loop (`tick` via `requestAnimationFrame`)
1. A–B loop: if `pbTime() >= loopEnd − 0.04s` → `pbSeek(loopStart)`, re-arm cues.
2. Cue fire: `trigger = cue.time − leadIn`. Fire on forward-crossing: `lastT < trigger ≤ currentTime`. Handles loops and manual seeks automatically.
3. `updateProgress` / `updateHint` / `renderTimeline` (only when duration changes).

**Perf behavior (don't regress):**
- The loop **stops itself when paused** (`tick` re-queues only while `playing || countActive`); `setPlaying(true)` restarts it via `startTick()`. Seeks made while paused must call `refreshUI(t)` for a one-shot repaint.
- Cue checks run every frame, but progress/hint **DOM writes are throttled to ~10Hz** (`lastUiMs`).
- `pbTime()` **interpolates YouTube time** between coarse `getCurrentTime()` polls using a `performance.now()` anchor (`ytAnchorT/ytAnchorMs`) × `playSpeed`, capped at +0.5s — this is what makes YT cue timing tight. `pbSeek` resets the anchor.

### Auto-cue (Gemini)
`autoGenerateCues` → `genViaDirect` (browser → Gemini, key in `x-goog-api-key` header — never in the URL) or `genViaProxy` (POST to a Cloud Function so the key stays server-side). Output parsed by `parseCuesJSON`/`parseTimeToSeconds`, applied by `applyGeneratedCues` (replace or merge per `autoMode`). Key/vocab/model/fnUrl persist in localStorage (`compas:geminiKey` etc.).

### PWA
`manifest.webmanifest` + `sw.js`, registered at init (https or localhost only). SW strategy: navigations network-first with cached `index.html` fallback; fonts + same-origin assets cache-first. YouTube/Gemini traffic is never intercepted. The YouTube IFrame API now **lazy-loads** (on first YT load or URL-field focus) — local-audio sessions never download it.

### Cue firing (`fireCue`)
Sets readout → CSS pulse animation → `speak(cue.name)`:
`duck()` → `speechSynthesis.speak(utterance)` → `onend: restore()`

### Persistence
```
localStorage key          contents
────────────────────────────────────────────────────────
"compas:<trackKey>"       cues + all settings for this track
"compas:index"            [{videoId, title, type}] saved-choreography list
```
`trackKey` = YouTube video ID for YT tracks, `"file:<filename>"` for local files.

### Share / export
`payload()` builds a compact object. `b64enc(JSON.stringify(payload()))` goes in the URL hash as `#c=…`. On load, `tryImportHash()` decodes and calls `importData()`. JSON export is the same payload as a `.compas.json` file download.

---

## Conventions — when adding features
1. **All playback calls go through `pb*`** — not `player.*` or `audioEl.*` directly.
2. **`sourceMode`** drives dispatch — adding a third source = branch in each pb* function.
3. **Auto-save after mutations** — call `save()` or `debouncedSave()` after changing `cues`, `loop`, or settings.
4. **`renderTimeline()`** after any cue or loop change, and when duration first becomes known.
5. **Spanish move names** come out correctly with `lang = 'es-US'` (vuelta, enchufla, sombrero, etc.).
6. **Bump `CACHE_NAME` in `sw.js`** (compas-v2, v3, …) in any commit that changes `index.html`, or returning visitors may briefly see the stale cached shell.
7. **Don't add per-frame DOM writes to `tick`** — UI writes go inside the 10Hz throttle block; the loop must keep stopping when paused.

---

## Known limitations
| Issue | Detail |
|---|---|
| YouTube needs real origin | Won't embed from `file://` or in-app previews — must serve over https |
| iPhone volume ducking | iOS owns OS volume; YouTube `setVolume` is ignored. Local audio ducks fine via Web Audio |
| YouTube ads | Can interrupt loops on free accounts |
| Local audio re-pick | Audio must be re-attached each session. Re-attach flow exists (tap saved 🎵 chip) |
| Cue timing on YouTube | Was ~100–300ms jitter; now interpolated in `pbTime()` — residual jitter is small. Local audio is tight |
| Offline | Service worker caches the shell + fonts; YouTube playback and Gemini still need internet (local audio works offline) |

---

## Backlog (priority order)
- [ ] **Per-cue notes** — text field per cue, visible on screen, not spoken. Quick UI add.
- [ ] **Tap-tempo BPM + 8-count snapping** — tap on the beat to detect BPM, snap cues to count positions (1–8). Core for Latin social dance teaching.
- [x] **PWA: manifest + service worker** — done: installable offline shell, fonts + index.html cached.
- [x] **AI auto-cue (Gemini)** — done: generates cues from the loaded YouTube video (direct key or Cloud Function proxy).
- [ ] **Beat-locked count-in** — time "cinco, seis, siete, ocho" to the actual beat grid once BPM exists.
- [ ] **Media Session API** — lock-screen / Bluetooth remote → prev/next cue hands-free mid-rehearsal.
- [ ] **Firestore cue sync** — store cues in Firestore; sync across devices; share a choreography by link.
- [ ] **Student view mode** — read-only cue player via shared link (no editing).
- [ ] **Firebase Auth** — gates student-sharing features above.

---

## Reference links
| Resource | URL |
|---|---|
| Live app (Firebase Hosting) | https://compas-d2b35.web.app |
| Live app (Netlify, deprecated) | https://magenta-phoenix-fcca34.netlify.app |
| Firebase console | https://console.firebase.google.com/project/compas-d2b35 |
| Firebase Hosting docs | https://firebase.google.com/docs/hosting |
| Claude Code docs | https://docs.claude.com/en/docs/claude-code/overview |
| YouTube IFrame API | https://developers.google.com/youtube/iframe_api_reference |
| Web Audio API | https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API |
| Web Speech API | https://developer.mozilla.org/en-US/docs/Web/API/Web_Speech_API |
| Screen Wake Lock API | https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API |
