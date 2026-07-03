# Handoff — Compás (continue from another account)

*Written 2026-07-02. Read this + `CLAUDE.md` first; they make any new session fully current.*

## What this project is
Compás is a single-file PWA (no build step — everything in `public/index.html`) that speaks
Latin-dance choreography cues over a YouTube or local-audio track. It is live on Firebase
Hosting and mid-way through being published to Google Play as an Android TWA.

- **Repo:** `taverasj1/compas` (GitHub)
- **Live site:** https://compas-d2b35.web.app (Firebase project `compas-d2b35`)
- **Privacy policy (Play requirement):** https://compas-d2b35.web.app/privacy.html

## Where things stand (verified as of 2026-07-02)

### Done ✅
- PWA complete and deployed: manifest (with `id` + `categories`), service worker
  (`CACHE_NAME = "compas-v2"`), icons (192 / 512 / 512-maskable), offline shell.
- App features shipped: spoken cues, A–B loop, speed control, AI auto-cue (Gemini, key via
  `x-goog-api-key` header), perf work (tick loop stops when paused, 10Hz DOM throttle,
  interpolated YouTube time).
- **Android TWA built** via PWABuilder against the live URL:
  - Package name: `app.web.compas_d2b35.twa`
  - Artifacts: `.aab` (for Play upload), `.apk` (for sideload testing), `signing.keystore`
    + key-info file — **held by the user only; gitignored; never commit them.**
- **Digital Asset Links live and verified:**
  `https://compas-d2b35.web.app/.well-known/assetlinks.json` serves the real package name +
  upload-key SHA-256 (`5D:74:88:…:DF:7A`). The `firebase.json` `**/.*` ignore bug that
  silently dropped `.well-known/` was fixed.
- Play launch kit written: `docs/play-store-listing.md` (paste-ready listing copy, Data
  Safety answers, graphics checklist, publish flow).
- All of the above merged to `main` (commit `bbdc280`); live deploy matches it.

### Blocked / waiting ⏳
- **Google Play developer account verification** (started ~2026-06-13, user: taverasj1@gmail.com).
  Everything else is staged; publishing waits only on this.

### Remaining steps (in order, once the account clears)
1. Play Console → create app → paste listing from `docs/play-store-listing.md` §2, Data
   Safety §3, content rating.
2. Upload the `.aab` to **Internal testing**.
3. **CRITICAL GOTCHA:** Play re-signs the app (Play App Signing). Get the **App signing key
   certificate SHA-256** from Play Console → Setup → App integrity, **append** it to the
   `sha256_cert_fingerprints` array in `public/.well-known/assetlinks.json` (keep the
   existing one — it covers sideloaded builds), then `firebase deploy`. Skipping this leaves
   a browser URL bar on Play-installed builds.
4. Still-missing store assets: feature graphic 1024×500 + ≥2 phone screenshots
   (specs & ideas in `docs/play-store-listing.md` §4).
5. Internal test on a device (expect: no URL bar, cues speak) → promote to Production.

## Environment & operational facts a new session needs

- **Remote sandboxes cannot deploy or push to Play** — no Firebase credentials there.
  Deploys run on the user's Windows machine: `firebase deploy` from the repo folder
  (Firebase CLI installed, logged in as taverasj1@gmail.com).
- **Local clones:** the user has had several OneDrive-synced clones across two machines
  (`…\Cloude\compas`, `…\Claude\Projects\compas`, `…\Claude\compas`). Before trusting any
  local folder: `git fetch && git status -sb` and compare to `origin/main`.
  Consolidating to one clone outside OneDrive is recommended (see improvements doc).
- **Branch state:** app code source of truth is `main`. The long-running work branch
  `claude/access-verification-E5wnV` is merged for app code but carries two later
  docs-only commits (`docs/setup-improvements.md`, this file) — merge it into `main`
  (fast-forward) then delete it.
- **Conventions that matter when editing** (full list in `CLAUDE.md`):
  - All playback calls go through the `pb*` helpers, never `player.*`/`audioEl.*` directly.
  - Bump `CACHE_NAME` in `public/sw.js` in any commit that changes `index.html`.
  - Don't add per-frame DOM writes to `tick`; the loop must keep stopping when paused.
- **Deferred improvement backlog:** `docs/setup-improvements.md` — headline item is a
  GitHub Action for auto-deploy on push to `main` (one-time local step:
  `firebase init hosting:github` to mint the service-account secret). App-feature backlog
  is in `CLAUDE.md` (next up: per-cue notes, tap-tempo BPM).

## Access the new account will need
- GitHub repo `taverasj1/compas` (or a fork/transfer).
- Firebase project `compas-d2b35` (console access: add the new account as a member in
  Firebase console → Project settings → Users and permissions).
- Google Play Console (the developer account being verified belongs to taverasj1@gmail.com;
  apps can be transferred between Play accounts later if needed).
- The signing keystore + its passwords (user holds these privately — without them the
  sideload/test APK can't be rebuilt; Play App Signing protects the store key).

## Secrets hygiene (do not regress)
`.gitignore` blocks `*.keystore`, `*.jks`, `*.aab`, `*.apk`, and the key-info text files.
Keep it that way; the keystore passwords exist only in the user's private copy of
`signing-key-info.txt`.
