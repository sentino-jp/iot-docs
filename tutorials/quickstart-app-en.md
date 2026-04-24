# Quick Start — App Side

This document helps you run the App-side core flow with curl in **10 minutes**: login -> get account ID -> query device info -> bind agent.

> **Prerequisites**: We recommend reading [Architecture & Concepts](../architecture-en.md) first to understand the overall architecture.

> [!IMPORTANT]
> **App credentials (`app_id` / `channel_identifier` / `package_name`) and test-device triplets (UUID / KEY / MAC) must be issued by the Sentino team for your application.**
> Sample values in this document are for quickstart demonstration only; do not reuse them in production.

---

## What You Need

- A computer with internet access (macOS / Linux / Windows)
- The curl command-line tool (built-in on macOS / Linux)
- jq (optional, for formatting JSON output)

**Test environment**:

| Item | Value |
|---|---|
| REST API base URL | `https://api-iot.sentino.jp/api` |
| app_id | `krkfvb4s5e91hq` |
| Authorization (for login) | `Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |

---

## Step 1: User Login

This quickstart uses **UID mode** (`grant_type=uid`): if the UID does not exist, an account is **created automatically**, so a single curl call gets you a token — ideal for quick verification.

> Sentino also supports **Password mode** (`grant_type=password`, three-step email + password + verification-code signup); the white-label Flutter App template (`sentino-app-sample`) uses password mode by default. Tokens issued by both modes are fully equivalent — pick whichever fits your scenario. See [REST API §3.1 User Login](../reference/ref-rest-api-en.md#31-user-login-authorization).

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/auth/oauth/token?grant_type=uid&area_code=86&app_id=krkfvb4s5e91hq" \
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
  -d "grant_type=uid&uid=test_user_001&password=test123456&area_code=86&user_country_key=CN" | jq .
```

> **Note**: The login endpoint also requires the public request headers (`client_id`, `channel_identifier`, etc.); otherwise the issued Token will not pass validation on subsequent business endpoints.

**Expected response**:

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
    "username": "test_user_001"
  }
}
```

**Save the Token** (used in subsequent steps):

```bash
# Extract access_token and save as a variable
TOKEN="6ea8368a-127c-4203-b7e8-83fbeb9d0239"

# Or extract automatically with jq
TOKEN=$(curl -s -X POST "https://api-iot.sentino.jp/api/auth/oauth/token?grant_type=uid&area_code=86&app_id=krkfvb4s5e91hq" \
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
  -d "grant_type=uid&uid=test_user_001&password=test123456&area_code=86&user_country_key=CN" | jq -r '.data.access_token')

echo "Token: $TOKEN"
```

---

## Step 2: Get the Account ID

Binding a device requires specifying an `assetId` (account ID). Call the following endpoint to obtain it, and use the `assetId` of the root node directly.

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "{}" | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
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

Take note of the root node's `id` (the account ID, i.e. assetId); it will be used during provisioning and binding. Note that the response `data` is an array, and the field name is `id` (not `assetId`).

---

## Step 3: Get Product Info

Query the product's provisioning mode and basic configuration by product ID.

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/product/getByProductId?productId=OQm9yRoaLq1gbK" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "OQm9yRoaLq1gbK",
    "name": "Kumamoto",
    "imageUrl": "https://...",
    "model": "...",
    "tenantId": "...",
    "typeId": "...",
    "protocolType": "MQTT",
    "protocolName": "MQTT 5.0",
    "connectCloudType": 1,
    "nodeType": 1
  }
}
```

> Depending on the product type, the server may return additional fields such as `distributionNetMode` / `bindMode`; rely on the actual response.

---

## Step 4: Query Device Info

Query a device's basic info and binding status by device UUID and product ID.

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"productId": "OQm9yRoaLq1gbK", "uuid": "ct013P3QJR5SdaW9"}' | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "...",
    "uuid": "ct013P3QJR5SdaW9",
    "productId": "OQm9yRoaLq1gbK",
    "name": "Smart Speaker",
    "imageUrl": "https://...",
    "barcode": "SN...",
    "protocolType": "MQTT",
    "nodeType": 1,
    "distributionNetMode": "...",
    "distributionNetModes": ["..."],
    "bindStatus": 0,
    "isIpc": false,
    "isLowPower": false
  }
}
```

A `bindStatus` of `0` indicates the device is unbound and ready for provisioning; non-`0` (e.g. `1`) means it is already bound by some user.

---

## Step 5: List Available Agents

Get the list of Agents maintained by the Sentino platform (AI characters).

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/sentino-agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}" | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "agentId": "2046112542823174144",
      "avatarUrl": "https://...",
      "name": "Little Helper",
      "description": "I'm your smart assistant",
      "tagList": []
    }
  ]
}
```

---

## Step 6: Bind an Agent to the Device

Pick an agent and bind it to the device. After binding, the device will use this agent when initiating an AI conversation.

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "agentId": "2046112542823174144",
    "agentType": "sentino",
    "deviceId": "your-device-id"
  }' | jq .
```

> Note: `deviceId` is the cloud-assigned ID (not the UUID), available via [§5.4 Get Device List](../reference/ref-rest-api-en.md#54-get-device-list); this step succeeds only when your account already has a real bound device.

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## Next Steps After Verification

You have completed the App-side core flow. What's next:

| Goal | Where to look |
|---|---|
| Learn the full App-side integration flow | [App Integration Guide](../guides/guide-app-en.md) |
| Learn BLE provisioning implementation details | [BLE Protocol Reference](../reference/ref-ble-en.md) |
| Browse all REST API endpoints | [REST API Reference](../reference/ref-rest-api-en.md) |
| Learn how to integrate the device side | [Quick Start — Device Side](./quickstart-device-en.md) |

---

## FAQ

### Login returns 401

1. Check that the `Authorization` header is correct (note: it is `Basic` auth, not `Bearer`)
2. Check that the `app_id` matches

### Token Expiration

- The `access_token` is valid for 7200 seconds (2 hours)
- After expiration, refresh it with the `refresh_token`, or log in again

### Account ID Is Empty

- If the response is empty, confirm that login succeeded and check that the account is configured in the Sentino backend

---

**Next**:

- Extend this verification into a full integration -> [App Integration Guide](../guides/guide-app-en.md) + [REST API Reference](../reference/ref-rest-api-en.md)
- **Take the white-label Flutter template and rebrand the UI** -> fork [`sentino-jp/sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample) and follow its [`doc/quick-start.md`](https://github.com/sentino-jp/sentino-app-sample/blob/main/doc/quick-start.md) to get running
