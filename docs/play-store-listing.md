# Google Play — launch prep (Compás)

Everything here can be prepared **before** the Play developer account finishes
verification. The only thing that truly needs the verified account is the final
upload + publish. Copy/paste straight from this file into the Play Console.

---

## 1. While you wait — do these now (no account needed)

1. **Build the TWA** at https://www.pwabuilder.com → paste
   `https://compas-d2b35.web.app` → **Package For Stores → Android**.
   - Download the `.aab` (the upload artifact) **and** the signing key `.zip`
     (keep it safe — losing it blocks future updates).
   - Note the two values PWABuilder shows you:
     - **Package name** (e.g. `app.web.compas_d2b35.twa`)
     - **SHA-256 fingerprint**
   - Send those two values back to Claude → they go into
     `public/.well-known/assetlinks.json`, then `firebase deploy`.
2. **`firebase deploy`** so the live PWA + assetlinks file are public (the TWA
   verifies against the live site).
3. **Install the PWA on your phone** (Chrome → Add to Home screen) and confirm:
   cues speak aloud, A–B loop works, offline opens the shell.
4. **Capture screenshots** from that phone session (see §4 for specs).

---

## 2. Store listing copy (paste-ready)

**App name:** `Compás — Dance Cue Trainer`

**Short description** (≤80 chars):
> Choreography cue trainer: hear your Latin dance moves called out on the beat.

**Full description** (≤4000 chars):
> Compás calls your choreography out loud so you can keep your eyes up and your
> feet moving. Place a cue on any moment of a song — "vuelta," "enchufla,"
> "sombrero" — and Compás speaks it just before the move arrives, in Spanish or
> English.
>
> Built for Latin social dance practice: salsa, bachata, cumbia, and more.
>
> • Works with YouTube or your own audio files
> • Spoken cues in Spanish or English, timed with an adjustable lead-in
> • A–B loop any tricky section and drill it on repeat
> • Slow the music to 0.5× without changing the pitch
> • AI auto-cue (optional): generate a starting set of cues from a video
> • Count-in ("cinco, seis, siete, ocho") before you start
> • Keeps the screen awake mid-rehearsal
> • Works offline once loaded (local audio); installable as an app
> • No account, no ads, no tracking — your cues stay on your device
>
> Whether you're a teacher building a routine or a student drilling a combo,
> Compás keeps the count so you don't have to.

**What's new** (release notes, v1):
> First release: timed spoken cues, A–B looping, slow-down, AI auto-cue, and
> offline support.

**Category:** Health & Fitness (alt: Music & Audio)
**Tags:** dance, choreography, salsa, bachata, practice
**Contact email:** taverasj1@gmail.com
**Privacy policy URL:** https://compas-d2b35.web.app/privacy.html
**Website:** https://compas-d2b35.web.app

---

## 3. Data Safety form answers

Compás has **no backend and no developer-side data collection** — cues,
settings, and any Gemini API key live only in your browser's local storage.
Two third-party services process data **only when the user chooses to use
them**, and you must still declare these:

- **YouTube IFrame player** (when a YouTube track is loaded) — Google receives
  standard request data (approximate device info, IP-derived location, usage).
- **Google Generative Language API / Gemini** (only if the user enters a key
  and runs AI auto-cue) — the request is sent device → Google.

Suggested form answers (review against your final build):

| Question | Answer |
|---|---|
| Does your app collect or share user data? | Yes (via embedded YouTube + optional Gemini) |
| Is all data encrypted in transit? | Yes (HTTPS) |
| Can users request data deletion? | Data is on-device only; clearing app/browser data removes it |
| Data types — App activity | Shared (YouTube usage when a video is played) |
| Data types — Device or other IDs | Collected/shared by YouTube |
| Personal info / contacts / messages / photos | No |
| Is data used for tracking / ads? | No |

> The API key the user enters is **their own credential**, stored locally and
> sent only to Google's API — not collected by you. Note this in the form's
> free-text if asked.

---

## 4. Required graphics (gather while waiting)

| Asset | Spec | Status |
|---|---|---|
| App icon | 512×512 PNG, 32-bit | ✅ have `icons/icon-512.png` |
| Feature graphic | 1024×500 PNG/JPG (no alpha) | ⬜ needs making |
| Phone screenshots | ≥2, 16:9 or 9:16, 1080px+ on short side | ⬜ capture on device |
| (optional) 7"/10" tablet shots | only if targeting tablets | ⬜ |

Screenshot ideas: (1) a loaded track with cues on the timeline, (2) the cue
firing with the big spoken-move readout, (3) the AI auto-cue card, (4) the A–B
loop in action.

---

## 5. Publish flow (once verified)

1. Create app in Play Console → fill listing from §2, graphics from §4.
2. Complete Data Safety (§3) + content rating questionnaire (answer honestly:
   no violence/sexual/gambling content → expect "Everyone").
3. Upload the `.aab` to **Internal testing** first → install on your phone →
   confirm **no URL bar** (means assetlinks verified) and cues speak.
4. Promote Internal → Production. First review typically a few days.

---

## 6. Going forward

- Day-to-day content/feature changes: just `firebase deploy` — the installed
  app updates itself (TWA loads the live site).
- Only shell/manifest/package/version changes need a new `.aab` + resubmit
  (bump `versionCode`). Keep bumping `CACHE_NAME` in `sw.js` per existing convention.
