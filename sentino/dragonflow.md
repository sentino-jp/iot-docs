# Agora × DragonFlow (Sentino) 架构图

```mermaid
graph TB
    subgraph User["👤 用户端"]
        App["Sentino Web App<br/>(React + Vite)"]
        MicSpk["🎤 麦克风 / 🔊 扬声器"]
    end

    subgraph Frontend["workflow-web (前端)"]
        VoiceModal["VoiceTestModal<br/>MeetingPage"]
        AgoraHook["useAgoraVoiceTest<br/>useMeetingAgora"]
        RTC_SDK["Agora RTC SDK<br/>(agora-rtc-sdk-ng)"]
        RTM_SDK["Agora RTM SDK<br/>(agora-rtm-sdk)"]
        ConvoAI_Lib["ConversationalAIAPI<br/>TranscriptMessageParser"]
        ApiSvc["agoraApiService<br/>startAgent / stopAgent"]
    end

    subgraph Agora["Agora 云服务"]
        SDR["SD-RTN™<br/>实时音频传输网络"]
        ConvoEngine["Conversational AI Engine<br/>/v2/projects/{appId}/join<br/>/v2/.../agents/{id}/leave"]
        ASR["ASR 语音识别<br/>(Agora / Soniox)"]
        AgentBot["Agora AI Agent 实例<br/>(运行在 Agora 侧)"]
    end

    subgraph Backend["DragonFlow 后端 (workflow-api)"]
        AgentCtrl["AgoraAgentController<br/>POST /api/agora/agent/start|stop"]
        AgentSvc["AgoraAgentService<br/>buildAgentConfig / startAgent"]
        MeetCtrl["MeetingController<br/>POST /api/meeting/chat"]
        AudioCtrl["AudioController<br/>POST /api/audio/chat"]
        WFExec["AgentWorkflowExecutor<br/>WorkflowNavigator"]
        DB["PostgreSQL<br/>(Agent 配置 / 版本 / Voice)"]
        Redis["Redis<br/>(会话状态缓存)"]
    end

    subgraph Engine["workflow-engine (核心引擎)"]
        LLM["LLMProviderFactory<br/>Gemini / OpenRouter / Grok<br/>VertexClaude / MiniMax"]
        TTS_E["TTS Provider<br/>Minimax / ElevenLabs / Qwen3"]
        FuncCall["FunctionCallProcessor<br/>ToolDispatchExecutor"]
        Memory["Jarvis Memory Service<br/>(长期记忆)"]
    end

    %% 用户交互
    MicSpk -->|音频采集| App
    App --> VoiceModal
    VoiceModal --> AgoraHook
    AgoraHook --> RTC_SDK
    AgoraHook --> RTM_SDK
    RTM_SDK --> ConvoAI_Lib
    AgoraHook --> ApiSvc

    %% 前端 → 后端 (Agent 生命周期管理)
    ApiSvc -->|"HTTP: 创建/停止 Agent"| AgentCtrl

    %% 后端 → Agora (注册 Agent)
    AgentCtrl --> AgentSvc
    AgentSvc -->|"HTTP POST: Agent 配置<br/>(ASR/LLM/TTS/VAD)"| ConvoEngine
    AgentSvc --> DB

    %% Agora 内部
    ConvoEngine -->|创建实例| AgentBot
    AgentBot -->|加入频道| SDR
    AgentBot --> ASR

    %% 实时音频流
    RTC_SDK <-->|"RTC 音频流<br/>(双向实时)"| SDR

    %% 实时转写
    RTM_SDK <-->|"RTM 消息<br/>(实时转写文本)"| SDR

    %% Agora Agent → DragonFlow 后端 (LLM 回调)
    AgentBot -->|"HTTP Callback<br/>用户语音转文本"| AudioCtrl
    AgentBot -->|"HTTP Callback<br/>(会议模式)"| MeetCtrl

    %% 后端处理链
    AudioCtrl --> WFExec
    MeetCtrl --> WFExec
    WFExec --> LLM
    WFExec --> FuncCall
    WFExec --> Memory
    FuncCall --> LLM
    WFExec --> TTS_E
    WFExec --> Redis

    %% 响应回流
    LLM -->|"SSE Streaming<br/>LLM 生成文本"| AgentBot
    TTS_E -->|音频合成| AgentBot

    %% 样式
    classDef agora fill:#1a73e8,stroke:#0d47a1,color:#fff
    classDef dragon fill:#34a853,stroke:#1b5e20,color:#fff
    classDef user fill:#f9ab00,stroke:#e65100,color:#000
    classDef engine fill:#7c4dff,stroke:#4a148c,color:#fff

    class SDR,ConvoEngine,ASR,AgentBot agora
    class AgentCtrl,AgentSvc,MeetCtrl,AudioCtrl,WFExec,DB,Redis dragon
    class App,MicSpk,VoiceModal,AgoraHook,RTC_SDK,RTM_SDK,ConvoAI_Lib,ApiSvc user
    class LLM,TTS_E,FuncCall,Memory engine
```

## 关键交互说明

| 阶段 | 流程 |
|------|------|
| **1. 创建 Agent** | 前端 → DragonFlow 后端 → Agora ConvoAI API (`/join`) → Agent 实例加入 RTC 频道 |
| **2. 实时语音** | 用户麦克风 → RTC SDK → SD-RTN™ ↔ Agora Agent（双向音频流） |
| **3. LLM 回调** | Agora Agent 将 ASR 文本通过 HTTP 回调发送到 DragonFlow 后端 (`/api/audio/chat`) |
| **4. 工作流执行** | DragonFlow 后端执行 Agent 工作流：LLM 推理 → Function Calling → 记忆检索 |
| **5. 响应回流** | LLM 流式文本 → Agora Agent → TTS 合成 → RTC 音频回传给用户 |
| **6. 实时转写** | Agora Agent 通过 RTM 通道将 ASR/TTS 文本推送到前端（实时字幕） |
| **7. 停止 Agent** | 前端 → DragonFlow 后端 → Agora ConvoAI API (`/leave`) → Agent 离开频道 |

**核心设计**: DragonFlow 不直接处理音频，而是将 LLM 推理能力作为 HTTP 回调服务暴露给 Agora Agent。Agora 负责所有音频传输、ASR 和 TTS，DragonFlow 负责 Agent 配置管理、工作流编排和 LLM 调用。