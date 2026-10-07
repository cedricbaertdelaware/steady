# Steady — recurring costs, utilities and savings tracker

Single-file, local-first PWA. `index.html` contains everything (HTML, CSS, JS). The companion files are optional: without them the app still works fully, you only lose the installable icon, offline caching of the shell, and the home-screen manifest.

| File | Purpose |
|---|---|
| `index.html` | The whole app (works by double-clicking it) |
| `manifest.webmanifest` | PWA manifest (name, icons, shortcuts) |
| `sw.js` | Service worker: caches the app shell for offline use |
| `icon-192.png`, `icon-512.png` | Home-screen icons |

Open `index.html?selftest=1` to run the built-in test suite (98 tests: recurrence, holidays, normalisation, forecasts, advance adequacy, inflation, anomalies, contracts, savings maths, cash flow, CSV parsing, detector, ICS validity, encryption, merge, migration, money rounding, backup round trip).

---

## 1. Deployment

Upload the five files to any static host (GitHub Pages, Cloudflare Pages, Netlify, an S3 bucket, a NAS). No build step, no server code.

- **HTTPS is required** for PWA install, the service worker, WebCrypto (PIN hashing and encryption) and browser notifications. GitHub Pages and the others above give you HTTPS for free.
- GitHub Pages: push the files to a repo, enable Pages on the branch root (or `/docs`). If the app lives in a sub-folder (`https://user.github.io/steady/`), keep all five files in that same folder; all paths are relative.
- After each release, bump `CACHE` in `sw.js` so installed clients fetch the new `index.html`.
- Opening `index.html` directly from disk (`file://`) works too (no service worker, Google Fonts may be blocked; the font stack falls back to the system UI font).

## 2. Firebase setup (optional — phone ↔ laptop sync)

Sync is **off by default** and the sync module stays completely inert while `CONFIG.firebase` is empty.

1. Create a Firebase project. In **Build → Authentication → Sign-in method**, enable **Email/Password** and **Google**.
2. In **Build → Firestore Database**, create a database in a European region (e.g. `europe-west1`), production mode.
3. **Project settings → Your apps → Web app**: copy the config object and paste it into `CONFIG.firebase` at the top of the script in `index.html`:
   ```js
   firebase: { apiKey:'…', authDomain:'…', projectId:'…', appId:'…', storageBucket:'…' }
   ```
   (`storageBucket` only matters if you later turn on attachment upload; it is off by default.)
4. **Authentication → Settings → Authorised domains**: add the domain you host the app on.
5. Apply these **Firestore security rules** (Firestore → Rules):
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{uid}/{document=**} {
         allow read, write: if request.auth != null && request.auth.uid == uid;
       }
     }
   }
   ```
6. Sign in from **Settings → Sync** on both devices. The first device uploads; the second downloads and **merges** (per record, newest `updatedAt` wins, deletions are tombstones, overwritten edits are listed in the conflict log). The Firebase SDK (pinned 10.12.2 compat build) is only downloaded when a config is present.

Data layout: `users/{uid}/records/{type}__{id}` with `{ type, updatedAt, deleted, payload }`. If end-to-end encryption is on, `payload` is an AES-GCM blob that Firebase cannot read.

**No Firebase?** Use **Sync via file**: Import & export → Full JSON backup on device A, then Import → *Merge* on device B. Same merge rules, nothing is lost.

## 3. Tunable constants (`CONFIG`, top of the script)

| Constant | Default | Meaning |
|---|---|---|
| `saveDebounceMs` | 300 | Debounce before persisting |
| `backupsToKeep` | 7 | Automatic daily snapshots kept (restorable in Settings) |
| `recycleBinDays` | 30 | Retention of deleted records |
| `occurrenceWindowPastMonths` / `FutureMonths` | 36 / 14 | How far generated payments are materialised |
| `renewalsBoardDays` / `renewalsActNowDays` | 120 / 30 | Contract board thresholds |
| `missingPaymentDays` | 5 | Days past due before a "missing payment" anomaly |
| `advanceAdequateBandPct` | 10 | ±% band considered adequate for utility advances |
| `pensionYearEndReminder` | `12-01` | Date (MM-DD) of the pension top-up reminder |
| `weeklyDigestDay` | 1 | Monday digest |
| `pbkdf2Iterations` | 150 000 | PIN hashing / key derivation |
| `pinThrottleBaseMs` | 2000 | Base delay after a wrong PIN (doubles each attempt) |
| `maxAttachmentBytes`, `imageMaxEdge` | 8 MB, 1600 px | Attachment limits |
| `storageWarnPct` | 80 | Warn at this % of the storage quota estimate |
| `demoMonths` | 24 | Months of generated demo history |
| `detector.*` | see code | Bank-CSV detector: min transactions, regularity, CV threshold, interval bands |
| `perYear`, `daysPerYear` | 365.2425 basis | Normalisation factors |

Per-user defaults (reminder lead times, quiet hours, forecast and anomaly thresholds, weekend rule, fiscal year start, holidays) live in **Settings** and are stored with the data. Holiday sets per country are in `holidays()` (Easter via Meeus/Jones/Butcher); provider lists, templates and the keyword dictionary are in the `CATALOGUE` block.

## 4. Privacy statement

- All data (costs, payments, readings, attachments, settings) is stored **on the device**, in IndexedDB (`steady-db`), with automatic fallback to localStorage and then memory. Nothing is sent anywhere unless you sign in to sync.
- No analytics, no trackers, no third-party scripts. The only external requests are Google Fonts (cosmetic, falls back to system fonts) and, **only if you configure it**, Firebase.
- The **PIN** is a privacy lock (PBKDF2-hashed, throttled after wrong attempts). It stops casual access; it does not encrypt data.
- **End-to-end encryption** (Settings → Security) encrypts the stored state, backups and synced payloads with AES-GCM using a key derived from your passphrase (PBKDF2, 150 000 iterations, random salt, random IV per record). **The passphrase is never stored.** If you lose it, the data cannot be recovered — export a plain backup first if you want a safety net.
- Accounts hold a label and at most the last 4 characters. The app warns if something that looks like a full IBAN or card number is typed into a free-text field and offers to mask it. Demo data is fictional and removable with one tap.

## 5. Moving your data to a new phone

On the old phone open **Import & export → Full JSON backup** (tick *Include attachments* if you want your invoices and meter photos too) and save the file to your cloud drive or send it to yourself. On the new phone open the app once, finish onboarding with **Start fresh**, go to **Import & export → Restore from JSON**, pick the file and choose **Replace** (or **Merge** if the new phone already has data). If you use Firebase sync, simply sign in on the new phone instead — it downloads and merges automatically. Either way, install the app from the browser menu ("Add to Home Screen") so it runs offline with its own icon, and set a PIN or passphrase again on the new device (security settings are deliberately not synced).
