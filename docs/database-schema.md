# 数据存储设计

> 技术：浏览器 `localStorage`（无后端、无数据库）
> 所有数据存储在用户浏览器本地，清除浏览器数据会导致丢失。

---

## localStorage Key 总览

| Key | 类型 | 内容 |
|-----|------|------|
| `rta_cfg` | Object | API Keys（base64混淆）+ 当前选用模型 |
| `rta_sess` | Array | 主圆桌所有对话 Session（最多50条） |
| `rta_favs` | Array | 收藏列表（完整对象，最多100条） |
| `rta_usage` | Object | 用量统计（token 数，按模型分类） |
| `rta_pa` | Object | Agent 自定义系统提示词（含场景会议室顾问） |
| `rtr_sessions` | Object | 场景会议室 sessions（key=roomId，每房间最多20条） |
| `rtr_custom_rooms` | Array | 用户自定义房间 |
| `rtr_last_sess` | Object | 各房间上次打开的 sessionId（key=roomId） |

---

## 详细结构

### `rta_cfg`（应用配置）

API Key 字段在 localStorage 中以 base64 混淆存储（btoa + encodeURIComponent），`cfg()` 读取时自动解码，`saveCfg()` 写入时自动编码。

```json
{
  "deepseekKey": "base64编码字符串",
  "siliconflowKey": "base64编码字符串",
  "tavilyKey": "base64编码字符串（可选）",
  "model": "deepseek-chat"
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `deepseekKey` | DeepSeek 官方 API Key（base64编码） | `""`（空字符串，需用户填写） |
| `siliconflowKey` | SiliconFlow API Key（base64编码） | `""`（空字符串，需用户填写） |
| `tavilyKey` | Tavily 联网搜索 API Key（base64编码，可选） | `""`（可选） |
| `model` | 当前使用模型 value | `"deepseek-chat"` |

---

### `rta_sess`（对话会话）

```json
{
  "sessions": [
    {
      "id": "sess_1748700000000",
      "title": "如何提高工作效率",
      "createdAt": 1748700000000,
      "updatedAt": 1748701000000,
      "messages": [
        {
          "id": "msg_1748700001000",
          "role": "user",
          "agentId": null,
          "agentName": null,
          "content": "如何提高工作效率",
          "createdAt": 1748700001000
        },
        {
          "id": "msg_1748700002000",
          "role": "assistant",
          "agentId": "munger",
          "agentName": "查理·芒格",
          "content": "先反过来想：如何确保效率低下？...",
          "createdAt": 1748700002000
        }
      ]
    }
  ]
}
```

**Session 字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | `"sess_" + Date.now()` |
| `title` | String | 首条用户消息前 20 字 |
| `createdAt` | Number | Unix 时间戳（ms） |
| `updatedAt` | Number | 每条消息写入后更新 |
| `messages` | Message[] | 该会话所有消息，按时间升序 |

**Message 字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | `"msg_" + Date.now()` |
| `role` | String | `"user"` 或 `"assistant"` |
| `agentId` | String \| null | Agent ID（仅 assistant 消息有值） |
| `agentName` | String \| null | Agent 名字快照（防止改名后历史显示错乱） |
| `content` | String | 消息正文 |
| `createdAt` | Number | Unix 时间戳（ms） |

---

### `rta_pa`（Agent 自定义提示词）

```json
{
  "munger":   "自定义提示词全文...",
  "adler":    "自定义提示词全文...",
  "karpathy": "自定义提示词全文...",
  "musk":     "自定义提示词全文...",
  "jobs":     "自定义提示词全文...",
  "feynman":  "自定义提示词全文...",
  "dalio":    "自定义提示词全文..."
}
```

支持主圆桌 4 位 Agent 及场景会议室全部 7 位顾问（ROOM_AGENTS）的自定义提示词。未自定义的 Agent 回退到代码内置默认提示词（`AGENTS` / `ROOM_AGENTS` 常量）。

---

### `rta_favs`（收藏列表）

存储完整的收藏对象，最多 100 条（`putFavs` 写入时 `slice(0,100)`）。

```json
[
  {
    "id": "uid字符串",
    "userMsgId": "用户消息ID",
    "question": "用户问题原文",
    "messages": [
      { "agentId": "munger", "content": "回复内容" }
    ],
    "savedAt": 1748700000000
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 收藏条目唯一 ID |
| `userMsgId` | String | 对应用户消息的 ID |
| `question` | String | 用户问题原文 |
| `messages` | Array | 各 Agent 回复摘要（agentId + content） |
| `savedAt` | Number | 收藏时间戳（ms） |

---

### `rta_usage`（用量统计）

记录 token 消耗，按模型分类统计。认知镜像雷达图数据从 `rta_sess` 中的 Agent 参与次数计算，不依赖此字段。

```json
{
  "inputTokens": 12345,
  "outputTokens": 6789,
  "byModel": {
    "deepseek-chat": { "input": 5000, "output": 3000 },
    "deepseek-ai/DeepSeek-V4-Flash": { "input": 7345, "output": 3789 }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `inputTokens` | Number | 全局输入 token 累计 |
| `outputTokens` | Number | 全局输出 token 累计 |
| `byModel` | Object | 各模型的 input/output 分类统计 |

---

### `rtr_sessions`（场景会议室对话）

每个房间最多 20 条 session（`rcSaveRoomSessions` 写入时 `slice(0,20)`），每条 session 最多 100 条消息（`slice(-100)`）。

```json
{
  "career": [
    {
      "id": "rc_uid字符串",
      "title": "对话标题（首条消息前20字）",
      "createdAt": 1748700000000,
      "updatedAt": 1748700001000,
      "messages": [
        {
          "id": "rc_uid",
          "role": "user",
          "agentId": null,
          "agentName": null,
          "content": "消息内容",
          "streaming": false
        },
        {
          "id": "rc_uid",
          "role": "agent",
          "agentId": "munger",
          "agentName": "查理·芒格",
          "content": "回复内容",
          "streaming": false
        }
      ]
    }
  ]
}
```

**Message 字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | `"rc_" + uid` |
| `role` | String | `"user"` 或 `"agent"` |
| `agentId` | String \| null | 顾问 ID（user 消息为 null） |
| `agentName` | String \| null | 顾问名字快照 |
| `content` | String | 消息内容 |
| `streaming` | Boolean | 流式进行中为 true，完成后为 false |

---

### `rtr_custom_rooms`（用户自定义房间）

```json
[
  {
    "id": "room_uid字符串",
    "name": "自定义房间名",
    "agentIds": ["munger", "jobs", "dalio"]
  }
]
```

---

### `rtr_last_sess`（各房间上次打开的 session）

```json
{
  "career": "rc_uid字符串",
  "startup": "rc_uid字符串"
}
```

---

## Agent 固定配置（代码内置，非 localStorage）

**主圆桌 AGENTS 常量（4 位）：**

| Agent ID | 名字 | 颜色 | 头像 |
|----------|------|------|------|
| `munger` | 查理·芒格 | `#3B82F6` | `assets/avatar-munger.jpg` |
| `adler` | 阿尔弗雷德·阿德勒 | `#F97316` | `assets/avatar-adler.jpg` |
| `karpathy` | 安德烈·卡帕西 | `#10B981` | `assets/avatar-karpathy.jpg` |
| `musk` | 埃隆·马斯克 | `#64748B` | `assets/avatar-musk.jpg` |

**场景会议室 ROOM_AGENTS 常量（7 位）：**

| Agent ID | 名字 | 颜色 | role | 头像 |
|----------|------|------|------|------|
| `munger` | 查理·芒格 | `#3B82F6` | 智慧博学家 | `assets/avatar-munger.jpg` |
| `adler` | 阿尔弗雷德·阿德勒 | `#F97316` | 心理重构者 | `assets/avatar-adler.jpg` |
| `karpathy` | 安德烈·卡帕西 | `#10B981` | 技术极客 | `assets/avatar-karpathy.jpg` |
| `musk` | 埃隆·马斯克 | `#64748B` | 效率狂人 | `assets/avatar-musk.jpg` |
| `jobs` | 史蒂夫·乔布斯 | `#1d1d1f` | 产品哲学家 | 无图片，emoji 🍎 |
| `feynman` | 理查德·费曼 | `#D97706` | 科学思想家 | 无图片，emoji 🔬 |
| `dalio` | 雷·达利奥 | `#0F766E` | 原则主义者 | 无图片，emoji 📊 |

---

## 数据维护

```javascript
// 读取配置（自动 base64 解码）
cfg()

// 写入配置（自动 base64 编码）
saveCfg({ deepseekKey, siliconflowKey, tavilyKey, model })

// 读取所有主圆桌会话
getSessions()

// 清空所有对话（保留配置和提示词）
clearAllSessions()

// 写入收藏（自动截断到100条）
putFavs(favs)  // slice(0,100)

// 写入主圆桌 sessions（自动截断到50条）
putSessions(sessions)  // slice(0,50)

// 写入场景会议室 sessions（自动截断：每房间20条，每条100消息）
rcSaveRoomSessions(roomId, sessions)  // slice(0,20)，每条消息 slice(-100)

// 在设置面板操作：
// 「清除所有数据」→ 删除 rta_sess、rta_favs、rta_usage、rtr_sessions、rtr_custom_rooms、rtr_last_sess
```

**存储上限汇总：**

| 数据 | 上限 | 截断方式 |
|------|------|---------|
| 主圆桌 Session | 最多 50 条 | `putSessions` 写入时 `slice(0,50)` |
| 收藏夹 | 最多 100 条 | `putFavs` 写入时 `slice(0,100)` |
| 场景会议室 Session | 每房间最多 20 条 | `rcSaveRoomSessions` 写入时 `slice(0,20)` |
| 场景会议室 Session 内消息 | 每条最多 100 条 | 写入时 `slice(-100)` |

> ⚠️ **注意**：localStorage 存储上限约 5MB。长期高频使用后可在设置面板手动清理旧会话。
