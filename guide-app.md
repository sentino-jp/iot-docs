# App 端集成指南

> **TL;DR**：本文档覆盖 App 端完整集成：用户登录、BLE 配网（WiFi / 4G 两种模式）、设备管理、智能体管理。按本文档流程实现即可完成 App 端开发。

> **前置知识**：建议先阅读 [架构与概念](./architecture.md)，API 字段细节请查阅 [REST API 参考](./ref-rest-api.md)。

---

## 1. 集成总览

App 端需要实现的功能：

```mermaid
graph LR
    A["用户登录"] --> B["获取账户 ID"]
    B --> C["BLE 配网"]
    C --> D["轮询绑定状态"]
    D --> E["设备管理"]
    E --> F["智能体管理"]
```

| 功能模块 | 说明 |
|---|---|
| 用户登录 | UID 方式注册/登录，获取 access_token |
| 获取账户 ID | 获取用户账户结构，设备绑定时需要 assetId |
| BLE 配网 | 通过蓝牙将绑定信息（+ WiFi 凭证）传给设备 |
| 设备管理 | 设备列表、绑定状态查询、解绑、OTA 检查 |
| 智能体管理 | 浏览/创建 AI 角色，绑定角色到设备 |

---

## 2. 用户登录

### 2.1 登录方式

Sentino 使用 **UID 授权**方式，注册和登录合一：
- UID 不存在 → 自动注册新用户
- UID 已存在 → 直接登录

### 2.2 登录流程

```
POST /auth/oauth/token
```

**请求**：

```bash
curl -X POST "https://api-iot.sentino.jp/auth/oauth/token?grant_type=uid&area_code=86&app_id=krfjnsim9vs7yd" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "app_id: krfjnsim9vs7yd" \
  -H "Authorization: Basic Y2V0dXMtaW90LWFwcDpvbEFESkNtV2xGSVZYWTFxMWx4MHdVclViemU3WHdlUg==" \
  -d "grant_type=uid&uid=YOUR_UID&password=YOUR_PASSWORD&area_code=86&user_country_key=CN"
```

**响应**：

```json
{
  "code": 200,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "bearer",
    "expires_in": 7200,
    "refresh_token": "..."
  }
}
```

### 2.3 Token 使用

登录后所有业务接口需要在请求头中携带：

```
Authorization: Bearer {access_token}
```

Token 有效期 7200 秒（2 小时），过期后使用 `refresh_token` 刷新。

### 2.4 公共请求头

所有业务接口（除登录外）需要携带以下公共请求头：

| Header | 值 | 说明 |
|---|---|---|
| `timezone` | `Asia/Shanghai` | 用户时区 |
| `language` | `zh_CN` | 语言 |
| `data_center_code` | `cn` | 数据中心代码 |
| `client_id` | `Y2V0dXMtaW90LWFwcDpv...` | 客户端标识 |
| `encrypt_type` | `AES/ECB/PKCS5Padding` | 加密类型 |
| `channel_identifier` | `gk6853gq` | 渠道标识 |
| `package_name` | `com.yiyuan` | 包名 |
| `app_id` | `krfjnsim9vs7yd` | 应用 ID |

---

## 3. BLE 配网

配网是用户首次使用设备时将其绑定到自己账号的过程。App 通过 BLE 蓝牙将绑定信息传给设备。

### 3.1 配网前准备

在发起 BLE 配网前，App 需要先获取以下信息：

```mermaid
flowchart TD
    A["1. 用户登录获取 Token"] --> B["2. 获取账户 ID (assetId)"]
    B --> C["3. 扫描设备条码，获取 UUID"]
    C --> D["4. 查询设备信息（确认可配网）"]
    D --> E["5. 请求配网数据加密"]
    E --> F["6. BLE 扫描并连接设备"]
    F --> G["7. BLE 发送配网信息"]
    G --> H["8. 轮询绑定状态"]
```

**获取账户 ID**：

```
POST /business-app/v1/asset/assetTree
```

返回用户账户结构，记录根节点 `assetId`（账户 ID）用于绑定。

**获取设备信息**（可选，用于确认设备状态）：

```
POST /business-app/v1/device/getSimpleDeviceInfo
Body: {"productId": "sEF4ljjdH8mo", "uuid": "ct01wfjSNqGAqUUK"}
```

### 3.2 WiFi 模式配网

WiFi 模式需要将 WiFi 凭证和绑定信息一起传给设备。

#### 步骤 1：请求配网数据加密

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

| content 字段 | 说明 |
|---|---|
| `sid` | WiFi SSID |
| `pw` | WiFi 密码 |
| `bid` | 账户 ID（assetId） |
| `userId` | 用户 ID |
| `mq` | MQTT Broker 地址（来源：数据中心接口的 `mqttUrl` 字段） |
| `port` | MQTT 端口 |
| `mqttSslPort` | MQTT SSL 端口（来源：数据中心接口的 `mqttSslPort` 字段） |
| `country` | 国家代码 |
| `areaCode` | 地区代码（来源：用户信息的 `countryCode` 字段） |
| `tz` | 时区 |
| `force_bind` | 是否强制绑定 |

#### 步骤 2：BLE 扫描并连接设备

| BLE 参数 | 值 |
|---|---|
| Service UUID | `0xA101` |
| 广播数据中包含 | 产品 ID (PID) |
| 扫描应答数据包含 | 设备 UUID/MAC、配网状态标志 |

App 扫描 BLE 设备，筛选 Service UUID 为 `0xA101` 的设备，连接后：

1. 先发送 `device.information.get` 获取设备信息
2. 确认设备可配网后，发送配网信息

#### 步骤 3：获取设备信息

通过 BLE GATT 发送：

```json
{
  "type": "device.information.get",
  "ts": 1742536800
}
```

设备回复：

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

#### 步骤 4：发送配网信息

发送加密后的配网数据（或直接发送明文数据）：

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

设备回复：

```json
{
  "type": "thing.network.set.response",
  "code": 0,
  "ts": 1742536800
}
```

`code` 为 `0` 表示设备已收到配网信息并开始连接 WiFi。

#### 步骤 5：轮询绑定状态

配网信息发送后，App 需要轮询检查设备是否绑定成功：

```
POST /business-app/v1/device/bind/checkBindResult/{uuid}
```

- 轮询间隔：2 秒（参考值，另有文档建议 10 秒，请根据实际场景选择）
- 超时时间：建议 100 秒
- `data` 返回 `0` 表示绑定成功

**方式二：监听 MQTT 消息（推荐）**

App 也可以通过监听 MQTT 消息通道，接收设备绑定结果推送，无需轮询：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "bind_result",
  "data": {
    "devices": [
      {
        "deviceId": "设备ID",
        "uuid": "设备UUID",
        "gatewayId": "网关设备ID",
        "type": "add",
        "result": "success",
        "msg": "绑定成功"
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `data.devices[].deviceId` | string | 设备 ID |
| `data.devices[].uuid` | string | 设备 UUID |
| `data.devices[].type` | string | 操作类型（`add` = 添加） |
| `data.devices[].result` | string | 结果（`success` / `fail`） |

### 3.3 4G 模式配网

4G 模式更简单，设备上电即联网，BLE 仅传递绑定信息。

```json
{
  "type": "thing.network.set",
  "data": {
    "bid": "assetId",
    "userId": "userId"
  }
}
```

**与 WiFi 模式的区别**：
- 不需要传递 `sid`（SSID）、`pw`（密码）、`mq`（MQTT 地址）等字段
- 不需要用户选择 WiFi 网络
- 配网体验更快（扫码 → BLE → 完成）

### 3.4 BLE 数据传输要点

App 与设备之间的 BLE 数据传输使用 Rlink BLE V1 协议：

| 参数 | 值 |
|---|---|
| 每包最大大小 | 128 字节 |
| 协议头大小 | 10 字节 |
| 有效数据最大 | 118 字节/包 |
| 包间延迟 | 20 ms |
| 应用层格式 | JSON |

大于 118 字节的 JSON 消息会被自动分包传输。详细的分包协议请参考 [BLE 协议参考](./ref-ble.md)。

### 3.5 配网时序总览

```mermaid
sequenceDiagram
    participant App as 配网 App
    participant Device as IoT 设备
    participant Cloud as Sentino 云端

    App->>Cloud: 登录获取 Token
    App->>Cloud: 获取账户 ID（assetId）
    Note over App: 用户扫描设备条码

    App->>Device: BLE 扫描（Service UUID: 0xA101）
    App->>Device: BLE 连接
    App->>Device: BLE: device.information.get
    Device-->>App: BLE: 设备信息（pid, uuid, bind 状态）

    alt WiFi 模式
        Note over App: 用户选择 WiFi 并输入密码
        App->>Cloud: 请求配网数据加密
        Cloud-->>App: 返回加密数据
        App->>Device: BLE: thing.network.set（WiFi + 绑定信息）
    else 4G 模式
        App->>Device: BLE: thing.network.set（仅绑定信息）
    end

    Device-->>App: BLE: thing.network.set.response（code=0）

    Device->>Cloud: MQTT 连接 + 上报 bind
    Cloud-->>Device: 绑定成功

    loop 每 2 秒
        App->>Cloud: 轮询 checkBindResult
    end
    Cloud-->>App: 绑定成功（data=0）
    App->>App: 显示"添加成功"
```

---

## 4. 设备管理

### 4.1 获取设备列表

获取用户所有设备：

```
POST /business-app/v1/device/getHomeDeviceAndGroupList
Body: {"assetIds": ["assetId1", "assetId2"]}
```

响应中的关键字段：

| 字段 | 说明 |
|---|---|
| `deviceList` | 设备信息列表 |
| `deviceList[].deviceId` | 设备 ID（用于后续操作） |
| `deviceList[].uuid` | 设备 UUID |
| `deviceList[].online` | 是否在线 |
| `sortIdList` | 用户自定义排序 |
| `shareList` | 被分享的设备列表 |

### 4.2 设备解绑

```
POST /business-app/v1/device/bind/unbind
Body: {"deviceId": "设备ID", "isCleanData": 1}
```

| `isCleanData` | 说明 |
|---|---|
| `0` | 不清除设备数据 |
| `1` | 清除设备数据（恢复出厂） |

### 4.3 OTA 升级检查

```
POST /business-app/v1/ota/checkUpgrade/{deviceId}/{firmwareType}
```

响应：

```json
{
  "code": 200,
  "data": {
    "hasUpgrade": true,
    "version": "1.2.0",
    "downloadUrl": "https://...",
    "upgradeDesc": "修复已知问题"
  }
}
```

---

## 5. 智能体管理

### 5.1 浏览官方智能体

```
POST /business-app/v1/agents/recommend/agents-list
Body: {}
```

返回官方推荐的 AI 角色列表，每个角色包含名称、头像、描述、标签等信息。

### 5.2 查看智能体详情

```
POST /business-app/v1/agents/detail?agentId={agentId}
```

返回智能体的完整配置，包括 LLM 模型、TTS 声音等。

### 5.3 绑定智能体到设备

```
POST /business-app/v1/agents/device/bind-agent
Body: {
  "agentId": "智能体ID",
  "agentType": "official",
  "deviceId": "设备ID"
}
```

| `agentType` | 说明 |
|---|---|
| `official` | 官方智能体 |
| `customize` | 用户自定义智能体 |

绑定后，设备将使用该智能体。

### 5.4 智能体管理流程

```mermaid
flowchart TD
    A["获取推荐列表"] --> B{"用户选择"}
    B -->|"选择官方角色"| C["查看角色详情"]
    B -->|"选择自定义角色"| D["查看自定义角色详情"]
    C --> E["绑定到设备"]
    D --> E
    E --> F["设备使用新角色"]
```

---

## 6. 错误处理

### 6.1 HTTP 错误码

| 错误码 | 说明 | 处理方式 |
|---|---|---|
| 200 | 成功 | — |
| 400 | 请求参数错误 | 检查请求体 |
| 401 | Token 无效或过期 | 刷新 Token 或重新登录 |
| 403 | 禁止访问 | 检查权限 |
| 404 | 资源不存在 | 检查 ID 是否正确 |
| 500 | 服务器内部错误 | 重试或联系技术支持 |

### 6.2 配网失败处理

| 失败场景 | 处理方式 |
|---|---|
| BLE 连接失败 | 确认设备在配网模式，重新扫描 |
| thing.network.set 回复 code 非 0 | 检查发送的数据格式 |
| 轮询超时（100 秒未绑定成功） | 提示用户重试，检查网络连接 |
| WiFi 连接失败 | 确认 SSID 和密码正确，路由器支持 2.4GHz |

---

**相关文档**：[架构与概念](./architecture.md) | [BLE 协议参考](./ref-ble.md) | [REST API 参考](./ref-rest-api.md)
