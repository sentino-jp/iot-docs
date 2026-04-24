# Sentino IoT 开发者文档中心

<p>
  <a href="./README-en.md">English</a> &nbsp;·&nbsp;
  <strong>简体中文</strong>
</p>

Sentino IoT 是面向 AI 语音交互设备的物联网平台。设备配网 → MQTT 接入 → AI 实时语音对话——App / 设备直接调 Sentino 云，无需自建后端。

> 自定义 Agent 行为（LLM/TTS 编排、记忆、工作流、自定义工具）属于 Sentino Agent 平台范畴，请联系 Sentino 团队获取相关文档。

---

## 开源仓库

| 仓库 | 用途 | 活跃度 |
|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 固件示例（声网 RTSA + ConvoAI）+ Web BLE 配网工具 | ![](https://img.shields.io/github/last-commit/sentino-jp/sentino-iot-sample) |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | Flutter 3.x 跨平台 App 模板（Android / iOS / Windows / Web） | ![](https://img.shields.io/github/last-commit/sentino-jp/sentino-app-sample) |

---

## 文档

先读 [架构与概念](./architecture.md)（整体架构 / 产品能力 / 通信协议 / 联网模式 / 术语表），然后按任务下钻：

**写嵌入式固件**
- 入门：[设备端 quickstart](tutorials/quickstart-device.md)（10 分钟跑通 MQTT）→ [设备集成](guides/guide-device.md) → [AI 语音](guides/guide-ai-voice.md)
- 协议：[MQTT 参考](reference/ref-mqtt.md) · [BLE 参考](reference/ref-ble.md)
- 实物：fork [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) 编译烧录

**做 App**
- 拿白标：fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample) → 改 UI → 字段查 [REST API 参考](reference/ref-rest-api.md)
- 从零写：[App quickstart](tutorials/quickstart-app.md) → [App 集成](guides/guide-app.md) → [REST API 参考](reference/ref-rest-api.md)

**评估方案**
- [AI 玩偶接入方案](solutions/solution-ai-toy.md)（用户旅程、NFC、家长控制、安全合规）

---

## 环境

| | |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883`（MQTT 5.0；生产用 TLS `8883`） |
| REST API | `https://api-iot.sentino.jp/api` |
| Agora RTC | 参数由云端动态下发 |

测试三元组（UUID / KEY / MAC）和 App 凭证（`app_id` / `channel_identifier` / `package_name`）请联系 Sentino 团队为你的应用分配。

---

## 反馈

- 文档错误 / 字段不一致 / 断链：[sentino-jp/iot-docs/issues](https://github.com/sentino-jp/iot-docs/issues)
- 商务接入 / 定制方案：联系 Sentino 团队
