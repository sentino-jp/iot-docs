# Sentino IoT × Agora — Solution Overview (Business View)

> Intended for product and business decision-makers. This document focuses on **responsibility layering** and **how a voice conversation happens**.
>
> - Product capabilities: see [Architecture & Concepts §1](./architecture-en.md#product-capabilities-at-a-glance)
> - Detailed technical architecture: see [Architecture & Concepts §2](./architecture-en.md#2-overall-architecture) and [Technical Architecture Deep Dive](./architecture-technical-en.md)

---

## 1. Who Does What — Responsibility Layers

```mermaid
graph TB
    subgraph Delivery["Sentino Full-Stack Support (Choose as Needed)"]
        C1["Hardware Development<br/>(MCU + Microphone + Speaker + NFC Optional)"]
        C2["Firmware Development<br/>(MQTT + Agora RTC SDK Integration)"]
        C3["App Development<br/>(Provisioning + Device Mgmt + Character Mgmt)"]
    end

    subgraph SentinoIoT["Sentino IoT Platform"]
        S1["Device Cloud Access<br/>MQTT Broker + Device Authentication"]
        S2["Provisioning & Binding Service<br/>BLE Protocol + Device Lifecycle"]
        S3["AI Agent Management<br/>Character Config + Session Orchestration"]
        S4["Operations Support<br/>OTA Updates + Status Monitoring"]
    end

    subgraph SentinoAgent["Sentino Agent Platform"]
        S5["LLM Inference Scheduling<br/>Multi-Model Adaptation + Conversation Context"]
        S6["TTS Scheduling<br/>Multi-Provider Adaptation"]
        S7["Workflow Orchestration<br/>Function Calling + Memory"]
    end

    subgraph Agora["Agora"]
        A1["Real-Time Audio Network<br/>SD-RTN™ Global Low-Latency Transmission"]
        A2["AI Agent Runtime<br/>ASR Speech Recognition + Conversation Engine"]
    end

    subgraph External["Choose as Needed"]
        E1["Large Language Models (LLM)<br/>(Claude / ChatGPT / Gemini, etc.)"]
        E2["Text-to-Speech (TTS)<br/>(ElevenLabs / MiniMax, etc.)"]
    end

    C1 -.->|"Integrate"| C2
    C2 -.->|"Call"| S1
    C3 -.->|"Call"| S2
    S3 -->|"Create AI Agent"| A2
    A2 --> A1
    A2 -->|"Callback"| S5
    S5 -->|"Call"| E1
    S5 -->|"Call"| E2

    style Delivery fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#000
    style SentinoIoT fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#000
    style SentinoAgent fill:#FFF8E1,stroke:#F9A825,stroke-width:2px,color:#000
    style Agora fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#000
    style External fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px,color:#000
```

> **Core Value**: Sentino provides full-stack support from platform to hardware and App. Customers choose what to build in-house or delegate, enabling rapid product delivery.

---

## 2. System Architecture

```mermaid
graph TB
    Device["IoT Smart Hardware<br/>Dolls / Story Machines / Robots / Speakers"]
    App["Management App<br/>(Mobile)"]
    NFC["NFC Character Cards<br/>(Optional)"]
    Cloud["Sentino IoT Platform<br/>Device Mgmt · Agent Mgmt · Provisioning"]
    SentinoAgent["Sentino Agent Platform<br/>LLM Scheduling · TTS Scheduling · Workflow Orchestration"]
    AgoraAI["Agora Conversational AI Engine<br/>AI Agent Mgmt · Speech Recognition"]
    AgoraRTC["Agora SD-RTN™<br/>Global Real-Time Audio Network"]
    LLM["Large Language Models<br/>(Claude / ChatGPT / Gemini, etc.)"]
    TTS["Text-to-Speech<br/>(ElevenLabs / MiniMax, etc.)"]

    NFC -.-|"Tap to switch character"| Device
    App <-->|"BLE Bluetooth (first-time provisioning only)"| Device
    App <-->|"HTTPS"| Cloud
    Device <-->|"MQTT<br/>Signaling · Status · Voice channel params"| Cloud
    Device <-->|"Agora RTC<br/>Real-time audio stream"| AgoraRTC
    Cloud -->|"HTTPS<br/>Create/Stop AI Agent"| AgoraAI
    AgoraAI <--> AgoraRTC
    AgoraAI <-->|"HTTP Callback"| SentinoAgent
    SentinoAgent -->|"API"| LLM
    SentinoAgent -->|"API"| TTS

    style Device fill:#1565C0,stroke:#0D47A1,color:#fff,stroke-width:2px
    style Cloud fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px
    style SentinoAgent fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px
    style AgoraAI fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style AgoraRTC fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style LLM fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
    style TTS fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
```

**The device has two communication links:**
- **MQTT (Signaling Channel)**: Device <-> Sentino IoT Platform, used for device authentication, status reporting, and obtaining voice channel parameters
- **Agora RTC (Audio Channel)**: Device <-> Agora Audio Network, used for real-time voice conversations

---

## 3. How a Voice Conversation Happens

```mermaid
sequenceDiagram
    participant Device as IoT Device
    participant Cloud as Sentino IoT Platform
    participant Agora as Agora
    participant SA as Sentino Agent Platform

    Device->>Cloud: User presses button, device sends MQTT request
    Cloud->>Cloud: Query AI character config bound to device
    Cloud->>Agora: Create AI Agent (pass character config)
    Agora-->>Cloud: Agent ready
    Cloud-->>Device: Return voice channel parameters

    Device->>Agora: Join voice channel

    rect rgb(232, 245, 233)
    Note over Device, SA: Real-Time Voice Conversation
    Device->>Agora: User speaks -> send audio
    Agora->>SA: Speech recognition text callback
    SA->>SA: LLM inference + TTS synthesis
    SA-->>Agora: AI reply audio
    Agora-->>Device: Play AI reply
    end

    Device->>Agora: User releases button, device leaves channel
    Agora-->>Cloud: Device departure detected
    Cloud->>Cloud: Automatically clean up AI session
```

**Key Points**:
- The device only needs to do two things: send an MQTT request + join the voice channel
- Agora handles audio transmission and speech recognition; Sentino Agent Platform handles LLM inference and TTS synthesis
- After the conversation ends, the cloud automatically cleans up — no additional device action required

---

## 4. Related Documents

| Document | Target Reader | Content |
|----------|---------------|---------|
| [Architecture & Concepts](./architecture-en.md) | Developers | Core concepts, product capabilities, protocol overview, two product paths, glossary |
| [Technical Architecture Deep Dive](./architecture-technical-en.md) | Technical Evaluators / Architects | Detailed system topology, full data-flow sequence, key design decisions |
| [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) | Product Managers | User journey, NFC design, mass production readiness |
| [Device Integration Guide](guides/guide-device-en.md) | Firmware Engineers | MQTT protocol integration |
| [AI Voice Conversation Integration](guides/guide-ai-voice-en.md) | Firmware Engineers | Agora RTC integration |
