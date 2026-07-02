# Claude Code Setup — Improvement Candidates

*Generated 2026-07-02 by reflecting on all Compás sessions (June 5 → July 2, 2026).*

**Method:** Four sub-agents pulled raw signals from the three session transcripts in this
environment (one continuously-resumed thread, 18 human messages across 5 active days), the
repo's git history, and its docs. Two agents (user-friction, repo-process) returned full
extractions; two (tool-errors, environment) hit a session token limit, so those signal sets
were backfilled from direct grep counts and in-session observation. Signals were clustered
across sessions; each cluster gets a verdict: **new skill / automation / fix / nothing**.

---

## Summary

| # | Cluster | Verdict | Leverage | Effort |
|---|---------|---------|----------|--------|
| 1 | Manual deploy gap & human context-ferrying | **Automation** — GitHub Action auto-deploy | ★★★★★ | ~30 min |
| 2 | Multi-machine, multi-clone OneDrive confusion | **Fix** — one clone per machine, outside OneDrive | ★★★★☆ | ~20 min |
| 3 | Zero automated verification | **Automation** — tiny CI check job | ★★★★☆ | ~30 min |
| 4 | CACHE_NAME bump ritual (memory-enforced) | **Automation** — derive/check in CI | ★★★☆☆ | folds into #1/#3 |
| 5 | `.claude/` is gitignored → no project skills possible | **Fix** — un-ignore, keep `settings.local.json` ignored | ★★★☆☆ | 5 min |
| 6 | Play Store release flow | **Nothing now** → optional skill after #5 | ★★☆☆☆ | — |
| 7 | Permission denials & tool misfires | **Nothing** (safety worked as intended) | — | — |
| 8 | MCP connector noise/flapping | **Fix** — detach unused connectors from code sessions | ★★☆☆☆ | 5 min |
| 9 | Branch/staleness hygiene | **Fix** — delete merged branch, work from `main` | ★☆☆☆☆ | 2 min |

**If you do only one thing: #1.** It answers the very first question you asked this thread
("if I wanted to do a live update of a feature from here, you would not be able to do so?")
— with an Action, remote sessions *can* ship live updates, end to end.

---

## 1. Manual deploy gap & human context-ferrying → **Automation**

**Evidence (strongest cluster by far):**
- The sandbox has no Firebase credentials, so every deploy required you at a keyboard:
  "The sandbox has no GitHub credentials, so the push has to run from your machine."
- The PWABuilder "Missing Name" incident was purely a deploy-ordering bug: code was correct,
  the live site was stale.
- You acted as the transport between four surfaces (this remote thread, a Cowork session, a
  local CLI, raw PowerShell): a 3,602-char Cowork summary pasted in, a 16,665-char local CLI
  transcript pasted in, six build artifacts uploaded by hand, and a remote session left
  "standing by" for values only you could fetch.
- Unpushed Cowork work sat at risk in an ephemeral container until you manually pushed it.

**Candidate:** Add `.github/workflows/deploy.yml` using `FirebaseExtended/action-hosting-deploy`:
- On push to `main` → deploy to the live channel of `compas-d2b35`.
- On pull requests → deploy a **preview channel** and comment the preview URL on the PR
  (you can eyeball a change from your phone before merging — huge for remote-mobile usage).
- One-time setup on your machine: `firebase init hosting:github` generates the service-account
  secret (`FIREBASE_SERVICE_ACCOUNT_COMPAS_D2B35`) in the repo automatically.

**Result:** any session — remote, mobile, local — ships by merging to `main`. No more
"deploy from the right folder on the right machine."

## 2. Multi-machine, multi-clone OneDrive confusion → **Fix**

**Evidence:** Three distinct Windows paths were used for the same repo, all inside OneDrive
(one work tenant, one personal): `…\Cloude\compas`, `…\Claude\Projects\compas`,
`…\Claude\compas`. Sessions repeatedly opened with orientation questions: "do you have
access to this", "Can you see the updates made to the folder?", "where is the compas folder?",
"is this folder up to date?". One CLI session launched from the home directory instead of the
repo. Local `main` on the sandbox sat 10 commits behind for weeks.

**Why it matters:** OneDrive syncing a `.git` directory across machines is a known corruption
and conflicted-copy risk — and it's redundant: GitHub *is* the sync. Multiple clones also mean
"which folder is real?" never has a stable answer.

**Candidate fix (one-time, per machine):**
1. Pick one canonical path outside OneDrive, e.g. `C:\dev\compas`, and `git clone` fresh there.
2. Delete (or archive) the OneDrive copies after confirming `git status` is clean and pushed.
3. Habit: start any local work with `git pull`, end with `git push`. With cluster #1 in place,
   most days you won't need a local clone at all.

## 3. Zero automated verification → **Automation**

**Evidence:** The app is a 1,011-line single-file `index.html` with a ~1,000-line inline IIFE.
There are no tests, no linter, no CI, no JSON validation — nothing mechanical guards a deploy.
All verification is human: open in Chrome, install on a phone, listen for spoken cues. The one
regression that did happen (`**/.*` ignore silently dropping `assetlinks.json` from deploys)
is exactly the class of failure a post-deploy check catches.

**Candidate:** A `checks` job (same workflow as #1, runs before deploy):
- Extract the inline `<script>` and run `node --check` on it (catches syntax errors in the IIFE).
- `python3 -m json.tool` on `manifest.webmanifest`, `public/.well-known/assetlinks.json`,
  `firebase.json`.
- After deploy: `curl` the live `manifest.webmanifest` and `.well-known/assetlinks.json`,
  fail loudly if either 404s or serves HTML (guards the `**/.*` footgun forever).

## 4. CACHE_NAME bump ritual → **Automation**

**Evidence:** The convention ("bump `CACHE_NAME` in any commit that changes `index.html`") is
stated in three places with three phrasings (CLAUDE.md, sw.js header, play-store doc) and is
enforced only by memory. History shows it's never actually been exercised post-SW — its
durability is unproven.

**Candidate (pick one):**
- **Best:** eliminate the ritual — in the deploy Action (#1), `sed` the deployed `sw.js` so
  `CACHE_NAME = "compas-<short-sha>"`. Every deploy gets a unique cache automatically; the
  repo convention becomes obsolete.
- Lighter: a CI check that fails the build if `index.html` changed and `sw.js` didn't.

## 5. `.claude/` is gitignored → **Fix**

**Evidence:** The repo's `.gitignore` excludes `.claude/` entirely, so project-level Claude
Code config — committed skills, permission allowlists, hooks — cannot exist. Every session
re-derives conventions from CLAUDE.md prose alone.

**Candidate:** Replace `.claude/` in `.gitignore` with `.claude/settings.local.json`.
That unlocks committing `.claude/settings.json` (e.g. allowlist `firebase deploy`,
`npx serve`) and project skills (see #6).

## 6. Play Store release flow → **Nothing now**, optional skill later

**Evidence:** The flow is well-documented and it worked — `docs/play-store-listing.md`
carried the listing copy, Data Safety answers, and checklist across sessions. Remaining
steps are mostly one-time (upload `.aab`, add the **Play App Signing SHA-256** to
`assetlinks.json` — the one remaining gotcha — screenshots, feature graphic).

**Verdict:** The docs are the right tool for a mostly-one-time flow. If updates become
recurring (versionCode bumps, new `.aab`s), add a small project skill `/release-android`
(possible once #5 lands) that walks the checklist and edits `assetlinks.json` given a
fingerprint. Not worth building today.

## 7. Permission denials & tool misfires → **Nothing**

**Evidence:** 2 permission denials, 5 tool-use errors, 4 input-validation errors across the
thread. The notable denial — `git reset --hard` blocked by the safety classifier — was
*correct*: it protected potentially-unpushed work, and the merge-instead workaround was
better anyway. The rest were transient assistant-side mistakes (wrong tool param,
edit-before-read) that self-corrected within one step.

**Verdict:** Working as intended. No setup change earns its keep here.

## 8. MCP connector noise/flapping → **Fix (small)**

**Evidence:** Sessions carried many MCP connectors irrelevant to this project (Canva,
Descript, Higgfield, Google Drive…), and servers flapped repeatedly (dozens of
connect/disconnect notices in the current session alone). Cost is noise, latency, and
occasional "tool unavailable" moments mid-task.

**Candidate:** In claude.ai → Settings → Connectors, detach the media/design connectors from
coding sessions (or scope them to the projects that use them). Keep GitHub.

## 9. Branch/staleness hygiene → **Fix (trivial)**

**Evidence:** `claude/access-verification-E5wnV` (named after the June 5 "do you have access"
test!) remained the working branch for a month, including after being fully merged to `main`.
Sandbox-local `main` sat 10 commits behind.

**Candidate:** Delete the merged branch (local + remote); let each new piece of work get a
fresh, descriptively-named branch off current `main`. Zero tooling needed — just the habit,
which future Claude sessions will follow if `main` is the visible default.

---

## Suggested order of attack

1. **#5** (5 min, unblocks committing project config) →
2. **#1 + #3 + #4 as one workflow file** (~1 hour total: checks → deploy → post-deploy probe,
   with CACHE_NAME derived per-deploy) →
3. **#2 + #9** housekeeping the next time you're at each machine →
4. **#8** whenever the connector noise next annoys you.

Items #6 and #7 need nothing today.
