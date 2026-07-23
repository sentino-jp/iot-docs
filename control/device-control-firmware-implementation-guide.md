# Sentino 设备控制协议 · 固件开发指南

> Sentino 设备控制协议（`device_control`）的**通用固件接入指南**，与具体硬件 / RTOS / SoC 无关。面向任何要让自己的设备接入 Sentino 云端 LLM agent 的固件团队。
>
> **三段结构**：
> - **Part A 协议契约** —— 你的固件必须正确实现的 wire format 和字段定义
> - **Part B 协议参考实现方案** —— 经过实测验证的工程模板（producer / worker / dispatch / executor）
> - **Part C 首批落地案例** —— 验证原型踩过的坑、与上游 Agora 参考实现的对照、排查口诀
>
> ℹ️ **R1 与 BK7258 是这套协议的首批验证原型**——本文出现的 R1 emotion / BK7258 specific 代码作为具体实例（"长这样的实现是可以跑通的"），不代表协议只为这两款设计。所有协议字段、wire format、行为规则对任何接入方都同样适用。

---

## 0. 这份文档帮你做什么

| 部分 | 内容 | 谁要看 |
|---|---|---|
| **Part A（§1-3）** | 协议契约：必须正确解析的 wire format 和字段定义 | 所有接入方 |
| **Part B（§4-8）** | 协议参考实现方案：经过验证的工程模板 | 要从零写设备端的 |
| **Part C（§9-14）** | 首批落地案例（R1 / BK7258）的踩坑、与上游 Agora 参考实现的对照、排查口诀 | 接手 / 排查问题的 |

> 本文对应 `protocol_version 1.0.0`。协议演进时本文会同步更新。

---

## 准备工作（开始之前）

接入前先确认你的 RTOS / SDK 具备以下能力。本文假设你是有嵌入式 C + RTOS 经验的固件工程师。

### 必需依赖

| 依赖 | 用途 | 推荐 |
|---|---|---|
| **JSON 解析** | parse Content Envelope | cJSON（单文件实现，可移植性好），或同等 JSON 库 |
| **Base64 解码** | DataStream 负载是 base64 | mbedTLS `mbedtls_base64_decode` / 平台自带 / 自实现（≈30 行）。详见 §5.4 |
| **RTC SDK** | 接收 datastream | Agora RTSA SDK（具体版本以你的接入要求为准，需支持 DataStream 通道） |

### RTOS 能力要求

- **消息队列**：producer → worker 解耦
- **独立任务**：worker 跑在独立线程
- **软件定时器**：可选，仅当你需要 idle timer 模式（§6.4）
- **堆外大块内存**：用于 reassembly buffer（推荐用 PSRAM / 外置 RAM；没 PSRAM 的可以用普通堆，注意预算）

### 内存预算（推荐起步值）

| 项 | 大小 | 说明 |
|---|---|---|
| Worker 任务栈 | 8 KB | cJSON 解析占栈较多 |
| 队列消息缓冲 | 4 × 1 KB ≈ 4 KB | `QUEUE_DEPTH=4`，每条最大 1 KB |
| Reassembly buffer | 8 KB | `MAX_ACCUM_LEN`，最多容纳 8 帧 |
| Base64 decode 临时 buffer | ~6 KB | 8 KB base64 → ~6 KB 解码后 |
| cJSON 节点（堆） | ~4 KB | 单条命令解析时占用 |
| **合计** | **~30 KB** | 不含资源加载（AVI / 音频 / 灯效）的额外内存 |

### 资源体积参考
实际需求根据设备控制需求与资源文件估算，以下是 Demo 实现所需资源。
R1 emotion AVI 共 ≈9 MB（10 个文件），不能放 BK7258 的 8 MB flash → 走 SD 卡。**先估算 executor 资源大小再选 flash vs SD 卡**（§6.3）。

---

# Part A：协议（讲清楚要实现什么）

---

## 1. 端到端链路（先有全局观）

```
┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐
│ Sentino    │→ │   LLM tool   │→ │ ConvoAI 透传 │→ │ Agora        │→ │   你的固件    │
│ Product 配 │  │ schema 自动   │  │ + DataStream │  │ DataStream   │  │  解析 + 执行 │
│ capability │  │ 生成给 LLM    │  │ envelope 包装 │ │ binary 广播   │  │             │
└────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  └─────────────┘
     │                │                  │                                   │
   product表        OpenAI tool         actions[]                          executor
   capability_      function call       executor/parameters/                dispatch
   config (JSONB)   arguments           priority
```

**设备端关心的边界**：

- ✅ 收到 datastream binary → 解帧 → base64 解码 → JSON parse → executor dispatch
- ✅ 命令验证（object 判别 / type 判别 / 白名单）
- ✅ 资源缺失兜底
- ❌ 不关心 LLM 怎么决定调 device_control（云端 PO 配置）
- ❌ 不关心 capability schema 校验（云端 ActionEnvelopeBuilder 已经卡过一次）

### 1.1 设备端关注点分离

这条链路上有三件事必须**职责分明**，但**具体怎么分层、放在哪个文件、叫什么名字，由你的项目自己决定**——本文不强加任何代码组织方式。

| 关注点 | 性质 | 为什么要独立 |
|---|---|---|
| **A. RTC 数据接收** | 跑在 RTC SDK 的回调线程，不能阻塞 | 阻塞会拖慢音频路径 |
| **B. 协议解析与命令分发** | 跑在独立 worker 线程，可以慢操作 | cJSON parse + 多帧重组占时间 |
| **C. 执行器落地** | 业务代码，驱动具体外设 / 资源 | 协议解析不应该感知 LCD / 震动器细节 |

**最低约束**：

- A 和 B 之间用**队列**解耦（A 把 raw frame 入队就返回，B 在 worker 里慢慢处理）
- B 和 C 之间用 **callback 注册或函数表**解耦（协议解析层不直接调 `lvgl_app_play()` 这种业务函数）
- C 不应该被塞进 A 或 B 里——跨设备 / 跨项目复用、单元测试都会变难

**至于具体怎么组织代码**：你可能用 4 层（如 BK7258 R1：`RTSA SDK / adapter / sentino_interface / business`，详见 §13），可能用 2 层（小项目把 A+B 合并），可能完全不同的命名。**只要 ABC 三个关注点没被混在同一个函数里就行**。

---

## 2. Transport：Agora RTC DataStream

### 2.1 物理层

ConvoAI 通过 Agora RTC **DataStream** 通道（binary 类型）广播命令到加入同一频道的设备。

- **单帧上限**：≤1KB（RTSA SDK 限制，见 https://doc.shengwang.cn/doc/rtsa/c/advanced-features/data-stream）
- **可靠性**：DataStream 提供顺序和可靠性保证，但**应用层仍要做去重**（用 `command_id`）
- **delivery**：广播给频道内所有 subscriber，单设备绑定靠云端 agent 调度

### 2.2 同一 datastream 混跑多种 object

**关键事实**：同一个 DataStream 不只跑 device_control 命令。云端会把这些都往这条流上发：

| object 值 | 内容 | 设备处理 |
|---|---|---|
| `message.user` | device_control 命令、用户消息 | **如果是命令则解析**（看 §3.3） |
| `message.state` | session 状态变更 | 静默 skip |
| `user.transcription` | 用户语音转写 | 静默 skip |
| `assistant.transcription` | agent 回复文本 | 静默 skip |
| 其他未来 object | TBD | 静默 skip |

**关键原则**：**只对你关心的 object 做处理，其他静默 drop（不要 LOGW）**——这些 object 会高频出现，刷屏会淹没真正的协议错误。

---

## 3. Wire Format

数据按从外到内三层封装：**应用层分片** → **DataStream Envelope** → **Content Envelope**。

### 3.1 应用层分片 envelope（最外层，固件要拼）

由于 1KB 单帧限制，单个命令往往要分 2-3 帧。每帧的格式：

```
<msgid_hex_8>|<frag_idx>|<frag_total>|<base64(json)>
```

字段说明：

| 字段 | 类型 | 含义 |
|---|---|---|
| `msgid_hex_8` | 8 位 hex 字符串 | 同一命令的所有帧共享，用于重组识别 |
| `frag_idx` | 整数（1-based 或 0-based 视实现，但同 msgid 内连续） | 当前帧序号 |
| `frag_total` | 整数 | 这条命令总共多少帧（≥1） |
| `base64(json)` | base64 字符串 | 实际负载，拼完所有帧再 base64 解码就是 JSON |

**抽样观测分布**（BK7258 R1 一次抽样，仅供量级参考——实际占比受 capability schema 大小、参数复杂度影响很大）：

- 单帧命令（`total==1`）：emotion-only 等小命令
- 双帧命令（`total==2`）：参数较多 / 多 capability 一次下发
- ≥3 帧：罕见但**协议允许**，必须支持

**实抓 wire dump 例子**（base64 略）：

```
7a5babea|1|2|eyJtZXNzYWdlX2lkIjoiMzc2NjYzNzgiLCJvYmplY3QiOiJtZXNzYWdlLn...
7a5babea|2|2|aWQiOiJjbWRfMTc3ODI0MjIxODA1NiIsInByb3RvY29sX3ZlcnNpb24iOi...
```

拼完 base64 解码后得到一段 JSON，就是 §3.2 的 DataStream Envelope。

### 3.2 DataStream Envelope（base64 解码后的顶层 JSON）

```jsonc
{
  "message_id": "37666378",          // ConvoAI 生成，可用于去重（与 command_id 不同）
  "object":     "message.user",       // 必须 == "message.user" 才进设备控制分支
  "content":    { /* 见 §3.3 */ },    // 实际命令体（object 类型）
  "data_type":  "message"             // ConvoAI 字段，固件可忽略
}
```

固件验证步骤：

1. `object == "message.user"` → 是发布消息（device_control / 媒体类）
2. `content` 字段类型是 **JSON object**（不是 string）→ Generic 协议分支
3. 进 §3.3 看 `content.type` 判别

### 3.3 Content Envelope（命令体）

```jsonc
{
  "type":             "command",                    // 判别字段，固定 "command"
  "command_id":       "cmd_1778242218056",          // 唯一 ID（ms 时间戳），用于去重 / ack
  "protocol_version": "1.0.0",                      // 协议版本，当前锁 1.0.0
  "timestamp":        "2026-05-08T09:15:20.600Z",   // ISO 8601 UTC，命令产生时间
  "actions": [                                      // 一个或多个动作，按数组顺序
    {
      "executor":   "display_emotion",              // ← 固件 dispatch 用这个
      "parameters": { "emotion_type": "happy" },    // ← capability 自定义参数
      "priority":   7                               // 可选，缺省时不带
    }
  ]
}
```

**判别字段 `content.type`**：当前只有 `"command"`，**未来可能新增** `"image"` / `"video"` / `"audio"` / `"ui_event"` 等。**未知 type 静默忽略**，不要 panic。设计决策见 [`device-control-application.md` §13.25](../../DragonFlow/docs/device-control-application.md)。

### 3.4 actions[] 字段表

| 字段 | 类型 | 必有 | 固件用途 |
|---|---|---|---|
| `executor` | string | ✅ | dispatch key，对照你的固件白名单。**case-sensitive**。 |
| `parameters` | object | 视 capability | executor-specific 参数，由云端 capability schema 定义 |
| `priority` | integer | 可选 | 数字越大越先执行；**缺省时按 actions 数组顺序** |

### 3.5 真实样例

**R1 单 action（emotion）**：

```json
{
  "message_id": "37666378",
  "object": "message.user",
  "content": {
    "type": "command",
    "command_id": "cmd_1778242218056",
    "protocol_version": "1.0.0",
    "timestamp": "2026-05-08T09:15:20.600Z",
    "actions": [
      {
        "executor": "display_emotion",
        "parameters": { "emotion_type": "happy" },
        "priority": 7
      }
    ]
  },
  "data_type": "message"
}
```

**StarBuddy 多 action**：

```json
{
  "object": "message.user",
  "content": {
    "type": "command",
    "command_id": "cmd_1778231101764",
    "protocol_version": "1.0.0",
    "timestamp": "2026-05-08T09:05:01.764Z",
    "actions": [
      { "executor": "lcd",       "parameters": { "animation_id": "happy_stars", "duration": 2000 }, "priority": 8 },
      { "executor": "vibration", "parameters": { "pattern": "excited_bounce", "intensity": 70 },     "priority": 7 }
    ]
  },
  "data_type": "message"
}
```

固件应同时驱动 LCD 与震动器，按 priority 高到低排或并发触发（依设备实现）。

### 3.6 协议演进规则（向前兼容）

- **加字段**：加性变更，老固件忽略未知字段即可（**don't fail on unknown keys**）
- **新增 content.type**：固件应**只识别自己关心的 type**（当前只有 `"command"`），未知 type 静默 skip
- **新增 executor**：在你的 dispatch 表加分支即可，不需要协调云端发版
- **删字段 / 改语义**：会升 `protocol_version`，可能触发协议演进流程
- **当前固定** `protocol_version="1.0.0"`，本文按此版本

---

# Part B：推荐实现方案（怎么实现）

---

## 4. 整体架构推荐

### 4.1 producer / consumer 流水线

落实 §1.1 的 ABC 关注点分离。下图以 BK7258 R1 项目的命名为示例（`adapter` / `sentino_interface` / `business` 是该项目的层名，**你的项目可以叫任何名字**——只要保留 producer→queue→consumer→executor 的职责拆分即可）：

```
RTC SDK 回调 (1KB binary frame)
     ↓
[A] producer：alloc + push to queue          ← 跑在 RTC 回调线程，不能阻塞
     ↓ (queue)
[B] worker thread
     │  ├─ 解帧（按 | split）
     │  ├─ 多帧重组（accumulator）
     │  ├─ base64 decode
     │  ├─ cJSON parse
     │  ├─ object / type 双重 gate
     │  └─ 按 executor 名 dispatch
     ↓
[C] executor callback                         ← 业务代码，驱动 LCD / 震动 / ...
```

**关键约束**：

- producer 在 RTC 回调里**只做 alloc + push**，cJSON parse 一律下沉到 worker
- worker 线程独立，可以慢操作
- executor 通过 callback / 函数表注册，协议解析层无感知具体业务
- 队列消息用堆 / psram 分配，**不要在栈上**——分片重组要跨 worker iteration

### 4.2 不要做的事

- ❌ **不要把 executor 直接塞 RTC 回调或 adapter 代码里**——破坏关注点分离，复用与单元测试都受影响
- ❌ **不要在 RTC 回调里 cJSON parse**——会阻塞 RTC 主线程
- ❌ **不要 alloc 在栈上**——分片重组要跨 worker iteration

---

## 5. Worker 实现模板

> ⚠️ **示例 API 为 BK7258 / Beken SDK 风格**
>
> 本章所有代码示例使用 Beken SDK 的 API（`psram_alloc` / `beken_queue_t` / `rtos_init_oneshot_timer` / `BK_LOGW` / `kNoErr` 等）。**请按你的 RTOS 自行替换**——命名是表达职责，逻辑通用，别让 API 差异挡了你抄代码。常见对照：
>
> | 角色 | Beken SDK | FreeRTOS / ESP-IDF | RT-Thread |
> |---|---|---|---|
> | 大块内存 alloc/free | `psram_alloc` / `psram_free` | `heap_caps_malloc(MALLOC_CAP_SPIRAM)` / `heap_caps_free` | `rt_malloc_align` / `rt_free_align` |
> | 消息队列句柄 | `beken_queue_t` | `QueueHandle_t` | `rt_mq_t` |
> | 入队 / 出队 | `rtos_push_to_queue` / `rtos_pop_from_queue` | `xQueueSend` / `xQueueReceive` | `rt_mq_send` / `rt_mq_recv` |
> | 任务句柄 / 创建 | `beken_thread_t` / `rtos_create_thread` | `TaskHandle_t` / `xTaskCreate` | `rt_thread_t` / `rt_thread_create` |
> | 软件定时器 | `beken2_timer_t` / `rtos_init_oneshot_timer` | `TimerHandle_t` / `xTimerCreate` | `rt_timer_t` / `rt_timer_create` |
> | 日志（warn 级） | `BK_LOGW` | `ESP_LOGW` / `LOG_W` | `LOG_W` |
> | 错误码（成功 / 等待） | `kNoErr` / `BEKEN_NO_WAIT` | `pdPASS` / `0` | `RT_EOK` / `RT_WAITING_NO` |

### 5.1 队列 + worker thread 骨架

```c
/* queue：psram 中的 raw frame buffer */
typedef struct {
    uint8_t *data;
    size_t   len;
} datastream_msg_t;

static beken_queue_t  s_datastream_queue;
static beken_thread_t s_worker_handle;

#define QUEUE_DEPTH 4   /* RTC busy 时不够，靠 LOGW 暴露后再调 */

/* producer 在 adapter 层（agora_rtc.c 风格） */
static void on_datastream_message(const uint8_t *bytes, size_t len)
{
    datastream_msg_t msg = { 0 };
    msg.data = psram_alloc(len);
    if (!msg.data) { LOGW("datastream alloc fail, len=%zu", len); return; }
    memcpy(msg.data, bytes, len);
    msg.len = len;

    if (rtos_push_to_queue(&s_datastream_queue, &msg, BEKEN_NO_WAIT) != kNoErr) {
        LOGW("queue full, drop msg len=%zu", len);
        psram_free(msg.data);   /* ★ 关键：入队失败必须 free，否则 leak */
    }
}

/* consumer 在 sentino_interface 层 */
static void worker_thread(void *arg)
{
    LOGW("conv_ai worker started");   /* ★ LOGW，release build 不被 strip */
    datastream_msg_t msg;
    while (1) {
        if (rtos_pop_from_queue(&s_datastream_queue, &msg, BEKEN_WAIT_FOREVER) == kNoErr) {
            parse_and_dispatch(msg.data, msg.len);
            psram_free(msg.data);
        }
    }
}
```

**队列深度推荐**：4 起步，靠 `queue full` 的 LOGW 出现频率调整。RTC 高负载场景需要更深。

### 5.2 解帧

```c
/* 入参：单帧 raw bytes（最多 1KB）
   格式：<msgid_hex_8>|<frag_idx>|<frag_total>|<base64> */
static int deframe(const char *frame, size_t len,
                   char msgid[9], int *idx, int *total, const char **b64_start)
{
    /* 找前 3 个 '|' 分隔符 */
    const char *p1 = memchr(frame, '|', len);
    if (!p1) return -1;
    const char *p2 = memchr(p1 + 1, '|', len - (p1 - frame) - 1);
    if (!p2) return -1;
    const char *p3 = memchr(p2 + 1, '|', len - (p2 - frame) - 1);
    if (!p3) return -1;

    /* msgid: 8 hex */
    size_t msgid_len = p1 - frame;
    if (msgid_len != 8) return -1;
    memcpy(msgid, frame, 8);
    msgid[8] = '\0';

    *idx   = atoi(p1 + 1);
    *total = atoi(p2 + 1);
    if (*idx < 1 || *total < 1 || *idx > *total) return -1;

    *b64_start = p3 + 1;
    return 0;
}
```

**推荐**：用静态 buffer，不要每帧 malloc（worker 单线程，无并发）。

### 5.3 多帧重组（关键）

```c
#define MAX_FRAG_TOTAL  8
#define MAX_ACCUM_LEN   (8 * 1024)

static char    s_accum_msgid[9];
static int     s_accum_total;        /* 0 表示 idle */
static int     s_accum_received;
static char   *s_accum_buf;          /* psram 拼 base64 */
static size_t  s_accum_len;

/** 一刀切：清掉所有累积状态，进入 idle */
static void accum_reset(void)
{
    if (s_accum_buf) { psram_free(s_accum_buf); s_accum_buf = NULL; }
    s_accum_msgid[0] = '\0';
    s_accum_total = 0;
    s_accum_received = 0;
    s_accum_len = 0;
}

static void parse_and_dispatch(const uint8_t *bytes, size_t len)
{
    char msgid[9];
    int idx, total;
    const char *b64;
    if (deframe((const char *)bytes, len, msgid, &idx, &total, &b64) != 0) {
        LOGW("deframe fail");
        return;
    }

    /* fast path：单帧 */
    if (total == 1) {
        decode_and_dispatch(b64, len - (b64 - (const char *)bytes));
        return;
    }

    /* 防 OOM */
    if (total > MAX_FRAG_TOTAL) {
        LOGW("frag total %d > MAX, drop msgid=%s", total, msgid);
        accum_reset();
        return;
    }

    /* 状态切换：msgid 变了 / 第一帧到了 / total 不一致 → 全清 */
    if (s_accum_total == 0 ||
        strcmp(s_accum_msgid, msgid) != 0 ||
        s_accum_total != total ||
        idx == 1) {
        accum_reset();
        strcpy(s_accum_msgid, msgid);
        s_accum_total = total;
        s_accum_buf = psram_alloc(MAX_ACCUM_LEN);
        s_accum_len = 0;
    }

    /* append 这一帧的 base64 内容 */
    size_t b64_len = len - (b64 - (const char *)bytes);
    if (s_accum_len + b64_len > MAX_ACCUM_LEN) {
        LOGW("accum overflow, drop msgid=%s", msgid);
        accum_reset();
        return;
    }
    memcpy(s_accum_buf + s_accum_len, b64, b64_len);
    s_accum_len += b64_len;
    s_accum_received++;

    if (s_accum_received >= s_accum_total) {
        LOGI("reassembled msgid=%s frags=%d b64_len=%zu", msgid, total, s_accum_len);
        decode_and_dispatch(s_accum_buf, s_accum_len);
        accum_reset();
    }
}
```

**`accum_reset()` 一刀切**是关键设计——msgid 跳变、新 idx=1、total 不一致都清零。这跟某些上游参考实现的状态清理时机有差异，详见 §10。

### 5.4 Base64 解码

多帧重组完成后，`s_accum_buf` 里是拼好的 base64 字符串，需要 decode 成原始 JSON 字节流再喂给 cJSON。

#### 库选型

| 选项 | 优 | 劣 | 推荐场景 |
|---|---|---|---|
| **mbedTLS** `mbedtls_base64_decode` | 大多数 RTOS 都有，API 稳定 | 链入 mbedTLS 体积稍大 | 项目里已经用 mbedTLS（TLS / 加密 / hash） |
| **平台自带** | 开箱即用，零依赖 | 平台绑定（ESP-IDF `mbedtls_base64`、HarmonyOS `osal_base64` 等） | 只跑单一平台 |
| **自实现** | 单文件，30 行左右 | 自己维护边界处理 | 想保持依赖最小、已有公司内部库 |

BK7258 R1 用了 mbedTLS。

#### 输出 buffer 大小

base64 解码后的字节数 = `b64_len × 3 / 4 - padding_bytes`。预留 buffer 简单按 `b64_len × 3 / 4 + 4` 即可（多 4 字节防 padding 边界 + NUL 终结）。

#### 实现示例（mbedTLS）

```c
#include "mbedtls/base64.h"

static int decode_and_dispatch(const char *b64, size_t b64_len)
{
    /* 输出 buffer 大小估算：base64 → raw 比例 4:3，再 +4 防 padding + NUL */
    size_t out_cap = (b64_len / 4) * 3 + 4;
    uint8_t *out = psram_alloc(out_cap);
    if (!out) {
        LOGW("base64 alloc fail, len=%zu", b64_len);
        return -1;
    }

    size_t out_len = 0;
    int ret = mbedtls_base64_decode(out, out_cap, &out_len,
                                    (const unsigned char *)b64, b64_len);
    if (ret != 0) {
        LOGW("base64 decode fail, ret=%d, len=%zu", ret, b64_len);
        psram_free(out);
        return -1;
    }

    out[out_len] = '\0';   /* NUL 终结，cJSON_Parse 期望 C 字符串 */
    parse_json_envelope((const char *)out, out_len);

    psram_free(out);
    return 0;
}
```

#### 几个坑

- **必须等所有帧重组完整再 decode**：base64 是 4 字节对齐的，半个 group 解不出来。**不要按帧增量解码**——§5.3 的重组就是为了凑齐再一次性解。
- **NUL 终结**：cJSON_Parse 要 NUL 终结的 C 字符串，buffer 必须多分配 1 字节。
- **base64 padding（`=`）**：合法 base64 末尾可能有 0-2 个 `=`，mbedTLS 都能处理；自实现要注意。
- **失败必须暴露**：base64 decode 失败通常意味着拼包错位 / 单帧不是合法 base64 / 传输位错误，必须 LOGW + drop，不要 silent fail。
- **大块内存释放**：每条命令解码完立刻 free，不要常驻——一次命令最大 ~6 KB，常驻会占 PSRAM。

### 5.5 过滤决策表（必须实现）

base64 decode + cJSON parse 之后：

| 条件 | 处理 | 原因 |
|---|---|---|
| 帧解析失败（缺 `\|` / idx 越界） | LOGW + drop | 协议错位，需要暴露 |
| base64 decode 失败 | LOGW + drop | 同上 |
| cJSON parse 失败 | LOGW + drop | 同上 |
| `object != "message.user"` | **静默 skip** | 同 channel 跑大量 transcription / state |
| `content` 不是 object | **静默 skip** | 旧 Lily 协议，新设备不用管 |
| `content.type != "command"` | **静默 skip** | 未来 image / video 等 |
| `content.actions` 不是数组 | LOGW + drop | command 必须有 actions |
| action 缺 `executor` 字段 | LOGW + skip 该 action | schema 错 |
| executor 不在你的白名单 | LOGW + skip 该 action | 云端配错 / 未来新 capability |
| 总帧数 > MAX_FRAG_TOTAL | LOGW + reset | 防 OOM |

**关键原则**：**高频帧静默，协议错位暴露**。打错日志级别会淹没真正的问题。

### 5.6 日志级别策略

`BK_LOGI` 在 release build 会被编译器 **strip**（很多 RTOS 都这样）。**只有 LOGW / LOGE 进 binary**。

提升到 LOGW（关键诊断、低频）：

- `worker started`（开机一次性，自检价值大）
- `cmd <command_id>, N action(s)`（命令到达）
- `action: executor=... priority=... params=...`（dispatch 证据）

留 LOGI（高频或重复）：

- 重组完成 `reassembled msgid=...`（每多帧命令一次，会刷屏）
- 单帧 fast path decode

---

## 6. Executor 实现模式

### 6.1 注册表 vs strcmp 链

| 执行器数量 | 推荐 |
|---|---|
| <3 类 | 单 callback 内 `if-else` strcmp 链 |
| ≥3 类 | `name → cb` 注册表（hash 或线性查找都行，固件量小不必上 hash） |

### 6.2 参数白名单（必须做）

**禁止**直接把 LLM 字符串透传给底层 SDK。举个坑：

```c
/* ❌ 危险 */
void on_emotion(const char *emotion_type) {
    char path[64];
    snprintf(path, sizeof(path), "/%s.avi", emotion_type);
    lvgl_app_play(path);   /* 资源加载 API 通常不做存在性校验，文件缺失会导致后续 NULL deref → MemFault */
}
```

云端配置今天有 9 种 emotion，明天 PO 可能加第 10 种但 SD 卡没刷文件。LLM 字符串透传 → 设备重启。

**正确做法**：

```c
typedef struct { const char *name; emotion_id_t id; } emotion_map_t;
static const emotion_map_t s_emotion_map[] = {
    { "happy",     EMOTION_HAPPY },
    { "sad",       EMOTION_SAD },
    { "angry",     EMOTION_ANGRY },
    { "surprised", EMOTION_SURPRISED },
    { "neutral",   EMOTION_NEUTRAL },
    { "thinking",  EMOTION_THINKING },
    { "sleepy",    EMOTION_SLEEPY },
    { "loving",    EMOTION_LOVING },
    { "curious",   EMOTION_CURIOUS },
};

void on_emotion(const char *emotion_type) {
    for (int i = 0; i < sizeof(s_emotion_map)/sizeof(s_emotion_map[0]); i++) {
        if (strcmp(emotion_type, s_emotion_map[i].name) == 0) {
            play_emotion(s_emotion_map[i].id);
            return;
        }
    }
    LOGW("unknown emotion: %s, ignore", emotion_type);   /* 不进 SDK，不 panic */
}
```

### 6.3 资源缺失兜底（双保险）

**双保险原则**：

1. **白名单**（代码层）：拦云端发的新词，未列入直接 LOGW return
2. **资源齐全**（部署层）：确保 SD 卡 / flash 文件齐全

任一保险失效都可能挂板。**新增 executor / capability 上线前必查这两处**。

#### SD 卡 vs flash 取舍

| 选项 | 优 | 劣 | 适用 |
|---|---|---|---|
| flash | 不依赖外设、随固件出厂 | 容量小（BK7258 8MB 已饱和），加资源要重新规划 partition | 资源 ≤ 几 MB 的设备 |
| SD 卡 | 容量大、可独立更新 | 用户可能不插卡 / 损坏，得有兜底提示 | 资源 ≥ 10MB 或迭代频繁 |

BK7258 R1 走 SD 卡（emotion AVI ≈ 9MB），具体部署见 §12。

### 6.4 Idle timer 模式（持续状态类 executor）

如果你的 executor 是「持续状态」类（emotion 播放、灯光长亮），需要回 default 机制，否则一直停在最后一个状态。

#### 关键约束：timer 回调不能直接调"重活"

RTOS 软件 timer 的回调跑在 **timer-service 线程**。这个线程：

- 通常**栈很小**（几 KB），不能跑大开销操作
- 在很多 RTOS 上**不允许 block 在跨核 mailbox / 跨任务同步原语**上
- 一旦阻塞，整个 timer service 都卡住，影响其他 oneshot/periodic timer

所以 timer 回调里**不能直接**调 `lvgl_app_play()` / 跨核同步 API / 任何会 block 在 mailbox 上的函数。**正确做法**：timer 回调只做一件事——**post 一个事件给业务 worker 线程**，让 worker 去执行实际操作。

#### 错误示范（会挂）

```c
/* ❌ 不要这样写 */
static void idle_timer_cb(void *arg) {
    lvgl_app_play(CONV_AI_IDLE_DEFAULT_AVI);   /* 跨核 mailbox sync,timer 线程跑不了 */
}
```

BK7258 上观测到的现场（serial log）：

```
GENIE      :W: emotion idle timeout, restoring /genie_eye.avi
cpu1       :    media_major_mailbox_mailbox_rx_isr 368 ack flag error 97 ...
media_ap   :W: media_app_mailbox_send_msg_to_media_major_mailbox 260 send request FAILED 50004
media_ap   :E: msg_send_req_to_media_app_mailbox_sync failed 0xffffffff
media_ap   :E: media_send_msg_sync failed 0xffffffff
media_ap   :W: media_app_lvgl_send_data complete ffffffff
```

跨核 mailbox 在 timer-service 上下文里发不成 → 资源切不回 default。

#### 正确做法：post 到 worker 线程

```c
#define CONV_AI_IDLE_MS              8000
#define CONV_AI_IDLE_DEFAULT_AVI     "/genie_eye.avi"

static beken2_timer_t s_idle_timer;
static bool s_idle_timer_inited;

/* timer 回调只 post event,不做实际工作 */
static void idle_timer_cb(void *larg, void *rarg) {
    (void)larg; (void)rarg;
    BK_LOGW(TAG, "emotion idle timeout, restoring %s\n", CONV_AI_IDLE_DEFAULT_AVI);
    /* 跨核 mailbox sync 不能在 timer 线程跑,转发给业务 worker */
    app_event_send_msg(APP_EVT_CONVOAI_RESTORE_IDLE_AVI, 0);
}

/* 业务 worker 线程里处理事件,这里可以安全调跨核 API */
static void on_app_event(app_event_t evt) {
    switch (evt) {
    case APP_EVT_CONVOAI_RESTORE_IDLE_AVI:
        if (!image_recognition_mode_enable) {     /* 别的模式占用时跳过 */
            lvgl_app_play("/genie_eye.avi");
        }
        break;
    /* ... */
    }
}

/** 每次新命令派发都调一次:stop 旧 timer + start 新的 */
static void arm_idle_timer(void) {
    if (!s_idle_timer_inited) {
        rtos_init_oneshot_timer(&s_idle_timer, CONV_AI_IDLE_MS, idle_timer_cb, NULL, NULL);
        s_idle_timer_inited = true;
    }
    rtos_stop_oneshot_timer(&s_idle_timer);
    rtos_start_oneshot_timer(&s_idle_timer);
}
```

每次 emotion 派发都 `arm_idle_timer()`。8s 没新命令 → timer 触发 → post event → worker 线程切回 default。新命令立即打断 + 重置。

> **通用原则**：**timer 回调 = post 事件，不做实事**。在绝大多数 RTOS 上，timer 线程栈小、不允许长时间阻塞或跨核同步，需要走跨核 mailbox / 长持锁 / 大栈占用的操作都应该 post 到业务 worker 线程执行。具体限制以你 RTOS 的 timer 服务文档为准。

**不是所有 executor 都需要 idle timer**：瞬时动作（震动 200ms、播一段音）不需要，自然结束就好。**按你的能力语义决定**。

---

## 7. Producer 侧细节（防 leak）

producer 通常在 RTSA 回调（`agora_rtc.c` 之类）：

```c
static void on_stream_message(...) {
    msg.data = psram_alloc(len);
    if (!msg.data) {
        LOGW("alloc fail, drop msg len=%zu", len);   /* ★ 暴露 */
        return;
    }
    memcpy(msg.data, ..., len);
    msg.len = len;
    if (push_to_queue(&q, &msg, NO_WAIT) != kNoErr) {
        LOGW("queue full, drop msg len=%zu", len);
        psram_free(msg.data);   /* ★ 必须 free，否则每次满都 leak */
    }
}
```

**不要相信"不会失败"**——RTC busy 时 queue 会满，psram 紧张时 alloc 会失败。**两条路径都要补 LOGW + free**。

---

## 8. 上行 ack（可选）

每个命令都有 `command_id`，理论上设备可以回 ack 让云端知道命令是否落地。当前协议**未要求**固件实现 ack 上行——后续协议如有命令追溯需求，会通过协议演进流程统一对齐字段语义和上行通道，届时再补充本节。

**建议**：当前阶段**不做**。固件保留对 `command_id` 的解析能力即可，未来启用 ack 时无需重构。

---

# Part C：首批落地案例（R1 / BK7258）

> 本节内容来自协议**首批验证原型**（R1 + BK7258 + Agora RTSA SDK）的落地实战。具体的 SoC / RTOS / 外设 API 不一定跟你的项目相同，但**踩到的坑分类**和**排查思路**对所有接入方都有参考价值。看的时候把"BK7258"和"R1"当作一个具体例子，不要当成协议要求。

---

## 9. 踩坑清单（首批落地实战）

按发生顺序。每条都是协议设计或固件实现某处的真实暴露点，对所有接入方都值得一看：

| # | 现象 | 根因 | 修法 |
|---|---|---|---|
| 1 | 串口 `json parse fail` 6/6 | wire 不是 plan 假设的顶层 JSON | 先 LOGW 把 prefix 打出来对照，认出 `<id>\|<idx>\|<total>\|<b64>` envelope（§3.1） |
| 2 | 单帧能解多帧 drop | `total>1` 当时认为不会出现，抽样观测中相当一部分命令是多帧 | 加 reassembly buffer + state（§5.3） |
| 3 | release build 看不到 `worker started` | LOGI 被编译器 strip | 关键诊断字符串提升 LOGW（§5.6） |
| 4 | producer alloc 后入队失败漏内存 | 老代码不 free | 入队失败 `psram_free(msg.data)` + LOGW 暴露（§7） |
| 5 | LCD 播 `/happy.avi` 直接 MemFault 重启 | flash 没烧资源 + 资源加载 API 不做存在性校验 | SD 卡放资源 + emotion 白名单（§6.2 / §6.3 / §12） |
| 6 | 命令一发 emotion 就一直循环播 | 没有 idle 回 default 的机制 | 8s idle timer 回 `/genie_eye.avi`（§6.4） |
| 7 | `content.type` 字段云端可能没下发 | 协议决策时间晚于初版 | 跨团队 issue 跟云端对齐，固件保持 gate |
| 8 | idle timer 触发后 LCD 没切回 default，serial 出 `media_major_mailbox ack flag error 97` / `media_send_msg_sync failed` | timer 回调直接调 `lvgl_app_play()`，跨核 mailbox sync 在 timer-service 线程上下文跑不了 | timer 回调改成 `app_event_send_msg(APP_EVT_CONVOAI_RESTORE_IDLE_AVI, 0)`，由 worker 线程实际调 lvgl（§6.4） |

---

## 10. 与 Agora 上游 R1 参考实现的对照

> 仅适用于**基于 Agora 上游 R1 参考代码做起点**的项目。如果你的固件不来自这条参考链，跳过本节即可。

Agora 上游提供了一份 R1 设备的 device-control 参考实现。Sentino 协议的接入与之有如下 4 处关键差异——抄上游代码作为起点的团队**注意以下偏离点**：

| 项 | 上游 R1 参考 | 推荐 |
|---|---|---|
| reassembly 状态清理 | 状态清理时机偏少（msgid 跳变、新 idx=1 等切换路径未必清） | `accum_reset()` 一刀切，所有切换路径都清 |
| producer 失败路径 | alloc 后入队失败的资源释放不齐 | alloc 失败和入队失败都补 free + LOGW |
| transcripts / state 帧处理 | 全部 LOGW，日志会刷屏 | 静默 skip（object 不匹配就 drop，不打日志） |
| schema 假设 | `{action_type, emotion_type}` 扁平结构 | `{type:"command", command_id, actions[]}` Sentino 标准（§3.3） |

---

## 11. 排查口诀（出问题时按这个顺序查）

**推荐先看云端 Agent Preview Chat**：浏览器打开 agent preview，Chat 面板看有没有：

```
📡 command [cmd_xxx] v1.0.0
  • <executor> p=<priority> {<params>}
```

- **没有这一行**：问题在 PO 配置侧或 LLM 没触发，跟固件无关。推 PO 看 [`product-capability-guide.md` §7](./product-capability-guide.md)。
- **有这一行**：拿 `cmd_xxx` 这个 ID 来设备侧串口对照。

然后按链路顺序：

| 现象 | 根因方向 | 处理 |
|---|---|---|
| 串口完全没 cmd 行 | datastream 没到这台设备 | 检查 agent ↔ device 绑定（云端配置） |
| 有 cmd 行但没 `action: executor=` | actions[] 解析失败 | dump 整段 payload，对照 §3.3 检查 schema |
| 有 `action:` 但设备没动 | executor 名字 / 参数白名单 / 资源缺失 | §6.2 检查白名单，§12 检查资源 |
| 板子重启 | LCD MemFault 类，资源/白名单兜底失效 | §6.3 双保险 |
| `json parse fail` | deframe 失败 | dump 帧 prefix，对照 `<id>\|<idx>\|<total>\|<b64>` envelope（§3.1） |

---

## 12. 资源部署案例（BK7258 + R1 emotion AVI / SD 卡）

> 这一节是**首批落地的具体部署方案**，作为「资源类 executor 应该怎么管」的参照。你的设备如果走不同 SoC / 不同存储介质，原则照搬，路径自行替换。

### 12.1 文件清单与映射

R1 emotion 在 BK7258 上：

| LLM 传的 `emotion_type` | enum | 文件名 |
|---|---|---|
| -（idle 默认） | - | `genie_eye.avi` |
| `happy` | `EMOTION_HAPPY` | `happy.avi` |
| `sad` | `EMOTION_SAD` | `sad.avi` |
| `angry` | `EMOTION_ANGRY` | `angry.avi` |
| `surprised` | `EMOTION_SURPRISED` | `surprise.avi` ⚠️ 缺 `d` |
| `neutral` | `EMOTION_NEUTRAL` | `neutral.avi` |
| `thinking` | `EMOTION_THINKING` | `thinking.avi` |
| `sleepy` | `EMOTION_SLEEPY` | `sleepy.avi` |
| `loving` | `EMOTION_LOVING` | `love.avi` ⚠️ 不是 `loving.avi` |
| `curious` | `EMOTION_CURIOUS` | `curious.avi` |

> **加新 emotion 必须三处同步**：云端 product schema enum / 固件 enum + 文件名映射 / SD 卡放对应 `.avi`。三处少一处都会挂。

### 12.2 拷贝命令（macOS）

```bash
# 1. 找到 SD 卡挂载点（FAT32 标签常是 NO NAME）
diskutil list | grep -iE "fat|exfat"
SD="/Volumes/NO NAME"     # 按实际改

# 2. 拷 emotion 资源（10 个）
cp /path/to/resource/{genie_eye,happy,sad,angry,surprise,neutral,thinking,sleepy,love,curious}.avi \
   "$SD/"

# 3. 验证
ls "$SD"/*.avi    # 应该 10 个

# 4. 卸载（必须，直接拔卡会损坏 FAT 表）
diskutil eject "$SD"
```

### 12.3 兜底（双保险）

文件缺失 / SD 卡没插 → 资源加载链路 NULL deref → MemFault 重启。**双保险**：

1. **emotion 白名单**（§6.2）—— 拦云端发的新词，未列入直接 LOGW return，**根本不进 lvgl_app_play**
2. **SD 卡文件齐全** —— §12.2 一次拷完

任一保险失效都可能挂板。**新增 emotion 上线前同时改这两处**。

### 12.4 如果你的设备走 flash 资源

- partition 表预留够（参考 BK7258 8MB 已饱和的教训，新设备规划 ≥16MB）
- 加资源要走固件 OTA，比 SD 卡更新慢
- 优势：不依赖外设，量产返修少
- 推荐：**emotion 这种小尺寸资源 ≤ 2MB 时走 flash，> 5MB 走 SD 卡**

---

## 13. 参考实现索引（BK7258 / R1 首批落地）

> 这些是首批验证原型的具体代码位置，留作 cite 参照。**不是协议要求**——你的项目用任何代码组织都行。

### 13.1 BK7258 / R1 具体文件 / 行号

| 模块 | 文件 |
|---|---|
| Worker / 解帧 / 重组 / dispatch | `device/projects/common_components/sentino_interface/sentino_conv_ai_command.c` |
| Worker API | `sentino_conv_ai_command.h` |
| Producer leak fix | `device/projects/common_components/network_transfer/agora_rtc/agora_rtc.c:283` |
| Init 调用点 | `device/projects/beken_genie/main/app_main.c`（在 `bk_sconf_init_datastream_resource()` 之后） |
| Emotion 派发 + idle timer | `app_main.c: conv_ai_executor_dispatch / arm_idle_timer / idle_timer_cb` |
| CPU1 落点 | `app_event.c:451-457 change_lgvl_avi_resouce → lvgl_app_play(app_emotion_2_avi_file(value))` |
| Emotion 枚举 | `app_event.h:5-14 EMOTION_HAPPY/.../CURIOUS` |

### 13.2 接入设备应有的实现要素清单

无关具体代码组织，任何接入 Sentino 设备控制协议的固件**应当具备以下要素**：

- [ ] **Producer**（关注点 A）：alloc + 入队，不阻塞 RTC 线程；alloc 失败 / 入队失败都要 free + LOGW
- [ ] **Worker thread**（关注点 B）：循环 pop queue → 解帧 → 重组 → base64 decode → JSON parse → dispatch
- [ ] **解帧函数**：按 `|` split，校验 `idx` / `total` 范围
- [ ] **重组状态机** + 一刀切 reset：`MAX_FRAG_TOTAL` / `MAX_ACCUM_LEN` 防 OOM（§5.3）
- [ ] **过滤 gate**：`object` / `content.type` 双重判别 + 高频帧静默（§5.5）
- [ ] **Executor 注册 / dispatch**：name → callback 表（§6.1）
- [ ] **参数白名单**：每个 executor 自己的 enum mapping（§6.2）
- [ ] **Idle timer**（持续状态类 executor 必备，§6.4）
- [ ] **资源部署 SOP**：双保险——白名单 + 文件齐全（§6.3 / §12.3）
- [ ] **关键诊断 LOGW**：worker started / cmd 到达 / action 派发（§5.6）

---

## 14. 参考

- **云端协议设计文档**（决策记录）：`~/local/DragonFlow/docs/device-control-application.md`
  - §3 框架/业务层划分 · §4 Capability Schema · §7.5 Content Envelope 契约 · §13.25 判别字段决策
- **云端字段速查 + cJSON 模板**：`~/local/DragonFlow/docs/device-control-firmware-integration-guide.md`
- **PO 配置指南**：[`./product-capability-guide.md`](./product-capability-guide.md)
- **整体架构设计**：[`./design.md`](./design.md)
- **LLM 调用 demo（StarBuddy / Lily）**：`~/local/DragonFlow/docs/device-control-demo.md`
- **Agora RTSA datastream 文档**：https://doc.shengwang.cn/doc/rtsa/c/advanced-features/data-stream
- **BK7258 实现复盘原始文档**：`~/local/Conversational-AI-IOT-Sample/plans/device-control-firmware-retrospective.md`
- **BK7258 主 commit**：`f1063d1 feat(conv-ai): parse cloud datastream device-control commands`（branch `bk7258/conv-ai-command`）
