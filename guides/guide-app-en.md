# App Integration Guide

> **TL;DR**: This document covers the complete App-side integration: user login, BLE provisioning (WiFi / 4G modes), device management, and agent management. Follow this document to complete App-side development.

> **Prerequisites**: We recommend reading [Architecture & Concepts](../architecture-en.md) first. For API field details, refer to [REST API Reference](../reference/ref-rest-api-en.md).

---

## 1. Integration Overview

Features the App side needs to implement:

```mermaid
graph LR
    A["User Login"] --> B["Get Account ID"]
    B --> C["BLE Provisioning"]
    C --> D["Poll Binding Status"]
    D --> E["Device Management"]
    E --> F["Agent Management"]
```

| Feature Module | Description |
|---|---|
| User Login | Register/login via UID, obtain access_token |
| Get Account ID | Retrieve user account structure; `assetId` is required for device binding |
| BLE Provisioning | Transfer binding info (+ WiFi credentials) to the device over Bluetooth |
| Device Management | Device list, binding status query, unbind, OTA check |
| Agent Management | Browse/create AI characters, bind a character to a device |

---

## 2. User Login

### 2.1 Login Method

Sentino uses **UID authorization**, which combines registration and login:
- UID does not exist → automatically register a new user
- UID already exists → log in directly

### 2.2 Login Flow

```
POST /auth/oauth/token
```

**Request**:

```bash
curl -X POST "https://api-iot.sentino.jp/api/auth/oauth/token?grant_type=uid&area_code=86&app_id=krkfvb4s5e91hq" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==" \
  -H "client_id: Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==" \
  -H "app_id: krkfvb4s5e91hq" \
  -H "channel_identifier: kfvb4s5e" \
  -H "package_name: jp.sentino.general" \
  -H "encrypt_type: AES/ECB/PKCS5Padding" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "grant_type=uid&uid=YOUR_UID&password=YOUR_PASSWORD&area_code=86&user_country_key=CN"
```

> **Note**: The login endpoint also requires the public request headers; otherwise, the issued Token will fail validation on subsequent business APIs.

**Response**:

```json
{
  "code": 200,
  "data": {
    "access_token": "6ea8368a-127c-4203-b7e8-83fbeb9d0239",
    "token_type": "bearer",
    "expires_in": 2591999,
    "refresh_token": "...",
    "userId": "cn2042488223219761152",
    "username": "test_user_001"
  }
}
```

### 2.3 Token Usage

After login, all business APIs must include the following request header:

```
Authorization: Bearer {access_token}
```

The Token is valid for approximately 30 days (`expires_in` in seconds). After expiration, use `refresh_token` to refresh it.

### 2.4 Public Request Headers

All endpoints (including login) must include the following public request headers:

| Header | Value | Description |
|---|---|---|
| `timezone` | `Asia/Shanghai` | User timezone |
| `language` | `zh_CN` | Language |
| `data_center_code` | `cn` | Data center code |
| `client_id` | `Y2V0dXMtaW90LWFwcDpv...` | Client identifier |
| `encrypt_type` | `AES/ECB/PKCS5Padding` | Encryption type |
| `channel_identifier` | `kfvb4s5e` | Channel identifier |
| `package_name` | `jp.sentino.general` | Package name |
| `app_id` | `krkfvb4s5e91hq` | Application ID |

---

## 3. BLE Provisioning

Provisioning is the process by which a user binds a device to their account on first use. The App transfers the binding info to the device via BLE Bluetooth.

> **Reference implementation**:
> - BLE scan / connect / GATT write: [`sentino-app-sample/lib/services/ble_service.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/services/ble_service.dart)
> - V1 packet protocol (Dart): [`sentino-app-sample/lib/utils/generic_ble_packet_protocol.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/generic_ble_packet_protocol.dart)
> - BLE advertisement parsing: [`sentino-app-sample/lib/utils/generic_ble_advertisement_parser.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/generic_ble_advertisement_parser.dart)
> - Web BLE tool (browser implementation, easier to read): [`sentino-iot-sample/web-app/js/ble.js`](https://github.com/sentino-jp/sentino-iot-sample/blob/main/web-app/js/ble.js)

### 3.1 Pre-Provisioning Preparation

Before initiating BLE provisioning, the App needs to obtain the following information:

```mermaid
flowchart TD
    A["1. User logs in and obtains Token"] --> B["2. Get account ID (assetId)"]
    B --> C["3. Scan device barcode, obtain UUID"]
    C --> D["4. Query device info (confirm provisioning is allowed)"]
    D --> E["5. Request provisioning data encryption"]
    E --> F["6. BLE scan and connect to device"]
    F --> G["7. BLE send provisioning info"]
    G --> H["8. Poll binding status"]
```

**Get account ID**:

```
POST /business-app/v1/asset/assetTree
```

Returns the user's account structure. Record the root node's `assetId` (account ID) for use in binding.

**Get device info** (optional, used to confirm device status):

```
POST /business-app/v1/device/getSimpleDeviceInfo
Body: {"productId": "sEF4ljjdH8mo", "uuid": "ct01wfjSNqGAqUUK"}
```

### 3.2 WiFi Mode Provisioning

WiFi mode requires the WiFi credentials and binding info to be passed to the device together.

#### Step 1: Request Provisioning Data Encryption

```
POST /business-app/v1/distributionNet/dataEncrypt
```

```json
{
  "content": "{\"sid\":\"MyWiFi\",\"pw\":\"wifi_password\",\"bid\":\"assetId\",\"userId\":\"userId\",\"mq\":\"mqtt-iot.sentino.jp\",\"port\":1883,\"mqttSslPort\":\"8883\",\"country\":\"CN\",\"areaCode\":\"CN\",\"tz\":\"Asia/Shanghai\",\"force_bind\":true}",
  "encryptType": 0,
  "protocol": "1",
  "type": "thing.network.set"
}
```

| content field | Description |
|---|---|
| `sid` | WiFi SSID |
| `pw` | WiFi password |
| `bid` | Account ID (assetId) |
| `userId` | User ID |
| `mq` | MQTT Broker address (source: `mqttUrl` field from the data center API) |
| `port` | MQTT port |
| `mqttSslPort` | MQTT SSL port (source: `mqttSslPort` field from the data center API) |
| `country` | Country code |
| `areaCode` | Area code (source: `countryCode` field from user info) |
| `tz` | Timezone |
| `force_bind` | Whether to force binding |

#### Step 2: BLE Scan and Connect to Device

| BLE Parameter | Value |
|---|---|
| Advertising name | `"RY"` (default value; can be customized per product) |
| Advertising data contains | Service UUID `0xA101` + Product ID (PID) |
| Scan response data contains | Device UUID/MAC, provisioning status flag |

The App scans for BLE devices, filters by advertising name (default `"RY"`; the actual name follows the product definition), and after connecting:

1. First send `device.information.get` to obtain device info
2. After confirming the device is provisionable, send the provisioning info

#### Step 3: Get Device Info

Send via BLE GATT:

```json
{
  "type": "device.information.get",
  "ts": 1742536800
}
```

The device replies:

```json
{
  "type": "device.information.get.response",
  "ts": 1742536800,
  "data": {
    "version": "1.0.0",
    "pid": "sEF4ljjdH8mo",
    "bind": false,
    "wifi_mac": "aa:bb:cc:dd:ee:ff",
    "ble_mac": "11:22:33:44:55:66"
  }
}
```

#### Step 4: Send Provisioning Info

Send the encrypted provisioning data (or send the plaintext data directly):

```json
{
  "type": "thing.network.set",
  "data": {
    "sid": "MyWiFi",
    "pw": "wifi_password",
    "bid": "assetId",
    "userId": "userId",
    "mq": "mqtt-iot.sentino.jp",
    "port": 1883,
    "country": "CN",
    "tz": "Asia/Shanghai",
    "force_bind": true
  }
}
```

The device replies:

```json
{
  "type": "thing.network.set.response",
  "code": 0,
  "ts": 1742536800
}
```

A `code` of `0` indicates the device has received the provisioning info and is starting to connect to WiFi.

#### Step 5: Poll Binding Status

After sending the provisioning info, the App needs to poll to check whether the device has been bound successfully:

```
POST /business-app/v1/device/bind/checkBindResult/{uuid}
```

- Polling interval: 10 seconds
- Timeout: 120 seconds (max 12 attempts)
- A `data` value of `0` indicates a successful binding

**Option 2: Listen to MQTT messages (recommended)**

The App can also listen on the MQTT message channel to receive device binding result push notifications, eliminating the need for polling:

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "bind_result",
  "data": {
    "devices": [
      {
        "deviceId": "device ID",
        "uuid": "device UUID",
        "gatewayId": "gateway device ID",
        "type": "add",
        "result": "success",
        "msg": "binding succeeded"
      }
    ]
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data.devices[].deviceId` | string | Device ID |
| `data.devices[].uuid` | string | Device UUID |
| `data.devices[].type` | string | Operation type (`add` = add) |
| `data.devices[].result` | string | Result (`success` / `fail`) |

### 3.3 4G Mode Provisioning

4G mode is simpler: the device connects to the network as soon as it powers on, and BLE only carries the binding info.

```json
{
  "type": "thing.network.set",
  "data": {
    "bid": "assetId",
    "userId": "userId"
  }
}
```

**Differences from WiFi mode**:
- No need to pass `sid` (SSID), `pw` (password), `mq` (MQTT address), and similar fields
- No need for the user to select a WiFi network
- Faster provisioning experience (scan code → BLE → done)

### 3.4 BLE Data Transmission Notes

BLE data transmission between the App and the device uses the Rlink BLE V1 protocol:

| Parameter | Value |
|---|---|
| Max size per packet | 128 bytes |
| Protocol header size | 10 bytes |
| Max payload | 118 bytes/packet |
| Inter-packet delay | 10 ms |
| Application-layer format | JSON |

JSON messages larger than 118 bytes are automatically split into multiple packets. For the detailed packet-splitting protocol, refer to [BLE Protocol Reference](../reference/ref-ble-en.md).

### 3.5 Provisioning Sequence Overview

```mermaid
sequenceDiagram
    participant App as Management App
    participant Device as IoT Device
    participant Cloud as Sentino IoT Platform

    App->>Cloud: Login and obtain Token
    App->>Cloud: Get account ID (assetId)
    Note over App: User scans device barcode

    App->>Device: BLE scan (advertising name: "RY")
    App->>Device: BLE connect
    App->>Device: BLE: device.information.get
    Device-->>App: BLE: device info (pid, uuid, bind status)

    alt WiFi mode
        Note over App: User selects WiFi and enters password
        App->>Cloud: Request provisioning data encryption
        Cloud-->>App: Return encrypted data
        App->>Device: BLE: thing.network.set (WiFi + binding info)
    else 4G mode
        App->>Device: BLE: thing.network.set (binding info only)
    end

    Device-->>App: BLE: thing.network.set.response (code=0)

    Device->>Cloud: MQTT connect + report bind
    Cloud-->>Device: Binding succeeded

    loop Every 10 seconds
        App->>Cloud: Poll checkBindResult
    end
    Cloud-->>App: Binding succeeded (data=0)
    App->>App: Show "Added successfully"
```

---

## 4. Device Management

### 4.1 Get Device List

Retrieve all of the user's devices:

```
POST /business-app/v1/device/getHomeDeviceAndGroupList
Body: {"assetIds": ["assetId1", "assetId2"]}
```

Key fields in the response:

| Field | Description |
|---|---|
| `deviceList` | Device info list |
| `deviceList[].deviceId` | Device ID (used for subsequent operations) |
| `deviceList[].uuid` | Device UUID |
| `deviceList[].online` | Whether the device is online |
| `sortIdList` | User-defined sort order |
| `shareList` | List of shared devices |

### 4.2 Unbind Device

```
POST /business-app/v1/device/bind/unbind
Body: {"deviceId": "device ID", "isCleanData": 1}
```

| `isCleanData` | Description |
|---|---|
| `0` | Do not clear device data |
| `1` | Clear device data (factory reset) |

### 4.3 OTA Upgrade Check

```
POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}
```

Response:

```json
{
  "code": 200,
  "data": {
    "hasUpgrade": true,
    "version": "1.2.0",
    "downloadUrl": "https://...",
    "upgradeDesc": "Fixes known issues"
  }
}
```

---

## 5. Agent Management

### 5.1 Browse Official Agents

```
POST /business-app/v1/agents/recommend/agents-list
Body: {}
```

Returns the list of officially recommended AI characters. Each character includes name, avatar, description, tags, and similar information.

### 5.2 View Agent Details

```
POST /business-app/v1/agents/detail?agentId={agentId}
```

Returns the Agent's full configuration, including LLM model, TTS voice, and so on.

### 5.3 Bind Agent to Device

```
POST /business-app/v1/agents/device/bind-agent
Body: {
  "agentId": "agent ID",
  "agentType": "official",
  "deviceId": "device ID"
}
```

| `agentType` | Description |
|---|---|
| `official` | Official Agent |
| `customize` | User-customized Agent |

After binding, the device will use the specified Agent.

### 5.4 Agent Management Flow

```mermaid
flowchart TD
    A["Get recommended list"] --> B{"User selects"}
    B -->|"Choose official character"| C["View character details"]
    B -->|"Choose custom character"| D["View custom character details"]
    C --> E["Bind to device"]
    D --> E
    E --> F["Device uses new character"]
```

---

## 6. Error Handling

### 6.1 HTTP Error Codes

| Error Code | Description | How to Handle |
|---|---|---|
| 200 | Success | — |
| 400 | Invalid request parameters | Check the request body |
| 401 | Token invalid or expired | Refresh the Token or log in again |
| 403 | Forbidden | Check permissions |
| 404 | Resource not found | Check whether the ID is correct |
| 500 | Internal server error | Retry or contact technical support |

### 6.2 Provisioning Failure Handling

| Failure Scenario | How to Handle |
|---|---|
| BLE connection failed | Confirm the device is in provisioning mode and rescan |
| `thing.network.set` reply has non-zero `code` | Check the format of the data sent |
| Polling timeout (not bound within 120 seconds) | Prompt the user to retry; check the network connection |
| WiFi connection failed | Confirm the SSID and password are correct, and that the router supports 2.4 GHz |

---

**Related Documents**: [Architecture & Concepts](../architecture-en.md) | [BLE Protocol Reference](../reference/ref-ble-en.md) | [REST API Reference](../reference/ref-rest-api-en.md)
