# Sentino IoT × Agora 技术架构详解

> 本文面向技术评估者和架构师，重点呈现**详细系统全景**和**完整数据流时序图**。
>
> - 中等抽象层级架构 + 通信协议总览见 [架构与概念 §2](./architecture.md#2-整体架构)
> - 两条产品路径对比见 [架构与概念 §8](./architecture.md#8-两条产品路径对比)
> - 商务视角的责任分层与简化对话流程见 [方案概览](./architecture-overview.md)

---

## 1. 系统全景架构

```mermaid
graph TB
    subgraph DeviceLayer["IoT 设备"]
        Device["智能硬件<br/>玩偶 / 故事机 / 机器人 / 音箱"]
        DevMQTT["MQTT Client<br/>MQTT 5.0 · HMAC-SHA256"]
        DevRTC["Agora RTC SDK<br/>OPUS · 16kHz · mono"]
        DevBLE["BLE GATT<br/>Service UUID 0xA101"]
        DevNFC["NFC Reader（可选）"]
        Device --- DevMQTT
        Device --- DevRTC
        Device --- DevBLE
        Device --- DevNFC
    end

    subgraph AppLayer["管理 App"]
        App["手机 App"]
        AppBLE["BLE Central"]
        AppREST["REST API Client"]
        App --- AppBLE
        App --- AppREST
    end

    NFC["NFC 角色卡片（可选）"]

    subgraph SentinoCloud["Sentino IoT 平台"]
        MQTTBroker["MQTT Broker<br/>设备认证 · 消息路由"]
        DevMgmt["设备生命周期管理<br/>配网绑定 · 在线状态 · OTA"]
        AgentMgmt["AI 智能体管理<br/>角色配置 · 会话编排"]
        RESTAPI["REST API 网关<br/>用户认证 · 设备管理"]
        MQTTBroker --- DevMgmt
        MQTTBroker --- AgentMgmt
        RESTAPI --- DevMgmt
        RESTAPI --- AgentMgmt
    end

    subgraph SentinoAgent["Sentino Agent 平台"]
        DFCtrl["Agent Controller<br/>Agent 生命周期管理"]
        DFExec["Workflow Executor<br/>工作流编排 · Function Calling"]
        DFProvider["LLM / TTS Provider<br/>多模型适配"]
        DFDB["PostgreSQL · Redis<br/>配置存储 · 会话缓存"]
        DFCtrl --- DFExec
        DFExec --- DFProvider
        DFExec --- DFDB
    end

    subgraph AgoraPlatform["Agora 声网平台"]
        ConvoAI["Conversational AI Engine<br/>Agent 实例管理 · ASR 语音识别"]
        SDRTN["SD-RTN™<br/>全球实时音频传输网络"]
        AgentInst["AI Agent 实例<br/>（运行在 Agora 侧）"]
        ConvoAI --> AgentInst
        AgentInst <--> SDRTN
    end

    subgraph ExternalAI["外部 AI 服务"]
        LLM["大语言模型 LLM<br/>GPT · DeepSeek · 通义千问"]
        TTS["语音合成 TTS<br/>MiniMax · ElevenLabs"]
    end

    %% IoT 路径
    NFC -.-|"触碰"| DevNFC
    AppBLE <-->|"BLE（仅首次配网）"| DevBLE
    AppREST <-->|"HTTPS"| RESTAPI
    DevMQTT <-->|"MQTT 5.0（WiFi / 4G）<br/>信令 · 状态 · RTC 参数"| MQTTBroker
    DevRTC <-->|"Agora RTC（UDP）<br/>实时语音流"| SDRTN
    AgentMgmt -->|"HTTPS<br/>创建/停止 Agent"| ConvoAI

    %% Agora Agent 回调 Sentino Agent 平台
    AgentInst -->|"HTTP Callback<br/>ASR 文本回调"| DFExec
    DFProvider -->|"SSE Streaming<br/>LLM 推理结果"| AgentInst

    %% Sentino Agent 平台调度外部 AI 服务
    DFProvider -->|"API 调用"| LLM
    DFProvider -->|"API 调用"| TTS

    %% Sentino Agent 平台也可独立创建 Agent
    DFCtrl -->|"HTTPS<br/>创建/停止 Agent"| ConvoAI

    %% 样式
    style DeviceLayer fill:#E3F2FD,stroke:#1565C0,stroke-width:2px
    style AppLayer fill:#E3F2FD,stroke:#1565C0,stroke-width:1px
    style SentinoCloud fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px
    style SentinoAgent fill:#FFF8E1,stroke:#F9A825,stroke-width:2px
    style AgoraPlatform fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px
    style ExternalAI fill:#F3E5F5,stroke:#7B1FA2,stroke-width:2px
```

---

## 2. IoT 设备语音对话 — 完整数据流

```mermaid
sequenceDiagram
    participant User as 用户
    participant Device as IoT 设备
    participant Cloud as Sentino IoT 平台
    participant ConvoAI as Agora AI 引擎
    participant RTN as SD-RTN™
    participant SA as Sentino Agent 平台

    Note over User, SA: 前提：设备已完成配网绑定，MQTT 在线，已绑定 AI 智能体

    rect rgb(227, 242, 253)
    Note over User, ConvoAI: 会话建立
    User->>Device: 按键（或 NFC 触碰）
    Device->>Cloud: MQTT: agora_agent_device_access
    Cloud->>Cloud: 查询设备绑定的智能体配置<br/>（Prompt · LLM 模型 · TTS 声音）
    Cloud->>ConvoAI: HTTPS POST: 创建 AI Agent<br/>（传入智能体配置 + RTC 频道参数）
    ConvoAI-->>Cloud: 返回 agent_id（Agent 已就绪）
    ConvoAI->>RTN: AI Agent 加入 RTC 频道
    Cloud-->>Device: MQTT: 返回 RTC 参数<br/>（appId · token · channelName · uid）
    Device->>RTN: Agora RTC SDK: 加入频道
    RTN-->>Device: 加入成功
    end

    rect rgb(232, 245, 233)
    Note over User, SA: 实时语音对话（循环）
    User->>Device: 说话
    Device->>RTN: 发送音频流（OPUS · 16kHz · mono）
    RTN->>ConvoAI: 转发音频给 AI Agent
    ConvoAI->>ConvoAI: ASR 语音识别
    ConvoAI->>SA: HTTP Callback: 发送识别文本 + 对话上下文
    SA->>SA: LLM 推理（调用大模型）
    SA->>SA: TTS 语音合成
    SA-->>ConvoAI: SSE Streaming: 返回 AI 回复语音
    ConvoAI->>RTN: 发送 AI 语音
    RTN-->>Device: 转发 AI 语音
    Device->>User: 扬声器播放 AI 回复
    end

    rect rgb(252, 228, 236)
    Note over User, ConvoAI: 会话结束
    User->>Device: 松开按键 / 超时
    Device->>RTN: Agora RTC SDK: 离开频道
    RTN-->>Cloud: 检测到设备离开
    Cloud->>ConvoAI: HTTPS POST: 停止 AI Agent
    ConvoAI->>RTN: AI Agent 离开频道
    Cloud->>Cloud: 清理会话资源
    end
```

---

## 3. 关键设计决策

| 决策 | 说明 |
|------|------|
| **MQTT 只做信令** | MQTT 负责设备认证、状态上报、获取 RTC 参数，不承载音频流。音频走 Agora RTC（UDP），延迟更低 |
| **Agora 负责音频传输和 ASR** | Agora 提供实时音频网络、AI Agent 运行时和语音识别，LLM 和 TTS 通过 HTTP Callback 回调 Sentino Agent 平台处理 |
| **设备端极简** | 设备只需：(1) 发一条 MQTT 消息 (2) 用返回的参数加入 RTC 频道。AI 配置、Agent 创建、会话清理全部在云端 |
| **AI Agent 先于设备就绪** | Sentino 云先在 Agora 创建 Agent，Agent 加入频道等待，然后才通知设备加入。保证设备进来就能对话 |
| **自动清理** | 设备只需离开 RTC 频道，云端自动检测并清理 Agent 和会话资源，无需设备发额外消息 |
| **NFC 即切即聊**（可选） | 设备如配备 NFC，卡片触碰后上报标识，云端自动匹配角色并创建新 Agent，一步完成切换 + 开始对话 |

---

> 通信协议总览（设备 / App / Sentino / Agora / Sentino Agent / LLM-TTS 之间所有通道）见 [架构与概念 §2 通信协议总览](./architecture.md#通信协议总览)。

---

**相关文档**：[架构与概念](./architecture.md) | [方案概览（销售版）](./architecture-overview.md)
