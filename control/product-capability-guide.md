# Product Capability 配置指南

> 面向 **Product Owner / 客户运营 / 接入产品经理**。读完这一份你能：从零创建一个 Product、写好它的 capability 配置、用 Agent Preview 自己验证下发命令是否正确，不需要看代码、不需要等固件团队联调。

---

## 0. 这份文档帮你做什么 / 不做什么

**做**：

- 5 分钟看懂 Product / Capability / Executor 三个概念
- 从零配一个 Product（带可直接抄改的完整 JSON 示例）
- 用 Agent Preview 自验证云端到底有没有正确下发命令
- 改完之后什么时候生效、怎么灰度、怎么回滚
- 一份联调 Checklist 上线前过一遍

**不做**：

- 协议设计原理（→ `~/local/DragonFlow/docs/device-control-application.md`，65KB 决策记录）
- 固件怎么解析这条命令（→ `./firmware-implementation-guide.md`）
- 后端代码改动（→ DragonFlow workflow-api / workflow-engine）

---

## 1. 5 分钟概念入门

### 1.1 一张图

```
┌─────────────────┐    ┌────────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ 你（PO）配置    │ →  │  LLM 看到你    │ →  │ LLM 决定调用 │ →  │ 平台发送到   │ →  │  设备    │
│ Product 的能力  │    │  暴露的能力    │    │ device_      │    │ Agora 数据流 │    │  执行    │
│ capability_     │    │ （tool schema）│    │ control      │    │              │    │          │
│ config (JSONB)  │    │                │    │              │    │              │    │          │
└─────────────────┘    └────────────────┘    └──────────────┘    └──────────────┘    └──────────┘
        ↑                                            │
        │                                            ↓
        │                                  Agent Preview Chat
        │                                  显示一行：
        │                                  📡 command [cmd_xxx] v1.0.0
        └─────── 你能在这里直接看到 ──────  • display_emotion p=7 {...}
```

**核心理念**：配 Product = 告诉 LLM "这台设备能做什么"。LLM 在对话中**自主**判断要不要触发能力、传什么参数。你只配能力清单，不写规则代码。

### 1.2 三个核心名词

以 R1（一款带眼睛的方块机器人）为例：

| 名词 | R1 上的对应 | 一句话解释 |
|---|---|---|
| **Product** | 「R1 v1」这款产品 | 一种 SKU 级别的硬件抽象（与阿里云 IoT Product 同义）。同一款产品的所有设备共用一份 capability 配置。 |
| **Capability** | 「emotion」（眼神情绪） | 设备能做的一类事。每个 capability 有自己的参数 schema（比如 emotion 有一个 `emotion_type` 枚举字段）。 |
| **Executor** | 「display_emotion」（固件 dispatch key） | 固件代码里实际执行 capability 的函数标识。LLM 不需要知道，但 PO 要跟固件团队对齐**字符串严格一致**。 |

### 1.3 为什么不直接让 LLM 生成 raw 命令？

业务方常问。三句话：

1. **Schema 校验**：LLM 偶尔幻觉，平台用 JSON Schema 卡住非法值（不在 enum、超出 min/max 等），不让坏命令到设备。
2. **协议演进解耦**：明天加新能力 = 后台加一行 capability，**不需要 LLM 厂商帮你改 prompt**。
3. **跨设备复用**：同一个 LLM agent 接 R1 / StarBuddy / Lily，capability 不同但 LLM 调用方式统一是 `device_control(...)`。

更深入论证见 [`device-control-application.md` §1.4 FAQ](../../DragonFlow/docs/device-control-application.md)。

---

## 2. 配置工作流（按这个顺序做）

### 2.1 准入清单（开始前确认）

- [ ] 你接入的设备**形态**清楚（R1 / StarBuddy / 新硬件……）
- [ ] **能力清单**已经跟硬件团队对齐（这台设备物理上能做什么：屏幕？震动？灯？喇叭？）
- [ ] 每个能力的 **executor 名字**跟固件团队对齐（白名单，固件按这个字符串 dispatch，**严格大小写一致**）
- [ ] 每个能力的**参数取值范围**确定（比如 emotion 一共支持哪 9 种？震动 intensity 0-100 还是 0-10？）
- [ ] 知道目标 agent 是哪个（capability 通过 agent 关联生效）

### 2.2 创建 Product

进管理后台 → Sidebar 找 **Products** → 点右上角 **Create**。

弹窗里只填四个字段：

| 字段 | 写法 | 说明 |
|---|---|---|
| Name | `r1_v1` / `starbuddy_v1` | SKU 级标识，建议 snake_case + 版本号 |
| Type | `hardware` | 目前几乎都是 hardware；web/mobile/desktop 是 P2 |
| Description | `R1 表情玩具,9 种情绪` | 给团队看的备注，可选 |
| Enabled | `true` | 灰度时可以临时关 |

**注意**：创建时**不填** capability 和 transport——它们独立编辑，避免一次性表单太长。后端会自动给你填默认 transport（`agora_convoai`）和空 capability schema。

### 2.3 编辑 capability_config

回到产品列表，在产品卡片上点 **Capabilities** 按钮，弹出独立编辑器。

当前形态是一个 textarea，直接写 JSON。**P2 表单化编辑器在路上**（见 `~/local/DragonFlow/docs/capability-editor-form-design.md`），但当前阶段就是手写。

写之前先看 §3 的两个完整示例直接抄。

### 2.4 transport_type / transport_config

**目前只有一个选项**：`agora_convoai`，照填即可。这是平台目前唯一打通的下发通道。`transport_config` 留空（`{}`）。

> 未来可能加 `websocket_voice` / `push_notification` 等，那时这一节会展开。

### 2.5 关联 Agent

capability 配置好以后，要**通过 agent 才能生效**。去 Agent 编辑器：

1. 找到目标 agent
2. 在它的 tool 列表里加 `device_control`（或某个产品定制版）
3. 在 tool 的 config 里指定 `product_id` = 你刚才创建的 product

具体 UI 路径见 [`device-control-application.md` §8.4](../../DragonFlow/docs/device-control-application.md)。

### 2.6 ★ 用 Agent Preview 验证下发命令（重点章节）

**这一节是 PO 自验证的关键工具**。不要等固件联调才知道配错了。

打开你刚才关联的 agent → 进入 **Agent Preview** 页面 → 打开 **Chat** 面板 → 跟 agent 自然对话触发能力。

举个例子（截图来自 weather agent + R1 emotion）：

```
You         你夸一夸我啊。
Weather    你真是个很棒的人！我很喜欢和你聊天，你总是能给我带来新的想法和乐趣。
Weather    📡 command [cmd_1778584772600] v1.0.0
              • display_emotion p=7 {"emotion_type":"loving"}
```

最后那一行 `📡 command [...] v1.0.0` 加 `• <executor> p=<priority> {<params>}` **就是 LLM 实际下发到设备的命令**。看到它，说明云端这一段（capability 配置 → tool schema → LLM 调用 → envelope 构造）全通了。

**字段对照**：

- `cmd_1778584772600` ← `command_id`，唯一 ID（可拿这个 ID 去找固件团队对账）
- `v1.0.0` ← `protocol_version`，目前固定
- `display_emotion` ← `executor`，必须跟固件白名单里的字符串完全一致
- `p=7` ← `priority`，多 action 同时下发时设备执行顺序
- `{"emotion_type":"loving"}` ← `parameters`，按 capability schema 校验过的参数

**看不到这一行的常见原因**：

| 现象 | 排查方向 |
|---|---|
| 完全没 📡 行 | LLM 没调 device_control。description 太抽象、capability 没正确关联 agent、system prompt 没引导、或 LLM 模型不支持 function call |
| 有 📡 行但 executor 名字不对 | capability_config 里 `executor` 字段拼错了，或跟固件白名单不一致 |
| 有 📡 行但参数缺字段 | LLM 没传必填字段；检查 capability `parameters.required` + description 是否清晰 |
| 参数值非法 | schema 校验失败的 capability 会被 drop（log warn 但不上 wire），改 LLM prompt 或放宽 enum |

> **同样可以做的**：在 capability 编辑器里有 **Preview** 按钮（带 Mock Args 输入框），不用真 agent 跑也能预览 envelope。但 Agent Preview Chat 是更接近真实链路的验证（包含 LLM 自由决策、prompt 触发率），强烈推荐。

### 2.7 灰度 / 回滚

当前**没有 product 级版本号**，灰度策略：

- **临时关闭**：把 product 的 `enabled` 设 `false`，所有引用它的 agent 立刻拿不到 device_control tool（caveat：在飞会话不受影响）
- **新建一份切流**：直接 `create new product` + 在测试 agent 上关联，用 agent 版本流量分配做 A/B（agent 已有版本机制）
- **回滚**：还原 capability_config JSON 到上一版（建议自己留备份；后端目前不存历史）

---

## 3. 两个完整示例（直接抄改）

### 3.1 R1：emotion-only（最简单的形态）

R1 只有一对眼睛能播 9 种情绪 AVI，capability 配置只有一项：

```json
{
  "capabilities": [
    {
      "name": "emotion",
      "executor": "display_emotion",
      "priority": 7,
      "parameters": {
        "type": "object",
        "properties": {
          "emotion_type": {
            "type": "string",
            "enum": ["happy", "sad", "angry", "surprised", "neutral",
                     "thinking", "sleepy", "loving", "curious"],
            "description": "Emotion to display through R1's eyes"
          }
        },
        "required": ["emotion_type"]
      }
    }
  ]
}
```

**LLM 看到的 tool schema**（自动生成，`ProductToolSchemaBuilder` 干的事）：

```jsonc
{
  "name": "device_control",
  "description": "...",
  "parameters": {
    "type": "object",
    "properties": {
      "emotion": {                                // ← capability.name
        "type": "object",
        "properties": {
          "emotion_type": {
            "type": "string",
            "enum": ["happy", "sad", ...]
          }
        },
        "required": ["emotion_type"]
      }
    }
  }
}
```

**LLM 调用样例**：

```json
{ "emotion": { "emotion_type": "loving" } }
```

**设备收到的 actions[]**（envelope 完整结构见 [`firmware-implementation-guide.md` §3](./firmware-implementation-guide.md)）：

```json
{
  "command_id": "cmd_1778584772600",
  "protocol_version": "1.0.0",
  "timestamp": "2026-05-12T11:19:32.600Z",
  "actions": [
    {
      "executor": "display_emotion",
      "parameters": { "emotion_type": "loving" },
      "priority": 7
    }
  ]
}
```

> ⚠️ **R1 表达约束**：R1 是个方块机器人，正面只有两只大眼睛。给 LLM 写 prompt 时**不要**让 agent 说"我亮起肚皮屏幕"、"我屏幕上显示"等——所有情绪通过眼神表达。详见内部 R1 形态约束。

### 3.2 StarBuddy：多 capability + priority

StarBuddy 是带肚皮屏幕 + 震动 + 音量调节的星星玩具，三个 capability：

```json
{
  "capabilities": [
    {
      "name": "lcd",
      "executor": "lcd",
      "priority": 8,
      "parameters": {
        "type": "object",
        "properties": {
          "animation_id": {
            "type": "string",
            "enum": ["happy_stars", "blush_shy", "sleepy_moon",
                     "alert_surprise", "cheer_up", "heart_love"],
            "description": "Animation to play on belly screen"
          },
          "duration": {
            "type": "integer",
            "description": "Duration in ms",
            "default": 2000,
            "minimum": 500,
            "maximum": 10000
          },
          "loop": { "type": "boolean", "default": false }
        },
        "required": ["animation_id"]
      }
    },
    {
      "name": "vibration",
      "executor": "vibration",
      "priority": 7,
      "parameters": {
        "type": "object",
        "properties": {
          "pattern": {
            "type": "string",
            "enum": ["gentle_shake", "excited_bounce",
                     "nervous_tremble", "sleepy_drift"]
          },
          "intensity": {
            "type": "integer",
            "minimum": 0,
            "maximum": 100,
            "default": 50
          }
        },
        "required": ["pattern"]
      }
    },
    {
      "name": "volume",
      "executor": "volume",
      "priority": 6,
      "parameters": {
        "type": "object",
        "properties": {
          "level": {
            "type": "integer",
            "minimum": 0,
            "maximum": 100,
            "description": "30=whisper, 50=normal, 80=excited, 100=urgent"
          }
        },
        "required": ["level"]
      }
    }
  ]
}
```

**LLM 一次调用可同时触发多个能力**：

```json
{
  "lcd":       { "animation_id": "happy_stars", "duration": 2000 },
  "vibration": { "pattern": "excited_bounce", "intensity": 70 }
}
```

**设备收到的 actions[]**（一个 envelope 含两个 action，按 priority 排序参考）：

```json
{
  "command_id": "cmd_1778231101764",
  "protocol_version": "1.0.0",
  "timestamp": "2026-05-08T09:05:01.764Z",
  "actions": [
    { "executor": "lcd",       "parameters": { "animation_id": "happy_stars", "duration": 2000 }, "priority": 8 },
    { "executor": "vibration", "parameters": { "pattern": "excited_bounce", "intensity": 70 },     "priority": 7 }
  ]
}
```

Agent Preview Chat 里会渲染成：

```
📡 command [cmd_1778231101764] v1.0.0
  • lcd p=8 {"animation_id":"happy_stars","duration":2000}
  • vibration p=7 {"pattern":"excited_bounce","intensity":70}
```

---

## 4. capability_config 字段速查

**只列你（PO）会动的字段**，固件协议契约在 [`firmware-implementation-guide.md` §3](./firmware-implementation-guide.md)。

### 4.1 顶层

```jsonc
{
  "capabilities": [ /* 一个或多个 capability */ ]
}
```

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `capabilities` | array | ✅ | 这款 product 支持的能力清单。空数组等于没暴露任何 device_control 能力。 |

### 4.2 单个 capability

| 字段 | 类型 | 必填 | 含义 / 踩坑提示 |
|---|---|---|---|
| `name` | string | ✅ | LLM 看到的能力名（成为 device_control args 的 key）。**踩坑**：要语义化、面向自然语言；这个不是 executor，跟固件无关。 |
| `executor` | string | ✅ | 固件 dispatch key（设备代码里 `strcmp` 这个字符串）。**踩坑**：设备 case-sensitive，必须跟固件白名单**完全一致**。建议建一个跨团队对齐表。 |
| `priority` | integer | 选填 | 多 action 同时下发时设备执行顺序参考，数字越大越先。**踩坑**：缺省时 wire 上不带这字段，固件按 actions[] 数组顺序处理；如果你的设备依赖 priority，必须显式填。 |
| `description` | string | 选填 | 给 LLM 看的能力描述（写好坏直接影响 LLM 触发率）。**踩坑**：写"一句白话"不要写"参数列表"——参数 LLM 自己会从 schema 看到，description 是告诉它"什么场景下用这个能力"。 |
| `parameters` | object | 选填 | 标准 JSON Schema (draft-2020-12)。**踩坑**：`required` 字段没列对会让 LLM 漏传必填项；`enum` 没列全会导致用户"听起来合理"的请求被 drop（schema 校验失败）。 |

### 4.3 parameters JSON Schema 子集（白名单）

后端用 `networknt/json-schema-validator` 校验 LLM args，并不是所有 schema 关键字都支持。**安全的子集**：

- 类型：`string` / `integer` / `number` / `boolean` / `object` / `array`
- 约束：`enum` / `minimum` / `maximum` / `minLength` / `maxLength`
- 默认：`default`（schema 没传的字段会自动填到 wire 上）
- 嵌套：`properties` / `required` / `items`
- 描述：`description`（透传给 LLM，影响触发率）

**不要用**：`$ref` / `oneOf` / `if-then-else` / 复杂 regex pattern。这些在白名单外，校验行为不保证。

完整白名单见 [`device-control-application.md` §4.1](../../DragonFlow/docs/device-control-application.md)。

---

## 5. description 怎么写（让 LLM 听话）

description 写得好坏，直接决定 LLM 在自然对话里**愿不愿意**触发你的能力。这是 PO 工作里**最容易被低估**的部分。

### 5.1 写法对照：差 vs 好

❌ **差**：写参数清单 / 实现细节

```json
"description": "Set emotion to one of: happy, sad, angry..."
```

✅ **好**：写**使用场景**

```json
"description": "Show an emotion through R1's eyes. Use this to express how R1 is feeling — when reacting to compliments (loving), confusion (curious), tired moments (sleepy), or surprises (surprised). Trigger this in every response that has an emotional tone."
```

LLM 看 enum 自己知道有哪些值；它不知道的是"什么时候该用这个能力"。

### 5.2 enum 命名建议

- **用日常词**：`happy` / `sad` / `loving`，不要 `EMOTION_TYPE_HAPPY`
- **同概念前缀对齐**：要么都加前缀（`emotion_happy` / `emotion_sad`），要么都不加，不要混
- **避免缩写**：`surprised` 不要 `surp`，LLM 会犹豫

### 5.3 priority 数值约定

数字越大越先执行。建议分段：

| 段位 | 用途 |
|---|---|
| 9-10 | 安全 / 中断类（stop / cancel / emergency） |
| 7-8 | 主要表达（emotion / 主动作） |
| 5-6 | 辅助效果（音量、震动伴随） |
| 1-4 | 后台调整 |

**踩坑**：priority 是给设备执行参考的——如果你的设备目前不读这个字段（很多设备直接按 actions[] 顺序执行），priority 配再多也没用。先跟固件确认。

---

## 6. 改了之后什么时候生效

### 6.1 缓存与生效延迟

后端用 Redis 缓存 product 配置，TTL `cache.product.ttl`（默认 **3600s = 1 小时**）。但**改了之后会主动清缓存**：

```
你点 Save Capability
  ↓
后端 ProductService.updateProduct()
  ↓
1. 写库
2. invalidate product cache (productId 这一行)
3. 反查所有引用此 product 的 agent → invalidate 这些 agent 的 cache
  ↓
新 session 立刻拿到新配置
```

所以**新会话立即生效**，不需要等 TTL。

### 6.2 在飞会话不受影响

已经建立的 LLM 会话**不会热更**——它的 tool schema 是会话开始时就固化在 system prompt 里的。要让它生效：

- 重启会话（断开重连 agent / 退出聊天再进）
- 或者等用户自然结束当前对话

这是 LLM 协议层面的限制，不是 bug。

### 6.3 没生效的排查顺序

1. 确认你 Save 成功（toast 没报错）
2. 在 Agent Preview 里**重新 Connect**（不要复用旧 session）
3. Chat 看 📡 command 行有没有变化
4. 还不对：确认你改的是**目标 agent 关联的那个 product**（同名 product 可能有多份）

---

## 7. 常见错误与排查（用 Agent Preview 自查为主）

按出现频率排：

### 7.1 配了但 Preview Chat 里看不到 📡 command 行

LLM 没调 device_control。可能：

- **description 太抽象** → 改成"使用场景"风格（§5.1）
- **enum 没列清楚** → LLM 不确定该传什么值，可能放弃调用
- **system prompt 没引导触发场景** → 在 agent 的 system prompt 加"在每个有情感色彩的回复里调用 device_control"
- **capability 没正确关联 agent** → 回 §2.5 检查 tool 配置里 product_id 对不对
- **LLM 模型不支持 function call** → 换成 GPT-4o / Gemini-2.5-flash / Claude-Sonnet 等支持 tool 的模型

### 7.2 Preview 看到了 📡 command 但设备没动

**云端到设备这段不通，已经是固件侧问题**。把这一行 cmd_id 给固件团队，让他们查串口 log 是否收到。

固件侧排查口诀见 [`firmware-implementation-guide.md` §11](./firmware-implementation-guide.md)。

也可能是：

- **executor 名字跟固件白名单对不上**（设备 case-sensitive，`Display_Emotion` ≠ `display_emotion`）
- **参数缺必填**或值不在固件支持范围（云端 schema 通过但固件白名单更严）

### 7.3 改了配置 Preview 行为没变

- 在飞会话不会热更（§6.2）→ Disconnect + 重新 Connect
- 缓存没清（理论上不会，但万一）→ 等 1 小时 TTL，或重启 agent service

### 7.4 JSON schema 校验失败

后端 `networknt` 校验。常见错误：

| 错误信息 | 含义 |
|---|---|
| `instance value (...) not found in enum` | LLM 传的值不在你配的 enum 里 → 加进 enum 或改 description 引导 |
| `missing required field: xxx` | LLM 没传必填字段 → description 写清楚必填要求 |
| `value is greater than max` | 超出 maximum → 放宽或在 description 提醒范围 |
| `expected type: integer, found: string` | 类型不匹配 → 检查 schema 类型定义，或在 description 强调"传数字不要传字符串" |
| `failed to parse capability_config: invalid JSON` | 你的 JSON 本身有语法错（多/少逗号、引号没闭合）→ 用 JSON formatter 验证 |

### 7.5 多 capability 时只触发了一个

- LLM 没意识到能同时调多个 → 在 description 里举例"可以同时设置 lcd 和 vibration 表达激动情绪"
- 校验失败被 drop → Preview Chat 看后端 log（capability `xxx` rejected: ...）

---

## 8. 联调 Checklist（上线前过一遍）

```
□ capability_config JSON 通过 schema 校验（点 Save 没报错）
□ 每个 executor 字符串跟固件团队**对齐过**（截图存档）
□ 每个 capability 的 enum / 取值范围跟固件团队对齐过
□ 在 Agent Preview 里看到 📡 command 行（云端这一段确认 OK）
□ 拿 Preview 里的 cmd_id 跟固件确认设备实际执行 OK
□ description 至少跑过 5 个 user prompt 验证触发率（Preview Chat 直接看）
  - "你夸夸我" → emotion=loving / happy
  - "我有点害羞" → emotion=loving / surprised
  - "我累了" → emotion=sleepy
  - "为什么会这样" → emotion=curious / thinking
  - 一个负面场景 → emotion=sad / angry
□ priority 在多 action 场景下行为符合预期（如果设备读 priority）
□ 灰度方案确定（enabled flag / 新 product 切流 / agent 版本分流，三选一）
□ 备份了 capability_config JSON 到本地 / 文档（后端不存历史）
```

---

## 9. 进一步阅读

- **协议字段权威定义**（设计决策）→ `~/local/DragonFlow/docs/device-control-application.md`
- **固件怎么解析这条命令**（envelope / wire format / 推荐实现）→ [`./firmware-implementation-guide.md`](./firmware-implementation-guide.md)
- **DragonFlow 内部固件接入指南**（字段速查 / cJSON 模板）→ `~/local/DragonFlow/docs/device-control-firmware-integration-guide.md`
- **LLM 调用 demo**（StarBuddy / Lily 的 system prompt + 触发场景）→ `~/local/DragonFlow/docs/device-control-demo.md`
- **整体架构设计**（跨端定位 / 状态属性 vs 事件动作）→ [`./design.md`](./design.md)
- **P2 表单化编辑器**（即将到来的 capability editor 升级）→ `~/local/DragonFlow/docs/capability-editor-form-design.md`
