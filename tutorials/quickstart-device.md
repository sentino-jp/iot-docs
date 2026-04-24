# 快速入门 — 设备端

本文档帮助你在 **10 分钟内**验证与 Sentino MQTT Broker 的连通性。你将使用测试三元组连接 MQTT，发送一条时间查询请求并收到云端回复。

> **前置知识**：建议先阅读 [架构与概念](../architecture.md) 了解整体架构和术语。

---

## 你需要准备什么

- 一台能联网的电脑（macOS / Linux / Windows）
- Python 3.7+（方式一）或 mosquitto CLI 工具（方式二）
- Sentino 提供的**测试三元组**（本文档使用示例数据）

**测试三元组（示例）：**

| 字段 | 值 |
|---|---|
| UUID | `ct01wfjSNqGAqUUK` |
| KEY | `944e53cda6ac4491ad7d453e3d2934bb` |

**产品 ID（pid）**：`sEF4ljjdH8mo`（用于 MQTT Topic 路径，非三元组字段）

> 正式开发时请使用 Sentino 分配给你的三元组，不要使用示例数据。

---

## 方式一：Python 脚本（推荐）

### 第 1 步：安装依赖

```bash
pip install paho-mqtt
```

### 第 2 步：创建测试脚本

将以下内容保存为 `sentino_quickstart.py`：

```python
"""
Sentino IoT Quick Start — 设备端 MQTT 连通性验证
"""
import json
import hmac
import hashlib
import time
import uuid as uuid_lib

import paho.mqtt.client as mqtt

# ============================================================
# 配置区 — 替换为你的三元组和产品 ID
# ============================================================
DEVICE_UUID = "ct01wfjSNqGAqUUK"
DEVICE_KEY  = "944e53cda6ac4491ad7d453e3d2934bb"
PRODUCT_ID  = "sEF4ljjdH8mo"

BROKER_HOST = "mqtt-iot.sentino.jp"
BROKER_PORT = 1883

# ============================================================
# 计算 MQTT 连接参数
# ============================================================
ts = str(int(time.time()))

# Client ID
client_id = f"rlink_{DEVICE_UUID}_V2"

# Username
username = f"{DEVICE_UUID}|signMethod=hmacSha256,ts={ts}"

# Password (HMAC-SHA256 签名)
content = f"uuid={DEVICE_UUID},ts={ts}"
password = hmac.new(
    DEVICE_KEY.encode(), content.encode(), hashlib.sha256
).hexdigest()

# Topic
TOPIC_REPORT   = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/report"
TOPIC_RESPONSE = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/report_response"
TOPIC_ISSUE    = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/issue"

# ============================================================
# MQTT 回调
# ============================================================
def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        print("=" * 50)
        print("MQTT 连接成功!")
        print(f"  Broker:    {BROKER_HOST}:{BROKER_PORT}")
        print(f"  Client ID: {client_id}")
        print(f"  UUID:      {DEVICE_UUID}")
        print("=" * 50)

        # 订阅回复和下发 Topic
        client.subscribe(TOPIC_RESPONSE)
        client.subscribe(TOPIC_ISSUE)
        print(f"\n已订阅:")
        print(f"  {TOPIC_RESPONSE}")
        print(f"  {TOPIC_ISSUE}")

        # 发送时间查询请求
        send_time_request(client)
    else:
        print(f"连接失败, 错误码: {rc}")
        print("  请检查三元组和签名是否正确")


def on_message(client, userdata, msg):
    print(f"\n收到消息 [{msg.topic}]:")
    try:
        data = json.loads(msg.payload.decode())
        print(json.dumps(data, indent=2, ensure_ascii=False))

        code = data.get("code", "")
        res = data.get("res")

        if code == "time" and res == 0:
            print("\n" + "=" * 50)
            print("验证通过! 云端时间查询成功")
            local_time = data.get("data", {}).get("local_date_time", "")
            timezone = data.get("data", {}).get("sys_tz", "")
            print(f"  云端时间: {local_time}")
            print(f"  时区:     {timezone}")
            print("=" * 50)
            print("\n你的 MQTT 连接和通信已验证成功。")
            print("接下来可以尝试发送 bind、info 等其他协议。")
            print("按 Ctrl+C 退出。")

    except json.JSONDecodeError:
        print(msg.payload.decode())


def on_disconnect(client, userdata, rc, properties=None):
    print(f"\nMQTT 断开连接 (rc={rc})")


# ============================================================
# 发送消息
# ============================================================
def send_time_request(client):
    """发送 time 请求 — 获取云端时间"""
    msg_id = str(uuid_lib.uuid4())
    payload = {
        "id": msg_id,
        "ts": int(time.time()),
        "code": "time",
        "ack": 1,
    }
    client.publish(TOPIC_REPORT, json.dumps(payload), qos=1)
    print(f"\n已发送 time 请求 (id: {msg_id[:8]}...)")
    print("  等待云端回复...")


# ============================================================
# 主流程
# ============================================================
def main():
    print("Sentino IoT Quick Start")
    print("-" * 50)
    print(f"正在连接 {BROKER_HOST}:{BROKER_PORT} ...")
    print(f"  Client ID: {client_id}")
    print(f"  Username:  {username}")
    print(f"  Password:  {password[:16]}...")
    print()

    # 创建 MQTT 5.0 客户端
    client = mqtt.Client(
        client_id=client_id,
        protocol=mqtt.MQTTv5,
    )
    client.username_pw_set(username, password)

    # 注册回调
    client.on_connect = on_connect
    client.on_message = on_message
    client.on_disconnect = on_disconnect

    # 连接
    try:
        client.connect(BROKER_HOST, BROKER_PORT, keepalive=60)
        client.loop_forever()
    except KeyboardInterrupt:
        print("\n\n已退出。")
        client.disconnect()
    except Exception as e:
        print(f"\n连接错误: {e}")
        print("请检查网络连接和 Broker 地址。")


if __name__ == "__main__":
    main()
```

### 第 3 步：运行

```bash
python sentino_quickstart.py
```

**预期输出：**

```
Sentino IoT Quick Start
--------------------------------------------------
正在连接 mqtt-iot.sentino.jp:1883 ...
  Client ID: rlink_ct01wfjSNqGAqUUK_V2
  Username:  ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=1742536800
  Password:  894972927a0a6d1a...

==================================================
MQTT 连接成功!
  Broker:    mqtt-iot.sentino.jp:1883
  Client ID: rlink_ct01wfjSNqGAqUUK_V2
  UUID:      ct01wfjSNqGAqUUK
==================================================

已订阅:
  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response
  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue

已发送 time 请求 (id: a1b2c3d4...)
  等待云端回复...

收到消息 [rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response]:
{
  "res": 0,
  "msg": "success",
  "code": "time",
  "data": {
    "ts": 1742536800,
    "sys_tz": "Asia/Shanghai",
    "local_date_time": "2026-03-21 18:00:00",
    ...
  }
}

==================================================
验证通过! 云端时间查询成功
  云端时间: 2026-03-21 18:00:00
  时区:     Asia/Shanghai
==================================================
```

---

## 方式二：mosquitto CLI

如果你更熟悉命令行工具，可以使用 `mosquitto_sub` 和 `mosquitto_pub`。

### 第 1 步：安装 mosquitto

```bash
# macOS
brew install mosquitto

# Ubuntu/Debian
sudo apt install mosquitto-clients

# 或使用 Docker
docker run -it eclipse-mosquitto sh
```

### 第 2 步：计算签名

先用 Python 或在线工具计算 HMAC-SHA256 签名：

```bash
# 计算签名（替换 ts 为当前时间戳）
TS=$(date +%s)
CONTENT="uuid=ct01wfjSNqGAqUUK,ts=$TS"
KEY="944e53cda6ac4491ad7d453e3d2934bb"

# macOS
PASSWORD=$(echo -n "$CONTENT" | openssl dgst -sha256 -hmac "$KEY" | awk '{print $2}')

# Linux
PASSWORD=$(echo -n "$CONTENT" | openssl dgst -sha256 -hmac "$KEY" -hex | awk '{print $NF}')

echo "Username: ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=$TS"
echo "Password: $PASSWORD"
```

### 第 3 步：订阅回复 Topic（终端 1）

```bash
TS=$(date +%s)
CONTENT="uuid=ct01wfjSNqGAqUUK,ts=$TS"
KEY="944e53cda6ac4491ad7d453e3d2934bb"
PASSWORD=$(echo -n "$CONTENT" | openssl dgst -sha256 -hmac "$KEY" | awk '{print $2}')

mosquitto_sub \
  -h mqtt-iot.sentino.jp \
  -p 1883 \
  -V mqttv5 \
  -i "rlink_ct01wfjSNqGAqUUK_V2" \
  -u "ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=$TS" \
  -P "$PASSWORD" \
  -t "rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response" \
  -t "rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue" \
  -v
```

### 第 4 步：发送 time 请求（终端 2）

> 注意：由于 MQTT Client ID 不能重复连接，mosquitto_pub 需要使用不同的 Client ID 或者在同一连接中操作。实际使用时推荐 Python 方式。

```bash
mosquitto_pub \
  -h mqtt-iot.sentino.jp \
  -p 1883 \
  -V mqttv5 \
  -i "rlink_ct01wfjSNqGAqUUK_V2b" \
  -u "ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=$TS" \
  -P "$PASSWORD" \
  -t "rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report" \
  -m '{"id":"test-001","ts":'"$(date +%s)"',"code":"time","ack":1}'
```

在终端 1 中你应该能看到云端的 `time` 回复。

---

## 验证成功后的下一步

你已验证了 MQTT 连接和通信。接下来：

| 目标 | 去哪里看 |
|---|---|
| 了解设备绑定、信息上报等完整流程 | [设备端集成指南](../guides/guide-device.md) |
| 查阅所有 MQTT 协议的字段定义 | [MQTT 协议参考](../reference/ref-mqtt.md) |
| 了解 BLE 配网的实现 | [BLE 协议参考](../reference/ref-ble.md) |
| 了解 AI 语音对话的集成 | [AI 语音对话集成指南](../guides/guide-ai-voice.md) |

---

## 常见问题

### 连接失败，错误码 4 (用户名或密码错误)

1. 检查三元组是否正确（UUID / KEY 不要有多余空格）
2. 检查签名内容格式：`uuid=<UUID>,ts=<时间戳>`（注意逗号前后没有空格）
3. 检查 `ts` 是否为当前时间戳（秒），不能和服务器时间偏差太大
4. 检查 Client ID 格式：`rlink_<UUID>_V2`（固定 `_V2` 后缀）

### 连接成功但收不到回复

1. 检查是否已订阅 `report_response` Topic
2. 检查上报消息的 `ack` 字段是否为 `1`（`ack=0` 时云端不回复）
3. 检查消息 `id` 是否唯一（重复 `id` 的消息会被忽略）

### 连不上 Broker

1. 确认网络能访问 `mqtt-iot.sentino.jp`：`ping mqtt-iot.sentino.jp`
2. 确认端口 `1883` 未被防火墙拦截：`telnet mqtt-iot.sentino.jp 1883`
3. 确认使用的是 MQTT 5.0 协议版本

---

**下一步**：

- 把验证扩展为完整集成 → [设备端集成指南](../guides/guide-device.md) + [MQTT 协议参考](../reference/ref-mqtt.md)
- **跑实物固件（BK7258 + Agora RTC）** → fork [`sentino-jp/sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample)，按 [`device/BUILD_GUIDE.md`](https://github.com/sentino-jp/sentino-iot-sample/blob/main/device/BUILD_GUIDE.md) 编译烧录
- 回到全局 → [架构与概念](../architecture.md)
