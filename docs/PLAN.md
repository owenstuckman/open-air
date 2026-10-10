# OpenAir — Plan & Wireframes

_Phone-first Fitbit Air dashboard. Draft v1, 2026-10-09. See [RESEARCH.md](RESEARCH.md) for the evidence behind each decision._

## Goal

Keep syncing the Fitbit Air through the Google Health app as usual. OpenAir pulls **all** your data into its own store, shows OpenFit-style dashboards plus the things Google Health doesn't show (**HRV trends, a Stress estimate, Recovery**), and **alerts you** when something changes. It lives as an app icon on your phone.

> **Update (2026-10-09): direct Bluetooth.** See [BLE_RESEARCH.md](BLE_RESEARCH.md) / [ble.html](ble.html). The band's history (sleep, nightly HRV, SpO₂) **can't** be read locally yet because the sync format is encrypted. **Live HR, and probably RR intervals, can** be read straight from the band via its standard Heart Rate broadcast. The direction is now **hybrid**: a BLE live source for real-time HR, HRV and stress, plus the Google Health API for history. The app becomes a Capacitor native app (Android first), since iOS PWAs have no Bluetooth. Next is **Phase 0b**: a 1-hour nRF Connect test on your band.

## Architecture (recommended: PWA + small backend)

```
 Fitbit Air ─BLE─▶ Google Health app ─▶ Google Health cloud
                                          │  ▲
                         webhook (new data)│  │ REST v4 reads (OAuth, read-only scopes)
                                          ▼  │
                ┌─────────────── OpenAir backend (Next.js on Vercel) ───────────────┐
                │ /api/auth/google     OAuth code exchange (secret stays here)      │
                │ /api/webhooks/health verify → 204 → enqueue fetch for intervals   │
                │ /api/cron/sync       fallback poll + daily baseline recompute     │
                │ sync engine          fetch → raw store → normalize → derive       │
                │ alert engine         rules → Web Push                             │
                └──────────────┬───────────────────────────────┬────────────────────┘
                               │                               │
                        Postgres (Neon via                Web Push (VAPID)
                        Vercel Marketplace)                    │
                               │                               ▼
                               └──────────▶  OpenAir PWA on your phone (home-screen icon)
```

| Layer | Choice | Why |
|---|---|---|
| App | **Next.js (App Router) PWA**, Tailwind + shadcn/ui, Recharts | One codebase for iPhone and Android, installable, offline cache |
| Hosting | **Vercel** | Cron, serverless functions, HTTPS webhook endpoint for free |
| DB | **Postgres** (Neon, via Vercel Marketplace) + Drizzle ORM | Time-series fits fine in Postgres at single-user scale |
| Auth to the app | Sign in with Google, allow-list your email only | You already have the Google account. One login does double duty |
| Health data | Google Health API v4, read-only scopes | The Fitbit Web API dies 2026-10-30 |
| Alerts | Web Push (VAPID) | Works on Android and on iOS 16.4+ once installed to the home screen |
| Later | Capacitor wrapper | Native notifications and widgets without a rewrite |

### Data model (first cut)

```
oauth_tokens      (user_id, refresh_token_enc, access_token_enc, expires_at, scopes)
data_points       (id, user_id, data_type, start_at, end_at, record_id UNIQUE, device, payload jsonb, fetched_at)
daily_metrics     (user_id, date, steps, azm, rhr, hrv_rmssd, sleep_min, sleep_eff, deep_min, rem_min,
                   spo2_avg, resp_rate, skin_temp_delta, stress_load, recovery, PRIMARY KEY(user_id,date))
baselines         (user_id, metric, window_days, mean, sd, n, computed_at)
stress_events     (user_id, start_at, end_at, peak_hr, avg_hr, rhr_ref)
sync_runs         (id, trigger[webhook|cron|manual|backfill], data_types, status, errors jsonb, started_at, ended_at)
alert_rules       (id, user_id, kind, params jsonb, enabled, quiet_hours)
alert_events      (id, rule_id, fired_at, message, payload jsonb, acknowledged)
push_subscriptions(id, user_id, endpoint, keys jsonb, device_label)
```

Raw payloads go into `data_points` untouched, so we can re-derive anything later without re-fetching. `record_id` plus upserts make webhook retries idempotent.

### Sync strategy

1. **Backfill** on first connect: the last 90 days, day by day, rate-limited (under 5 req/s, back off on 429).
2. **Webhook** (main trigger): Google says "data changed for type X in interval Y". We return 204 immediately, then fetch only that interval.
3. **Cron fallback**: every 30–60 min, re-pull today and yesterday in case a webhook was missed. Recompute baselines nightly. _(Vercel Hobby limits cron frequency. If it's too coarse, webhooks still cover us, or we use Pro or an external scheduler.)_
4. **"Sync now"** button: pulls from the cloud immediately and offers an "Open Google Health" link that makes the band sync to the phone. **We can't trigger the band → phone sync ourselves.**

### Derived metrics (v1 formulas, tunable)

- **Baselines:** 28-day rolling mean and SD per metric. Show a "learning" state until there are at least 7 nights.
- **Stress Load (0–100):** `50 + 12·(−z_HRV) + 9·(z_RHR) + 6·(sleep_debt_h) + 0.15·(elevated_sedentary_min)`, clamped 0–100.
- **Stress moment:** HR ≥ RHR + 22 bpm for ≥ 10 min with steps < 30 in that window and no exercise logged.
- **Recovery (0–100):** weighted HRV z (40%), RHR z inverted (25%), sleep vs goal (20%), resp. rate and skin-temp deviation (15%).
- **Every score is labeled "OpenAir estimate"** and breaks down into its inputs.

### Alerts (initial rule set)

| Rule | Default trigger | When |
|---|---|---|
| Morning brief | Recovery + sleep + HRV vs baseline | After the night's sleep record syncs |
| Low HRV | Daily HRV < baseline − 1.5 SD | On sync |
| High RHR | RHR > baseline + 5 bpm | On sync |
| Illness watch | 2 of: RHR ↑, resp ↑, skin temp ↑ > 0.5 °C, HRV ↓, for 2 nights | Morning |
| Stress moment | Intraday rule above | Near-real-time (throttled, quiet hours) |
| Move nudge | Sedentary > 60 min, daytime | Hourly |
| Sync stale | No new data in > 6 h while awake | Hourly |
| Token / sync error | OAuth refresh failed, repeated 4xx/5xx | Immediately |

---

## Roadmap

| Phase | Scope | Exit criteria |
|---|---|---|
| **0 · Spike** (1–2 days) | GCP project, enable Google Health API, consent screen "In production" (unverified), Web OAuth client; a script that pulls every data type for 7 days; register a webhook to a test endpoint | Real HRV, sleep, SpO2, and resting HR come back for your Air. Webhook fires after a phone sync. Refresh token survives past 7 days |
| **1 · Sync foundation** | Next.js app, Google sign-in, Postgres schema, OAuth token storage (encrypted), backfill, webhook receiver, cron, Sync screen | 90 days of data in the DB; new data appears within minutes of a phone sync |
| **2 · Dashboards** | Today, Sleep, Heart & HRV, Activity, Trends; PWA manifest and install | Installed on the phone, usable one-handed, works offline from cache |
| **3 · Derived metrics** | Baselines, Stress Load, stress moments, Recovery, illness watch | Scores show their inputs; a backtest on the 90-day history looks sane |
| **4 · Alerts** | Web Push, rules engine, quiet hours, morning brief, alert history | You get a morning brief and a test alert on your phone |
| **5 · Extras** | AI coach chat (Claude), CSV/JSON export, Capacitor app with widgets, journaling tags (caffeine, alcohol, workouts) for correlations | Pick as wanted |

---

## Wireframes

_Visual versions are in [plan.html](plan.html)._

### 1 · Today (home)
```
┌──────────────────────────────┐
│ OpenAir        Thu 9 Oct  ⟳ │  ⟳ = sync now · "synced 4m ago"
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
│ [Today][Sleep][Heart][Trends][Alerts]
└──────────────────────────────┘
```

### 2 · Heart & HRV
```
┌──────────────────────────────┐
│ ← Heart & HRV        7d 30d 90d
├──────────────────────────────┤
│ HRV (RMSSD)   52 ms          │
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

### 3 · Stress
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
│ Elevated-calm HR 18m▲        │
├──────────────────────────────┤
│ Moments                      │
│ 11:40–11:58 · peak 104 bpm  [tag]
│ 15:05–15:16 · peak 98 bpm   [tag]
├──────────────────────────────┤
│ 14-day ▂▃▅▂▁▃▄▂▂▃▅▃▂▃        │
└──────────────────────────────┘
```

### 4 · Sleep
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

### 5 · Alerts
```
┌──────────────────────────────┐
│ Alerts               + rule  │
├──────────────────────────────┤
│ ● Morning brief   07:30   [on]
│ ● Low HRV  < base−1.5σ    [on]
│ ● Illness watch           [on]
│ ● Stress moments  quiet 22–7 [on]
│ ○ Move nudge 60m          [off]
│ ● Sync stale > 6h         [on]
├──────────────────────────────┤
│ History                      │
│ Today 07:31  Recovery 78 ✔   │
│ Wed 15:06    Stress moment   │
│ Tue 07:30    HRV low (−1.8σ) │
└──────────────────────────────┘
```

### 6 · Sync & Settings
```
┌──────────────────────────────┐
│ Sync                         │
├──────────────────────────────┤
│ Fitbit Air · battery 64%     │
│ Band → phone   last 07:02    │
│ Cloud → OpenAir  4 min ago ✔ │
│ [ Sync now ] [Open Google Health]
├──────────────────────────────┤
│ Data types   ✔ HR ✔ HRV ✔ Sleep
│              ✔ SpO2 ✔ Temp ⚠ VO2
│ Webhook  active · 38 today   │
│ Backfill 90/90 days ✔        │
├──────────────────────────────┤
│ Google account  connected    │
│ Notifications   enabled      │
│ Goals · Units · Export       │
└──────────────────────────────┘
```

---

## Open questions for you

1. **iPhone or Android?** The PWA works on both. It changes push setup (iOS needs "Add to Home Screen") and whether a native Health Connect path is worth doing later.
2. **Cloud storage OK?** The plan keeps your data in your own Postgres on your own Vercel account. The alternative is browser-only storage, which rules out background alerts.
3. **Name:** "OpenAir" (matches the repo) unless you prefer something else.
4. **Alert priorities:** which 2–3 alerts matter most for v1?

## Next step

After you confirm the open questions: **Phase 0 spike**. I'll scaffold the Next.js app and a `scripts/probe.ts` that pulls every data type once you've created the Google Cloud OAuth client. I'll write a step-by-step setup guide for that part, since only you can create it in your account.
