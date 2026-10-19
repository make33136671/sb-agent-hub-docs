# 阶段二：Bug 分析

> 日期：2026-08-20
> 说明：结合源码审查。严重度：🔴 严重 / 🟠 中 / 🟡 低。
> **最重要发现：客服并发计数 `CurrentCount` 只增不减，长期运行后会导致所有访客永远排队。**

---

## 1. 严重（🔴）— 影响核心可用性

### B1. 客服并发计数 `CurrentCount` 只增不减（最严重）
- **位置**：`AgentAllocationService.cs:112`（用 `CurrentCount < MaxConcurrent` 判断可用）、`KefuService.cs:194/347`（转接/接入排队时 `+1`）
- **问题**：全代码库中 `CurrentCount` 只在「转接转出」时 `-1`、在「接入排队」时 `+1`。**没有任何「会话正常结束/访客离线」时递减 `CurrentCount` 的逻辑**。`MessageCleanupJob`、访客离线（`VisitorOffline`）都未递减。
- **后果**：客服服务时间越长，`CurrentCount` 越接近 `MaxConcurrent`，直到全部客服被认为"满载"（`availableAgents` 为空）→ **所有访客进入无限排队**，即使客服实际空闲。
- **根因**：并发计数没有与真实进行中的会话状态（访客在线/关闭）绑定，而是独立累加字段。
- **影响面**：`AgentAllocationService`（分配）、`KefuService`（工作台）、排队功能。

### B2. OpenIM 群组会话在后台列表无法匹配（会话列表数据缺失）
- **位置**：`ConversationService.cs:120`（key = `{agentImUserId}_{elem.UserID}`）、`ChatAuthService.cs:347`（前端下发 `ConversationId = sg_{groupId}`）
- **问题**：统一群聊/广播模式下会话是 **Group**，OpenIM 会话 `elem.UserID` = **groupId**，而合并 key 用的是 `chatUser.ChatUserId`。两者不相等，`conversationMap.TryGetValue` 必然 miss。
- **后果**：群聊/广播模式下，后台「会话列表」的**未读数、最后一条消息、会话 ID 全部为空**。单聊模式（`si_`）正常。

### B3. 前端 XSS：用户可控内容拼入 innerHTML
- **位置**：`agent-chat/src/ChatPage.ts` `showQueueOverlay`（`:346`）、`showVisitorFormOverlay`（`:450-455,:463`）、`showRatingOverlay`（`:519`）
- **问题**：`config.formName`、`field.label`、`field.placeholder`、`field.options`（管理端配置的表单字段）以及访客昵称等被直接 `innerHTML` 拼接。管理端配置被篡改或含 `<script>`/`<img onerror>` 可执行脚本。
- **后果**：存储型/反射型 XSS。
- **同样需核查**：`openim-chat-ui-js` 的 `MessageListRenderer` 渲染消息内容时是否转义（文本消息 content 来自用户输入）。

---

## 2. 中（🟠）— 稳定性 / 正确性

### B4. `RefreshSessionAsync` 刷新即重分配客服 → 会话碎片化
- **位置**：`ChatAuthService.cs:446-545`
- **问题**：访客网络抖动、token 过期重连都会走 `RefreshSession`，而它每次都重新 `AllocateAgent`（`:486`）。只要新分配成功就更新 `AssignedAgentId`，**把访客转到不同客服**。
- **后果**：一次简单的刷新就可能换客服、重复体验；会话上下文丢失，服务碎片化。
- **建议**：原客服仍在线且未满载时应优先复用。

### B5. `RefreshSessionAsync` 重分配客服后未同步 `GroupId`
- **位置**：`ChatAuthService.cs:496-504`
- **问题**：更新了 `AssignedAgentId` 但**没有**更新/迁移 `ChatUser.GroupId`，新客服不在原群组里，前端仍用旧 `groupId`。
- **后果**：重分配后客服可能收不到该访客消息。

### B6. 管理员接管模式下无法建群 / 无法发消息
- **位置**：`AgentAllocationService.cs:62-70`
- **问题**：管理员接管分支返回的 `AgentAllocationResult` **未设置 `OpenImUserId`、`AgentOnline`、`WelcomeMessage`**。而 `ChatAuthService.VisitorAuthAsync` 用 `allocation.OpenImUserId` 判空来决定是否建群（`:249`）、发 session_init（`:258`）、发欢迎语（`:266`）。
- **后果**：管理员接管时 `OpenImUserId` 为空 → **不建群、不发消息**，客服端无法通过 OpenIM 收到访客消息，功能静默失效。

### B7. 广播模式群成员管理 O(N) 串行 HTTP
- **位置**：`KefuService.cs:146-160`
- **问题**：客服上线/下线时，对 owner 下**所有**活跃会话逐个 `InviteToGroupAsync`/`KickFromGroupAsync`，串行阻塞。
- **后果**：会话多时上线/下线操作极慢甚至超时；且 `Task.Run` 后台执行，失败仅记日志。

### B8. 会话并发判断依赖不稳定的 `OnlineStatus`
- **位置**：`AgentAllocationService.cs:89-93`
- **问题**：`Count(OnlineStatus == Online)` 依赖访客心跳 / `beforeunload` 通知。访客直接关页/断网时 `VisitorOffline` 可能未送达（`fetch keepalive` 不可靠），`OnlineStatus` 长期滞留 Online → 并发名额被占用 → 其他访客误排队。
- **后果**：排队误判、并发上限失真。

### B9. 已分配客服但 `GroupId` 已存在时，客服变更不会迁群
- **位置**：`ChatAuthService.cs:248-252`
- **问题**：`groupId` 非空时跳过建群。若复用已分配客服时换了客服（旧 GroupId 里的客服不在群），新客服收不到消息。
- **与 B5 同类问题**。

### B10. 转接操作非事务
- **位置**：`KefuService.cs:186-201`
- **问题**：`AssignedAgentId` 更新 + 双方 `CurrentCount` 增减用 `SetRaw`，中间任何失败都会造成数据不一致（访客归属与实际计数不符）。

---

## 3. 低（🟡）— 体验 / 健壮性

### B11. 满意度评分无法关联客服与会话
- **位置**：`agent-chat/src/ChatPage.ts:559-565`（硬编码 `agentId: 0`、`conversationId: ''`）
- **问题**：提交满意度时未带真实客服 ID 和会话 ID，后台无法统计到具体客服/会话的服务评分。
- **功能缺陷**：满意度报表失真。

### B12. 欢迎语 fire-and-forget 静默丢失
- **位置**：`ChatAuthService.cs:268`（`_ = SendGroupTextMessageAsync(...)`）
- **问题**：未 await，OpenIM 调用失败/慢则欢迎语丢失。

### B13. 对话列表 N+1 串行 HTTP（性能）
- **位置**：`ConversationService.cs:106-125`
- **问题**：每页对每个客服串行 `await GetSortedConversationsAsync`。客服数量多时分页接口延迟线性增长。

### B14. `KefuService.GetMyVisitors` 每页都查 OpenIM 在线状态
- **位置**：`KefuService.cs:79`
- **问题**：列表接口强依赖 OpenIM HTTP 在线查询，移除 OpenIM 后在线状态失真（阶段一已述），且拖慢接口。

### B15. 在线状态与排队状态存在竞态
- **位置**：`ChatAuthService.cs:214-234`（排队分支）
- **问题**：返回排队后客户端 `QueuePosition` 轮询 `connectAndLogin`，与后端 `AcceptFromQueue`/`MarkAssigned` 之间无幂等/原子保护，存在重复接入或接入后仍显示排队。

### B16. 代码中残留 TODO（未完成支付/签名验证）
- **位置**：`PaymentService.cs:124,149`（支付宝/微信支付接入）、`PaymentController.cs:59,79`（回调签名验证）
- **问题**：支付回调未做签名验证即处理，存在被伪造回调的风险（若上线需先补齐）。

---

## 4. 已知 OpenIM SDK 层面的 Bug（已被本项目 hack 修复）

这些是官方 `@openim/client-sdk@3.8.3` 的稳定性问题，本项目用脆弱的 monkey-patch 兜底（`openim-core-js/src/client/OpenIMClient.ts`）：

| SDK Bug | 本项目 hack | 位置 |
|---------|-------------|------|
| 同步后 `cachedHasReadAndMaxSeqs` 被置 undefined 崩溃 | 属性守卫 + 方法 monkey-patch | `:304-377` |
| `totalUnreadCount` 被置 undefined | 属性守卫 | `:336-339` |
| 同步完成/失败后内部缓存未初始化 | `fixSdkCacheAfterSync` | `:382-384` |
| fetch 请求头问题 | `patchFetchForOpenIM` 拦截 | `client/FetchInterceptor.ts` |

> **结论**：这些 hack 恰恰证明 OpenIM SDK 不可靠，是弃用 OpenIM 的最直接技术理由。

---

## 5. Bug 优先级建议

| 优先级 | Bug | 建议 |
|--------|-----|------|
| P0 | B1 并发计数只增不减 | 立即修复：会话结束/离线时递减；或改为实时统计进行中会话 |
| P0 | B2 群聊会话列表无法匹配 | 修复 key 逻辑或改按 groupId 匹配 |
| P0 | B3 XSS | 前端转义所有注入 innerHTML 的内容 |
| P1 | B4/B5/B9 重分配/迁群 | 重分配优先复用原客服；同步 GroupId |
| P1 | B6 管理员接管不建群 | 接管模式补充 OpenImUserId/建群 |
| P1 | B8 并发判断不稳定 | 用真实进行中会话数 + 心跳超时判定 |
| P2 | B7/B13/B14 性能 | 批量/并行化、减少 OpenIM 依赖 |
| P2 | B10 转接事务 | 加事务 |
| P2 | B11 满意度归属 | 前端传真实 agentId/conversationId |
| P3 | B12/B15/B16 | 健壮性补强 |

---

## 6. 下一步

进入阶段三：客服系统常见功能与实现成本评估。
