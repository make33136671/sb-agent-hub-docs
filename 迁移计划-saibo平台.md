# 迁移计划：sb-agent-hub → saibo 平台

> 创建：2026-08-20
> 目标：将项目从 `D:\projects\sb-agent-hub` 迁移到 `D:\projects\saibo\apps\agenthub\`，遵循 saibo 平台命名规范，框架引用改为本地路径，框架级功能沉淀到 sb 框架。

## 一、路径映射

### 源目录 → 目标目录

| 当前路径 | 目标路径 | 说明 |
|----------|----------|------|
| `D:\projects\sb-agent-hub\agent-api` | `D:\projects\saibo\apps\agenthub\agenthub-api` | 后端 API |
| `D:\projects\sb-agent-hub\agent-admin` | `D:\projects\saibo\apps\agenthub\agenthub-admin` | 管理后台 |
| `D:\projects\sb-agent-hub\agent-chat` | `D:\projects\saibo\apps\agenthub\agenthub-chat` | 独立聊天页 |
| `D:\projects\sb-agent-hub\agent-widget-js` | `D:\projects\saibo\apps\agenthub\agenthub-widget-js` | JS 挂件 |
| `D:\projects\sb-agent-hub\agent-widget-vue` | `D:\projects\saibo\apps\agenthub\agenthub-widget-vue` | Vue 挂件 |
| `D:\projects\sb-agent-hub\realtime-core-js` | `D:\projects\saibo\sb\sb-framework-chat-js\` 或留在 agenthub | 前端实时核心库（框架级，见下方决策） |
| `D:\projects\sb-agent-hub\realtime-chat-ui-js` | `D:\projects\saibo\sb\sb-framework-chat-js\` 或留在 agenthub | 聊天 UI 组件库（框架级，见下方决策） |
| `D:\projects\sb-agent-hub\docs` | `D:\projects\saibo\apps\agenthub\agenthub-docs` | 项目文档 |

### sb 框架路径（已存在）

| 模块 | 路径 |
|------|------|
| sb-framework（核心框架） | `D:\projects\saibo\sb\sb-framework\` |
| sb-framework-chat（Chat/Realtime 模块） | `D:\projects\saibo\sb\sb-framework-chat\` |
| sb-framework-admin（前端基座） | `D:\projects\saibo\sb\sb-framework-admin\` |
| sb-framework-docs（框架文档） | `D:\projects\saibo\sb\sb-framework-docs\` |

## 二、命名规范

遵循 saibo 平台约定（参考 `D:\projects\saibo\AGENTS.md` 和 `05-新项目开发指南.md`）：

### 后端命名

| 项 | 当前 | 目标 | 说明 |
|----|------|------|------|
| 命名空间 | `SbAgentHub.*` | `Sb.Apps.AgentHub.*` | 遵循 `Sb.Apps.<App>.*` 规范 |
| 解决方案 | `SbAgentHub.sln` | `sb-apps-agenthub.slnx` | 新 .slnx 格式，遵循 tgvault 先例 |
| 表前缀 | `SB_` | `sb_agenthub_` | 遵循 `sb_<app>_` 规范 |
| 实体表名 | `SB_ChatUser` | `sb_agenthub_chat_user` | 全小写下划线 |
| C# 类名 | `ChatUser` | `ChatUser`（不变） | PascalCase 保持 |

### 前端命名

| 项 | 当前 | 目标 |
|----|------|------|
| 包名 | `@saibotianlong/realtime-core` | 视归属决定（见下方决策） |
| 目录名 | `agent-api` 等 | `agenthub-api` 等 |

### 目标目录结构

```
D:\projects\saibo\apps\agenthub\
├── agenthub-api/              # 后端 API（Sb.Apps.AgentHub.*）
├── agenthub-admin/            # 管理后台（fork from sb-framework-admin）
├── agenthub-chat/             # 独立聊天页
├── agenthub-widget-js/        # JS 挂件
├── agenthub-widget-vue/       # Vue 挂件
└── agenthub-docs/             # 项目文档
```

## 三、框架引用方式变更

### 当前方式：NuGet 包引用

```xml
<!-- 当前：通过 NuGet BaGet 服务器引用 -->
<PackageReference Include="SB.Framework.WebApi.NoLicense" Version="0.4.14" />
<PackageReference Include="SB.Framework.Core.NoLicense" Version="0.4.6" />
<PackageReference Include="SB.Framework.Models.NoLicense" Version="0.4.7" />
<PackageReference Include="Sb.Framework.Chat.Realtime" Version="0.1.0-preview.1" />
```

### 目标方式：本地 ProjectReference

```xml
<!-- 目标：直接引用本地 sb 框架源码 -->
<ProjectReference Include="D:\projects\saibo\sb\sb-framework\SB.Framework.WebApi\SB.Framework.WebApi.csproj" />
<ProjectReference Include="D:\projects\saibo\sb\sb-framework\SB.Framework.Core\SB.Framework.Core.csproj" />
<ProjectReference Include="D:\projects\saibo\sb\sb-framework\SB.Framework.Models\SB.Framework.Models.csproj" />
<ProjectReference Include="D:\projects\saibo\sb\sb-framework-chat\Sb.Framework.Chat.Realtime\Sb.Framework.Chat.Realtime.csproj" />
```

### 引用映射表

| 当前 NuGet 包 | 目标本地 .csproj 路径 |
|---------------|----------------------|
| `SB.Framework.WebApi.NoLicense` | `D:\projects\saibo\sb\sb-framework\SB.Framework.WebApi\SB.Framework.WebApi.csproj` |
| `SB.Framework.Core.NoLicense` | `D:\projects\saibo\sb\sb-framework\SB.Framework.Core\SB.Framework.Core.csproj` |
| `SB.Framework.Models.NoLicense` | `D:\projects\saibo\sb\sb-framework\SB.Framework.Models\SB.Framework.Models.csproj` |
| `Sb.Framework.Chat.Realtime` | `D:\projects\saibo\sb\sb-framework-chat\Sb.Framework.Chat.Realtime\Sb.Framework.Chat.Realtime.csproj` |

### NuGet.config 变更

保留 NuGet.config（用于第三方依赖），但框架级依赖改为 ProjectReference，不再需要发布到 BaGet。

## 四、框架级功能沉淀

### 需要沉淀到 sb 框架的功能

| 功能 | 当前位置 | 目标位置 | 说明 |
|------|----------|----------|------|
| SignalR Hub + 消息持久化 | 已在 `sb-framework-chat` | 已完成 | `Sb.Framework.Chat.Realtime` |
| realtime-core-js（前端 SignalR 客户端） | `sb-agent-hub\realtime-core-js` | `D:\projects\saibo\sb\sb-framework-chat-js\`（新建） | 前端实时通讯 SDK，应作为框架级共享库 |
| realtime-chat-ui-js（聊天 UI 组件库） | `sb-agent-hub\realtime-chat-ui-js` | `D:\projects\saibo\sb\sb-framework-chat-js\`（新建） | 可复用的聊天 UI 组件，应作为框架级共享库 |
| 认证/Token/会话管理逻辑 | realtime-core-js 内 | 随 realtime-core-js 一起迁移 | |

### 新建 sb-framework-chat-js 仓库

```
D:\projects\saibo\sb\sb-framework-chat-js\
├── packages/
│   ├── realtime-core/       # @saibotianlong/realtime-core（原 realtime-core-js）
│   └── chat-ui/             # @saibotianlong/realtime-chat-ui（原 realtime-chat-ui-js）
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

> 决策：前端实时核心库和聊天 UI 组件库是框架级共享能力（不绑定 agenthub 业务），应沉淀到 sb 框架家族，供其他产品线复用。

### 不迁移到框架的部分（业务逻辑留在 agenthub）

- 访客排队/分配逻辑（`VisitorQueue` / `SessionOrchestrator`）
- 客服工作台业务（`KefuService` / `ChatAccessValidator`）
- 平台/域名校验（`ChatAccessValidator`）
- 配额/订单/支付（`QuotaService` / `OrderService`）
- 自动回复规则（`AutoReply`）
- 满意度评价（`SatisfactionRating`）

## 五、迁移步骤

### Phase G：目录迁移与命名变更

| 步骤 | 内容 | 依赖 |
|------|------|------|
| G1 | 创建 `D:\projects\saibo\apps\agenthub\` 目录 | 无 |
| G2 | 复制 `agent-api` → `agenthub-api`，重命名命名空间 `SbAgentHub.*` → `Sb.Apps.AgentHub.*` | G1 |
| G3 | 复制 `agent-admin` → `agenthub-admin` | G1 |
| G4 | 复制 `agent-chat` → `agenthub-chat` | G1 |
| G5 | 复制 `agent-widget-js` → `agenthub-widget-js` | G1 |
| G6 | 复制 `agent-widget-vue` → `agenthub-widget-vue` | G1 |
| G7 | 复制 `docs` → `agenthub-docs` | G1 |
| G8 | 新建 `sb-framework-chat-js` 仓库，迁移 realtime-core-js 和 realtime-chat-ui-js | G1 |

### Phase H：框架引用切换

| 步骤 | 内容 | 依赖 |
|------|------|------|
| H1 | agenthub-api 所有 .csproj 改 PackageReference → ProjectReference | G2 |
| H2 | NuGet.config 保留但移除框架包依赖 | H1 |
| H3 | 验证编译通过（0 error） | H1 |
| H4 | 前端项目包引用从 `file:../realtime-core-js` 改为 `file:../../../sb/sb-framework-chat-js/packages/realtime-core` | G8 |

### Phase I：框架功能沉淀

| 步骤 | 内容 | 依赖 |
|------|------|------|
| I1 | realtime-core-js 移入 `sb-framework-chat-js/packages/realtime-core` | G8 |
| I2 | realtime-chat-ui-js 移入 `sb-framework-chat-js/packages/chat-ui` | G8 |
| I3 | 包名改为 `@sb/realtime-core` 和 `@sb/chat-ui`（或保持 `@saibotianlong/*`） | I1, I2 |
| I4 | 更新所有前端项目的依赖路径 | I3 |

### Phase J：数据库迁移

| 步骤 | 内容 | 依赖 |
|------|------|------|
| J1 | 生成表名重命名脚本：`SB_*` → `sb_agenthub_*` | G2 |
| J2 | 更新所有实体 `[Table(Name = "...")]` 特性 | G2 |
| J3 | 更新 SQL 迁移脚本中的表名 | G2 |
| J4 | 验证迁移脚本可在测试环境执行 | J1-J3 |

### Phase K：验证与清理

| 步骤 | 内容 | 依赖 |
|------|------|------|
| K1 | 后端完整编译 + 启动验证 | H3 |
| K2 | 前端各项目构建验证 | H4 |
| K3 | SignalR 连接 + 消息收发验证 | K1, K2 |
| K4 | 删除旧 `D:\projects\sb-agent-hub` 目录 | K1-K3 全部通过 |

## 六、风险与注意事项

1. **命名空间重构范围大**：`SbAgentHub.*` → `Sb.Apps.AgentHub.*` 涉及所有 .cs 文件的 using 和 namespace 声明
2. **表前缀变更**：`SB_` → `sb_agenthub_` 需要数据库迁移脚本，且影响所有实体特性
3. **ProjectReference 路径**：本地路径依赖，CI/CD 需要确保 sb 框架源码同时检出
4. **前端包路径**：从 `file:../realtime-core-js` 改为跨目录引用，需验证 pnpm 解析
5. **sb-framework-chat-js 是新仓库**：需要初始化 pnpm workspace 和发布配置

## 七、关键本地路径速查

```
# saibo 工作区根目录
D:\projects\saibo\

# 业务应用
D:\projects\saibo\apps\agenthub\               # agenthub 产品线
D:\projects\saibo\apps\agenthub\agenthub-api\   # 后端
D:\projects\saibo\apps\agenthub\agenthub-admin\ # 管理后台
D:\projects\saibo\apps\agenthub\agenthub-docs\  # 文档

# sb 框架
D:\projects\saibo\sb\sb-framework\              # 核心框架
D:\projects\saibo\sb\sb-framework-chat\         # Chat/Realtime 模块
D:\projects\saibo\sb\sb-framework-chat-js\      # 前端 JS 库（新建）
D:\projects\saibo\sb\sb-framework-admin\        # 前端基座

# 当前项目（迁移源）
D:\projects\sb-agent-hub\                       # 待迁移，K4 完成后删除

# saibo 平台规范文档
D:\projects\saibo\AGENTS.md                     # 目录放置约定
D:\projects\saibo\sb\sb-framework-docs\05-新项目开发指南.md  # 新项目开发指南
```
