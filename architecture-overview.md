# Sentino IoT × Agora 方案概览（商务视角）

> 本文面向产品和商务决策者，重点呈现**责任分层**和**一次语音对话的发生过程**。
>
> - 产品能力一览见 [架构与概念 §1](./architecture.md#产品能力一览)
> - 详细技术架构见 [架构与概念 §2](./architecture.md#2-整体架构) 与 [技术架构详解](./architecture-technical.md)

---

## 1. 谁做什么 — 责任分层

```mermaid
graph TB
    subgraph Delivery["Sentino 全栈支持（按需选择）"]
        C1["硬件开发<br/>（主控 + 麦克风 + 扬声器 + NFC 可选）"]
        C2["固件开发<br/>（MQTT + Agora RTC SDK 集成）"]
        C3["App 开发<br/>（配网 + 设备管理 + 角色管理）"]
    end

    subgraph SentinoIoT["Sentino IoT 平台"]
        S1["设备云端接入<br/>MQTT Broker + 设备认证"]
        S2["配网绑定服务<br/>BLE 协议 + 设备生命周期"]
        S3["AI 智能体管理<br/>角色配置 + 会话编排"]
        S4["运维支撑<br/>OTA 升级 + 状态监控"]
    end

    subgraph SentinoAgent["Sentino Agent 平台"]
        S5["LLM 推理调度<br/>多模型适配 + 对话上下文"]
        S6["TTS 语音合成调度<br/>多供应商适配"]
        S7["工作流编排<br/>Function Calling + 记忆"]
    end

    subgraph Agora["Agora 声网提供"]
        A1["实时音频网络<br/>SD-RTN™ 全球低延迟传输"]
        A2["AI Agent 运行时<br/>语音识别 ASR + 对话引擎"]
    end

    subgraph External["按需选择"]
        E1["大语言模型 LLM<br/>（GPT / DeepSeek / 通义等）"]
        E2["语音合成 TTS<br/>（MiniMax / ElevenLabs 等）"]
    end

    C1 -.->|"集成"| C2
    C2 -.->|"调用"| S1
    C3 -.->|"调用"| S2
    S3 -->|"创建 AI Agent"| A2
    A2 --> A1
    A2 -->|"回调"| S5
    S5 -->|"调用"| E1
    S5 -->|"调用"| E2

    style Delivery fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#000
    style SentinoIoT fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#000
    style SentinoAgent fill:#FFF8E1,stroke:#F9A825,stroke-width:2px,color:#000
    style Agora fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#000
    style External fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px,color:#000
```

> **核心价值**：Sentino 提供从平台到硬件、App 的全栈支持，客户按需选择自研或委托，快速实现产品落地。

---

## 2. 系统架构

```mermaid
graph TB
    Device["IoT 智能硬件<br/>玩偶 / 故事机 / 机器人 / 音箱"]
    App["管理 App<br/>（手机）"]
    NFC["NFC 角色卡片<br/>（可选）"]
    Cloud["Sentino IoT 平台<br/>设备管理 · 智能体管理 · 配网服务"]
    SentinoAgent["Sentino Agent 平台<br/>LLM 调度 · TTS 调度 · 工作流编排"]
    AgoraAI["Agora 对话式 AI 引擎<br/>AI Agent 管理 · 语音识别"]
    AgoraRTC["Agora SD-RTN™<br/>全球实时音频网络"]
    LLM["大语言模型<br/>（GPT / DeepSeek 等）"]
    TTS["语音合成<br/>（MiniMax 等）"]

    NFC -.-|"触碰切换角色"| Device
    App <-->|"BLE 蓝牙（仅首次配网）"| Device
    App <-->|"HTTPS"| Cloud
    Device <-->|"MQTT<br/>信令 · 状态 · 获取语音通道参数"| Cloud
    Device <-->|"Agora RTC<br/>实时语音流"| AgoraRTC
    Cloud -->|"HTTPS<br/>创建/停止 AI Agent"| AgoraAI
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

**设备有两条通信链路：**
- **MQTT（信令通道）**：设备 ↔ Sentino IoT 平台，用于设备认证、状态上报、获取语音通道参数
- **Agora RTC（音频通道）**：设备 ↔ Agora 音频网络，用于实时语音对话

---

## 3. 一次语音对话是怎么发生的

```mermaid
sequenceDiagram
    participant Device as IoT 设备
    participant Cloud as Sentino IoT 平台
    participant Agora as Agora 声网
    participant SA as Sentino Agent 平台

    Device->>Cloud: 用户按键，设备发送 MQTT 请求
    Cloud->>Cloud: 查询设备绑定的 AI 角色配置
    Cloud->>Agora: 创建 AI Agent（传入角色配置）
    Agora-->>Cloud: Agent 就绪
    Cloud-->>Device: 返回语音通道参数

    Device->>Agora: 加入语音频道

    rect rgb(232, 245, 233)
    Note over Device, SA: 实时语音对话
    Device->>Agora: 用户说话 → 发送音频
    Agora->>SA: 语音识别文本回调
    SA->>SA: LLM 推理 + TTS 语音合成
    SA-->>Agora: AI 回复语音
    Agora-->>Device: 播放 AI 回复
    end

    Device->>Agora: 用户松键，设备离开频道
    Agora-->>Cloud: 检测到设备离开
    Cloud->>Cloud: 自动清理 AI 会话
```

**要点**：
- 设备只需做两件事：发 MQTT 请求 + 加入语音频道
- Agora 负责音频传输和语音识别，Sentino Agent 平台负责 LLM 推理和 TTS 语音合成
- 对话结束后云端自动清理，无需设备额外操作

---

## 4. 相关文档

| 文档 | 适用读者 | 内容 |
|------|----------|------|
| [架构与概念](./architecture.md) | 开发者 | 核心概念、产品能力一览、通信协议总览、两条产品路径对比、术语表 |
| [技术架构详解](./architecture-technical.md) | 技术评估 / 架构师 | 详细系统全景架构、完整数据流时序图、关键设计决策 |
| [AI 玩偶接入方案](solutions/solution-ai-toy.md) | 产品经理 | 用户旅程、NFC 设计、量产准备 |
| [设备端集成指南](guides/guide-device.md) | 固件工程师 | MQTT 协议集成 |
| [AI 语音对话集成](guides/guide-ai-voice.md) | 固件工程师 | Agora RTC 集成 |
