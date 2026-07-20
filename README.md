# DRM Time & Motion Study

Single-file web app for the Digital Risk Management team (LeanGroup WFM) to time
**Offer Vetting** and **CargoNet / BOLO** sessions, track average handle time per
agent, and log approval-wait separately from active handle time.

Backend: **Firebase Firestore** (project `lean-wfm`), anonymous sign-in, real-time
multi-agent sync. The UI never mentions Firebase — it only shows "Database".

---

## Live app

Once deployed (see below): `https://<username>.github.io/<repo>/`

All agents opening this URL share one live database. Data syncs in real time; if a
connection drops, entries queue on the device and sync automatically on reconnect.

---

## Deploy on GitHub Pages

1. This repo already contains **`index.html`** (the app). Nothing to edit — the
   Firebase config is baked in.
2. **Settings → Pages → Source: Deploy from a branch → `main` / `/root` → Save.**
3. Wait ~1 minute, then open `https://<username>.github.io/<repo>/`.

> The app loads the Firebase SDK as ES modules from Google's CDN, so it must be
> served over http(s) (GitHub Pages, Netlify, or a local `python -m http.server`).
> Opening `index.html` by double-click (`file://`) will not load the SDK.

---

## Firebase — one-time setup (already partly done)

In the [Firebase console](https://console.firebase.google.com) for project **lean-wfm**:

1. **Firestore Database** — created (`(default)`). ✅
2. **Authentication → Sign-in method → Anonymous → Enable.**
   Agents never log in; the app signs in anonymously on load. Without this the
   header pill shows "Offline".
3. **Firestore → Rules** — paste and **Publish**:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /drm_sessions/{docId} {
         allow read, write: if request.auth != null;
       }
     }
   }
   ```
4. **Authentication → Settings → Authorized domains → Add domain** →
   `<username>.github.io`
   (Also add any custom domain, or your Netlify domain, if used. `localhost` is
   already authorized for local testing.)

---

## A note on the API key

The Firebase web config (including `apiKey`) is committed in `index.html`. **This is
expected and safe** — Firebase web keys are designed to ship in client code and are
not secrets. Access to the data is controlled by the Firestore security rules above
(`request.auth != null`), not by hiding the key.

---

## Data model — Firestore collection `drm_sessions`

One document per session, keyed by the session `id`.

| Field | Type | Meaning |
|---|---|---|
| id | string | Document key |
| agent | string | Agent name / ID as typed |
| proc | string | `vetting` or `cargonet` |
| ref | string | Load / reference number |
| start / end | number | Epoch ms |
| handleSec | number | Active handle time (seconds) |
| outcome | string | approved / dnu / sent_approval / escalated / no_action |
| approvalEnd / approvalWaitSec | number \| null | Approval wait, kept separate from AHT |
| updatedAt | number | Last-write timestamp |

The collection is created automatically on the first saved session — no need to
"Start collection" manually in the console.

---

## Getting data out for analysis

- **In-app:** the **Export CSV** button (Session Log tab). Times are formatted and
  approval waits are in minutes, with clean-AHT and gross-AHT (incl. approval)
  as separate columns.
- **Bulk / automated:** Firestore → BigQuery via the official
  "Stream Firestore to BigQuery" extension, or a small Admin SDK collector for a
  scheduled feed into capacity workbooks.

---

## Targets & metrics

- **Offer Vetting** target: 20 min average (15–25 typical). Timer turns amber past
  target, red past 1.5×.
- **CargoNet / BOLO** target: 10 min.
- **Approval wait** is timed separately when an outcome of "Sent for Approval" is
  chosen, and is **excluded** from vetting AHT so handle-time stays clean.

---

## Local testing

```
python -m http.server 8000
# then open http://localhost:8000/index.html
```
