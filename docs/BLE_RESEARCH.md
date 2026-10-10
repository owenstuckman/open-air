# OpenAir: Direct Bluetooth (Bypass the Google Cloud) Research

_Research date: 2026-10-09. Follow-up to [RESEARCH.md](RESEARCH.md)._

## TL;DR

| Goal | Possible today? | How |
|---|---|---|
| **Live heart rate straight from the band** | ✅ **Yes** (verify on your unit) | Standard Bluetooth Heart Rate Profile (`0x180D`), enabled with the Air's **"Share heart rate"** setting |
| **Live HRV / stress straight from the band** | 🟡 **Probably**, if the band includes **RR intervals** in its broadcast | Compute RMSSD ourselves from beat-to-beat intervals |
| **History: sleep stages, nightly HRV, SpO₂, skin temp, steps** without the cloud | ❌ **Not currently** | The Air's sync protocol has an inner encryption layer that nobody has broken (status as of 2026-09-29) |
| Replace the Google Health app completely | ❌ No | Pairing (onboarding) itself needs Google's servers |

**Recommendation: go hybrid.** Read **live HR + RR directly over BLE** for real-time heart rate, HRV, stress moments and instant alerts, with no cloud involved. Keep the **Google Health API** only for what the band won't give us any other way: sleep, nightly HRV, SpO₂, skin temperature and steps. If the community breaks the sync encryption later, we add a fully local adapter and can drop the cloud entirely.

---

## 1. The two Bluetooth "doors" on the Fitbit Air

```
Fitbit Air
├── Door A: Standard GATT services (open)
│   ├── 0x180D Heart Rate Service ← only advertised while "Share heart rate" is ON
│   │     └── 0x2A37 Heart Rate Measurement (notify): bpm, sensor contact, RR intervals*
│   ├── 0x180F Battery Service
│   └── 0x180A Device Information
│
└── Door B: Proprietary Fitbit "GoldenGate" stack (locked)
      Gattlink transport → DTLS (per-device "MD-..." pre-shared key)
        → CoAP → sync "dumps" ← inner encryption, key on tracker + Google backend only
          (contains steps, sleep, HRV, SpO₂, temperature history)
```
\* RR intervals appear only if the band sets bit 4 of the measurement flags. **This is unconfirmed for the Air.**

### Door A: Heart Rate Profile (works, with caveats)
- Google lets the Air (along with the Charge 6 and Pixel Watch 2+) act as a **Bluetooth HR sensor** for gym gear and apps. Turn it on in Google Health: device → Connections → **Share heart rate**. **"Always visible"** keeps it broadcasting in the background.
- An MIT-licensed project, [`AyushSagar16/fitbit-air`](https://github.com/AyushSagar16/fitbit-air), already reads the Air's live HR over `0x180D` and reports **RR intervals** in its sample output. It notes the Air **may require bonding (pairing)** before it streams.
- **Conflicting report:** [the5krunner](https://the5krunner.com/2026/06/03/fitbit-air-accuracy-sampling-rate/) says Garmin and Wahoo head units *can't* use it ("the Air does not support" the standard profile). The most likely explanation is that the Air requires an encrypted, bonded link that those head units don't do. Phones and computers can bond. **We confirm this on your unit in the spike.**
- **Costs:**
  - Battery drain. Google warns that sharing reduces battery life.
  - Typically **one connection at a time**.
  - Sharing can stop after a workout and need toggling back on.
  - Wrist optical (PPG) RR intervals are noisier than a chest strap, especially when you move.

Wire format (`0x2A37` notification):
```
byte 0  flags: bit0 = HR is uint16 · bit1-2 = sensor contact · bit3 = energy field · bit4 = RR present
then    HR (uint8 or uint16) · [energy uint16] · RR[] as uint16 in 1/1024 s units
```

### Door B: Proprietary sync (locked, for now)
Status from the Gadgetbridge tracker [#504](https://codeberg.org/Freeyourgadget/Gadgetbridge/issues/504) and the work-in-progress PR [#6273](https://codeberg.org/Freeyourgadget/Gadgetbridge/pulls/6273):

- **Charge 6 (furthest along):** You onboard in the official app first, then pull the `MD-...` DTLS key from the app's data **on a rooted Android phone**. With that key you get battery, live HR and live-activity snapshots (steps, calories, zone minutes). **Sleep, HRV, SpO₂ and full history stay out of reach.** They live in an "opaque sync dump" that Google's backend answers. The band rejects replayed replies with `airlink.nak`. The author paused, blocked on server-side decryption (Aug 2026).
- **Fitbit Air (2026-09-22 → 09-29):** Researchers recovered the MD transport keys and got an **authenticated connection from Linux**. They then hit the **inner encryption** on the sync dumps: "the key lives only on the tracker and Fitbit's backend." It hasn't been broken.
- **Onboarding can't happen offline.** Even then, the band must first be paired through Google's app and servers.
- Some processing, such as sleep staging, likely happens in Google's cloud. **Even a decrypted dump might be raw data** that we'd have to turn into sleep stages and HRV ourselves.

**Risks if we chase Door B anyway:**
- It needs a rooted phone.
- It can break at any firmware update.
- It may violate Google's Terms of Service.
- A second bonded app might interfere with Google Health's own syncing.
- US law (DMCA §1201(f)) has an interoperability exemption that may cover personal use. This isn't legal advice.

**My take:** don't build on Door B. Watch the Gadgetbridge thread and design OpenAir so a "local sync" adapter can slot in if it's ever cracked.

---

## 2. Getting BLE onto your phone

| Platform | Live BLE in foreground | 24/7 background capture | Notes |
|---|---|---|---|
| **Android, Chrome PWA** | ✅ Web Bluetooth | ❌ stops when the screen or tab closes | Fine for a "live" screen |
| **Android, native (Capacitor + BLE plugin)** | ✅ | ✅ foreground service with a persistent notification | Best phone option |
| **iPhone, Safari PWA** | ❌ **no Web Bluetooth on iOS** | ❌ | A PWA can't touch BLE on iPhone |
| **iPhone, native (Capacitor + CoreBluetooth)** | ✅ | 🟡 `bluetooth-central` background mode works while connected; iOS may suspend it | Needs Xcode and an Apple dev account for long-term installs |
| **Bedside hub** (Raspberry Pi / old Android phone) | ✅ | ✅ while you're in range | Good for overnight HRV and stress; uses OpenFit-style local storage |

So "bypass the cloud" pushes the app from **PWA → native app (Capacitor)**. That's the same React UI wrapped in a native shell, which was already Phase 5 in the plan and now moves earlier.

---

## 3. What we can compute ourselves from live HR + RR

| Metric | Method | Quality notes |
|---|---|---|
| Live HR, HR zones | Directly from `0x2A37` | Good |
| **Live HRV (RMSSD, SDNN)** | 60 s–5 min rolling window over RR, artifact rejection (drop RR outside 300–2000 ms or differing >20% from neighbors) | Only meaningful while **still**. PPG RR is noisy during movement |
| **Real-time stress** | HRV below personal baseline + HR above resting, sustained ≥ 5–10 min | We can't separate exercise from stress without motion data. The HR profile carries no accelerometer, so we use the phone's motion sensors or later cloud steps as context |
| **Overnight HRV (our own)** | Bedside hub or phone records RR all night, then deep-sleep-ish windows (lowest stable HR) | Needs "Always visible" overnight, so it costs battery |
| Resting HR | Lowest 30-min rolling average while still | Good |
| Breathing rate | Respiratory sinus arrhythmia from the RR series | Experimental |

Instant alerts become possible, because there's no waiting on cloud sync. For example: "HR 35 bpm above resting for 10 min while sitting, HRV −40% → take a breath."

---

## 4. Spike: verify on your Air (≈1 hour, no code)

1. In Google Health, turn on **Share heart rate** (and **Always visible**) for the Air.
2. Install **nRF Connect** (Android or iOS). Scan, find the Air advertising `0x180D`, and connect. Accept the pairing prompt if it appears.
3. Subscribe to **`0x2A37`** and watch the raw bytes:
   - Is **flags bit 4** set? If yes, RR intervals are present.
   - Do the RR values **vary beat to beat** while you sit still and breathe slowly? If yes, they're real RR. If they always equal 60000/bpm, they're synthetic and useless for HRV.
4. While connected, open Google Health and check that it still syncs normally.
5. Leave "Always visible" on for 24 h and note the **battery drain**.

Send me what you see (screenshots are fine). If RR is real, the hybrid plan is a go. If RR is missing, direct BLE gives live HR only, and HRV stays cloud-sourced, from nightly values.

---

## 5. Impact on the plan

- **Architecture becomes hybrid:** `BLE live source` (phone/hub, local) + `Google Health API` (history), merged into one store.
- **Form factor:** Capacitor native app (Android first, since it's simpler for background BLE). A PWA remains for viewing on any device.
- **New phase "0b · BLE spike"** runs alongside the API spike.
- **The stress feature gets better:** real-time, from real beat-to-beat data, rather than inferred once a day.

## Sources
- [AyushSagar16/fitbit-air: live HR from Fitbit Air over BLE (MIT)](https://github.com/AyushSagar16/fitbit-air) and its [research notes](https://github.com/AyushSagar16/fitbit-air/blob/main/docs/RESEARCH.md)
- [Gadgetbridge #504: Support for Fitbit devices (2026 comments on Charge 6 and Air)](https://codeberg.org/Freeyourgadget/Gadgetbridge/issues/504)
- [Gadgetbridge PR #6273: WIP Fitbit integration](https://codeberg.org/Freeyourgadget/Gadgetbridge/pulls/6273)
- [ZonePoints: Fitbit Air live heart rate on Android](https://zonepoints.app/articles/fitbit-air-live-heart-rate/)
- [FITIV: live HR on Fitbit Air](https://www.fitiv.com/blog/live-hr-and-workout-tracking-on-ios-how-to-maximize-your-fitbit-air-experience)
- [HeartCast: Fitbit Air support](https://www.heartcast.app/heartcast-adds-support-for-google-fitbit-air-and-all-external-bluetooth-heart-rate-monitors/)
- [the5krunner: Fitbit Air accuracy and sampling rate](https://the5krunner.com/2026/06/03/fitbit-air-accuracy-sampling-rate/)
- Earlier Fitbit protocol research: [Classen & Wegemer, REcon 2018](https://recon.cx/2018/montreal/schedule/system/event_attachments/attachments/000/000/045/original/RECON-MTL-2018-Fitbit_Firmware_Hacking.pdf), [Dissecting the Fitbit Cloud, App, and Firmware (IMWUT 2018)](https://homepages.inf.ed.ac.uk/ppatras/pub/imwut18.pdf), [Breaking Fitness Records without Moving (RAID 2017)](https://homepages.inf.ed.ac.uk/ppatras/pub/raid17.pdf), [Schellevis thesis 2016](https://www.cs.ru.nl/bachelors-theses/2016/Maarten_Schellevis___4142616___Getting_access_to_your_own_Fitbit_data.pdf)
