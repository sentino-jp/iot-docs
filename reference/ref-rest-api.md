# REST API 参考

> **TL;DR**：Sentino IoT 平台 REST API 完整参考。涵盖认证、配网、设备管理、智能体管理共 22 个接口，每个接口附带 curl 示例。

---

## 1. 概述

| 项目 | 值 |
|---|---|
| 基础 URL | `https://api-iot.sentino.jp` |
| 认证方式 | Bearer Token（登录接口使用 Basic Auth） |

### 1.1 环境变量准备

本文档所有 curl 示例使用环境变量。运行示例前先导出以下变量（按你测试的环境填写）：

```bash
# 应用凭据（由 Sentino 提供，每个客户应用不同）
export APP_ID="krfjnsim9vs7yd"
export CHANNEL_IDENTIFIER="gk6853gq"
export PACKAGE_NAME="jp.sentino.general"
export DATA_CENTER_CODE="cn"

# 固定值（直接复制）
export CLIENT_ID="Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg=="
export ENCRYPT_TYPE="AES/ECB/PKCS5Padding"

# 客户端属性
export TIMEZONE="Asia/Shanghai"
export LANGUAGE="zh_CN"

# 用户登录凭据（按需替换）
export UID="test_user_001"
export AREA_CODE="86"
export USER_COUNTRY_KEY="CN"

# 登录后获取（见 §3.1 用户登录）
export TOKEN=""
export USER_ID=""

# 业务上下文（按测试设备/账户填写）
export PRODUCT_ID="vqB8C7fniWRLWL"   # 当前 mock 设备 PID
export UUID="ct01kQBXBK7h63H8"        # 当前 mock 设备 UUID
export ASSET_ID=""                    # 通过 §5.1 获取
export DEVICE_ID=""                   # 通过 §5.4 获取
export AGENT_ID=""                    # 通过 §6.1 / §6.3 获取
export NFC_UUID=""                    # 物理 NFC 卡片 UUID
```

> 后续所有 curl 示例假设上述变量已导出。切换数据中心或测试设备只改环境变量即可。

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

所有接口（包括登录）需携带以下请求头。

**由 Sentino 提供（每个客户应用不同，需联系 Sentino 获取）**：

| Header | 说明 |
|--------|------|
| `app_id` | 应用标识 |
| `channel_identifier` | 渠道标识 |
| `package_name` | App 包名 |
| `data_center_code` | 数据中心（按部署区域） |

**固定值（直接复制）**：

| Header | 值 |
|--------|-----|
| `client_id` | `Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==` |
| `encrypt_type` | `AES/ECB/PKCS5Padding` |

**客户端设置（按用户实际情况）**：

| Header | 默认值 | 说明 |
|--------|--------|------|
| `timezone` | `Asia/Shanghai` | 用户时区 |
| `language` | `zh_CN` | 用户语言 |

### 2.4 公共响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

---

## 接口速查表

按业务分组，点击跳转到对应小节。

### 用户认证

| § | 接口 | Method + Path |
|---|---|---|
| [3.1](#31-用户登录授权) | 用户登录（注册/授权合一） | `POST /auth/oauth/token` |

### 配网

| § | 接口 | Method + Path |
|---|---|---|
| [4.1](#41-获取产品信息) | 获取产品信息（查询配网模式） | `POST /business-app/v1/product/getByProductId` |
| [4.2](#42-配网数据加密) | 配网数据加密 | `POST /business-app/v1/distributionNet/dataEncrypt` |
| [4.3](#43-获取数据中心列表) | 获取数据中心列表 | `POST /business-app/v1/common/getDataCenterList` |
| [4.4](#44-设备绑定状态查询) | 设备绑定状态查询（轮询） | `POST /business-app/v1/device/bind/checkBindResult/{uuid}` |

### 设备管理

| § | 接口 | Method + Path |
|---|---|---|
| [5.1](#51-获取账户-id) | 获取账户 ID（资产树） | `POST /business-app/v1/asset/assetTree` |
| [5.2](#52-获取设备信息) | 获取设备信息 | `POST /business-app/v1/device/getSimpleDeviceInfo` |
| [5.3](#53-获取产品品类) | 获取产品品类 | `POST /business-app/v1/category/top` |
| [5.4](#54-获取设备列表) | 获取设备列表 | `POST /business-app/v1/device/getHomeDeviceAndGroupList` |
| [5.5](#55-ota-升级检查) | OTA 升级检查 | `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{type}` |
| [5.6](#56-设备解绑) | 设备解绑 | `POST /business-app/v1/device/bind/unbind` |

### 智能体管理

| § | 接口 | Method + Path |
|---|---|---|
| [6.1](#61-获取推荐智能体列表) | 推荐智能体列表 | `POST /business-app/v1/agents/recommend/agents-list` |
| [6.2](#62-获取智能体详情) | 智能体详情 | `POST /business-app/v1/agents/detail` |
| [6.3](#63-获取自定义智能体列表) | 自定义智能体列表 | `POST /business-app/v1/agents/customize/agents-list` |
| [6.4](#64-创建自定义智能体) | 创建自定义智能体 | `POST /business-app/v1/agents/customize/create` |
| [6.5](#65-删除自定义智能体) | 删除自定义智能体 | `POST /business-app/v1/agents/customize/deleteById` |
| [6.6](#66-获取推荐-sentino-智能体列表) | Sentino 智能体列表 | `POST /business-app/v1/sentino-agents/recommend/agents-list` |
| [6.7](#67-获取-sentino-智能体详情) | Sentino 智能体详情 | `POST /business-app/v1/sentino-agents/detail` |
| [6.8](#68-获取用户已绑定智能体列表) | 已绑定智能体列表 | `POST /business-app/v1/user-agents/list` |
| [6.9](#69-绑定智能体到设备) | 绑定智能体到设备 | `POST /business-app/v1/agents/device/bind-agent` |
| [6.10](#610-nfc-卡片列表) | NFC 卡片列表 | `POST /business-app/v1/agents/nfc/list` |
| [6.11](#611-nfc-卡片绑定智能体) | NFC 卡片绑定智能体 | `POST /business-app/v1/agents/nfc/bind-agent` |

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
| Authorization | `Basic $CLIENT_ID` |
| 公共请求头 | 需携带 2.3 节中的所有公共请求头（`client_id`、`app_id` 等） |

> **注意**：登录接口也需要携带公共请求头，否则签发的 Token 将无法通过后续业务接口的验证。

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

**响应**：

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

## 4. 配网接口

### 4.1 获取产品信息

通过产品 ID 查询产品配置。

```
POST /business-app/v1/product/getByProductId
```

> **productId 来源**：从设备 BLE 广播的 Service Data 中解析得到（PID 字段）。详见 [BLE 协议参考 §2.1 广播数据](./ref-ble.md#21-广播数据)。

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `productId` | string | 是 | 产品 ID（即 BLE 广播中的 PID） |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=$PRODUCT_ID" \
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
| `bid` | string | 账户 ID（assetId） |
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
  -d @- <<EOF
{
  "content": "{\"sid\":\"MyWiFi\",\"pw\":\"password\",\"bid\":\"$ASSET_ID\",\"userID\":\"$USER_ID\",\"mq\":\"mqtt-iot.sentino.jp\",\"port\":1883,\"country\":\"$USER_COUNTRY_KEY\",\"tz\":\"$TIMEZONE\",\"force_bind\":true}",
  "encryptType": 0,
  "protocol": "1",
  "type": "thing.network.set"
}
EOF
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
| `uuid` | string | 是 | 设备 UUID（三元组中的 UUID，通过扫描设备条码获取） |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/checkBindResult/$UUID" \
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

**使用方式**：配网信息发送后，每 10 秒轮询一次，最多 120 秒（12 次）。

---

## 5. 设备管理接口

### 5.1 获取账户 ID

获取用户账户结构，返回 assetId（账户 ID）。

```
POST /business-app/v1/asset/assetTree
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "timezone: $TIMEZONE" \
  -H "language: $LANGUAGE" \
  -H "data_center_code: $DATA_CENTER_CODE" \
  -d "{}"
```

**响应**：

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

> **注意**：响应 `data` 为数组格式，使用 `id` 和 `name` 字段（非 `assetId` / `assetName`），子节点字段为 `childrens`。配网时使用根节点的 `id` 作为 `assetId`。

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
  -d "{\"productId\": \"$PRODUCT_ID\", \"uuid\": \"$UUID\"}"
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
| `assetIds` | string[] | 是 | 账户 ID 数组 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getHomeDeviceAndGroupList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"assetIds\": [\"$ASSET_ID\"]}"
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
curl -X POST "https://api-iot.sentino.jp/business-app/v1/ota/checkUpgrade/$DEVICE_ID/module" \
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
  -d "{\"deviceId\": \"$DEVICE_ID\", \"isCleanData\": 1}"
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
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/detail?agentId=$AGENT_ID" \
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

### 6.3 获取自定义智能体列表

获取用户自定义的智能体模板列表。

```
POST /business-app/v1/agents/customize/agents-list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/customize/agents-list" \
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

### 6.4 创建自定义智能体

创建自定义智能体模板。

```
POST /business-app/v1/agents/customize/create
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `name` | string | 是 | 智能体名称 |
| `description` | string | 是 | 智能体描述（角色设定） |
| `avatarUrl` | string | 是 | 头像 URL |
| `llmModelId` | string | 是 | 大模型 ID |
| `ttsVoiceId` | string | 是 | TTS 音色 ID |
| `langId` | string | 是 | 语言 ID |

**curl 示例**：

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

**响应**：

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 6.5 删除自定义智能体

删除指定的自定义智能体模板。

```
POST /business-app/v1/agents/customize/deleteById
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/customize/deleteById?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
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

### 6.6 获取推荐 Sentino 智能体列表

获取官方推荐的 Sentino 智能体模板列表。Sentino 智能体模板定义了智能体关联的 Sentino 角色。

```
POST /business-app/v1/sentino-agents/recommend/agents-list
```

**请求头**（除公共请求头外）：

| Header | 值 | 必填 |
|---|---|---|
| `client_id` | 应用 client_id | 是 |
| `app_id` | 应用 app_id | 是 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/sentino-agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "client_id: $CLIENT_ID" \
  -H "app_id: $APP_ID" \
  -d "{}"
```

**响应**：

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

### 6.7 获取 Sentino 智能体详情

获取指定 Sentino 智能体模板的详细信息。

```
POST /business-app/v1/sentino-agents/detail
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/sentino-agents/detail?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**响应**：

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

### 6.8 获取用户已绑定智能体列表

获取当前用户已绑定的智能体列表。

```
POST /business-app/v1/user-agents/list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/user-agents/list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
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
      "tags": ["助手"]
    }
  ]
}
```

---

### 6.9 绑定智能体到设备

将设备绑定到指定的智能体模板。

```
POST /business-app/v1/agents/device/bind-agent
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |
| `agentType` | string | 是 | `official`=官方, `customize`=自定义, `sentino`=Sentino 智能体 |
| `deviceId` | string | 是 | 设备 ID |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"agentId\": \"$AGENT_ID\", \"agentType\": \"official\", \"deviceId\": \"$DEVICE_ID\"}"
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

### 6.10 NFC 卡片列表

读取底座设备上已绑定的 NFC 卡片列表。

```
POST /business-app/v1/agents/nfc/list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/nfc/list" \
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
      "nfcUuid": "1111111111",
      "agentId": "1947925380891516929",
      "agentType": "official"
    }
  ]
}
```

---

### 6.11 NFC 卡片绑定智能体

将 NFC 卡片绑定到指定的智能体模板。

```
POST /business-app/v1/agents/nfc/bind-agent
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `nfcUuid` | string | 是 | NFC 卡片 UUID |
| `agentId` | string | 是 | 智能体 ID |
| `agentType` | string | 是 | `official`=官方, `customize`=自定义 |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/nfc/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"nfcUuid\": \"$NFC_UUID\", \"agentId\": \"$AGENT_ID\", \"agentType\": \"official\"}"
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

> 接口速查表已前移到本文顶部（§2 之后），按业务分组并支持锚点跳转。

---

**相关文档**：[App 端集成指南](../guides/guide-app.md) | [快速入门 — App 端](../tutorials/quickstart-app.md) | [架构与概念](../architecture.md)
