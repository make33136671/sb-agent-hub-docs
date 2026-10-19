# 阶段一：架构总览与 OpenIM 去耦合分析

> 调研日期：2026-08-20
> 结论先行：**OpenIM 可被替换而非彻底重建**。它是"消息通道 + 实时信令"基础设施层，客服业务中枢已自建。

---

## 1. 系统整体架构

```
[第三方网站] ──嵌入──> agent-widget-js / agent-widget-vue
        │                       │
        │  HTTP (访客认证/排队/表单)      │ WebSocket (实时消息)
        ▼                       ▼
  ┌─────────────────────────────────────────────┐
  │             agent-api (ASP.NET)             │
  │  ┌──────────────────────────────────────┐   │
  │  │ 自建客服业务中枢（不依赖 OpenIM）          │   │
  │  │ · QueueService        访客排队            │   │
  │  │ · AgentAllocationService 客服分配/滑落    │   │
  │  │ · KefuService         客服工作台/转接      │   │
  │  │ · ChatUser/Conversation/VisitorQueue 数据 │   │
  │  └──────────────────────────────────────┘   │
  │        │                                    │
  │        │ HTTP 封装（5 个 OpenIm*Service）      │
  │        ▼                                    │
  │  OpenIM 消息中台（注册/Token/群组/会话/发消息）  │
  │        │                                    │
  │        └───────────── HTTP ──> OpenIM Server │
  └─────────────────────────────────────────────┘
        │ HTTP (会话/消息数据)
        ▼
  agent-admin (Vue + Vben Admin)   <── WebSocket ── 客服端实时收发
  agent-chat  (全屏聊天页)         <── WebSocket ── 访客端实时收发
```

**关键技术栈（当前 agent-api）**：ASP.NET Core、FreeSql（MySQL）、JWT 双认证（JwtAdmin/JwtApi）、自研定时任务（`ScheduledJobBase`）、`ICacheService`（Memory/Redis/Null）、`BaseCrudService`、`[AutoDependencyInjection]` 自动注册。

> 注意：以上基础设施与 sb-framework（FreeSql / JWT / 后台任务 / 缓存 / BaseCrudService）**命名与约定高度重合**，说明 agent-api 很可能已按 sb 框架思路起步，迁移到 sb-framework NuGet 底座的成本低。

---

## 2. OpenIM 承担职责（后端视角）

5 个服务全部是 **HTTP REST 薄封装层**（走 `ImApiUrl`，非 gRPC/protobuf）：

| 服务 | 职责 | 关键端点 |
|------|------|----------|
| `OpenImAuthService` | 管理员/用户 Token、踢人 | `/auth/get_admin_token`、`/auth/get_user_token`、`/auth/force_logout` |
| `OpenImUserService` | 用户注册/更新/存在性/在线状态 | `/user/user_register`、`/user/get_users_online_status` |
| `OpenImMessageService` | 单聊/群聊文本、系统消息、已读回执、会话初始化 | `/msg/send_msg`、`/msg/mark_conversation_as_read` |
| `OpenImConversationService` | 会话列表、未读数、最新消息 | `/conversation/get_sorted_conversation_list` |
| `OpenImGroupService` | 群组创建/邀请/踢出/解散（统一群聊模式） | `/group/*` |

**OpenIM 承担的 4 类职责**：
1. **身份**：把客服/访客注册为 OpenIM userID，签发 IM Token 供 SDK 连 WebSocket。
2. **发消息**：文本、欢迎语、系统消息、已读回执（走 CustomMessage contentType=110）。
3. **会话（Conversation）**：会话列表、未读数、最新消息摘要。
4. **群组**：每个会话建一个 OpenIM WorkGroup（统一群聊模式 / 广播模式多客服监听）。

---

## 3. 受影响面（移除 OpenIM 后失效的部分）

| 消费者 | 使用点 | 移除后的后果 |
|--------|--------|--------------|
| `ChatAuthService.VisitorAuthAsync` | 注册访客+取 Token、建会话群组、发 session_init/欢迎语（`:172,:237,:249,:261,:268`） | **访客无法初始化聊天** |
| `KefuService.GetMyVisitors` | 用 OpenIM 在线状态覆盖 DB 状态（`:77-87`） | 在线状态失真 |
| `ConversationService.GetListAsync` | 按客服拉 OpenIM 会话补未读数/最后消息（`:99-125`） | **后台会话列表无未读数/最后消息** |
| `ChatAuthController` | 直接注入 `OpenImMessageService` 发已读/离线消息 | 回执/离线通知失效 |
| `AuthService` | 登录返回 `OpenImUserId` 字段（仅字段，`:234`） | 仅字段变化 |
| `AgentAllocationService` | 回填 `OpenImUserId` 结果字段 | 无实时依赖 |

**仍存活（不依赖 OpenIM）**：`QueueService`（排队）、`AgentAllocationService`（分配，仅注入 IFreeSql/Config/Quota/Queue）、`KefuService` 管理操作、FreeSql 持久化、JWT、全部定时任务、访客表单、配额、满意度、订单、支付等。

---

## 4. 前端 OpenIM 耦合（更深、更难缠）

### 4.1 依赖链

```
agent-chat ─┐
agent-widget-js ──> @saibotianlong/openim-core ──> @openim/client-sdk@3.8.3 (OpenIM 官方 WASM SDK)
agent-widget-vue ─┘
                  └─> @saibotianlong/chat-ui (纯 UI，相对可复用)
```

### 4.2 前端对 OpenIM 的核心依赖

- **WebSocket 实时收发**：`OpenIMClient.login(userID, token, apiAddr, wsAddr)` 连接 OpenIM 长连接。
- **历史消息**：`conversation.getHistory(conversationId, 100)` 拉取群聊/单聊历史。
- **已读回执**：`message:read-receipt` 事件 + API 回执。
- **事件模型**：`connection:*`、`message:*`、`conversation:changed`、`kicked:offline`、`token:expired` 等。
- **消息协议**：业务系统消息全部编码进 OpenIM `CustomMessage`（contentType=110），用 `data.type` 区分（`SystemMessage.ts`，与后端 `SystemMessageTypes.cs` 对齐）。**协议与 OpenIM 强耦合**。

### 4.3 强烈痛点证据：对 OpenIM SDK 的 monkey-patch

`openim-core-js/src/client/OpenIMClient.ts` 存在大量针对 `@openim/client-sdk@3.8.3` 的 hack：

- `ensureCacheGuard()` / `tryInstallCacheGuard()`（`:304-377`）：登录后轮询等待 `messageTrigger.cache` 就绪，对其 `cachedHasReadAndMaxSeqs` / `totalUnreadCount` 做**属性守卫 + 方法 monkey-patch**，防止被设为 `undefined`。
- `fixSdkCacheAfterSync()`（`:382-384`）：同步完成/失败后必须修复 SDK 内部缓存，否则后续会话/消息操作崩溃。
- 触发登出前先 `sdk.logout()`、`patchFetchForOpenIM`（拦截 fetch 修请求头）等。

**结论**：前端为弥补 OpenIM SDK 的稳定性缺陷付出了大量脆弱 hack 代码，升级/维护成本高，是"抛弃 OpenIM"最有力的业务理由。

---

## 5. 与 sb-framework 的对照

| 能力 | 当前 agent-api | sb-framework（新底座） | 兼容性 |
|------|----------------|------------------------|--------|
| ORM | FreeSql (MySQL) | FreeSql（`IFreeSqlConfigurator`，`sb_*` 前缀） | ✅ 高 |
| 认证 | JWT 双认证 | JWT 双认证 + **WebSocket query token 鉴权** | ✅ 高 |
| 定时任务 | `ScheduledJobBase` | `BaseBackgroundTask` + 熔断/心跳/日志 | ✅ 中 |
| 缓存 | `ICacheService` | `ICacheService`（三实现） | ✅ 高 |
| 通用 CRUD | `BaseCrudService` | `BaseCrudService` | ✅ 高 |
| 实时信令 | OpenIM WebSocket | **框架模块 `Sb.Framework.Chat.Realtime`**（SignalR + WS query token 鉴权） | 🔶 需开发框架模块 |
| 客服/聊天 | OpenIM 消息通道 | `sb-framework-chat` = **Telegram SDK**，非客服聊天；实时聊天用 `Sb.Framework.Chat.Realtime` | 🔶 需开发框架模块 |

**关键判断**：
- `sb-framework`（后端基础设施 NuGet 底座）可直接消费，升级路径清晰。
- `sb-framework-chat` 是 **Telegram Bot/MTProto 统一消息 SDK**，定位是"通道/消息 SDK"，**不含排队、坐席流转、实时信令/WebSocket 推送**，**不能**替代 OpenIM 的实时客服通道。
- **替代 OpenIM 的实时通道，沉淀为 sb 框架可复用模块**：扩展 `sb-framework-chat` 家族，新增 `Sb.Framework.Chat.Realtime`（SignalR 实时通讯底层），详见 `05-框架级OpenIM替代方案-模块设计.md`。该模块是通用实时通讯地基，任何项目可复用；客服业务（排队/分配等）留在项目层。

---

## 6. 去 OpenIM 的替代方案

| 方案 | 说明 | 优点 | 缺点 | 成本 |
|------|------|------|------|------|
| **A. 框架模块 `Sb.Framework.Chat.Realtime`**（已定） | 实时通讯沉淀为 sb 框架模块（SignalR Hub + 消息持久化 + 在线 + 回执 + 会话），项目复用 | 框架级沉淀、可控、无外部依赖、多项目复用、与 sb 框架一体 | 需自研消息分发/历史/在线/回执 | 框架 15-25 + 项目 20-30 人日 |
| **B. 引入其它可托管 IM**（弃用） | 换 IM 底座 | 复用现成 IM 能力 | 又引入第三方耦合，SDK 稳定性不可控 | 高 |
| **C. 双轨过渡**（已否决） | 先抽象 IM 接口（`IImTransport`）渐进切换 | 平滑、低风险 | 保留双套、过渡期双维护；不符合"彻底"要求 | 中 |

**已定路径**：**A（框架模块 `Sb.Framework.Chat.Realtime`）**，**彻底切换（D15），否决双轨**。后端业务直接依赖 Realtime 模块服务，删除 OpenIM 全部（无 `IImTransport` 抽象/垫片/开关）。详见 `09-彻底去OpenIM切换方案.md`。

---

## 7. 下一步

进入阶段二：深入分析已知与潜在 Bug。
