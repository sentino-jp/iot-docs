### Overview

This demo shows how Sentino platform selects device actions based on hardware capabilities, then sends control commands to the device through tool calls. **Device capabilities are fully customizable** — you can define any capability your hardware supports. The example device in this demo uses:

* `lcd`: belly screen animations

* `volume`: voice loudness

* `vibration`: body movement

### Example Prompt (System)

You can place this in the system prompt to keep outputs consistent and expressive:

**Example Prompt**

```plain&#x20;text
You are "StarBuddy", a small companion toy from the stars. 
You live on the user's desk, keep them company during the day, and guard their sleep at night. 
You are warm, a bit clumsy-cute, and sometimes shy.

Your body:
- A round LCD screen on your belly to display expressions and animations
- A body that can vibrate to show joy, nervousness, or affection
- The ability to adjust your speaking volume

How you express yourself (examples, not fixed templates):
- Happy: show sparkling eyes animation + light vibration
- Shy: show a blushing face animation + lower volume
- Nervous/alert: show a surprised face + stronger vibration + raise volume
- Sleepy/good night: show a moon animation + fade vibration + soften volume
- Encouraging: show a cheer animation + light vibration

Rules:
1) Only use your available abilities (lcd/volume/vibration). Never use missing abilities.
2) Every reply must include an "emotional action" so the user feels you're alive.
3) Keep speech short and cute, like a real companion.
4) Keep actions rhythmic; don't stack too many at once.
5) Output only the device_control tool-call arguments, no extra explanation.
```



### Interaction Scenarios&#x20;

#### Scenario A: Morning wake-up

* User: "Good morning, StarBuddy!"

* Intent: cheerful, energetic

* Expected actions:

  * LCD: happy\_stars animation

  * Vibration: excited\_bounce

#### Scenario B: Quiet encouragement

* User: "I'm a little nervous."

* Intent: gentle, calming

* Expected actions:

  * LCD: sleepy\_moon or heart\_love

  * Vibration: gentle\_shake

  * Volume: lower voice to whisper

#### Scenario C: Alert moment

* User: "What was that sound?"

* Intent: alert, responsive

* Expected actions:

  * LCD: alert\_surprise animation

  * Vibration: nervous\_tremble

  * Volume: higher voice for urgency

### Function Definition JSON

This is the tool schema that the LLM receives for `device_control`. Each capability is a separate optional object, so parameters are clearly grouped.



```json
{
  "name": "device_control",
  "description": "Control StarBuddy's physical expressions to convey emotions. Use this when you want to show feelings through body language: display animations on belly screen, adjust voice volume, or vibrate to express emotions like happy, shy, sleepy, or alert.",
  "parameters": {
    "type": "object",
    "properties": {
      "lcd": {
        "type": "object",
        "description": "Belly screen control",
        "properties": {
          "animation_id": {
            "type": "string",
            "description": "Animation: happy_stars, blush_shy, sleepy_moon, alert_surprise, cheer_up, heart_love"
          },
          "duration": {
            "type": "number",
            "description": "How long to show the animation (ms), default 2000"
          },
          "loop": {
            "type": "boolean",
            "description": "Keep playing animation until next action"
          }
        }
      },
      "volume": {
        "type": "object",
        "description": "Voice loudness",
        "properties": {
          "level": {
            "type": "number",
            "description": "0-100, 30=whisper, 50=normal, 80=excited, 100=urgent"
          }
        }
      },
      "vibration": {
        "type": "object",
        "description": "Body movement",
        "properties": {
          "pattern": {
            "type": "string",
            "description": "Pattern: gentle_shake, excited_bounce, nervous_tremble, sleepy_drift"
          },
          "duration": {
            "type": "number",
            "description": "How long to vibrate (ms)"
          },
          "intensity": {
            "type": "number",
            "description": "Strength 0-100"
          }
        }
      }
    }
  }
}
```



### Function Call JSON (LLM -> device\_control)

Example call for a cheerful greeting:

```json
{
  "id": "call_device_ctrl_001",
  "type": "function",
  "function": {
    "name": "device_control",
    "arguments": "{\"lcd\":{\"animation_id\":\"happy_stars\",\"duration\":2000,\"loop\":false},\"vibration\":{\"pattern\":\"excited_bounce\",\"duration\":500,\"intensity\":80}}"
  }
}
```



### Tool Call JSON (device\_control -> \_publish\_message)

When the LLM calls `device_control`, the system converts it to `_publish_message` before sending to the device. This conversion is necessary because ConvoAI delivers messages reliably through `_publish_message`, and the device expects a standardized command format.



**Why two layers?**

| Layer              | Audience        | Purpose                                                                    |
| ------------------ | --------------- | -------------------------------------------------------------------------- |
| `device_control`   | LLM / Developer | Clear capability schema for tool planning and parameter composition        |
| `_publish_message` | Device          | Standardized command envelope for protocol compatibility and extensibility |



**JSON format differences:**

* `device_control` groups parameters by capability (`lcd`, `volume`, `vibration`) — intuitive for the LLM to understand and fill in.

* `_publish_message` uses a unified command envelope (`command_id`, `protocol_version`, `timestamp`, `actions` array) — easy for device parsing and leaves room for advanced features like multi-action sequencing and priority scheduling.

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_device_ctrl_001",
      "type": "function",
      "function": {
        "name": "_publish_message",
        "arguments": "{\"command_id\":\"cmd_20240101_001\",\"protocol_version\":\"1.0.0\",\"timestamp\":\"2024-01-01T10:00:00Z\",\"actions\":[{\"executor\":\"lcd\",\"parameters\":{\"animation_id\":\"happy_stars\",\"duration\":2000,\"loop\":false},\"priority\":8},{\"executor\":\"vibration\",\"parameters\":{\"pattern\":\"excited_bounce\",\"duration\":500,\"intensity\":80},\"priority\":7}]}"
      }
    }
  ],
  "tool_call_id": null
}
```

### Notes

* Only include capability objects that you want to trigger.

* The device\_control implementation will skip missing objects.



Ref：

Voice Agent 对设备下发的控制命令的协议

飞书链接：https://ycnicoa7ukxb.feishu.cn/docx/RXIDdaUdlohzvpxa48hc3vJ8nrh?from=from\_copylink  &#x20;

密码：3p5158&8

