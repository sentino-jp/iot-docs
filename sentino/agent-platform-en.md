# Agora x Sentino Agent Platform Architecture

```mermaid
graph TB
    subgraph User["User"]
        App["Sentino Web App<br/>(React + Vite)"]
        MicSpk["Microphone / Speaker"]
    end

    subgraph Frontend["workflow-web (Frontend)"]
        VoiceModal["VoiceTestModal<br/>MeetingPage"]
        AgoraHook["useAgoraVoiceTest<br/>useMeetingAgora"]
        RTC_SDK["Agora RTC SDK<br/>(agora-rtc-sdk-ng)"]
        RTM_SDK["Agora RTM SDK<br/>(agora-rtm-sdk)"]
        ConvoAI_Lib["ConversationalAIAPI<br/>TranscriptMessageParser"]
        ApiSvc["agoraApiService<br/>startAgent / stopAgent"]
    end

    subgraph Agora["Agora Cloud Services"]
        SDR["SD-RTN™<br/>Real-Time Audio Transmission Network"]
        ConvoEngine["Conversational AI Engine<br/>/v2/projects/{appId}/join<br/>/v2/.../agents/{id}/leave"]
        ASR["ASR Speech Recognition<br/>(Agora / Soniox)"]
        AgentBot["Agora AI Agent Instance<br/>(Runs on Agora Side)"]
    end

    subgraph Backend["Sentino Agent Platform (workflow-api)"]
        AgentCtrl["AgoraAgentController<br/>POST /api/agora/agent/start|stop"]
        AgentSvc["AgoraAgentService<br/>buildAgentConfig / startAgent"]
        MeetCtrl["MeetingController<br/>POST /api/meeting/chat"]
        AudioCtrl["AudioController<br/>POST /api/audio/chat"]
        WFExec["AgentWorkflowExecutor<br/>WorkflowNavigator"]
        DB["PostgreSQL<br/>(Agent Config / Versions / Voice)"]
        Redis["Redis<br/>(Session State Cache)"]
    end

    subgraph Engine["workflow-engine (Core Engine)"]
        LLM["LLMProviderFactory<br/>Claude / ChatGPT / Gemini"]
        TTS_E["TTS Provider<br/>ElevenLabs / MiniMax"]
        FuncCall["FunctionCallProcessor<br/>ToolDispatchExecutor"]
        Memory["Jarvis Memory Service<br/>(Long-Term Memory)"]
    end

    %% User interaction
    MicSpk -->|Audio capture| App
    App --> VoiceModal
    VoiceModal --> AgoraHook
    AgoraHook --> RTC_SDK
    AgoraHook --> RTM_SDK
    RTM_SDK --> ConvoAI_Lib
    AgoraHook --> ApiSvc

    %% Frontend -> Backend (Agent lifecycle management)
    ApiSvc -->|"HTTP: Create/Stop Agent"| AgentCtrl

    %% Backend -> Agora (Register Agent)
    AgentCtrl --> AgentSvc
    AgentSvc -->|"HTTP POST: Agent Config<br/>(ASR/LLM/TTS/VAD)"| ConvoEngine
    AgentSvc --> DB

    %% Agora internals
    ConvoEngine -->|Create instance| AgentBot
    AgentBot -->|Join channel| SDR
    AgentBot --> ASR

    %% Real-time audio stream
    RTC_SDK <-->|"RTC Audio Stream<br/>(Bidirectional Real-Time)"| SDR

    %% Real-time transcription
    RTM_SDK <-->|"RTM Messages<br/>(Real-Time Transcription Text)"| SDR

    %% Agora Agent -> Sentino Agent Platform (LLM callback)
    AgentBot -->|"HTTP Callback<br/>User speech-to-text"| AudioCtrl
    AgentBot -->|"HTTP Callback<br/>(Meeting mode)"| MeetCtrl

    %% Backend processing chain
    AudioCtrl --> WFExec
    MeetCtrl --> WFExec
    WFExec --> LLM
    WFExec --> FuncCall
    WFExec --> Memory
    FuncCall --> LLM
    WFExec --> TTS_E
    WFExec --> Redis

    %% Response flow
    LLM -->|"SSE Streaming<br/>LLM generated text"| AgentBot
    TTS_E -->|Audio synthesis| AgentBot

    %% Styles
    classDef agora fill:#1a73e8,stroke:#0d47a1,color:#fff
    classDef dragon fill:#34a853,stroke:#1b5e20,color:#fff
    classDef user fill:#f9ab00,stroke:#e65100,color:#000
    classDef engine fill:#7c4dff,stroke:#4a148c,color:#fff

    class SDR,ConvoEngine,ASR,AgentBot agora
    class AgentCtrl,AgentSvc,MeetCtrl,AudioCtrl,WFExec,DB,Redis dragon
    class App,MicSpk,VoiceModal,AgoraHook,RTC_SDK,RTM_SDK,ConvoAI_Lib,ApiSvc user
    class LLM,TTS_E,FuncCall,Memory engine
```

## Key Interaction Descriptions

| Stage | Flow |
|------|------|
| **1. Create Agent** | Frontend -> Sentino Agent Platform -> Agora ConvoAI API (`/join`) -> Agent instance joins RTC channel |
| **2. Real-Time Voice** | User microphone -> RTC SDK -> SD-RTN™ <-> Agora Agent (bidirectional audio stream) |
| **3. LLM Callback** | Agora Agent sends ASR text to Sentino Agent Platform via HTTP callback (`/api/audio/chat`) |
| **4. Workflow Execution** | Sentino Agent Platform executes Agent workflow: LLM inference -> Function Calling -> Memory retrieval |
| **5. Response Flow** | LLM streaming text -> Agora Agent -> TTS synthesis -> RTC audio sent back to user |
| **6. Real-Time Transcription** | Agora Agent pushes ASR/TTS text to frontend via RTM channel (real-time captions) |
| **7. Stop Agent** | Frontend -> Sentino Agent Platform -> Agora ConvoAI API (`/leave`) -> Agent leaves channel |

**Core Design**: The Sentino Agent Platform does not handle audio directly. Instead, it exposes LLM inference capabilities as an HTTP callback service for Agora Agents. Agora handles all audio transmission and ASR; the Sentino Agent Platform handles Agent configuration management, workflow orchestration, and LLM invocation.
