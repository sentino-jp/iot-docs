# 设备端命令协议设计

> ⚠️ **本文档已废弃（DEPRECATED 2026-05-15）**
>
> 本文档提出的"设备 → 服务端协议版本协商 + 服务端按版本生成命令 + 设备 `command_result` 反馈"机制**从未实施**——`grep protocol_version|device_capability|capability_ack` 在设备代码（`Conversational-AI-IOT-Sample/device/projects/common_components/sentino_interface/sentino_conv_ai_command.c`）中均无命中。
>
> 此机制已被 [`design.md §A.4`](./design.md#a4-协议版本协商机制评估) 6 条理由否决；2026-05-15 capability 协商架构决策再度确认放弃版本协商方向。
>
> **当前实际机制**：
> - 设计层：[`design.md`](./design.md)（IoT 整体架构 + §A.4 否决理由 + §4 实际版本管理 + §9 跨端归属）
> - 上游 SaaS 设计：DragonFlow `device-control-application.md`（Product 抽象 + Capability Schema 通用框架）
> - 固件接入：[`device-control-firmware-implementation-guide.md`](./device-control-firmware-implementation-guide.md) + DragonFlow `device-control-firmware-integration-guide.md`
> - Capability 协商决策：见 obs `raw/notes/2026-05-15-iot-agent-capability-decision-memo-v5-final.md`（设备实际能力声明走 `/api/v1/conversations.device_info.supported_executors` 字段，per-conversation 上下文 + 服务端做能力交集，**与本文版本协商方向完全不同**）
>
> 本文档保留作**历史记录与决策追溯**，不再作为协议参考。新人请勿按本文实现。
>
> ---
>
> Links: https://ycnicoa7ukxb.feishu.cn/docx/RXIDdaUdlohzvpxa48hc3vJ8nrh
> Author: Agora 张鹏

## 概述

本文档设计了一套支持多版本固件兼容的设备端命令协议方案，解决不同固件版本对动作类型支持差异的问题。该方案确保服务端能够适配不同版本的设备固件，实现向后兼容和渐进式升级。

***

## 设计目标

1. **版本适配**：服务端根据设备协议版本，下发对应版本格式的命令

2. **版本隔离**：不同协议版本使用各自支持的动作类型，互不干扰

3) **版本协商**：设备上报协议版本，服务端记录并适配

4) **版本演进**：支持平滑的版本升级路径，新版本可以新增动作类型

***

## 协议版本管理


### 版本号格式



采用语义化版本号（Semantic Versioning）：`MAJOR.MINOR.PATCH`



* **MAJOR**：协议结构发生重大变更（如命令格式改变、必需字段变更等），但服务端仍需支持旧版本协议，设备可选择升级

* **MINOR**：新增动作类型或功能，服务端可同时支持新旧版本，设备可选择升级

* **PATCH**：bug修复或参数优化，完全兼容，无需关注版本差异

### 版本演进示例



| 固件版本  | 协议版本  | 新增动作                                      | 说明                                       |
| ----- | ----- | ----------------------------------------- | ---------------------------------------- |
| 1.0.0 | 1.0.0 | 基础14种动作                                   | 初始版本                                     |
| 1.0.1 | 1.0.1 | `led_rainbow_effect`, `play_custom_sound` | 新增LED彩虹效果和自定义音效，服务端同时支持1.0.0和1.0.1       |
| 1.1.0 | 1.1.0 | `display_gradient`, `led_breathe`         | 新增渐变显示和呼吸灯效果，服务端同时支持1.0.0、1.0.1和1.1.0    |
| 2.0.0 | 2.0.0 | 协议结构变更                                    | 重大变更（如命令结构改变），服务端同时支持1.x和2.0.0协议，设备可选择升级 |



### 版本升级策略



1. **服务端先升级**：服务端升级到新版本后，仍然支持所有旧版本协议，旧设备继续正常工作

2. **设备选择性升级**：设备可以选择升级到新版本使用新功能，或不升级继续使用旧版本

3) **版本隔离**：不同协议版本的设备可以同时在线，服务端根据设备协议版本下发对应版本的命令

***



## 版本协商机制



### 1. 设备协议版本上报



设备在连接或启动时，向服务端上报协议版本。



#### 设备协议版本上报消息格式



```json
{
  "message_type": "device_capability",
  "device_id": "device_001",
  "protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:00Z"
}
```



#### 字段说明



* `protocol_version`：设备支持的协议版本号（**必需字段**，服务端根据此版本下发命令）

### 2. 服务端版本确认



服务端收到设备协议版本后，记录并确认。



#### 服务端版本确认消息格式



```json
{
  "message_type": "capability_ack",
  "device_id": "device_001",
  "device_protocol_version": "1.0.0",
  "command_protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:01Z"
}
```



#### 字段说明



* `device_protocol_version`：设备上报的协议版本（回显确认）

* `command_protocol_version`：后续命令将使用的协议版本（等于设备协议版本，**关键字段**）

***



## 命令适配机制



### 1. 服务端命令生成策略



服务端在生成命令时，按照以下流程进行：



1. **获取设备协议版本**：从设备协议版本缓存中获取设备的`protocol_version`

2. **查询支持的动作类型**：根据设备协议版本，查询该版本支持的所有动作类型列表

3) **业务层转换**：业务层（如游戏逻辑、语音交互等）将业务需求转换为具体的动作需求

4) **生成动作列表**：

   * 遍历业务层提供的动作需求

   * 验证每个动作类型是否在设备协议版本支持的动作列表中

   * 只添加设备协议版本支持的动作到命令中

5. **组装命令**：

   * 生成唯一的`command_id`

   * 设置`protocol_version`为设备协议版本

   * 添加时间戳

   * 添加动作列表

6. **下发命令**：将生成的命令发送给设备

### 2. 命令结构



命令协议设计为通用的设备控制协议，支持LCD、LED、Audio三类执行器：



```json
{
  "command_id": "cmd_20240101_001",
  "protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:00Z",
  "actions": [
    {
      "executor": "lcd",
      "parameters": {
        "animation_id": "LCD_001",
        "duration": 2000,
        "loop": false
      },
      "priority": 8
    },
    {
      "executor": "led",
      "parameters": {
        "color": "#00FF00",
        "pattern": "blink",
        "blink_count": 3,
        "blink_interval": 200,
        "brightness": 100,
        "duration": 1000
      },
      "priority": 7
    },
    {
      "executor": "audio",
      "parameters": {
        "audio_id": "audio_001",
        "volume": 80,
        "loop": false
      },
      "priority": 9
    }
  ]
}
```



#### 字段说明



* `command_id`：命令唯一标识（必需）

* `protocol_version`：命令使用的协议版本（必需，必须与设备协议版本一致）

* `timestamp`：时间戳（ISO 8601格式，必需）

* `actions`：动作列表（必需，可包含多个动作）

  * `executor`：动作执行者（必需），支持的值：

    * `lcd`：LCD显示屏

    * `led`：LED灯

    * `audio`：音频播放器

  * `parameters`：动作参数（必需，根据不同执行者有不同的参数结构）

  * `priority`：动作优先级（可选，1-10，数字越大优先级越高）

#### 执行器参数定义



##### 1. LCD显示屏（executor: "lcd"）



**基础参数**：

* `animation_id`：动画编号（必需），如"LCD\_001"、"LCD\_002"等

* `duration`：动画持续时间（可选，毫秒）

* `loop`：是否循环播放（可选，布尔值，默认false）

* `speed`：播放速度（可选，如"normal"、"fast"、"slow"）

**示例**：

```json
{
  "executor": "lcd",
  "parameters": {
    "animation_id": "LCD_001",
    "duration": 2000,
    "loop": false
  }
}
```



##### 2. LED灯（executor: "led"）



**基础参数**：

* `color`：LED颜色（可选，十六进制RGB，如"#00FF00"）

* `pattern`：灯效模式（可选），支持的值：

  * `solid`：常亮

  * `blink`：闪烁

  * `breathe`：呼吸灯

  * `rainbow`：彩虹效果

* `brightness`：亮度（可选，0-100）

* `duration`：持续时间（可选，毫秒）

**pattern特定参数**：

* 当`pattern`为`blink`时：

  * `blink_count`：闪烁次数（可选）

  * `blink_interval`：闪烁间隔（可选，毫秒）

* 当`pattern`为`breathe`时：

  * `breathe_speed`：呼吸速度（可选，如"slow"、"normal"、"fast"）

* 当`pattern`为`rainbow`时：

  * `rainbow_speed`：彩虹速度（可选，如"slow"、"normal"、"fast"）

**示例1 - 绿色闪烁**：

```json
{
  "executor": "led",
  "parameters": {
    "color": "#00FF00",
    "pattern": "blink",
    "blink_count": 3,
    "blink_interval": 200,
    "brightness": 100,
    "duration": 1000
  }
}
```



**示例2 - 彩虹效果**：

```json
{
  "executor": "led",
  "parameters": {
    "pattern": "rainbow",
    "rainbow_speed": "medium",
    "brightness": 80,
    "duration": 3000
  }
}
```



##### 3. 音频播放器（executor: "audio"）



**基础参数**：

* `audio_id`：音频编号（必需），如"audio\_001"、"audio\_002"等

* `volume`：音量（可选，0-100，默认80）

* `loop`：是否循环播放（可选，布尔值，默认false）

**示例**：

```json
{
  "executor": "audio",
  "parameters": {
    "audio_id": "audio_001",
    "volume": 80,
    "loop": false
  }
}
```



#### 设计原则



* **业务解耦**：命令协议不包含业务场景信息，业务层负责将业务需求转换为具体的执行器动作

* **通用性**：使用通用的执行器（lcd/led/audio）和参数结构，不限定具体业务场景

* **可扩展性**：参数结构可以灵活扩展，新版本可以新增可选参数，通过版本匹配机制确保设备只收到其版本支持的参数

* **专注性**：命令协议只关注"让哪个执行器执行什么动作"，不关心"为什么执行"

***



## 协议版本参数支持映射表



服务端需要维护每个协议版本对各执行器支持的参数列表：



### 协议版本参数定义



```json
{
  "protocol_versions": {
    "1.0.0": {
      "lcd": {
        "required_parameters": ["animation_id"],
        "optional_parameters": ["duration", "loop", "speed"]
      },
      "led": {
        "required_parameters": [],
        "optional_parameters": ["color", "pattern", "brightness", "duration"],
        "supported_patterns": ["solid", "blink"],
        "pattern_parameters": {
          "blink": ["blink_count", "blink_interval"]
        }
      },
      "audio": {
        "required_parameters": ["audio_id"],
        "optional_parameters": ["volume", "loop"]
      }
    },
    "1.0.1": {
      "lcd": {
        "required_parameters": ["animation_id"],
        "optional_parameters": ["duration", "loop", "speed", "brightness", "position"]
      },
      "led": {
        "required_parameters": [],
        "optional_parameters": ["color", "pattern", "brightness", "duration", "led_index"],
        "supported_patterns": ["solid", "blink", "breathe", "rainbow"],
        "pattern_parameters": {
          "blink": ["blink_count", "blink_interval"],
          "breathe": ["breathe_speed"],
          "rainbow": ["rainbow_speed"]
        }
      },
      "audio": {
        "required_parameters": ["audio_id"],
        "optional_parameters": ["volume", "loop", "fade_in", "fade_out", "playback_rate"]
      }
    }
  }
}
```



### 版本参数继承规则（建议）



* **向后兼容**：新版本协议建议包含所有旧版本的必需参数和可选参数，保持功能连续性

* **参数扩展**：新版本可以新增可选参数来扩展功能

* **参数废弃**：如果确需废弃某个参数，应该在下一个MAJOR版本中处理

* **版本匹配**：服务端根据设备协议版本精确下发命令，确保设备只收到其版本支持的参数（技术保障）

***



## 设备端处理机制



### 1. 动作执行流程



设备端收到命令后的处理流程：

1. 解析命令，提取protocol\_version

2. 验证protocol\_version是否与设备协议版本一致

3. 如果版本不一致：

   1. 拒绝执行命令

   2. 返回错误信息

4. 如果版本一致：

   1. 遍历actions列表

   2. 对每个action验证：

      1. executor是否是设备支持的执行器（lcd/led/audio）

      2. parameters中的必需参数是否存在

      3. 验证参数有效性（类型、取值范围等）

         1. 如果executor不支持，记录错误但继续处理其他动作

         2. 如果executor支持，使用parameters执行动作

5. 返回执行结果

注：正常情况下，由于服务端按设备协议版本精确下发命令，设备不应收到不认识的参数

### 2. 执行结果反馈



设备端执行命令后，向服务端反馈执行结果：



```json
{
  "message_type": "command_result",
  "command_id": "cmd_20240101_001",
  "device_id": "device_001",
  "protocol_version": "1.0.0",
  "status": "success|partial|failed",
  "executed_actions": [
    {
      "executor": "lcd",
      "status": "success"
    },
    {
      "executor": "led",
      "status": "success"
    }
  ],
  "failed_actions": [],
  "timestamp": "2024-01-01T10:00:02Z"
}
```



#### 字段说明



* `protocol_version`：设备协议版本（用于确认）

* `status`：

  * `success`：所有动作都成功执行

  * `partial`：部分动作成功执行（有失败的动作）

  * `failed`：所有动作都失败

* `executed_actions`：成功执行的动作列表

* `failed_actions`：执行失败的动作列表（包括不支持的executor）

***



## 版本兼容性规则



### 1. 向后兼容规则（建议遵循）



虽然服务端按版本精确下发命令，技术上新版本可以删除旧版本的参数，但为了保持功能连续性和简化维护，强烈建议遵循以下规则：



**规则1：新版本协议应包含所有旧版本的基础功能**



* 协议1.0.1应该支持协议1.0.0的所有执行器（lcd/led/audio）和基础参数

* 新版本可以新增参数扩展功能，但不应删除旧版本的核心参数

* **目的**：避免升级后失去功能，保持用户体验一致性

**规则2：参数扩展应向后兼容**



* 新版本优先通过新增可选参数来扩展功能

* 避免删除或修改已有必需参数的语义

* 避免改变参数的数据类型

**参数兼容示例**：

```json
// 协议1.0.0 - LED基础参数
{
  "executor": "led",
  "parameters": {
    "color": "#00FF00",
    "pattern": "blink",
    "blink_count": 3,
    "blink_interval": 200,
    "brightness": 100
  }
}

// 协议1.0.1（向后兼容）- LED扩展参数
{
  "executor": "led",
  "parameters": {
    "color": "#00FF00",
    "pattern": "blink",
    "blink_count": 3,
    "blink_interval": 200,
    "brightness": 100,
    "led_index": 0,         // 新增可选参数
    "gradient": {...}       // 新增可选参数
  }
}
```



```plain&#x20;text

**何时可以打破向后兼容**：

在以下情况下，可以在新版本中删除或修改旧版本的参数：

1. **MAJOR版本升级**（如1.x.x → 2.0.0）：可以进行破坏性变更
2. **废弃过时功能**：某个功能确实不再需要或被更好的方案替代
3. **修复设计缺陷**：旧版本设计有严重问题需要重构

### 2. 版本匹配规则（技术保障）

**规则1：使用设备协议版本**

- 服务端下发命令时，必须使用设备的协议版本
- 命令中的`protocol_version`字段必须与设备协议版本一致
- 命令中的`actions`只能包含该协议版本支持的执行器和参数

**规则2：版本一致性验证**

- 设备端收到命令后，验证`protocol_version`是否与自身协议版本一致
- 如果版本不一致，设备端应拒绝执行命令并返回错误

---
```


完整交互流程示例
--------

### 场景1：设备协议版本1.0.0，服务端知道协议1.0.1

#### 步骤1：设备协议版本上报



```json
{
  "message_type": "device_capability",
  "device_id": "device_001",
  "protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:00Z"
}
```

#### 步骤2：服务端版本确认



```json
{
  "message_type": "capability_ack",
  "device_id": "device_001",
  "device_protocol_version": "1.0.0",
  "command_protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:01Z"
}
```



#### 步骤3：服务端生成命令



服务端根据设备协议版本1.0.0，只使用1.0.0支持的执行器和参数生成命令：



```json
{
  "command_id": "cmd_001",
  "protocol_version": "1.0.0",
  "timestamp": "2024-01-01T10:00:02Z",
  "actions": [
    {
      "executor": "lcd",
      "parameters": {
        "animation_id": "LCD_001",
        "duration": 2000,
        "loop": false
      },
      "priority": 8
    },
    {
      "executor": "led",
      "parameters": {
        "color": "#00FF00",
        "pattern": "blink",
        "blink_count": 3,
        "blink_interval": 200,
        "brightness": 100
      },
      "priority": 7
    },
    {
      "executor": "audio",
      "parameters": {
        "audio_id": "audio_001",
        "volume": 80,
        "loop": false
      },
      "priority": 9
    }
  ]
}
```



#### 步骤4：设备执行并反馈



```json
{
  "message_type": "command_result",
  "command_id": "cmd_001",
  "device_id": "device_001",
  "protocol_version": "1.0.0",
  "status": "success",
  "executed_actions": [
    {"executor": "lcd", "status": "success"},
    {"executor": "led", "status": "success"},
    {"executor": "audio", "status": "success"}
  ],
  "timestamp": "2024-01-01T10:00:03Z"
}
```



### 场景2：设备协议版本1.0.1，服务端知道协议1.0.1



#### 步骤1：设备协议版本上报



```json
{
  "message_type": "device_capability",
  "device_id": "device_002",
  "protocol_version": "1.0.1",
  "timestamp": "2024-01-01T10:05:00Z"
}
```



#### 步骤2：服务端版本确认



```json
{
  "message_type": "capability_ack",
  "device_id": "device_002",
  "device_protocol_version": "1.0.1",
  "command_protocol_version": "1.0.1",
  "timestamp": "2024-01-01T10:05:01Z"
}
```



#### 步骤3：服务端生成命令



服务端根据设备协议版本1.0.1，可以使用1.0.1新增的参数生成命令：



```json
{
  "command_id": "cmd_002",
  "protocol_version": "1.0.1",
  "timestamp": "2024-01-01T10:05:02Z",
  "actions": [
    {
      "executor": "lcd",
      "parameters": {
        "animation_id": "LCD_002",
        "duration": 2000,
        "loop": false,
        "brightness": 90,       // 1.0.1新增参数
        "position": {"x": 0, "y": 0}  // 1.0.1新增参数
      },
      "priority": 8
    },
    {
      "executor": "led",
      "parameters": {
        "pattern": "rainbow",   // 1.0.1新增的pattern值
        "rainbow_speed": "medium",
        "brightness": 80,
        "duration": 3000
      },
      "priority": 7
    },
    {
      "executor": "audio",
      "parameters": {
        "audio_id": "audio_002",
        "volume": 80,
        "loop": false,
        "fade_in": 500,         // 1.0.1新增参数
        "fade_out": 500         // 1.0.1新增参数
      },
      "priority": 9
    }
  ]
}
```

&#x20;     &#x20;

````plain&#x20;text

#### 步骤4：设备执行并反馈

```json
{
  "message_type": "command_result",
  "command_id": "cmd_002",
  "device_id": "device_002",
  "protocol_version": "1.0.1",
  "status": "success",
  "executed_actions": [
    {"executor": "lcd", "status": "success"},
    {"executor": "led", "status": "success"},
    {"executor": "audio", "status": "success"}
  ],
  "timestamp": "2024-01-01T10:05:03Z"
}
````



***



## 版本升级建议



### 1. 服务端升级策略



* **服务端先升级**：服务端升级到新版本后，需要同时支持所有旧版本协议

* **多版本并存**：服务端维护协议版本参数映射表，支持多个协议版本同时存在

* **向后兼容**：新版本服务端必须能够处理旧版本设备的命令请求

### 2. 设备升级策略



* **选择性升级**：设备可以选择升级到新版本使用新功能，或不升级继续使用旧版本

* **OTA推送**：逐步推送设备固件OTA更新，不强制所有设备同时升级

* **版本隔离**：不同协议版本的设备可以同时在线，互不干扰

***



## 实现建议



### 1. 服务端实现



* 维护设备协议版本缓存（Redis/内存），key为device\_id，value为protocol\_version

* 维护协议版本参数映射表（配置文件或数据库），定义每个版本支持的执行器参数

* 实现命令生成器，根据设备协议版本选择对应的执行器和参数

* 业务层（如游戏逻辑）生成抽象的动作需求，命令生成器将其转换为具体的执行器指令

* 记录版本匹配和执行日志

### 2. 设备端实现



* 实现执行器注册表（Registry Pattern），按协议版本注册支持的执行器和参数

* 实现参数验证器，验证执行器参数的有效性（类型、取值范围等）

* 实现版本验证器，验证命令的protocol\_version是否与设备协议版本一致

* 实现执行结果上报

***



## 总结



本协议设计通过以下机制实现了多版本固件兼容：



1. **版本上报机制**：设备上报协议版本，服务端记录

2. **版本匹配机制**：服务端根据设备协议版本，查找协议映射表，生成该版本支持的命令（核心机制）

3) **版本一致性验证**：设备端验证命令的协议版本是否与自身版本一致

4) **版本隔离机制**：不同协议版本使用各自支持的执行器参数，互不干扰

5. **精确匹配保障**：服务端精确下发设备版本支持的参数，设备不会收到不认识的参数

6. **向后兼容建议**：建议新版本包含旧版本的核心功能，让业务层无需编写版本判断逻辑，保持用户体验一致

### 协议设计优势



* **业务解耦**：命令协议与具体业务场景（如游戏）解耦，具备通用性

* **硬件直接映射**：executor直接对应硬件组件（LCD、LED、Audio），易于理解和实现

* **灵活扩展**：通过新增可选参数或新的pattern值，可以灵活扩展功能

* **版本隔离**：不同版本设备可以同时在线，互不干扰

* **实现简单**：无需复杂的降级策略和fallback机制，服务端直接按版本匹配下发

### 实施保证



* 服务端可以根据设备协议版本精确下发对应版本的命令

* 不同版本的设备可以同时在线，互不干扰

* 版本升级过程平滑，新版本设备可以使用新功能，旧版本设备继续使用旧功能

* 业务层只需关注"想达到什么效果"，命令生成器负责转换为设备能理解的指令
