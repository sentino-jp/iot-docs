# Sentino IoT 开发者文档中心

<p>
  <a href="./README-en.md">English</a> &nbsp;·&nbsp;
  <strong>简体中文</strong>
</p>

欢迎使用 Sentino IoT 开发者文档。Sentino IoT 是面向 AI 语音交互设备的物联网平台，提供从设备配网、云端通信到 AI 实时语音对话的完整方案。

> **平台定位**：Sentino IoT 是端到端 IoT 方案——你的 App / 设备直接调 Sentino 云，无需客户自建后端。本文档只覆盖「物」的部分；自定义 Agent 行为（LLM/TTS 编排、记忆、工作流、自定义工具）属于 Sentino Agent 平台范畴，请联系 Sentino 团队获取相关文档。

---

## 相关开源仓库

| 仓库 | 内容 | 活跃度 | 用法 |
|---|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 固件示例工程（基于声网 RTSA Lite + ConvoAI）+ Web BLE 配网工具 | ![last commit](https://img.shields.io/github/last-commit/sentino-jp/sentino-iot-sample) | 拿模板编译/烧录/跑实物，详见仓内 `device/BUILD_GUIDE.md` |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | 基于 Flutter 3.x 的跨平台 IoT App 模板（Android / iOS / Windows / Web） | ![last commit](https://img.shields.io/github/last-commit/sentino-jp/sentino-app-sample) | 拿白标模板改 UI 与品牌即可发布；协议字段定义以本仓 `ref-rest-api.md` 为准 |

---

## 阅读指引

根据你的角色选择阅读路径：

| 你的角色 | 推荐阅读顺序 |
|---|---|
| **嵌入式固件工程师**（基础） | [架构与概念](./architecture.md) → [快速入门(设备端)](tutorials/quickstart-device.md) → [设备端集成指南](guides/guide-device.md) → [AI 语音对话集成指南](guides/guide-ai-voice.md) |
| **嵌入式固件工程师**（参考 + 实物） | [MQTT 协议参考](reference/ref-mqtt.md) / [BLE 协议参考](reference/ref-ble.md) → fork [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) 编译烧录 |
| **App 开发者（拿白标 Flutter 模板改）** | [架构与概念](./architecture.md) → 直接 fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample)，跟其 `doc/quick-start.md` 跑起来 → 字段细节回到本仓 [REST API 参考](reference/ref-rest-api.md) |
| **App 开发者（从零写 App）** | [架构与概念](./architecture.md) → [快速入门(App端)](tutorials/quickstart-app.md) → [App 端集成指南](guides/guide-app.md) → [REST API 参考](reference/ref-rest-api.md) |
| **产品 / 项目经理** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](solutions/solution-ai-toy.md) |
| **合作伙伴技术负责人** | [架构与概念](./architecture.md) → [AI 玩偶接入方案](solutions/solution-ai-toy.md) → 浏览[相关开源仓库](#相关开源仓库)看 sample 厚度 |

---

## 文档列表

### 入门

| 文档 | 说明 | 状态 |
|---|---|---|
| [架构与概念](./architecture.md) | 整体架构、产品能力、核心概念、联网模式、AI 语音对话模型、两条产品路径对比、术语表 | 已完成 |
| [快速入门 — 设备端](tutorials/quickstart-device.md) | 10 分钟跑通 MQTT 连接，验证签名和 Topic | 已完成 |
| [快速入门 — App 端](tutorials/quickstart-app.md) | curl 跑通登录 → 资产树 → 产品 → 设备 → 智能体 → 绑定的 6 步链路 | 已完成 |

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
| [BLE 协议参考](reference/ref-ble.md) | 协议分层、广播格式、GATT 服务、V1 分包协议、应用层消息、状态码、配网状态机、平台适配 | 已完成 |
| [REST API 参考](reference/ref-rest-api.md) | 认证（UID + Password 双模式）、配网、设备管理、Sentino 智能体接入与对话历史 | 已完成 |

### 产品方案

| 文档 | 说明 | 状态 |
|---|---|---|
| [AI 玩偶接入方案](solutions/solution-ai-toy.md) | 用户旅程、智能体管理、NFC 卡片、家长控制、安全合规 | 已完成 |

---

## 环境信息

| 项目 | 测试环境 |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883` (MQTT 5.0，明文，仅供 quickstart 验证使用；生产环境请走 TLS 端口 `8883`) |
| REST API | `https://api-iot.sentino.jp/api` |
| Agora RTC | 参数由云端动态下发 |
| 测试三元组 (UUID / KEY / MAC) | 联系 Sentino 团队为你的应用分配 |
| 测试 App 凭证 (`app_id` / `channel_identifier` / `package_name`) | 同上；本文档示例值仅供参考，生产请用 Sentino 为你分配的实际值 |

---

## 反馈与联系

| 渠道 | 用途 |
|---|---|
| GitHub issue | 文档错误、字段不一致、断链等技术问题：[sentino-jp/iot-docs/issues](https://github.com/sentino-jp/iot-docs/issues) |
| 商务对接 | 平台接入、定价、定制化方案：联系 Sentino 团队 |
| Sentino Agent 平台 | 自定义 LLM/TTS/工作流/记忆等能力：见顶部「平台定位」说明 |

---

**维护者**: Sentino IoT Team
**最后更新**: 2026-04-24
