# OpenAir: Plan & Wireframes

_Phone-first Fitbit Air dashboard. **v2, 2026-10-09.** Evidence: [RESEARCH.md](RESEARCH.md) (Google Health API) and [BLE_RESEARCH.md](BLE_RESEARCH.md) (direct Bluetooth)._

## Decisions so far

| Question | Decision |
|---|---|
| Phone | **iPhone** |
| Data storage | **Cloud Postgres is fine** |
| Name | **OpenAir** |
| Data sources | **Hybrid**: live HR + RR **straight from the band over Bluetooth**, history from the **Google Health API** |
| App type | **Native iOS app** (iOS web apps can't use Bluetooth) built with **Expo / React Native** |
| Top alerts | _Still open, see [Open questions](#open-questions)_ |

### Why Expo instead of a web app or SwiftUI
- **iPhone Safari has no Web Bluetooth**, so a PWA can't talk to the band. Talking to the band needs a native app.
- **No Mac needed.** Expo's EAS Build compiles the iOS app in the cloud and installs it via TestFlight. (Assumption: you develop on Windows. If you have a Mac, SwiftUI is also an option.)
- `react-native-ble-plx` supports **iOS background Bluetooth** (`bluetooth-central` mode + state restoration), so it keeps receiving HR while the phone is locked.
- One TypeScript codebase across the app and the backend.
- **Cost:** the Apple Developer Program costs **$99/yr**. It's needed for TestFlight / long-lived installs; free provisioning expires every 7 days and requires a Mac.

## Goal

Keep syncing the Fitbit Air through the Google Health app as usual. OpenAir does three things:
1. Reads **live heart rate and beat-to-beat intervals directly from the band** to compute real-time HRV and stress.
2. Pulls **all history** (sleep, nightly HRV, SpO₂, skin temperature, steps) from the Google Health API.
3. Shows OpenFit-style dashboards plus what Google Health lacks (**HRV trends, Stress, Recovery**), and **alerts you** via push notifications.

## Architecture

```
                       ┌──────────── iPhone ─────────────────────────────┐
 Fitbit Air ──BLE──────┼─▶ Google Health app (unchanged, syncs history)  │
     │                 │                                                 │
     │ "Share heart    │   OpenAir app (Expo / React Native)             │
     │  rate" 0x180D   │   ├─ BLE live source: HR + RR (background)      │
     └─────────────────┼─▶ ├─ on-device: live HRV, stress moments        │
                       │   ├─ local cache (SQLite) → batched upload      │
                       │   └─ dashboards · push notifications            │
                       └─────────┬──────────────────────▲────────────────┘
                                 │ upload live samples   │ APNs push (via Expo)
                                 ▼                       │
   Google Health cloud ──webhook──▶ OpenAir backend (Next.js API on Vercel)
          ▲                         ├─ /api/auth/google   OAuth (secret server-side)
          └──── REST v4 reads ──────├─ /api/webhooks/health  verify → 204 → fetch interval
                                    ├─ /api/cron/sync     fallback poll · nightly baselines
                                    ├─ /api/ingest/live   receives BLE samples from phone
                                    ├─ derive: baselines · stress · recovery
                                    └─ alerts: rules → Expo push → APNs
                                               │
                                     Postgres (Neon via Vercel Marketplace)
```

| Layer | Choice | Why |
|---|---|---|
| iOS app | **Expo (React Native) + TypeScript**, expo-router, `react-native-ble-plx`, Victory Native / Skia charts, expo-sqlite | Native Bluetooth, cloud builds without a Mac |
| Distribution | EAS Build → **TestFlight** | Installs like a normal app and auto-updates |
| Backend | **Next.js API routes on Vercel** + Cron | OAuth secret, webhook endpoint, scheduled jobs |
| DB | **Postgres (Neon)** + Drizzle ORM | Single-user time series fits easily |
| App login | Sign in with Google, allow-listing only your email; the same consent grants health scopes | One login |
| Push | **Expo Notifications → APNs** | Reliable on iPhone (Web Push is no longer needed) |
| Later | iOS **Live Activity** (live HR on the Lock Screen), home-screen **widgets**, Claude coach | Native extras |

### Data model (first cut)

```
oauth_tokens      (user_id, refresh_token_enc, access_token_enc, expires_at, scopes)
data_points       (id, user_id, data_type, start_at, end_at, record_id UNIQUE, device, payload jsonb, fetched_at)
live_hr           (user_id, ts, bpm, rr_ms int[], contact bool, source='ble')           -- from the phone
live_hrv_windows  (user_id, start_at, end_at, rmssd, sdnn, mean_hr, n_beats, artifact_pct, still bool)
daily_metrics     (user_id, date, steps, azm, rhr, hrv_rmssd, sleep_min, sleep_eff, deep_min, rem_min,
                   spo2_avg, resp_rate, skin_temp_delta, stress_load, recovery, PRIMARY KEY(user_id,date))
baselines         (user_id, metric, window_days, mean, sd, n, computed_at)
stress_events     (user_id, start_at, end_at, peak_hr, min_rmssd, rhr_ref, tag)
sync_runs         (id, trigger[webhook|cron|manual|backfill|ble_upload], data_types, status, errors jsonb, started_at, ended_at)
alert_rules       (id, user_id, kind, params jsonb, enabled, quiet_hours)
alert_events      (id, rule_id, fired_at, message, payload jsonb, acknowledged)
push_tokens       (id, user_id, expo_push_token, device_label)
```

Raw cloud payloads stay untouched in `data_points`. `record_id` + upserts make webhook retries idempotent. Live samples are buffered on the phone in SQLite and uploaded in batches, so a bad connection loses nothing.

### Sync strategy

**Cloud history (Google Health API):**
1. **Backfill** 90 days on first connect, rate-limited (under 5 req/s, back off on 429).
2. **Webhook**: Google says "type X changed in interval Y", we return 204 and fetch only that interval.
3. **Cron fallback** every 30–60 min for today and yesterday, plus a nightly baseline recompute. _(Vercel Hobby limits cron frequency; webhooks cover most cases.)_
4. **"Sync now"** in the app, plus a button that opens Google Health so the band syncs to the phone. **We can't trigger band → phone history sync ourselves.**

**Live (Bluetooth, on the iPhone):**
1. The band has **Share heart rate + Always visible** turned on (Google Health → device → Connections).
2. OpenAir connects to `0x180D` and subscribes to `0x2A37`. **Background mode `bluetooth-central` + state restoration** keep it running while the phone is locked. If you force-quit the app, the stream stops until you reopen it.
3. The phone computes rolling HRV and stress moments **on the device**, so instant alerts don't need the network. It uploads samples every few minutes.
4. If a reading arrives from both sources (e.g. HR), the cloud wins for history and BLE wins for "right now".

### Derived metrics (v1 formulas, tunable)

- **Baselines:** 28-day rolling mean and SD per metric. Show a "learning" state until there are at least 7 nights of data.
- **Live HRV:** RMSSD/SDNN over rolling 60 s–5 min windows. Reject RR outside 300–2000 ms or more than 20% from neighbors. Mark windows with over 10% artifacts as invalid. **Only trust windows where you're still** (uses the iPhone's motion sensors via CoreMotion activity, since the HR broadcast carries no accelerometer data).
- **Stress moment (live):** HR ≥ RHR + 22 bpm **and** RMSSD < baseline − 1 SD, sustained ≥ 10 min, phone says stationary, no workout.
- **Stress Load (0–100, daily):** `50 + 12·(−z_HRV) + 9·(z_RHR) + 6·(sleep_debt_h) + 0.15·(stress_moment_min)`, clamped.
- **Recovery (0–100, morning):** HRV z (40%), inverted RHR z (25%), sleep vs goal (20%), resp. rate + skin temp deviation (15%).
- **Every score is labeled "OpenAir estimate"** and breaks down into its inputs.

### Alerts (initial rule set; pick your top 2–3)

| Rule | Default trigger | Source | When |
|---|---|---|---|
| Stress moment | Live rule above | BLE (on device) | Real time, throttled, quiet hours |
| Morning brief | Recovery + sleep + HRV vs baseline | Cloud | After the night's sleep syncs |
| Low HRV | Nightly HRV < baseline − 1.5 SD | Cloud | On sync |
| High RHR | RHR > baseline + 5 bpm | Cloud | On sync |
| Illness watch | 2+ of RHR ↑, resp ↑, skin temp ↑ > 0.5 °C, HRV ↓, for 2 nights | Cloud | Morning |
| High HR at rest | HR > threshold for 10 min while stationary | BLE | Real time |
| Move nudge | Sedentary > 60 min, daytime | Cloud / CoreMotion | Hourly |
| Band disconnected | No BLE samples for > 30 min while awake | Phone | Real time |
| Sync stale / auth error | No cloud data > 6 h, or OAuth refresh failed | Backend | Immediately |

---

## Roadmap

| Phase | Scope | Exit criteria |
|---|---|---|
| **0a · BLE test** (you, ~1 h) | nRF Connect on iPhone: Share heart rate + Always visible, subscribe to `0x2A37`, check RR presence and variability, check Google Health still syncs, note 24 h battery use | We know whether real RR intervals exist |
| **0b · API spike** (1–2 days) | GCP project, enable Google Health API, consent screen "In production" (unverified), Web OAuth client, probe script for every data type, test webhook | Real HRV, sleep, SpO₂ and RHR for your Air; webhook fires; refresh token outlives 7 days |
| **1 · Foundations** | Backend: Next.js API, Postgres schema, Google sign-in, encrypted tokens, backfill, webhook, cron. App: Expo skeleton, Apple Developer account, EAS → TestFlight | The app is installed on your iPhone and shows 90 days of synced data |
| **2 · Live Bluetooth** | BLE live source, background mode, state restoration, SQLite buffer, upload, Live screen | Live HR + HRV keep updating with the phone locked for 8+ h |
| **3 · Dashboards** | Today, Sleep, Heart & HRV, Stress, Activity, Trends | Usable one-handed and fast to open |
| **4 · Derived metrics** | Baselines, live HRV windows, stress moments, Stress Load, Recovery, illness watch | Scores explain themselves; a backtest on history looks sane |
| **5 · Alerts** | Expo push / APNs, rules engine, quiet hours, morning brief, history | The morning brief and a live stress alert arrive on your phone |
| **6 · Extras** | Lock-screen Live Activity with live HR, widgets, Claude coach chat, export, journaling tags for correlations | Pick as wanted |

---

## Wireframes

### 1 · Today (home)
```
┌──────────────────────────────┐
│ OpenAir        Thu 9 Oct  ⟳ │  ⟳ = sync now · "synced 4m ago"
│ ● Live  72 bpm · HRV 48      │  live strip from the band (BLE)
├──────────────────────────────┤
│ ┌──────────┐  Recovery  78   │
│ │   78     │  ▲ good to push │
│ │ recovery │  HRV  52ms  +8% │
│ └──────────┘  RHR  56   -2   │
├──────────────────────────────┤
│ Stress load  ▁▂▃▅▃▂  34 low  │
│ 2 stress moments · 11:40, 15:05
├──────────────────────────────┤
│ Sleep 7h12  ████▓▓░░ eff 91% │
│ Steps 6,420 / 9,000  ▓▓▓▓▓░░ │
├──────────────────────────────┤
│ Night signals                │
│ SpO2 96%  · Resp 14.2 · Temp +0.1°
├──────────────────────────────┤
│ [Today][Live][Sleep][Heart][Alerts]
└──────────────────────────────┘
```

### 2 · Live (direct from the band)
```
┌──────────────────────────────┐
│ Live        ● connected  🔋64%│
├──────────────────────────────┤
│        72 bpm                │
│  ▁▂▂▃▃▂▂▃▄▅▄▃▂▂▂▃▃▂  10 min  │
├──────────────────────────────┤
│ HRV (RMSSD, 2-min)   48 ms   │
│ baseline 44–58 ════●════     │
│ still ✔ · artifacts 3%       │
├──────────────────────────────┤
│ Stress now   calm            │
│ ░░░░░░░░▓▓░░░░░░  last hour  │
├──────────────────────────────┤
│ RR  833 812 845 801 828 ms   │  raw beat-to-beat feed
│ [ Breathe 1 min ]            │
├──────────────────────────────┤
│ [Today][Live][Sleep][Heart][Alerts]
└──────────────────────────────┘
```

### 3 · Heart & HRV
```
┌──────────────────────────────┐
│ ← Heart & HRV        7d 30d 90d
├──────────────────────────────┤
│ HRV nightly (RMSSD)  52 ms   │
│  ·  ·    ·  · ·   ·  ●       │
│ ════ baseline band 44–58 ════│
├──────────────────────────────┤
│ Resting HR    56 bpm  ↓2     │
│ ‾‾\__/‾‾\___/‾‾‾\__          │
├──────────────────────────────┤
│ Intraday HR (today)          │
│ ▁▁▂▅▇▅▃▂▂▃▆█▆▃▂▁ ◆ ◆         │  ◆ = stress moment
├──────────────────────────────┤
│ Why recovery is 78:          │
│ HRV +0.6σ ▲ · RHR −0.4σ ▲    │
│ Sleep 96% of goal · Temp ok  │
└──────────────────────────────┘
```

### 4 · Stress
```
┌──────────────────────────────┐
│ ← Stress     OpenAir estimate ⓘ
├──────────────────────────────┤
│   34 / 100   LOW             │
│ ░░░░░░▓▓▓▓░░░░░░░░░░░░░░░░   │  timeline 6am→now
├──────────────────────────────┤
│ Contributors                 │
│ HRV vs baseline     ▼ lowers │
│ Resting HR          — neutral│
│ Sleep debt 0h20     ▲ small  │
│ Stress moments 18m  ▲        │
├──────────────────────────────┤
│ Moments                      │
│ 11:40–11:58 · peak 104 · RMSSD 21 [tag]
│ 15:05–15:16 · peak 98  · RMSSD 25 [meeting]
├──────────────────────────────┤
│ 14-day ▂▃▅▂▁▃▄▂▂▃▅▃▂▃        │
└──────────────────────────────┘
```

### 5 · Sleep
```
┌──────────────────────────────┐
│ ← Sleep   last night  ‹ ›    │
├──────────────────────────────┤
│ 7h 12m   23:41 → 07:02  91%  │
│ Awake ▔▔ ▔                   │
│ REM     ▔▔  ▔▔▔   ▔▔         │
│ Light ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔       │
│ Deep    ▔▔▔ ▔▔               │
├──────────────────────────────┤
│ Deep 1h05 · REM 1h38 · Light 4h02
│ HRV 52 · SpO2 96% · Resp 14.2│
├──────────────────────────────┤
│ 14 nights ▅▆▇▅▆▇▇▆▅▆▇▇▆▇ goal┄│
└──────────────────────────────┘
```

### 6 · Alerts
```
┌──────────────────────────────┐
│ Alerts               + rule  │
├──────────────────────────────┤
│ ● Stress moments  quiet 22–7 [on]
│ ● Morning brief   07:30   [on]
│ ● Low HRV  < base−1.5σ    [on]
│ ● Illness watch           [on]
│ ○ Move nudge 60m          [off]
│ ● Band disconnected 30m   [on]
├──────────────────────────────┤
│ History                      │
│ Today 07:31  Recovery 78 ✔   │
│ Wed 15:06    Stress moment   │
│ Tue 07:30    HRV low (−1.8σ) │
├──────────────────────────────┤
│ ┌ push preview ────────────┐ │
│ │ OpenAir · Morning        │ │
│ │ Recovery 78. HRV +8%,    │ │
│ │ slept 7h12. Go hard today│ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

### 7 · Sync & Settings
```
┌──────────────────────────────┐
│ Sync                         │
├──────────────────────────────┤
│ Fitbit Air · battery 64%     │
│ Bluetooth live   ● connected │
│ Band → Google    last 07:02  │
│ Google → OpenAir 4 min ago ✔ │
│ [ Sync now ] [Open Google Health]
├──────────────────────────────┤
│ Data types   ✔ HR ✔ HRV ✔ Sleep
│              ✔ SpO2 ✔ Temp ⚠ VO2
│ Webhook  active · 38 today   │
│ Backfill 90/90 days ✔        │
│ Live upload  queued 0        │
├──────────────────────────────┤
│ Google account  connected    │
│ Notifications   enabled      │
│ Goals · Units · Export       │
└──────────────────────────────┘
```

### 8 · Lock screen (Phase 6)
```
┌──────────────────────────────┐
│ 09:41                        │
│ ┌──────────────────────────┐ │
│ │ ♥ 72  HRV 48  calm       │ │  Live Activity
│ │ ▁▂▂▃▃▂▂▃▄▅▄▃  ● Air 64%  │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

---

## Open questions

1. **Top 2–3 alerts for v1?** My suggestion: stress moment, morning brief, illness watch.
2. **Mac or Windows only?** The plan assumes Windows + Expo cloud builds. A Mac opens up SwiftUI as an option.
3. **OK with the $99/yr Apple Developer Program?** It's required for TestFlight installs that don't expire.

## What's missing / not yet verified

- [ ] Whether the Air's Bluetooth heart rate broadcast includes **real RR intervals** (Phase 0a).
- [ ] Whether **HRV values actually come back** from the Google Health API for your account (Phase 0b).
- [ ] Whether the band can **share HR over Bluetooth while Google Health syncs**. iOS can share one connected peripheral between apps, but this is unconfirmed for the Air.
- [ ] **Battery cost** of "Always visible" sharing over a full day.
- [ ] How reliably iOS keeps OpenAir's Bluetooth session alive in the background over many hours.
- [ ] A Google Cloud OAuth client setup guide (gets written in Phase 0b).
- [ ] A LICENSE for this repo. OpenFit is UNLICENSED, so we use its ideas only and copy none of its code.

## Next step

**Phase 0a**, on your iPhone: install **nRF Connect**, turn on Share heart rate + Always visible for the Air, connect, subscribe to Heart Rate Measurement, and send me screenshots of the raw values. Steps are in [BLE_RESEARCH.md §4](BLE_RESEARCH.md#4-spike-verify-on-your-air-1-hour-no-code). In parallel, I can scaffold the backend and Expo app.
