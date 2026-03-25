# Sentino IoT 开发者文档中心

欢迎使用 Sentino IoT 开发者文档。Sentino 是面向 AI 语音交互设备的物联网平台，提供从设备配网、云端通信到 AI 实时语音对话的完整解决方案。

---

## 阅读指引

根据你的角色选择阅读路径：

| 你的角色 | 推荐阅读顺序 |
|---|---|
| **嵌入式固件工程师** | [架构与概念](./architecture.md) → [快速入门(设备端)](./quickstart-device.md) → [设备端集成指南](./guide-device.md) → [AI 语音对话集成指南](./guide-ai-voice.md) → [MQTT 协议参考](./ref-mqtt.md) / [BLE 协议参考](./ref-ble.md) |
| **App 开发者** | [架构与概念](./architecture.md) → [快速入门(App端)](./quickstart-app.md) → [App 端集成指南](./guide-app.md) → [REST API 参考](./ref-rest-api.md) |
| **产品 / 项目经理** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](./solution-ai-toy.md) |
| **合作伙伴技术负责人** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](./solution-ai-toy.md) → [快速入门(设备端)](./quickstart-device.md) |

---

## 文档列表

### 入门

| 文档 | 说明 | 状态 |
|---|---|---|
| [架构与概念](./architecture.md) | 整体架构、核心概念、联网模式、术语表 | 已完成 |
| [快速入门 — 设备端](./quickstart-device.md) | 10 分钟跑通 MQTT 连接并验证 AI 语音接入 | 已完成 |
| [快速入门 — App 端](./quickstart-app.md) | curl 跑通登录 → 资产树 → 设备绑定的完整链路 | 已完成 |

### 集成指南

| 文档 | 说明 | 状态 |
|---|---|---|
| [设备端集成指南](./guide-device.md) | MQTT 鉴权、设备绑定、属性管理、OTA、断线重连、状态恢复 | 已完成 |
| [App 端集成指南](./guide-app.md) | BLE 配网(WiFi/4G)、绑定状态查询、设备管理 | 已完成 |
| [AI 语音对话集成指南](./guide-ai-voice.md) | 对话生命周期、Agora RTC 集成、打断/超时/断网处理 | 已完成 |

### 协议与 API 参考

| 文档 | 说明 | 状态 |
|---|---|---|
| [MQTT 协议参考](./ref-mqtt.md) | 连接参数、Topic、消息格式、全部事件/指令的字段定义 | 已完成 |
| [BLE 协议参考](./ref-ble.md) | 广播格式、V1 分包协议、应用层消息、状态码 | 已完成 |
| [REST API 参考](./ref-rest-api.md) | 认证、配网、设备管理、智能体管理接口 | 已完成 |

### 产品方案

| 文档 | 说明 | 状态 |
|---|---|---|
| [AI 玩偶接入方案](./solution-ai-toy.md) | 用户旅程、智能体管理、NFC 卡片、家长控制、安全合规 | 已完成 |

---

## 环境信息

| 项目 | 测试环境 |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883` (MQTT 5.0) |
| REST API | `https://api-iot.sentino.jp` |
| Agora RTC | 参数由云端动态下发 |

---

**维护者**: Sentino IoT Team
**最后更新**: 2026-03-25
