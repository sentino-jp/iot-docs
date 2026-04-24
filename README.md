# Sentino IoT 开发者文档中心

欢迎使用 Sentino IoT 开发者文档。Sentino IoT 是面向 AI 语音交互设备的物联网平台，提供从设备配网、云端通信到 AI 实时语音对话的完整方案。

> **平台定位**：Sentino IoT 是 **Tuya 模式**（你的 App / 设备直接调 Sentino 云，没有客户后端这一层），不是 AWS IoT 模式。本文档只覆盖「物」的部分；自定义 Agent 行为（LLM/TTS 编排、记忆、工作流、自定义工具）属于 Sentino Agent 平台范畴，请联系 Sentino 团队获取相关文档。

---

## 相关开源仓库

| 仓库 | 内容 | 用法 |
|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 固件示例工程（基于声网 RTSA Lite + ConvoAI）+ Web BLE 配网工具 | 拿模板编译/烧录/跑实物，详见仓内 `device/BUILD_GUIDE.md` |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | 基于 Flutter 3.x 的跨平台 IoT App 模板（Android / iOS / Windows / Web） | 拿白标模板改 UI 与品牌即可发布；协议字段定义以本仓 `ref-rest-api.md` 为准 |

---

## 阅读指引

根据你的角色选择阅读路径：

| 你的角色 | 推荐阅读顺序 |
|---|---|
| **嵌入式固件工程师** | [架构与概念](./architecture.md) → [快速入门(设备端)](tutorials/quickstart-device.md) → [设备端集成指南](guides/guide-device.md) → [AI 语音对话集成指南](guides/guide-ai-voice.md) → [MQTT 协议参考](reference/ref-mqtt.md) / [BLE 协议参考](reference/ref-ble.md) → 拿 [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) 跑实物 |
| **App 开发者（拿白标 Flutter 模板改）** | [架构与概念](./architecture.md) → 直接 fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample)，跟其 `doc/quick-start.md` 跑起来 → 字段细节回到本仓 [REST API 参考](reference/ref-rest-api.md) |
| **App 开发者（从零写 App）** | [架构与概念](./architecture.md) → [快速入门(App端)](tutorials/quickstart-app.md) → [App 端集成指南](guides/guide-app.md) → [REST API 参考](reference/ref-rest-api.md) |
| **产品 / 项目经理** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](solutions/solution-ai-toy.md) |
| **合作伙伴技术负责人** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](solutions/solution-ai-toy.md) → [快速入门(设备端)](tutorials/quickstart-device.md) |

---

## 文档列表

### 入门

| 文档 | 说明 | 状态 |
|---|---|---|
| [架构与概念](./architecture.md) | 整体架构、核心概念、联网模式、术语表 | 已完成 |
| [快速入门 — 设备端](tutorials/quickstart-device.md) | 10 分钟跑通 MQTT 连接并验证 AI 语音接入 | 已完成 |
| [快速入门 — App 端](tutorials/quickstart-app.md) | curl 跑通登录 → 获取账户 ID → 设备绑定的完整链路 | 已完成 |

### 集成指南

| 文档 | 说明 | 状态 |
|---|---|---|
| [设备端集成指南](guides/guide-device.md) | MQTT 鉴权、设备绑定、属性管理、OTA、断线重连、状态恢复 | 已完成 |
| [App 端集成指南](guides/guide-app.md) | BLE 配网(WiFi/4G)、绑定状态查询、设备管理 | 已完成 |
| [AI 语音对话集成指南](guides/guide-ai-voice.md) | 对话生命周期、Agora RTC 集成、打断/超时/断网处理 | 已完成 |

### 协议与 API 参考

| 文档 | 说明 | 状态 |
|---|---|---|
| [MQTT 协议参考](reference/ref-mqtt.md) | 连接参数、Topic、消息格式、全部事件/指令的字段定义 | 已完成 |
| [BLE 协议参考](reference/ref-ble.md) | 广播格式、V1 分包协议、应用层消息、状态码 | 已完成 |
| [REST API 参考](reference/ref-rest-api.md) | 认证、配网、设备管理、智能体管理接口 | 已完成 |

### 产品方案

| 文档 | 说明 | 状态 |
|---|---|---|
| [AI 玩偶接入方案](solutions/solution-ai-toy.md) | 用户旅程、智能体管理、NFC 卡片、家长控制、安全合规 | 已完成 |

---

## 环境信息

| 项目 | 测试环境 |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883` (MQTT 5.0) |
| REST API | `https://api-iot.sentino.jp/api` |
| Agora RTC | 参数由云端动态下发 |

---

**维护者**: Sentino IoT Team
**最后更新**: 2026-03-25
