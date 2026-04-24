# 快速入门 — App 端

本文档帮助你在 **10 分钟内**用 curl 跑通 App 端核心链路：登录 → 获取账户 ID → 查询设备信息 → 绑定智能体。

> **前置知识**：建议先阅读 [架构与概念](../architecture.md) 了解整体架构。

---

## 你需要准备什么

- 一台能联网的电脑（macOS / Linux / Windows）
- curl 命令行工具（macOS / Linux 自带）
- jq（可选，用于格式化 JSON 输出）

**测试环境信息**：

| 项目 | 值 |
|---|---|
| REST API 基础 URL | `https://api-iot.sentino.jp/api` |
| app_id | `krkfvb4s5e91hq` |
| Authorization (登录用) | `Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |

---

## 第 1 步：用户登录

Sentino 使用 UID 方式登录 — 如果 UID 不存在则自动注册。

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

> **注意**：登录接口也需要携带公共请求头（`client_id`、`channel_identifier` 等），否则签发的 Token 将无法通过后续业务接口的验证。

**预期响应**：

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

**保存 Token**（后续步骤使用）：

```bash
# 提取 access_token 并保存为变量
TOKEN="6ea8368a-127c-4203-b7e8-83fbeb9d0239"

# 或者用 jq 自动提取
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

## 第 2 步：获取账户 ID

绑定设备时需要指定 `assetId`（账户 ID）。调用以下接口获取，直接使用根节点的 `assetId`。

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "{}" | jq .
```

**预期响应**：

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

记下根节点的 `id`（账户 ID，即 assetId），后续配网绑定时需要使用。注意响应 `data` 为数组格式，字段名为 `id`（非 `assetId`）。

---

## 第 3 步：获取产品信息

通过产品 ID 查询产品的配网模式和基本配置。

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/product/getByProductId?productId=sEF4ljjdH8mo" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer $TOKEN" | jq .
```

**预期响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "sEF4ljjdH8mo",
    "productName": "智能玩具",
    "protocolType": "MQTT",
    "distributionNetMode": "1",
    "bindMode": 1,
    "status": 1
  }
}
```

| 关键字段 | 说明 |
|---|---|
| `distributionNetMode` | 配网模式：`1`=WiFi+BLE, `2`=WiFi, `3`=BLE |
| `bindMode` | 绑定模式：`1`=强绑（同一时间仅一个用户可绑定），`2`=弱绑 |

---

## 第 4 步：查询设备信息

通过设备 UUID 和产品 ID 查询设备基本信息和绑定状态。

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"productId": "sEF4ljjdH8mo", "uuid": "ct01wfjSNqGAqUUK"}' | jq .
```

**预期响应**：

```json
{
  "code": 200,
  "message": "success",
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

`bindStatus` 为 `0` 表示设备未绑定，可以进行配网。

---

## 第 5 步：查看智能体列表

获取官方推荐的智能体（AI 角色）列表。

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}" | jq .
```

**预期响应**：

```json
{
  "code": 200,
  "message": "success",
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

## 第 6 步：绑定智能体到设备

选择一个智能体，将其绑定到设备。绑定后设备发起 AI 对话时将使用该智能体。

```bash
curl -s -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "agentId": "1980570366486159361",
    "agentType": "official",
    "deviceId": "你的设备ID"
  }' | jq .
```

> 注意：这里的 `deviceId` 是设备绑定后由云端分配的 ID，不是 UUID。可通过设备列表接口获取。

**预期响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 验证成功后的下一步

你已跑通了 App 端核心链路。接下来：

| 目标 | 去哪里看 |
|---|---|
| 了解完整的 App 端集成流程 | [App 端集成指南](../guides/guide-app.md) |
| 了解 BLE 配网的实现细节 | [BLE 协议参考](../reference/ref-ble.md) |
| 查看所有 REST API 接口 | [REST API 参考](../reference/ref-rest-api.md) |
| 了解设备端的接入方式 | [快速入门 — 设备端](./quickstart-device.md) |

---

## 常见问题

### 登录返回 401

1. 检查 `Authorization` header 是否正确（注意是 `Basic` 认证，不是 `Bearer`）
2. 检查 `app_id` 是否匹配

### Token 过期

- `access_token` 有效期为 7200 秒（2 小时）
- 过期后使用 `refresh_token` 刷新，或重新登录

### 账户 ID 为空

- 如果返回为空，请确认登录是否成功，并检查是否已在 Sentino 后台配置账户

---

**下一步**：

- 把验证扩展为完整集成 → [App 端集成指南](../guides/guide-app.md) + [REST API 参考](../reference/ref-rest-api.md)
- **拿白标 Flutter 模板改 UI 与品牌** → fork [`sentino-jp/sentino-app-sample`](https://github.com/sentino-jp/sentino-app-sample)，按其 [`doc/quick-start.md`](https://github.com/sentino-jp/sentino-app-sample/blob/main/doc/quick-start.md) 跑起来
