# Gemini Voice Assistant — Product & Technical Spec (v0.2)

Ephemeral voice assistant for macOS: push-to-talk → Gemini → spoken/text reply. Spec-first; implementation not started.

---

## 1. Vision

Always-available, low-friction voice assistant on macOS for **temporary chats** with Gemini — quick questions, drafting, summarizing — without opening a browser or full chat app.

**Core promise:** Activate → speak → get an answer → session disappears unless explicitly kept.

---

## 2. Primary UI (macOS “widget”)

| Approach | Role | Voice | Notes |
|----------|------|-------|-------|
| **Menu bar / status item** | Primary v1 | Excellent | Always on; red-hat icon; hotkey support |
| **Floating panel** | Primary v1 (optional) | Excellent | Large activate button; minimal transcript |
| **WidgetKit** | Later | Poor | Glanceable status only; not full voice session |

**v1 recommendation:** Menu bar app + optional floating red-hat button. Not WidgetKit as the foundation.

---

## 3. Activation UX

- **Modes (pick for v1):** push-to-talk (hold), toggle (click), or both.
- **Red hat button:** generic fedora mascot preferred over Red Hat® corporate logo unless licensed.
- **States:** `idle` · `listening` · `processing` · `speaking` · `error`

---

## 4. Temp chat policy

| Policy | v1 default |
|--------|------------|
| Session-scoped | Messages in memory only |
| Idle timeout | Auto-clear after N minutes (e.g. 15) |
| Explicit save | Optional “pin thread” in later version |
| Persistence | **Off by default** |

---

## 5. Voice & Gemini architecture

**v1 (recommended):** Native STT/TTS + Gemini text API

```
Mic → Apple Speech → text → Gemini (streaming) → AVSpeechSynthesizer / transcript
```

**Later:** Gemini Live API for duplex conversation (replace transport behind same coordinator).

| Item | Choice |
|------|--------|
| API | Google AI Gemini (`generativelanguage.googleapis.com`) |
| Model | e.g. `gemini-2.0-flash` (configurable) |
| Auth | API key in Keychain |
| System prompt | Short “concise work assistant” persona |

---

## 6. Proposed app structure

```
GeminiVoiceAssistant.app
├── UI/           MenuBarController, FloatingPanelView, SettingsView
├── Voice/        SpeechRecognizer, SpeechSynthesizer, VoiceSessionCoordinator
├── Gemini/       GeminiClient (streaming), ChatSession (in-memory)
└── Storage/      KeychainSecrets
```

Optional later: `GeminiVoiceWidgetExtension` (WidgetKit status + deep link).

---

## 7. Permissions

- Microphone
- Speech Recognition
- Keychain (API key)
- Accessibility (optional v1.5: selected text / focused app context)

No audio on disk by default. Clear listening indicator when mic is active.

---

## 8. MVP scope (v0.1)

**In:** Menu bar + floating hat, push-to-talk hotkey, speech → Gemini → reply, ephemeral session, settings (API key, model, prompt, hotkey), copy last response.

**Out:** WidgetKit, Gemini Live, screen capture, TasksSync integration, multi-user.

---

## 9. Open decisions

1. Primary UI: menu bar vs floating panel vs both
2. Activation: push-to-talk, toggle, or both
3. Reply mode: voice, text transcript, or both
4. Context: none, clipboard (with confirm), selected text
5. Repo: new Xcode project vs separate repo from TasksSync
6. Gemini: personal API key vs enterprise (Vertex / Workspace)

---

## 10. Phased roadmap

| Phase | Deliverable |
|-------|-------------|
| 0 | Lock spec (this doc) + wireframes |
| 1 | Spike: mic → text → Gemini → stdout |
| 2 | MVP UI: menu bar + floating hat + settings |
| 3 | Polish: streaming TTS, hotkeys, idle timeout |
| 4 | WidgetKit status widget (optional) |
| 5 | Gemini Live (optional) |

---

## 11. Deployment options

### 11.1 Rejected: RHEL container as foundation

**Proposal considered:** Run the assistant inside a container with **RHEL as the base image**, using the container as the primary runtime on macOS.

**Decision:** **Rejected** for v1 and as the desktop foundation.

#### Why it was considered

- Familiarity with RHEL / container workflows
- Possible alignment with red-hat branding motif
- Containers help standardize server deployments and CI
- Enterprise teams sometimes require Linux-only backends for API keys and policy

#### Why it does not help (and makes things harder)

| Concern | Impact |
|---------|--------|
| **macOS-native UX** | Menu bar, floating panel, hotkeys, WidgetKit, and system permissions require a native Mac process. A Linux container cannot host that layer. |
| **Voice I/O** | Microphone and low-latency audio on Docker/Podman for Mac is unreliable (device passthrough, extra latency). Voice assistants need a direct audio path. |
| **Speech stack** | Apple Speech / AVFoundation are macOS APIs. Reimplementing STT/TTS in RHEL adds dependencies and worse integration vs on-device Apple frameworks. |
| **Two runtimes** | You still need a macOS shell **plus** a container, with IPC or HTTP between them — more complexity, not less. |
| **Wrong tool for desktop** | Containers on Mac run Linux VMs; they are optimized for services, not interactive desktop voice widgets. |

#### What we would have ended up building anyway

```
[Required] macOS native app  →  mic, UI, permissions, hotkeys
       ↕ (network/IPC)
[Optional] RHEL container    →  Gemini client, logging, proxy
```

The container does not replace the Mac app; it only duplicates or relocates logic that fits cleanly in the native app for a personal tool.

#### When RHEL/containers *might* appear later (not as foundation)

| Pattern | When it makes sense |
|---------|---------------------|
| **macOS client + RHEL-hosted API proxy** | Org policy: API keys only on internal infra; audit logs; rate limits; shared prompts |
| **Linux desktop target** | Product is a RHEL/Fedora workstation app or web UI — not a macOS widget |
| **OpenShift / RHEL AI** | Corporate deployment of model gateway; Mac app stays thin client |

These are **Phase 4+** or enterprise add-ons, not substitutes for the macOS native core.

---

### 11.2 Accepted: Native macOS foundation (v1)

| Layer | Technology |
|-------|------------|
| **UI & system integration** | Swift, SwiftUI/AppKit, menu bar extra, optional floating panel |
| **Voice** | `Speech` framework, `AVFoundation` |
| **Gemini** | Direct HTTPS from app (API key in Keychain) |
| **Distribution** | Local dev build → optional signed/notarized `.app` |
| **Container** | None for v1 |

#### Optional hybrid (document only; not v1)

If enterprise requirements appear:

```
macOS GeminiVoiceAssistant.app
    → HTTPS → gemini-proxy.service (RHEL container on internal host)
                  → Gemini / Vertex with org credentials
```

The Mac app retains all voice and UI responsibilities; the container is a **policy gateway**, not the product foundation.

---

### 11.3 Deployment comparison

| Criterion | RHEL container foundation | Native macOS v1 |
|-----------|---------------------------|-----------------|
| Time to MVP | Slower (two stacks) | Faster |
| Voice latency | Higher risk | Lower |
| Menu bar / widget UX | Not in container | Native |
| Personal API key tool | Overkill | Appropriate |
| Enterprise audit trail | Possible (backend) | Add proxy later |
| Ops burden | Image, compose, updates | Single `.app` |

---

## 12. Revision history

| Version | Date | Changes |
|---------|------|---------|
| v0.1 | 2026-06-19 | Initial vision, architecture, MVP scope |
| v0.2 | 2026-06-19 | Added §11 Deployment options; rejected RHEL container as foundation |
