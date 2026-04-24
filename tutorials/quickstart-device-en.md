# Quick Start — Device

This document helps you verify connectivity with the Sentino MQTT Broker **in 10 minutes**. You will use a test triplet to connect to MQTT, send a time query request, and receive a reply from the cloud.

> **Prerequisites**: We recommend reading [Architecture & Concepts](../architecture-en.md) first to understand the overall architecture and terminology.

---

## What You Need

- A computer with internet access (macOS / Linux / Windows)
- Python 3.7+ (Method 1) or the mosquitto CLI tools (Method 2)
- A **test triplet** provided by Sentino (this document uses sample data)

**Test triplet (sample):**

| Field | Value |
|---|---|
| UUID | `ct01wfjSNqGAqUUK` |
| KEY | `944e53cda6ac4491ad7d453e3d2934bb` |

**Product ID (pid)**: `sEF4ljjdH8mo` (used in MQTT Topic paths; not part of the triplet)

> For real development, use the triplet that Sentino assigns to you. Do not use the sample data.

---

## Method 1: Python Script (Recommended)

### Step 1: Install Dependencies

```bash
pip install paho-mqtt
```

### Step 2: Create the Test Script

Save the following content as `sentino_quickstart.py`:

```python
"""
Sentino IoT Quick Start — Device-side MQTT connectivity verification
"""
import json
import hmac
import hashlib
import time
import uuid as uuid_lib

import paho.mqtt.client as mqtt

# ============================================================
# Configuration — Replace with your triplet and product ID
# ============================================================
DEVICE_UUID = "ct01wfjSNqGAqUUK"
DEVICE_KEY  = "944e53cda6ac4491ad7d453e3d2934bb"
PRODUCT_ID  = "sEF4ljjdH8mo"

BROKER_HOST = "mqtt-iot.sentino.jp"
BROKER_PORT = 1883

# ============================================================
# Compute MQTT connection parameters
# ============================================================
ts = str(int(time.time()))

# Client ID
client_id = f"rlink_{DEVICE_UUID}_V2"

# Username
username = f"{DEVICE_UUID}|signMethod=hmacSha256,ts={ts}"

# Password (HMAC-SHA256 signature)
content = f"uuid={DEVICE_UUID},ts={ts}"
password = hmac.new(
    DEVICE_KEY.encode(), content.encode(), hashlib.sha256
).hexdigest()

# Topic
TOPIC_REPORT   = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/report"
TOPIC_RESPONSE = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/report_response"
TOPIC_ISSUE    = f"rlink/v2/{PRODUCT_ID}/{DEVICE_UUID}/issue"

# ============================================================
# MQTT callbacks
# ============================================================
def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        print("=" * 50)
        print("MQTT connected successfully!")
        print(f"  Broker:    {BROKER_HOST}:{BROKER_PORT}")
        print(f"  Client ID: {client_id}")
        print(f"  UUID:      {DEVICE_UUID}")
        print("=" * 50)

        # Subscribe to the response and issue Topics
        client.subscribe(TOPIC_RESPONSE)
        client.subscribe(TOPIC_ISSUE)
        print(f"\nSubscribed:")
        print(f"  {TOPIC_RESPONSE}")
        print(f"  {TOPIC_ISSUE}")

        # Send a time query request
        send_time_request(client)
    else:
        print(f"Connection failed, error code: {rc}")
        print("  Please verify that the triplet and signature are correct")


def on_message(client, userdata, msg):
    print(f"\nMessage received [{msg.topic}]:")
    try:
        data = json.loads(msg.payload.decode())
        print(json.dumps(data, indent=2, ensure_ascii=False))

        code = data.get("code", "")
        res = data.get("res")

        if code == "time" and res == 0:
            print("\n" + "=" * 50)
            print("Verification passed! Cloud time query succeeded")
            local_time = data.get("data", {}).get("local_date_time", "")
            timezone = data.get("data", {}).get("sys_tz", "")
            print(f"  Cloud time: {local_time}")
            print(f"  Timezone:   {timezone}")
            print("=" * 50)
            print("\nYour MQTT connection and communication have been verified.")
            print("Next, try sending other protocols such as bind and info.")
            print("Press Ctrl+C to exit.")

    except json.JSONDecodeError:
        print(msg.payload.decode())


def on_disconnect(client, userdata, rc, properties=None):
    print(f"\nMQTT disconnected (rc={rc})")


# ============================================================
# Send messages
# ============================================================
def send_time_request(client):
    """Send a time request — query the cloud time"""
    msg_id = str(uuid_lib.uuid4())
    payload = {
        "id": msg_id,
        "ts": int(time.time()),
        "code": "time",
        "ack": 1,
    }
    client.publish(TOPIC_REPORT, json.dumps(payload), qos=1)
    print(f"\nSent time request (id: {msg_id[:8]}...)")
    print("  Waiting for cloud reply...")


# ============================================================
# Main
# ============================================================
def main():
    print("Sentino IoT Quick Start")
    print("-" * 50)
    print(f"Connecting to {BROKER_HOST}:{BROKER_PORT} ...")
    print(f"  Client ID: {client_id}")
    print(f"  Username:  {username}")
    print(f"  Password:  {password[:16]}...")
    print()

    # Create an MQTT 5.0 client
    client = mqtt.Client(
        client_id=client_id,
        protocol=mqtt.MQTTv5,
    )
    client.username_pw_set(username, password)

    # Register callbacks
    client.on_connect = on_connect
    client.on_message = on_message
    client.on_disconnect = on_disconnect

    # Connect
    try:
        client.connect(BROKER_HOST, BROKER_PORT, keepalive=60)
        client.loop_forever()
    except KeyboardInterrupt:
        print("\n\nExited.")
        client.disconnect()
    except Exception as e:
        print(f"\nConnection error: {e}")
        print("Please check your network connection and the Broker address.")


if __name__ == "__main__":
    main()
```

### Step 3: Run

```bash
python sentino_quickstart.py
```

**Expected output:**

```
Sentino IoT Quick Start
--------------------------------------------------
Connecting to mqtt-iot.sentino.jp:1883 ...
  Client ID: rlink_ct01wfjSNqGAqUUK_V2
  Username:  ct01wfjSNqGAqUUK|signMethod=hmacSha256,ts=1742536800
  Password:  894972927a0a6d1a...

==================================================
MQTT connected successfully!
  Broker:    mqtt-iot.sentino.jp:1883
  Client ID: rlink_ct01wfjSNqGAqUUK_V2
  UUID:      ct01wfjSNqGAqUUK
==================================================

Subscribed:
  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response
  rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/issue

Sent time request (id: a1b2c3d4...)
  Waiting for cloud reply...

Message received [rlink/v2/sEF4ljjdH8mo/ct01wfjSNqGAqUUK/report_response]:
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
Verification passed! Cloud time query succeeded
  Cloud time: 2026-03-21 18:00:00
  Timezone:   Asia/Shanghai
==================================================
```

---

## Method 2: mosquitto CLI

If you prefer command-line tools, you can use `mosquitto_sub` and `mosquitto_pub`.

### Step 1: Install mosquitto

```bash
# macOS
brew install mosquitto

# Ubuntu/Debian
sudo apt install mosquitto-clients

# Or use Docker
docker run -it eclipse-mosquitto sh
```

### Step 2: Compute the Signature

First, compute the HMAC-SHA256 signature with Python or an online tool:

```bash
# Compute the signature (replace ts with the current timestamp)
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

### Step 3: Subscribe to the Response Topic (Terminal 1)

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

### Step 4: Send a time Request (Terminal 2)

> Note: Because an MQTT Client ID cannot be connected twice simultaneously, `mosquitto_pub` needs to use a different Client ID (or operate within the same connection). For real use, the Python approach is recommended.

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

In Terminal 1 you should see the cloud's `time` reply.

---

## Next Steps After Successful Verification

You have verified MQTT connectivity and communication. Next:

| Goal | Where to Go |
|---|---|
| Learn the full flow including device binding and info reporting | [Device Integration Guide](../guides/guide-device-en.md) |
| Look up field definitions for all MQTT protocols | [MQTT Protocol Reference](../reference/ref-mqtt-en.md) |
| Learn how BLE provisioning is implemented | [BLE Protocol Reference](../reference/ref-ble-en.md) |
| Learn how to integrate AI voice conversation | [AI Voice Conversation Integration Guide](../guides/guide-ai-voice-en.md) |

---

## FAQ

### Connection failed, error code 4 (bad username or password)

1. Verify the triplet is correct (UUID / KEY must not contain extra whitespace)
2. Verify the signature content format: `uuid=<UUID>,ts=<timestamp>` (no spaces around the comma)
3. Verify that `ts` is the current timestamp (in seconds) and not too far off from server time
4. Verify the Client ID format: `rlink_<UUID>_V2` (the `_V2` suffix is fixed)

### Connected successfully but no reply received

1. Verify that you have subscribed to the `report_response` Topic
2. Verify that the `ack` field of the report message is `1` (the cloud does not reply when `ack=0`)
3. Verify that the message `id` is unique (messages with duplicate `id` values are ignored)

### Cannot connect to the Broker

1. Verify the network can reach `mqtt-iot.sentino.jp`: `ping mqtt-iot.sentino.jp`
2. Verify that port `1883` is not blocked by a firewall: `telnet mqtt-iot.sentino.jp 1883`
3. Verify that you are using MQTT 5.0 protocol version

---

**Next**:

- Extend this verification into a full integration → [Device Integration Guide](../guides/guide-device-en.md) + [MQTT Protocol Reference](../reference/ref-mqtt-en.md)
- **Run real-hardware firmware (BK7258 + Agora RTC)** → fork [`sentino-jp/sentino-iot-sample`](https://github.com/sentino-jp/sentino-iot-sample) and follow [`device/BUILD_GUIDE.md`](https://github.com/sentino-jp/sentino-iot-sample/blob/main/device/BUILD_GUIDE.md) to build and flash
- Back to the big picture → [Architecture & Concepts](../architecture-en.md)
