# Sentino IoT API 接口文档

**版本**: v1.0  
**基础URL**: `https://api-iot.sentino.jp`  
**更新时间**: 2026年2月11日

---

## 目录

1. [认证说明](#认证说明)
2. [公共参数](#公共参数)
3. [用户认证接口](#用户认证接口)
4. [配网接口](#配网接口)
5. [设备管理接口](#设备管理接口)
6. [智能体管理接口](#智能体管理接口)
7. [智能体设备接口](#智能体设备接口)
8. [错误码说明](#错误码说明)

---

## 认证说明

### 认证方式
所有业务接口(除登录接口外)使用 Bearer Token 认证方式。

### 获取 Token
通过 [用户登录接口](#1-用户登录授权) 获取 `access_token`。

### 使用方式
在请求头中添加:
```
Authorization: Bearer {access_token}
```

---

## 公共参数

### 公共请求头

所有业务接口都需要携带以下请求头:

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| timezone | string | 是 | 时区 | Asia/Shanghai |
| language | string | 是 | 语言 | zh_CN |
| data_center_code | string | 是 | 数据中心代码 | cn |
| client_id | string | 是 | 客户端ID | Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg== |
| encrypt_type | string | 是 | 加密类型 | AES/ECB/PKCS5Padding |
| channel_identifier | string | 是 | 渠道标识 | gk6853gq |
| package_name | string | 是 | 包名 | com.yiyuan |
| app_id | string | 是 | 应用ID | krljcdkukdng6v |

### 公共响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| code | integer | 响应码,200表示成功 |
| message | string | 响应消息 |
| data | object | 响应数据 |

---

## 用户认证接口

### 1. 用户登录(授权)

注册登录一体接口,使用 UID 方式进行授权。

**接口地址**: `POST /auth/oauth/token`

**接口说明**:
- 如果 UID 之前不存在,则自动注册新用户
- 如果 UID 已存在,则直接登录
- UID 长度限制:50个字符,无其他格式限制

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/x-www-form-urlencoded |
| app_id | string | 是 | 应用ID |
| Authorization | string | 是 | Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg== |

**Query 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| grant_type | string | 是 | 固定值: uid |
| area_code | string | 是 | 区号,默认86 |
| anonymousId | string | 否 | 匿名ID |
| app_id | string | 是 | 应用ID |

**Body 参数** (application/x-www-form-urlencoded):

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| grant_type | string | 是 | 固定值: uid |
| uid | string | 是 | 用户唯一标识,最长50个字符 |
| password | string | 是 | 用户密码 |
| area_code | string | 是 | 区号,默认86 |
| user_country_key | string | 是 | 国家代码,默认CN |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&anonymousId=8c465469cfcdab9c&app_id=krfjnsim9vs7yd" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "app_id: krfjnsim9vs7yd" \
  -H "Authorization: Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==" \
  -d "grant_type=uid&uid=YOUR_UID&password=YOUR_PASSWD&area_code=86&user_country_key=CN"
```

**响应示例**:

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

## 配网接口

### 2. 获取产品信息

通过产品ID获取产品的详细配置信息,包括配网模式、通讯协议等。

**接口地址**: `POST /business-app/v1/product/getByProductId`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/x-www-form-urlencoded |
| Authorization | string | 是 | Bearer {access_token} |

**Query 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| productId | string | 是 | 产品ID |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/product/getByProductId?productId=zuNuadqzsxEh75" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Authorization: Bearer {access_token}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": "zuNuadqzsxEh75",
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
  },
  "reqId": "4a0ede228f9f6fac",
  "time": 1620641786
}
```

**响应字段说明**:
- `id`: 产品ID
- `productName`: 产品名称
- `model`: 产品型号
- `imageUrl`: 产品图片
- `protocolType`: 通讯协议(MQTT/BLE等)
- `distributionNetMode`: 配网模式(1=wifi+ble, 2=wifi, 3=ble)
- `nodeType`: 节点类型(1=普通设备, 2=网关, 3=边缘网关, 4=子设备)
- `bindMode`: 绑定模式(1=强绑, 2=弱绑)
- `deviceShare`: 是否支持设备共享(0=否, 1=是)
- `deviceUpgrade`: 是否支持设备升级(0=否, 1=是)
- `status`: 状态(0=禁用, 1=启用)

---

### 3. 配网数据加密

配网过程中对数据进行加密处理。

**接口地址**: `POST /business-app/v1/distributionNet/dataEncrypt`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| content | string | 是 | 待加密内容，为 JSON 序列化后的字符串，详见下方 content 字段说明 |
| encryptType | integer | 是 | 加密类型，固定值：0 |
| protocol | string | 是 | 协议版本，固定值：1 |
| type | string | 是 | 配网类型，固定值：thing.network.set |

**content 字段说明**（序列化为字符串后传入）:

| 字段 | 类型 | 说明 |
|------|------|------|
| force_bind | boolean | 是否强制绑定设备 |
| sid | string | 路由器 SSID |
| pw | string | 路由器密码 |
| mq | string | MQTT host 地址 |
| areaCode | string | 区域码，如 US、CN |
| mqttSslPort | string | MQTT SSL 端口，默认 8883 |
| bid | string | Bind ID，设备绑定标识即设备资产ID |
| port | integer | MQTT 端口，默认 1883 |
| country | string | 国家代码：CN / US / JP / EP / AU |
| userID | string | 用户ID |
| tz | string | 用户所在时区，如 Asia/Shanghai |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/distributionNet/dataEncrypt" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{
    "content": "{ \"areaCode\":\"US\", \"bid\":\"ssssss321345e\", \"country\":\"CN\", \"force_bind\":true, \"mq\":\"web.tlyzn.com\", \"mqttSslPort\":\"8883\", \"port\":1883, \"pw\":\"31415926\", \"sid\":\"XXsmart-2.4G\", \"tz\":\"Asia/Shanghai\", \"userID\":\"\" }",
    "encryptType": 0,
    "protocol": "1",
    "type": "thing.network.set"
  }'
```

**响应示例**:

```json
{
    "reqId": "b90b3f0a1ebb157c",
    "time": 1771920886,
    "code": 200,
    "message": "成功",
    "data": "{\"type\":\"thing.network.set\",\"data\":{\"country\":\"CN\",\"force_bind\":true,\"areaCode\":\"US\",\"mq\":\"web.tlyzn.com\",\"port\":1883,\"tz\":\"Asia/Shanghai\",\"pw\":\"31415926\",\"mqttSslPort\":\"8883\",\"bid\":\"ssssss321345e\",\"userID\":\"\",\"sid\":\"XXsmart-2.4G\"}}"
}
```

**响应字段说明**:
- `code`: 响应状态码
- `message`: 响应消息
- `data`: 加密后的数据
- `reqId`: 请求ID
- `time`: 请求时间戳(10位)

---

### 4. 获取数据中心列表

获取可用的数据中心列表,包含各数据中心的API地址、MQTT地址等配置信息。

**接口地址**: `POST /business-app/v1/common/getDataCenterList`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**请求参数**: 无

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/common/getDataCenterList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}"
```

**响应示例**:

```json
{
  "code": 200,
  "data": [
    {
      "apiUrl": "",
      "appId": "",
      "areaId": "",
      "createTime": "",
      "dataCenterCode": "",
      "dataCenterId": "",
      "dataCenterName": "",
      "iotUrl": "",
      "mqttSslPort": "",
      "mqttUrl": "",
      "sort": 0
    }
  ],
  "message": "success",
  "reqId": "4a0ede228f9f6fac",
  "time": 1620641786
}
```

**响应字段说明**:
- `code`: 响应状态码
- `data`: 数据中心列表
  - `apiUrl`: API地址
  - `appId`: 数据中心对应的appId
  - `areaId`: 区域Id(旧字段,暂时保留作兼容)
  - `createTime`: 创建时间
  - `dataCenterCode`: 数据中心编码
  - `dataCenterId`: 主键id
  - `dataCenterName`: 数据中心名称
  - `iotUrl`: IoT地址
  - `mqttSslPort`: MQTT SSL端口
  - `mqttUrl`: MQTT地址
  - `sort`: 排序
- `message`: 响应消息
- `reqId`: 请求ID
- `time`: 请求时间戳(10位)

---

### 5. 设备绑定状态查询

App 通过轮询定时拉取(10s)获取设备绑定状态。

**接口地址**: `POST /business-app/v1/device/bind/checkBindResult/{uuid}`

**Path 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| uuid | string | 是 | 设备UUID |

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/checkBindResult/ct010t9dxnKmfTdk" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": 0
}
```

**响应说明**:
- `data`: 0=成功,其他值=失败

**使用场景**: 在设备配网过程中,需要定时调用此接口检查设备是否已成功绑定

---

## 设备管理接口

### 6. 获取资产ID

获取用户的家庭房间树状结构,返回资产ID(家庭ID),用于后续设备绑定。

**接口地址**: `POST /business-app/v1/asset/assetTree`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

```json
{}
```

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/asset/assetTree" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -H "timezone: Asia/Shanghai" \
  -H "language: zh_CN" \
  -H "data_center_code: cn" \
  -d "{}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
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

### 7. 获取设备信息

通过蓝牙协议获取配网设备的 UUID 和产品ID,获取设备基础信息。

**接口地址**: `POST /business-app/v1/device/getSimpleDeviceInfo`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| productId | string | 是 | 产品ID |
| uuid | string | 是 | 设备UUID |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getSimpleDeviceInfo" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{"productId": "zuNuadqzsxEh75", "uuid": "ct010t9dxnKmfTdk"}'
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "uuid": "ct010t9dxnKmfTdk",
    "productId": "zuNuadqzsxEh75",
    "deviceName": "智能设备",
    "icon": "https://...",
    "protocolType": "MQTT",
    "configMode": "BLE",
    "bindStatus": 0
  }
}
```

---

### 8. 获取产品品类

列出应用支持的品类。可选择某个品类直接进行配网,或通过蓝牙扫描找到正在配网的设备。

**接口地址**: `POST /business-app/v1/category/top`

**接口说明**: 此接口为非必须接口

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

```json
{}
```

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/category/top" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d "{}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
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

### 9. 获取用户设备以及群组列表

获取用户首页设备、群组以及分享设备列表。

**接口地址**: `POST /business-app/v1/device/getHomeDeviceAndGroupList`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| assetIds | array | 是 | 资产ID数组(家庭ID列表) |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/getHomeDeviceAndGroupList" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{"assetIds":["1983052957022670848","2008080482956288000"]}'
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "deviceList": [
      {
        "deviceId": "2008424975449309184",
        "uuid": "ct010t9dxnKmfTdk",
        "deviceName": "智能设备",
        "online": true,
        "productId": "zuNuadqzsxEh75"
      }
    ],
    "sortIdList": ["2008424975449309184"],
    "shareGroupList": [],
    "shareList": []
  }
}
```

**响应字段说明**:
- `deviceList`: 用户设备信息列表
- `sortIdList`: 用户自定义设备排序,前端可通过此列表调整设备展示顺序
- `shareGroupList`: 被分享的群组列表
- `shareList`: 被分享的设备列表

---

### 10. OTA检查

检查设备是否有可用的固件升级。

**接口地址**: `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}`

**Path 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| deviceId | string | 是 | 设备ID |
| firmwareType | string | 否 | 固件类型 |

**接口说明**:
- 如果只传了设备ID,默认情况下只会检查`模组固件`是否需要升级
- 可通过设置`firmwareType`参数指定固件类型进行检查

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/ota/checkUpgrade/2008424975449309184/module" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "hasUpgrade": true,
    "version": "1.2.0",
    "downloadUrl": "https://...",
    "md5": "abc123...",
    "fileSize": 1024000,
    "upgradeDesc": "修复已知问题,提升稳定性"
  }
}
```

---

### 11. 设备解绑

设备解绑,可选择是否清除数据。

**接口地址**: `POST /business-app/v1/device/bind/unbind`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| deviceId | string | 是 | 设备ID |
| isCleanData | integer | 是 | 是否清除数据(1=清除,0=不清除) |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/bind/unbind" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{"isCleanData": 1, "deviceId": "2008424975449309184"}'
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 智能体管理接口

### 12. 获取官方推荐智能体模板列表

获取官方推荐的智能体模板列表。

**接口地址**: `POST /business-app/v1/agents/recommend/agents-list`

**接口说明**: 智能体模板定义了智能体的 ASR、LLM 和 TTS 相关内容

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

```json
{}
```

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d "{}"
```

**响应示例**:

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

### 13. 获取官方智能体模板详细

获取指定官方智能体模板的详细信息。

**接口地址**: `POST /business-app/v1/agents/detail`

**Query 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| agentId | string | 是 | 智能体ID |

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/detail?agentId=1980570366486159361" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}"
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
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



## 智能体设备接口

### 14. 设备绑定智能体模板

将设备绑定到指定的智能体模板。

**接口地址**: `POST /business-app/v1/agents/device/bind-agent`

**请求头**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| Content-Type | string | 是 | application/json |
| Authorization | string | 是 | Bearer {access_token} |

**Body 参数**:

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| agentId | string | 是 | 智能体ID |
| agentType | string | 是 | 智能体类型(official=官方,customize=自定义) |
| deviceId | string | 是 | 设备ID |

**请求示例**:

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/agents/device/bind-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {access_token}" \
  -d '{
    "agentId": "1947925380891516929",
    "agentType": "official",
    "deviceId": "1111111111"
  }'
```

**响应示例**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

## 错误码说明

| 错误码 | 说明 |
|--------|------|
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未授权,token无效或过期 |
| 403 | 禁止访问 |
| 404 | 资源不存在 |
| 500 | 服务器内部错误 |

---

## 附录

### 应用配置信息

| 配置项 | 值 |
|--------|-----|
| app_id | krfjnsim9vs7yd |
| package_name | com.kingstar.lululand |
| channel_identifier | gk6853gq |
| client_id | Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg== |

### 常用配置

| 配置项 | 默认值 |
|--------|--------|
| timezone | Asia/Shanghai |
| language | zh_CN |
| data_center_code | cn |
| encrypt_type | AES/ECB/PKCS5Padding |

---

**文档版本**: v1.0  
**最后更新**: 2026年2月11日  
**维护者**: Sentino IoT Team
