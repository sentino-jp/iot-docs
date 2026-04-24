# Sentino IoT Developer Documentation Hub

<p>
  <strong>English</strong> &nbsp;·&nbsp;
  <a href="./README.md">简体中文</a>
</p>

Sentino IoT is an IoT platform purpose-built for AI voice-interactive devices. Provisioning → MQTT → real-time AI voice — your App / device calls the Sentino cloud directly; no customer-owned backend required.

> Customizing Agent behavior (LLM/TTS orchestration, memory, workflows, custom tools) belongs to the Sentino Agent Platform — please contact the Sentino team for those docs.

---

## Open-source repositories

| Repository | Purpose | Activity |
|---|---|---|
| [sentino-jp/sentino-iot-sample](https://github.com/sentino-jp/sentino-iot-sample) | BK7258 firmware sample (Agora RTSA + ConvoAI) + Web BLE provisioning tool | ![](https://img.shields.io/github/last-commit/sentino-jp/sentino-iot-sample) |
| [sentino-jp/sentino-app-sample](https://github.com/sentino-jp/sentino-app-sample) | Flutter 3.x cross-platform App template (Android / iOS / Windows / Web) | ![](https://img.shields.io/github/last-commit/sentino-jp/sentino-app-sample) |

---

## Documentation

Start with [Architecture & Concepts](./architecture-en.md) (overall architecture / product capabilities / communication protocols / connectivity modes / glossary), then dive in by task:

**Writing embedded firmware**
- Onboarding: [Device quickstart](tutorials/quickstart-device-en.md) (10-min MQTT verification) → [Device integration](guides/guide-device-en.md) → [AI voice](guides/guide-ai-voice-en.md)
- Protocols: [MQTT reference](reference/ref-mqtt-en.md) · [BLE reference](reference/ref-ble-en.md)
- Real hardware: fork [`sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) and follow `device/BUILD_GUIDE.md`

**Building an App**
- White-label template: fork [`sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample) → rebrand the UI → check field details in [REST API reference](reference/ref-rest-api-en.md)
- From scratch: [App quickstart](tutorials/quickstart-app-en.md) → [App integration](guides/guide-app-en.md) → [REST API reference](reference/ref-rest-api-en.md)

**Evaluating a solution**
- [AI Toy Integration Solution](solutions/solution-ai-toy-en.md) (user journey, NFC, parental controls, security & compliance)

---

## Environment

| | |
|---|---|
| MQTT Broker | `mqtt-iot.sentino.jp:1883` (MQTT 5.0; production uses TLS port `8883`) |
| REST API | `https://api-iot.sentino.jp/api` |
| Agora RTC | Parameters delivered by the cloud at runtime |

Test triplets (UUID / KEY / MAC) and App credentials (`app_id` / `channel_identifier` / `package_name`) must be issued by the Sentino team for your application.

---

## Feedback

- Documentation errors / field mismatches / broken links: [sentino-jp/iot-docs/issues](https://github.com/sentino-jp/iot-docs/issues)
- Platform onboarding / custom integration: contact the Sentino team
