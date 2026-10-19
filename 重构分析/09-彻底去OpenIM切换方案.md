# 阶段九：彻底去 OpenIM 切换方案（一次整体切换）

> 日期：2026-08-20
> 决策：**不存在过渡期/双轨/兼容垫片**。后端 + 前端彻底一次切，删除 OpenIM 全部。
> 撤销此前 M1 的"双轨过渡"设计（IImTransport 抽象 + OpenImTransport + ImTransport 配置开关 + RealtimeTransport 兼容垫片 + AgentConfig.OpenIM）。

---

## 1. 彻底切换的原则

- **无过渡期**：业务不面向任何"为兼容 OpenIM"的抽象，直接依赖 Realtime 模块服务。
- **无兼容垫片**：不保留伪 Token、不保留 OpenIM 语义方法（CreateGroup/GetSortedConversations/伪 IM Token）。
- **删除 OpenIM 全部**：服务/模型/配置/测试/前端 SDK 依赖。
- **前端一次切**：openim-core-js 底层从 `@openim/client-sdk` 换成 SignalR client，三个前端入口（agent-chat/widget-js/widget-vue）随之切换。

---

## 2. 后端删除清单

| 项 | 位置 | 处理 |
|----|------|------|
| 5 个 OpenIm*Service | `SbAgentHub.Services/OpenIm{User,Auth,Message,Conversation,Group}Service.cs` | 删除 |
| OpenImTransport | `SbAgentHub.Services/OpenImTransport.cs` | 删除 |
| IImTransport 抽象 | `SbAgentHub.Services/IImTransport.cs` | 删除（业务直接用 Realtime） |
| RealtimeTransport | `SbAgentHub.Services/RealtimeTransport.cs` | 删除（业务直接用 Realtime 服务） |
| ImTransport 配置开关 | `appsettings.json` + `ServiceSetup` | 删除 |
| OpenIM 配置 | `AgentConfig.OpenIM`/`AgentOptions`/`appsettings "OpenIM"` 节 | 删除 |
| OpenImModels | `SbAgentHub.Models/Dto/OpenImModels.cs` | 删除 |
| OpenIm 集成测试 | `SbAgentHub.Tests.Integration/OpenImTests` | 删除 |
| 前端 OpenIM SDK 依赖 | `openim-core-js` 的 `@openim/client-sdk` | 替换为 SignalR client |

---

## 3. 后端业务改造（直接依赖 Realtime 模块）

| 现状（OpenIM 调用） | 改为（Realtime 模块服务） |
|---------------------|---------------------------|
| 注册访客+取 IM Token | **访客改签 JWT**（Realtime Hub 用 query token 鉴权） |
| CreateSessionGroupAsync（OpenIM 群） | `IConversationService.CreateAsync`（group 会话） |
| SendGroupSessionInitMessage / 欢迎语 | `IMessageService.SendSystemAsync` / `SendAsync` |
| GetUsersOnlineStatusAsync | `IPresenceService.IsOnline` |
| GetSortedConversationsAsync | `IConversationService.GetConversationsAsync` |
| InviteToGroup / KickFromGroup | `IConversationService.AddMember/RemoveMember` |
| 已读回执（OpenIM） | `IReadStateService.MarkReadAsync` |
| 会话列表未读 | `IConversationService`（含 UnreadCount） |

**服务依赖注入**：`ChatAuthService`/`KefuService`/`ConversationService`/`ChatAuthController` 注入 `IMessageService`/`IConversationService`/`IPresenceService`/`IReadStateService`。

---

## 4. 访客 JWT 认证（Realtime Hub [Authorize] 前提）

- Realtime Hub 用 `[Authorize]` + sb-framework WS `access_token` query 鉴权（`UseQueryAccessTokenForWebSocket`）。
- 访客（匿名 ChatUser）需持有 JWT 才能连 Hub。
- **方案**：新增 `JwtVisitor` 认证节（独立 Key/Issuer/Audience），`ChatAuthService` 为访客签发含 `ChatUserId`/`PlatformId` claims 的 JWT；新增 `JwtVisitor` Bearer 方案（无 Member 状态校验，仅验签 + 有效），Realtime Hub 的 `[Authorize]` 指向该方案。
- 前端 Realtime client 用该 JWT 连 `/hubs/realtime?access_token=...`。

---

## 5. 前端切换

- `openim-core-js`：
  - 移除 `@openim/client-sdk` 依赖。
  - 新增 SignalR client（`@microsoft/signalr`），实现与 Realtime Hub 的协议映射。
  - **保持对外 API/事件名不变**（`login(userId,token,apiAddr)`、`on('message:received')`、`getHistory`、`conversation:*` 等），使 agent-chat/widget 三个入口改动最小。
  - 移除 `cacheGuard`/`fixSdkCacheAfterSync`/`patchFetchForOpenIM` 等 SDK hack。
- `agent-chat`/`agent-widget-js`/`agent-widget-vue`：仅底层换，接口不变。
- 修复 B3（XSS）。
- `agent-admin`：客服工作台的在线/会话/消息改为 Realtime。

---

## 6. 执行顺序（无过渡期，整体切换）

1. ✅ **后端删 OpenIM 基础设施**：删除 5 个 OpenIm*Service、OpenImTransport、IImTransport、RealtimeTransport、OpenImModels、OpenImTests、ImTransport 开关、OpenIM 配置（AgentConfig.OpenIM + appsettings OpenIM 节）。
2. ✅ **后端业务重构**：ChatAuthService/KefuService/ConversationService/ChatAuthController 直接依赖 Realtime 模块服务（IMessageService/IConversationService/IPresenceService/IReadStateService）；访客改签 JWT（复用 JwtAdmin 验签 + ChatUserId/role=visitor claims），Realtime Hub [Authorize] 默认 scheme 可验访客；GetAgentImToken/RefreshImToken 改为签 JWT。
3. ✅ **后端编译 + 启动验证**：全解决方案 0 错误；完整启动 /health 200；访客认证接口返回正常业务响应（链路通）；`chat_rt_*` 5 表已建（Program.cs 显式 SyncStructure，因模块命名非 SB.Feature.* 不被 feature 体系自动建表）。
4. ⏳ **前端 openim-core-js 换 SignalR client**（仅换底层；移除 cacheGuard 等 SDK hack；**目录/包名/类名全部按新命名**，见第 9 节）。
5. ⏳ **三个前端入口 + admin 切换**（**所有 import 路径、模块名按新命名**）。
6. ⏳ **后端命名清理**（OpenImIdGenerator/OpenImUserId/GetImToken/SdkImConfig 等 → 新命名，见第 9 节；含 DB 列迁移）。
7. ⏳ **整体回归 + 移除 OpenIM 部署/文档**（含测试库补业务数据后端到端验证：真实平台/客服下 访客认证→JWT→建会话→收发消息；清理 docs/openim/ 与 dist 残留）。

> 注：**D20 撤销原"保留旧 DTO/字段名"约定**。所有 OpenIM 命名彻底重命名（见下方第 9 节命名映射表），不保留任何 OpenIM 语义的标识符/注释/目录/包名。DTO 的 Token 语义为访客 JWT、ConversationId/GroupId 改为 chat_rt_conversation.id、原 SdkImConfig 改为 Realtime Hub 地址。

---

## 7. 风险与待确认

- **访客 JWT 是新认证面**：需与现有 JwtAdmin/JwtApi 隔离，避免越权（访客仅能访问自己的会话）。
- **Realtime 模块能力完备性**：在线/已读/会话/未读/媒体已具备；如缺能力（如"强制踢连接"）需补 Realtime 模块。
- **前端协议**：Realtime Hub 的 SendMessage/MarkRead/JoinConversation/GetHistory 与前端事件需逐一映射。
- **整体切换是一次性发布**：切换瞬间客服短暂不可用（前后端必须同版本部署）。
- **数据库**：存量 OpenIM 历史不迁移（D10），chat_rt_* 为全新数据。

---

## 8. 文档索引（追加）

| 文档 | 内容 |
|------|------|
| `09-彻底去OpenIM切换方案.md` | 本文：彻底切换原则、删除清单、业务改造、访客 JWT、前端切换 |

---

## 9. 命名映射表（D20：彻底重命名）

> 原则：命名前缀对齐框架模块 `Sb.Framework.Chat.Realtime`，使用 `Realtime`/`Chat` 语义；不保留任何 `OpenIm`/`Im` 命名。

### 9.1 后端源码

| 旧名 | 新名 | 位置 |
|------|------|------|
| `OpenImIdGenerator.cs` | `RealtimeIdGenerator.cs` | `SbAgentHub.Core/Util/` |
| `OpenImIdGenerator` | `RealtimeIdGenerator` | 类名 |
| `Member.OpenImUserId` | `Member.RealtimeUserId` | `SbAgentHub.Models/Member.cs`（含 DB 列 `OpenImUserId` → `RealtimeUserId`，需迁移脚本） |
| `AgentAllocationResult.OpenImUserId` | `RealtimeUserId` | `SbAgentHub.Models/Dto/AgentAllocationResult.cs` |
| `VisitorAuthResponse.ImToken` | `Token` | `ChatAuthDto.cs` |
| `VisitorAuthResponse.ImTokenExpire` | `TokenExpire` | `ChatAuthDto.cs` |
| `VisitorAuthResponse.AgentImUserId` | `AgentUserId` | `ChatAuthDto.cs` |
| `RefreshImTokenRequest` | `RefreshTokenRequest` | `ChatAuthDto.cs` |
| `ImTokenResponse` | `TokenResponse` | `ChatAuthDto.cs` |
| `GetImTokenFormRequest` | `GetTokenFormRequest` | `ChatAuthDto.cs` |
| `GetImTokenSdkResponse` | `GetTokenSdkResponse` | `ChatAuthDto.cs` |
| `SdkImConfig` | `SdkRealtimeConfig` | `ChatAuthDto.cs` |
| `SdkImConfig.ApiUrl` / `WsUrl` | `HubUrl` / `WsUrl`（注释改为 Realtime Hub） | `ChatAuthDto.cs` |
| `VisitorOfflineRequest.AgentImUserId` | `AgentUserId` | `ChatAuthDto.cs` |
| `MessageReadRequest.AgentImUserId` | `AgentUserId` | `ChatAuthDto.cs` |
| `RefreshSessionResponse.AgentImUserId` | `AgentUserId` | `ChatAuthDto.cs` |
| `GetImToken`（action） | `GetToken` | `ChatAuthController.cs` |
| `RefreshImToken`（action） | `RefreshToken` | `ChatAuthController.cs` |
| `GetAgentImToken`（action） | `GetAgentToken` | `AuthController.cs` |
| `SdkAgentInfo.ConversationId` 注释 | 注释改为 `chat_rt_conversation.id` | `ChatAuthDto.cs` |
| `SystemMessageTypes.cs` OpenIM 注释 | 注释改为 Realtime 语义 | `SbAgentHub.Models/Dto/` |
| `ChatUser.ChatUserId` 注释 | 注释改为"Realtime 用户ID" | `ChatUser.cs` |
| `ChatUser.GroupId` 注释 | 注释改为"会话群组 ID" | `ChatUser.cs` |
| `VisitorQueue.ChatUserId` 注释 | 注释改为"Realtime 聊天用户ID" | `VisitorQueue.cs` |
| `Program.cs` "OpenIM 等" 注释 | 注释改为"客服系统专属配置" | `Program.cs` |
| `dist/*.xml` | 重新编译生成 | `agent-api/dist/` |
| `docs/openim/` | 删除（OpenIM 集成文档，已废弃） | `agent-api/docs/` |
| `SQL/migration_phase2_quota.sql` 注释 | 注释改为 Realtime 用户ID | `agent-api/SQL/` |

### 9.2 前端（切换时执行）

| 旧名 | 新名 | 位置 |
|------|------|------|
| `openim-core-js/` | `realtime-core-js/` | 项目根目录 |
| `openim-chat-ui-js/` | `chat-ui-js/` | 项目根目录 |
| `@saibotianlong/openim-core` | `@saibotianlong/realtime-core` | npm 包名 |
| `OpenIMClient` | `RealtimeClient` | 核心类 |
| `createOpenIMCore` | `createRealtimeCore` | 工厂函数 |
| `getCore` / `hasCore` / `destroyCore` / `updateCoreImAddrs` | `updateCoreRealtimeAddrs` | agent-admin 模块 |
| `useOpenIM` | `useRealtime` | agent-widget-vue composable |
| `modules/openim/` | `modules/realtime/` | agent-admin |
| `isSystemMessage` / `MessageType` 等导出 | 保留（无 OpenIM 语义） | — |

### 9.3 DB 迁移

```sql
-- SB_Member.OpenImUserId → RealtimeUserId
ALTER TABLE SB_Member CHANGE OpenImUserId RealtimeUserId VARCHAR(100) NULL;
```
