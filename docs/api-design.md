# API 设计

> 架构说明：无后端服务器。所有 AI 请求直接由浏览器发出，数据存储在 localStorage。
> 本文档描述：① 浏览器直连的外部 AI API、② 内部数据操作函数接口。

---

## 一、外部 AI API（浏览器直调）

### 通用规范

- 协议：`fetch` + SSE（`text/event-stream`）
- 路由逻辑：`getAPIConf(modelVal)` 按模型的 `provider` 字段决定使用哪个 endpoint 和 key

```javascript
function getAPIConf(modelVal) {
  const m = MODELS.find(x => x.value === modelVal);
  const c = cfg();
  if (m?.provider === 'deepseek') return { base: DEEPSEEK_BASE, key: c.deepseekKey };
  return { base: API_BASE, key: c.siliconflowKey };
}
```

### 1.1 DeepSeek 官方 API

- **Base URL**：`https://api.deepseek.com/v1`
- **适用模型**：`deepseek-chat`（V3）、`deepseek-reasoner`（R1）

### 1.2 SiliconFlow API

- **Base URL**：`https://api.siliconflow.cn/v1`
- **适用模型**：`deepseek-ai/DeepSeek-V4-Flash`、`Qwen/Qwen3-235B-A22B`、`moonshotai/Kimi-K2-Instruct` 等

### 1.3 Chat Completions（SSE 流式）

**请求：**
```
POST {base}/chat/completions
Authorization: Bearer {apiKey}
Content-Type: application/json
```

```json
{
  "model": "deepseek-chat",
  "messages": [
    { "role": "system", "content": "Agent 系统提示词 + 输出硬性规则" },
    { "role": "user",   "content": "用户消息" },
    { "role": "assistant", "content": "前序 Agent 回复（若有）" },
    ...
  ],
  "stream": true,
  "max_tokens": 512,
  "temperature": 0.85
}
```

**SSE 响应片段（每次 token）：**
```
data: {"choices":[{"delta":{"content":"这个问题..."}}]}
data: [DONE]
```

**注入的输出硬性规则（每次请求末尾追加到 system prompt）：**
```
【输出硬性规则】① 禁止使用**或任何Markdown符号，强调用"引号"代替。
② 总字数严格不超过200字。
③ 禁止自我介绍、套话开头、总结段，直接给出核心判断。
④ 用该人物的语气和思维方式直接作答，不加任何免责前缀。
```

### 1.4 联网搜索（fetchWebContext）

输入框工具栏联网开关启用后，`doSend` 发送前调用 `fetchWebContext(query)` 抓取相关资讯，将结果注入用户消息上下文。

**降级链（依次尝试）：**

1. **Tavily API** — `POST https://api.tavily.com/search`，需 `tavilyKey`
2. **Brave Search** — 公共端点，无需 Key
3. **DuckDuckGo** — 公共端点，无需 Key

所有通道均失败时返回空字符串，UI 显示 toast 提示，不中断对话流程。

---

## 二、内部数据操作函数（localStorage 层）

### 2.1 配置（`rta_cfg`）

```javascript
// 读取（自动 base64 解码：atob + decodeURIComponent）
cfg()  // → { deepseekKey, siliconflowKey, tavilyKey, model }

// 写入（自动 base64 编码：btoa + encodeURIComponent）
saveCfg({ deepseekKey, siliconflowKey, tavilyKey, model })

// 默认值（首次运行自动写入，Key 已移除硬编码默认值）
{
  deepseekKey: "",        // 空字符串，需用户填写
  siliconflowKey: "",     // 空字符串，需用户填写
  tavilyKey: "",          // 可选，用于联网搜索优先通道
  model: "deepseek-chat"
}
```

API Key 在 localStorage 中以 base64 混淆形式存储，读写对用户透明。个人中心显示脱敏格式（前6位 + ●●●●●●●●●● + 后4位）。

---

### 2.2 Session（对话会话，`rta_sess`）

```javascript
// 数据结构
{
  sessions: [
    {
      id: "sess_1748700000000",       // Date.now() 生成
      title: "用户首条消息前20字",
      createdAt: 1748700000000,
      updatedAt: 1748700000000,
      messages: [ /* Message[] */ ]
    }
  ]
}

// 读取全部
getSessions()  // → Session[]

// 读取单条
getSession(id) // → Session | null

// 新建
createSession(firstMsg)  // → Session

// 追加消息
addMessage(sessionId, message)  // → void

// 删除
deleteSession(id)  // → void
```

**Message 结构：**
```javascript
{
  id: "msg_1748700000001",
  role: "user" | "assistant",
  agentId: null | "munger" | "adler" | "karpathy" | "musk",
  agentName: null | "查理·芒格" | ...,
  content: "消息正文",
  createdAt: 1748700000001
}
```

---

### 2.3 Agent 系统提示词（`rta_pa`）

```javascript
// 读取某 Agent 的自定义提示词
getPrompt(agentId)  // → string（未自定义则返回内置默认值）

// 保存自定义提示词
savePrompt(agentId, promptText)  // → void
```

**主圆桌 Agent ID：** `munger` / `adler` / `karpathy` / `musk`

**场景会议室顾问 ID（ROOM_AGENTS）：** `munger` / `adler` / `karpathy` / `musk` / `jobs` / `feynman` / `dalio`

`rta_pa` 支持为上述所有 ID 存储自定义提示词。

---

### 2.4 发言流程（`doSend` → `streamOne`）

```javascript
// 用户发消息触发
doSend()
  └→ 解析 @mention，确定参与 Agent 列表
  └→ 渲染用户气泡
  └→ 依次调用 streamOne(agent, context)

// 单个 Agent 流式发言
streamOne(agent, messages)
  └→ 拼装 context（用户消息 + 前序 Agent 回复）
  └→ getAPIConf(model) → 选择 endpoint + key
  └→ fetch POST /chat/completions { stream: true }
  └→ 逐 token 调用 patchContent(mid, content)（RAF 批处理）
  └→ 完成后写入 localStorage
```

---

### 2.5 场景会议室发言流程（streamOneRoom）

场景会议室内用户发消息时，触发 `streamOneRoom`，流程与主圆桌 `streamOne` 类似但数据写入路径不同：

```javascript
// 场景会议室发言触发
rcDoSend(roomId)
  └→ 解析 @mention，确定参与顾问列表（从该房间 ROOM_AGENTS 中筛选）
  └→ 渲染用户气泡，写入 rtr_sessions[roomId]
  └→ 依次调用 streamOneRoom(agent, context, roomId)

// 单个顾问流式发言（场景会议室）
streamOneRoom(agent, messages, roomId)
  └→ 拼装 context（用户消息 + 前序顾问回复）
  └→ getAPIConf(model) → 选择 endpoint + key
  └→ fetch POST /chat/completions { stream: true }
  └→ 逐 token 调用 patchContent(mid, content)（RAF 批处理）
  └→ 完成后写入 rtr_sessions[roomId]（slice(0,20) + 每条 slice(-100)）
```

**与主圆桌的差异：**
- Agent 来源：`ROOM_AGENTS`（7位），而非 `AGENTS`（4位）
- 消息 role 字段：`"agent"`（而非 `"assistant"`）
- 消息含 `streaming` 布尔字段（流式进行中为 true，完成后为 false）
- 写入目标：`rtr_sessions` 而非 `rta_sess`

---

## 三、模型列表

| value | label | provider |
|-------|-------|----------|
| `deepseek-chat` | DeepSeek V3 (官方) ⚡ 推荐 | deepseek |
| `deepseek-reasoner` | DeepSeek R1 (官方) 深度思考 | deepseek |
| `deepseek-ai/DeepSeek-V4-Flash` | DeepSeek V4 Flash | siliconflow |
| `deepseek-ai/DeepSeek-V3` | DeepSeek V3 | siliconflow |
| `deepseek-ai/DeepSeek-R1` | DeepSeek R1 | siliconflow |
| `Pro/deepseek-ai/DeepSeek-R1` | DeepSeek R1 (Pro) | siliconflow |
| `Qwen/Qwen3-235B-A22B` | Qwen3 235B | siliconflow |
| `Qwen/Qwen3-30B-A3B` | Qwen3 30B | siliconflow |
| `Qwen/Qwen3-8B` | Qwen3 8B | siliconflow |
| `Qwen/Qwen2.5-72B-Instruct` | Qwen2.5 72B | siliconflow |
| `Pro/moonshotai/Kimi-K2.6` | Kimi K2.6 (Pro) | siliconflow |
| `Pro/zai-org/GLM-5.1` | GLM-5.1 (Pro) | siliconflow |

---

## 四、联网搜索

输入框工具栏开关控制。启用后在 `doSend` 前执行搜索，将结果注入用户消息上下文。

**降级链（依次尝试）：**

1. **Tavily API** — 需 `tavilyKey`，`POST https://api.tavily.com/search`
2. **Brave Search** — 公共端点，无需 Key
3. **DuckDuckGo** — 公共端点，无需 Key

所有通道均失败时返回空字符串，UI 显示 toast 提示，不中断对话流程。

---

## 五、辩论模式

输入框工具栏开关控制。启用后 `doSend` 执行两轮发言：

**第一轮：** 各 Agent 依次发表初始立场（同普通模式的 `streamOne`）。

**第二轮：** 各 Agent 看到其他 Agent 的第一轮发言后，进行交叉反驳（context 中包含其他 Agent 第一轮内容）。

UI 在两轮之间插入分隔线元素。

---

## 六、场景会议室数据操作（`rtr_*`）

```javascript
// 读取某房间的 sessions
rcGetRoomSessions(roomId)  // → Session[]

// 写入（每房间最多 20 条，每条最多 100 消息）
rcSaveRoomSessions(roomId, sessions)

// 读取/写入自定义房间
rcGetCustomRooms()          // → Room[]
rcSaveCustomRooms(rooms)

// 上次打开的 session
rcGetLastSess(roomId)       // → sessionId | null
rcSetLastSess(roomId, id)
```

**场景会议室 Message 结构（`rtr_sessions`）：**

```javascript
{
  id: "rc_uid字符串",
  role: "user" | "agent",
  agentId: null | "munger" | "adler" | ...,
  agentName: null | "查理·芒格" | ...,
  content: "消息内容",
  streaming: false
}
```

---

## 七、错误处理

| 场景 | 处理方式 |
|------|---------|
| API Key 未填 | 发送按钮禁用，输入框提示 |
| HTTP 非 200 | 读取 `res.json().error` 显示红色 toast |
| SSE 解析失败 | 跳过当前 token，继续读流 |
| `fetch` 网络异常 | `AbortController` 捕获，Agent 卡片显示错误态 |
| `[DONE]` 信号 | 结束流，保存消息到 localStorage |
| API 未返回 usage chunk | 按内容长度估算（÷1.8）写入统计，确保模型分布有数据 |
| 联网搜索全部失败 | 返回空字符串，显示 toast，继续对话 |
