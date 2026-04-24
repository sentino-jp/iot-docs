# REST API Reference

> **TL;DR**: Complete reference for the Sentino IoT platform REST API. Covers authentication, provisioning, device management, and Agent management — 22 endpoints in total, each accompanied by a curl example.

---

## 1. Overview

| Item | Value |
|---|---|
| Base URL | `https://api-iot.sentino.jp` |
| Authentication | Bearer Token (the login endpoint uses Basic Auth) |

### 1.1 Environment Variable Setup

All curl examples in this document use environment variables. Before running them, export the following variables (fill in values for your test environment):

```bash
# Application credentials (provided by Sentino, unique per customer application)
export APP_ID="krkfvb4s5e91hq"
export CHANNEL_IDENTIFIER="kfvb4s5e"
export PACKAGE_NAME="jp.sentino.general"
export DATA_CENTER_CODE="cn"

# Fixed values (copy as is)
export CLIENT_ID="Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg=="
export ENCRYPT_TYPE="AES/ECB/PKCS5Padding"

# Client attributes
export TIMEZONE="Asia/Shanghai"
export LANGUAGE="zh_CN"

# User login credentials (replace as needed)
export UID="test_user_001"
export AREA_CODE="86"
export USER_COUNTRY_KEY="CN"

# Obtained after login (see §3.1 User Login)
export TOKEN=""
export USER_ID=""

# Business context (fill in based on your test device/account)
export PRODUCT_ID="vqB8C7fniWRLWL"   # Current mock device PID
export UUID="ct01kQBXBK7h63H8"        # Current mock device UUID
export ASSET_ID=""                    # Obtained via §5.1
export DEVICE_ID=""                   # Obtained via §5.4
export AGENT_ID=""                    # Obtained via §6.1 / §6.3
export NFC_UUID=""                    # Physical NFC card UUID
```

> All subsequent curl examples assume the above variables are exported. To switch data center or test device, simply change the environment variables.

---

## 2. Authentication

### 2.1 Obtaining the Token

Obtain an `access_token` via the login endpoint (see the `expires_in` field in the login response for validity).

### 2.2 Using the Token

All business endpoints (except login) must include the following request header:

```
Authorization: Bearer {access_token}
```

### 2.3 Common Request Headers

All endpoints (including login) must include the following request headers.

**Provided by Sentino (unique per customer application — contact Sentino to obtain)**:

| Header | Description |
|--------|------|
| `app_id` | Application identifier |
| `channel_identifier` | Channel identifier |
| `package_name` | App package name |
| `data_center_code` | Data center (depends on deployment region) |

**Fixed values (copy as is)**:

| Header | Value |
|--------|-----|
| `client_id` | `Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |
| `encrypt_type` | `AES/ECB/PKCS5Padding` |

**Client settings (based on the user's actual context)**:

| Header | Default | Description |
|--------|--------|------|
| `timezone` | `Asia/Shanghai` | User timezone |
| `language` | `zh_CN` | User language |

### 2.4 Common Response Format

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

---

## Endpoint Quick Reference

Grouped by business area — click to jump to the corresponding section.

### User Authentication

| § | Endpoint | Method + Path |
|---|---|---|
| [3.1](#31-user-login-authorization) | User login (registration/authorization combined) | `POST /auth/oauth/token` |

### Provisioning

| § | Endpoint | Method + Path |
|---|---|---|
| [4.1](#41-get-product-info) | Get product info (query provisioning mode) | `POST /business-app/v1/product/getByProductId` |
| [4.2](#42-encrypt-provisioning-data) | Encrypt provisioning data | `POST /business-app/v1/distributionNet/dataEncrypt` |
| [4.3](#43-get-data-center-list) | Get data center list | `POST /business-app/v1/common/getDataCenterList` |
| [4.4](#44-query-device-binding-status) | Query device binding status (polling) | `POST /business-app/v1/device/bind/checkBindResult/{uuid}` |

### Device Management

| § | Endpoint | Method + Path |
|---|---|---|
| [5.1](#51-get-account-id) | Get account ID (asset tree) | `POST /business-app/v1/asset/assetTree` |
| [5.2](#52-get-device-info) | Get device info | `POST /business-app/v1/device/getSimpleDeviceInfo` |
| [5.3](#53-get-product-categories) | Get product categories | `POST /business-app/v1/category/top` |
| [5.4](#54-get-device-list) | Get device list | `POST /business-app/v1/device/getHomeDeviceAndGroupList` |
| [5.5](#55-ota-upgrade-check) | OTA upgrade check | `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{type}` |
| [5.6](#56-unbind-device) | Unbind device | `POST /business-app/v1/device/bind/unbind` |

### Agent Management

| § | Endpoint | Method + Path |
|---|---|---|
| [6.1](#61-get-recommended-agent-list) | Recommended Agent list | `POST /business-app/v1/agents/recommend/agents-list` |
| [6.2](#62-get-agent-detail) | Agent detail | `POST /business-app/v1/agents/detail` |
| [6.3](#63-get-customized-agent-list) | Customized Agent list | `POST /business-app/v1/agents/customize/agents-list` |
| [6.4](#64-create-customized-agent) | Create customized Agent | `POST /business-app/v1/agents/customize/create` |
| [6.5](#65-delete-customized-agent) | Delete customized Agent | `POST /business-app/v1/agents/customize/deleteById` |
| [6.6](#66-get-recommended-sentino-agent-list) | Sentino Agent list | `POST /business-app/v1/sentino-agents/recommend/agents-list` |
| [6.7](#67-get-sentino-agent-detail) | Sentino Agent detail | `POST /business-app/v1/sentino-agents/detail` |
| [6.8](#68-get-user-bound-agent-list) | Bound Agent list | `POST /business-app/v1/user-agents/list` |
| [6.9](#69-bind-agent-to-device) | Bind Agent to device | `POST /business-app/v1/agents/device/bind-agent` |
| [6.10](#610-nfc-card-list) | NFC card list | `POST /business-app/v1/agents/nfc/list` |
| [6.11](#611-bind-agent-to-nfc-card) | Bind Agent to NFC card | `POST /business-app/v1/agents/nfc/bind-agent` |

---

## 3. User Authentication Endpoints

### 3.1 User Login (Authorization)

Registration and login are combined; authorization uses the UID method. If the UID does not exist, the user is automatically registered.

```
POST /auth/oauth/token
```

**Request headers**:

| Header | Value |
|---|---|
| Content-Type | `application/x-www-form-urlencoded` |
| Authorization | `Basic $CLIENT_ID` |
| Common headers | All common headers in §2.3 must be included (`client_id`, `app_id`, etc.) |

> **Note**: The login endpoint also requires the common request headers; otherwise the issued token will fail validation on subsequent business endpoints.

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | Yes | Fixed value: `uid` |
| `area_code` | string | Yes | Area code, default `86` |
| `anonymousId` | string | No | Anonymous ID |
| `app_id` | string | Yes | Application ID |

**Body parameters** (application/x-www-form-urlencoded):

| Parameter | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | Yes | Fixed value: `uid` |
| `uid` | string | Yes | Unique user identifier, max 50 characters |
| `password` | string | Yes | User password |
| `area_code` | string | Yes | Area code, default `86` |
| `user_country_key` | string | Yes | Country code, default `CN` |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=$AREA_CODE&app_id=$APP_ID" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Basic $CLIENT_ID" \
  -H "client_id: $CLIENT_ID" \
  -H "app_id: $APP_ID" \
  -H "channel_identifier: $CHANNEL_IDENTIFIER" \
  -H "package_name: $PACKAGE_NAME" \
  -H "encrypt_type: $ENCRYPT_TYPE" \
  -H "timezone: $TIMEZONE" \
  -H "language: $LANGUAGE" \
  -H "data_center_code: $DATA_CENTER_CODE" \
  -d "grant_type=uid&uid=$UID&password=test123456&area_code=$AREA_CODE&user_country_key=$USER_COUNTRY_KEY"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "access_token": "6ea8368a-127c-4203-b7e8-83fbeb9d0239",
    "token_type": "bearer",
    "expires_in": 2591999,
    "refresh_token": "...",
    "userId": "cn2042488223219761152",
    "memberId": "cn2042488223219761152",
    "tenantId": "1955088720032829440",
    "nickname": "z3dwog17",
    "username": "test_user_001"
  }
}
```

---

## 4. Provisioning Endpoints

### 4.1 Get Product Info

Query product configuration by product ID.

```
POST /business-app/v1/product/getByProductId
```

> **productId source**: Parsed from the Service Data field of the device's BLE advertisement (the PID field). See [BLE Protocol Reference §2.1 Advertisement Data](./ref-ble-en.md#21-广播数据).

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `productId` | string | Yes | Product ID (the PID from the BLE advertisement) |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=$PRODUCT_ID" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "id": "sEF4ljjdH8mo",
    "productName": "智能玩具",
    "model": "ST-001",
    "imageUrl": "https://...",
    "protocolType": "MQTT",
    "distributionNetMode": "1",
    "nodeType": 1,
    "bindMode": 1,
    "deviceShare": 1,
    "deviceUpgrade": 1,
    "status": 1
  }
}
```

**Field reference**:

| Field | Description |
|---|---|
| `protocolType` | Communication protocol: `MQTT` / `BLE` |
| `distributionNetMode` | Provisioning mode: `1`=WiFi+BLE, `2`=WiFi, `3`=BLE |
| `nodeType` | Node type: `1`=standard device, `2`=gateway, `3`=edge gateway, `4`=sub-device |
| `bindMode` | Binding mode: `1`=strong bind, `2`=weak bind |
| `deviceShare` | Whether device sharing is supported: `0`=no, `1`=yes |
| `deviceUpgrade` | Whether OTA upgrade is supported: `0`=no, `1`=yes |

---

### 4.2 Encrypt Provisioning Data

Encrypt the data payload used during the provisioning flow.

```
POST /business-app/v1/distributionNet/dataEncrypt
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `content` | string | Yes | Content to encrypt (a JSON string) |
| `encryptType` | int | Yes | Encryption type, fixed value `0` |
| `protocol` | string | Yes | Protocol version, fixed value `"1"` |
| `type` | string | Yes | Fixed value `"thing.network.set"` |

**Fields inside `content`**:

| Field | Type | Description |
|---|---|---|
| `force_bind` | boolean | Whether to force binding |
| `sid` | string | WiFi SSID |
| `pw` | string | WiFi password |
| `mq` | string | MQTT broker address |
| `mqttSslPort` | string | MQTT SSL port, default `"8883"` |
| `bid` | string | Account ID (assetId) |
| `port` | int | MQTT port, default `1883` |
| `country` | string | Country code: `CN` / `US` / `JP` / `EP` / `AU` |
| `userID` | string | User ID |
| `tz` | string | Timezone |
| `areaCode` | string | Area code |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/distributionNet/dataEncrypt" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d @- <<EOF
{
  "content": "{\"sid\":\"MyWiFi\",\"pw\":\"password\",\"bid\":\"$ASSET_ID\",\"userID\":\"$USER_ID\",\"mq\":\"mqtt-iot.sentino.jp\",\"port\":1883,\"country\":\"$USER_COUNTRY_KEY\",\"tz\":\"$TIMEZONE\",\"force_bind\":true}",
  "encryptType": 0,
  "protocol": "1",
  "type": "thing.network.set"
}
EOF
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": "{encrypted provisioning data string}"
}
```

---

### 4.3 Get Data Center List

Get the list of available data centers, including each center's API and MQTT addresses.

```
POST /business-app/v1/common/getDataCenterList
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/common/getDataCenterList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "dataCenterCode": "",
      "dataCenterName": "",
      "apiUrl": "",
      "mqttUrl": "",
      "mqttSslPort": ""
    }
  ]
}
```

> The response also contains additional fields such as `appId`, `areaId`, `createTime`, `dataCenterId`, `iotUrl`, `sort`, etc., which are omitted here.

---

### 4.4 Query Device Binding Status

Poll whether the device has been bound successfully during the provisioning flow.

```
POST /business-app/v1/device/bind/checkBindResult/{uuid}
```

**Path parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | Yes | Device UUID (the UUID from the triplet, obtained by scanning the device barcode) |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/checkBindResult/$UUID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": 0
}
```

| `data` Value | Description |
|---|---|
| `0` | Binding succeeded |
| Other | Binding not completed or failed |

**Usage**: After provisioning data has been sent, poll once every 10 seconds for up to 120 seconds (12 attempts).

---

## 5. Device Management Endpoints

### 5.1 Get Account ID

Retrieve the user's account structure; returns the assetId (account ID).

```
POST /business-app/v1/asset/assetTree
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "timezone: $TIMEZONE" \
  -H "language: $LANGUAGE" \
  -H "data_center_code: $DATA_CENTER_CODE" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "id": "2042488223647580161",
      "parentId": "0",
      "name": "My Home",
      "childrens": [],
      "sortNumber": 999,
      "isLock": false,
      "memberRole": 1,
      "isShare": false
    }
  ]
}
```

> **Note**: The `data` field of the response is an array. Use the `id` and `name` fields (not `assetId` / `assetName`); the child-node field is `childrens`. During provisioning, use the root node's `id` as the `assetId`.

---

### 5.2 Get Device Info

Query basic device information by UUID and product ID.

```
POST /business-app/v1/device/getSimpleDeviceInfo
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `productId` | string | Yes | Product ID |
| `uuid` | string | Yes | Device UUID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"productId\": \"$PRODUCT_ID\", \"uuid\": \"$UUID\"}"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "uuid": "ct01wfjSNqGAqUUK",
    "productId": "sEF4ljjdH8mo",
    "deviceName": "智能设备",
    "protocolType": "MQTT",
    "configMode": "BLE",
    "bindStatus": 0
  }
}
```

---

### 5.3 Get Product Categories

List the categories supported by the application (this endpoint is optional).

```
POST /business-app/v1/category/top
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/category/top" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "categoryId": "1001",
      "categoryName": "智能玩具",
      "supportProtocols": ["MQTT", "BLE"]
    }
  ]
}
```

---

### 5.4 Get Device List

Retrieve the user's full device and group list.

```
POST /business-app/v1/device/getHomeDeviceAndGroupList
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetIds` | string[] | Yes | Array of account IDs |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getHomeDeviceAndGroupList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"assetIds\": [\"$ASSET_ID\"]}"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "deviceList": [
      {
        "deviceId": "2008424975449309184",
        "uuid": "ct01wfjSNqGAqUUK",
        "deviceName": "智能设备",
        "online": true,
        "productId": "sEF4ljjdH8mo"
      }
    ],
    "sortIdList": ["2008424975449309184"],
    "shareGroupList": [],
    "shareList": []
  }
}
```

| Field | Description |
|---|---|
| `deviceList` | List of device information |
| `sortIdList` | User-defined ordering |
| `shareGroupList` | Groups shared with the user |
| `shareList` | Devices shared with the user |

---

### 5.5 OTA Upgrade Check

Check whether a firmware upgrade is available for a device.

```
POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}
```

**Path parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |
| `firmwareType` | string | No | Firmware type (if omitted, checks the module firmware by default) |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/ota/checkUpgrade/$DEVICE_ID/module" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "hasUpgrade": true,
    "version": "1.2.0",
    "downloadUrl": "https://...",
    "md5": "abc123...",
    "fileSize": 1024000,
    "upgradeDesc": "Bug fixes and stability improvements"
  }
}
```

---

### 5.6 Unbind Device

Unbind a device, with the option to clear its data.

```
POST /business-app/v1/device/bind/unbind
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |
| `isCleanData` | int | Yes | `1`=clear data, `0`=do not clear |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/unbind" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"deviceId\": \"$DEVICE_ID\", \"isCleanData\": 1}"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 6. Agent Management Endpoints

### 6.1 Get Recommended Agent List

Retrieve the list of officially recommended Agent templates.

```
POST /business-app/v1/agents/recommend/agents-list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "agentId": "1980570366486159361",
      "avatar": "https://...",
      "name": "小助手",
      "description": "我是你的智能助手",
      "tags": ["助手", "智能"]
    }
  ]
}
```

---

### 6.2 Get Agent Detail

Retrieve the detailed configuration of a specified Agent.

```
POST /business-app/v1/agents/detail
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/detail?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "agentId": "1980570366486159361",
    "avatar": "https://...",
    "name": "小助手",
    "description": "我是你的智能助手",
    "tags": ["助手", "智能"],
    "llmModel": "GPT-4",
    "ttsVoice": "女声-温柔"
  }
}
```

---

### 6.3 Get Customized Agent List

Retrieve the list of Agent templates customized by the user.

```
POST /business-app/v1/agents/customize/agents-list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/customize/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "agentId": "2000436153759151101",
      "name": "樱木花道",
      "avatar": "https://...",
      "description": "我是樱木花道，热爱篮球的少年",
      "langId": "1947925380891516929",
      "llmModelId": "1980896877851869185",
      "ttsVoiceId": "1947925380891516931"
    }
  ]
}
```

---

### 6.4 Create Customized Agent

Create a customized Agent template.

```
POST /business-app/v1/agents/customize/create
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Agent name |
| `description` | string | Yes | Agent description (character persona) |
| `avatarUrl` | string | Yes | Avatar URL |
| `llmModelId` | string | Yes | LLM model ID |
| `ttsVoiceId` | string | Yes | TTS voice ID |
| `langId` | string | Yes | Language ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/customize/create" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "樱木花道",
    "description": "我是樱木花道，热爱篮球的少年",
    "avatarUrl": "https://example.com/avatar.jpg",
    "llmModelId": "1980896877851869185",
    "ttsVoiceId": "1947925380891516931",
    "langId": "1947925380891516929"
  }'
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 6.5 Delete Customized Agent

Delete a specified customized Agent template.

```
POST /business-app/v1/agents/customize/deleteById
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/customize/deleteById?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 6.6 Get Recommended Sentino Agent List

Retrieve the list of officially recommended Sentino Agent templates. A Sentino Agent template defines the Sentino character associated with the Agent.

```
POST /business-app/v1/sentino-agents/recommend/agents-list
```

**Request headers** (in addition to the common headers):

| Header | Value | Required |
|---|---|---|
| `client_id` | Application client_id | Yes |
| `app_id` | Application app_id | Yes |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/sentino-agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "client_id: $CLIENT_ID" \
  -H "app_id: $APP_ID" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "agentId": "2000436153759152123",
      "avatar": "https://...",
      "name": "Sentino 助手",
      "description": "Sentino 智能体角色",
      "tags": ["Sentino"]
    }
  ]
}
```

---

### 6.7 Get Sentino Agent Detail

Retrieve detailed information for a specified Sentino Agent template.

```
POST /business-app/v1/sentino-agents/detail
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/sentino-agents/detail?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": {
    "agentId": "2000436153759152123",
    "avatar": "https://...",
    "name": "Sentino 助手",
    "description": "Sentino 智能体角色",
    "tags": ["Sentino"]
  }
}
```

---

### 6.8 Get User-Bound Agent List

Retrieve the list of Agents currently bound by the user.

```
POST /business-app/v1/user-agents/list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/user-agents/list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "agentId": "1980570366486159361",
      "avatar": "https://...",
      "name": "小助手",
      "description": "我是你的智能助手",
      "tags": ["助手"]
    }
  ]
}
```

---

### 6.9 Bind Agent to Device

Bind a device to a specified Agent template.

```
POST /business-app/v1/agents/device/bind-agent
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |
| `agentType` | string | Yes | `official`=official, `customize`=customized, `sentino`=Sentino Agent |
| `deviceId` | string | Yes | Device ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"agentId\": \"$AGENT_ID\", \"agentType\": \"official\", \"deviceId\": \"$DEVICE_ID\"}"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 6.10 NFC Card List

Read the list of NFC cards bound to the base-station device.

```
POST /business-app/v1/agents/nfc/list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/nfc/list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

```json
{
  "code": 200,
  "data": [
    {
      "nfcUuid": "1111111111",
      "agentId": "1947925380891516929",
      "agentType": "official"
    }
  ]
}
```

---

### 6.11 Bind Agent to NFC Card

Bind an NFC card to a specified Agent template.

```
POST /business-app/v1/agents/nfc/bind-agent
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `nfcUuid` | string | Yes | NFC card UUID |
| `agentId` | string | Yes | Agent ID |
| `agentType` | string | Yes | `official`=official, `customize`=customized |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/nfc/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"nfcUuid\": \"$NFC_UUID\", \"agentId\": \"$AGENT_ID\", \"agentType\": \"official\"}"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 7. Error Codes

| Code | Description |
|---|---|
| 200 | Success |
| 400 | Bad request parameters |
| 401 | Unauthorized (token invalid or expired) |
| 403 | Forbidden |
| 404 | Resource not found |
| 500 | Internal server error |

---

> The endpoint quick reference has been moved to the top of this document (after §2), grouped by business area with anchor links.

---

**Related documents**: [App Integration Guide](../guides/guide-app-en.md) | [Quick Start — App](../tutorials/quickstart-app-en.md) | [Architecture & Concepts](../architecture-en.md)
