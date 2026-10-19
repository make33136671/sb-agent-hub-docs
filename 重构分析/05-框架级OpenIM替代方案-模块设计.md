# 阶段五：框架级 OpenIM 替代方案 — Sb.Framework.Chat.Realtime 模块设计

> 日期：2026-08-20
> 定位：把替代 OpenIM 的**实时通讯能力沉淀为 sb 框架可复用模块**，项目复用；遵循 sb-framework-chat 家族命名规范。
> 归属：扩展 `sb-framework-chat` 家族（独立子包 `Sb.Framework.Chat.Realtime`，与 Telegram 三件套平级）。

---

## 1. 设计原则（框架级 vs 项目级）

| 层级 | 内容 | 归属 |
|------|------|------|
| **框架级（本模块）** | 通用实时通讯底层：连接/鉴权、消息收发、持久化、会话、在线状态、已读回执、媒体 | `Sb.Framework.Chat.Realtime` |
| **项目级（agent-api）** | 客服业务语义：访客/客服映射、排队、分配、转接、广播策略、欢迎语、自动回复、满意度 | 客服项目自己实现 |

> 框架只提供"通讯地基"，不掺入客服业务。任何项目（不止客服）都能复用该实时通讯模块。

---

## 2. 命名与归属（对齐家族规范）

- **仓库**：并入 `D:\projects\saibo\sb\sb-framework-chat`
- **工程/程序集**：`Sb.Framework.Chat.Realtime`
- **命名空间**：`Sb.Framework.Chat.Realtime.*`（对齐 `Sb.Framework.Chat.*` 大小写）
- **包名**：`Sb.Framework.Chat.Realtime`（+ `.NoLicense` 变体，对齐 `Sb.Framework.Chat`）
- **配置节**：`Chat:Realtime`（已定 D8，与 Telegram SDK 共用 `Chat` 根节，避免冲突）
- **装配方法**：`AddSbRealtime()`（对齐 `AddTelegramChat()`）
- **表前缀**：`chat_rt_*`（已定 D9，表名如 `chat_rt_message`；与 chat 家族 `chat_*` 统一归属）
- **目标框架**：`net10.0`；依赖 `SB.Framework.Core`/`SB.Framework.Models` + `Microsoft.AspNetCore.App`（FrameworkReference，SignalR 需要）

---

## 3. 模块结构

```
Sb.Framework.Chat.Realtime/
├── Extensions/  SbRealtimeExtensions.cs        # AddSbRealtime() 装配
├── Hub/         RealtimeHub.cs                  # SignalR Hub（客户端↔服务器）
├── Presence/    IPresenceService.cs / PresenceService.cs / PresenceChange.cs
├── Messages/    RealtimeMessage.cs / IMessageService.cs / MessageService.cs
│                / MessageDeliveryService.cs / SystemMessage.cs
├── Conversations/ IConversationService.cs / ConversationService.cs / ConversationModels.cs
├── ReadState/   IReadStateService.cs / ReadStateService.cs
├── Media/       IMediaService.cs / MediaService.cs / IMediaStorageProvider.cs
├── Storage/     RealtimeModuleInitializer.cs / RtConversation.cs / RtConversationMember.cs
│                / RtMessage.cs / RtReadState.cs / RtMedia.cs
├── Auth/        IRealtimeAuthProvider.cs / IRealtimeUserProvider.cs
├── Events/      IRealtimeEventSink.cs           # 项目业务钩子（敏感词/审计/自动回复）
├── Options/     RealtimeOptions.cs
└── Sb.Framework.Chat.Realtime.csproj
```

---

## 4. 核心抽象（供项目插桩）

| 接口 | 职责 | 项目如何实现 |
|------|------|--------------|
| `IRealtimeUserProvider` | 用户 id → 资料（昵称/头像） | 客服项目映射 `Member`/`ChatUser` |
| `IRealtimeAuthProvider` | 连接鉴权/准入校验 | 复用 sb 框架 WS query token + 项目校验 |
| `IRealtimeEventSink` | 消息/连接事件业务钩子 | 客服项目注入：敏感词过滤、审计、自动回复 |
| `IMediaStorageProvider` | 媒体落盘（可插拔） | 默认本地磁盘，可换 OSS/对象存储 |

> 全部提供默认实现（`Default*`），项目按需覆盖。

---

## 5. 通讯模型（对齐/映射 OpenIM）

### 5.1 实时通道
- **传输**：SignalR（ASP.NET Core Hub），非 OpenIM WASM SDK。
- **鉴权**：**直接复用 sb-framework 现有 `UseQueryAccessTokenForWebSocket`**（`JwtSetup.cs`），浏览器 WS 握手带 `?access_token=` 即可，无需新造。
- **连接映射**：一个 JWT 用户 = 一个实时用户 id（客服项目把 `ChatUserId`/`Member.OpenImUserId` 映射为实时用户 id）。

### 5.2 Hub 方法（客户端 → 服务器）

| 方法 | 语义 | 对应 OpenIM |
|------|------|-------------|
| `SendMessage(conversationId, contentType, payload)` | 发消息（落库 + 投递给会话成员） | `/msg/send_msg` |
| `SendSystemMessage(conversationId, type, data)` | 发系统消息（欢迎语/session_init/离线等） | CustomMessage contentType=110 |
| `MarkRead(conversationId, lastMsgId)` | 已读回执 | `/msg/mark_conversation_as_read` |
| `JoinConversation/LeaveConversation(id)` | 加入/离开会话（广播模式多客服） | `/group/*` |
| `GetHistory(conversationId, beforeId, count)` | 历史消息分页 | `/conversation/get_owner_conversation` + 历史 |

### 5.3 服务器 → 客户端事件

| 事件 | 对应 OpenIM `CbEvents` |
|------|------------------------|
| `OnMessage(RealtimeMessage)` | `OnRecvNewMessage` |
| `OnReadReceipt` | `OnRecvC2CReadReceipt` |
| `OnPresenceChanged` | `get_users_online_status` |
| `OnConversationChanged` | `OnConversationChanged` |
| `OnConnected/OnReconnected` | `OnConnectSuccess` |
| `OnKicked` | `OnKickedOffline` |

### 5.4 能力映射总表

| OpenIM 能力 | 现状（5 个服务） | Realtime 模块 |
|-------------|------------------|---------------|
| 注册用户 | `OpenImUserService.RegisterUserAsync` | 无需注册，`IRealtimeUserProvider` 映射项目用户 |
| 取 Token | `OpenImAuthService.GetUserTokenAsync` | 复用 sb 框架 JWT |
| 发消息 | `OpenImMessageService` | `MessageService` + `RtMessage` 落库 |
| 已读回执 | `mark_conversation_as_read` | `ReadStateService` + `RtReadState` |
| 会话列表/未读 | `OpenImConversationService` | `ConversationService` + `RtConversation`（未读聚合） |
| 群组(广播) | `OpenImGroupService` | `RtConversation` + `RtConversationMember` |
| 在线状态 | `get_users_online_status` | `PresenceService`（心跳驱动） |

---

## 6. 存储实体（`chat_rt_*` 表，随 `DatabaseSeeder` 建表）

| 实体 | 表 | 关键字段 | 作用 |
|------|----|---------|------|
| `RtConversation` | `chat_rt_conversation` | Id、OwnerId、Title、Type(1:1/group/system)、Ext | 会话主体 |
| `RtConversationMember` | `chat_rt_conversation_member` | ConversationId、UserId、Role、JoinedTime、LastReadMsgId | 会话成员 + 未读定位（支撑广播模式，D11） |
| `RtMessage` | `chat_rt_message` | Id、ConversationId、SenderId、ContentType、ContentJson、MediaIds、Seq、Timestamp、IsSystem、IsWithdrawn | 消息持久化（含系统消息/撤回） |
| `RtReadState` | `chat_rt_read_state` | ConversationId、UserId、LastReadSeq | 已读游标 |
| `RtMedia` | `chat_rt_media` | Id、OwnerId、Url、Type、Size、Mime | 媒体元数据 |

> 复用 chat 家族"收到即入库、撤回仅标记"的防撤回语义。

---

## 7. 消息协议（系统消息）

沿用现状的做法但不再依赖 OpenIM CustomMessage：系统消息用 `RtMessage.ContentType` 标记（如 `sys.session_init`、`sys.welcome`、`sys.transfer`、`sys.offline`），`ContentJson` 存结构化负载。前端 `SystemMessage.ts` 的类型判断逻辑保留，但数据来源从 OpenIM 改为自有事件。

---

## 8. 装配与配置（对齐家族）

```csharp
// Program.cs —— 客服项目
builder.Services.AddSbRealtime(configuration, o =>
{
    o.WebSocketPath = "/hubs/realtime";
    o.HeartbeatInterval = TimeSpan.FromSeconds(30);
    o.MessageRetentionDays = 90;      // 对齐 MessageCleanupJob
});
```

```jsonc
// appsettings.json
"Chat": {
  "Realtime": {
    "WebSocketPath": "/hubs/realtime",
    "HeartbeatIntervalSeconds": 30,
    "MessageRetentionDays": 90
  }
}
```

`RealtimeModuleInitializer : IModuleInitializer` 声明 `rt_*` 5 张表（`ModuleName="Realtime"`, `ModuleKey="realtime"`），随框架 `DatabaseSeeder` 建表，不声明菜单（与 chat SDK 一致）。

---

## 9. 客服项目落地映射（agent-api 改造点）

| 现状（OpenIM 调用） | 改造后（Realtime 模块） |
|---------------------|--------------------------|
| `ChatAuthService.VisitorAuthAsync` 注册访客 + 取 IM Token | 发 JWT（含 ChatUserId）；WS 握手带 `access_token` |
| `CreateSessionGroupAsync` 建 OpenIM 群 | 调 `ConversationService.Create(group)` 建 `RtConversation` |
| `SendGroupSessionInitMessageAsync` / 欢迎语 | 调 `MessageService.SendSystemMessage`（项目侧，或经 EventSink 自动） |
| `KefuService.GetMyVisitors` 查 OpenIM 在线 | 查 `PresenceService.IsOnline` |
| `ConversationService.GetListAsync` 拉 OpenIM 会话 | 查 `ConversationService`（含未读，修复 B2/B13） |
| 广播模式成员管理（`OpenImGroupService`） | `RtConversationMember` 增删 |
| `ChatAuthController.MessageRead/VisitorOffline` | `ReadStateService` / `PresenceService` |

**业务中枢不变**：`QueueService`、`AgentAllocationService`、`KefuService` 排队/分配/转接逻辑原样保留，仅把对 OpenIM 的读写换成 Realtime 模块。

---

## 10. 前端契约（保持 openim-core 对外 API 稳定）

`openim-core-js` 内部把 `@openim/client-sdk` 换成 SignalR client，**对外 API/事件名不变**（`login/userID/token`、`on('message:received')`、`getHistory` 等）。三个前端入口（agent-chat / widget-js / widget-vue）因此几乎零改动；`openim-chat-ui-js` 纯 UI 完全复用。

- `login` → SignalR 连接（JWT from query）+ `JoinConversation`
- 消息事件 → Hub `OnMessage`
- `getHistory` → Hub `GetHistory`
- 顺带在适配层修复 XSS（阶段二 B3）

---

## 11. 实施步骤（框架模块 + 项目改造）

### 阶段 F（框架模块，✅ 已完成）
1. ✅ 建 `Sb.Framework.Chat.Realtime` 工程 + 接入 `sb-framework-chat.sln`（23 源文件 + 9 单测）。
2. ✅ 实现存储实体 + `RealtimeModuleInitializer`（`chat_rt_*` 5 表）。
3. ✅ 实现 `RealtimeHub` + Presence / Message / Conversation / ReadState / Media 服务。
4. ✅ 实现 `AddSbRealtime`/`UseSbRealtime` 装配 + `RealtimeOptions`。
5. ✅ 单元测试：Presence / SystemMessageType（9 通过）。
6. ✅ **发布 NuGet**：`Sb.Framework.Chat.Realtime@0.1.0-preview.1` 已 push 到 `nuget.saibo.club`。

> 工程源码在 `D:\projects\saibo\sb\sb-framework-chat`；家族 `sb-framework-chat.sln` 0 错 0 警。

### 阶段 P（客服项目，agent-api）— 已按彻底切换（D15）执行
1. ✅ `AddSbRealtime` 装配 + `chat_rt_*` 表显式同步（模块非 `SB.Feature.*` 不被框架自动建表）。
2. ⏳ `IRealtimeUserProvider`/`IRealtimeAuthProvider`/`IRealtimeEventSink` 接入（客服映射/敏感词/审计/自动回复）。
3. ✅ 后端业务直接依赖 Realtime 模块服务（删 OpenIm 调用点，非"替换 5 个服务"，因已彻底删除）。
4. ⏳ P0 Bug：B1 并发计数、B3 XSS（B2 已在会话列表重构中修复）。
5. ⏳ 前端适配层切换（openim-core-js → SignalR client）。

> 详见 `08`（M0 底座）与 `09`（彻底切换）。

---

## 12. 决策记录（已全部定案）

1. **SignalR**（D5）：自动重连/协商/多传输，与 ASP.NET 一体，复用 sb 框架 WS 鉴权。
2. **消息持久化介质**（D7）：FreeSql（`chat_rt_*` 表）；量大再考虑内存 + 异步落库。
3. **配置节**（D8）：`Chat:Realtime`，沿用 `Chat` 根节与 Telegram SDK 并列。
4. **表前缀**（D9）：`chat_rt_*`，与 chat 家族统一。
5. **广播模式**（D11）：保留，`chat_rt_conversation_member` 支撑多客服监听。
6. **历史迁移**（D10）：存量 OpenIM 历史不迁移，仅保留会话映射。

> 完整决策记录见 `06-决策记录.md`（D1~D14 全部已定）。

---

## 13. 文档索引（追加）

| 文档 | 内容 |
|------|------|
| `05-框架级OpenIM替代方案-模块设计.md` | 本文：框架模块设计、命名、映射、实施 |

> 下一阶段：待确认决策点后，可在 `sb-framework-chat` 仓库脚手架 `Sb.Framework.Chat.Realtime` 工程骨架。
