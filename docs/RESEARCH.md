# OpenAir — Feasibility Research

_Research date: 2026-10-09_

**Verdict: Yes, this is possible.** The band can't stream data to us directly, but the Google Health app already uploads everything to Google's cloud, and the **Google Health API v4** lets a personal app read it back: HRV, sleep, SpO2, resting HR, skin temperature, intraday heart rate, and more. The API can also **push a webhook when new data lands**, so we get near-real-time updates and alerts without polling.

There is one catch. **No public API exposes a "Stress score"** (or Readiness, Sleep Score, or Cardio Load). OpenFit doesn't show one either; its docs say so directly. We'll **compute our own stress and recovery estimates** from HRV, resting HR, sleep, and intraday HR. That's an improvement on what the Google Health app shows today, as long as we label it as our estimate.

---

## 1. How OpenFit works (reference project)

Source: <https://github.com/FlavioAdamo/openfit> (`package.json` says `"license": "UNLICENSED"`. **Use it for ideas only. Don't copy its code.**)

| Aspect | OpenFit |
|---|---|
| Form factor | Electron desktop app (macOS/Windows/Linux) |
| Data source | Google Health API v4 (primary), legacy Fitbit Web API (fallback) |
| Auth | Your own Google Cloud project, Web OAuth client, PKCE, loopback redirect `127.0.0.1:42813` |
| Scopes | 9 read-only `googlehealth.*` scopes + `openid`, `profile` |
| Storage | Encrypted per-day cache on disk (Electron `safeStorage`) |
| UI | React + Tailwind v4 + shadcn; Today / Activity / Health / Sleep / Body / Devices |
| Extras | Codex-powered chat assistant (read-only) |
| Stress / Readiness | **Not shown.** Its docs say "Readiness Score, Cardio Load, Sleep Score, Stress Score … are not exposed. OpenFit shows base measurements and local insights." |

Ideas worth carrying over:
- **Compare to a personal baseline, don't diagnose.** HRV, RHR, SpO2, respiration, and skin temperature are compared with your own rolling average, not population norms.
- **Home answers four questions:** How's today going? What's different from my baseline? Which data explains it? Where's the detail?
- **Independent reads.** A failed ECG or temperature call never blocks steps or sleep.
- **Rate limit:** keep requests under about 5 req/s and back off on `429`.
- **Hide what's missing.** Sections without data disappear instead of showing zeros.

What we change:
- **Phone-first.** It installs from the browser as a PWA, sits on the home screen, and runs full-screen like an app.
- **Server-side sync** instead of a desktop loopback, so data updates while your phone is in your pocket.
- **Derived Stress and Recovery** scores, which OpenFit leaves out.
- **Push alerts.**

---

## 2. The data path (why "sync through Google Health, then pull" is the only route)

```
Fitbit Air ──proprietary BLE──▶ Google Health app (phone) ──▶ Google Health cloud
                                                                   │
                                         webhook "new data" ◀──────┤
                                                                   │
                     OpenAir server ──OAuth + REST v4 (read)───────┘
                           │
                           ▼
                     OpenAir PWA on your phone
```

- **No public Bluetooth interface.** Reverse-engineering the band's BLE protocol is fragile and against ToS. OpenFit made the same call.
- **We can't force the band to sync.** The Google Health app syncs the Air in the background on its own schedule. Opening that app forces a sync. Our "auto sync" means: (a) a webhook fires as soon as Google's cloud has new data, (b) a scheduled poll as a fallback, and (c) a "Sync now" button that pulls the latest from the cloud and links into the Google Health app to nudge a band sync.

---

## 3. Google Health API v4: what we can read

Base: `https://health.googleapis.com/v4/users/me/dataTypes/{type}/dataPoints`
Methods: `list` (intraday/granular), `rollUp`, `dailyRollUp`, `reconcile`, plus `users.getProfile`, `getSettings`, `pairedDevices`.

| Category | Data types | Fitbit Air? |
|---|---|---|
| Heart | `heart-rate` (~5 s intraday), `daily-resting-heart-rate`, **`heart-rate-variability`** (intraday, sleep), **`daily-heart-rate-variability`** | ✅ |
| Sleep | `sleep` (sessions + stages), respiratory-rate sleep summary | ✅ |
| Oxygen | `oxygen-saturation` (validated minute-level), `daily-oxygen-saturation` | ✅ |
| Temperature | `daily-sleep-temperature-derivations` (nightly skin-temp delta), `skin-temperature-sensors` (v4beta, added 2026-10-01) | ✅ |
| Activity | `steps`, `distance`, `floors`, `active-zone-minutes`, `activity-level`, `sedentary-period`, calories, `exercise` | ✅ |
| Fitness | `vo2-max`, `run-vo2-max` | ✅ (cardio fitness) |
| Rhythm | `irregular-rhythm-notification` | ✅ (where available) |
| ECG | `electrocardiogram` | ❌ Air has no ECG |
| Body / nutrition | `weight`, `body-fat`, `blood-glucose`, food, hydration | Logged data only |
| **Stress, Readiness, Sleep Score, Cardio Load** | — | ❌ **not in the API** |

**Known risk:** a Validic migration guide reported that HRV and SpO2 "weren't reporting as expected" for a while and that HRV attaches to the main sleep record. **The Phase 0 spike must confirm real HRV values come back for your account.**

### Webhooks (auto-sync trigger)
- **Subscriber** (project-level): an HTTPS endpoint, the data types, and a shared secret sent in `Authorization`. Creating one runs a 2-step verification handshake: an authorized POST must return 200, and an unauthenticated one must return 401 or 403.
- **Subscriptions** (per user): `AUTOMATIC` (created when you grant consent) or `MANUAL`.
- **Supported:** HRV, daily HRV, sleep, resting HR, heart rate, SpO2, respiratory rate, skin temperature, steps, exercise, and more.
- **The payload has no values.** It carries `healthUserId`, `dataType`, `operation` (UPSERT/DELETE), and time `intervals`. We then call the API for those intervals.
- **Delivery:** return `204` right away. Failed deliveries are retried with backoff for up to 7 days. Up to 99 messages per batch. Duplicates can happen, so handlers must be idempotent.
- Payloads are ECDSA-signed (`GOOGLE-HEALTH-API-SIGNATURE` header, per a secondary source; verify against the official docs).

### OAuth gotchas (important for a personal app)
- **The Fitbit Web API shuts off on 2026-10-30.** Old Fitbit tokens don't carry over. We build on Google Health v4 only.
- **Testing mode means refresh tokens expire every 7 days** (constant re-login). **Fix:** set the consent screen to **"In production"** while staying unverified. Refresh tokens then last until revoked or about 6 months unused. Unverified apps are capped at 100 users and show an "unverified app" warning, which is fine for personal use.
- All `googlehealth.*` scopes are **Restricted**. Public distribution (more than 100 users) requires Google's security review. That's not needed for personal use.
- **Client type:** Web Server client with a redirect URI. The secret lives on our server, never in the phone app. This also fixes the problem OpenFit flags for itself, where a secret ships inside a desktop binary.
- Don't send `include_granted_scopes=true` if a legacy `fitness.*` consent exists on the same client. It caused 403 errors in third-party testing.

---

## 4. Getting it onto your phone: options compared

> **Superseded (2026-10-09):** you're on iPhone, and we want direct Bluetooth (see [BLE_RESEARCH.md](BLE_RESEARCH.md)), so the choice is a **native iOS app (SwiftUI)**, not a PWA. iOS Safari has no Web Bluetooth. The table below is kept for reference. Current plan: [PLAN.md](PLAN.md).

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **A. PWA + small cloud backend** | One codebase; works on iPhone and Android; installs to the home screen; Web Push alerts; server syncs 24/7 via webhooks | iOS push only works once installed to the home screen (iOS 16.4+); no home-screen widgets | **✅ Recommended. Start here.** |
| B. Native Android app reading Health Connect | Fully on-device, no Google Cloud project | Android-only; Health Connect coverage of HRV and skin temperature from Google Health is unconfirmed; no server push | Possible later add-on |
| C. iOS app reading HealthKit | Native | Google Health → Apple Health writes **don't include HRV** (HRV is read-only, per 2026 reports) | ❌ Defeats the HRV goal |
| D. Capacitor wrap of the PWA | Real app icon, native notifications, widgets, store-ready | Extra build tooling | Phase 5 upgrade path |

"Worst case it's a website with a link on my phone": option A is exactly that, plus an app icon, full-screen launch, offline cache, and push notifications.

---

## 5. Stress, done honestly

Fitbit's own Stress Management Score uses EDA scans on some devices plus HRV, sleep, and responsiveness. **It isn't in the API, and we can't reproduce it exactly.** What we can build from data we do get:

| OpenAir metric | Inputs | Notes |
|---|---|---|
| **Stress Load (0–100, daily)** | HRV vs 28-day baseline (inverted z-score), RHR vs baseline, sleep debt vs goal, minutes of elevated HR while sedentary | Clearly labeled as an estimate |
| **Stress moments (intraday)** | HR ≥ RHR + 20–25 bpm for ≥ 10 min while steps ≈ 0 | Similar to Fitbit's "body response" idea |
| **Recovery (0–100, morning)** | HRV, RHR, sleep duration/efficiency, respiratory rate, skin-temp delta vs baseline | Our take on "Readiness" |
| **Illness watch** | RHR ↑, resp. rate ↑, skin temp ↑, HRV ↓ together for 2+ nights | Alert candidate |

Rule: every score shows its contributing inputs, so you can always see why.

---

## Sources
- OpenFit repo & docs: <https://github.com/FlavioAdamo/openfit> (`docs/DATA_COVERAGE.md`, `docs/ARCHITECTURE.md`, `docs/HOME_DASHBOARD_MODEL.md`)
- [Google Health API: API specifications](https://developers.google.com/health/migration/api-specifications)
- [Google Health API: Release notes](https://developers.google.com/health/release-notes)
- [Google Health API: Webhook subscriptions](https://developers.google.com/health/webhooks)
- [Google Health API: Set up Google Cloud and OAuth](https://developers.google.com/health/setup)
- [Google Health API: Data access & authorization](https://developers.google.com/health/migration/data-access)
- [Terra: How the new Google Health API works](https://tryterra.co/blog/everything-you-need-to-know-about-google-health-new-api)
- [Validic: Fitbit → Google Health metric mapping](https://help.validic.com/space/VCS/5436080132/Fitbit+to+Google+Health+Metric+Mapping+Guide)
- [Sahha: Fitbit Web API shutdown](https://sahha.ai/blog/fitbit-api-sunset-migration/)
- [Google Health Help: third-party devices / Health Connect / Apple Health](https://support.google.com/googlehealth/answer/14236613?hl=en)
- [Motion: syncing Google Health to Apple Health / Health Connect](https://motion-app.com/blog/sync-google-health-to-apple-health/)
- [Google Store: Fitbit Air](https://store.google.com/product/google_fitbit_air?hl=en-US)
