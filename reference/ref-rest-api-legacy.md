# REST API 参考 — 遗留与扩展接口

> 本文档收录两类**不在主流接入路径上的接口**：
> - **§8 自定义智能体**（`agentType=customize`）：用户在 App 端自行创建的 Agent 模板。如品牌方提供 Sentino 维护的智能体集已能满足需求，无需暴露这些接口给最终用户。
> - **§9 Legacy**（`agentType=official`）：早期路径，仅向后兼容保留。
>
> 主流路径（Sentino 智能体、绑定查询、对话历史）见 [REST API 参考主文档](./ref-rest-api.md)。

---

## 8. 自定义智能体接口

> 自定义智能体（`agentType=customize`）允许用户创建并维护自己的 Agent 模板。本节涵盖列表、创建、更新、删除及创建时填充下拉框用的辅助列表。

---

### 8.1 获取自定义智能体列表

获取用户自定义的智能体模板列表。

```
POST /business-app/v1/agents/customize/agents-list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/customize/agents-list" \
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

### 8.2 创建自定义智能体

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
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/customize/create" \
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

### 8.3 更新自定义智能体

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
| `langId` | string | 否 | 语言 ID（见 §8.5） |
| `llmModelId` | string | 否 | 模型 ID（见 §8.5） |
| `ttsVoiceId` | string | 否 | 音色 ID（见 §8.5） |

**响应**：`data: null`，`code: 200` 表示成功。

---

### 8.4 删除自定义智能体

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
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/customize/deleteById?agentId=$AGENT_ID" \
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

### 8.5 辅助接口（语言 / 音色 / LLM 列表）

创建/更新自定义智能体（§8.2 / §8.3）时用于填充下拉框。

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

## 9. 遗留接口

> 本节列出仅向后兼容保留的接口。**新接入请勿使用**。当前推荐路径见 §6 / §8。

---

### 9.1 获取推荐智能体列表 (legacy)

> **遗留接口**：早期「官方推荐」路径，对应 `agentType=official`。新接入请改用 [§6.6 Sentino 智能体列表](#66-获取推荐-sentino-智能体列表)。本接口返回字段集旧（`avatar` 而非 `avatarUrl`、`tags` 而非 `tagList`），未来可能下线。

获取官方推荐的智能体模板列表。

```
POST /business-app/v1/agents/recommend/agents-list
```

**curl 示例**：

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/recommend/agents-list" \
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

### 9.2 获取智能体详情 (legacy)

> **遗留接口**：与 §9.1 同源，仅查询 `agentType=official` 类智能体。新代码请用 [§6.7 Sentino 智能体详情](#67-获取-sentino-智能体详情)。

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
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/detail?agentId=$AGENT_ID" \
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

**相关文档**：[REST API 参考（主）](./ref-rest-api.md) | [App 端集成指南](../guides/guide-app.md) | [架构与概念](../architecture.md)
