# REST API Reference

> **TL;DR**: Complete reference for the Sentino IoT platform REST API. Covers authentication, provisioning, device management, and Agent management — 40+ endpoints in total, each accompanied by a curl example.

---

## 1. Overview

| Item | Value |
|---|---|
| Base URL | `https://api-iot.sentino.jp` |
| Authentication | Bearer Token (the login endpoint uses Basic Auth) |

> **Reference implementation**: Every endpoint has a corresponding Dart implementation, organized by business area into 4 repository files:
> - [`api_auth_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_auth_repository.dart) — §3 Authentication / Account
> - [`api_device_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_device_repository.dart) — §4 Provisioning + §5 Device Management
> - [`api_agent_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_agent_repository.dart) — §6 Agent Management
> - [`api_ota_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_ota_repository.dart) — OTA
>
> Common header injection: [`lib/utils/api_client.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/api_client.dart); constants: [`lib/utils/app_config.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/app_config.dart)

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
| `language` | `zh_CN` | User language (`zh_CN` / `en_US` / `ja_JP`) |

**Client environment identification (injected on every request)**:

| Header | Example | Description |
|--------|---------|------|
| `os_name` | `android` / `ios` / `windows` | Operating system type |
| `version` | `1.0.0-2604131058` | App version (used by the server for compatibility checks) |
| `devid` | androidId / IDFV | Unique device identifier |
| `ua` | Base64-encoded | Format: `brand\|model\|os\|resolution\|deviceName` |
| `request_id` | UUID v4 | Unique ID per request (for server-side log tracing) |

> These 5 headers are injected automatically by the client SDK. If you implement your own HTTP client (instead of the Flutter template), construct them manually.

### 2.4 Common Response Format

```json
{
  "reqId": "a3f8b2c1-...",
  "time": 1742536800,
  "code": 200,
  "message": "success",
  "data": {}
}
```

| Field | Type | Description |
|------|------|------|
| `reqId` | string | Trace ID assigned by the server for this request. Logging this value on the client is recommended for joint troubleshooting |
| `time` | int | Server processing timestamp (seconds) |
| `code` | int | Business code. `200` = success; for other values see [§7 Error Codes](#7-error-codes) |
| `message` | string | Result description |
| `data` | object/array/null | Business data; structure varies per endpoint |

---

## Endpoint Quick Reference

Grouped by business area — click to jump to the corresponding section.

### User Authentication / Account

| § | Endpoint | Method + Path |
|---|---|---|
| [3.1](#31-user-login-authorization) | User login (UID Mode + Password Mode) | `POST /auth/oauth/token` |
| [3.2](#32-send-registration-verification-code-password-mode) | Send registration verification code | `POST /business-app/v2/user/register/sendRegisterCode` |
| [3.3](#33-register-password-mode) | Register (email + password) | `POST /business-app/v1/user/register/registryByUserName` |
| [3.4](#34-recover-password) | Recover password (send code + reset) | `POST /business-app/v2/user/password/find/sendFindPasswordCode` etc. |
| [3.5](#35-change-password) | Change password | `POST /business-app/v1/user/password/update/updatePassword` |
| [3.6](#36-logout) | Logout | `POST /auth/oauth/logout` |
| [3.7](#37-get-user-profile) | Get user profile | `POST /business-app/v1/user/profile` |
| [3.8](#38-update-user-info) | Update user info | `POST /business-app/v1/user/updateInfo` |
| [3.9](#39-upload-file) | Upload file (e.g. avatar) | `POST /business-app/v1/file/uploadFile` |

### Provisioning

| § | Endpoint | Method + Path |
|---|---|---|
| [4.1](#41-get-product-info) | Get product info (query provisioning mode) | `POST /business-app/v1/product/getByProductId` |
| [4.2](#42-encrypt-provisioning-data) | Encrypt provisioning data | `POST /business-app/v1/distributionNet/dataEncrypt` |
| [4.3](#43-get-data-center-list) | Get data center list | `POST /business-app/v1/common/getDataCenterList` |
| [4.4](#44-query-device-binding-status) | Query device binding status (polling) | `POST /business-app/v1/device/bind/checkBindResult/{uuid}` |
| [4.5](#45-4g-binding-code-provisioning) | 4G binding code provisioning | `POST /business-app/v1/device/bind/bindDeviceBy4gCode` |
| [4.6](#46-barcode-provisioning) | Barcode provisioning | `POST /business-app/v1/device/bind/bindDeviceFromBarcode` |
| [4.7](#47-direct-bind-known-uuid) | Direct bind (known UUID) | `POST /business-app/v1/device/bind/bindDevice` |

### Device Management

| § | Endpoint | Method + Path |
|---|---|---|
| [5.1](#51-get-account-id) | Get account ID (asset tree) | `POST /business-app/v1/asset/assetTree` |
| [5.2](#52-get-device-info) | Get device info (by UUID/PID) | `POST /business-app/v1/device/getSimpleDeviceInfo` |
| [5.3](#53-get-product-categories) | Get product categories | `POST /business-app/v1/category/top` |
| [5.4](#54-get-device-list) | Get device list | `POST /business-app/v1/device/getHomeDeviceAndGroupList` |
| [5.5](#55-ota-upgrade-check) | OTA upgrade check | `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{type}` |
| [5.6](#56-unbind-device) | Unbind device | `POST /business-app/v1/device/bind/unbind` |
| [5.7](#57-get-device-detail) | Get device detail (by deviceId) | `POST /business-app/v1/device/getByDeviceId/{deviceId}` |
| [5.8](#58-rename-device) | Rename device | `POST /business-app/v1/device/initDevice` |
| [5.9](#59-issue-device-properties) | Issue device properties (control hardware) | `POST /business-app/v1/device/command/propsIssue` |
| [5.10](#510-device-network-check) | Device network check | `POST /business-app/v1/device/command/checkSignal` |
| [5.11](#511-get-device-thing-model-dps) | Get device Thing Model DPs | `POST /business-app/v1/device/getDpInfos/{deviceId}` |
| [5.12](#512-unbind-device-alias-unbindfromasset) | Unbind device (alias) | `POST /business-app/v1/device/unbindFromAsset` |

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
| [6.12](#612-update-customized-agent) | Update customized Agent | `POST /business-app/v1/agents/customize/update` |
| [6.13](#613-unbind-agent-device) | Unbind Agent (device) | `POST /business-app/v1/agents/device/unbind-agent` |
| [6.14](#614-get-agent-bound-to-device) | Get Agent bound to device | `POST /business-app/v1/agents/device/getAgentBaseByDeviceId` |
| [6.15](#615-helper-endpoints-language--voice--llm-lists) | Helper lists (language / voice / LLM) | `POST /business-app/v1/agents/customize/{language,voice,llm}-list` |
| [6.16](#616-get-conversation-history) | Get conversation history | `POST /business-app/v1/agents/conversation/history` |
| [6.17](#617-clear-conversation-history) | Clear conversation history | `POST /business-app/v1/agents/conversation/history/clean` |

---

## 3. User Authentication Endpoints

Sentino supports two authentication modes simultaneously. Pick one based on your integration scenario:

| Mode | grant_type | Use case | Related endpoints |
|---|---|---|---|
| **UID Mode** (B2B2C) | `uid` | The brand already has its own user system; pass `user_id` through to Sentino as `uid`. Registration and login are combined | §3.1 only |
| **Password Mode** (consumer-facing) | `password` | Use the Sentino consumer account system directly (email + password + verification code) | §3.1 + §3.2 ~ §3.12 |

> The server recognizes both modes and routes by the `grant_type` field. The white-label Flutter App template (`sentino-app-sample`) uses Password Mode by default.

---

### 3.1 User Login (Authorization)

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

#### Mode A: UID (registration + login combined)

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

#### Mode B: Password (email + password)

**Body parameters** (application/x-www-form-urlencoded):

| Parameter | Type | Required | Description |
|---|---|---|---|
| `grant_type` | string | Yes | Fixed value: `password` |
| `username` | string | Yes | Email address |
| `password` | string | Yes | Password (≥6 characters) |
| `areaCode` | string | Yes | Area code, default `86` |
| `countryKey` | string | Yes | Country code, default `CN` |

> Before using Password Mode, a new user must first complete registration via §3.2 + §3.4.

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token" \
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
  -d "grant_type=password&username=user@example.com&password=test123456&areaCode=86&countryKey=CN"
```

#### Response (same for both modes)

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

### 3.2 Send Registration Verification Code (Password Mode)

```
POST /business-app/v2/user/register/sendRegisterCode
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `countryCode` | string | Yes | Country code, e.g. `86` |
| `input` | string | Yes | Email address or phone number (auto-detected) |

**Response `data`**:

| Field | Type | Description |
|---|---|---|
| `sendTo` | string | Target the code was sent to (masked email/phone) |
| `verifyCodeLength` | int | Verification code length (used to render the input field dynamically) |
| `intervalSeconds` | int | Resend interval (seconds) |
| `expireSeconds` | int | Code expiration (seconds) |

---

### 3.3 Register (Password Mode)

```
POST /business-app/v1/user/register/registryByUserName
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `input` | string | Yes | Email address |
| `password` | string | Yes | Password (≥6 characters) |
| `verifyCode` | string | Yes | Verification code received in §3.2 |
| `countryCode` | string | Yes | Area code, default `86` |
| `countryKey` | string | Yes | Country code, default `CN` |

**Response**: `data: true` indicates successful registration; then call §3.1 Mode B to log in.

---

### 3.4 Recover Password

Send password recovery verification code:

```
POST /business-app/v2/user/password/find/sendFindPasswordCode
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `input` | string | Yes | Email address |
| `passwordFindType` | string | Yes | `email_code` or `sms_code` |

Reset password:

```
POST /business-app/v1/user/password/find/resetPassword
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `input` | string | Yes | Email address |
| `password` | string | Yes | New password |
| `passwordFindType` | string | Yes | Same as the one used when sending the code |
| `verifyCode` | string | Yes | Verification code |

---

### 3.5 Change Password

```
POST /business-app/v1/user/password/update/updatePassword
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `oldPassword` | string | Yes | Old password |
| `password` | string | Yes | New password |
| `passwordUpdateType` | string | Yes | Fixed value: `password` |

> After a successful change, automatically logging out (clearing the local token) and redirecting to the login page is recommended.

---

### 3.6 Logout

```
POST /auth/oauth/logout
```

No request parameters; must include `Authorization: Bearer $TOKEN`. Response: `data: null`.

---

### 3.7 Get User Profile

```
POST /business-app/v1/user/profile
```

No request parameters.

**Response `data`** (User object):

| Field | Type | Description |
|---|---|---|
| `id` / `userId` / `memberId` | string | User ID (the same value under three aliases) |
| `nickname` | string? | Nickname |
| `userName` | string? | Username |
| `email` | string? | Email |
| `phone` | string? | Phone number |
| `avatarUrl` | string? | Avatar URL |
| `areaCode` | string? | Area code |
| `userType` | int | User type |
| `tz` | string? | Timezone |
| `tempUnit` | string? | Temperature unit |

---

### 3.8 Update User Info

```
POST /business-app/v1/user/updateInfo
Content-Type: application/json
```

**Body parameters**: dynamic — pass any updatable field from the User model.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `nickname` | string | No | Nickname |
| `avatarUrl` | string | No | Avatar URL |
| Other fields | any | No | Any updatable field from the User model |

---

### 3.9 Upload File

```
POST /business-app/v1/file/uploadFile
Content-Type: multipart/form-data
```

**Form parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | File | Yes | File (e.g. avatar image) |

**Response `data`**: `string` (file URL). Commonly used by §3.8 to update the avatar.

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

### 4.5 4G Binding Code Provisioning

After a 4G device boots, it generates a 5-digit "binding code" that the user enters in the App to bind the device (no BLE involved).

```
POST /business-app/v1/device/bind/bindDeviceBy4gCode
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string | Yes | Asset ID (account ID) |
| `bindCode` | string | Yes | 5-digit binding code |

**Response**: `data: null`; `code: 200` indicates a successful bind.

---

### 4.6 Barcode Provisioning

Bind the device by scanning the barcode printed on its housing (works for both 4G and WiFi devices).

```
POST /business-app/v1/device/bind/bindDeviceFromBarcode
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string | Yes | Asset ID |
| `barcode` | string | Yes | Barcode printed on the device housing |

---

### 4.7 Direct Bind (Known UUID)

Bypass the BLE provisioning flow and bind directly when the device UUID is already known.

```
POST /business-app/v1/device/bind/bindDevice
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string | Yes | Asset ID |
| `deviceUuid` | string | Yes | Device UUID |

> Only used in test/debug scenarios. Production Apps typically do not expose this entry point.

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

> The server also accepts the equivalent alias [§5.12 unbindFromAsset](#512-unbind-device-alias-unbindfromasset). Both work; this section's path is the recommended one.

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

### 5.7 Get Device Detail

Retrieve full device information by device ID (different from §5.2 which queries by UUID/PID — this endpoint requires the device to already be bound).

```
POST /business-app/v1/device/getByDeviceId/{deviceId}
```

**Path parameters**: `deviceId` — device ID

**Response `data`** (Device object, fields same as §5.4):

```json
{
  "code": 200,
  "data": {
    "id": "dev_001",
    "uuid": "ct01CykKfw5SMybt",
    "productId": "zuNuadqzsxEh75",
    "name": "智能音箱",
    "imageUrl": "https://cdn.example.com/device/speaker.png",
    "onlineStatus": 1,
    "firmwareVersion": "1.2.3",
    "mac": "AA:BB:CC:DD:EE:FF",
    "ip": "192.168.1.100",
    "networkType": "WiFi",
    "timeZone": "Asia/Shanghai"
  }
}
```

---

### 5.8 Rename Device

Set or modify the device's display name (the endpoint is named `initDevice`, but Apps typically use it as "rename").

```
POST /business-app/v1/device/initDevice
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string | Yes | Asset ID |
| `deviceUuid` | string | Yes | Device UUID |
| `deviceName` | string | Yes | New name |

**Response**: `data: null`; `code: 200` indicates success.

---

### 5.9 Issue Device Properties

The App controls device hardware by writing property key/value pairs into the device. The exact property keys (such as `volume`, `led_color`) are defined by the device's Thing Model.

```
POST /business-app/v1/device/command/propsIssue
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |
| `data` | object | Yes | Property key/value pairs, e.g. `{"volume": 50}` |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/command/propsIssue" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"deviceId\": \"$DEVICE_ID\", \"data\": {\"volume\": 50}}"
```

> The synchronous execution result is delivered to the App via real-time MQTT messages (please contact the Sentino team for the protocol specification).

---

### 5.10 Device Network Check

Trigger the device to report its current WiFi/4G signal strength.

```
POST /business-app/v1/device/command/checkSignal
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |

> The endpoint itself only triggers the check; the result is delivered to the App via real-time MQTT messages (with `signal` 1=good / 2=medium / 3=poor and `signalValue` 0-100). Please contact the Sentino team for the protocol specification.

---

### 5.11 Get Device Thing Model DPs

Retrieve the device's Thing Model data points (DPs); the App uses this list to render the device control panel.

```
POST /business-app/v1/device/getDpInfos/{deviceId}
```

**Path parameters**: `deviceId` — device ID

**Response `data`** (DeviceDpInfoVO array):

| Field | Type | Description |
|---|---|---|
| `id` | string | DP ID |
| `key` | string | Property identifier (`volume_set`, etc. — used as the key when issuing) |
| `name` | string | Display name |
| `type` | string | Value type (`value` / `bool` / `enum` / `string`) |
| `value` | dynamic | Current value |
| `specs` | string | Model spec JSON (contains min/max/step/enum, etc.) |
| `dpBusiId` | string | Property business ID |
| `imageUrl` | string? | DP icon |
| `valueCastType` | int | `0` = raw value, `1` = percentage |

```json
{
  "code": 200,
  "data": [
    {
      "id": "dp_001",
      "key": "volume_set",
      "name": "音量设置",
      "type": "value",
      "value": 5,
      "specs": "{\"min\":0,\"max\":10,\"step\":1}",
      "dpBusiId": "busi_001",
      "imageUrl": null,
      "valueCastType": 0
    }
  ]
}
```

---

### 5.12 Unbind Device (alias: unbindFromAsset)

Equivalent alias of [§5.6](#56-unbind-device). The server accepts both paths with identical business effect: unbind a device.

```
POST /business-app/v1/device/unbindFromAsset
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |
| `isCleanData` | int | Yes | `1` = clear data, `0` = do not clear |

> Both paths exist for historical reasons. New code should prefer [§5.6 `bind/unbind`](#56-unbind-device) (consistent with the bind / unbind business naming); existing code already using `unbindFromAsset` may keep it.

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

### 6.12 Update Customized Agent

```
POST /business-app/v1/agents/customize/update
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |
| `name` | string | No | Name |
| `description` | string | No | Description |
| `avatarUrl` | string | No | Avatar URL |
| `langId` | string | No | Language ID (see §6.15) |
| `llmModelId` | string | No | Model ID (see §6.15) |
| `ttsVoiceId` | string | No | Voice ID (see §6.15) |

**Response**: `data: null`; `code: 200` indicates success.

---

### 6.13 Unbind Agent (Device)

```
POST /business-app/v1/agents/device/unbind-agent
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |

**Response**: `data: null`. After unbinding, the device enters an "Agent-less" state and must be re-bound via §6.9 before it can converse again.

---

### 6.14 Get Agent Bound to Device

Commonly used by the App to render the device panel — query the basic info of the Agent currently bound to the device.

```
POST /business-app/v1/agents/device/getAgentBaseByDeviceId
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `deviceId` | string | Yes | Device ID |

**Response `data`**: Agent object (fields same as §6.2). If the device is not bound, returns `data: null`.

---

### 6.15 Helper Endpoints (Language / Voice / LLM Lists)

Used to populate dropdowns when creating/updating customized Agents (§6.4 / §6.12).

#### 6.15.1 Get Language List

```
POST /business-app/v1/agents/customize/language-list
```

**Response `data`**:

```json
[
  {"id": "lang_zh", "name": "中文"},
  {"id": "lang_en", "name": "English"},
  {"id": "lang_ja", "name": "日本語"}
]
```

#### 6.15.2 Get Voice List

```
POST /business-app/v1/agents/customize/voice-list
```

**Response `data`**:

```json
[
  {"id": "voice_001", "name": "甜美女声"},
  {"id": "voice_002", "name": "磁性男声"}
]
```

#### 6.15.3 Get LLM Model List

```
POST /business-app/v1/agents/customize/llm-list
```

**Response `data`**:

```json
[
  {"id": "model_gpt4", "name": "GPT-4"},
  {"id": "model_gpt35", "name": "GPT-3.5"}
]
```

---

### 6.16 Get Conversation History

Fetch the message log for a device × Agent pair.

```
POST /business-app/v1/agents/conversation/history
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |
| `targetId` | string | Yes | Target ID (device ID) |
| `targetType` | string | Yes | Target type, fixed value `device` |

**Response data** (message array):

| Field | Type | Description |
|---|---|---|
| `role` | string | `user` or `assistant` |
| `content` | string | Message content |
| `createTime` | int | Timestamp in milliseconds |

```json
{
  "code": 200,
  "data": [
    {"role": "user", "content": "你好", "createTime": 1681234567000},
    {"role": "assistant", "content": "你好！有什么可以帮你的吗？", "createTime": 1681234568000}
  ]
}
```

---

### 6.17 Clear Conversation History

Clear the entire conversation history for the given Agent.

```
POST /business-app/v1/agents/conversation/history/clean
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**Response**: `data: null`, `code: 200` indicates success. Not recoverable.

---

## 7. Error Codes

Errors fall into two layers: HTTP status codes (transport layer) and business codes (the `code` field in the response body). **HTTP 200 does not mean success** — you must also check that the business code `code === 200`.

### 7.1 HTTP Status Codes

| HTTP status | Description |
|---|---|
| `200` | Request delivered successfully; still need to check the business code |
| `400` | Malformed request (missing required field, JSON parse failure, etc.) |
| `401` | Invalid token (Authorization header missing or malformed) |
| `403` | Forbidden (insufficient permission) |
| `404` | Endpoint not found (wrong path) |
| `429` | Rate limit (not yet enabled; will take effect in the future) |
| `500` | Internal server error |

### 7.2 Business Codes

| code | Meaning | Recommended client handling |
|------|---------|---------|
| `200` | Success | Process `data` normally |
| `11013` | Token invalid | Clear the cached `access_token` locally and redirect to the login page |

> The business code list will expand as the platform evolves. When you encounter a non-200 business code that is not listed here:
> 1. Log the full `code`, `message`, and `reqId`;
> 2. Show the user a "Service unavailable, please try again later" message in the UI;
> 3. Report it back to the Sentino team to add it to this table.

---

> The endpoint quick reference has been moved to the top of this document (after §2), grouped by business area with anchor links.

---

**Related documents**: [App Integration Guide](../guides/guide-app-en.md) | [Quick Start — App](../tutorials/quickstart-app-en.md) | [Architecture & Concepts](../architecture-en.md)
