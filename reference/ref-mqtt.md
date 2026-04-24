# MQTT 协议参考

本文档是 Sentino MQTT 协议的完整字段级参考。如需了解使用流程和集成方法，请参阅设备端集成指南。

> **前置知识**：请先阅读 [架构与概念](../architecture.md)。

---

## 1. 连接参数

| 参数 | 格式 | 说明 |
|:---|:---|:---|
| Broker 地址 | `mqtt-iot.sentino.jp` | 测试环境 |
| 端口 | `1883` | 明文，未启用 TLS |
| 协议版本 | MQTT 5.0 | Broker 兼容 MQTT 3.1.1 客户端（如 ali_mqtt 协议级别 4） |
| Client ID | `rlink_${uuid}_V2` | 固定以 `_V2` 结尾 |
| Username | `${uuid}\|signMethod=${signMethod},ts=${ts}` | `\|` 为分隔符 |
| Password | `hmacSha256("uuid=${uuid},ts=${ts}", KEY)` | KEY 为三元组中的设备密钥 |
| signMethod | `hmacSha256`（推荐）或 `hmacSha1` | — |
| Keep Alive | 60s（建议） | — |
| QoS | 1（发布时建议） | — |

**签名计算：**

```
content  = "uuid=${uuid},ts=${ts}"     // ts 为当前 UNIX 时间戳（秒）
password = hmacSha256(content, KEY)     // KEY 为三元组中的 32 字符密钥
```

---

## 2. Topic 定义

**变量说明：**
- `${pid}` — 产品 ID 或产品型号（第三方设备型号）
- `${uuid}` — 设备 UUID（每台设备不同，取自三元组）

| 方向 | Topic | 说明 | 设备需要 |
|:---|:---|:---|:---|
| 设备 → 云端 | `rlink/v2/${pid}/${uuid}/report` | 上报事件 | 发布 |
| 云端 → 设备 | `rlink/v2/${pid}/${uuid}/report_response` | 回复上报 | **订阅** |
| 云端 → 设备 | `rlink/v2/${pid}/${uuid}/issue` | 下发指令 | **订阅** |
| 设备 → 云端 | `rlink/v2/${pid}/${uuid}/issue_response` | 回复指令 | 发布 |

> 设备连接后需订阅 `report_response` 和 `issue`。

---

## 3. 消息格式

### 3.1 上报消息（设备 → 云端，通过 `report`）

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "事件编码",
  "data": {},
  "ack": 1
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `id` | string | 是 | 消息唯一 ID。同一设备短时间内不可重复，否则消息被忽略 |
| `ts` | int | 是 | 当前 UNIX 时间戳（秒） |
| `code` | string | 是 | 事件编码，见第 4 节 |
| `data` | object | 是 | 上报数据，见各事件定义 |
| `ack` | int | 是 | `0` = 不需要云端回复；`1` = 需要云端回复 |

### 3.2 云端回复（云端 → 设备，通过 `report_response`）

仅在上报 `ack=1` 时回复。

```json
{
  "res": 0,
  "msg": "success",
  "id": "与上报消息 id 一致",
  "ts": 1742536800,
  "code": "与上报事件编码一致",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `res` | int | `0` = 成功，非 `0` = 失败 |
| `msg` | string | 结果描述 |
| `id` | string | 与上报消息 id 一致 |
| `code` | string | 与上报事件编码一致 |
| `data` | object | 回复数据，见各事件定义 |

### 3.3 下发指令（云端 → 设备，通过 `issue`）

```json
{
  "id": "消息ID",
  "ts": 1742536800,
  "code": "指令编码",
  "data": {}
}
```

### 3.4 设备回复指令（设备 → 云端，通过 `issue_response`）

```json
{
  "res": 0,
  "msg": "success",
  "id": "与下发消息 id 一致",
  "ts": 1742536800,
  "code": "与下发指令编码一致",
  "data": {}
}
```

---

## 4. 设备上报事件

### 4.1 bind — 设备绑定

设备从 BLE 获取 userId / assetId 后上报绑定请求。

| 属性 | 值 |
|:---|:---|
| code | `bind` |
| 建议 ack | `1` |

**上报 data：**

```json
{
  "userId": "配网用户ID",
  "assetId": "账户ID",
  "version": "1.0.0",
  "mcuVersion": "1.0.0",
  "cleanData": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `userId` | string | 是 | 用户 ID，由 App 通过 BLE 传入 |
| `assetId` | string | 是 | 账户 ID，由 App 通过 BLE 传入（BLE 消息中的 `bid` 字段） |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `cleanData` | boolean | 否 | 是否清除数据，默认 `false` |

**云端回复 data：** 无额外字段。`res=0` 表示绑定成功。

---

### 4.2 info — 设备信息上报

> 强烈建议对接。未对接时，设备将无法收到因断电/断网导致云端未送达的遗留消息。

| 属性 | 值 |
|:---|:---|
| code | `info` |
| 建议 ack | `1` |

**上报 data：**

```json
{
  "checkPwd": 0,
  "bindStatus": 1,
  "version": "1.0.0",
  "mcuVersion": "1.0.0",
  "mcuBleVersion": "1.0.0",
  "mcuZigbeeVersion": "1.0.0",
  "mcuMatterVersion": "1.0.0",
  "iccid": "898600MFSSYYGXXXXXXP",
  "config": "{\"ipaddr\":\"192.168.1.1\",\"currentSsid\":\"MyWiFi\",\"wifiList\":[{\"ssid\":\"wifi1\",\"pwd\":\"12345678\"}]}"
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `checkPwd` | int | 否 | 是否检测未送达密码。`0` = 否（默认），`1` = 是 |
| `bindStatus` | int | 是 | `0` = 未绑定，`1` = 已绑定 |
| `version` | string | 是 | 固件版本号 |
| `mcuVersion` | string | 否 | MCU 版本号 |
| `mcuBleVersion` | string | 否 | MCU 蓝牙版本号 |
| `mcuZigbeeVersion` | string | 否 | MCU Zigbee 版本号 |
| `mcuMatterVersion` | string | 否 | MCU Matter 版本号 |
| `iccid` | string | 否 | ICCID 卡号（流量卡）（选填） |
| `config` | string | 否 | 设备网络信息，JSON 字符串。含 `ipaddr`（局域网 IP）、`currentSsid`（当前 WiFi）、`wifiList`（备用网络列表） |

**云端回复 data：**

```json
{
  "bindStatus": 0,
  "isClearData": 0,
  "logUploadUrl": "http://xxxx.com/xxxx",
  "tcpUrl": "xxx.xxx.com",
  "logSwitch": {
    "enable": 1,
    "ttltype": 1,
    "intervalTime": 10
  }
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `bindStatus` | int | 云端绑定状态：`0` = 未绑定，`1` = 已绑定 |
| `isClearData` | int | 是否已清除数据：`0` = 否，`1` = 是 |
| `logUploadUrl` | string | 日志上传地址 |
| `tcpUrl` | string | 低功耗 TCP 地址 |
| `logSwitch.enable` | int | 日志开关：`0` = 关，`1` = 开 |
| `logSwitch.ttltype` | int | 上报方式（enable=1 时必传）：`1` = 单次，`2` = 循环 |
| `logSwitch.intervalTime` | int | 上传周期（enable=1 时必传），1~360 分钟 |

---

### 4.3 reset — 设备主动重置

| 属性 | 值 |
|:---|:---|
| code | `reset` |
| 建议 ack | `0` |

**上报 data：**

```json
{
  "cleanData": false
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `cleanData` | boolean | 是否清除数据 |

**云端回复 data：** 无额外字段（ack=0 时不回复）。

---

### 4.4 time — 获取云端时间

| 属性 | 值 |
|:---|:---|
| code | `time` |
| 建议 ack | `1` |

**上报 data：** 无（不传 data 字段或传空对象）。

**云端回复 data：**

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

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `ts` | int | UTC 时间戳（秒） |
| `sys_tz` | string | 系统时区标识 |
| `zone_offset` | int | 时区偏移量（秒），已含夏令时偏移 |
| `local_date_time` | string | 当地时间 (`yyyy-MM-dd HH:mm:ss`) |
| `is_dst` | boolean | 当前是否处于夏令时 |
| `is_zone_dst` | boolean | 该地区是否使用夏令时 |
| `dst_start_ts` | int | 夏令时开始 UTC 时间戳（无夏令时则为空） |
| `dst_start_local_date_time` | string | 夏令时开始当地时间（无夏令时则为空） |
| `dst_end_ts` | int | 夏令时结束 UTC 时间戳（无夏令时则为空） |
| `dst_end_local_date_time` | string | 夏令时结束当地时间（无夏令时则为空） |

> 用户切换时区时，云端可能主动通过 `report_response` 下发 `time` 回复，设备需处理。

---

### 4.5 model — 获取物模型

| 属性 | 值 |
|:---|:---|
| code | `model` |
| 建议 ack | `1` |

**上报 data：**

```json
{
  "format": "complete"
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `format` | string | 请求格式：`complete`（完整版）、`simple`（精简版）、`mini`（迷你版） |

#### 云端回复 — complete（完整版）

```json
{
  "format": "complete",
  "profile": {
    "version": "V1.0.0",
    "productId": "产品ID"
  },
  "properties": [{
    "identifier": "brightness",
    "dpBusiId": "11",
    "name": "亮度",
    "accessMode": "rw",
    "required": true,
    "dataTypeStr": "int",
    "dataType": {
      "type": "int",
      "specs": {
        "min": "0",
        "max": "100",
        "unit": "%",
        "unitName": "百分比",
        "step": "1"
      }
    }
  }],
  "events": []
}
```

**properties 字段说明：**

| 字段 | 说明 |
|:---|:---|
| `identifier` | 属性唯一标识符（物模型内唯一） |
| `dpBusiId` | 业务 ID |
| `name` | 属性名称 |
| `accessMode` | 读写类型：`r`（只读）或 `rw`（读写） |
| `required` | 是否为标准功能必选属性 |
| `dataType.type` | 属性类型（见下表） |
| `dataType.specs` | 属性规格 |

**属性类型 (`dataType.type`)：**

| 类型 | 说明 | specs 特有字段 |
|:---|:---|:---|
| `int` | 整型 | `min`, `max`, `unit`, `unitName`, `step` |
| `float` | 单精度浮点 | `min`, `max`, `unit`, `unitName`, `step` |
| `double` | 双精度浮点 | `min`, `max`, `unit`, `unitName`, `step` |
| `text` | 字符串 | `length`（最大 10240） |
| `date` | 日期（String，UTC 毫秒） | — |
| `bool` | 布尔（int，0 或 1） | — |
| `enum` | 枚举（int） | `enums`（枚举值列表） |
| `struct` | 结构体 | `specs` 数组描述子字段 |
| `array` | 数组 | `size`（最大 512）, `item.type` |

#### 云端回复 — simple（精简版）

```json
{
  "format": "simple",
  "profile": { "version": "V1.0.0", "productId": "产品ID" },
  "properties": [{
    "identifier": "brightness",
    "busiId": "11",
    "dataType": "int",
    "specs": { "min": "0", "max": "100", "step": "1" }
  }]
}
```

#### 云端回复 — mini（迷你版）

```json
{
  "format": "mini",
  "profile": { "version": "V1.0.0", "productId": "产品ID" },
  "properties": [{
    "i": "brightness",
    "b": "11",
    "t": "int"
  }]
}
```

| 字段 | 说明 |
|:---|:---|
| `i` | identifier（属性标识） |
| `b` | busiId（业务 ID） |
| `t` | type（属性类型） |

---

### 4.6 property_report — 属性上报

| 属性 | 值 |
|:---|:---|
| code | `property_report` |
| 建议 ack | `0` |

**上报 data：**

```json
{
  "properties": {
    "color": "red",
    "brightness": 80
  }
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `properties` | object | 属性键值对集合 |

**云端回复 data：** 无额外字段。

---

### 4.7 ota_progress — OTA 进度上报

| 属性 | 值 |
|:---|:---|
| code | `ota_progress` |
| 建议 ack | `0` |

**上报 data（通用字段）：**

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `resCode` | int | 是 | 状态码，见下表 |
| `type` | string | 是 | 阶段类型，见下表 |
| `subPid` | string | 否 | 子设备产品 ID（网关子设备升级时必填） |
| `subUuid` | string | 否 | 子设备 UUID（网关子设备升级时必填） |

**type 类型：**

| type | 说明 | 额外字段 |
|:---|:---|:---|
| `downloading` | 下载中 | `percent`（int, 0~100） |
| `burning` | 烧录中 | — |
| `report` | 上报新版本号（升级完成） | `version`, `mcuVersion`, `mcuBleVersion`, `mcuZigbeeVersion`, `mcuMatterVersion` |
| `fail` | 升级失败 | — |

**resCode 状态码：**

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

**上报示例 — 下载中：**

```json
{
  "resCode": 0,
  "type": "downloading",
  "percent": 80
}
```

**上报示例 — 升级完成：**

```json
{
  "resCode": 0,
  "type": "report",
  "version": "1.2.0",
  "mcuVersion": "1.0.0"
}
```

---

### 4.8 agora_agent_device_access — 普通设备请求 AI 接入

| 属性 | 值 |
|:---|:---|
| code | `agora_agent_device_access` |
| 建议 ack | `1` |

**上报 data：** `{}`（空对象）

**云端回复 data：**

```json
{
  "appId": "935b8af0c93640b09d614e48f1a8f75a",
  "rtcToken": "007eJxTYBA...",
  "channelName": "convo_stn_TgLI80UukY948M5v",
  "uid": 25532
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `appId` | string | Agora 应用 ID |
| `rtcToken` | string | RTC 通道令牌 |
| `channelName` | string | RTC 通道名称 |
| `uid` | int | RTC 成员 ID |


---

### 4.9 agora_agent_nfc_report — NFC 设备请求 AI 接入

| 属性 | 值 |
|:---|:---|
| code | `agora_agent_nfc_report` |
| 建议 ack | `1` |

**上报 data：**

```json
{
  "nfcIdentifier": "NFC卡片唯一标识",
  "onlyReport": 0
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `nfcIdentifier` | string | 是 | NFC 卡片唯一标识 |
| `onlyReport` | int | 是 | `1` = 仅上报（不返回 RTC），`0` = 上报并返回 RTC 信息 |

**云端回复 data（onlyReport=0 时）：**

与 `agora_agent_device_access` 回复格式一致（`appId`, `rtcToken`, `channelName`, `uid`）。

---

## 5. 云端下发指令

### 5.1 reset — 远程重置设备

| 属性 | 值 |
|:---|:---|
| code | `reset` |

**下发 data：**

```json
{
  "clearData": true
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `clearData` | boolean | 是否清除设备数据 |

**设备回复 data：** 无额外字段。

---

### 5.2 ota — 固件升级

| 属性 | 值 |
|:---|:---|
| code | `ota` |

**下发 data：**

```json
{
  "subUuids": ["子设备UUID1", "子设备UUID2"],
  "firmwareType": 2,
  "fileSize": 708482,
  "silence": false,
  "md5sum": "36eb5951179db14a63a37a9322a2",
  "url": "https://ota.example.com/firmware.bin",
  "version": "1.2.0",
  "options": {
    "stable": "usr"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|:---|:---|:---|:---|
| `subUuids` | string[] | 否 | 子设备 UUID 列表（网关子设备升级时必填） |
| `firmwareType` | int | 是 | 固件类型（见下表） |
| `fileSize` | int | 是 | 文件大小（字节） |
| `silence` | boolean | 是 | 是否静默升级 |
| `md5sum` | string | 是 | 文件 MD5 校验值 |
| `url` | string | 是 | 固件下载地址 |
| `version` | string | 是 | 目标版本号 |
| `options.stable` | string | 否 | 升级分区：`usr` / `recovery` / `kernel` |

**firmwareType 固件类型：**

| 值 | 说明 |
|:---|:---|
| `1` | 工厂固件 |
| `2` | 标准固件 |
| `3` | MCU 固件 |
| `4` | MCU 蓝牙固件 |
| `5` | MCU Zigbee 固件 |
| `6` | MCU Matter 固件 |

**设备回复 data：** 无额外字段。

---

### 5.3 ping — 在线检测

| 属性 | 值 |
|:---|:---|
| code | `ping` |

**下发 data：** 无。

**设备回复 data：**

```json
{
  "currentSsid": "MyWiFi",
  "wifiList": [
    { "ssid": "MyWiFi", "rssi": -50 },
    { "ssid": "Neighbor", "rssi": -80 }
  ],
  "ipaddr": "192.168.1.100",
  "networkType": 1,
  "signal": 1,
  "signalValue": 85
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `currentSsid` | string | 当前连接的 WiFi 名称 |
| `wifiList` | array | 可用网络列表，每项含 `ssid` 和 `rssi` 信号强度 |
| `ipaddr` | string | 局域网 IP（不传时后台使用公网 IP） |
| `networkType` | int | `1` = WiFi，`2` = 有线 |
| `signal` | int | `1` = 好 (10~50ms)，`2` = 中 (50~100ms)，`3` = 差 (>100ms) |
| `signalValue` | int | 信号强度百分比，0~100 |

---

### 5.4 property_set — 设置属性

| 属性 | 值 |
|:---|:---|
| code | `property_set` |

**下发 data：**

```json
{
  "properties": {
    "color": "blue",
    "brightness": 60
  }
}
```

| 字段 | 类型 | 说明 |
|:---|:---|:---|
| `properties` | object | 属性键值对集合 |

**设备回复 data：**

```json
{
  "properties": {
    "color": "blue",
    "brightness": 60
  }
}
```

---

## 6. 事件编码速查表

### 设备上报 (report)

| code | 说明 | 建议 ack | 参考章节 |
|:---|:---|:---|:---|
| `bind` | 设备绑定 | 1 | [4.1](#41-bind--设备绑定) |
| `info` | 设备信息上报 | 1 | [4.2](#42-info--设备信息上报) |
| `reset` | 设备主动重置 | 0 | [4.3](#43-reset--设备主动重置) |
| `time` | 获取云端时间 | 1 | [4.4](#44-time--获取云端时间) |
| `model` | 获取物模型 | 1 | [4.5](#45-model--获取物模型) |
| `property_report` | 属性上报 | 0 | [4.6](#46-property_report--属性上报) |
| `ota_progress` | OTA 进度上报 | 0 | [4.7](#47-ota_progress--ota-进度上报) |
| `agora_agent_device_access` | 请求 AI 语音接入 | 1 | [4.8](#48-agora_agent_device_access--普通设备请求-ai-接入) |
| `agora_agent_nfc_report` | NFC 设备请求 AI 接入 | 1 | [4.9](#49-agora_agent_nfc_report--nfc-设备请求-ai-接入) |

### 云端下发 (issue)

| code | 说明 | 参考章节 |
|:---|:---|:---|
| `reset` | 远程重置设备 | [5.1](#51-reset--远程重置设备) |
| `ota` | 固件升级 | [5.2](#52-ota--固件升级) |
| `ping` | 在线检测 | [5.3](#53-ping--在线检测) |
| `property_set` | 设置属性 | [5.4](#54-property_set--设置属性) |

---

## 7. 集成注意事项

以下要点是基于 BK7258 + ali_mqtt 客户端的实际集成经验，其他 MQTT 客户端（如 paho、mosquitto）通常不会遇到。

> **参考实现**：
> - 详细原始记录：[`sentino-iot-sample/device/BUILD_GUIDE.md` §3.2「Sentino 集成踩坑记录」](https://github.com/sentino-jp/sentino-iot-sample/blob/main/device/BUILD_GUIDE.md#32-sentino-iot-模式)
> - MQTT 客户端代码：[`sentino-iot-sample/device/projects/common_components/sentino_iot_sdk/sentino_mqtt/sentino_mqtt.c`](https://github.com/sentino-jp/sentino-iot-sample/blob/main/device/projects/common_components/sentino_iot_sdk/sentino_mqtt/sentino_mqtt.c)

### 7.1 Username 必须包含签名方法和时间戳分隔符

Username 不是简单的 `uuid`，必须按 `${uuid}|signMethod=${signMethod},ts=${ts}` 拼接，否则 broker 返回 `CONNACK not_authorized`（rc=5）。完整格式见 §1。

### 7.2 ali_mqtt 客户端需要先初始化设备信息

如果使用阿里云 `ali_mqtt`（`iot-c-sdk`）作为 MQTT 客户端，必须在 `IOT_MQTT_Construct()` 之前先调用：

```c
iotx_device_info_init();
iotx_device_info_set(&device_info);
```

否则签名所需的随机种子生成失败（`pdev = nil`），导致连接失败。其他 MQTT 客户端不受影响。

### 7.3 Broker 兼容 MQTT 3.1.1，但建议指定 5.0

Sentino broker 同时接受 MQTT 5.0 和 3.1.1 客户端。如使用 `ali_mqtt`（协议级别 4，即 3.1.1）实测可正常连接。偶发连接不稳定时重启设备即可恢复。

---

**相关文档**：[架构与概念](../architecture.md) | [快速入门 — 设备端](../tutorials/quickstart-device.md) | [设备端集成指南](../guides/guide-device.md)
