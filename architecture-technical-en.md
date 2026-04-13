# Sentino IoT x Agora Technical Architecture Deep Dive

> This document is intended for technical evaluators and architects, providing a detailed view of the Sentino IoT platform and Agora's system architecture, communication protocols, and data flows.

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

## 3. Two Product Paths Compared

The Sentino ecosystem offers two paths to integrate Agora voice AI, sharing the same Agora Conversational AI Engine and SD-RTN:

| Comparison | IoT Device Path | Sentino Agent Platform Web Path |
|------------|----------------|--------------------------------|
| **Access Terminal** | Embedded hardware (dolls, speakers, etc.) | Web browser |
| **Audio Carrier** | Agora RTC C SDK (RTOS) | Agora RTC Web SDK (Browser) |
| **Signaling Channel** | MQTT 5.0 | HTTPS REST API |
| **Agent Creator** | Sentino IoT Platform | Sentino Agent Platform |
| **LLM/TTS Caller** | Sentino Agent Platform (Agora calls back via HTTP Callback) | Sentino Agent Platform (same) |
| **Workflow Capabilities** | Supports Function Calling, memory retrieval, workflow orchestration | Supports Function Calling, memory retrieval, workflow orchestration |
| **IoT-Exclusive Capabilities** | **Device Control** — Function Calling sends commands via RTC channel to control hardware (expressions, actions, LEDs, volume, etc.) | — |
| **Use Cases** | Consumer electronics (dolls, story machines, educational robots) | Enterprise & consumer AI Agent applications (customer service, meeting assistants, smart assistants, etc.) |

**Architectural Commonalities**: Both paths share the same Sentino Agent Platform workflow engine, both supporting Function Calling, memory retrieval, and workflow orchestration. Agora handles audio transmission and ASR; the Sentino Agent Platform handles LLM inference and TTS synthesis. Agora does not call LLM and TTS directly.

**Path Differences**:
- **IoT Path**: Minimal device complexity, triggered via MQTT signaling, supports full workflow capabilities, and can control device hardware via Function Calling through the RTC channel (e.g., expressions, actions, LEDs, volume, etc.)
- **Web Path**: Triggered via HTTPS, supports Function Calling, memory retrieval, and workflow orchestration

---

## 4. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **MQTT for Signaling Only** | MQTT handles device authentication, status reporting, and obtaining RTC parameters — it does not carry audio streams. Audio travels via Agora RTC (UDP) for lower latency |
| **Agora Handles Audio Transport and ASR** | Agora provides the real-time audio network, AI Agent runtime, and speech recognition. LLM and TTS are handled by the Sentino Agent Platform via HTTP Callback |
| **Minimal Device Footprint** | The device only needs to: (1) send one MQTT message, (2) join the RTC channel with the returned parameters. AI configuration, Agent creation, and session cleanup all happen in the cloud |
| **AI Agent Ready Before Device** | Sentino cloud creates the Agent on Agora first; the Agent joins the channel and waits, then notifies the device to join. This guarantees the device can start talking immediately upon entry |
| **Automatic Cleanup** | The device only needs to leave the RTC channel; the cloud automatically detects this and cleans up the Agent and session resources — no additional device messages required |
| **NFC Tap-and-Talk** (Optional) | If the device has NFC, a card tap reports the identifier; the cloud automatically matches the character and creates a new Agent — switching + starting a conversation in one step |

---

## 5. Communication Protocol Overview

| Channel | Protocol | Purpose | When Used |
|---------|----------|---------|-----------|
| Device <-> Sentino IoT Platform | **MQTT 5.0** | Device auth, binding, status reporting, command dispatch, obtaining RTC params | Always connected after device powers on |
| App <-> Device | **BLE** (GATT) | First-time provisioning to transfer binding info (WiFi credentials or userId) | First-time provisioning only |
| App <-> Sentino IoT Platform | **HTTPS** (REST API) | User login, device management, agent management | While App is running |
| Device <-> Agora | **RTC** (UDP) | Real-time bidirectional audio transmission | During voice conversations only |
| Sentino IoT Platform <-> Agora | **HTTPS** | Create/Stop AI Agent | At voice conversation start/end |
| Agora <-> Sentino Agent Platform | **HTTPS** + **HTTP Callback** + **SSE** | Agent management + ASR text callback + LLM/TTS result return | During voice conversations |
| Sentino Agent Platform <-> LLM/TTS | **HTTPS** | AI inference, speech synthesis | During voice conversations |

---

**Related Documents**: [Solution Overview](./architecture-overview-en.md) | [Architecture & Concepts](./architecture-en.md) | [Sentino Agent Platform Architecture](./sentino/agent-platform-en.md)
