# Sentino 物模型 · 固件开发指南

> 面向**固件团队**的物模型（Thing Model）对接文档：从 Sentino 平台后台 DP 定义 → MQTT 协议 → 固件实现 → 联调验证的完整闭环。
>
> **三段结构**：
> - **Part A 协议契约** —— 物模型 MQTT 协议字段、上行/下行消息、ack 规则
> - **Part B 固件实现模式** —— 经过 BK7258 实测验证的工程模板（snapshot / debounce / range mapping / defensive echo）
> - **Part C 接入新 DP & 排查** —— 给固件工程师跟 PO 协调 DP 时的考虑维度 + 联调 checklist

---

## 0. 这份文档帮你做什么

| 部分 | 内容 | 谁要看 |
|---|---|---|
| **Part A（§1-3）** | 物模型协议契约：MQTT 字段、消息格式、上下行流程 | 所有接入方 |
| **Part B（§4-7）** | 固件实现模式：注册时序、snapshot、debounce、range mapping、defensive echo | 要从零写设备端的 |
| **Part C（§8-10）** | 接入新 DP 流程 + DP 设计方法论 + 联调 checklist | 接入新 DP 或排查问题的 |

> 本文配套 `~/local/Conversational-AI-IOT-Sample/device/projects/beken_genie/main/app_dp_handler.{h,c}` 作为参考实现。文中所有 file:line 引用基于 2026-05-15 该 repo 的状态。

---

## 0.1 物模型 vs 设备控制协议（边界澄清）

Sentino 架构里有两套设备相关的协议，**不要混淆**：

| 维度 | 物模型（本文档） | 设备控制协议 |
|---|---|---|
| **传输通道** | MQTT 信令 (`rlink/v2/...`) | Agora RTC DataStream（二进制广播） |
| **服务对端** | Sentino IoT 平台 | Sentino Agent 平台（DragonFlow） |
| **数据形态** | DP 属性键值（`switch=true`、`volume_set=8`） | OpenAI tool calling envelope（`actions[]` + `executor` + `parameters`） |
| **典型例子** | 音量、亮度、电量、开关、充电状态 | LCD 动画、播表情、震动、播音效 |
| **上行/下行** | 双向（property_report ↔ property_set） | P1 阶段单向（agent → device） |
| **触发方** | App / 平台 / Agent 都可写 | Agent (LLM Function Call) |
| **生命周期** | 持久状态（断电后存在 NVS） | 瞬时事件（fire-and-forget） |
| **离线行为** | 设备本地仍生效 | 无意义（无连接没有 agent） |
| **协议参考** | `~/local/iot-docs/reference/ref-mqtt.md` §4.5 / §4.6 / §5.4 | DragonFlow `device-control-firmware-integration-guide.md` + `~/local/iot-docs/control/` 目录 |

→ 同一台设备**同时实现两套**，互不干扰。本文只讲物模型这一套。

---

# Part A：协议（讲清楚要实现什么）

---

## 1. 端到端链路

```
┌──────────┐  MQTT (rlink/v2/.../report)   ┌──────────┐
│  设备    │ ───────────property_report──→ │ Sentino  │
│ 固件     │ ←────property_set─────────── │ IoT 平台 │
│          │     (rlink/v2/.../issue)      │          │
└──────────┘                                └────┬─────┘
                                                  │
                                                  │ 后台展示
                                                  ▼
                                          ┌──────────────┐
                                          │ App / 用户   │
                                          │ 设备管理页   │
                                          │ "物模型数据" │
                                          └──────────────┘
```

**设备端关心的边界**：
- ✅ 开机后注册 DP set callback（接 `property_set`）+ cloud-ready callback（push snapshot）
- ✅ 边沿事件主动 `property_report`（电量阈值、按键、状态变化）
- ✅ 定期或事件驱动的 snapshot（断线重连后复位云端视图）
- ✅ 收到 `property_set` 后**应用值 + 回 echo report**（光靠 SDK 的 issue_response 不够，云端 shadow 不会更新）
- ❌ 不关心：MQTT 鉴权、Topic 拼接、连接管理（SDK 包了）
- ❌ 不关心：物模型 schema 校验（云端已经按 product 定义校验过）

## 2. MQTT 协议层（参考摘要）

完整协议见 `~/local/iot-docs/reference/ref-mqtt.md`。固件视角只需关注以下 4 个 code：

### 2.1 上行：`property_report`（属性上报）

设备→云。`code = "property_report"`，建议 ack = 0（云端不回）。Topic：`rlink/v2/${pid}/${uuid}/report`。

```jsonc
{
  "id": "msg-uuid-v4",
  "ts": 1747353600,
  "code": "property_report",
  "data": {
    "properties": {
      "switch": true,
      "volume_set": 8,
      "battery_percentage": 80,
      "charge_status": false
    }
  },
  "ack": 0
}
```

可单 DP 上报、可批量（snapshot 模式）。批量上报体现的是"对同一时刻设备状态的快照"，云端按字段全量更新。

### 2.2 下行：`property_set`（设置属性）

云→设备。`code = "property_set"`。Topic：`rlink/v2/${pid}/${uuid}/issue`。

```jsonc
{
  "id": "msg-uuid-v4",
  "ts": 1747353600,
  "code": "property_set",
  "data": {
    "properties": {
      "volume_set": 5
    }
  }
}
```

设备**必须**：
1. 应用 actuator（如 `volume_set_abs(level)`）
2. **再发一条 `property_report` 把 actually applied 的值回报**（仅靠 SDK 自动的 `issue_response` 不会更新云端 shadow，详见 §6）
3. 可选回 `issue_response`（SDK 自动处理，业务层不用管）

### 2.3 上行：`model`（获取产品物模型）

设备→云，`code = "model"`，建议 ack = 1。设备主动请求云端推送当前产品的物模型 schema 定义。

请求体：`{"format": "complete"}`（其他选项：`simple` / `mini`，详见 ref-mqtt.md §4.5）。

云端回三档详细程度的 schema，告诉设备"这个产品支持哪些 DP，每个 DP 的类型/范围/读写模式是什么"。**当前 BK7258 sample 没用这个**——DP 标识符是固件硬编码（见 §3.1）。

### 2.4 下行（罕用）：`clean_data` / `reset` 等

详见 ref-mqtt.md §5。本指南不展开。

---

## 3. 物模型与产品配置

### 3.1 三层声明的对齐

物模型的"事实源"分布在三个地方，必须保持一致：

| 层 | 谁维护 | 内容 | 失配后果 |
|---|---|---|---|
| **Sentino 平台后台**（产品管理 → 功能设计） | PO / 客户运营 | DP 列表 + identifier + 类型 + 范围 + 读写模式 | 后台不认识的 DP → 上报数据被 drop / 控制下发 reject |
| **固件**（`app_dp_handler.h` 等） | 固件团队 | DP_ID_xxx 常量字符串 | identifier 写错 → property_set 收不到 / 上报值在云端不显示 |
| **业务代码**（actuator） | 固件团队 | volume_set_abs / battery_get_charge_level / ... | 实现缺失 → 上报值假、控制无效 |

**强约束**：固件 `DP_ID_xxx` 字符串**必须精确匹配**云端 product 中定义的 identifier。云端按 key 查 schema，key 不匹配整条 message drop。例子见 `app_dp_handler.h:23-26`：

```c
#define DP_ID_SWITCH                "switch"
#define DP_ID_BATTERY_PERCENTAGE    "battery_percentage"
#define DP_ID_VOLUME_SET            "volume_set"
#define DP_ID_CHARGE_STATUS         "charge_status"
```

### 3.2 当前 BK7258 sample 的 4 个 DP

来自 IoT 平台"产品管理 → 功能设计"配置的 product OQm9yRoaLq1gbK：

| identifier | 中文名 | 类型 | 数据定义 | 读写 | 触发方式 | 实现位置 |
|---|---|---|---|---|---|---|
| `switch` | 开关 | bool | `false:关, true:开` | 可下发可上报 | 云端 set 即刻 echo + snapshot 2min | `app_dp_handler.c:112-126` |
| `volume_set` | 音量 | int | `[0, 10]`，步长 1 | 可下发可上报 | 本地按键 500ms debounce + 云端 set echo + snapshot | `app_dp_handler.c:127-147` |
| `battery_percentage` | 电量 | int | `[0, 100]`，步长 1 | **只上报**（read-only） | 周期采样 + 阈值变化触发 | `app_dp_handler.c:204-209` + `app_battery_dp.c` |
| `charge_status` | 充电状态 | enum | `0,1` | **只上报**（read-only） | 周期采样 + 状态变化触发 | `app_dp_handler.c:197-202` + `app_battery_dp.c` |

→ 「可下发可上报」DP 是 agent 控制候选（详见 [[concepts/iot-agent-capability-bridging]]，固件 SUPPORTED_EXECUTORS 集合的物模型来源）。

---

# Part B：固件实现模式（怎么实现）

---

## 4. 注册时序（关键）

```c
void app_dp_handler_init(void)
{
    Register_Sentino_Dp_Set_Cb(on_dp_set);                  /* 接 property_set 下发 */
    Register_Sentino_Cloud_Ready_Cb(app_dp_report_snapshot); /* 上线后 push 全量 snapshot */
}
```

**强约束**：`app_dp_handler_init()` **必须在 `sentino_iot_engine_init()` 之前调用**——

> Engine 在第一次 MQTT CONNECTED 时同步触发 cloud-ready callback，可能发生在 `engine_init()` 内部。如果晚于 engine 注册 callback，会**漏掉首次 snapshot**——这是 sample 实测踩过的坑（`app_dp_handler.h:31-33`）。

正确顺序：

```c
app_dp_handler_init();      /* 1. 先注册 callbacks */
sentino_iot_engine_init();  /* 2. 后启动 engine */
```

## 5. 上行模式

### 5.1 Cloud-Ready Snapshot（上线后批量推全 DP）

设备 MQTT 上线后，把所有 DP 的当前值打包一次性 push 给云端，**复位云端 shadow 与设备实际状态的一致性**。这一步必须做——否则断线重连后云端 shadow 是断线前的旧值。

实现要点（`app_dp_handler.c:165-191`）：

```c
void app_dp_report_snapshot(void)
{
    /* 冷却期防抖：reconnect storm 时避免一秒内 push 多次 */
    uint32_t now = (uint32_t)rtos_get_time();
    if (s_snapshot_done && (now - s_last_snapshot_ms) < DP_SNAPSHOT_COOLDOWN_MS) {
        LOGI("snapshot skipped (cooldown)\n");
        return;
    }
    s_last_snapshot_ms = now;
    s_snapshot_done = true;

    unsigned char pct = 0;
    bool charging = false;
    app_dp_battery_sample(&pct, &charging);

    dp_obj_t arr[4];
    dp_set_bool(&arr[0], DP_ID_SWITCH,             s_switch_state);
    dp_set_int (&arr[1], DP_ID_VOLUME_SET,         local_to_cloud_volume((uint8_t)volume_get_current()));
    dp_set_int (&arr[2], DP_ID_BATTERY_PERCENTAGE, (int32_t)pct);
    dp_set_bool(&arr[3], DP_ID_CHARGE_STATUS,      charging);

    Sentino_Dp_Report_Many_Export(arr, 4);  /* 批量一条 MQTT */
}
```

**注意点**：
- **批量上报用 `Sentino_Dp_Report_Many_Export(arr, n)`**——单条 MQTT 包含多个 DP，比逐个发省 N-1 次 publish
- **冷却期 `DP_SNAPSHOT_COOLDOWN_MS = 120000`**（2 分钟）——reconnect storm 场景下防止刷爆 broker
- **冷却 sentinel 用 `bool first_done`，不用 `INT32_MIN`**——后者在第一次 `now - s_last` 时触发 signed overflow UB（GCC 实测把首次 snapshot wrongly skip 掉，2026-05-15 验证）

### 5.2 边沿事件单 DP 上报

电量百分比变化、充电状态翻转等需要立即报告的事件，用单 DP push（`app_dp_handler.c:197-209`）：

```c
void app_dp_report_charge_status(bool charging)
{
    dp_obj_t dp;
    dp_set_bool(&dp, DP_ID_CHARGE_STATUS, charging);
    Sentino_Dp_Report_Export(&dp);
}

void app_dp_report_battery_percentage(unsigned char pct)
{
    dp_obj_t dp;
    dp_set_int(&dp, DP_ID_BATTERY_PERCENTAGE, (int32_t)pct);
    Sentino_Dp_Report_Export(&dp);
}
```

调用方（`app_battery_dp.c`）按业务规则触发——周期采样 + 阈值检查 + 边沿检测：

```
周期 sample 电量
  if (|pct - last_reported_pct| >= 5)  /* 5% 阈值 */
    app_dp_report_battery_percentage(pct);
    last_reported_pct = pct;
```

### 5.3 本地输入 Debounce（按键场景）

用户按音量键 +5 次，硬件中断会触发 5 次 VOLUME_UP 事件。每次都 publish 会造成中间值瞬间被覆盖（5 → 6 → 7 → 8 → 9 → 10），云端 shadow 闪一下 5,6,7,8,9 都是噪声。

**解决**：500ms one-shot timer，每次按键重置定时器，只 publish 最终值（`app_dp_handler.c:220-257`）：

```c
#define VOLUME_REPORT_DEBOUNCE_MS  500

void app_dp_request_volume_report(unsigned char local_level)
{
    s_vol_pending = local_level;

    if (rtos_is_oneshot_timer_init(&s_vol_timer)) {
        /* 已有 timer：reload 重置周期 */
        rtos_oneshot_reload_timer_ex(&s_vol_timer,
                                     VOLUME_REPORT_DEBOUNCE_MS,
                                     vol_debounce_fire, NULL, NULL);
    } else {
        /* 首次：init + start */
        rtos_init_oneshot_timer(...);
        rtos_start_oneshot_timer(&s_vol_timer);
    }
}

static void vol_debounce_fire(void *larg, void *rarg)
{
    /* 实际 publish 在这里 */
    int32_t cloud_v = local_to_cloud_volume(s_vol_pending);
    dp_obj_t dp;
    dp_set_int(&dp, DP_ID_VOLUME_SET, cloud_v);
    Sentino_Dp_Report_Export(&dp);
}
```

**关键**：cloud-set 路径不用 debounce——云端发 `property_set` 时期望 immediate ack，必须在 `on_dp_set()` 里同步 echo。

## 6. 下行模式：on_dp_set + Echo Back

### 6.1 SDK 的 `issue_response` 不更新云端 shadow

收到 `property_set` 后，SDK 默认会自动回 `issue_response`（命令应答）。**但 `issue_response` 只表示"命令收到"，不更新云端 DP shadow 状态**——云端 `getDpInfos` API 仍返回旧值，App UI 显示 stale。

→ **必须在业务侧 `on_dp_set()` 里再发一条 `property_report` 回 echo applied value**，云端才真正更新 shadow（2026-04-24 实测验证，`app_dp_handler.c:108-110`）。

### 6.2 Echo "Applied"，不是 "Requested"

云端发 `volume_set=5`，但本地 SPK_VOLUME_LEVEL 是 11 档（0-10），cloud value 5 → 映射 → local level → 应用 → **再读回 local current → 反映射 → cloud value → 回报**。

为什么不直接 echo cloud=5？因为：
- 如果 local volume 控制有最小步长（如 only even numbers），mapping 后实际 land on 4
- 如果不报真实 applied，下次 cloud query 时云端以为 5，App slider 在 5，但耳朵听到的是 4 档音量——**不同视图持续 drift**

实现（`app_dp_handler.c:127-147`）：

```c
} else if (0 == strcmp(dp->identifier, DP_ID_VOLUME_SET)) {
    if (dp->type != DP_TYPE_INT) {
        LOGW("DP %s: expected INT, got type=%d — ignoring\n", dp->identifier, dp->type);
        return;
    }
    /* Cloud [0,10] → local SPK_VOLUME_LEVEL */
    uint8_t local_lv = cloud_to_local_volume(dp->v.i);
    volume_set_abs(local_lv, 0);
    /* 读回当前 local，反映射回 cloud range */
    int32_t echoed_cloud = local_to_cloud_volume((uint8_t)volume_get_current());
    LOGI("DP %s <- cloud=%ld → local=%u → echo cloud=%ld\n",
         dp->identifier, (long)dp->v.i, local_lv, (long)echoed_cloud);

    dp_obj_t echo;
    dp_set_int(&echo, DP_ID_VOLUME_SET, echoed_cloud);
    Sentino_Dp_Report_Export(&echo);
}
```

### 6.3 防御性 Read-Only 拒绝

云端 product schema 把 DP 标为 read-only（如 `battery_percentage`、`charge_status`），但 IoT 平台 `propsIssue` HTTP 端点**不强制 server-side accessMode**，buggy client 可能调一个 set request 进来。

→ 固件**必须**在 `on_dp_set()` 里防御性拒绝，不要被骗着改 actuator：

```c
} else if (0 == strcmp(dp->identifier, DP_ID_BATTERY_PERCENTAGE) ||
           0 == strcmp(dp->identifier, DP_ID_CHARGE_STATUS)) {
    LOGW("DP %s is read-only — ignoring cloud-set\n", dp->identifier);
}
```

## 7. Range Mapping 模式（本地与云端值域不同时）

云端 product schema 经常用人类友好的 range（音量 0-10），固件可能用更细的 step（0-100 或硬件 step 数）。**双向映射器**保证两侧视图一致：

```c
/* 不要 bake in "云 10 = 本地 N"——本地档数会变 */
static int32_t local_to_cloud_volume(uint8_t local_lv)
{
    uint32_t max = volume_get_level_count();   /* 运行时取本地档数 */
    if (max <= 1) return 0;
    return (int32_t)(((uint32_t)local_lv * 10 + (max - 1) / 2) / (max - 1));
}

static uint8_t cloud_to_local_volume(int32_t cv)
{
    uint32_t max = volume_get_level_count();
    if (max <= 1) return 0;
    if (cv < 0)  cv = 0;
    if (cv > 10) cv = 10;
    return (uint8_t)(((uint32_t)cv * (max - 1) + 5) / 10);
}
```

**模式总结**：
- 用本地 SDK API 取真实档数，不写常量
- 取整时 `+ (max - 1) / 2` 是四舍五入而不是 floor，避免单方向 drift
- 双向映射器对称——`local_to_cloud(cloud_to_local(x))` 在合理输入下应能稳定收敛（不一定 == x，但相邻 x 收敛到同一值）

---

# Part C：接入新 DP & 排查（怎么扩展）

---

## 8. 接入新 DP 的 5 步流程

假设要加一个新 DP `night_mode`（bool，可下发可上报）：

### 步骤 1：与 PO 确认 DP 定义

跟 PO/客户运营对齐：
- identifier（命名规范：snake_case，与已有 DP 风格一致）
- 类型（bool / int / float / enum / text / struct / array）
- 取值范围、单位、步长
- 读写模式（r / rw）
- 上报触发方式（事件 / 周期 / 状态变化）

> **DP 设计方法论见 §9**——固件团队跟 PO 协调时的考虑维度

### 步骤 2：PO 在 IoT 平台后台添加 DP

PO 在"产品管理 → 功能设计"添加新自定义功能（identifier 必须与固件约定一致），保存后云端 product schema 立即生效，无需 Sentino 发版。

### 步骤 3：固件加 DP_ID 常量 + on_dp_set 分支

```c
/* app_dp_handler.h */
#define DP_ID_NIGHT_MODE  "night_mode"
```

```c
/* app_dp_handler.c on_dp_set() 加分支 */
} else if (0 == strcmp(dp->identifier, DP_ID_NIGHT_MODE)) {
    if (dp->type != DP_TYPE_BOOL) {
        LOGW("DP %s: expected BOOL, got type=%d — ignoring\n", dp->identifier, dp->type);
        return;
    }
    s_night_mode = dp->v.b;
    apply_night_mode(s_night_mode);   /* 业务侧 actuator */

    dp_obj_t echo;
    dp_set_bool(&echo, DP_ID_NIGHT_MODE, s_night_mode);
    Sentino_Dp_Report_Export(&echo);
}
```

### 步骤 4：snapshot 加新 DP

```c
/* app_dp_report_snapshot() arr 数组扩到 5 */
dp_obj_t arr[5];
dp_set_bool(&arr[0], DP_ID_SWITCH,             s_switch_state);
dp_set_int (&arr[1], DP_ID_VOLUME_SET,         ...);
dp_set_int (&arr[2], DP_ID_BATTERY_PERCENTAGE, ...);
dp_set_bool(&arr[3], DP_ID_CHARGE_STATUS,      ...);
dp_set_bool(&arr[4], DP_ID_NIGHT_MODE,         s_night_mode);
Sentino_Dp_Report_Many_Export(arr, 5);
```

### 步骤 5：边沿/事件触发上报（如果是只读或本地触发）

如果是只读或本地触发型 DP，按 §5.2 模式加单 DP push 接口；如果有 burst 风险（按键、滑块），按 §5.3 加 debounce timer。

## 9. DP 设计方法论（固件团队跟 PO 协调时的视角）

PO 来谈"加一个新 DP"时，固件工程师应该问以下问题：

### 9.1 类型选择

| 业务语义 | 推荐类型 | 注意 |
|---|---|---|
| 真二选一（开/关、是/否） | `bool` | 不要用 enum 0/1 —— bool 更直观 |
| 模式切换（白天/夜晚/勿扰） | `enum`（int 枚举） | 枚举值用 0,1,2，名字记在 schema 的 enum 描述里 |
| 数值（音量、亮度、温度） | `int` 或 `float` | 优先 int + 合理范围；只在物理量本身是连续才 float |
| 字符串（设备别名、最近播放歌名） | `text` | 注意 max length（默认 10240，DP 实际多用 ≤ 64） |
| 复合状态（一组相关字段） | `struct` | 慎用——多个独立 DP 通常更好维护 + 部分更新 |
| 时间戳 | `date`（UTC ms 字符串） | 不要用 int 表 unix timestamp —— 易混秒/毫秒 |

### 9.2 读写模式

- **r（只读）**：状态属性（电量、信号、传感器读数）。固件**必须**在 `on_dp_set()` 防御性拒绝（§6.3）
- **rw（可读可写）**：可被云端/App/Agent 修改的属性（音量、开关、模式）。**固件 echo applied value 是必须的**（§6.1）

### 9.3 上报频率与触发

| 触发模式 | 适用 | 实现 |
|---|---|---|
| **状态变化** | 开关、模式切换 | 应用后立即 push（见 §5.2） |
| **阈值变化** | 电量（每变 5%）、信号强度（每变一格） | 周期 sample + 阈值检测，超阈值才 push |
| **本地 burst → debounce** | 按键、滑块 | 500ms one-shot（见 §5.3） |
| **周期** | 几乎不用 | 大多数情况下都能改成上面三种之一 |

**反模式**：每秒 push 一次电量 → 浪费带宽 + 对端没用（App 不需要那么实时）。改为阈值触发（每变 5% 才报）。

### 9.4 范围与单位

- **数值范围**写在 schema 里（`min`, `max`），让云端能 reject 越界值。**不要靠固件在 `on_dp_set()` 里裁剪**——schema 是合同
- **单位**写在 schema `unit` 字段（如 `%`, `°C`, `Hz`），App UI 自动展示
- **步长**用 `step`，App slider 按步长走

### 9.5 命名规范

- `snake_case`（与 ref-mqtt 示例一致）
- 名词或"动词_对象"：`switch` / `volume_set` / `battery_percentage`
- 状态属性用名词（`brightness`、`battery_percentage`），可写动作用动词（`set_volume` 罕见——通常直接用属性名 `volume_set` 表示"该属性的设置值"）
- 不要用 camelCase / kebab-case — **必须**与 ref-mqtt 示例风格一致

### 9.6 与设备控制协议（agent capability）的关系

「可下发可上报」DP 是 agent 控制能力的物模型来源（详见 [[concepts/iot-agent-capability-bridging]]）。如果新增的 DP 是 Agent 应该能控制的（如 `night_mode`），固件还需要：
- 在 `SUPPORTED_EXECUTORS[]` 维护表里加 executor 名（与 DragonFlow product.capability_config 中的 executor 字段一致）
- 在固件 dispatch 里加分支翻译 executor → DP 操作

**注**：本指南只覆盖物模型层。Agent capability 协商架构详见 `~/local/iot-docs/control/design.md` §4 + 上述 wiki concept。当前 capability 协商章节按 IoT 团队 Layer 2 拍板进度待补——固件团队按"假设支持全部能力"实现 dispatch 即可。

## 10. 联调 Checklist

接入新 DP 后建议按顺序验证：

1. ✅ **PO 后台 DP 配置正确** — 在产品管理页确认 identifier / 类型 / 范围 / 读写模式与约定一致
2. ✅ **固件 DP_ID 常量与后台一致** — `grep DP_ID_xxx` 看代码常量，与后台拷贝对比，**字符大小写都要一致**
3. ✅ **开机 snapshot 上报** — 设备开机后 1-2 秒内，平台"设备管理 → 物模型数据"页面看到新 DP 出现且值正确
4. ✅ **本地触发 → 云端反映** — 操作设备本地（按键、状态变化），平台"物模型数据"页面更新到新值
5. ✅ **云端下发 → 设备执行 + echo** — 在平台调用 `propsIssue` 接口或 App 控件触发 set，设备本地真执行，且平台"物模型数据"页面**也更新到 set 后的值**（不是只更新 issue_response 的 ack）
6. ✅ **断线重连后 snapshot 复位** — 拔 WiFi → 重连 → 平台看到 snapshot 重新 push（cooldown 期内会 skip log），云端 shadow 与设备状态一致
7. ✅ **read-only DP 防御** — 用 `propsIssue` 强制下发一个 read-only DP（如 `battery_percentage=50`），看固件 log 是否打 `is read-only — ignoring cloud-set` 警告（不应真改值）

任一步骤断链，对照前后两段输出（设备 log + 平台"物模型数据"页面）即可定位问题层。

---

## 附录 A：常见错误

| 错误 | 症状 | 修法 |
|---|---|---|
| identifier 大小写不一致 | property_set 收不到 / property_report 上去云端不更新 | 严格对齐 IoT 平台后台与固件常量字符串 |
| 收到 set 后只回 issue_response 不发 property_report | 设备 actuator 真执行了，但平台 shadow / App UI 永远是旧值 | `on_dp_set()` 末尾必须 `Sentino_Dp_Report_Export(echo)` |
| Snapshot 漏首次 | 设备首次开机后云端看不到 DP 值，重启后才有 | 确认 `app_dp_handler_init()` 在 `sentino_iot_engine_init()` **之前**调用 |
| 按键上报刷屏 | App 滑动音量看到 5,6,7,8,9 闪烁 | 按 §5.3 加 500ms one-shot debounce |
| Read-only DP 被改 | 云端 buggy client 调 propsIssue 把 battery_percentage 改成 50 | `on_dp_set()` 加 read-only 分支 LOGW + 直接 return |
| Snapshot 冷却失效 | reconnect storm 时刷爆 broker | 别用 `INT32_MIN` 作 sentinel（signed overflow UB），用 `bool first_done` 单独判定（§5.1） |
| Range mapping 单方向 drift | 设 5 → echo 4 → 再设 5 → echo 4，永远收敛不到 5 | 检查双向映射器，取整用四舍五入而不是 floor |

## 附录 B：相关文档

- [`~/local/iot-docs/reference/ref-mqtt.md`](../reference/ref-mqtt.md) — MQTT 协议完整参考（§4.5/§4.6/§5.4 物模型部分）
- [`~/local/iot-docs/guides/guide-device.md`](./guide-device.md) — 设备整体集成指南（含 §7.3 model 接口简要说明）
- [`~/local/iot-docs/control/`](../control/) — 设备控制协议（**与本文档物模型完全独立**——见 §0.1 边界澄清）
  - `design.md` — 整体设计 + §4 能力协商架构（v5）
  - `device-control-firmware-implementation-guide.md` — 设备控制协议固件接入指南
  - `product-capability-guide.md` — Capability 配置指南（PO 视角）
- `~/local/Conversational-AI-IOT-Sample/device/projects/beken_genie/main/app_dp_handler.{h,c}` — 本文档所有代码引用的实现源文件
- `~/local/Conversational-AI-IOT-Sample/device/projects/beken_genie/main/app_battery_dp.c` — 边沿事件触发 DP 上报的具体实现（电量+充电状态）
