# Persephone V1 — Technical Spec

**Project recognition: Top 5 finalist · Robotics & Physical AI, Agentic AI Build Week 2026**

Built with [Nathan Hor](https://github.com/NathanHor22). [Project and award record](https://www.fovea.space/#recognition).

This public repository contains **V1 networking-badge design notes**. It does not contain the complete hackathon firmware or a finished deployment. The related [Whisp voice Q&A backend](https://github.com/jinchuntan/whisp-voice-qa) contains the public FastAPI server and host dashboard.

[Original design repository](https://github.com/NathanHor22/persephone)

**A wearable networking system for events.** Attendees register with a webcam, wear a BLE badge, press a button to log the people they meet, and review every encounter on a personal dashboard afterward with one-click connect to LinkedIn and GitHub.

**V1 scope: no camera on the badge, no face matching.** Identity comes from BLE beacons exchanged between badges. Face capture at registration and candid face matching are deferred to V2. This lets V1 ship for one real event without the longest-tuning, highest-risk component.

---

## 1. System overview

Three components, each independently useful:

1. **Registration system** — web app. Collects attendee profile and consent, assigns each attendee a badge ID, produces the attendee gallery. Runs at the desk on event day and as a pre-registration link beforehand.
2. **The badge** — ESP32-S3 wearable. Broadcasts its own attendee ID over BLE, listens for nearby badges, and on button press logs the strongest nearby badge as an encounter. Optional held-press attaches a short voice note. Buffers offline, syncs over WiFi.
3. **The dashboard** — web app. Per-attendee view of every logged encounter: who, when, plus their LinkedIn and GitHub, one-click connect, notes, and export.

```
Registration (webcam + form)
        |
        v
  Attendee gallery  <----------------+
  (id -> profile, consent)           |
        |                            |
        v                            |
   Badge assigned  ---> BLE beacons <-> other badges
        |                            |
   button press = encounter          |
        |                            |
   buffer -> WiFi sync ------------> Backend --> Dashboard
```

---

## 2. Attendee identity model

- Every attendee gets a short numeric **attendee ID** at registration (fits in a BLE payload, e.g. a 4-byte int).
- The badge is bound to one attendee ID for the duration of the event. Binding happens at the desk when the badge is handed over (scan a QR on the badge, or type the badge's short code into the registration app).
- An **encounter** is a directed record: `{owner_id, met_id, timestamp, optional voice_note, optional text_note}`. Owner is the person who pressed the button; met is the strongest nearby badge at that moment.

---

## 3. Registration system

### Data collected
- Name (required)
- LinkedIn URL or handle (required for the dashboard to be useful)
- GitHub handle (optional)
- Email (required, for account recovery and dashboard access)
- Company / role (optional)
- **Consent flags** (see below)

### Consent flags (explicit, logged, timestamped)
- `consent_gallery` — agree to be an attendee in the system at all. Required to register.
- `consent_discoverable` — agree to be logged as an encounter and appear on other attendees' dashboards. If false, the person can use their own badge and dashboard but is invisible to others. Default: opt-in, decided by the event's policy.

Store the consent text version alongside the flag so you can prove what they agreed to.

### Flows
- **Pre-registration:** public link, fills the form, gets a confirmation. Badge assigned on arrival.
- **On-site walk-up:** desk tablet form, or a QR they scan to open the form on their own phone (their phone does the typing, faster line and cleaner data). Badge assigned immediately after.
- **Manual fallback:** desk staff can always search by name and assign a badge manually. Registration is mission-critical on event day; the fast path must never be the only path.

### Badge assignment
- Each badge has a printed short code / QR mapping to its BLE beacon ID.
- Desk staff links `attendee_id <-> badge_beacon_id` in one action.
- System prevents double-assignment of a badge.

---

## 4. The badge (firmware)

### Hardware (V1)
Reuse the existing Persephone stack, minus the camera:
- ESP32-S3 DevKit N16R8 (16MB flash, 8MB PSRAM)
- 2.8" ILI9341 320x240 SPI TFT + XPT2046 touch (status display, kept off by default for battery)
- INMP441 I2S MEMS mic (voice notes)
- PBS-11B push button (the encounter trigger)
- **New for V1:** physical slide power switch on the side (true off, not GPIO sleep), LiPo battery + TP4056-class charger/protection, optional MAX98357A + speaker if talk-back is wanted (not required for V1 core loop)

### BLE behaviour
- **Advertise:** broadcast attendee ID in the BLE advertising payload continuously (low interval to conserve power).
- **Scan:** periodically scan for other Persephone badges, keep a short rolling table of `{beacon_id, RSSI, last_seen}`.
- **Encounter trigger:** on button press, pick the nearby badge with the strongest recent RSSI, record `{met_id, timestamp}` to the local queue, and show a brief confirmation on the TFT (wake screen ~2s, then sleep).
  - Guard against noise: only consider badges seen within the last few seconds and above an RSSI floor. If nothing qualifies, show "no one nearby, try again."
- **Voice note (optional):** hold the button past a threshold to also record ~5s of audio via the mic, saved to flash and attached to that encounter.

### Storage and sync
- Encounters buffer to flash (survive power loss). Venue WiFi is unreliable, so **never depend on live sync.**
- When WiFi is reachable, badge POSTs queued encounters (and any voice-note audio) to the backend, then clears the queue on ack.
- Encounter record on device: `{met_id, timestamp, has_voice_note, voice_note_ref}`. Owner ID is implicit (it's this badge).

### Power target: 3 days
This is a hard constraint and the main firmware risk.
- TFT **off by default**, wake only on button press for ~2s.
- BLE advertise + periodic scan is the baseline draw; tune advertising/scan intervals for battery, not latency.
- Physical off switch so it's genuinely off overnight between event days (3 event days, not 72 continuous hours, if the schedule allows).
- **Do the battery math explicitly** with measured current draw before committing to a battery size. Do not guess.

### States
- Off (physical switch)
- Idle (advertising + scanning, screen off)
- Logging (button pressed, screen wakes, encounter recorded)
- Recording (button held, capturing voice note)
- Syncing (WiFi reachable, uploading queue)

---

## 5. The dashboard (the real product)

The badge is the sensor. The dashboard is where the value lives.

### Per-attendee view
- Login via the email used at registration (magic link is fine for V1).
- List of encounters, most recent first: name, company/role, timestamp, and where relevant a context label ("Day 2, afternoon").
- Each encounter card:
  - **Connect on LinkedIn** — one click to their LinkedIn URL
  - **View GitHub** — one click, if provided
  - **Voice note** — play the attached clip if present
  - **Text note** — add/edit a written note ("follow up re: API deal")
- Search and filter across encounters.
- Export (CSV / vCard) for pushing into a CRM or contacts.

### Respecting consent
- Only show encounters where the *met* person set `consent_discoverable = true`. If someone opted out, their badge still works for them but they never appear on anyone else's dashboard.

---

## 6. Backend

- Attendee + consent store (source of truth for the gallery).
- Badge-to-attendee binding table.
- Encounter ingest endpoint (badge -> backend), idempotent so repeated syncs don't duplicate.
- Voice-note blob storage.
- Dashboard API (auth, encounter list, notes, export).
- Reuse the existing Whisp backend patterns where sensible: FastAPI, Docker Compose, Hetzner VPS. MQTT/STT from Whisp are not required for V1's core loop; add STT later if you want auto-transcribed voice notes.

---

## 7. Explicitly out of scope for V1 (V2 candidates)

- Camera on the badge and candid face capture
- Face matching against a registration gallery (face embeddings, confidence thresholds, confirm-match UI)
- Webcam face capture at registration (the check-in-by-face flow)
- AI talk-back / speaker audio-out
- Live real-time dashboard updates during the event

---

## 8. Build order

1. **Backend + registration + dashboard skeleton** — get the web loop working with fake encounters first.
2. **Manual registration and badge binding** — the unglamorous, mission-critical desk flow. Build the fallback before the fast path.
3. **Badge BLE advertise + scan + button-press encounter** — the core sensor loop, syncing to the backend.
4. **Offline buffering + WiFi sync** — make it survive bad venue WiFi.
5. **Voice notes** — layer on top once the identity loop is solid.
6. **Power tuning to hit 3 days** — measure, then size the battery and set BLE intervals.

Ship 1-4 for one real event. 5-6 harden it. Camera and face matching are a separate V2 track.

---

## 9. Open questions

- Walk-up volume vs pre-registration: shapes how much to invest in the walk-up path.
- Default for `consent_discoverable`: opt-in or opt-out, per event policy.
- Whether V1 needs the TFT at all, or a single status LED is enough for the core loop (LED is far cheaper on power).
- Talk-back speaker: keep in V1 hardware for future use, or drop entirely until V2.
