# MQTT Protocol Reference

This document is the complete field-level reference for the Sentino MQTT protocol. For usage flows and integration methods, see the device-side integration guide.

> **Prerequisites**: Please first read [Architecture & Concepts](../architecture-en.md).

---

## 1. Connection Parameters

| Parameter | Format | Description |
|:---|:---|:---|
| Broker address | `mqtt-iot.sentino.jp` | Test environment |
| Port | `1883` | Plaintext, TLS not enabled |
| Protocol version | MQTT 5.0 | Broker is compatible with MQTT 3.1.1 clients (such as ali_mqtt protocol level 4) |
| Client ID | `rlink_${uuid}_V2` | Always ends with `_V2` |
| Username | `${uuid}\|signMethod=${signMethod},ts=${ts}` | `\|` is the delimiter |
| Password | `hmacSha256("uuid=${uuid},ts=${ts}", KEY)` | KEY is the device key from the triplet |
| signMethod | `hmacSha256` (recommended) or `hmacSha1` | — |
| Keep Alive | 60s (recommended) | — |
| QoS | 1 (recommended for publish) | — |

**Signature calculation:**

```
content  = "uuid=${uuid},ts=${ts}"     // ts is the current UNIX timestamp (seconds)
password = hmacSha256(content, KEY)     // KEY is the 32-character key from the triplet
```

---

## 2. Topic Definitions

**Variable definitions:**
- `${pid}` — Product ID or product model (third-party device model)
- `${uuid}` — Device UUID (unique per device, taken from the triplet)

| Direction | Topic | Description | Device Action |
|:---|:---|:---|:---|
| Device -> Cloud | `rlink/v2/${pid}/${uuid}/report` | Report events | Publish |
| Cloud -> Device | `rlink/v2/${pid}/${uuid}/report_response` | Reply to reports | **Subscribe** |
| Cloud -> Device | `rlink/v2/${pid}/${uuid}/issue` | Issue commands | **Subscribe** |
| Device -> Cloud | `rlink/v2/${pid}/${uuid}/issue_response` | Reply to commands | Publish |

> After connecting, the device must subscribe to `report_response` and `issue`.

---

## 3. Message Format

### 3.1 Report Message (Device -> Cloud, via `report`)

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "ts": 1742536800,
  "code": "event code",
  "data": {},
  "ack": 1
}
```

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `id` | string | Yes | Unique message ID. Must not repeat within a short time on the same device, otherwise the message is ignored |
| `ts` | int | Yes | Current UNIX timestamp (seconds) |
| `code` | string | Yes | Event code, see Section 4 |
| `data` | object | Yes | Report data, see each event definition |
| `ack` | int | Yes | `0` = no cloud reply needed; `1` = cloud reply required |

### 3.2 Cloud Reply (Cloud -> Device, via `report_response`)

Only replied when the report has `ack=1`.

```json
{
  "res": 0,
  "msg": "success",
  "id": "matches the report message id",
  "ts": 1742536800,
  "code": "matches the report event code",
  "data": {}
}
```

| Field | Type | Description |
|:---|:---|:---|
| `res` | int | `0` = success, non-`0` = failure |
| `msg` | string | Result description |
| `id` | string | Matches the report message id |
| `code` | string | Matches the report event code |
| `data` | object | Reply data, see each event definition |

### 3.3 Issued Command (Cloud -> Device, via `issue`)

```json
{
  "id": "message ID",
  "ts": 1742536800,
  "code": "command code",
  "data": {}
}
```

### 3.4 Device Reply to Command (Device -> Cloud, via `issue_response`)

```json
{
  "res": 0,
  "msg": "success",
  "id": "matches the issued message id",
  "ts": 1742536800,
  "code": "matches the issued command code",
  "data": {}
}
```

---

## 4. Device Report Events

### 4.1 bind — Device Binding

After obtaining `userId` / `assetId` from BLE, the device reports a binding request.

| Property | Value |
|:---|:---|
| code | `bind` |
| Recommended ack | `1` |

**Report data:**

```json
{
  "userId": "provisioning user ID",
  "assetId": "account ID",
  "version": "1.0.0",
  "mcuVersion": "1.0.0",
  "cleanData": false
}
```

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `userId` | string | Yes | User ID, passed in by the App via BLE |
| `assetId` | string | Yes | Account ID, passed in by the App via BLE (the `bid` field in the BLE message) |
| `version` | string | Yes | Firmware version |
| `mcuVersion` | string | No | MCU version |
| `cleanData` | boolean | No | Whether to clear data, defaults to `false` |

**Cloud reply data:** No additional fields. `res=0` indicates binding succeeded.

---

### 4.2 info — Device Info Report

> Strongly recommended to integrate. Without it, the device cannot receive pending messages that the cloud failed to deliver due to power loss or network outage.

| Property | Value |
|:---|:---|
| code | `info` |
| Recommended ack | `1` |

**Report data:**

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

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `checkPwd` | int | No | Whether to check undelivered passwords. `0` = no (default), `1` = yes |
| `bindStatus` | int | Yes | `0` = unbound, `1` = bound |
| `version` | string | Yes | Firmware version |
| `mcuVersion` | string | No | MCU version |
| `mcuBleVersion` | string | No | MCU Bluetooth version |
| `mcuZigbeeVersion` | string | No | MCU Zigbee version |
| `mcuMatterVersion` | string | No | MCU Matter version |
| `iccid` | string | No | ICCID (cellular SIM card) (optional) |
| `config` | string | No | Device network info, JSON string. Includes `ipaddr` (LAN IP), `currentSsid` (current WiFi), `wifiList` (backup network list) |

**Cloud reply data:**

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

| Field | Type | Description |
|:---|:---|:---|
| `bindStatus` | int | Cloud-side binding status: `0` = unbound, `1` = bound |
| `isClearData` | int | Whether data has been cleared: `0` = no, `1` = yes |
| `logUploadUrl` | string | Log upload URL |
| `tcpUrl` | string | Low-power TCP address |
| `logSwitch.enable` | int | Log switch: `0` = off, `1` = on |
| `logSwitch.ttltype` | int | Reporting mode (required when enable=1): `1` = single, `2` = recurring |
| `logSwitch.intervalTime` | int | Upload interval (required when enable=1), 1-360 minutes |

---

### 4.3 reset — Device-Initiated Reset

| Property | Value |
|:---|:---|
| code | `reset` |
| Recommended ack | `0` |

**Report data:**

```json
{
  "cleanData": false
}
```

| Field | Type | Description |
|:---|:---|:---|
| `cleanData` | boolean | Whether to clear data |

**Cloud reply data:** No additional fields (no reply when ack=0).

---

### 4.4 time — Get Cloud Time

| Property | Value |
|:---|:---|
| code | `time` |
| Recommended ack | `1` |

**Report data:** None (omit the data field or send an empty object).

**Cloud reply data:**

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

| Field | Type | Description |
|:---|:---|:---|
| `ts` | int | UTC timestamp (seconds) |
| `sys_tz` | string | System timezone identifier |
| `zone_offset` | int | Timezone offset (seconds), already includes DST offset |
| `local_date_time` | string | Local time (`yyyy-MM-dd HH:mm:ss`) |
| `is_dst` | boolean | Whether DST is currently in effect |
| `is_zone_dst` | boolean | Whether the region observes DST |
| `dst_start_ts` | int | DST start UTC timestamp (empty if no DST) |
| `dst_start_local_date_time` | string | DST start local time (empty if no DST) |
| `dst_end_ts` | int | DST end UTC timestamp (empty if no DST) |
| `dst_end_local_date_time` | string | DST end local time (empty if no DST) |

> When the user changes timezone, the cloud may proactively push a `time` reply via `report_response`, and the device must handle it.

---

### 4.5 model — Get Thing Model

| Property | Value |
|:---|:---|
| code | `model` |
| Recommended ack | `1` |

**Report data:**

```json
{
  "format": "complete"
}
```

| Field | Type | Description |
|:---|:---|:---|
| `format` | string | Requested format: `complete` (full), `simple` (compact), `mini` (minimal) |

#### Cloud Reply — complete (Full)

```json
{
  "format": "complete",
  "profile": {
    "version": "V1.0.0",
    "productId": "product ID"
  },
  "properties": [{
    "identifier": "brightness",
    "dpBusiId": "11",
    "name": "brightness",
    "accessMode": "rw",
    "required": true,
    "dataTypeStr": "int",
    "dataType": {
      "type": "int",
      "specs": {
        "min": "0",
        "max": "100",
        "unit": "%",
        "unitName": "percent",
        "step": "1"
      }
    }
  }],
  "events": []
}
```

**properties field descriptions:**

| Field | Description |
|:---|:---|
| `identifier` | Unique property identifier (unique within the Thing Model) |
| `dpBusiId` | Business ID |
| `name` | Property name |
| `accessMode` | Access type: `r` (read-only) or `rw` (read-write) |
| `required` | Whether this is a required standard property |
| `dataType.type` | Property type (see table below) |
| `dataType.specs` | Property specs |

**Property types (`dataType.type`):**

| Type | Description | specs-specific fields |
|:---|:---|:---|
| `int` | Integer | `min`, `max`, `unit`, `unitName`, `step` |
| `float` | Single-precision float | `min`, `max`, `unit`, `unitName`, `step` |
| `double` | Double-precision float | `min`, `max`, `unit`, `unitName`, `step` |
| `text` | String | `length` (max 10240) |
| `date` | Date (String, UTC milliseconds) | — |
| `bool` | Boolean (int, 0 or 1) | — |
| `enum` | Enum (int) | `enums` (list of enum values) |
| `struct` | Struct | `specs` array describing sub-fields |
| `array` | Array | `size` (max 512), `item.type` |

#### Cloud Reply — simple (Compact)

```json
{
  "format": "simple",
  "profile": { "version": "V1.0.0", "productId": "product ID" },
  "properties": [{
    "identifier": "brightness",
    "busiId": "11",
    "dataType": "int",
    "specs": { "min": "0", "max": "100", "step": "1" }
  }]
}
```

#### Cloud Reply — mini (Minimal)

```json
{
  "format": "mini",
  "profile": { "version": "V1.0.0", "productId": "product ID" },
  "properties": [{
    "i": "brightness",
    "b": "11",
    "t": "int"
  }]
}
```

| Field | Description |
|:---|:---|
| `i` | identifier (property identifier) |
| `b` | busiId (business ID) |
| `t` | type (property type) |

---

### 4.6 property_report — Property Report

| Property | Value |
|:---|:---|
| code | `property_report` |
| Recommended ack | `0` |

**Report data:**

```json
{
  "properties": {
    "color": "red",
    "brightness": 80
  }
}
```

| Field | Type | Description |
|:---|:---|:---|
| `properties` | object | Property key-value pairs |

**Cloud reply data:** No additional fields.

---

### 4.7 ota_progress — OTA Progress Report

| Property | Value |
|:---|:---|
| code | `ota_progress` |
| Recommended ack | `0` |

**Report data (common fields):**

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `resCode` | int | Yes | Status code, see table below |
| `type` | string | Yes | Stage type, see table below |
| `subPid` | string | No | Sub-device product ID (required for gateway sub-device upgrades) |
| `subUuid` | string | No | Sub-device UUID (required for gateway sub-device upgrades) |

**type values:**

| type | Description | Additional fields |
|:---|:---|:---|
| `downloading` | Downloading | `percent` (int, 0-100) |
| `burning` | Flashing | — |
| `report` | Report new version (upgrade complete) | `version`, `mcuVersion`, `mcuBleVersion`, `mcuZigbeeVersion`, `mcuMatterVersion` |
| `fail` | Upgrade failed | — |

**resCode status codes:**

| Value | Description |
|:---|:---|
| `0` | Success |
| `-1` | Download timeout |
| `-2` | File not found |
| `-3` | Signature expired |
| `-4` | MD5 mismatch |
| `-5` | Firmware update failed |
| `-6` | Update timeout |
| `-7` | Upgrade already in progress |

**Report example — downloading:**

```json
{
  "resCode": 0,
  "type": "downloading",
  "percent": 80
}
```

**Report example — upgrade complete:**

```json
{
  "resCode": 0,
  "type": "report",
  "version": "1.2.0",
  "mcuVersion": "1.0.0"
}
```

---

### 4.8 agora_agent_device_access — Standard Device Requests AI Access

| Property | Value |
|:---|:---|
| code | `agora_agent_device_access` |
| Recommended ack | `1` |

**Report data:** `{}` (empty object)

**Cloud reply data:**

```json
{
  "appId": "935b8af0c93640b09d614e48f1a8f75a",
  "rtcToken": "007eJxTYBA...",
  "channelName": "convo_stn_TgLI80UukY948M5v",
  "uid": 25532
}
```

| Field | Type | Description |
|:---|:---|:---|
| `appId` | string | Agora app ID |
| `rtcToken` | string | RTC channel token |
| `channelName` | string | RTC channel name |
| `uid` | int | RTC member ID |


---

### 4.9 agora_agent_nfc_report — NFC Device Requests AI Access

| Property | Value |
|:---|:---|
| code | `agora_agent_nfc_report` |
| Recommended ack | `1` |

**Report data:**

```json
{
  "nfcIdentifier": "NFC card unique identifier",
  "onlyReport": 0
}
```

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `nfcIdentifier` | string | Yes | NFC card unique identifier |
| `onlyReport` | int | Yes | `1` = report only (no RTC returned), `0` = report and return RTC info |

**Cloud reply data (when onlyReport=0):**

Same format as the `agora_agent_device_access` reply (`appId`, `rtcToken`, `channelName`, `uid`).

---

## 5. Cloud-Issued Commands

### 5.1 reset — Remote Device Reset

| Property | Value |
|:---|:---|
| code | `reset` |

**Issued data:**

```json
{
  "clearData": true
}
```

| Field | Type | Description |
|:---|:---|:---|
| `clearData` | boolean | Whether to clear device data |

**Device reply data:** No additional fields.

---

### 5.2 ota — Firmware Update

| Property | Value |
|:---|:---|
| code | `ota` |

**Issued data:**

```json
{
  "subUuids": ["subDeviceUUID1", "subDeviceUUID2"],
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

| Field | Type | Required | Description |
|:---|:---|:---|:---|
| `subUuids` | string[] | No | Sub-device UUID list (required for gateway sub-device upgrades) |
| `firmwareType` | int | Yes | Firmware type (see table below) |
| `fileSize` | int | Yes | File size (bytes) |
| `silence` | boolean | Yes | Whether to perform a silent update |
| `md5sum` | string | Yes | File MD5 checksum |
| `url` | string | Yes | Firmware download URL |
| `version` | string | Yes | Target version |
| `options.stable` | string | No | Upgrade partition: `usr` / `recovery` / `kernel` |

**firmwareType values:**

| Value | Description |
|:---|:---|
| `1` | Factory firmware |
| `2` | Standard firmware |
| `3` | MCU firmware |
| `4` | MCU Bluetooth firmware |
| `5` | MCU Zigbee firmware |
| `6` | MCU Matter firmware |

**Device reply data:** No additional fields.

---

### 5.3 ping — Online Check

| Property | Value |
|:---|:---|
| code | `ping` |

**Issued data:** None.

**Device reply data:**

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

| Field | Type | Description |
|:---|:---|:---|
| `currentSsid` | string | Currently connected WiFi name |
| `wifiList` | array | List of available networks; each entry contains `ssid` and `rssi` signal strength |
| `ipaddr` | string | LAN IP (if omitted, the backend uses the public IP) |
| `networkType` | int | `1` = WiFi, `2` = wired |
| `signal` | int | `1` = good (10-50ms), `2` = medium (50-100ms), `3` = poor (>100ms) |
| `signalValue` | int | Signal strength percentage, 0-100 |

---

### 5.4 property_set — Set Properties

| Property | Value |
|:---|:---|
| code | `property_set` |

**Issued data:**

```json
{
  "properties": {
    "color": "blue",
    "brightness": 60
  }
}
```

| Field | Type | Description |
|:---|:---|:---|
| `properties` | object | Property key-value pairs |

**Device reply data:**

```json
{
  "properties": {
    "color": "blue",
    "brightness": 60
  }
}
```

---

## 6. Event Code Quick Reference

### Device Reports (report)

| code | Description | Recommended ack | Reference |
|:---|:---|:---|:---|
| `bind` | Device binding | 1 | [4.1](#41-bind--device-binding) |
| `info` | Device info report | 1 | [4.2](#42-info--device-info-report) |
| `reset` | Device-initiated reset | 0 | [4.3](#43-reset--device-initiated-reset) |
| `time` | Get cloud time | 1 | [4.4](#44-time--get-cloud-time) |
| `model` | Get Thing Model | 1 | [4.5](#45-model--get-thing-model) |
| `property_report` | Property report | 0 | [4.6](#46-property_report--property-report) |
| `ota_progress` | OTA progress report | 0 | [4.7](#47-ota_progress--ota-progress-report) |
| `agora_agent_device_access` | Request AI voice access | 1 | [4.8](#48-agora_agent_device_access--standard-device-requests-ai-access) |
| `agora_agent_nfc_report` | NFC device requests AI access | 1 | [4.9](#49-agora_agent_nfc_report--nfc-device-requests-ai-access) |

### Cloud Issues (issue)

| code | Description | Reference |
|:---|:---|:---|
| `reset` | Remote device reset | [5.1](#51-reset--remote-device-reset) |
| `ota` | Firmware update | [5.2](#52-ota--firmware-update) |
| `ping` | Online check | [5.3](#53-ping--online-check) |
| `property_set` | Set properties | [5.4](#54-property_set--set-properties) |

---

## 7. Integration Notes

The following notes are based on real-world integration experience with BK7258 + the ali_mqtt client; other MQTT clients (such as paho or mosquitto) typically do not encounter these issues.

### 7.1 Username Must Include the Sign Method and Timestamp Delimiters

Username is not simply `uuid`—it must be assembled as `${uuid}|signMethod=${signMethod},ts=${ts}`, otherwise the broker returns `CONNACK not_authorized` (rc=5). For the complete format, see §1.

### 7.2 The ali_mqtt Client Requires Device Info Initialization First

If you use Alibaba Cloud's `ali_mqtt` (`iot-c-sdk`) as the MQTT client, you must call the following before `IOT_MQTT_Construct()`:

```c
iotx_device_info_init();
iotx_device_info_set(&device_info);
```

Otherwise generation of the random seed required for signing fails (`pdev = nil`), causing connection to fail. Other MQTT clients are not affected.

### 7.3 Broker Is Compatible with MQTT 3.1.1, but Specifying 5.0 Is Recommended

The Sentino broker accepts both MQTT 5.0 and 3.1.1 clients. Connection has been verified to work normally with `ali_mqtt` (protocol level 4, i.e., 3.1.1). For occasional connection instability, restarting the device restores normal operation.

---

**Related documents**: [Architecture & Concepts](../architecture-en.md) | [Quick Start — Device Side](../tutorials/quickstart-device-en.md) | [Device-Side Integration Guide](../guides/guide-device-en.md)
