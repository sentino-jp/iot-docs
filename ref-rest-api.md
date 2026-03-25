# REST API 参考

> **TL;DR**：Sentino IoT 平台 REST API 完整参考。涵盖认证、配网、设备管理、智能体管理共 14 个接口，每个接口附带 curl 示例。

---

## 1. 概述

| 项目 | 值 |
|---|---|
| 基础 URL | `https://api-iot.sentino.jp` |
| 认证方式 | Bearer Token（登录接口使用 Basic Auth） |

---

## 2. 认证

### 2.1 获取 Token

通过登录接口获取 `access_token`（有效期参见登录响应中的 `expires_in` 字段）。

### 2.2 使用 Token

所有业务接口（除登录外）需要在请求头中携带：

```
Authorization: Bearer {access_token}
```

### 2.3 公共请求头

所有业务接口需携带以下请求头：

| Header | 值 | 必填 |
|---|---|---|
| `timezone` | `Asia/Shanghai` | 是 |
| `language` | `zh_CN` | 是 |
| `data_center_code` | `cn` | 是 |
| `client_id` | `Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` | 是 |
| `encrypt_type` | `AES/ECB/PKCS5Padding` | 是 |
| `channel_identifier` | `gk6853gq` | 是 |
| `package_name` | `com.yiyuan` | 是 |
| `app_id` | `krfjnsim9vs7yd` | 是 |

### 2.4 公共响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

---

## 3. 用户认证接口

### 3.1 用户登录（授权）

注册登录一体，使用 UID 方式授权。UID 不存在则自动注册。

```
POST /auth/oauth/token
```

**请求头**：

| Header | 值 |
|---|---|
| Content-Type | `application/x-www-form-urlencoded` |
| app_id | `krfjnsim9vs7yd` |
| Authorization | `Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `grant_type` | string | 是 | 固定值：`uid` |
| `area_code` | string | 是 | 区号，默认 `86` |
| `anonymousId` | string | 否 | 匿名 ID |
| `app_id` | string | 是 | 应用 ID |

**Body 参数** (application/x-www-form-urlencoded)：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `grant_type` | string | 是 | 固定值：`uid` |
| `uid` | string | 是 | 用户唯一标识，最长 50 字符 |
| `password` | string | 是 | 用户密码 |
| `area_code` | string | 是 | 区号，默认 `86` |
| `user_country_key` | string | 是 | 国家代码，默认 `CN` |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&app_id=krfjnsim9vs7yd" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "app_id: krfjnsim9vs7yd" \
  -H "Authorization: Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==" \
  -d "grant_type=uid&uid=test_user_001&password=test123456&area_code=86&user_country_key=CN"
```

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "expires_in": 7200,
    "refresh_token": "..."
  }
}
```

---

## 4. 配网接口

### 4.1 获取产品信息

通过产品 ID 查询产品配置。

```
POST /business-app/v1/product/getByProductId
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `productId` | string | 是 | 产品 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=sEF4ljjdH8mo" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

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

**字段说明**：

| 字段 | 说明 |
|---|---|
| `protocolType` | 通讯协议：`MQTT` / `BLE` |
| `distributionNetMode` | 配网模式：`1`=WiFi+BLE, `2`=WiFi, `3`=BLE |
| `nodeType` | 节点类型：`1`=普通设备, `2`=网关, `3`=边缘网关, `4`=子设备 |
| `bindMode` | 绑定模式：`1`=强绑, `2`=弱绑 |
| `deviceShare` | 是否支持设备共享：`0`=否, `1`=是 |
| `deviceUpgrade` | 是否支持 OTA 升级：`0`=否, `1`=是 |

---

### 4.2 配网数据加密

配网过程中对数据进行加密处理。

```
POST /business-app/v1/distributionNet/dataEncrypt
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `content` | string | 是 | 待加密内容（JSON 字符串） |
| `encryptType` | int | 是 | 加密类型，固定值 `0` |
| `protocol` | string | 是 | 协议版本，固定值 `"1"` |
| `type` | string | 是 | 固定值 `"thing.network.set"` |

**content 字段内容**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `force_bind` | boolean | 是否强制绑定 |
| `sid` | string | WiFi SSID |
| `pw` | string | WiFi 密码 |
| `mq` | string | MQTT Broker 地址 |
| `mqttSslPort` | string | MQTT SSL 端口，默认 `"8883"` |
| `bid` | string | 资产 ID（assetId） |
| `port` | int | MQTT 端口，默认 `1883` |
| `country` | string | 国家代码：`CN` / `US` / `JP` / `EP` / `AU` |
| `userID` | string | 用户 ID |
| `tz` | string | 时区 |
| `areaCode` | string | 区域码 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/distributionNet/dataEncrypt" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "content": "{\"sid\":\"MyWiFi\",\"pw\":\"password\",\"bid\":\"assetId\",\"userID\":\"userId\",\"mq\":\"mqtt-iot.sentino.jp\",\"port\":1883,\"country\":\"CN\",\"tz\":\"Asia/Shanghai\",\"force_bind\":true}",
    "encryptType": 0,
    "protocol": "1",
    "type": "thing.network.set"
  }'
```

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": "{加密后的配网数据字符串}"
}
```

---

### 4.3 获取数据中心列表

获取可用的数据中心列表，包含各中心的 API 地址和 MQTT 地址。

```
POST /business-app/v1/common/getDataCenterList
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/common/getDataCenterList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

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

> 响应中还包含 `appId`、`areaId`、`createTime`、`dataCenterId`、`iotUrl`、`sort` 等字段，此处省略。

---

### 4.4 设备绑定状态查询

配网过程中轮询设备是否已绑定成功。

```
POST /business-app/v1/device/bind/checkBindResult/{uuid}
```

**Path 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `uuid` | string | 是 | 设备 UUID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/checkBindResult/ct01wfjSNqGAqUUK" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": 0
}
```

| `data` 值 | 说明 |
|---|---|
| `0` | 绑定成功 |
| 其他 | 绑定未完成或失败 |

**使用方式**：配网信息发送后，定时轮询（10s）。

---

## 5. 设备管理接口

### 5.1 获取资产树

获取用户的家庭房间树结构，返回 assetId。

```
POST /business-app/v1/asset/assetTree
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "{}"
```

**响应**：

```json
{
  "code": 200,
  "data": {
    "assetId": "1983052957022670848",
    "assetName": "我的家",
    "children": [
      {
        "assetId": "1983052957022670849",
        "assetName": "客厅",
        "children": []
      }
    ]
  }
}
```

---

### 5.2 获取设备信息

通过 UUID 和产品 ID 查询设备基本信息。

```
POST /business-app/v1/device/getSimpleDeviceInfo
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `productId` | string | 是 | 产品 ID |
| `uuid` | string | 是 | 设备 UUID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"productId": "sEF4ljjdH8mo", "uuid": "ct01wfjSNqGAqUUK"}'
```

**响应**：

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

### 5.3 获取产品品类

列出应用支持的品类（非必须接口）。

```
POST /business-app/v1/category/top
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/category/top" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**响应**：

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

### 5.4 获取设备列表

获取用户所有设备和群组列表。

```
POST /business-app/v1/device/getHomeDeviceAndGroupList
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assetIds` | string[] | 是 | 资产 ID 数组 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getHomeDeviceAndGroupList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"assetIds": ["1983052957022670848"]}'
```

**响应**：

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

| 字段 | 说明 |
|---|---|
| `deviceList` | 设备信息列表 |
| `sortIdList` | 用户自定义排序 |
| `shareGroupList` | 被分享的群组 |
| `shareList` | 被分享的设备 |

---

### 5.5 OTA 升级检查

检查设备是否有可用的固件升级。

```
POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}
```

**Path 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |
| `firmwareType` | string | 否 | 固件类型（不传则默认检查模组固件） |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/ota/checkUpgrade/2008424975449309184/module" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

```json
{
  "code": 200,
  "data": {
    "hasUpgrade": true,
    "version": "1.2.0",
    "downloadUrl": "https://...",
    "md5": "abc123...",
    "fileSize": 1024000,
    "upgradeDesc": "修复已知问题，提升稳定性"
  }
}
```

---

### 5.6 设备解绑

解绑设备，可选择是否清除数据。

```
POST /business-app/v1/device/bind/unbind
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |
| `isCleanData` | int | 是 | `1`=清除数据，`0`=不清除 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/unbind" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"deviceId": "2008424975449309184", "isCleanData": 1}'
```

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 6. 智能体管理接口

### 6.1 获取推荐智能体列表

获取官方推荐的智能体模板列表。

```
POST /business-app/v1/agents/recommend/agents-list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**响应**：

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

### 6.2 获取智能体详情

获取指定智能体的详细配置。

```
POST /business-app/v1/agents/detail
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/detail?agentId=1980570366486159361" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

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

### 6.3 绑定智能体到设备

将设备绑定到指定的智能体模板。

```
POST /business-app/v1/agents/device/bind-agent
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |
| `agentType` | string | 是 | `official`=官方, `customize`=自定义 |
| `deviceId` | string | 是 | 设备 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"agentId": "1980570366486159361", "agentType": "official", "deviceId": "2008424975449309184"}'
```

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 7. 错误码

| 错误码 | 说明 |
|---|---|
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未授权（Token 无效或过期） |
| 403 | 禁止访问 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

---

## 8. 接口速查表

| # | 接口 | 路径 | 用途 |
|---|---|---|---|
| 1 | 用户登录 | `POST /auth/oauth/token` | 注册/登录获取 Token |
| 2 | 获取产品信息 | `POST /business-app/v1/product/getByProductId` | 查询产品配网模式 |
| 3 | 配网数据加密 | `POST /business-app/v1/distributionNet/dataEncrypt` | 加密配网数据 |
| 4 | 获取数据中心列表 | `POST /business-app/v1/common/getDataCenterList` | 查询数据中心 |
| 5 | 绑定状态查询 | `POST /business-app/v1/device/bind/checkBindResult/{uuid}` | 轮询配网结果 |
| 6 | 获取资产树 | `POST /business-app/v1/asset/assetTree` | 获取家庭/房间结构 |
| 7 | 获取设备信息 | `POST /business-app/v1/device/getSimpleDeviceInfo` | 查询设备状态 |
| 8 | 获取产品品类 | `POST /business-app/v1/category/top` | 品类列表（可选） |
| 9 | 获取设备列表 | `POST /business-app/v1/device/getHomeDeviceAndGroupList` | 用户全部设备 |
| 10 | OTA 检查 | `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{type}` | 检查固件升级 |
| 11 | 设备解绑 | `POST /business-app/v1/device/bind/unbind` | 解绑设备 |
| 12 | 推荐智能体列表 | `POST /business-app/v1/agents/recommend/agents-list` | 官方 AI 角色 |
| 13 | 智能体详情 | `POST /business-app/v1/agents/detail` | 角色详细配置 |
| 14 | 绑定智能体 | `POST /business-app/v1/agents/device/bind-agent` | 绑定 AI 角色到设备 |

---

## 附录：应用配置

| 配置项 | 值 |
|---|---|
| `app_id` | `krfjnsim9vs7yd` |
| `package_name` | `com.kingstar.lululand` |
| `channel_identifier` | `gk6853gq` |
| `client_id` | `Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |

---

**相关文档**：[App 端集成指南](./guide-app.md) | [快速入门 — App 端](./quickstart-app.md) | [架构与概念](./architecture.md)
