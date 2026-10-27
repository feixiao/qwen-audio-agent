# Demo Feature Overview and Scenario Guide

This document provides a comprehensive analysis of the demos, client modalities, and vertical business scenarios included in **qwen-audio-agent**, explaining their core capabilities, architectural interactions, and getting-started guidance.

---

## Core Design Philosophy

All demos revolve around the **"Full-Duplex Realtime Voice Frontend + Asynchronous Backend Agent"** architecture:

```
 ┌──────────────────────┐        ┌──────────────────────┐
 │   Clients (Web/App)  │        │ Backend Agent (ACP)  │
 │ Web / Desktop / TUI  │        │ OpenCode/OpenClaw/etc│
 └──────────┬───────────┘        └──────────▲───────────┘
            │                               │ (Async task delegate)
            ▼                               │
 ┌──────────────────────────────────────────┴───────────┐
 │                   Gateway Runtime                    │
 │   • Full-duplex voice stream (DashScope/Omni/etc)    │
 │   • Frontend MCP tools (Instant actions, car control)│
 │   • Long-term memory & context fusion                │
 └──────────────────────────────────────────────────────┘
```

- **Always Present**: Conversations are never stalled by background tasks, web searches, or tool executions.
- **Talk While Working**: The frontend handles natural communication and instant answers, while heavy lifting is delegated to backend agents.
- **Natural Feedback**: When backend tasks complete, results and artifacts are smoothly reported back into the active voice conversation.

---

## 1. General Client Modalities

The repository provides three built-in client modalities for general-purpose usage:

| Client Form | Launch Command | Core Capabilities |
| --- | --- | --- |
| **Web UI** | `npm run dev`<br>or `qwenaudio webui` | • **Full-duplex bidirectional voice**: Low-latency streaming voice, automatic VAD detection, and natural interruption.<br>• **Multimodal input**: Supports text input, photo upload, snapshot capture, and live camera feed.<br>• **Task Cards**: Live visibility into backend Agent stages, tool call logs, and artifact inspection.<br>• **Permission Actions**: Interactive oral or button approval for sensitive operations.<br>• **Knowledge & Memory Library**: Visual inspection of cross-session memories and mounted knowledge documents. |
| **Desktop App** | `npm run desktop` | • **Desktop Floating Orb**: Supports Fluid, Goo, and Sprite Orb skins, compatible with the Awesome Codex Pet gallery.<br>• **State-reactive animations**: Mapped to listening, thinking, speaking, tool calling, and background task execution states.<br>• **Offline Wake Word**: Built-in "Ni Hao Qian Wen" offline wake word detection.<br>• **Idle Sleep**: Automatic window hiding and energy-saving sleep when idle. |
| **Terminal TUI** | `npm run tui`<br>or `qwenaudio tui` | • **Lightweight terminal voice**: Capture microphone audio and stream playback directly in terminal environments without a GUI.<br>• **Realtime event stream**: Live tracking of backend lifecycle events, MCP tool invocations, and state transitions. |

---

## 2. Vertical Scenario Demos (`examples/`)

### 1. Smart Cockpit Demo (`examples/smart-cockpit`)

A reference implementation for automotive in-cabin intelligent voice interaction, including a 3D digital vehicle model, vehicle service simulator, cockpit agent, and gateway.

- **Multi-Intent Concurrent Vehicle Control**:
  - Handles multiple independent vehicle control intents in a single natural sentence (e.g., *"Roll down rear windows, set AC to 22 degrees, turn on headlights, and set music volume to 6"*).
  - Triggers frontend MCP tools in parallel and delivers a unified summary once completed.
- **Amap Multi-Waypoint Navigation**:
  - Natural language planning for multi-stop routes (e.g., *"From here, go to Wuchang metro station first, then Intime City, followed by Wulin Square, Hangzhou East Station, and finally Xiaoshan People's Square"*).
  - Intelligently orders `waypoints` and `destination`, displaying the real route on the digital map.
- **Silent Screen Interaction Sensing**:
  - Manual touch operations on the vehicle screen (such as toggling "Avoid Highways") update the context silently via GCP events without conversational interruption. The assistant leverages updated preferences when queried verbally later.
- **Parallel Chat & Background News Search**:
  - When asked to compile news (*"Find 3 key AI news updates from the past 24 hours with sources"*), the backend agent searches and verifies sources asynchronously while the frontend remains available for continuous conversation and car controls.
- **Voice-Created Conditional Skills**:
  - Users can create custom automated triggers using voice (e.g., *"Create a low-temperature reminder: when AC is set to 19°C or lower, remind me to stay warm"*).
  - The vehicle service listens for temperature changes and triggers voice reminders automatically.
- **Personalized Memory & POI Recommendations**:
  - Memorizes long-term preferences (e.g., *"Loves barbecue but cannot eat spicy food"*); automatically incorporates these preferences when recommending nearby dining POIs.

---

### 2. Customer Service Demo (`examples/customer-service`)

Enterprise-grade customer service demonstrations covering Retail and Airline domains, illustrating clean isolation between conversational frontends and strict business transactions.

- **A2A Architecture for Read vs. Write Isolation**:
  - Identity verification and query operations are processed instantly by the realtime frontend.
  - State-altering operations (refunds, ticket rebooking, cancellations, address changes) are delegated to the backend Agent via A2A with policy enforcement.
- **Two-Phase Action Confirmation**:
  - Changes affecting financials or bookings generate a structured preview; operations are only submitted upon explicit customer confirmation.
- **Workstation Matrix**:
  - **Service Console**: Realtime audio waveform, live transcription, order cards, and tool call audit logs.
  - **Human Handover Desk**: Smooth escalation to human agents with preserved context when encountering policy conflicts or permission limits.
  - **Policy Workbench**: Visual policy extraction, decision matrix editing, diff preview, backup, and change audit.

---

### 3. Multimodal Vision Demo (`examples/x-omni`)

Multimodal realtime interaction reference implementation powered by Qwen3.5-Omni or MiniCPM-o.

- **Multimodal Visual Inputs**:
  - Feeds camera video, screen sharing, or static photos into the model alongside streaming audio.
- **On-Demand Picture Analysis & Continuous Video**:
  - Supports queries regarding the current view (*"What book am I holding right now?"*).
- **Active Visual Observation**:
  - Handles timed observation tasks (e.g., *"Watch the boiling pot and alert me when water boils"*), equipped with automatic timeout, visual change deduplication, and proactive voice interruption.
- **WebRTC / WebSocket Dual Transport**:
  - Supports low-latency WebRTC media streams alongside standard WebSocket connections.

---

### 4. WebRTC Client Demo (`examples/webrtc`)

- **Ultra-low latency connection**:
  - Directly connects browser clients to the Gateway over WebRTC, providing superior resilience against packet loss and built-in acoustic echo cancellation (AEC).
  - Compatible with Qwen Audio and Qwen Omni streaming video.

---

### 5. AI Passport Hardware Card Demo (`examples/ai-passport`)

- **IoT Edge-Cloud Collaboration**:
  - Runs audio recording, playback, and character animations on an ESP32-C3 hardware card ("Qwen Voice Bean").
- **Local Device Relay**:
  - A PC relay daemon converts lightweight hardware packets to Gateway Client Protocol (GCP), granting resource-constrained hardware full agent capabilities.

---

### 6. VoiceMem Long-term Memory Demo (`examples/voicemem`)

- **Cross-Session Memory Sidecar**:
  - Automatically extracts user preferences and factual records from continuous voice dialogues.
  - Injects relevant long-term memory into subsequent session contexts.

---

## 3. Quick Launch Reference

| Demo Scenario | Start Command | Required Configuration & Environment |
| --- | --- | --- |
| **Standard Web UI** | `npm run dev` | Set `QWEN_AUDIO_REALTIME_API_KEY` in root configuration |
| **Smart Cockpit** | `npm run example:smart-cockpit` | Amap API Key (for real POI & routing) + Bailian API Key |
| **Customer Service** | `npm run example:customer-service` | Bailian API Key (for realtime frontend & backend agent) |
| **X-Omni Vision** | `npm run example:x-omni` | `QWEN_AUDIO_REALTIME_MODEL=qwen3.5-omni-plus-realtime` |
| **WebRTC** | `npm run example:webrtc` | WebRTC-enabled gateway configuration |
| **AI Passport** | `npm run example:ai-passport` | LAN-connected ESP32-C3 card device |
