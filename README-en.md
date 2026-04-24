# Sentino IoT Developer Documentation Hub

English (current) · [简体中文](./README.md)

Welcome to the Sentino IoT developer documentation. Sentino IoT is an IoT platform purpose-built for AI voice-interaction devices, providing an end-to-end solution from device provisioning and cloud communication to real-time AI voice conversation.

> **Platform Positioning**: Sentino IoT follows the **Tuya model** (your App / device calls the Sentino cloud directly; **there is no customer-owned backend** in between), not the AWS IoT model. This documentation covers only the "thing" layer; customizing Agent behavior (LLM/TTS orchestration, memory, workflows, custom tools) belongs to the Sentino Agent Platform — please contact the Sentino team for those docs.

---

## Related Open-Source Repositories

| Repository | Contents | Usage |
|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 firmware sample project (built on Agora RTSA Lite + ConvoAI) + Web BLE provisioning tool | Use it as a template to build / flash / run on real hardware; see `device/BUILD_GUIDE.md` inside the repo for details |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | Cross-platform IoT App template based on Flutter 3.x (Android / iOS / Windows / Web) | Take the white-label template, modify the UI and branding, then ship; protocol field definitions follow `ref-rest-api.md` in this repo |

---

## Reading Guide

Pick a reading path based on your role:

| Your Role | Recommended Reading Order |
|---|---|
| **Embedded Firmware Engineer** | [Architecture & Concepts](./architecture-en.md) -> [Quick Start (Device)](tutorials/quickstart-device-en.md) -> [Device Integration Guide](guides/guide-device-en.md) -> [AI Voice Conversation Integration Guide](guides/guide-ai-voice-en.md) -> [MQTT Protocol Reference](reference/ref-mqtt-en.md) / [BLE Protocol Reference](reference/ref-ble-en.md) -> Use [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) to run on real hardware |
| **App Developer (using the white-label Flutter template)** | [Architecture & Concepts](./architecture-en.md) -> Fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample) directly and follow its `doc/quick-start.md` to get it running -> Come back to this repo's [REST API Reference](reference/ref-rest-api-en.md) for field details |
| **App Developer (writing an App from scratch)** | [Architecture & Concepts](./architecture-en.md) -> [Quick Start (App)](tutorials/quickstart-app-en.md) -> [App Integration Guide](guides/guide-app-en.md) -> [REST API Reference](reference/ref-rest-api-en.md) |
| **Product / Project Manager** | [Architecture & Concepts](./architecture-en.md) -> [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) |
| **Partner Technical Lead** | [Architecture & Concepts](./architecture-en.md) -> [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) -> [Quick Start (Device)](tutorials/quickstart-device-en.md) |

---

## Document List

### Getting Started

| Document | Description | Status |
|---|---|---|
| [Architecture & Concepts](./architecture-en.md) | Overall architecture, product capabilities, core concepts, connectivity modes, AI voice conversation model, two product paths, glossary | Completed |
| [Quick Start — Device](tutorials/quickstart-device-en.md) | Get MQTT connected in 10 minutes — verifies signing and topics | Completed |
| [Quick Start — App](tutorials/quickstart-app-en.md) | Run the 6-step curl flow: login -> asset tree -> product -> device -> agent -> bind | Completed |

### Integration Guides

| Document | Description | Status |
|---|---|---|
| [Device Integration Guide](guides/guide-device-en.md) | MQTT authentication, device binding, property management, OTA, reconnect, state recovery | Completed |
| [App Integration Guide](guides/guide-app-en.md) | BLE provisioning (WiFi/4G), binding status query, device management | Completed |
| [AI Voice Conversation Integration Guide](guides/guide-ai-voice-en.md) | Conversation lifecycle, Agora RTC integration, interrupt / timeout / disconnect handling | Completed |

### Protocol & API References

| Document | Description | Status |
|---|---|---|
| [MQTT Protocol Reference](reference/ref-mqtt-en.md) | Connection parameters, topics, message formats, field definitions for all events / commands | Completed |
| [BLE Protocol Reference](reference/ref-ble-en.md) | Protocol layering, advertisement format, GATT services, V1 packet protocol, application-layer messages, status codes, provisioning state machine, platform adaptation | Completed |
| [REST API Reference](reference/ref-rest-api-en.md) | Authentication (UID + Password dual mode), provisioning, device management, Sentino Agent integration, conversation history | Completed |

### Product Solutions

| Document | Description | Status |
|---|---|---|
| [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) | User journey, Agent management, NFC cards, parental controls, security & compliance | Completed |

---

## Environment Information

| Item | Test Environment |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883` (MQTT 5.0, plaintext — for quickstart only; production should use TLS port `8883`) |
| REST API | `https://api-iot.sentino.jp/api` |
| Agora RTC | Parameters dynamically delivered by the cloud |

---

**Maintainer**: Sentino IoT Team
**Last Updated**: 2026-04-24
