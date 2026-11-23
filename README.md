# bt_rider

**An open, phone-centric motorcycle communications hub built on ordinary Bluetooth helmet headsets and a custom Android app.**

bt_rider makes the rider's Android phone the "brain" of the helmet comms setup. Any standard Bluetooth headset (Sena, Cardo, or a generic helmet kit) provides speakers, microphone, and buttons. The app handles voice commands, rider-to-rider group audio, calls, navigation prompts, messages, and music, and decides what you hear and when.

> Status: **concept / early research.** There is no app code yet. This README defines the problem, the goals, and the initial architecture direction.

---

## Why this project exists

### Voice control broke when Assistant became Gemini

For years a basic Bluetooth helmet setup plus **Google Assistant** worked well for hands-free use. You pressed the headset's voice button or said "Hey Google", then said things like "call Mike", "navigate home", "read my messages", or "play music". It was reliable and fast, and it worked with gloves on at speed.

Google is replacing Assistant with **Gemini**, and that workflow has degraded:

- Headset voice buttons and older Bluetooth accessories may not route to Gemini properly, or at all.
- Core actions such as placing calls have broken for some users during the transition.
- Gemini is tuned for open-ended conversation. A rider needs **short, deterministic, low-latency commands**, not a chatbot that wants to discuss the request.
- Behavior changes with server-side updates the rider can't control.

A rider needs a voice layer that is **predictable, works offline or on poor signal, and is built for the helmet**.

### Group comms are locked to vendor ecosystems

Mesh intercoms (Cardo DMC, Sena Mesh) work well within one brand and poorly across brands. A group riding with mixed hardware usually ends up on a phone app, a WhatsApp call, or nothing. Existing phone-based intercom apps are mostly closed source, internet-only, or abandoned.

---

## Core concept

```
        ┌─────────────── Rider's phone (bt_rider app) ───────────────┐
        │                                                            │
 Helmet │  Audio router / mixer  ◄── priority rules (ducking, etc.)  │
headset │      ▲   ▲   ▲   ▲                                         │
(HFP /  │      │   │   │   └── Music / media                       │
 A2DP / │      │   │   └────── Nav prompts (Maps / OsmAnd / etc.)   │
 LE     │      │   └────────── Phone calls & messages (TTS)         │
 Audio) │      └────────────── Group voice channel ◄──────────────┐ │
   ◄────┤                                                         │ │
   ────►│  Voice command engine (wake word / button → intent)     │ │
        │                                                         │ │
        └─────────────────────────────────────────────────────────┼─┘
                                                                  │
                    Cellular (WebRTC / Opus)  ◄───────────────────┤
                    Local fallback (Wi-Fi Direct / hotspot) ◄─────┘
                                                                  │
                                              Other riders' phones running bt_rider
```

- **The phone is the hub.** Each rider's headset pairs to their own phone using standard profiles. Nothing depends on proprietary intercom firmware.
- **The app owns the audio session.** It mixes and prioritizes streams (group voice, calls, navigation, music) and ducks or pauses them according to rider-defined rules.
- **The app is the voice assistant.** It registers as the device's assistant and/or listens for headset button events, so the headset's voice button opens bt_rider instead of Gemini. Commands are matched against a fixed, rider-oriented grammar.
- **Riders connect phone-to-phone.** Group audio travels over cellular data by default, with a local, no-internet fallback for riders who are close together.

---

## Goals

### 1. Reliable hands-free voice commands
- Trigger by **headset button** (primary) and optionally an **offline wake word**.
- **On-device speech recognition** with a constrained command grammar for speed, accuracy in wind noise, and offline operation.
- A minimum command set:
  - Calls: "call `<contact>`", "answer", "hang up", "redial"
  - Messages: "read messages", "reply `<text>`", "send `<contact>` I'm running late"
  - Navigation: "navigate to `<place>`", "next fuel", "how far", "cancel route"
  - Group: "talk to group", "mute group", "who's online", "call `<rider>`"
  - Media: "play", "pause", "skip", "volume up/down"
  - Status: "battery", "time", "weather ahead"
- Spoken confirmations kept short. Every action must be completable without looking at the phone.
- An optional cloud LLM fallback for open-ended questions, never on the critical path for core commands.

### 2. Multi-rider group communication
- Group voice for **2 to ~10 riders** with low latency (target under 300 ms mouth-to-ear on decent LTE).
- Works with **any brand of Bluetooth headset**. Mixed Sena, Cardo, and generic hardware in one group.
- Simple group formation: QR code or short join code at the meetup, persistent named groups.
- Voice-activated transmit with noise gating tuned for wind and engine noise, plus push-to-talk as an option.
- Graceful degradation: automatic reconnect, and a "you've lost the group" spoken alert.
- **Local fallback mode** (Wi-Fi Direct / hotspot) for riding without cell coverage, within its range limits.

### 3. Smart audio arbitration
- Configurable priority, for example: *emergency / call > group voice > nav prompt > message readout > music*.
- Ducking instead of hard cuts where possible.
- Per-source volume and mute.

### 4. Rider safety and usability
- Glove-friendly, high-contrast UI for setup and when stopped. Mostly zero-UI while riding.
- Battery-aware design for all-day rides.
- Optional future features: crash/fall detection with group alert, and a "rider down / stopping" group broadcast.

### 5. Open and hackable
- Open source, documented architecture, pluggable components (speech-to-text engine, transport, command set).

### Non-goals (for now)
- Custom headset hardware or firmware. We use what riders already own.
- iOS. Android first, because it allows the assistant, audio, and background control this project needs.
- Replacing true device-to-device Bluetooth mesh. Phone-to-phone over IP is the practical path with off-the-shelf headsets.

---

## Key technical challenges (to be validated)

| Area | Challenge | Initial direction |
|---|---|---|
| Headset mic | Two-way voice over Bluetooth Classic requires **HFP/SCO** (narrowband or wideband mSBC), and Android generally allows one active SCO link. | Run SCO via `AudioManager` communication device APIs while in group or command mode. Evaluate **LE Audio** headsets as they appear. |
| Calls vs. group | A phone call and group voice both want the SCO link. | Define call handling: pause group, mix group into the call, or let the rider choose. |
| Assistant trigger | The headset voice button (HFP `AT+BVRA` / voice-dial) normally opens the system assistant. | Hold the **default digital assistant role** (`VoiceInteractionService`) and/or handle media button events through a `MediaSession`. |
| Background survival | Android aggressively kills background audio and network apps. | Foreground service with `microphone` / `connectedDevice` types, and per-OEM battery-optimization guidance. |
| Noise | Wind and engine noise at speed. | VAD + noise suppression (WebRTC APM / RNNoise), a constrained grammar, and confirmation prompts for destructive actions. |
| Transport | Cellular dead zones, NAT, latency. | WebRTC with Opus for multi-party audio (SFU for larger groups), with a local multicast fallback. |
| Reading messages | Reading notifications from other apps. | `NotificationListenerService` + TTS, with reply via notification actions. |
| Multi-headset broadcast | LE Audio / **Auracast** could let one phone feed several headsets, but Android's broadcast-assistant APIs appear to be largely system-privileged. | Watch and prototype. Not on the critical path. |

---

## Prior art and inspiration

None of these solve the whole problem, but each covers part of it.

### Group voice for riders
- **[PackChat](https://cr0wn-gh0ul.github.io/Cr0wn-Gh0ul/posts/2026-03-26)** (open source). A motorcycle-first group voice app built for mixed Cardo/Sena groups. It uses WebRTC through an SFU over cellular data, and has no offline fallback. It is the closest in spirit to this project's group-comms goal.
- **[Nojre](https://github.com/hurui200320/Nojre)** (open source, Android). Broadcasts phone mic audio over **UDP multicast on a shared Wi-Fi network** with no internet. A useful reference for the local fallback mode.
- **[HelmChat](https://www.rideapart.com/news/444532/helmchat-free-helmet-communications-app/about:blank)** (open source). A free helmet comms app built on **Jitsi**.
- **[Mumla](https://www.github.com/quite/mumla)** / Humla (GPLv3, Android). A mature Mumble client with low-latency Opus voice, push-to-talk, and VAD. A good reference for voice-chat UX and a possible self-hosted transport.
- **Sena +Mesh adapter** (commercial). A small battery-powered unit that pairs to an older Sena Bluetooth-only headset and bridges it into Sena Mesh, with mesh range up to about 800 m. It's a single-purpose version of the bt_rider hub: it bridges one protocol to another, but only Sena to Sena, with no mixer, no radio integration, and no app control.
- Closed-source comparables: Sena WAVE, Cardo Connect, Ruroc Chain, Intermoto, and others. They show the demand and the known pain points (Android audio glitches, reconnect noise).

### Voice assistants and speech
- **[Dicio](https://github.com/Stypox/dicio-android)** (GPLv3, Kotlin). An open-source, privacy-focused Android assistant with on-device intent matching and **Vosk** speech recognition. The best reference for the voice command layer, including how to register as the system assistant.
- **[Vosk](https://alphacephei.com/vosk/)**. An offline speech recognizer with small Android models and grammar-constrained recognition, which suits a fixed command set.
- **[openWakeWord](https://github.com/dscripka/openWakeWord)**. Open-source wake-word models (TFLite).
- **[Porcupine](https://github.com/Picovoice/porcupine)** (Picovoice). A polished on-device wake word engine that needs a commercial AccessKey.
- **[OpenClaw Assistant](https://dev.to/y_h_3450b0df12444f6ab7cde/introducing-openclaw-assistant-open-source-android-voice-assistant-with-wake-word-detection-2olp)**. An open-source Android assistant using Vosk for offline wake-word detection.

### Bluetooth / LE Audio
- [Android BLE Audio overview](https://developer.android.com/guide/topics/connectivity/ble-audio/overview). Platform support for LE Audio starting in Android 13.
- [Google Bumble `auracast` tool](https://google.github.io/bumble/apps_and_tools/auracast.html). A Python reference for Auracast broadcast, useful for understanding the protocol.

### Context: the Assistant → Gemini transition
- [TechRadar: Android Auto users report broken Assistant functionality during the Gemini rollout](https://www.techradar.com/ai-platforms-assistants/gemini/android-auto-users-complain-about-broken-google-assistant-functionality-as-the-slow-gemini-rollout-continues)
- [9to5Google: Gemini suddenly can't make calls on Android and Android Auto for some users](https://9to5google.com/2026/06/16/gemini-suddenly-cant-make-calls-on-android-and-android-auto-for-some/)
- [Google AI forum: how to trigger Gemini via Bluetooth (Classic & BLE)?](https://discuss.ai.google.dev/t/documentation-request-how-to-trigger-wake-gemini-ai-via-bluetooth-classic-ble/121049)

> **Licensing note:** Several references (Dicio, Mumla) are GPLv3. Code taken from them requires a GPL-compatible license for this project. Choose the license before borrowing code.

---

## Proposed tech stack (initial)

- **Language / UI:** Kotlin, Jetpack Compose
- **Min SDK:** Android 12+ (API 31), with newer audio-routing APIs where available
- **Speech:** Vosk (offline STT with grammar), Android `TextToSpeech`, a pluggable wake-word engine
- **Group audio transport:** WebRTC (Opus) and a lightweight signaling server; local multicast fallback
- **Server (minimal):** signaling/rooms service, plus an optional self-hosted SFU (e.g. LiveKit, mediasoup, or Janus) for larger groups

---

## Roadmap

1. **Spike: audio plumbing.** Open SCO to a real helmet headset, capture the mic, play TTS, and catch the headset's voice/media button.
2. **Spike: voice commands.** Vosk with a grammar for ~10 commands, tested in real wind noise (recorded on a ride).
3. **MVP voice assistant.** Calls, messages readout and reply, navigation launch, and media control from the headset button.
4. **Spike: 2-rider group voice** over WebRTC on cellular.
5. **Audio arbitration** across calls, group voice, nav, and music.
6. **Groups of N**, join codes, reconnect logic, and the local fallback mode.
7. Field testing with mixed Sena/Cardo/generic headsets, then iterate.

---

## Repository layout (planned)

```
bt_rider/
├── android/     # Android app (Kotlin)
├── server/      # Signaling / group rooms service
├── docs/        # Architecture notes, research, device compatibility matrix
└── README.md
```

## Contributing

The project is in early design. Useful contributions now include headset compatibility reports (model, firmware, phone, what worked), field audio recordings for noise testing, and feedback on the command set.

## License

To be decided (see the licensing note above).
