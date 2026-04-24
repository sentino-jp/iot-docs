# Sentino IoT Developer Documentation Hub

<p>
  <strong>English</strong> &nbsp;·&nbsp;
  <a href="./README.md">简体中文</a>
</p>

Welcome to the Sentino IoT developer documentation. Sentino IoT is an IoT platform purpose-built for AI voice-interaction devices, providing an end-to-end solution from device provisioning and cloud communication to real-time AI voice conversation.

> **Platform Positioning**: Sentino IoT is an end-to-end IoT solution — your App / device calls the Sentino cloud directly; **no customer-owned backend** is required. This documentation covers only the "thing" layer; customizing Agent behavior (LLM/TTS orchestration, memory, workflows, custom tools) belongs to the Sentino Agent Platform — please contact the Sentino team for those docs.

---

## Related Open-Source Repositories

| Repository | Contents | Activity | Usage |
|---|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 firmware sample project (built on Agora RTSA Lite + ConvoAI) + Web BLE provisioning tool | ![last commit](https://img.shields.io/github/last-commit/sentino-jp/sentino-iot-sample) | Use it as a template to build / flash / run on real hardware; see `device/BUILD_GUIDE.md` inside the repo for details |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | Cross-platform IoT App template based on Flutter 3.x (Android / iOS / Windows / Web) | ![last commit](https://img.shields.io/github/last-commit/sentino-jp/sentino-app-sample) | Take the white-label template, modify the UI and branding, then ship; protocol field definitions follow `ref-rest-api.md` in this repo |

---

## Reading Guide

Pick a reading path based on your role:

| Your Role | Recommended Reading Order |
|---|---|
| **Embedded Firmware Engineer** (basics) | [Architecture & Concepts](./architecture-en.md) -> [Quick Start (Device)](tutorials/quickstart-device-en.md) -> [Device Integration Guide](guides/guide-device-en.md) -> [AI Voice Conversation Integration Guide](guides/guide-ai-voice-en.md) |
| **Embedded Firmware Engineer** (reference + hardware) | [MQTT Protocol Reference](reference/ref-mqtt-en.md) / [BLE Protocol Reference](reference/ref-ble-en.md) -> fork [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) to build & flash |
| **App Developer (using the white-label Flutter template)** | [Architecture & Concepts](./architecture-en.md) -> Fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample) directly and follow its `doc/quick-start.md` to get it running -> Come back to this repo's [REST API Reference](reference/ref-rest-api-en.md) for field details |
| **App Developer (writing an App from scratch)** | [Architecture & Concepts](./architecture-en.md) -> [Quick Start (App)](tutorials/quickstart-app-en.md) -> [App Integration Guide](guides/guide-app-en.md) -> [REST API Reference](reference/ref-rest-api-en.md) |
| **Product / Project Manager** | [Architecture & Concepts](./architecture-en.md) -> [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) |
| **Partner Technical Lead** | [Architecture & Concepts](./architecture-en.md) -> [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) -> Browse the [Related Open-Source Repositories](#related-open-source-repositories) for sample depth |

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
| Test triplet (UUID / KEY / MAC) | Contact the Sentino team to provision one for your application |
| Test App credentials (`app_id` / `channel_identifier` / `package_name`) | Same as above; example values in this documentation are for reference — use values issued by Sentino in production |

---

## Feedback & Contact

| Channel | Use case |
|---|---|
| GitHub issue | Documentation errors, field inconsistencies, broken links: [sentino-jp/iot-docs/issues](https://github.com/sentino-jp/iot-docs/issues) |
| Business contact | Platform onboarding, pricing, custom integration: contact the Sentino team |
| Sentino Agent Platform | Custom LLM/TTS/workflow/memory capabilities: see "Platform Positioning" at the top |

---

**Maintainer**: Sentino IoT Team
**Last Updated**: 2026-04-24
