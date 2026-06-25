# Session 信息与对话历史存储分析

> 问题：每次新建 session 后，session 的信息、对话历史消息这些放到哪里的？

---

## 结论

**对话历史消息存在 Agent Server 后端**，前端只是展示层。前端的 localStorage 只保存轻量的 UI 偏好和元数据。

---

## 存储分层

### 1. Agent Server 后端（持久化 — 真正的数据归宿）

| 数据 | 说明 | 获取方式 |
|------|------|---------|
| Conversation 记录 | id、title、created_at、updated_at、execution_status、metrics、agent、workspace | REST: `GET /api/conversations` 分页搜索 |
| 对话历史 Events | 所有 OpenHandsEvent（用户消息、Agent 动作、执行结果、错误等） | REST: `GET /api/conversations/{id}/events/search`（分页）<br>WebSocket: 实时推送新事件 |
| 工作区文件 | conversation 的工作目录（代码、PLAN.md 等） | REST: 文件读取 API |

Cloud 模式额外：App API 持久化 event history（`/api/v1/conversation/{id}/events/search`），可在 sandbox 销毁后存活。

---

### 2. 浏览器 localStorage（持久化 — 只存 UI 偏好，不存消息）

| Key | 内容 | 代码位置 |
|-----|------|---------|
| `conversation-state-{conversationId}` | 每个对话的 UI 状态：选中的 tab、code/plan 模式、草稿消息、文件视图偏好 | `src/utils/conversation-local-storage.ts` |
| `openhands-agent-server-conversation-metadata` | 所有对话的元数据 map：关联仓库、分支、git provider、workspace 路径、LLM profile | `src/api/conversation-metadata-store.ts` |
| `openhands-backends` | 已注册的后端列表（host、apiKey、kind） | `src/api/backend-registry/storage.ts` |
| `openhands-active-backend` | 当前激活的后端 | `src/api/backend-registry/storage.ts` |
| `pinned-conversations` | 置顶对话 ID 列表 | `src/stores/pinned-conversations-store.ts` |

---

### 3. 浏览器内存（刷新即丢）

| 位置 | 内容 | 代码位置 |
|------|------|---------|
| Zustand `useEventStore` | 当前打开对话的所有事件（消息列表），切换对话时清空 | `src/stores/use-event-store.ts` |
| Zustand `useConversationStore` | 当前 UI 瞬态状态（面板开关、待发送附件等） | `src/stores/conversation-store.ts` |
| React Query Cache | conversation-history 缓存（gcTime=30分钟） | `src/hooks/query/use-conversation-history.ts` |

---

## 新建 Session 流程中数据的去向

```
用户点击"新建对话"
       │
       ▼
1. 前端调用 AgentServerConversationService.createConversation()
       │
       ▼
2. Agent Server 后端：
   • 生成 conversation_id (UUID)
   • 创建工作目录
   • 持久化 conversation 记录
   • 启动 Agent runtime
       │
       ▼
3. 前端收到响应后：
   • localStorage 写入 metadata（仓库、workspace、LLM profile）
   • 导航到 /conversations/{id}
       │
       ▼
4. 对话页面挂载：
   • REST 请求后端加载最近 50 条历史事件 → 存入内存 useEventStore
   • WebSocket 连接（resend_mode='since'）→ 实时接收新事件
       │
       ▼
5. 用户发消息：
   • REST POST /api/conversations/{id}/events 发送到后端
   • 后端持久化该事件
   • Agent 响应通过 WebSocket 实时推回前端
   • 前端 useEventStore 追加事件（仅内存缓存）
```

---

## 重新打开已有 Session

1. 从 localStorage 恢复 UI 偏好（tab、mode）
2. REST 查后端获取 conversation 信息
3. REST `searchEvents(TIMESTAMP_DESC, limit=50)` 获取最近历史
4. WebSocket 连接 `resend_mode='since'` + 最新事件时间戳（不重复接收）
5. 用户上滑 → `useLoadOlderEvents` 分页加载更早事件

---

## 一句话总结

前端 = 缓存 + UI 偏好；后端 = 对话内容的唯一持久化源。
