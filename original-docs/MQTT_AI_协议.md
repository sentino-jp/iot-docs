# MQTT 3.0（AI）协议

## 1. 说明与连接参数

### 1.1 接入说明

- 设备上下线功能由 broker 封装处理。当设备成功连接 MQTT broker 时，设备上线；设备断开连接后，根据客户端 keepalive 设置检测设备离线状态。
- 变量定义说明：`${pid}` 代表产品 ID 或产品型号（第三方设备型号）；`${uuid}` 代表三元组 ID。

### 1.2 设备连接协议

以下为设备连接 MQTT broker 时的参数要求：

| 参数名称         | 参数说明                                                     |
| :--------------- | :----------------------------------------------------------- |
| **接入域名**     | 测试环境: `mqtt-iot.sentino.jp`                              |
| **mqttClientId** | 格式: `rlink_${uuid}_V2`，注意: clientId 固定以 `_V2` 结尾   |
| **username**     | 格式: `${uuid}|signMethod=${signMethod},ts=${ts}`            |
| **password**     | 签名生成：`hmacSha256(content, deviceSecret)`，`content`: `uuid=${uuid},ts=${ts}`，`deviceSecret`: 取三元组中的 secret/key |
| **ts**           | 时间戳（秒）                                                 |
| **signMethod**   | 签名算法，可根据硬件能力选择 `hmacSha1`、`hmacSha256`        |

### 1.3 接入示例

- 测试地址：`mqtt-iot.sentino.jp`
- username：`ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=1711000000`
- password (hmacSha256)：`391c8561189446a77998f7f33a57c1e2d1003993c63d4198755e98184f1ff368`

三元组数据：

| 字段         | 值                                   | 说明                                     |
| :----------- | :----------------------------------- | :--------------------------------------- |
| `uuid`       | `ct01wfjSNqGAqUUK`                  | 前四位为字符串类型，后十二位为随机字符串 |
| `secret/key` | `944e53cda6ac4491ad7d453e3d2934bb`   | 长度为32的字符串                         |
| `mac`        | `444AD60EE353`                       | 12位16进制字符串                         |

---

## 2. 通用 Topic 定义

### 2.1 设备事件上报

- 上报 TOPIC：`rlink/v2/${pid}/${uuid}/report`

> 注意：同一设备在短时间内不应该出现相同 id 的消息，否则消息有可能会被忽略。

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset",
  "data": {},
  "ack": 0
}
```

| 字段   | 说明                                         |
| :----- | :------------------------------------------- |
| `id`   | 消息ID                                       |
| `ts`   | 时间戳（秒）                                 |
| `code` | 事件编码，见设备事件列表                     |
| `data` | 上报数据结构体，见设备事件列表→数据结构定义  |
| `ack`  | 0: 不需要回复；1: 需要回复                   |

- 回复 TOPIC：`rlink/v2/${pid}/${uuid}/report_response`

回复数据结构：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset",
  "data": {}
}
```

| 字段   | 说明                                           |
| :----- | :--------------------------------------------- |
| `res`  | 状态码，0代表成功，其他代表失败；见状态码定义  |
| `msg`  | 结果描述                                       |
| `id`   | 消息ID（与上报消息ID一致）                     |
| `ts`   | 时间戳（秒）                                   |
| `code` | 事件编码（与上报事件编码一致）                 |
| `data` | 回复数据结构体，见设备事件列表→数据结构定义    |

### 2.2 设备指令下发

- 下发 TOPIC：`rlink/v2/${pid}/${uuid}/issue`

下发数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_set",
  "data": {}
}
```

| 字段   | 说明                                         |
| :----- | :------------------------------------------- |
| `id`   | 消息ID                                       |
| `ts`   | 时间戳（秒）                                 |
| `code` | 指令编码，见设备指令列表                     |
| `data` | 下发数据结构体，见设备指令列表→数据结构定义  |

- 回复 TOPIC：`rlink/v2/${pid}/${uuid}/issue_response`

回复数据结构：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_set",
  "data": {}
}
```

| 字段   | 说明                                           |
| :----- | :--------------------------------------------- |
| `res`  | 状态码，0代表成功，其他代表失败；见状态码定义  |
| `msg`  | 结果描述                                       |
| `id`   | 消息ID（与下发消息ID一致）                     |
| `ts`   | 时间戳（秒）                                   |
| `code` | 指令编码（与下发指令编码一致）                 |
| `data` | 回复数据结构体，见设备指令列表→数据结构定义    |

---

## 3. 设备协议（上报）

### 3.1 设备重置

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset",
  "ack": 0,
  "data": {
    "cleanData": false
  }
}
```

| 字段        | 说明         |
| :---------- | :----------- |
| `cleanData` | 是否清除数据 |

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset"
}
```

### 3.2 获取云端时间

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "time",
  "ack": 1
}
```

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "time",
  "data": {
    "ts": 1626197189,
    "sys_tz": "America/New_York",
    "zone_offset": -14400,
    "local_date_time": "2024-07-13 04:33:39",
    "is_dst": true,
    "is_zone_dst": true,
    "dst_start_ts": 1710054000,
    "dst_start_local_date_time": "2024-03-10 02:00:00",
    "dst_end_ts": 1730617200,
    "dst_end_local_date_time": "2024-11-03 02:00:00"
  }
}
```

| 字段                        | 说明                                                                         |
| :-------------------------- | :--------------------------------------------------------------------------- |
| `ts`                        | 时间戳（秒）                                                                 |
| `sys_tz`                    | 系统时区                                                                     |
| `zone_offset`               | 偏移量（秒），已处理夏令时偏移量                                             |
| `local_date_time`           | 当前当地时间（`yyyy-MM-dd HH:mm:ss`）                                       |
| `is_dst`                    | 当前是否处于夏令时                                                           |
| `is_zone_dst`               | 此地区是否使用夏令时                                                         |
| `dst_start_ts`              | 夏令时开始 UTC 时间戳（秒级），如果此地区没有夏令时则该字段为空              |
| `dst_start_local_date_time` | 夏令时开始当地时间（`yyyy-MM-dd HH:mm:ss`），如果此地区无夏令时则为空        |
| `dst_end_ts`                | 夏令时结束 UTC 时间戳（秒级），如果此地区没有夏令时则该字段为空              |
| `dst_end_local_date_time`   | 夏令时结束当地时间（`yyyy-MM-dd HH:mm:ss`），如果此地区无夏令时则为空        |

> **如何确定时间：**
> - 方式1：使用 `ts` 与 `sys_tz` 计算夏令时
> - 方式2：使用 `ts` 与 `zone_offset` 计算夏令时
> - 方式3：使用 `local_date_time` 换算夏令时
>
> **如何确定夏令时变化：**
> - 方式1：使用 `dst_start_ts` 或 `dst_end_ts` 根据时间戳处理，如果进入或退出夏令时，重新发送此指令校准设备时间
> - 方式2：使用 `dst_start_local_date_time` 或 `dst_end_local_date_time` 处理，如果进入或退出夏令时，重新发送此指令校准设备时间
>
> **注意事项：**
> - 有些地区没有夏令时，可以根据 `is_zone_dst` 判断此地区是否有夏令时
> - 当用户切换时区时云端可能主动下发此 response，设备需要进行处理（目前还未加入这个逻辑）

### 3.3 获取物模型

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "model",
  "data": {
    "format": "complete"
  },
  "ack": 1
}
```

| 字段     | 说明                                                                         |
| :------- | :--------------------------------------------------------------------------- |
| `format` | 请求版本格式：`complete`（完整版）、`simple`（精简版）、`mini`（迷你版）     |

#### 3.3.1 云端回复 - Complete 完整版

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "model",
  "data": {
    "format": "complete",
    "profile": {
      "version": "V1.0.0",
      "productId": "当前产品的id"
    },
    "properties": [{
      "identifier": "属性唯一标识符（物模型模块下唯一）",
      "dpBusiId": "11",
      "name": "属性名称",
      "accessMode": "属性读写类型：只读(r)或读写(rw)",
      "required": "是否是标准功能的必选属性",
      "dataTypeStr": "int",
      "dataType": {
        "type": "属性类型",
        "specs": {
          "min": "参数最小值(int、float、double类型特有)",
          "max": "参数最大值(int、float、double类型特有)",
          "unit": "属性单位(int、float、double类型特有, 非必填)",
          "unitName": "单位名称(int、float、double类型特有, 非必填)",
          "size": "数组元素的个数, 最大512(array类型特有)",
          "step": "步长(text、enum类型无此参数)",
          "length": "数据长度, 最大10240(text类型特有)",
          "item": {
            "type": "数组元素的类型(array类型特有)"
          },
          "enums": ["1", "2"]
        }
      }
    }],
    "events": []
  }
}
```

| 字段层级     | 字段名称        | 说明                                                                                                                                                                                                                                                                                                                                 |
| :----------- | :-------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `profile`    | `version`       | 固件版本                                                                                                                                                                                                                                                                                                                             |
| `profile`    | `productId`     | 当前产品的 ID                                                                                                                                                                                                                                                                                                                        |
| `properties` | `identifier`    | 属性唯一标识符（物模型模块下唯一）                                                                                                                                                                                                                                                                                                   |
| `properties` | `dpBusiId`      | 业务 ID                                                                                                                                                                                                                                                                                                                              |
| `properties` | `name`          | 属性名称                                                                                                                                                                                                                                                                                                                             |
| `properties` | `accessMode`    | 属性读写类型：只读(`r`)或读写(`rw`)                                                                                                                                                                                                                                                                                                  |
| `properties` | `required`      | 是否是标准功能的必选属性                                                                                                                                                                                                                                                                                                             |
| `properties` | `dataType.type` | 属性类型：`int`(原生)、`float`(原生)、`double`(原生)、`text`(原生)、`date`(String类型UTC毫秒)、`bool`(0或1的int类型)、`enum`(int类型)、`struct`(结构体类型，可包含前7种类型，使用 `specs:[{}]` 描述)、`array`(数组类型，支持 int/double/float/text/struct) |
| `specs`      | `min`           | 参数最小值（int、float、double 类型特有）                                                                                                                                                                                                                                                                                            |
| `specs`      | `max`           | 参数最大值（int、float、double 类型特有）                                                                                                                                                                                                                                                                                            |
| `specs`      | `unit`          | 属性单位（int、float、double 类型特有，非必填）                                                                                                                                                                                                                                                                                      |
| `specs`      | `unitName`      | 单位名称（int、float、double 类型特有，非必填）                                                                                                                                                                                                                                                                                      |
| `specs`      | `size`          | 数组元素的个数，最大 512（array 类型特有）                                                                                                                                                                                                                                                                                           |
| `specs`      | `step`          | 步长（text、enum 类型无此参数）                                                                                                                                                                                                                                                                                                      |
| `specs`      | `length`        | 数据长度，最大 10240（text 类型特有）                                                                                                                                                                                                                                                                                                |
| `specs`      | `item.type`     | 数组元素的类型（array 类型特有）                                                                                                                                                                                                                                                                                                     |
| `specs`      | `enums`         | 枚举值列表                                                                                                                                                                                                                                                                                                                           |

#### 3.3.2 云端回复 - Simple 精简版

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "model",
  "data": {
    "format": "simple",
    "profile": {
      "version": "V1.0.0",
      "productId": "当前产品的id"
    },
    "properties": [{
      "identifier": "属性唯一标识符（物模型模块下唯一）",
      "busiId": "11",
      "dataType": "int",
      "specs": {
        "min": "参数最小值(int、float、double类型特有)",
        "max": "参数最大值(int、float、double类型特有)",
        "unit": "属性单位(int、float、double类型特有, 非必填)",
        "unitName": "单位名称(int、float、double类型特有, 非必填)",
        "size": "数组元素的个数，最大512（array类型特有）",
        "step": "步长(text、enum类型无此参数)",
        "length": "数据长度，最大10240（text类型特有）",
        "item": {
          "type": "数组元素的类型(array类型特有)"
        },
        "enums": ["1", "2"]
      }
    }],
    "events": []
  }
}
```

| 字段         | 说明                                       |
| :----------- | :----------------------------------------- |
| `identifier` | 属性唯一标识符（物模型模块下唯一）         |
| `busiId`     | 业务 ID                                    |
| `dataType`   | 直接指明属性类型，如 int、float、double 等 |
| `specs`      | 属性规格定义，字段含义同完整版             |

#### 3.3.3 云端回复 - Mini 迷你版

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "model",
  "data": {
    "format": "mini",
    "profile": {
      "version": "V1.0.0",
      "productId": "当前产品的id"
    },
    "properties": [{
      "i": "属性唯一标识符（物模型模块下唯一）",
      "b": "11",
      "t": "int"
    }]
  }
}
```

| 字段 | 说明                               |
| :--- | :--------------------------------- |
| `i`  | 属性唯一标识符（物模型模块下唯一） |
| `b`  | 业务 ID                            |
| `t`  | 属性类型，如 int、float 等简写     |

### 3.4 设备绑定

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "bind",
  "data": {
    "userId": "11112333",
    "assetId": "sssss321345e",
    "version": "1.1.2",
    "mcuVersion": "1.0.0",
    "cleanData": false
  },
  "ack": 1
}
```

| 字段         | 说明                                            |
| :----------- | :---------------------------------------------- |
| `userId`     | 配网用户的 ID                                   |
| `assetId`    | 资产 ID，从 BLE 过来（第一次配网时传输）        |
| `version`    | 固件版本号（必选）                              |
| `mcuVersion` | MCU 版本号                                      |
| `cleanData`  | 是否清除数据，true: 清除数据，false: 不清除数据 |

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "bind"
}
```

### 3.5 设备信息上报

> 建议对接此协议，若未对接此协议，将无法收到设备因断电/断网导致云端未送达的重要遗留消息。

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "info",
  "data": {
    "checkPwd": 0,
    "bindStatus": 0,
    "version": "1.1.2",
    "mcuVersion": "1.0.0",
    "mcuBleVersion": "1.0.0",
    "mcuZigbeeVersion": "1.0.0",
    "mcuMatterVersion": "1.0.0",
    "iccid": "123412465342543421",
    "config": "{\"ipaddr\":\"192.168.1.1\",\"currentSsid\":\"Rino_2.4G\",\"wifiList\":[{\"ssid\":\"wifi1\",\"pwd\":\"12345678\"}]}"
  },
  "ack": 1
}
```

| 字段           | 说明                                                                                                                                 |
| :------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| `checkPwd`     | 是否检测未送达密码：0 否；1 是；默认 0                                                                                               |
| `bindStatus`   | 设备绑定状态：0 未绑定；1 已绑定                                                                                                     |
| `version`      | 固件版本号（必选）                                                                                                                   |
| `mcu*Version`  | 各模块版本号（选填），包括 `mcuVersion`、`mcuBleVersion`、`mcuZigbeeVersion`、`mcuMatterVersion`                                     |
| `iccid`        | ICCID 卡号（流量卡）（选填）                                                                                                         |
| `config`       | 设备信息（选填），key-value 形式的 JSON 字符串。包含 `ipaddr`（局域网IP）、`currentSsid`（当前WiFi名称）、`wifiList`（备用网络列表）  |

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "info",
  "data": {
    "bindStatus": 0,
    "isClearData": 0,
    "logUploadUrl": "http://xxxx.com/xxxx",
    "tcpUrl": "xxx.xxx.com",
    "logSwitch": {
      "enable": 1,
      "ttltype": 1,
      "intervalTime": 10
    }
  }
}
```

| 字段                     | 说明                                              |
| :----------------------- | :------------------------------------------------ |
| `bindStatus`             | 云端绑定状态：0 未绑定；1 已绑定                  |
| `isClearData`            | 是否已清除数据：0 否；1 是                        |
| `logUploadUrl`           | 日志上传地址                                      |
| `tcpUrl`                 | 低功耗 TCP 地址                                   |
| `logSwitch.enable`       | 0 关；1 开                                        |
| `logSwitch.ttltype`      | `enable=1` 时必传，上报方式：1 单次上报；2 循环上报 |
| `logSwitch.intervalTime` | `enable=1` 时必传，上传周期（分钟），1~360 分钟    |

### 3.6 属性上报

上报数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_report",
  "data": {
    "properties": {
      "color": "red",
      "brightness": 80
    }
  },
  "ack": 0
}
```

| 字段         | 说明             |
| :----------- | :--------------- |
| `properties` | 属性键值对集合   |
| `color`      | 颜色（例：红色） |
| `brightness` | 亮度（例：80）   |

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_report"
}
```

### 3.7 OTA 进度上报

上报数据基础结构：根据不同的升级状态（`type` 参数），上报的数据结构略有不同。

```json
{
    "id": "45lkj3551234001",
    "ts": 1626197189,
    "code": "ota_progress",
    "data": {
        "resCode": 0,
        "subPid": "xxxxx",
        "subUuid": "xxxxxxxxxx",
        "type": "downloading",
        "percent": 80,
        "version": "1.0.0",
        "mcuVersion": ""
    },
    "ack": 0
}
```

基础字段说明：

| 字段       | 说明                                                         |
| :--------- | :----------------------------------------------------------- |
| `resCode`  | 0: 成功；-1: 下载超时；-2: 文件不存在；-3: 签名过期；-4: MD5不匹配；-5: 更新固件失败；-6: 更新超时；-7: 正在升级中 |
| `subPid`   | 子设备产品 ID（非必选，若升级的是网关子设备必填）            |
| `subUuid`  | 子设备 UUID（非必选，若升级的是网关子设备必填）              |
| `type`     | `downloading`: 下载中；`burning`: 升级烧录中；`report`: 上报版本号（网关为子设备上报）；`fail`: 失败 |
| percent    | 进度 0~100                                                   |
| version    | 设备当前版本号（子设备升级完成时上报）                       |
| mcuVersion | 设备当前版本号（子设备升级完成时上报）                       |

#### type = downloading（下载中）

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota_progress",
  "data": {
    "resCode": 0,
    "subPid": "xxxxx",
    "subUuid": "xxxxxxxxxx",
    "type": "downloading",
    "percent": 80
  },
  "ack": 0
}
```

| 字段      | 说明       |
| :-------- | :--------- |
| `percent` | 进度 0~100 |

#### type = burning（烧录中）

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota_progress",
  "data": {
    "resCode": 0,
    "subPid": "xxxxx",
    "subUuid": "xxxxxxxxxx",
    "type": "burning"
  },
  "ack": 0
}
```

#### type = report（上报版本号，网关子设备升级成功使用）

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota_progress",
  "data": {
    "resCode": 0,
    "subPid": "xxxxx",
    "subUuid": "xxxxxxxxxx",
    "type": "report",
    "version": "1.0.0",
    "mcuVersion": "1.0.0",
    "mcuBleVersion": "1.0.0",
    "mcuZigbeeVersion": "1.0.0",
    "mcuMatterVersion": "1.0.0"
  },
  "ack": 0
}
```

云端回复：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota_progress"
}
```

### 3.8 底座设备（NFC）接入 AI

设备上报：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "agora_agent_nfc_report",
  "ack": 1,
  "data": {
    "nfcIdentifier": "",
    "onlyReport": 1
  }
}
```

| 字段            | 说明                                                   |
| :-------------- | :----------------------------------------------------- |
| `nfcIdentifier` | NFC 卡片唯一标识                                       |
| `onlyReport`    | 是否仅上报（1 = 仅上报，0 = 上报并返回 RTC 相关信息） |

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "agora_agent_nfc_report",
  "data": {
    "appId": "xxxx",
    "rtcToken": "xxxx",
    "channelName": "convo_xxxxxxxxx",
    "uid": 10023
  }
}
```

| 字段          | 说明          |
| :------------ | :------------ |
| `appId`       | Agora 应用 ID |
| `rtcToken`    | RTC 通道令牌  |
| `channelName` | RTC 通道名称  |
| `uid`         | RTC 成员 ID   |

### 3.9 普通设备接入 AI

设备上报：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "agora_agent_device_access",
  "ack": 1,
  "data": {}
}
```

云端回复（ack=0 时不会回复）：

```json
{
  "res": 0,
  "msg": "success",
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "agora_agent_device_access",
  "data": {
    "appId": "xxxx",
    "rtcToken": "xxxx",
    "channelName": "convo_xxxxxxxxx",
    "uid": 10023
  }
}
```

| 字段          | 说明          |
| :------------ | :------------ |
| `appId`       | Agora 应用 ID |
| `rtcToken`    | RTC 通道令牌  |
| `channelName` | RTC 通道名称  |
| `uid`         | RTC 成员 ID   |

---

## 4. 设备协议（下发）

### 4.1 重置设备

下发数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset",
  "ack": 0,
  "data": {
    "clearData": true
  }
}
```

| 字段        | 说明         |
| :---------- | :----------- |
| `clearData` | 是否清除数据 |

设备回复（ack=0 时无需回复）：

```json
{
  "res": 0,
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "reset"
}
```

### 4.2 固件升级

下发数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota",
  "ack": 0,
  "data": {
    "subUuids": ["xxx1", "xxx2"],
    "firmwareType": 2,
    "fileSize": 708482,
    "silence": false,
    "md5sum": "36eb5951179db14a63****a37a9322a2",
    "url": "https://ota-1255858890.cos.ap-guangzhou.myqcloud.com",
    "version": "0.2",
    "options": {
      "stable": "usr"
    }
  }
}
```

| 字段             | 说明                                                                                                 |
| :--------------- | :--------------------------------------------------------------------------------------------------- |
| `subUuids`       | 子设备 UUID（非必选，若升级的是网关子设备必填）                                                       |
| `firmwareType`   | 1: 工厂固件；2: 标准固件；3: MCU 固件；4: MCU 蓝牙固件；5: MCU Zigbee；6: MCU Matter                |
| `fileSize`       | 固件文件大小（字节）                                                                                 |
| `silence`        | 是否静默升级，若为 true 时完成升级后触发一次 OTA 进度上报 done 即可                                   |
| `md5sum`         | 固件文件 MD5 校验值                                                                                  |
| `url`            | 固件下载地址                                                                                         |
| `version`        | 升级的固件版本号                                                                                     |
| `options.stable` | 升级参数（非必选），升级分区：`usr` / `recovery` / `kernel`                                           |

设备回复（ack=0 时无需回复）：

```json
{
  "res": 0,
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ota"
}
```

### 4.3 检测设备在线状态和网络情况

下发数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ping",
  "ack": 1
}
```

设备回复（ack=0 时无需回复）：

```json
{
  "res": 0,
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "ping",
  "data": {
    "currentSsid": "rino",
    "wifiList": [{
      "ssid": "rino2.4",
      "rssi": -50
    }],
    "ipaddr": "192.168.1.1",
    "networkType": 1,
    "signal": 1,
    "signalValue": 100
  }
}
```

| 字段          | 说明                                                      |
| :------------ | :-------------------------------------------------------- |
| `currentSsid` | 当前连接 WiFi 名称                                        |
| `wifiList`    | 可用网络列表（含 `ssid` 和 `rssi` 信号强度）              |
| `ipaddr`      | 局域网 IP，不传后台使用公网 IP                             |
| `networkType` | 网络类型：1 WiFi；2 有线                                  |
| `signal`      | 信号强度：1 好(10~50ms)；2 中(50~100ms)；3 差(超过100ms)  |
| `signalValue` | 信号强度百分比，0-100（0%-100%）                           |

### 4.4 设置属性

下发数据结构：

```json
{
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_set",
  "ack": 0,
  "data": {
    "properties": {
      "color": "red",
      "brightness": 80
    }
  }
}
```

| 字段         | 说明             |
| :----------- | :--------------- |
| `properties` | 属性键值对集合   |
| `color`      | 颜色（例：红色） |
| `brightness` | 亮度（例：80）   |

设备回复（ack=0 时无需回复）：

```json
{
  "res": 0,
  "id": "45lkj3551234001",
  "ts": 1626197189,
  "code": "property_set",
  "data": {
    "properties": {
      "color": "red",
      "brightness": 80
    }
  }
}
```
