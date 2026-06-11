# 圆桌 AI — 产品需求文档（PRD）

> 版本：v1.5 | 日期：2026-06-07 | 状态：已上线运行（单文件 SPA）

---

## 一、产品概述

**产品名称：** 圆桌 AI（Roundtable · Decision Sanctuary）

**一句话描述：** 一个私人思考工具——你提出问题，4 个 AI 角色依次发言、互相看到彼此观点，帮你从不同维度反思和决策。

**核心体验：**
你发消息 → 4 个 Agent 依次实时生成回应（后者能看到前者的内容）→ 你追问或结束 → 对话自动保存，可在「决策历史」回看 → 「认知镜像」分析你的思维偏好

**当前运行状态：**
- 入口文件：`index.html`（单文件 SPA，无需构建）
- 开发服务器：Python HTTP Server，端口 `8765`（`python3 -m http.server 8765`）
- AI 服务：DeepSeek 官方 API + SiliconFlow API（双通道，按模型自动路由）
- 数据存储：浏览器 `localStorage`（无后端、无数据库）
- 本地资源：`assets/`（头像 × 4 + 背景图）

---

## 二、用户画像

**唯一用户：产品所有者本人**

使用场景：
- 做重要决策前，听不同思维框架的分析
- 遇到认知困境，想被挑战和反思
- 整理想法，对某个议题深度思考
- 通过「认知镜像」发现自己的思维盲区

---

## 三、信息架构（SPA 路由）

| 视图 ID | 入口 | 说明 |
|---------|------|------|
| `home` | 首页 | 品牌落地页，包含 Hero、功能卡片 |
| `roundtable` | 开启圆桌 / 开始思考 | 主功能：多 Agent 对话（fixed 全屏覆盖层） |
| `product` | 导航「产品」 | 产品特性介绍 |
| `methodology` | 导航「方法论」 | 四维认知框架说明 |
| `pricing` | 导航「定价」 | 定价页 |
| `history` | 首页「决策历史」卡片 | 历史对话回顾 |
| `mirror` | 首页「认知镜像」卡片 | 思维偏好分析 |
| `rooms` | 左侧边栏「场景会议室」按钮 | 场景会议室选择页 |

路由方式：`showView(viewId)` — 切换 DOM 显示/隐藏，不刷新页面。`roundtable` 视图为 fixed 全屏覆盖层，与其他视图独立叠加。

---

## 四、核心功能

### 4.1 圆桌讨论（主功能）

**发言流程：**
```
用户发消息（doSend）
  └→ Agent 1 读取 [用户消息] → SSE 流式输出
  └→ Agent 2 读取 [用户消息 + Agent1 回应] → SSE 流式输出
  └→ Agent 3 读取 [用户消息 + Agent1+2 回应] → SSE 流式输出
  └→ Agent 4 读取 [用户消息 + Agent1+2+3 回应] → SSE 流式输出
用户可继续追问，对话自动存入 localStorage
```

**流式输出细节：**
- RAF 批处理（`patchContent`）减少 DOM 写入，避免滚动卡顿
- 仅在距底部 160px 以内才自动滚动（`scrollBottom` nearBottom 检测）

**Agent 状态机：**

| 状态 | 视觉效果 |
|------|---------|
| `waiting`（排队中） | 完整色显示，彩色 badge "连接中…" |
| `streaming`（生成中） | 玻璃卡片高亮 + 彩色阴影 + "Streaming" badge + 打字光标 |
| `completed`（已完成） | 正常显示内容，无 badge |

**输出硬性规则（注入每次请求）：**
- 禁止 `**` 等 Markdown 符号，强调用"引号"代替
- 总字数严格不超过 200 字
- 禁止套话开头、总结段，直接给核心判断
- 用该人物语气直接作答，不加免责前缀

**@mention 精准提问：**
- 输入 `@` 弹出 Agent 选择下拉（玻璃面板）
- 键盘 ↑↓ 导航，Enter/Tab 确认，Esc 关闭
- 包含 `@名字` 时只有被 @ 的 Agent 参与本轮

---

### 4.2 Agent 配置

**主圆桌 4 个预设 Agent：**

| Agent | 专属色 | 核心思维框架 |
|-------|--------|-------------|
| 查理·芒格 | `#3B82F6` | 格栅思维、逆向思维（Jacobi）、25 种人类误判 |
| 阿尔弗雷德·阿德勒 | `#F97316` | 目的论、「阿德勒之问」、社会兴趣 |
| 安德烈·卡帕西 | `#10B981` | Software 2.0/3.0、Jagged Frontier、Vibe Coding |
| 埃隆·马斯克 | `#64748B` | 第一性原理、五步算法、白痴指数 |

**场景会议室专属顾问（ROOM_AGENTS，共 7 位）：**

| Agent ID | 名字 | 专属色 | role | 头像 |
|----------|------|--------|------|------|
| `munger` | 查理·芒格 | `#3B82F6` | 智慧博学家 | `assets/avatar-munger.jpg` |
| `adler` | 阿尔弗雷德·阿德勒 | `#F97316` | 心理重构者 | `assets/avatar-adler.jpg` |
| `karpathy` | 安德烈·卡帕西 | `#10B981` | 技术极客 | `assets/avatar-karpathy.jpg` |
| `musk` | 埃隆·马斯克 | `#64748B` | 效率狂人 | `assets/avatar-musk.jpg` |
| `jobs` | 史蒂夫·乔布斯 | `#1d1d1f` | 产品哲学家 | emoji 🍎 |
| `feynman` | 理查德·费曼 | `#D97706` | 科学思想家 | emoji 🔬 |
| `dalio` | 雷·达利奥 | `0F766E` | 原则主义者 | emoji 📊 |

**提示词设计原则：**
- 基于真实演讲/著作/采访，不捏造
- 已移除「防捏造规则」（会导致回复出现"X 未就此表态"等套话）
- 改为「回答原则」：用该人物的说话风格和思维框架直接回答，遇专业知识参考公开信息

**每个 Agent 可自定义：** 名字、头像、系统提示词、发言顺序（`rta_pa` 支持场景会议室顾问 ID）

---

### 4.3 模型与 API

**双 API 通道（`getAPIConf` 按模型 `provider` 字段自动路由）：**

| 模型 | 标签 | 通道 |
|------|------|------|
| `deepseek-chat` | DeepSeek V3 (官方) ⚡ 推荐 | DeepSeek 官方 `api.deepseek.com` |
| `deepseek-reasoner` | DeepSeek R1 (官方) 深度思考 | DeepSeek 官方 `api.deepseek.com` |
| `deepseek-ai/DeepSeek-V4-Flash` | DeepSeek V4 Flash | SiliconFlow |
| `deepseek-ai/DeepSeek-V3` | DeepSeek V3 | SiliconFlow |
| `deepseek-ai/DeepSeek-R1` | DeepSeek R1 | SiliconFlow |
| `Pro/deepseek-ai/DeepSeek-R1` | DeepSeek R1 (Pro) | SiliconFlow |
| `Qwen/Qwen3-235B-A22B` | Qwen3 235B | SiliconFlow |
| `Qwen/Qwen3-30B-A3B` | Qwen3 30B | SiliconFlow |
| `Qwen/Qwen3-8B` | Qwen3 8B | SiliconFlow |
| `Qwen/Qwen2.5-72B-Instruct` | Qwen2.5 72B | SiliconFlow |
| `Pro/moonshotai/Kimi-K2.6` | Kimi K2.6 (Pro) | SiliconFlow |
| `Pro/zai-org/GLM-5.1` | GLM-5.1 (Pro) | SiliconFlow |

**API Key 存储：** localStorage（`rta_cfg`），base64 混淆存储（btoa + encodeURIComponent），读取时自动解码，对用户透明。首次运行时 deepseekKey / siliconflowKey 均为空字符串（已移除硬编码默认 Key）。个人中心显示脱敏格式（前6位 + ●●●●●●●●●● + 后4位）。

---

### 4.4 设置面板

- DeepSeek API Key（推荐，优先）
- SiliconFlow API Key（选填，国内模型）
- Tavily API Key（可选，用于联网搜索优先通道）
- 模型选择（下拉）
- Agent 系统提示词编辑（每个 Agent 独立）
- 清除所有对话数据

### 4.5a 联网搜索

输入框工具栏提供联网搜索开关，启用后在每次发问前抓取相关资讯注入上下文。

降级链：**Tavily API**（需 tavilyKey）→ **Brave Search** → **DuckDuckGo**

所有通道均失败时返回空字符串，UI 显示 toast 提示，不中断对话流程。

### 4.5b 辩论模式

输入框工具栏提供辩论模式开关，启用后固定两轮：
- **第一轮**：各 Agent 依次发表初始立场
- **第二轮**：各 Agent 看到其他 Agent 的第一轮发言后进行交叉反驳

UI 以分隔线区分两轮，其余流式输出逻辑与普通模式相同。

### 4.5c 场景会议室

从主圆桌左侧边栏「场景会议室」按钮进入（`showView('rooms')`）。

**5 个预设房间：**

| 房间 ID | 名称 | 默认顾问阵容 |
|---------|------|-------------|
| `career` | 职业决策室 | 从 ROOM_AGENTS 选取 |
| `startup` | 创业沙盘室 | 从 ROOM_AGENTS 选取 |
| `philosophy` | 人生哲学室 | 从 ROOM_AGENTS 选取 |
| `tech` | 技术评审室 | 从 ROOM_AGENTS 选取 |
| `investment` | 投资思维室 | 从 ROOM_AGENTS 选取 |

- 支持创建自定义房间
- 每个房间有独立顾问阵容（从 ROOM_AGENTS 中选择）
- 房间内对话流程同主圆桌：用户发消息 → 房间内各顾问依次 SSE 流式输出
- 支持 @mention 单独提问某位顾问
- 支持联网搜索、辩论模式（与主圆桌相同）
- 会话数据存储于 `rtr_sessions`，每房间最多 20 条 session，每条 session 最多 100 条消息

---

### 4.6 决策历史（`view-history`）

从首页「决策历史」卡片进入，展示所有历史对话：

- 每条记录显示：对话标题（首条消息）、相对时间、参与顾问头像
- 展开各 Agent 回复摘要（前 ~100 字）
- 「复盘此决策」按钮：跳转到该对话继续追问
- 无记录时显示空状态 + 「开启第一次圆桌」CTA
- 返回首页按钮

---

### 4.7 认知镜像（`view-mirror`）

从首页「认知镜像」卡片进入，基于 localStorage 历史数据动态计算：

**左侧 — 思维倾向雷达：**
- 四轴 SVG 雷达图：逻辑分析（芒格）/ 技术执行（卡帕西）/ 效率优化（马斯克）/ 心理洞察（阿德勒）
- 各维度使用百分比（基于该 Agent 在历史中的参与次数）
- 底部数字展示各维度百分比

**左侧 — 思维盲点：**
- 自动检测使用率最低的顾问视角
- 显示盲点名称 + 使用率 + 行动建议（尝试从该角度提问）

**右侧 — 最活跃话题：**
- 从历史对话标题提取 tag 展示

**右侧 — 顾问偏好：**
- 4 位顾问各自使用率条形图 + 百分比

**右侧 — 本月洞察：**
- 根据数据生成个性化文字（最常用顾问 + 最低使用率顾问 + 行动建议）

---

### 4.8 演示 Modal（首页「观看演示」）

- 点击「观看演示」按钮打开全屏蒙层
- 显示虚构议题「AI 会取代我们吗？」
- 4 个 Agent 依次打字机效果呈现示例回复
- 完成后出现「开始我的圆桌」CTA 按钮
- × 关闭按钮 / 点击背景关闭

---

## 五、UI 设计系统

**设计风格：** Glassmorphism（玻璃拟态）+ Modern Corporate

**布局：**
- 顶部固定导航栏：Logo + 中部链接 + 右侧「开始思考」按钮
- 主内容区：`pt-32`（128px）居中，max-w 限制

**色彩：**
- 品牌色：`#6b38d4`（紫色）
- 背景：`linear-gradient(135deg, #f3ebff → #ede5f3)`
- 玻璃卡片：`background:rgba(255,255,255,0.X)` + `backdrop-filter:blur()`

**字体：**
- Plus Jakarta Sans — 品牌标题、按钮、正文
- JetBrains Mono — 状态 badge、代码标签

**本地资源（`assets/`）：**
- `bg-hero.jpg` — 首页 Hero 背景
- `avatar-munger.jpg` / `avatar-adler.jpg` / `avatar-karpathy.jpg` / `avatar-musk.jpg`

**外部 CDN 依赖：**
- Tailwind CSS（`cdn.tailwindcss.com`）
- Google Fonts — Plus Jakarta Sans / Inter / JetBrains Mono
- Material Symbols Outlined — 图标字体

---

## 六、数据存储（localStorage）

| Key | 内容 |
|-----|------|
| `rta_cfg` | API Keys（base64混淆存储）+ 当前模型 |
| `rta_sess` | 主圆桌所有 Session（Array，最多50条） |
| `rta_favs` | 收藏列表（Array，最多100条，存完整对象） |
| `rta_usage` | 用量统计（token 数，按模型分类） |
| `rta_pa` | Agent 自定义提示词（含场景会议室顾问） |
| `rtr_sessions` | 场景会议室 sessions（Object，key=roomId，每房间最多20条，每条最多100消息） |
| `rtr_custom_rooms` | 用户自定义房间（Array） |
| `rtr_last_sess` | 各房间上次打开的 sessionId（Object，key=roomId） |

---

## 七、边界与约束

| 场景 | 处理方式 |
|------|---------|
| API Key 未填写 | 发送按钮不可用，输入框显示提示 |
| 网络请求失败 | 显示红色错误条，支持关闭 |
| 流式中途关闭页面 | 已完成部分已写入 localStorage，未完成丢弃 |
| @mention 不存在的名字 | 忽略无效 @，退回全员回复 |
| 认知镜像无数据 | 所有维度显示 0%，提示「开始对话以积累数据」 |
| 决策历史无记录 | 空状态 UI + CTA |

---

## 八、技术架构

| 层 | 技术 |
|----|------|
| 入口 | 单文件 `index.html`（HTML + CSS + JS 全内联） |
| 样式框架 | Tailwind CSS CDN（Play CDN，运行时编译） |
| 交互框架 | 原生 Vanilla JS，无任何前端框架 |
| 路由 | 自实现 SPA：`showView()` 切换 DOM display |
| 数据持久化 | 浏览器 localStorage |
| AI 接口 | OpenAI 兼容 SSE（DeepSeek 官方 + SiliconFlow） |
| 流式渲染 | RAF 批处理（`patchContent`）+ nearBottom 防抖滚动 |
| 本地服务 | Python `http.server 8765` |

---

## 九、待迭代 / 已知限制

| 问题 | 影响 | 优先级 |
|------|------|--------|
| 字体/图标仍依赖 Google CDN | 网络差时加载慢 | 中 |
| Tailwind CDN 为运行时编译 | 初次加载有轻微延迟 | 低 |
| localStorage 无备份 | 清除浏览器数据会丢失历史 | 中 |
| 认知镜像话题 tag 无 NLP 提取 | 显示的是完整标题而非关键词 | 低 |
| API Key 仅 base64 混淆，非真正加密 | 本地安全性有限，仅防肉眼查看 | 低 |
| 场景会议室暂无数据导出 | 无法备份或迁移会议室历史 | 低 |

---

## 十、暂不做的功能

- 知识库 / 文件上传
- 多用户系统
- 云端同步
- 移动端适配（桌面优先）
- Agent 之间主动多轮辩论
- 语音输入/输出
