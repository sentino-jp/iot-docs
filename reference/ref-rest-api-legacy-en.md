# REST API Reference — Legacy & Extension Endpoints

> This document collects two classes of endpoints **outside the mainstream integration path**:
> - **§8 Customize Agents** (`agentType=customize`): user-created Agent templates from the App. If your brand serves only the Sentino-curated Agent set, you may not need to expose these to end users.
> - **§9 Legacy** (`agentType=official`): early-path endpoints retained for backwards compatibility.
>
> The mainstream path (Sentino Agents, binding & query, conversation history) lives in the [main REST API Reference](./ref-rest-api.md).

---

## 8. Customize Agents

> Customize Agents (`agentType=customize`) let users create and maintain their own Agent templates. This section covers list, create, update, delete, and the helper lists used when populating dropdowns during creation.

---

### 8.1 Get Customized Agent List

Retrieve the list of Agent templates customized by the user.

```
POST /business-app/v1/agents/customize/agents-list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/customize/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

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

### 8.2 Create Customized Agent

Create a customized Agent template.

```
POST /business-app/v1/agents/customize/create
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Agent name |
| `description` | string | Yes | Agent description (character persona) |
| `avatarUrl` | string | Yes | Avatar URL |
| `llmModelId` | string | Yes | LLM model ID |
| `ttsVoiceId` | string | Yes | TTS voice ID |
| `langId` | string | Yes | Language ID |

**curl example**:

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

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 8.3 Update Customized Agent

```
POST /business-app/v1/agents/customize/update
Content-Type: application/json
```

**Body parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |
| `name` | string | No | Name |
| `description` | string | No | Description |
| `avatarUrl` | string | No | Avatar URL |
| `langId` | string | No | Language ID (see §8.5) |
| `llmModelId` | string | No | Model ID (see §8.5) |
| `ttsVoiceId` | string | No | Voice ID (see §8.5) |

**Response**: `data: null`; `code: 200` indicates success.

---

### 8.4 Delete Customized Agent

Delete a specified customized Agent template.

```
POST /business-app/v1/agents/customize/deleteById
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/customize/deleteById?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

```json
{
  "code": 200,
  "message": "success",
  "data": true
}
```

---

### 8.5 Helper Endpoints (Language / Voice / LLM Lists)

Used to populate dropdowns when creating/updating customized Agents (§8.2 / §8.3).

#### 6.15.1 Get Language List

```
POST /business-app/v1/agents/customize/language-list
```

**Response `data`**:

```json
[
  {"id": "lang_zh", "name": "中文"},
  {"id": "lang_en", "name": "English"},
  {"id": "lang_ja", "name": "日本語"}
]
```

#### 6.15.2 Get Voice List

```
POST /business-app/v1/agents/customize/voice-list
```

**Response `data`**:

```json
[
  {"id": "voice_001", "name": "甜美女声"},
  {"id": "voice_002", "name": "磁性男声"}
]
```

#### 6.15.3 Get LLM Model List

```
POST /business-app/v1/agents/customize/llm-list
```

**Response `data`**:

```json
[
  {"id": "model_gpt4", "name": "GPT-4"},
  {"id": "model_gpt35", "name": "GPT-3.5"}
]
```

---

## 9. Legacy Endpoints

> Endpoints retained for backwards compatibility only. **Do not use them in new integrations.** Current recommended paths are in §6 / §8.

---

### 9.1 Get Recommended Agent List (legacy)

> **Legacy endpoint**: the early "official recommendation" path, corresponding to `agentType=official`. New integrations should use [§6.6 Sentino Agent list](#66-get-recommended-sentino-agent-list). Fields are stale (`avatar` rather than `avatarUrl`; `tags` rather than `tagList`); may be deprecated in the future.

Retrieve the list of officially recommended Agent templates.

```
POST /business-app/v1/agents/recommend/agents-list
```

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/recommend/agents-list" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{}"
```

**Response**:

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

### 9.2 Get Agent Detail (legacy)

> **Legacy endpoint**: same source as §9.1, queries `agentType=official` only. New code should use [§6.7 Sentino Agent detail](#67-get-sentino-agent-detail).

Retrieve the detailed configuration of a specified Agent.

```
POST /business-app/v1/agents/detail
```

**Query parameters**:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `agentId` | string | Yes | Agent ID |

**curl example**:

```bash
curl -X POST "https://api-iot.sentino.jp/api/business-app/v1/agents/detail?agentId=$AGENT_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN"
```

**Response**:

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

**Related documents**: [REST API Reference (main)](./ref-rest-api.md) | [App Integration Guide](../guides/guide-app.md) | [Architecture & Concepts](../architecture-en.md)
