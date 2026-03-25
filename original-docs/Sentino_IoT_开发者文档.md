# Sentino IoT 开发者文档

**版本**: v1.0
**更新日期**: 2026-03-25

---

## 目录

1. [平台简介](#1-平台简介)
2. [整体架构](#2-整体架构)
3. [快速开始](#3-快速开始)
4. [设备三元组](#4-设备三元组)
5. [MQTT 接入](#5-mqtt-接入)
   - 5.1 [连接鉴权](#51-连接鉴权)
   - 5.2 [Topic 定义](#52-topic-定义)
   - 5.3 [消息格式](#53-消息格式)
6. [BLE 蓝牙配网](#6-ble-蓝牙配网)
   - 6.1 [广播协议](#61-广播协议)
   - 6.2 [BLE 传输协议 (V1)](#62-ble-传输协议-v1)
   - 6.3 [配网流程 — WiFi 设备](#63-配网流程--wifi-设备)
   - 6.4 [配网流程 — 4G 设备](#64-配网流程--4g-设备)
   - 6.5 [配网状态码](#65-配网状态码)
7. [设备生命周期协议](#7-设备生命周期协议)
   - 7.1 [设备绑定 (bind)](#71-设备绑定-bind)
   - 7.2 [设备信息上报 (info)](#72-设备信息上报-info)
   - 7.3 [设备重置 (reset)](#73-设备重置-reset)
   - 7.4 [获取云端时间 (time)](#74-获取云端时间-time)
   - 7.5 [获取物模型 (model)](#75-获取物模型-model)
   - 7.6 [属性上报 (property_report)](#76-属性上报-property_report)
   - 7.7 [属性设置 (property_set)](#77-属性设置-property_set)
   - 7.8 [OTA 固件升级](#78-ota-固件升级)
   - 7.9 [在线检测 (ping)](#79-在线检测-ping)
8. [AI 语音对话](#8-ai-语音对话)
   - 8.1 [接入流程](#81-接入流程)
   - 8.2 [普通设备接入 AI](#82-普通设备接入-ai)
   - 8.3 [NFC 设备接入 AI](#83-nfc-设备接入-ai)
   - 8.4 [Agora RTC 集成](#84-agora-rtc-集成)
9. [REST API 参考](#9-rest-api-参考)
   - 9.1 [认证说明](#91-认证说明)
   - 9.2 [用户登录](#92-用户登录)
   - 9.3 [获取产品信息](#93-获取产品信息)
   - 9.4 [获取资产树](#94-获取资产树)
   - 9.5 [获取设备信息](#95-获取设备信息)
   - 9.6 [配网数据加密](#96-配网数据加密)
   - 9.7 [绑定状态查询](#97-绑定状态查询)
   - 9.8 [获取设备列表](#98-获取设备列表)
   - 9.9 [设备解绑](#99-设备解绑)
   - 9.10 [OTA 升级检查](#910-ota-升级检查)
   - 9.11 [智能体管理](#911-智能体管理)
10. [断线重连策略](#10-断线重连策略)
11. [错误排查](#11-错误排查)
12. [附录](#12-附录)

---

## 1. 平台简介

Sentino IoT 是一个面向 AI 语音交互设备的物联网平台，为硬件厂商提供从设备配网、云端通信到 AI 语音对话的完整解决方案。

**核心能力：**

- 基于 MQTT 5.0 的设备接入与管理
- BLE 蓝牙配网（支持 WiFi 双模配网与 4G 设备绑定）
- 集成 Agora RTC 的实时 AI 语音对话
- 智能体 (Agent) 管理与设备绑定
- OTA 固件升级

---

## 2. 整体架构

```mermaid
graph LR
    App["配网 App<br/>(手机蓝牙)"]
    Device["IoT 设备"]
    Cloud["Sentino 云端<br/>(MQTT Broker +<br/>AI Agent 管理)"]
    Agora["Agora RTC<br/>云端"]

    App <-->|BLE| Device
    Device <-->|WiFi / 4G| Cloud
    Cloud <-->|音频传输| Agora
```

| 角色 | 职责 |
|:---|:---|
| **配网 App** | 蓝牙扫描设备 → 从 Sentino 获取绑定信息 → 通过 BLE 写入设备 |
| **IoT 设备** | 连接 MQTT Broker；接收 BLE 配网信息；通过 MQTT 获取 RTC 参数；集成 Agora RTC SDK 进行语音通话 |
| **Sentino 云端** | 提供设备三元组；管理 MQTT Broker；管理 AI Agent 生命周期；生成 RTC 鉴权参数 |
| **Agora RTC** | 提供低延迟实时音频传输通道 |

**两种联网模式：**

| 模式 | 说明 | BLE 配网传递的内容 |
|:---|:---|:---|
| **WiFi 模式** | 设备通过 WiFi 联网 | WiFi SSID/密码 + userId + assetId + MQTT 地址 |
| **4G 模式** | 设备通过蜂窝网络直连 | 仅 userId + assetId（无需 WiFi 信息） |

---

## 3. 快速开始

以下是设备从出厂到完成首次 AI 语音对话的核心步骤：

```mermaid
graph LR
    A["1. 烧录三元组<br/>(量产阶段)"] --> B["2. 连接 MQTT<br/>(设备上电自动)"]
    B --> C["3. BLE 配网绑定<br/>(用户首次使用)"]
    C --> D["4. 请求 AI 接入<br/>(用户触发对话)"]
    D --> E["5. 加入 RTC<br/>(实时语音)"]
```

**前置准备：**

1. 从 Sentino 获取**产品 ID (pid)** 和**设备三元组**文件（UUID / KEY / MAC / Barcode）
2. 将三元组信息烧录到每台设备的 NVS 分区
3. 将 Barcode 印刷在设备外壳或包装上，供 App 扫码

**最小可用流程：**

1. 设备上电 → 使用三元组连接 MQTT → 订阅 `report_response` 和 `issue` Topic
2. 用户扫码 → App 从 Sentino 获取 userId / assetId → BLE 传给设备
3. 设备通过 MQTT 上报 `bind`（绑定）→ 云端回复成功
4. 用户触发对话 → 设备上报 `agora_agent_device_access` → 云端返回 RTC 参数
5. 设备初始化 Agora SDK → 加入频道 → 开始语音对话

---

## 4. 设备三元组

Sentino 为每台设备预分配唯一的三元组信息，用于 MQTT 鉴权和设备标识。三元组以 CSV 文件批量交付。

| 字段 | 说明 | 示例 |
|:---|:---|:---|
| `UUID` | 设备唯一标识（前 4 位为类型标识，后 12 位随机） | `ct01wfjSNqGAqUUK` |
| `KEY` | 设备密钥，32 位字符串，用于 MQTT 签名 | `944e53cda6ac4491ad7d453e3d2934bb` |
| `MAC` | 设备 MAC 地址，12 位十六进制 | `444AD60EE353` |
| `Barcode` | 配网条码，印刷在设备上供 App 扫描 | `7337549374871` |

> **安全提醒**：`KEY` 是设备身份凭证，必须妥善保管。禁止明文传输、日志打印或硬编码到源代码中。量产时需将 UUID、KEY、MAC 烧录到设备 NVS 分区。

---

## 5. MQTT 接入

> **TL;DR**: 设备使用三元组（UUID + KEY）通过 HMAC 签名连接 MQTT 5.0 Broker，通过 4 个 Topic 实现双向通信。

### 5.1 连接鉴权

| 参数 | 值 / 格式 | 说明 |
|:---|:---|:---|
| **Broker 地址** | `mqtt-iot.sentino.jp` | 测试环境 |
| **端口** | `1883` | 明文连接 |
| **协议版本** | MQTT 5.0 | — |
| **Client ID** | `rlink_${uuid}_V2` | 固定以 `_V2` 结尾 |
| **Username** | `${uuid}\|signMethod=${signMethod},ts=${ts}` | `\|` 为分隔符 |
| **Password** | `hmacSha256("uuid=${uuid},ts=${ts}", KEY)` | KEY 为三元组中的设备密钥 |
| **Sign Method** | `hmacSha256`（推荐）或 `hmacSha1` | 根据硬件能力选择 |
| **Keep Alive** | 60 秒（建议） | — |
| **QoS** | QoS 1（发布时建议） | 确保消息至少送达一次 |

**签名计算示例：**

以三元组 `UUID=ct01wfjSNqGAqUUK`、`KEY=944e53cda6ac4491ad7d453e3d2934bb`、`ts=1742536800` 为例：

```
Client ID  = "rlink_ct01wfjSNqGAqUUK_V2"
Username   = "ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=1742536800"
Password   = hmacSha256("uuid=ct01wfjSNqGAqUUK,ts=1742536800", KEY)
           = "894972927a0a6d1a22a89883b9fe187a891a5b5dec4afa374034b703f2455bdd"
```

> `ts` 为当前 UNIX 时间戳（秒），每次连接时需重新计算。可用上述示例验证签名实现是否正确。

**签名计算代码参考：**

<details>
<summary>Python</summary>

```python
import hmac, hashlib, time

uuid = "ct01wfjSNqGAqUUK"
key  = "944e53cda6ac4491ad7d453e3d2934bb"
ts   = str(int(time.time()))

content  = f"uuid={uuid},ts={ts}"
password = hmac.new(key.encode(), content.encode(), hashlib.sha256).hexdigest()

client_id = f"rlink_{uuid}_V2"
username  = f"{uuid}|signMethod=hmacSha256,ts={ts}"

print(f"Client ID: {client_id}")
print(f"Username:  {username}")
print(f"Password:  {password}")
```

</details>

<details>
<summary>Node.js</summary>

```javascript
const crypto = require('crypto');

const uuid = 'ct01wfjSNqGAqUUK';
const key  = '944e53cda6ac4491ad7d453e3d2934bb';
const ts   = Math.floor(Date.now() / 1000).toString();

const content  = `uuid=${uuid},ts=${ts}`;
const password = crypto.createHmac('sha256', key).update(content).digest('hex');

const clientId = `rlink_${uuid}_V2`;
const username = `${uuid}|signMethod=hmacSha256,ts=${ts}`;

console.log(`Client ID: ${clientId}`);
console.log(`Username:  ${username}`);
console.log(`Password:  ${password}`);
```

</details>

<details>
<summary>C (嵌入式)</summary>

```c
// content = "uuid=ct01wfjSNqGAqUUK,ts=1742536800"
// key     = "944e53cda6ac4491ad7d453e3d2934bb"
// password = hmac_sha256(key, key_len, content, content_len, output)

char content[128];
snprintf(content, sizeof(content), "uuid=%s,ts=%ld", uuid, (long)ts);

char client_id[64];
snprintf(client_id, sizeof(client_id), "rlink_%s_V2", uuid);

char username[128];
snprintf(username, sizeof(username), "%s|signMethod=hmacSha256,ts=%ld", uuid, (long)ts);

// password 使用平台提供的 HMAC-SHA256 函数计算
uint8_t hmac_out[32];
hmac_sha256((uint8_t*)key, strlen(key), (uint8_t*)content, strlen(content), hmac_out);
// 将 hmac_out 转为 hex 字符串即为 password
```

</details>

### 5.2 Topic 定义

所有 Topic 使用以下变量：
- `${pid}` — 产品 ID（由 Sentino 分配，同一产品下所有设备共享）
- `${uuid}` — 设备 UUID（每台设备不同，取自三元组）

| 方向 | Topic | 说明 |
|:---|:---|:---|
| 设备 → 云端 | `rlink/v2/${pid}/${uuid}/report` | 设备事件上报 |
| 云端 → 设备 | `rlink/v2/${pid}/${uuid}/report_response` | 云端回复上报（需订阅） |
| 云端 → 设备 | `rlink/v2/${pid}/${uuid}/issue` | 云端下发指令（需订阅） |
| 设备 → 云端 | `rlink/v2/${pid}/${uuid}/issue_response` | 设备回复指令 |

**实际示例**（假设 pid=`sEF4ljjdH8mo`，uuid=`ct01wfjSNqGAqUUK`）：

```
发布（上报）:  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report
订阅（回复）:  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response
订阅（下发）:  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue
发布（回复）:  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue_response
```

> 设备连接 MQTT 后，需**立即订阅** `report_response` 和 `issue` 两个 Topic。

### 5.3 消息格式

**上报消息（设备 → 云端）：**

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "事件编码",
  "data": {},
  "ack": 1
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `id` | string | 消息唯一 ID，建议使用 UUID v4 生成。同一设备短时间内不可重复，否则消息会被云端忽略 |
| `ts` | int | 当前 UNIX 时间戳（秒） |
| `code` | string | 事件编码，详见各协议章节 |
| `data` | object | 上报数据 |
| `ack` | int | `0` = 不需要云端回复；`1` = 需要云端回复 |

**云端回复（云端 → 设备，通过 `report_response`）：**

```json
{
  "res": 0,
  "msg": "success",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "事件编码",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `res` | int | 状态码。`0` = 成功，非 `0` = 失败 |
| `msg` | string | 结果描述 |

**云端下发指令（云端 → 设备，通过 `issue`）：**

```json
{
  "id": "消息ID",
  "ts": 1742536800,
  "code": "指令编码",
  "data": {}
}
```

**设备回复指令（设备 → 云端，通过 `issue_response`）：**

```json
{
  "res": 0,
  "msg": "success",
  "id": "消息ID",
  "ts": 1742536800,
  "code": "指令编码",
  "data": {}
}
```

---

## 6. BLE 蓝牙配网

> **TL;DR**: 设备通过 BLE 广播（Service UUID `0xA101`）被 App 发现。App 通过 BLE 向设备写入配网信息（WiFi 凭证或绑定 ID），设备收到后自动联网并完成绑定。

### 6.1 广播协议

设备进入配网模式后启动 BLE 广播，App 通过扫描发现设备。

**广播数据结构：**

| 段 | 类型 | 内容 |
|:---|:---|:---|
| FLAGS | `0x01` | `0x06` |
| Service UUID | `0x02` | `0xA101`（Sentino 配网服务标识） |
| Service Data | `0x16` | UUID `0xA101` + 标志位 + PID 字符串 |

**扫描应答数据结构：**

| 段 | 类型 | 内容 |
|:---|:---|:---|
| 完整名称 | `0x09` | `"RY"` |
| 厂商数据 | `0xFF` | 厂商 ID + FLAG + 协议版本 + 加密方式 + 通信能力 + UUID/MAC |

**FLAG 标志位（1 字节）：**

| 位 | 掩码 | 说明 |
|:---|:---|:---|
| bit7 | `0x80` | 配网状态：`0` = 正在配网，`1` = 不在配网 |
| bit6 | `0x40` | 绑定状态：`0` = 未绑定，`1` = 已绑定 |
| bit5 | `0x20` | WiFi 联网：`0` = 离线，`1` = 在线 |
| bit4 | `0x10` | 通用固件：`0` = 否，`1` = 是 |
| bit1 | `0x02` | 聚合协议：`0` = 否，`1` = 是 |
| bit0 | `0x01` | 请求连接：`0` = 否，`1` = 是 |

**协议版本（1 字节）：**

| 值 | 含义 |
|:---|:---|
| `0x03` | 双模配网（BLE + WiFi） |
| `0x04` | 单蓝牙配网 |
| `0x10` | 双模配网 + 动态 MTU |
| `0x11` | 单蓝牙配网 + 动态 MTU |

**加密方式（1 字节）：**

| 值 | 含义 |
|:---|:---|
| `0x00` | 基于 auth key + device id 加密 |
| `0x01` | 基于 ECB 算法加密 |
| `0x02` | 不加密（透传） |

**通信能力（2 字节，大端序）：**

| 位 | 含义 |
|:---|:---|
| bit0 | 是否通过 BLE 注册绑定 |
| bit1 | 是否支持 MESH |
| bit2 | WiFi 2.4G 能力 |
| bit3 | WiFi 5G 能力 |
| bit4 | Zigbee 能力 |
| bit5 | NB-IoT 能力 |

### 6.2 BLE 传输协议 (V1)

应用层数据通过 V1 分包协议在 BLE 上传输。每个 BLE 数据包最大 **128 字节**，其中协议头 10 字节，有效数据最大 118 字节。

**帧格式：**

```
+------+------+------+------+------+------+------+------+------+------+------+
| HEAD | TYPE |   SN (2B)   | TOTAL (2B)  |  LEN (2B)   |C_LEN | DATA | CRC  |
|  1B  |  1B  |  1B  |  1B  |  1B  |  1B  |  1B  |  1B  |  1B  | N B  |  1B  |
+------+------+------+------+------+------+------+------+------+------+------+
```

| 字段 | 大小 | 说明 |
|:---|:---|:---|
| HEAD | 1B | 固定 `0xFF` |
| TYPE | 1B | 数据类型，固定 `0x01` |
| SN | 2B | 当前包序号（大端序），从 0 开始 |
| TOTAL | 2B | 总包数（大端序） |
| LEN | 2B | 有效数据总长度（大端序） |
| C_LEN | 1B | 当前包有效数据长度 |
| DATA | NB | 当前包有效数据 |
| CRC | 1B | 校验和：从 TYPE 到 DATA 末尾的逐字节累加和 |

**分包发送流程：**

1. 计算总包数：`total = ceil(data_len / 118)`
2. 为每个分包添加 10 字节协议头
3. 按 128 字节为单位通过 BLE Notify 发送
4. 每包之间间隔 **20ms**

**分包接收流程：**

1. 验证包头 (`0xFF`)
2. 检查包序号连续性
3. 验证 CRC 校验和
4. 首包时分配接收缓冲区
5. 拷贝有效数据
6. 末包时验证总长度，回调应用层

**应用层消息格式（JSON）：**

所有应用层消息均为 JSON 格式，通过 V1 分包协议承载：

```json
{
  "type": "消息类型",
  "ts": 1234567890,
  "msgId": "消息ID",
  "code": 0,
  "data": {}
}
```

**BLE 消息类型一览：**

| 方向 | type | 说明 |
|:---|:---|:---|
| App → 设备 | `device.information.get` | 获取设备信息 |
| 设备 → App | `device.information.get.response` | 设备信息响应 |
| App → 设备 | `thing.network.getwifis` | 获取 WiFi 列表 |
| 设备 → App | `thing.network.getwifis.response` | WiFi 列表响应 |
| App → 设备 | `thing.network.set` | 设置网络配置（配网） |
| 设备 → App | `thing.network.set.response` | 配网响应 |
| App → 设备 | `thing.property.set` | 设置设备属性 |
| App → 设备 | `thing.property.get` | 获取设备属性 |
| 设备 → App | `thing.property.report` | 属性上报 |
| App → 设备 | `ota.upgrade.initiate` | OTA 升级初始化 |
| App → 设备 | `device.data.clear` | 清除设备数据 |

### 6.3 配网流程 — WiFi 设备

适用于通过 WiFi 联网的设备。

**设备配网状态机：**

```mermaid
stateDiagram-v2
    [*] --> 待机: 上电
    待机 --> BLE配网模式: 用户触发配网
    BLE配网模式 --> 尝试连接: 收到配网信息
    尝试连接 --> STA模式: WiFi 连接成功
    尝试连接 --> 待机: 超时(60s) / 失败
    STA模式 --> [*]: 正常运行
```

**配网时序：**

```mermaid
sequenceDiagram
    participant App as 配网 App
    participant Device as IoT 设备
    participant Cloud as Sentino 云端
    participant WiFi as WiFi / MQTT

    Device-->>App: BLE 广播 (Service UUID: 0xA101)
    App->>App: 解析广播包获取 PID

    App->>Cloud: 请求产品信息 (PID)
    Cloud-->>App: 返回产品名称 / 图片

    App->>Device: BLE 连接
    App->>Device: device.information.get
    Device-->>App: 设备信息响应

    App->>Device: thing.network.getwifis
    Device-->>App: WiFi 列表响应

    App->>Cloud: 配网数据加密请求
    Cloud-->>App: 返回加密后的配网数据

    App->>Device: thing.network.set (sid, pw, bid, userId, mq...)
    Device-->>App: 配网响应 (code: 0)

    Device->>WiFi: 连接 WiFi
    WiFi-->>Device: 连接成功
    Device->>WiFi: 连接 MQTT
    Device->>Cloud: MQTT 上报 bind

    loop 每 2 秒轮询
        App->>Cloud: 查询绑定状态
    end
    Cloud-->>App: 绑定成功
```

**关键步骤说明：**

**步骤 1 — App 扫描发现设备**

App 扫描 BLE 设备，过滤蓝牙名称为 `RY` 或 Service UUID 为 `0xA101` 的设备。从广播包中解析出 PID。

**步骤 2 — 获取设备信息**

App 通过 BLE 发送 `device.information.get`：

```json
{ "type": "device.information.get", "ts": 1234567890 }
```

设备响应：

```json
{
  "type": "device.information.get.response",
  "ts": 1234567890,
  "data": {
    "version": "1.0.0",
    "pid": "产品ID",
    "bind": false,
    "wifi_mac": "aa:bb:cc:dd:ee:ff",
    "ble_mac": "11:22:33:44:55:66"
  }
}
```

**步骤 3 — 获取 WiFi 列表**

```json
{ "type": "thing.network.getwifis", "scan": 1 }
```

设备响应：

```json
{
  "type": "thing.network.getwifis.response",
  "code": 0,
  "data": {
    "wifis": [
      { "ssid": "MyWiFi", "rssi": -45, "security": true }
    ]
  }
}
```

**步骤 4 — 发送配网信息**

App 先通过 REST API 加密配网数据（参见 [9.6 配网数据加密](#96-配网数据加密)），然后通过 BLE 发送：

```json
{
  "type": "thing.network.set",
  "data": {
    "sid": "WiFi名称",
    "pw": "WiFi密码",
    "bid": "资产ID(assetId)",
    "userId": "用户ID",
    "mq": "MQTT地址",
    "port": 1883,
    "country": "CN",
    "tz": "Asia/Shanghai",
    "force_bind": false
  }
}
```

设备响应：

```json
{
  "type": "thing.network.set.response",
  "code": 0,
  "ts": 1234567890
}
```

**步骤 5 — 设备自动连接 WiFi → MQTT → 绑定**

设备收到配网信息后自动执行：连接 WiFi → 获取 IP → 连接 MQTT → 上报 `bind`。
配网超时时间为 **60 秒**，超时后设备自动重启。

**步骤 6 — App 轮询绑定结果**

App 每 2 秒调用 REST API 查询绑定状态（参见 [9.7 绑定状态查询](#97-绑定状态查询)），或监听 MQTT `bind_result` 消息。

### 6.4 配网流程 — 4G 设备

适用于通过蜂窝网络（4G/LTE）联网的设备。与 WiFi 模式的主要区别：

- 设备上电后通过 4G **直连 MQTT**，无需 WiFi 信息
- BLE 仅传递 `userId` 和 `assetId`（即 `bid`），不传递 WiFi SSID/密码

```mermaid
sequenceDiagram
    participant App as 配网 App
    participant Device as IoT 设备 (4G)
    participant Cloud as Sentino 云端

    Device->>Cloud: 1. 使用三元组连接 MQTT
    Device->>Cloud: 2. 订阅 report_response + issue

    App->>App: 3. 扫描设备条码 (Barcode)
    App->>Cloud: 4. 请求 userId / assetId
    Cloud-->>App: 5. 返回 userId / assetId

    App->>Device: 6. BLE 连接 (Service UUID: 0xA101)
    App->>Device: 7. thing.network.set (bid, userId)

    Device->>Cloud: 8. MQTT 上报 bind (userId, assetId)
    Cloud-->>Device: 9. 绑定成功 (res: 0)
```

**BLE 配网消息（4G 场景）：**

```json
{
  "type": "thing.network.set",
  "data": {
    "bid": "从Sentino服务器获取的assetId",
    "userId": "从Sentino服务器获取的userId"
  }
}
```

> 4G 场景下 `sid`（WiFi SSID）、`pw`（WiFi 密码）、`mq`（MQTT 地址）等字段留空或不传。

### 6.5 配网状态码

设备通过 `thing.network.set.response` 的 `code` 字段返回配网进度：

**WiFi 连接：**

| 状态码 | 说明 |
|:---|:---|
| `1006` | WiFi 连接成功 |
| `1007` | WiFi 密码错误 |
| `1008` | 未发现路由器（信号或设备问题） |

**BLE 数据传输：**

| 状态码 | 说明 |
|:---|:---|
| `1501` | 包头异常 |
| `1502` | 包序号异常 |
| `1503` | CRC 校验失败 |
| `1504` | 设备缓存 1 异常 |
| `1505` | 设备缓存 2 异常 |
| `1506` | 总数据长度异常 |

**JSON 与服务器连接：**

| 状态码 | 说明 |
|:---|:---|
| `1700` | JSON 解析失败 |
| `1701` | JSON 解析成功，开始联网 |
| `1702` | 服务器连接失败 |
| `1703` | 服务器连接成功 |

**绑定：**

| 状态码 | 说明 |
|:---|:---|
| `1801` | 绑定成功 |
| `1802` | 绑定失败 |
| `1803` | 解绑成功 |
| `1804` | 解绑失败 |

---

## 7. 设备生命周期协议

> **TL;DR**: 设备通过 MQTT 上报/接收 JSON 消息来完成绑定、信息同步、属性管理、OTA 等操作。每条消息通过 `code` 字段区分协议类型。

所有协议均通过 MQTT 传输，使用 [5.2 节](#52-topic-定义)定义的 Topic 和 [5.3 节](#53-消息格式)定义的消息格式。

### 7.1 设备绑定 (bind)

设备从 BLE 获取到 `userId` 和 `assetId` 后，通过 MQTT 上报绑定请求。

**code**: `bind`
**方向**: 设备 → 云端（上报）

**上报 data：**

```json
{
  "userId": "配网用户ID",
  "assetId": "资产ID",
  "version": "1.0.0",
  "mcuVersion": "1.0.0",
  "cleanData": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `userId` | string | 是 | 由 App 通过 BLE 传入 |
| `assetId` | string | 是 | 资产 ID，由 App 通过 BLE 传入（BLE 消息中的 `bid` 字段） |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `cleanData` | boolean | 否 | 是否清除数据，默认 `false` |

**云端回复 data：** 无额外字段，`res` 为 `0` 表示绑定成功。

### 7.2 设备信息上报 (info)

> **强烈建议对接此协议。** 若未对接，设备将无法收到因断电/断网导致云端未送达的遗留消息。

**code**: `info`
**方向**: 设备 → 云端（上报）
**建议 ack**: `1`

**上报 data：**

```json
{
  "bindStatus": 1,
  "version": "1.0.0",
  "mcuVersion": "1.0.0",
  "iccid": "898600MFSSYYGXXXXXXP",
  "config": "{\"ipaddr\":\"192.168.1.1\",\"currentSsid\":\"MyWiFi\"}"
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `bindStatus` | int | 是 | `0` = 未绑定，`1` = 已绑定 |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `iccid` | string | 否 | 4G 流量卡 ICCID（4G 设备填写） |
| `checkPwd` | int | 否 | 是否检测未送达密码。`0` = 否，`1` = 是 |
| `config` | string | 否 | WiFi 设备填写，JSON 字符串，含 `ipaddr`、`currentSsid`、`wifiList` |

**云端回复 data：**

```json
{
  "bindStatus": 1,
  "isClearData": 0,
  "logUploadUrl": "http://...",
  "logSwitch": {
    "enable": 1,
    "ttltype": 1,
    "intervalTime": 10
  }
}
```

| 字段 | 说明 |
|:---|:---|
| `bindStatus` | 云端绑定状态 |
| `isClearData` | 是否已清除数据 |
| `logUploadUrl` | 日志上传地址 |
| `logSwitch.enable` | 日志开关（`0` = 关，`1` = 开） |
| `logSwitch.ttltype` | 上报方式：`1` = 单次，`2` = 循环 |
| `logSwitch.intervalTime` | 上传周期（分钟），1~360 |

### 7.3 设备重置 (reset)

**上报（设备主动重置）：**

**code**: `reset`

```json
{ "cleanData": false }
```

**下发（云端远程重置）：**

**code**: `reset`

```json
{ "clearData": true }
```

| 字段 | 说明 |
|:---|:---|
| `cleanData` / `clearData` | 是否清除设备数据 |

### 7.4 获取云端时间 (time)

**code**: `time`
**方向**: 设备 → 云端（上报，`ack=1`）

上报 data 为空，云端回复：

```json
{
  "ts": 1626197189,
  "sys_tz": "America/New_York",
  "zone_offset": -14400,
  "local_date_time": "2024-07-13 04:33:39",
  "is_dst": true,
  "is_zone_dst": true,
  "dst_start_ts": 1710054000,
  "dst_start_local_date_time": "2024-03-10 02:00:00",
  "dst_end_ts": 1730617200,
  "dst_end_local_date_time": "2024-11-03 02:00:00"
}
```

| 字段 | 说明 |
|:---|:---|
| `ts` | UTC 时间戳（秒） |
| `sys_tz` | 系统时区标识 |
| `zone_offset` | 时区偏移量（秒），已含夏令时偏移 |
| `local_date_time` | 当地时间（`yyyy-MM-dd HH:mm:ss`） |
| `is_dst` | 当前是否处于夏令时 |
| `is_zone_dst` | 该地区是否使用夏令时 |
| `dst_start_ts` / `dst_end_ts` | 夏令时起止 UTC 时间戳 |

> 当用户切换时区时，云端可能主动通过 `report_response` 下发 `time` 回复，设备需处理。

### 7.5 获取物模型 (model)

**code**: `model`
**方向**: 设备 → 云端（上报，`ack=1`）

**上报 data：**

```json
{ "format": "complete" }
```

`format` 支持三种模式：

| 格式 | 说明 | 适用场景 |
|:---|:---|:---|
| `complete` | 完整版，含属性全部元数据 | 初始化、调试 |
| `simple` | 精简版，省略部分元数据 | 常规使用 |
| `mini` | 迷你版，字段名缩写 | 低资源设备 |

**云端回复（complete 版）：**

```json
{
  "format": "complete",
  "profile": {
    "version": "V1.0.0",
    "productId": "产品ID"
  },
  "properties": [{
    "identifier": "属性标识符",
    "dpBusiId": "业务ID",
    "name": "属性名称",
    "accessMode": "rw",
    "dataType": {
      "type": "int",
      "specs": {
        "min": "0",
        "max": "100",
        "step": "1",
        "unit": "%"
      }
    }
  }]
}
```

**属性类型（`dataType.type`）：**

| 类型 | 说明 |
|:---|:---|
| `int` | 整型 |
| `float` | 单精度浮点 |
| `double` | 双精度浮点 |
| `text` | 字符串 |
| `date` | 日期（String 类型，UTC 毫秒） |
| `bool` | 布尔（int 型，0 或 1） |
| `enum` | 枚举（int 型） |
| `struct` | 结构体 |
| `array` | 数组 |

### 7.6 属性上报 (property_report)

**code**: `property_report`
**方向**: 设备 → 云端

```json
{
  "properties": {
    "color": "red",
    "brightness": 80
  }
}
```

### 7.7 属性设置 (property_set)

**code**: `property_set`
**方向**: 云端 → 设备（下发）

```json
{
  "properties": {
    "color": "blue",
    "brightness": 60
  }
}
```

设备收到后执行属性变更，并通过 `issue_response` 回复实际设置后的属性值。

### 7.8 OTA 固件升级

#### 云端下发升级指令

**code**: `ota`
**方向**: 云端 → 设备

```json
{
  "firmwareType": 2,
  "fileSize": 708482,
  "silence": false,
  "md5sum": "36eb5951179db14a63a37a9322a2",
  "url": "https://ota.example.com/firmware.bin",
  "version": "1.2.0",
  "options": { "stable": "usr" }
}
```

| 字段 | 说明 |
|:---|:---|
| `firmwareType` | `1` = 工厂固件，`2` = 标准固件，`3` = MCU，`4` = MCU 蓝牙，`5` = MCU Zigbee，`6` = MCU Matter |
| `fileSize` | 固件文件大小（字节） |
| `silence` | 是否静默升级 |
| `md5sum` | 文件 MD5 校验值 |
| `url` | 固件下载地址 |
| `version` | 目标版本号 |
| `options.stable` | 升级分区：`usr` / `recovery` / `kernel` |

#### 设备上报升级进度

**code**: `ota_progress`
**方向**: 设备 → 云端

```json
{
  "resCode": 0,
  "type": "downloading",
  "percent": 80
}
```

| type | 说明 | 额外字段 |
|:---|:---|:---|
| `downloading` | 下载中 | `percent`（0~100） |
| `burning` | 烧录中 | — |
| `report` | 上报新版本号 | `version`、`mcuVersion` |
| `fail` | 失败 | — |

`resCode` 错误码：

| 值 | 说明 |
|:---|:---|
| `0` | 成功 |
| `-1` | 下载超时 |
| `-2` | 文件不存在 |
| `-3` | 签名过期 |
| `-4` | MD5 不匹配 |
| `-5` | 更新固件失败 |
| `-6` | 更新超时 |
| `-7` | 正在升级中 |

### 7.9 在线检测 (ping)

**code**: `ping`
**方向**: 云端 → 设备（下发，`ack=1`）

设备回复：

```json
{
  "currentSsid": "MyWiFi",
  "wifiList": [{ "ssid": "MyWiFi", "rssi": -50 }],
  "ipaddr": "192.168.1.100",
  "networkType": 1,
  "signal": 1,
  "signalValue": 85
}
```

| 字段 | 说明 |
|:---|:---|
| `networkType` | `1` = WiFi，`2` = 有线 |
| `signal` | `1` = 好(10~50ms)，`2` = 中(50~100ms)，`3` = 差(>100ms) |
| `signalValue` | 信号强度百分比（0~100） |

---

## 8. AI 语音对话

> **TL;DR**: 设备通过 MQTT 上报 `agora_agent_device_access` 获取 Agora RTC 参数，然后用 Agora SDK 加入频道即可与 AI Agent 实时语音对话。结束时 `leave_channel` 即可，云端自动清理。

### 8.1 接入流程

```mermaid
sequenceDiagram
    participant Device as IoT 设备
    participant Cloud as Sentino 云端
    participant Agora as Agora RTC

    Device->>Cloud: MQTT 上报 agora_agent_device_access
    Cloud->>Agora: 创建 AI Agent + RTC 频道
    Cloud-->>Device: 回复 appId, channelName, rtcToken, uid

    Device->>Agora: agora_rtc_init(appId)
    Device->>Agora: agora_rtc_join_channel(channelName, uid, rtcToken)

    Note over Device,Agora: 实时语音对话 (RTC)

    Device->>Agora: agora_rtc_leave_channel()
    Agora-->>Cloud: 检测设备离开
    Cloud->>Cloud: 自动清理 AI 会话
```

### 8.2 普通设备接入 AI

**code**: `agora_agent_device_access`
**方向**: 设备 → 云端（上报，`ack=1`）
**上报 data**: `{}`（空对象）

**云端回复 data：**

```json
{
  "appId": "935b8af0c93640b09d614e48f1a8f75a",
  "rtcToken": "007eJxTYBA...",
  "channelName": "convo_stn_TgLI80UukY948M5v",
  "uid": 25532
}
```

| 字段 | 类型 | 说明 | 设备端用途 |
|:---|:---|:---|:---|
| `appId` | string | Agora 应用 ID | 初始化 SDK |
| `rtcToken` | string | RTC 通道令牌（有时效） | 加入频道鉴权 |
| `channelName` | string | RTC 通道名称 | 加入频道 |
| `uid` | int | RTC 成员 ID | 加入频道 |

> Token 有时效性，获取后应尽快加入频道。会话启动后如长时间无人加入或无语音活动，云端会自动关闭会话。

### 8.3 NFC 设备接入 AI

**code**: `agora_agent_nfc_report`
**方向**: 设备 → 云端（上报，`ack=1`）

**上报 data：**

```json
{
  "nfcIdentifier": "NFC卡片唯一标识",
  "onlyReport": 0
}
```

| 字段 | 说明 |
|:---|:---|
| `nfcIdentifier` | NFC 卡片唯一标识 |
| `onlyReport` | `1` = 仅上报（不返回 RTC 信息），`0` = 上报并返回 RTC 信息 |

`onlyReport=0` 时，云端回复与普通设备接入 AI 相同（包含 `appId`、`rtcToken`、`channelName`、`uid`）。

### 8.4 Agora RTC 集成

设备端需集成 Agora RTC SDK（嵌入式版，C API）。

**音频参数：**

| 参数 | 值 |
|:---|:---|
| 编码格式 | OPUS |
| 采样率 | 16000 Hz |
| 声道数 | 1（单声道） |
| PCM 位深 | 16 bit |
| 帧长 | 20 ms |

**集成示例（C）：**

```c
// 1. 初始化
agora_rtc_init(appId, &event_handler);

// 2. 音频配置
agora_rtc_config_t config = {0};
config.audio_codec      = AUDIO_CODEC_OPUS;
config.pcm_sample_rate  = 16000;
config.pcm_channel_num  = 1;

// 3. 加入频道（AI Agent 已在频道内等待）
agora_rtc_join_channel(channelName, uid, rtcToken, &config);

// 4. 音频回调
static void on_audio_data(const char *channel, uint32_t uid,
                          const void *data, size_t len) {
    // 收到 AI Agent 的音频，送入扬声器播放
    audio_play(data, len);
}

// 5. 结束对话
agora_rtc_leave_channel();
// 云端自动检测设备离开并清理 AI 会话，无需额外 MQTT 上报
```

---

## 9. REST API 参考

> **TL;DR**: App 端通过 `https://api-iot.sentino.jp` 调用 REST API 完成用户登录、设备配网、设备管理和智能体绑定。所有接口（除登录外）需携带 Bearer Token。

**基础 URL**: `https://api-iot.sentino.jp`

### 9.1 认证说明

**认证方式：** 除登录接口外，所有接口使用 Bearer Token 认证。

```
Authorization: Bearer {access_token}
```

**公共请求头（所有业务接口必须携带）：**

| 参数名 | 类型 | 说明 | 示例 |
|:---|:---|:---|:---|
| `timezone` | string | 时区 | `Asia/Shanghai` |
| `language` | string | 语言 | `zh_CN` |
| `data_center_code` | string | 数据中心代码 | `cn` |
| `client_id` | string | 客户端 ID | （由 Sentino 分配） |
| `encrypt_type` | string | 加密类型 | `AES/ECB/PKCS5Padding` |
| `channel_identifier` | string | 渠道标识 | （由 Sentino 分配） |
| `package_name` | string | 应用包名 | （由 Sentino 分配） |
| `app_id` | string | 应用 ID | （由 Sentino 分配） |

**统一响应格式：**

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "reqId": "请求追踪ID",
  "time": 1620641786
}
```

### 9.2 用户登录

注册登录一体接口，UID 不存在时自动注册。

`POST /auth/oauth/token`

**请求头：**

| 参数名 | 值 |
|:---|:---|
| Content-Type | `application/x-www-form-urlencoded` |
| Authorization | `Basic {client_credentials}` |
| app_id | 应用 ID |

**Body（form）：**

| 参数 | 必填 | 说明 |
|:---|:---|:---|
| `grant_type` | 是 | 固定值 `uid` |
| `uid` | 是 | 用户唯一标识，最长 50 字符 |
| `password` | 是 | 用户密码 |
| `area_code` | 是 | 区号，默认 `86` |
| `user_country_key` | 是 | 国家代码，默认 `CN` |

<details>
<summary>curl 示例</summary>

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&app_id=YOUR_APP_ID" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "app_id: YOUR_APP_ID" \
  -H "Authorization: Basic YOUR_CLIENT_CREDENTIALS" \
  -d "grant_type=uid&uid=test_user_001&password=your_password&area_code=86&user_country_key=CN"
```

</details>

**响应 data：**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 7200,
  "refresh_token": "..."
}
```

### 9.3 获取产品信息

`POST /business-app/v1/product/getByProductId?productId={productId}`

<details>
<summary>curl 示例</summary>

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=YOUR_PRODUCT_ID" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer {access_token}"
```

</details>

**响应 data 关键字段：**

| 字段 | 说明 |
|:---|:---|
| `productName` | 产品名称 |
| `protocolType` | 通讯协议（MQTT / BLE） |
| `distributionNetMode` | 配网模式：`1` = WiFi+BLE，`2` = WiFi，`3` = BLE |
| `bindMode` | 绑定模式：`1` = 强绑，`2` = 弱绑 |
| `nodeType` | 节点类型：`1` = 普通设备，`2` = 网关，`3` = 边缘网关，`4` = 子设备 |
| `internetType` | 联网方式：`wifi` / `wired` / `4G` |

### 9.4 获取资产树

获取用户的家庭/房间结构，返回 `assetId` 用于设备绑定。

`POST /business-app/v1/asset/assetTree`

**Body**: `{}`

<details>
<summary>curl 示例</summary>

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "{}"
```

</details>

**响应 data：**

```json
{
  "assetId": "1983052957022670848",
  "assetName": "我的家",
  "children": [
    { "assetId": "1983052957022670849", "assetName": "客厅", "children": [] }
  ]
}
```

### 9.5 获取设备信息

`POST /business-app/v1/device/getSimpleDeviceInfo`

**Body：**

```json
{ "productId": "产品ID", "uuid": "设备UUID" }
```

**响应 data：**

```json
{
  "uuid": "设备UUID",
  "productId": "产品ID",
  "deviceName": "设备名称",
  "protocolType": "MQTT",
  "configMode": "BLE",
  "bindStatus": 0
}
```

### 9.6 配网数据加密

将配网信息加密后通过 BLE 发送给设备。

`POST /business-app/v1/distributionNet/dataEncrypt`

**Body：**

```json
{
  "content": "{\"sid\":\"WiFi名\",\"pw\":\"密码\",\"force_bind\":true,\"mq\":\"mqtt地址\",\"port\":1883,\"mqttSslPort\":\"8883\",\"bid\":\"资产ID\",\"userId\":\"用户ID\",\"country\":\"CN\",\"tz\":\"Asia/Shanghai\",\"areaCode\":\"US\"}",
  "encryptType": 0,
  "protocol": "1",
  "type": "thing.network.set"
}
```

| content 字段 | 说明 | 来源 |
|:---|:---|:---|
| `sid` | WiFi SSID | 用户输入 |
| `pw` | WiFi 密码 | 用户输入 |
| `bid` | 资产 ID | [获取资产树](#94-获取资产树) |
| `userId` | 用户 ID | 登录接口返回 |
| `mq` | MQTT 地址 | 数据中心配置 |
| `force_bind` | 是否强制绑定 | 业务逻辑 |

<details>
<summary>curl 示例</summary>

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/distributionNet/dataEncrypt" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{
    "content": "{\"sid\":\"MyWiFi\",\"pw\":\"12345678\",\"force_bind\":true,\"mq\":\"mqtt-iot.sentino.jp\",\"port\":1883,\"mqttSslPort\":\"8883\",\"bid\":\"asset_id_here\",\"userId\":\"user_id_here\",\"country\":\"CN\",\"tz\":\"Asia/Shanghai\",\"areaCode\":\"CN\"}",
    "encryptType": 0,
    "protocol": "1",
    "type": "thing.network.set"
  }'
```

</details>

**响应 data：** 加密后的配网数据字符串，直接通过 BLE 发送给设备。

### 9.7 绑定状态查询

App 通过轮询（建议每 2~10 秒）检查设备是否绑定成功。

`POST /business-app/v1/device/bind/checkBindResult/{uuid}`

<details>
<summary>curl 示例</summary>

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/checkBindResult/ct01wfjSNqGAqUUK" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}"
```

</details>

**响应 data：** `0` = 绑定成功，其他值 = 未完成

### 9.8 获取设备列表

`POST /business-app/v1/device/getHomeDeviceAndGroupList`

**Body：**

```json
{ "assetIds": ["家庭ID1", "家庭ID2"] }
```

**响应 data：**

```json
{
  "deviceList": [
    {
      "deviceId": "设备ID",
      "uuid": "设备UUID",
      "deviceName": "设备名称",
      "online": true,
      "productId": "产品ID"
    }
  ],
  "sortIdList": ["设备排序ID列表"],
  "shareList": [],
  "shareGroupList": []
}
```

### 9.9 设备解绑

`POST /business-app/v1/device/bind/unbind`

**Body：**

```json
{ "deviceId": "设备ID", "isCleanData": 1 }
```

| 字段 | 说明 |
|:---|:---|
| `isCleanData` | `0` = 保留数据，`1` = 清除数据 |

### 9.10 OTA 升级检查

`POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}`

- `firmwareType` 不传时默认检查模组固件

**响应 data：**

```json
{
  "hasUpgrade": true,
  "version": "1.2.0",
  "downloadUrl": "https://...",
  "md5": "abc123...",
  "fileSize": 1024000,
  "upgradeDesc": "修复已知问题"
}
```

### 9.11 智能体管理

#### 获取推荐智能体列表

`POST /business-app/v1/agents/recommend/agents-list`

**Body**: `{}`

**响应 data：**

```json
[
  {
    "agentId": "智能体ID",
    "avatar": "头像URL",
    "name": "小助手",
    "description": "角色描述",
    "tags": ["助手"]
  }
]
```

#### 获取智能体详情

`POST /business-app/v1/agents/detail?agentId={agentId}`

#### 设备绑定智能体

`POST /business-app/v1/agents/device/bind-agent`

**Body：**

```json
{
  "agentId": "智能体ID",
  "agentType": "official",
  "deviceId": "设备ID"
}
```

| 字段 | 说明 |
|:---|:---|
| `agentType` | `official` = 官方模板，`customize` = 自定义 |

---

## 10. 断线重连策略

### MQTT 断连重连

| 策略 | 说明 |
|:---|:---|
| 重连间隔 | 指数退避：1s → 2s → 4s → 8s → ... 最大 60s |
| 重新认证 | 使用相同三元组重新计算签名（更新 `ts` 为当前时间戳） |
| 重连后操作 | 重新订阅 `report_response` 和 `issue` 两个 Topic |

### RTC 断连重连

| 策略 | 说明 |
|:---|:---|
| 优先复用 | 使用已有的 RTC 参数（appId / channelName / rtcToken / uid）直接重连 |
| Token 过期 | 若复用失败，重新上报 `agora_agent_device_access` 获取新参数 |

---

## 11. 错误排查

| 场景 | 现象 | 排查方向 |
|:---|:---|:---|
| MQTT 连接被拒 | 无法连接 Broker | 检查三元组是否正确；签名算法是否匹配；`ts` 时间戳偏差是否过大 |
| 绑定失败 | `bind` 回复 `res != 0` | 检查 `userId` 和 `assetId` 是否正确 |
| AI 接入失败 | `agora_agent_device_access` 回复 `res != 0` | 检查设备是否已绑定；智能体是否已配置 |
| RTC 加入频道失败 | `join_channel` 返回错误 | 检查 4 个 RTC 参数是否完整；Token 是否过期 |
| 无声音 | RTC 连接成功但无音频 | 检查音频参数（16kHz / 单声道 / 16bit / OPUS）；检查麦克风和扬声器硬件 |
| 上报消息无回复 | 消息发出后无响应 | 检查 `id` 是否重复（需 UUID 保证唯一）；`ack` 是否设为 `1` |
| BLE 配网失败 | 无法发现设备 | 确认 BLE 广播已启动；Service UUID 是否为 `0xA101` |
| BLE 数据传输错误 | 收到 150x 错误码 | 检查分包格式、CRC 校验、包序号连续性 |
| WiFi 连接失败 | 收到 1007/1008 | 检查 WiFi 密码；确认路由器支持 2.4G；检查信号强度 |

---

## 12. 附录

### 附录 A：REST API 错误码

| 错误码 | 说明 |
|:---|:---|
| `200` | 成功 |
| `400` | 请求参数错误 |
| `401` | 未授权（Token 无效或过期） |
| `403` | 禁止访问 |
| `404` | 资源不存在 |
| `500` | 服务器内部错误 |

### 附录 B：MQTT 事件编码速查表

**设备上报（report）：**

| code | 说明 | ack |
|:---|:---|:---|
| `bind` | 设备绑定 | 1 |
| `info` | 设备信息上报 | 1 |
| `reset` | 设备重置 | 0 |
| `time` | 获取云端时间 | 1 |
| `model` | 获取物模型 | 1 |
| `property_report` | 属性上报 | 0 |
| `ota_progress` | OTA 进度上报 | 0 |
| `agora_agent_device_access` | 请求 AI 语音接入 | 1 |
| `agora_agent_nfc_report` | NFC 设备请求 AI 接入 | 1 |

**云端下发（issue）：**

| code | 说明 |
|:---|:---|
| `reset` | 远程重置设备 |
| `ota` | 固件升级 |
| `ping` | 在线检测 |
| `property_set` | 设置属性 |

### 附录 C：BLE 配网状态码速查表

| 范围 | 类别 | 关键状态码 |
|:---|:---|:---|
| 1006~1008 | WiFi 连接 | `1006` 成功 / `1007` 密码错误 / `1008` 路由器未发现 |
| 1501~1506 | BLE 传输 | 包头/序号/CRC/缓存/长度异常 |
| 1700~1703 | JSON 与服务器 | `1700` JSON 解析失败 / `1703` 服务器连接成功 |
| 1801~1806 | 绑定/初始化 | `1801` 绑定成功 / `1802` 绑定失败 |

### 附录 D：完整时序图 — 4G 设备首次绑定到 AI 对话

```mermaid
sequenceDiagram
    participant App as 配网 App
    participant Device as IoT 设备 (4G)
    participant Cloud as Sentino 云端
    participant Agora as Agora RTC

    rect rgb(240, 248, 255)
    Note over App,Agora: 首次绑定

    Device->>Cloud: 1. 使用三元组连接 MQTT
    Device->>Cloud: 2. 订阅 report_response + issue

    App->>App: 3. 扫描设备条码 (Barcode)
    App->>Cloud: 4. 请求 userId / assetId
    Cloud-->>App: 5. 返回 userId / assetId

    App->>Device: 6. BLE 连接 (Service UUID: 0xA101)
    App->>Device: 7. thing.network.set (bid, userId)

    Device->>Cloud: 8. MQTT 上报 bind (userId, assetId, version)
    Cloud-->>Device: 9. 绑定成功 (res: 0)

    Device->>Cloud: 10. MQTT 上报 info (bindStatus, version, iccid)
    Cloud-->>Device: 11. info 回复
    end

    rect rgb(255, 248, 240)
    Note over App,Agora: 每次对话

    Device->>Cloud: 12. MQTT 上报 agora_agent_device_access
    Cloud->>Agora: 13. 创建 AI Agent + RTC 频道
    Cloud-->>Device: 14. 回复 appId, channelName, rtcToken, uid

    Device->>Agora: 15. agora_rtc_join_channel
    Note over Device,Agora: 16. 实时语音对话 (RTC)
    Device->>Agora: 17. agora_rtc_leave_channel
    Agora-->>Cloud: 18. 检测离开，清理会话
    end
```

---

**文档版本**: v1.0
**最后更新**: 2026-03-25
**维护者**: Sentino IoT Team
