# Sentino AI 语音对话 — 移远接入指南

**客户:** 移远  
**设备:** BK7258  
**日期:** 2026-03-21

---

## 接入参数

| 参数 | 值 |
| :--- | :--- |
| **MQTT 接入域名** | `mqtt-iot.sentino.jp` |
| **MQTT 端口** | `1883`（明文，未启用 TLS） |
| **MQTT 协议版本** | 5.0 |
| **产品 ID (pid)** | `sEF4ljjdH8mo` |
| **API KEY** | `<待分配>` |
| **Agent ID** | `<待分配>` |

### 测试角色简介

| 项目 | 内容 |
| :--- | :--- |
| **角色名** | `<待分配>` |
| **一句话介绍** | `<待分配>` |
| **角色描述** | `<待分配>` |

---

## 整体架构

```
┌────────────┐  BLE   ┌──────────────┐  4G    ┌──────────────────┐        ┌───────────┐
│  配网 App   │◄─────►│ BK7258 设备   │◄─────►│  Sentino 云端     │        │ Agora RTC │
│ (手机蓝牙)  │       │ Agora RTC SDK│        │  (MQTT Broker +   │◄──────►│  云端      │
│             │       │ (嵌入式)      │        │   AI Agent 管理)  │        │  音频传输  │
└────────────┘       └──────────────┘        └──────────────────┘        └───────────┘
```

| 角色 | 职责 |
| :--- | :--- |
| **配网 App（手机）** | 扫描设备配网条码，从 Sentino 服务器获取 userId / assetId，通过 BLE 蓝牙传给设备 |
| **BK7258 设备** | 通过 4G 连接 MQTT broker；接收 BLE 传入的 userId / assetId 完成绑定；通过 MQTT 获取 RTC 参数；内置 Agora RTC SDK 直连 Agora 进行语音通话 |
| **Sentino** | 提供设备三元组；管理 MQTT Broker；管理 AI Agent 生命周期；生成 RTC 鉴权参数 |

> 与 Wi-Fi 配网方案不同，移远方案中设备通过 4G 网络直连 MQTT broker 和 Agora 云端。手机 App 仅在首次绑定时通过 BLE 传递 userId / assetId，不参与语音通话，也不传递网络信息。
>
> BLE 蓝牙通信的底层协议（广播格式、GATT 服务、分包传输等）请参考《蓝牙配网技术说明文档》(`蓝牙配网技术说明文档.md`)。移远 4G 方案中，BLE 仅用于传递 `userId` / `assetId` 绑定信息，无需传递 WiFi SSID / 密码等网络配置。

---

## 设备三元组

Sentino 为每台设备预分配三元组信息，用于 MQTT 鉴权。三元组以文件形式批量交付给移远。

**三元组字段说明：**

| 字段 | 说明 | 示例 |
| :--- | :--- | :--- |
| `UUID` | 设备唯一标识 | `ct01wfjSNqGAqUUK` |
| `KEY` | 设备密钥，长度 32，用于 MQTT 签名 | `944e53cda6ac4491ad7d453e3d2934bb` |
| `MAC` | 设备 MAC 地址，12 位十六进制 | `444AD60EE353` |
| `Barcode` | 配网条码，印刷在设备上供 App 扫描 | `7337549374871` |

**三元组示例数据（前 3 条）：**

| UUID | KEY | MAC | Barcode |
| :--- | :--- | :--- | :--- |
| `ct01wfjSNqGAqUUK` | `944e53cda6ac4491ad7d453e3d2934bb` | `444AD60EE353` | `7337549374871` |
| `ct01PYcg8Ac9p8wn` | `bc50edcd2c3e4967885782aa1752138c` | `444AD60EE354` | `9115511194840` |
| `ct01eZRDs0KgKFzg` | `60586ceb82074dc1884206735dc29def` | `444AD60EE355` | `8494875779129` |

> 量产时需将 UUID、KEY、MAC 烧录到每台设备的 NVS 分区中，Barcode 印刷在设备外壳或包装上。

---

## MQTT 连接

### 连接参数

| 参数 | 值 / 格式 | 说明 |
| :--- | :--- | :--- |
| **Broker 地址** | `mqtt-iot.sentino.jp` | 测试环境 |
| **端口** | `1883` | 明文连接，未启用 TLS |
| **协议版本** | MQTT 5.0 | — |
| **Client ID** | `rlink_${uuid}_V2` | 固定以 `_V2` 结尾 |
| **Username** | `${uuid}\|signMethod=${signMethod},ts=${ts}` | — |
| **Password** | `hmacSha256("uuid=${uuid},ts=${ts}", KEY)` | KEY 取三元组中的密钥 |
| **Sign Method** | `hmacSha256`（推荐）或 `hmacSha1` | 根据硬件能力选择 |
| **Keep Alive** | 建议 60 秒 | — |
| **QoS** | 建议 QoS 1（发布时） | 确保消息至少送达一次 |

### 连接示例（使用真实三元组）

以第一条三元组为例，`ts` 取 `1742536800`（2025-03-21 10:00:00 UTC）：

- **UUID:** `ct01wfjSNqGAqUUK`
- **KEY:** `944e53cda6ac4491ad7d453e3d2934bb`

| 参数 | 值 |
| :--- | :--- |
| Broker | `mqtt-iot.sentino.jp:1883` |
| Client ID | `rlink_ct01wfjSNqGAqUUK_V2` |
| Username | `ct01wfjSNqGAqUUK\|signMethod=hmacSha256,ts=1742536800` |
| Password (hmacSha256) | `894972927a0a6d1a22a89883b9fe187a891a5b5dec4afa374034b703f2455bdd` |
| Password (hmacSha1) | `5c8e8e70dc8104e5b3fb8279a8d3e52180e49789` |

**签名计算过程：**

```
signMethod = hmacSha256
content    = "uuid=ct01wfjSNqGAqUUK,ts=1742536800"
key        = "944e53cda6ac4491ad7d453e3d2934bb"
password   = hmacSha256(content, key)
           = 894972927a0a6d1a22a89883b9fe187a891a5b5dec4afa374034b703f2455bdd
```

> 工程师可使用上述示例验证自己的签名实现是否正确。`ts` 为当前时间戳（秒），每次连接时需重新计算。

### MQTT Topic 定义

所有 Topic 中的变量：
- `${pid}` = `sEF4ljjdH8mo`（移远产品 ID，固定值）
- `${uuid}` = 设备三元组中的 UUID（每台设备不同）

| 用途 | Topic | 说明 |
| :--- | :--- | :--- |
| 设备上报 | `rlink/v2/sEF4ljjdH8mo/${uuid}/report` | 设备 → 云端 |
| 云端回复上报 | `rlink/v2/sEF4ljjdH8mo/${uuid}/report_response` | 云端 → 设备（需订阅） |
| 云端下发指令 | `rlink/v2/sEF4ljjdH8mo/${uuid}/issue` | 云端 → 设备（需订阅） |
| 设备回复指令 | `rlink/v2/sEF4ljjdH8mo/${uuid}/issue_response` | 设备 → 云端 |

**以第一台设备为例，实际 Topic：**

| 用途 | Topic |
| :--- | :--- |
| 设备上报 | `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report` |
| 云端回复 | `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response` |
| 云端下发 | `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue` |
| 设备回复 | `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue_response` |

> 设备连接 MQTT 后，需立即订阅 `report_response` 和 `issue` 两个 Topic。

### 消息通用格式

**上报消息（设备 → 云端）：**

```json
{
  "id": "消息唯一ID",
  "ts": 1742536800,
  "code": "事件编码",
  "data": {},
  "ack": 1
}
```

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | string | 消息唯一 ID，使用 UUID 生成，同一设备短时间内不可重复，否则消息可能被忽略 |
| `ts` | int | 当前时间戳（秒） |
| `code` | string | 事件编码 |
| `data` | object | 上报数据 |
| `ack` | int | 0: 不需要云端回复；1: 需要云端回复 |

**云端回复消息：**

```json
{
  "res": 0,
  "msg": "success",
  "id": "与上报消息ID一致",
  "ts": 1742536800,
  "code": "与上报事件编码一致",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `res` | int | 状态码，0 = 成功，非 0 = 失败 |
| `msg` | string | 结果描述 |

---

## 完整接入流程

### 一、首次绑定（仅首次使用）

#### 步骤 1：设备通过 4G 连接 MQTT

设备上电后，使用烧录的三元组信息按上述连接参数连接 MQTT broker，并订阅 `report_response` 和 `issue` Topic。

#### 步骤 2：App 扫码获取绑定信息

用户使用配网 App 扫描设备上的配网条码（Barcode）。App 将条码发送到 Sentino 服务器，服务器返回该设备对应的 `userId` 和 `assetId`。

> 此流程已封装在配网 App 中，移远无需关心服务端逻辑。

#### 步骤 3：App 通过 BLE 将绑定信息传给设备

App 通过 BLE 蓝牙连接设备，使用 Rlink BLE 协议将 `userId` 和 `assetId` 写入设备。

**BLE 广播与连接：**

- 设备进入配网模式后启动 BLE 广播，Service UUID 为 `0xA101`
- App 扫描到设备后建立 BLE 连接
- App 先发送 `device.information.get` 获取设备信息
- 然后发送 `thing.network.set` 传递绑定信息

**App 发送的 BLE 配网消息（`thing.network.set`）：**

4G 场景下只需传递 `bid`（即 assetId）和 `userId`，无需传递 WiFi 信息：

```json
{
  "type": "thing.network.set",
  "data": {
    "bid": "从Sentino服务器获取的assetId",
    "userId": "从Sentino服务器获取的userId"
  }
}
```

| 字段 | 说明 |
| :--- | :--- |
| `bid` | 资产 ID（assetId），由 App 从 Sentino 服务器获取 |
| `userId` | 用户 ID，由 App 从 Sentino 服务器获取 |

> 与 Wi-Fi 配网不同，4G 场景下 `sid`（WiFi SSID）、`pw`（WiFi 密码）、`mq`（MQTT 地址）等字段留空或不传。

**设备回复：**

```json
{
  "type": "thing.network.set.response",
  "code": 0,
  "ts": 1742536800
}
```

**BLE 传输协议要点：**

- 数据通过 Rlink BLE V1 分包协议传输，每包最大 128 字节（协议头 10 字节 + 有效数据 118 字节）
- 应用层消息为 JSON 格式
- 详细的 BLE 传输协议（分包格式、CRC 校验等）请参考《Rlink IoT SDK 蓝牙配网技术说明文档》

#### 步骤 4：设备通过 MQTT 完成绑定

设备从 BLE 收到 `userId` 和 `assetId`（即 `bid`）后，通过 MQTT 上报绑定请求。

**上报 Topic：** `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "bind",
  "data": {
    "userId": "从BLE获取的userId",
    "assetId": "从BLE获取的assetId(即bid)",
    "version": "1.0.0",
    "mcuVersion": "1.0.0",
    "cleanData": false
  },
  "ack": 1
}
```

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `userId` | string | 是 | 由 App 通过 BLE 传入 |
| `assetId` | string | 是 | 由 App 通过 BLE 传入（BLE 消息中的 `bid` 字段） |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `cleanData` | boolean | 否 | 是否清除数据，默认 false |

**云端回复 Topic：** `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response`

```json
{
  "res": 0,
  "msg": "success",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "bind"
}
```

`res` 为 0 表示绑定成功。

#### 步骤 5：设备信息上报（建议）

绑定成功后，建议上报设备信息。若未对接此协议，将无法收到设备因断电/断网导致云端未送达的重要遗留消息。

**上报 Topic：** `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report`

```json
{
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "ts": 1742536800,
  "code": "info",
  "data": {
    "bindStatus": 1,
    "version": "1.0.0",
    "mcuVersion": "1.0.0",
    "iccid": "898600MFSSYYGXXXXXXP"
  },
  "ack": 1
}
```

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| `bindStatus` | int | 是 | 设备绑定状态：0 未绑定；1 已绑定 |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `iccid` | string | 否 | 4G 流量卡 ICCID 号 |

> 4G 设备无需填写 `currentSsid`、`wifiList` 等 Wi-Fi 相关字段，填写 `iccid`（4G 流量卡号）即可。`config` 字段中的 WiFi 相关内容也无需填写。

**云端回复：**

```json
{
  "res": 0,
  "msg": "success",
  "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "ts": 1742536800,
  "code": "info",
  "data": {
    "bindStatus": 1,
    "isClearData": 0
  }
}
```

---

### 二、每次对话

#### 步骤 1：设备请求 AI 接入

用户触发对话时（如按键），设备通过 MQTT 上报 `agora_agent_device_access`。Sentino 云端内部自动完成 AI 会话创建，并将 RTC 连接信息返回给设备。

**上报 Topic：** `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report`

```json
{
  "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "ts": 1742536800,
  "code": "agora_agent_device_access",
  "ack": 1,
  "data": {}
}
```

**云端回复 Topic：** `rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response`

```json
{
  "res": 0,
  "msg": "success",
  "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "ts": 1742536800,
  "code": "agora_agent_device_access",
  "data": {
    "appId": "935b8af0c93640b09d614e48f1a8f75a",
    "rtcToken": "007eJxTYBA...",
    "channelName": "convo_stn_TgLI80UukY948M5v",
    "uid": 25532
  }
}
```

| 字段 | 类型 | 说明 | 设备端用途 |
| :--- | :--- | :--- | :--- |
| `appId` | string | Agora 应用 ID | `agora_rtc_init(appId, ...)` |
| `rtcToken` | string | RTC 通道令牌 | `agora_rtc_join_channel(..., rtcToken, ...)` |
| `channelName` | string | RTC 通道名称 | `agora_rtc_join_channel(channelName, ...)` |
| `uid` | int | RTC 成员 ID | `agora_rtc_join_channel(..., uid, ...)` |

#### 步骤 2：初始化 Agora SDK 并加入频道

```c
// 初始化
agora_rtc_init(appId, &event_handler);

// 音频配置
agora_rtc_config_t config = {0};
config.audio_codec = AUDIO_CODEC_OPUS;
config.pcm_sample_rate = 16000;    // 采样率 16kHz
config.pcm_channel_num = 1;        // 单声道

// 加入频道 → AI Agent 已在频道内等待
agora_rtc_join_channel(channelName, uid, rtcToken, &config);
```

**音频参数：**

| 参数 | 值 |
| :--- | :--- |
| 编码格式 | OPUS |
| 采样率 | 16000 Hz |
| 声道数 | 1（单声道） |
| PCM 位深 | 16 bit |
| 帧长 | 20 ms |

#### 步骤 3：处理音频回调

```c
static void on_join_channel_success(const char *channel, uint32_t uid, int elapsed) {
    // 成功加入频道，开始采集麦克风音频并发送
    start_audio_capture_and_send();
}

static void on_audio_data(const char *channel, uint32_t uid,
                          const void *data, size_t len) {
    // 收到 AI Agent 的音频数据，送入扬声器播放
    audio_play(data, len);
}
```

#### 步骤 4：结束对话

```c
agora_rtc_leave_channel();
```

设备调用 `leave_channel()` 后，Sentino 云端会自动检测到设备离开并清理 AI 会话，无需额外 MQTT 上报。

---

## 完整时序图

```
配网App(手机)    BK7258设备(4G)          Sentino 云端 (MQTT + API)       Agora
    │               │                          │                          │
    │  ── 首次绑定 ─────────────────────────────────────────────────────  │
    │               │                          │                          │
    │               │ 1.使用三元组连接MQTT       │                          │
    │               │─────────────────────────►│                          │
    │               │ 2.订阅 report_response    │                          │
    │               │   和 issue Topic          │                          │
    │               │─────────────────────────►│                          │
    │               │                          │                          │
    │ 3.扫描设备条码 │                          │                          │
    │  (Barcode:    │                          │                          │
    │   7337549374871)                         │                          │
    │──────────────►│                          │                          │
    │               │                          │                          │
    │ 4.App向Sentino服务器请求userId/assetId    │                          │
    │─────────────────────────────────────────►│                          │
    │ 5.返回userId/assetId                     │                          │
    │◄─────────────────────────────────────────│                          │
    │               │                          │                          │
    │ 6.BLE连接设备  │                          │                          │
    │  (Service UUID: 0xA101)                  │                          │
    │──────────────►│                          │                          │
    │ 7.BLE发送 thing.network.set              │                          │
    │  (bid, userId)│                          │                          │
    │──────────────►│                          │                          │
    │               │                          │                          │
    │               │ 8.MQTT上报 bind           │                          │
    │               │  (userId, assetId,        │                          │
    │               │   version)                │                          │
    │               │─────────────────────────►│                          │
    │               │ 9.绑定成功回复 (res=0)    │                          │
    │               │◄─────────────────────────│                          │
    │               │                          │                          │
    │               │ 10.MQTT上报 info          │                          │
    │               │  (bindStatus, version,    │                          │
    │               │   iccid)                  │                          │
    │               │─────────────────────────►│                          │
    │               │◄─────────────────────────│                          │
    │               │                          │                          │
    │  ── 每次对话 ─────────────────────────────────────────────────────  │
    │               │                          │                          │
    │               │ 11.MQTT上报               │                          │
    │               │  agora_agent_device_access│                          │
    │               │─────────────────────────►│                          │
    │               │                          │ 12.内部创建AI会话+Agent   │
    │               │                          │─────────────────────────►│
    │               │                          │                          │
    │               │ 13.回复 appId,            │                          │
    │               │   channelName,            │                          │
    │               │   rtcToken, uid           │                          │
    │               │◄─────────────────────────│                          │
    │               │                          │                          │
    │               │ 14.agora_rtc_init(appId)                            │
    │               │ 15.agora_rtc_join_channel(channelName, uid, rtcToken)│
    │               │────────────────────────────────────────────────────►│
    │               │                          │                          │
    │               │◄══════ 16. 用户与AI实时语音对话 (RTC) ════════════►│
    │               │                          │                          │
    │               │ 17.agora_rtc_leave_channel()                        │
    │               │────────────────────────────────────────────────────►│
    │               │                          │ 18.检测离开,清理会话      │
    │               │                          │◄─────────────────────────│
    │               │                          │                          │
```

---

## 各方职责一览

| 角色 | 做什么 | 不做什么 |
| :--- | :--- | :--- |
| **配网 App** | 扫描设备条码；从 Sentino 获取 userId/assetId；通过 BLE 写入设备 | 不参与语音通话，不传递网络信息 |
| **BK7258 设备** | 使用三元组连接 MQTT；BLE 接收 userId/assetId 完成绑定；通过 MQTT 获取 RTC 参数；初始化 Agora SDK、加入频道、采集/播放音频 | 不调 Sentino HTTP API，不生成 Token |
| **Sentino** | 提供三元组和配网条码；提供配网 App；管理 MQTT Broker；管理 AI Agent 生命周期；生成 RTC 参数；内部处理会话启停 | — |

---

## Agora RTC SDK

Sentino 提供适配 BK7258 平台的 Agora RTC SDK 嵌入式版（C API），随接入文档一起交付给移远。

SDK 交付物包含：
- 静态库 / 动态库（适配 BK7258 RTOS 平台）
- 头文件
- 集成示例代码

---

## 断线重连策略

4G 网络不稳定是常态，设备需要实现断线重连机制。

### MQTT 断连重连

| 策略 | 说明 |
| :--- | :--- |
| 重连间隔 | 指数退避：1s → 2s → 4s → 8s → ... 最大 60s |
| 重连认证 | 使用相同三元组重新计算签名（更新 `ts` 为当前时间戳） |
| 重连后操作 | 重新订阅 `report_response` 和 `issue` Topic |

### RTC 断连重连

| 策略 | 说明 |
| :--- | :--- |
| 重连方式 | 使用旧的 RTC 参数（appId / channelName / rtcToken / uid）直接重连 |
| Token 过期 | 若旧参数重连失败，重新上报 `agora_agent_device_access` 获取新参数 |

---

## 常见错误与排查

| 场景 | 现象 | 排查方向 |
| :--- | :--- | :--- |
| MQTT 连接被拒绝 | 无法连接 broker | 检查三元组是否正确；签名算法是否匹配；`ts` 时间戳是否合理（不能偏差太大） |
| 绑定失败 | `bind` 回复 `res` 非 0 | 检查 `userId` 和 `assetId` 是否正确（来自 BLE） |
| AI 接入失败 | `agora_agent_device_access` 回复 `res` 非 0 | 检查设备是否已完成绑定；Agent 是否已配置 |
| RTC 加入频道失败 | `join_channel` 返回错误 | 检查 4 个 RTC 参数是否完整；Token 是否过期 |
| 无声音 | RTC 连接成功但无音频 | 检查音频参数（16kHz/单声道/16bit/OPUS）；检查麦克风和扬声器硬件 |
| 消息被忽略 | 上报后无回复 | 检查消息 `id` 是否重复（需使用 UUID 保证唯一）；检查 `ack` 是否设为 1 |

---

## 注意事项

1. **三元组保密**：设备三元组中的 `KEY` 是设备身份凭证，需妥善保管，禁止明文传输或日志打印。
2. **三元组烧录**：量产时需将 UUID、KEY、MAC 烧录到每台设备的 NVS 分区，Barcode 印刷在设备外壳或包装上。
3. **消息 ID**：每条 MQTT 上报消息的 `id` 字段使用 UUID 生成，同一设备短时间内不可重复，否则消息可能被云端忽略。
4. **RTC Token 有时效**：获取到 RTC 连接信息后应尽快加入频道。Token 过期后需重新请求 `agora_agent_device_access` 获取新参数。
5. **会话超时**：会话启动后如长时间无人加入或无语音活动，Sentino 会自动关闭会话。
6. **设备信息上报**：建议对接 `info` 协议。4G 设备无需填写 Wi-Fi 相关字段，填写 `iccid` 即可。
7. **BLE 配网协议**：BLE 传输协议细节（分包格式、CRC 校验、广播格式等）请参考《Rlink IoT SDK 蓝牙配网技术说明文档》。
8. **MQTT 协议详情**：完整的 MQTT 协议定义（包括设备重置、获取云端时间、属性上报、OTA 升级等）请参考 `MQTT_3.0_AI_协议.md` 文档。
9. **生产环境**：本文档所有参数均为测试环境，生产环境参数另行提供。
