# Sentino IoT × Agora — Technical Architecture Deep Dive

> Intended for technical evaluators and architects. This document focuses on the **detailed system topology** and the **full voice-conversation data flow**.
>
> - Mid-level architecture + protocol overview: see [Architecture & Concepts §2](./architecture-en.md#2-overall-architecture)
> - Two product paths comparison: see [Architecture & Concepts §8](./architecture-en.md#8-two-product-paths)
> - Business view (responsibility layering, simplified conversation flow): see [Solution Overview](./architecture-overview-en.md)

---

## 1. System Architecture Overview

```mermaid
graph TB
    subgraph DeviceLayer["IoT Device"]
        Device["Smart Hardware<br/>Dolls / Story Machines / Robots / Speakers"]
        DevMQTT["MQTT Client<br/>MQTT 5.0 · HMAC-SHA256"]
        DevRTC["Agora RTC SDK<br/>OPUS · 16kHz · mono"]
        DevBLE["BLE GATT<br/>Service UUID 0xA101"]
        DevNFC["NFC Reader (Optional)"]
        Device --- DevMQTT
        Device --- DevRTC
        Device --- DevBLE
        Device --- DevNFC
    end

    subgraph AppLayer["Management App"]
        App["Mobile App"]
        AppBLE["BLE Central"]
        AppREST["REST API Client"]
        App --- AppBLE
        App --- AppREST
    end

    NFC["NFC Character Cards (Optional)"]

    subgraph SentinoCloud["Sentino IoT Platform"]
        MQTTBroker["MQTT Broker<br/>Device Auth · Message Routing"]
        DevMgmt["Device Lifecycle Management<br/>Provisioning & Binding · Online Status · OTA"]
        AgentMgmt["AI Agent Management<br/>Character Config · Session Orchestration"]
        RESTAPI["REST API Gateway<br/>User Auth · Device Management"]
        MQTTBroker --- DevMgmt
        MQTTBroker --- AgentMgmt
        RESTAPI --- DevMgmt
        RESTAPI --- AgentMgmt
    end

    subgraph SentinoAgent["Sentino Agent Platform"]
        DFCtrl["Agent Controller<br/>Agent Lifecycle Management"]
        DFExec["Workflow Executor<br/>Workflow Orchestration · Function Calling"]
        DFProvider["LLM / TTS Provider<br/>Multi-Model Adaptation"]
        DFDB["PostgreSQL · Redis<br/>Config Storage · Session Cache"]
        DFCtrl --- DFExec
        DFExec --- DFProvider
        DFExec --- DFDB
    end

    subgraph AgoraPlatform["Agora Platform"]
        ConvoAI["Conversational AI Engine<br/>Agent Instance Mgmt · ASR Speech Recognition"]
        SDRTN["SD-RTN™<br/>Global Real-Time Audio Transmission Network"]
        AgentInst["AI Agent Instance<br/>(Runs on Agora Side)"]
        ConvoAI --> AgentInst
        AgentInst <--> SDRTN
    end

    subgraph ExternalAI["External AI Services"]
        LLM["Large Language Models (LLM)<br/>Claude · ChatGPT · Gemini"]
        TTS["Text-to-Speech (TTS)<br/>ElevenLabs · MiniMax"]
    end

    %% IoT path
    NFC -.-|"Tap"| DevNFC
    AppBLE <-->|"BLE (first-time provisioning only)"| DevBLE
    AppREST <-->|"HTTPS"| RESTAPI
    DevMQTT <-->|"MQTT 5.0 (WiFi / 4G)<br/>Signaling · Status · RTC Params"| MQTTBroker
    DevRTC <-->|"Agora RTC (UDP)<br/>Real-time audio stream"| SDRTN
    AgentMgmt -->|"HTTPS<br/>Create/Stop Agent"| ConvoAI

    %% Agora Agent callbacks to Sentino Agent Platform
    AgentInst -->|"HTTP Callback<br/>ASR text callback"| DFExec
    DFProvider -->|"SSE Streaming<br/>LLM inference results"| AgentInst

    %% Sentino Agent Platform dispatches to external AI services
    DFProvider -->|"API Call"| LLM
    DFProvider -->|"API Call"| TTS

    %% Sentino Agent Platform can also independently create Agents
    DFCtrl -->|"HTTPS<br/>Create/Stop Agent"| ConvoAI

    %% Styles
    style DeviceLayer fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    style AppLayer fill:#E3F2FD,stroke:#1565C0,stroke-width:1px
    style SentinoCloud fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px
    style SentinoAgent fill:#FFF8E1,stroke:#F9A825,stroke-width:2px
    style AgoraPlatform fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style ExternalAI fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
```

---

## 2. IoT Device Voice Conversation — Complete Data Flow

```mermaid
sequenceDiagram
    participant User as User
    participant Device as IoT Device
    participant Cloud as Sentino IoT Platform
    participant ConvoAI as Agora AI Engine
    participant RTN as SD-RTN™
    participant SA as Sentino Agent Platform

    Note over User, SA: Prerequisite: Device provisioned & bound, MQTT online, AI agent assigned

    rect rgb(227, 242, 253)
    Note over User, ConvoAI: Session Establishment
    User->>Device: Press button (or NFC tap)
    Device->>Cloud: MQTT: agora_agent_device_access
    Cloud->>Cloud: Query agent config bound to device<br/>(Prompt · LLM model · TTS voice)
    Cloud->>ConvoAI: HTTPS POST: Create AI Agent<br/>(pass agent config + RTC channel params)
    ConvoAI-->>Cloud: Return agent_id (Agent ready)
    ConvoAI->>RTN: AI Agent joins RTC channel
    Cloud-->>Device: MQTT: Return RTC params<br/>(appId · token · channelName · uid)
    Device->>RTN: Agora RTC SDK: Join channel
    RTN-->>Device: Join successful
    end

    rect rgb(232, 245, 233)
    Note over User, SA: Real-Time Voice Conversation (Loop)
    User->>Device: Speak
    Device->>RTN: Send audio stream (OPUS · 16kHz · mono)
    RTN->>ConvoAI: Forward audio to AI Agent
    ConvoAI->>ConvoAI: ASR speech recognition
    ConvoAI->>SA: HTTP Callback: Send recognized text + conversation context
    SA->>SA: LLM inference (call language model)
    SA->>SA: TTS synthesis
    SA-->>ConvoAI: SSE Streaming: Return AI reply audio
    ConvoAI->>RTN: Send AI audio
    RTN-->>Device: Forward AI audio
    Device->>User: Speaker plays AI reply
    end

    rect rgb(252, 228, 236)
    Note over User, ConvoAI: Session Termination
    User->>Device: Release button / timeout
    Device->>RTN: Agora RTC SDK: Leave channel
    RTN-->>Cloud: Device departure detected
    Cloud->>ConvoAI: HTTPS POST: Stop AI Agent
    ConvoAI->>RTN: AI Agent leaves channel
    Cloud->>Cloud: Clean up session resources
    end
```

---

## 3. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **MQTT for Signaling Only** | MQTT handles device authentication, status reporting, and obtaining RTC parameters — it does not carry audio streams. Audio travels via Agora RTC (UDP) for lower latency |
| **Agora Handles Audio Transport and ASR** | Agora provides the real-time audio network, AI Agent runtime, and speech recognition. LLM and TTS are handled by the Sentino Agent Platform via HTTP Callback |
| **Minimal Device Footprint** | The device only needs to: (1) send one MQTT message, (2) join the RTC channel with the returned parameters. AI configuration, Agent creation, and session cleanup all happen in the cloud |
| **AI Agent Ready Before Device** | Sentino cloud creates the Agent on Agora first; the Agent joins the channel and waits, then notifies the device to join. This guarantees the device can start talking immediately upon entry |
| **Automatic Cleanup** | The device only needs to leave the RTC channel; the cloud automatically detects this and cleans up the Agent and session resources — no additional device messages required |
| **NFC Tap-and-Talk** (Optional) | If the device has NFC, a card tap reports the identifier; the cloud automatically matches the character and creates a new Agent — switching + starting a conversation in one step |

---

> The full communication protocol overview (every channel between Device / App / Sentino / Agora / Sentino Agent / LLM-TTS) is in [Architecture & Concepts §2 Communication Protocol Overview](./architecture-en.md#communication-protocol-overview).

---

**Related Documents**: [Architecture & Concepts](./architecture-en.md) | [Solution Overview](./architecture-overview-en.md)
