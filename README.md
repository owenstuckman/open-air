# OpenAir

A phone-first dashboard for the Google Fitbit Air: HRV, stress, recovery, sleep, and alerts, on iPhone.

- **Live** heart rate + beat-to-beat (RR) intervals read straight from the band over Bluetooth → real-time HRV and stress
- **History** (sleep, nightly HRV, SpO₂, skin temp, steps) pulled from the Google Health API
- Native iOS app (Swift / SwiftUI + CoreBluetooth) + small backend (Next.js API on Vercel, Postgres)

Status: **shelved (2026-10-17), not feasible for now.**

## Why it's shelved

The goal changed to talking to the band directly over Bluetooth, built from scratch, with no Google Health API and no Google cloud. The Air doesn't allow that today:

- **Only one open channel.** Apps can only use the standard Bluetooth heart rate service (`0x180D`): live heart rate, and beat-to-beat (RR) intervals if the band includes them (unconfirmed). There's no raw sensor access: no optical or accelerometer stream, and no SpO₂ or temperature data.
- **Everything else is locked.** Steps, sleep, HRV history, SpO₂ and skin temperature only leave the band through Fitbit's proprietary sync. That sync has two layers of encryption: a transport key for the Bluetooth link, plus an inner end-to-end key that exists only on the band and Google's servers. The inner layer hadn't been broken as of 2026-09-29.
- **Custom firmware isn't realistic.** Firmware is very likely signed by Google, so it can't be replaced to expose the sensors.
- **Google is still needed.** First-time pairing has to go through Google's app and servers.

A Bluetooth-only app would give live heart rate, and HRV if RR is real, and nothing else. That isn't enough for the dashboard planned here.

**Revisit if:**
- the community breaks the Air's sync encryption (watch [Gadgetbridge #504](https://codeberg.org/Freeyourgadget/Gadgetbridge/issues/504)), or
- the hardware changes to something with an open sensor SDK: a Polar Verity Sense or H10 (raw optical, accelerometer and ECG data, plus chest-strap RR on the H10), or an open-source watch like Bangle.js 2 or PineTime.

The docs below are kept as-is for reference.

| Doc | Contents |
|---|---|
| [docs/PLAN.md](docs/PLAN.md) | Decisions, architecture, data model, metrics, alerts, roadmap, wireframes, open questions |
| [docs/RESEARCH.md](docs/RESEARCH.md) | Google Health API feasibility |
| [docs/BLE_RESEARCH.md](docs/BLE_RESEARCH.md) | Direct-Bluetooth feasibility and the Phase 0a test |
