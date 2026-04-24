# REST API 参考

> **TL;DR**：Sentino IoT 平台 REST API 完整参考。涵盖认证、配网、设备管理、智能体管理 40+ 接口，每个接口附带 curl 示例。

---

## 1. 概述

| 项目 | 值 |
|---|---|
| 基础 URL | `https://api-iot.sentino.jp` |
| 认证方式 | Bearer Token（登录接口使用 Basic Auth） |

> **参考实现**：每个端点都有对应的 Dart 实现，按业务分到 4 个 repository 文件：
> - [`api_auth_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_auth_repository.dart) — §3 认证 / 账号
> - [`api_device_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_device_repository.dart) — §4 配网 + §5 设备管理
> - [`api_agent_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_agent_repository.dart) — §6 智能体管理
> - [`api_ota_repository.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/repositories/api/api_ota_repository.dart) — OTA
>
> 公共请求头注入：[`lib/utils/api_client.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/api_client.dart)；常量：[`lib/utils/app_config.dart`](https://github.com/sentino-jp/sentino-app-sample/blob/main/lib/utils/app_config.dart)

### 1.1 环境变量准备

本文档所有 curl 示例使用环境变量。运行示例前先导出以下变量（按你测试的环境填写）：

```bash
# 应用凭据（由 Sentino 提供，每个客户应用不同）
export APP_ID="krkfvb4s5e91hq"
export CHANNEL_IDENTIFIER="kfvb4s5e"
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
| `language` | `zh_CN` | 用户语言（`zh_CN` / `en_US` / `ja_JP`） |

**客户端环境标识（每次请求都注入）**：

| Header | 示例 | 说明 |
|--------|------|------|
| `os_name` | `android` / `ios` / `windows` | 操作系统类型 |
| `version` | `1.0.0-2604131058` | App 版本号（用于服务端兼容性判断） |
| `devid` | androidId / IDFV | 设备唯一标识 |
| `ua` | Base64 编码 | 格式：`brand\|model\|os\|resolution\|deviceName` |
| `request_id` | UUID v4 | 每次请求唯一 ID（服务端日志追踪） |

> 这 5 个 Header 由客户端 SDK 自动注入。如自己实现 HTTP 客户端（非 Flutter 模板），需手动构造。

### 2.4 公共响应格式

```json
{
  "reqId": "a3f8b2c1-...",
  "time": 1742536800,
  "code": 200,
  "message": "success",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `reqId` | string | 服务端为本次请求分配的 trace ID。客户端日志推荐打印此值，便于联合排障 |
| `time` | int | 服务端处理时间戳（秒） |
| `code` | int | 业务码。`200` = 成功；其他值见 [§7 错误码](#7-错误码) |
| `message` | string | 结果描述 |
| `data` | object/array/null | 业务数据，结构见各接口 |

---

## 接口速查表

按业务分组，点击跳转到对应小节。

### 用户认证 / 账号

| § | 接口 | Method + Path |
|---|---|---|
| [3.1](#31-用户登录授权) | 用户登录（UID 模式 + Password 模式） | `POST /auth/oauth/token` |
| [3.2](#32-发送注册验证码password-模式) | 发送注册验证码 | `POST /business-app/v2/user/register/sendRegisterCode` |
| [3.3](#33-注册password-模式) | 注册（邮箱+密码） | `POST /business-app/v1/user/register/registryByUserName` |
| [3.4](#34-找回密码) | 找回密码（发码 + 重置） | `POST /business-app/v2/user/password/find/sendFindPasswordCode` 等 |
| [3.5](#35-修改密码) | 修改密码 | `POST /business-app/v1/user/password/update/updatePassword` |
| [3.6](#36-登出) | 登出 | `POST /auth/oauth/logout` |
| [3.7](#37-获取用户资料) | 获取用户资料 | `POST /business-app/v1/user/profile` |
| [3.8](#38-更新用户信息) | 更新用户信息 | `POST /business-app/v1/user/updateInfo` |
| [3.9](#39-上传文件) | 上传文件（如头像） | `POST /business-app/v1/file/uploadFile` |

### 配网

| § | 接口 | Method + Path |
|---|---|---|
| [4.1](#41-获取产品信息) | 获取产品信息（查询配网模式） | `POST /business-app/v1/product/getByProductId` |
| [4.2](#42-配网数据加密) | 配网数据加密 | `POST /business-app/v1/distributionNet/dataEncrypt` |
| [4.3](#43-获取数据中心列表) | 获取数据中心列表 | `POST /business-app/v1/common/getDataCenterList` |
| [4.4](#44-设备绑定状态查询) | 设备绑定状态查询（轮询） | `POST /business-app/v1/device/bind/checkBindResult/{uuid}` |
| [4.5](#45-4g-绑定码配网) | 4G 绑定码配网 | `POST /business-app/v1/device/bind/bindDeviceBy4gCode` |
| [4.6](#46-条形码配网) | 条形码配网 | `POST /business-app/v1/device/bind/bindDeviceFromBarcode` |
| [4.7](#47-直接绑定已知-uuid) | 直接绑定（已知 UUID） | `POST /business-app/v1/device/bind/bindDevice` |

### 设备管理

| § | 接口 | Method + Path |
|---|---|---|
| [5.1](#51-获取账户-id) | 获取账户 ID（资产树） | `POST /business-app/v1/asset/assetTree` |
| [5.2](#52-获取设备信息) | 获取设备信息（按 UUID/PID） | `POST /business-app/v1/device/getSimpleDeviceInfo` |
| [5.3](#53-获取产品品类) | 获取产品品类 | `POST /business-app/v1/category/top` |
| [5.4](#54-获取设备列表) | 获取设备列表 | `POST /business-app/v1/device/getHomeDeviceAndGroupList` |
| [5.5](#55-ota-升级检查) | OTA 升级检查 | `POST /business-app/v1/ota/checkUpgrade/{deviceId}/{type}` |
| [5.6](#56-设备解绑) | 设备解绑 | `POST /business-app/v1/device/bind/unbind` |
| [5.7](#57-获取设备详情) | 获取设备详情（按 deviceId） | `POST /business-app/v1/device/getByDeviceId/{deviceId}` |
| [5.8](#58-设备重命名) | 设备重命名 | `POST /business-app/v1/device/initDevice` |
| [5.9](#59-下发设备属性) | 下发设备属性（控制硬件） | `POST /business-app/v1/device/command/propsIssue` |
| [5.10](#510-设备网络检测) | 设备网络检测 | `POST /business-app/v1/device/command/checkSignal` |
| [5.11](#511-获取设备物模型-dp-点) | 获取设备物模型 DP 点 | `POST /business-app/v1/device/getDpInfos/{deviceId}` |

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
| [6.12](#612-更新自定义智能体) | 更新自定义智能体 | `POST /business-app/v1/agents/customize/update` |
| [6.13](#613-解绑智能体设备) | 解绑智能体（设备） | `POST /business-app/v1/agents/device/unbind-agent` |
| [6.14](#614-获取设备绑定的智能体) | 获取设备绑定的智能体 | `POST /business-app/v1/agents/device/getAgentBaseByDeviceId` |
| [6.15](#615-辅助接口语言--音色--llm-列表) | 辅助列表（语言 / 音色 / LLM） | `POST /business-app/v1/agents/customize/{language,voice,llm}-list` |

---

## 3. 用户认证接口

Sentino 同时支持两套认证模式，按你的接入场景二选一：

| 模式 | grant_type | 适用场景 | 关联接口 |
|---|---|---|---|
| **UID 模式**（B2B2C） | `uid` | 品牌方已有自己的用户体系；把 user_id 当 uid 透传给 Sentino，注册登录合一 | 仅 §3.1 |
| **Password 模式**（C 端） | `password` | 直接用 Sentino 的 C 端账号系统（邮箱+密码+验证码） | §3.1 + §3.2~§3.12 |

> 服务端两套都识别，按 `grant_type` 字段路由。白标 Flutter App 模板 (`sentino-app-sample`) 默认走 password 模式。

---

### 3.1 用户登录（授权）

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

#### 模式 A：UID（注册登录合一）

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

#### 模式 B：Password（邮箱+密码）

**Body 参数** (application/x-www-form-urlencoded)：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `grant_type` | string | 是 | 固定值：`password` |
| `username` | string | 是 | 邮箱地址 |
| `password` | string | 是 | 密码（≥6 位） |
| `areaCode` | string | 是 | 国际区号，默认 `86` |
| `countryKey` | string | 是 | 国家代码，默认 `CN` |

> 走 password 模式前，新用户必须先调用 §3.2 + §3.4 完成注册。

**curl 示例**：

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

#### 响应（两种模式相同）

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

### 3.2 发送注册验证码（Password 模式）

```
POST /business-app/v2/user/register/sendRegisterCode
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `countryCode` | string | 是 | 国家码，如 `86` |
| `input` | string | 是 | 邮箱地址或手机号（自动识别） |

**响应 data**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `sendTo` | string | 验证码发送目标（脱敏后的邮箱/手机号） |
| `verifyCodeLength` | int | 验证码长度（动态生成输入框用） |
| `intervalSeconds` | int | 重新发送间隔（秒） |
| `expireSeconds` | int | 验证码过期时间（秒） |

---

### 3.3 注册（Password 模式）

```
POST /business-app/v1/user/register/registryByUserName
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `input` | string | 是 | 邮箱地址 |
| `password` | string | 是 | 密码（≥6 位） |
| `verifyCode` | string | 是 | §3.2 收到的验证码 |
| `countryCode` | string | 是 | 国际区号，默认 `86` |
| `countryKey` | string | 是 | 国家代码，默认 `CN` |

**响应**：`data: true` 表示注册成功；之后调 §3.1 模式 B 登录。

---

### 3.4 找回密码

发送找回密码验证码：

```
POST /business-app/v2/user/password/find/sendFindPasswordCode
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `input` | string | 是 | 邮箱地址 |
| `passwordFindType` | string | 是 | `email_code` 或 `sms_code` |

重置密码：

```
POST /business-app/v1/user/password/find/resetPassword
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `input` | string | 是 | 邮箱地址 |
| `password` | string | 是 | 新密码 |
| `passwordFindType` | string | 是 | 与发送时一致 |
| `verifyCode` | string | 是 | 验证码 |

---

### 3.5 修改密码

```
POST /business-app/v1/user/password/update/updatePassword
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `oldPassword` | string | 是 | 旧密码 |
| `password` | string | 是 | 新密码 |
| `passwordUpdateType` | string | 是 | 固定值：`password` |

> 修改成功后建议自动登出（清除本地 Token）并跳转登录页。

---

### 3.6 登出

```
POST /auth/oauth/logout
```

无请求参数，需带 `Authorization: Bearer $TOKEN`。响应 `data: null`。

---

### 3.7 获取用户资料

```
POST /business-app/v1/user/profile
```

无请求参数。

**响应 data**（User 对象）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` / `userId` / `memberId` | string | 用户 ID（同一值 3 个别名） |
| `nickname` | string? | 昵称 |
| `userName` | string? | 用户名 |
| `email` | string? | 邮箱 |
| `phone` | string? | 手机号 |
| `avatarUrl` | string? | 头像 URL |
| `areaCode` | string? | 区号 |
| `userType` | int | 用户类型 |
| `tz` | string? | 时区 |
| `tempUnit` | string? | 温度单位 |

---

### 3.8 更新用户信息

```
POST /business-app/v1/user/updateInfo
Content-Type: application/json
```

**Body 参数**：动态字段，传任意 User 模型中的可更新字段。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `nickname` | string | 否 | 昵称 |
| `avatarUrl` | string | 否 | 头像 URL |
| 其他字段 | any | 否 | 支持 User 模型中的任意可更新字段 |

---

### 3.9 上传文件

```
POST /business-app/v1/file/uploadFile
Content-Type: multipart/form-data
```

**表单参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `file` | File | 是 | 文件（如头像图片） |

**响应 data**：`string`（文件 URL）。常用于 §3.8 更新头像。

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

### 4.5 4G 绑定码配网

4G 设备开机后产生 5 位数字「绑定码」，用户在 App 输入即可绑定（不走 BLE）。

```
POST /business-app/v1/device/bind/bindDeviceBy4gCode
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assetId` | string | 是 | 资产 ID（账户 ID） |
| `bindCode` | string | 是 | 5 位数字绑定码 |

**响应**：`data: null`，`code: 200` 表示绑定成功。

---

### 4.6 条形码配网

扫描设备外壳条形码完成绑定（4G 与 WiFi 设备通用）。

```
POST /business-app/v1/device/bind/bindDeviceFromBarcode
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assetId` | string | 是 | 资产 ID |
| `barcode` | string | 是 | 设备外壳印刷的条形码 |

---

### 4.7 直接绑定（已知 UUID）

旁路 BLE 配网流程，已知设备 UUID 时直接绑定。

```
POST /business-app/v1/device/bind/bindDevice
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assetId` | string | 是 | 资产 ID |
| `deviceUuid` | string | 是 | 设备 UUID |

> 仅在测试 / 调试场景使用。生产 App 通常不暴露此入口。

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

### 5.7 获取设备详情

按设备 ID 获取完整设备信息（与 §5.2 按 UUID/PID 查询不同，本接口要求设备已绑定）。

```
POST /business-app/v1/device/getByDeviceId/{deviceId}
```

**路径参数**：`deviceId` — 设备 ID

**响应 data**（Device 对象，字段同 §5.4）：

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

### 5.8 设备重命名

设置或修改设备显示名称（接口名为 `initDevice`，但 App 通常作为「重命名」使用）。

```
POST /business-app/v1/device/initDevice
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assetId` | string | 是 | 资产 ID |
| `deviceUuid` | string | 是 | 设备 UUID |
| `deviceName` | string | 是 | 新名称 |

**响应**：`data: null`，`code: 200` 表示成功。

---

### 5.9 下发设备属性

App 控制设备硬件——把属性键值对写入设备。具体属性键（如 `volume`、`led_color`）由设备物模型定义。

```
POST /business-app/v1/device/command/propsIssue
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |
| `data` | object | 是 | 属性键值对，如 `{"volume": 50}` |

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/business-app/v1/device/command/propsIssue" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"deviceId\": \"$DEVICE_ID\", \"data\": {\"volume\": 50}}"
```

> 同步执行结果通过 App 端 MQTT 实时消息推送（具体协议请联系 Sentino 团队获取）。

---

### 5.10 设备网络检测

触发设备上报当前 WiFi/4G 信号强度。

```
POST /business-app/v1/device/command/checkSignal
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |

> 接口本身仅触发；检测结果通过 App 端 MQTT 实时消息推送（含 `signal` 1 好/2 中/3 差、`signalValue` 0-100）。具体协议请联系 Sentino 团队获取。

---

### 5.11 获取设备物模型 DP 点

获取设备的物模型功能点（Data Point）列表，App 据此渲染设备控制面板。

```
POST /business-app/v1/device/getDpInfos/{deviceId}
```

**路径参数**：`deviceId` — 设备 ID

**响应 data**（DeviceDpInfoVO 数组）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | string | DP 点 ID |
| `key` | string | 属性标识符（`volume_set` 等，下发时用作 key） |
| `name` | string | 属性显示名 |
| `type` | string | 取值类型（`value` / `bool` / `enum` / `string`） |
| `value` | dynamic | 当前值 |
| `specs` | string | 模型规格 JSON（含 min/max/step/enum 等） |
| `dpBusiId` | string | 属性业务 ID |
| `imageUrl` | string? | 功能点图标 |
| `valueCastType` | int | `0` = 原始值，`1` = 百分比 |

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

### 6.12 更新自定义智能体

```
POST /business-app/v1/agents/customize/update
Content-Type: application/json
```

**Body 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `agentId` | string | 是 | 智能体 ID |
| `name` | string | 否 | 名称 |
| `description` | string | 否 | 描述 |
| `avatarUrl` | string | 否 | 头像 URL |
| `langId` | string | 否 | 语言 ID（见 §6.15） |
| `llmModelId` | string | 否 | 模型 ID（见 §6.15） |
| `ttsVoiceId` | string | 否 | 音色 ID（见 §6.15） |

**响应**：`data: null`，`code: 200` 表示成功。

---

### 6.13 解绑智能体（设备）

```
POST /business-app/v1/agents/device/unbind-agent
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |

**响应**：`data: null`。解绑后设备进入「无智能体」状态，需要 §6.9 重新绑定才能对话。

---

### 6.14 获取设备绑定的智能体

App 渲染设备面板时常用——查询设备当前绑定的智能体基础信息。

```
POST /business-app/v1/agents/device/getAgentBaseByDeviceId
```

**Query 参数**：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `deviceId` | string | 是 | 设备 ID |

**响应 data**：Agent 对象（字段同 §6.2）。如设备未绑定，返回 `data: null`。

---

### 6.15 辅助接口（语言 / 音色 / LLM 列表）

创建/更新自定义智能体（§6.4 / §6.12）时用于填充下拉框。

#### 6.15.1 获取语言列表

```
POST /business-app/v1/agents/customize/language-list
```

**响应 data**：

```json
[
  {"id": "lang_zh", "name": "中文"},
  {"id": "lang_en", "name": "English"},
  {"id": "lang_ja", "name": "日本語"}
]
```

#### 6.15.2 获取音色列表

```
POST /business-app/v1/agents/customize/voice-list
```

**响应 data**：

```json
[
  {"id": "voice_001", "name": "甜美女声"},
  {"id": "voice_002", "name": "磁性男声"}
]
```

#### 6.15.3 获取 LLM 模型列表

```
POST /business-app/v1/agents/customize/llm-list
```

**响应 data**：

```json
[
  {"id": "model_gpt4", "name": "GPT-4"},
  {"id": "model_gpt35", "name": "GPT-3.5"}
]
```

---

## 7. 错误码

错误分两层：HTTP 状态码（传输层）和业务码（响应体 `code` 字段）。**HTTP 200 不代表成功**——必须再检查业务码 `code === 200`。

### 7.1 HTTP 状态码

| HTTP 状态码 | 说明 |
|---|---|
| `200` | 请求成功送达；仍需检查业务码 |
| `400` | 请求格式错误（缺必填字段、JSON 解析失败等） |
| `401` | Token 无效（Authorization 头丢失或格式错） |
| `403` | 禁止访问（权限不足） |
| `404` | 接口不存在（路径错） |
| `429` | 速率限制（暂未启用，未来生效） |
| `500` | 服务器内部错误 |

### 7.2 业务码

| code | 含义 | 客户端建议处理 |
|------|------|---------------|
| `200` | 成功 | 正常处理 `data` |
| `11013` | Token 失效 | 清除本地缓存的 access_token，跳转登录页 |

> 业务码列表会随平台迭代扩展。遇到本表未列出的非 200 业务码：
> 1. 把 `code`、`message`、`reqId` 完整记录到日志；
> 2. UI 提示用户「服务异常，请稍后重试」；
> 3. 反馈到 Sentino 团队补充本表。

---

> 接口速查表已前移到本文顶部（§2 之后），按业务分组并支持锚点跳转。

---

**相关文档**：[App 端集成指南](../guides/guide-app.md) | [快速入门 — App 端](../tutorials/quickstart-app.md) | [架构与概念](../architecture.md)
