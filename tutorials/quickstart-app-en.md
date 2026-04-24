# Quick Start — App Side

This document helps you run the App-side core flow with curl in **10 minutes**: login -> get account ID -> query device info -> bind agent.

> **Prerequisites**: We recommend reading [Architecture & Concepts](../architecture-en.md) first to understand the overall architecture.

---

## What You Need

- A computer with internet access (macOS / Linux / Windows)
- The curl command-line tool (built-in on macOS / Linux)
- jq (optional, for formatting JSON output)

**Test environment**:

| Item | Value |
|---|---|
| REST API base URL | `https://api-iot.sentino.jp` |
| app_id | `krkfvb4s5e91hq` |
| Authorization (for login) | `Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |

---

## Step 1: User Login

Sentino uses UID-based login — if the UID does not exist, an account is automatically created.

```bash
curl -s -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&app_id=krkfvb4s5e91hq" \
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
TOKEN=$(curl -s -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&app_id=krkfvb4s5e91hq" \
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
curl -s -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
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
curl -s -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=sEF4ljjdH8mo" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "sEF4ljjdH8mo",
    "productName": "AI Toy",
    "protocolType": "MQTT",
    "distributionNetMode": "1",
    "bindMode": 1,
    "status": 1
  }
}
```

| Key field | Description |
|---|---|
| `distributionNetMode` | Provisioning mode: `1`=WiFi+BLE, `2`=WiFi, `3`=BLE |
| `bindMode` | Binding mode: `1`=strong binding (only one user may bind at a time), `2`=weak binding |

---

## Step 4: Query Device Info

Query a device's basic info and binding status by device UUID and product ID.

```bash
curl -s -X POST "https://api-iot.sentino.jp/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"productId": "sEF4ljjdH8mo", "uuid": "ct01wfjSNqGAqUUK"}' | jq .
```

**Expected response**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "uuid": "ct01wfjSNqGAqUUK",
    "productId": "sEF4ljjdH8mo",
    "deviceName": "Smart Device",
    "protocolType": "MQTT",
    "configMode": "BLE",
    "bindStatus": 0
  }
}
```

A `bindStatus` of `0` indicates the device is unbound and ready for provisioning.

---

## Step 5: List Available Agents

Get the list of officially recommended agents (AI characters).

```bash
curl -s -X POST "https://api-iot.sentino.jp/business-app/v1/agents/recommend/agents-list" \
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
      "agentId": "1980570366486159361",
      "avatar": "https://...",
      "name": "Little Helper",
      "description": "I'm your smart assistant",
      "tags": ["assistant", "smart"]
    }
  ]
}
```

---

## Step 6: Bind an Agent to the Device

Pick an agent and bind it to the device. After binding, the device will use this agent when initiating an AI conversation.

```bash
curl -s -X POST "https://api-iot.sentino.jp/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "agentId": "1980570366486159361",
    "agentType": "official",
    "deviceId": "your-device-id"
  }' | jq .
```

> Note: The `deviceId` here is the ID assigned by the cloud after the device is bound, not the UUID. It can be obtained via the device list endpoint.

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
