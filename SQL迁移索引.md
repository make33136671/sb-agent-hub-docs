# SQL 迁移脚本索引

> 所有迁移脚本位于 `agenthub-api/SQL/`，按时间顺序执行。
> 每个迁移在 `sb_agenthub_migration_version` 表注册版本号。
> 迁移后表名已统一为 `sb_agenthub_*`（snake_case），见 `migration_g_rename_saibo.sql`。

## 执行顺序

| 序号 | 文件 | 版本号 | 说明 |
|------|------|--------|------|
| 1 | `seed.sql` | — | 初始种子数据（菜单、管理员、系统设置） |
| 2 | `system_setting_seed.sql` | — | 系统设置种子数据 |
| 3 | `insert_admin.sql` | — | 初始管理员账号 |
| 4 | `full_menu_seed.sql` | — | 完整菜单种子数据 |
| 5 | `add-profile-menu.sql` | — | 个人资料菜单 |
| 6 | `migration_quota_system.sql` | — | 配额系统建表（QuotaConfig/MemberQuota/QuotaUsage/QuotaPackage） |
| 7 | `migration_phase2_quota.sql` | — | 二期配额补充表 |
| 8 | `migration_phase3_features.sql` | — | 三期功能表（AutoReply/VisitorForm/SatisfactionRating/VisitorQueue/VisitorFormSubmission） |
| 9 | `migration_phase3_menus.sql` | — | 三期菜单种子 |
| 10 | `migration_phase4_order.sql` | — | 订单系统建表 |
| 11 | `migration_phase4_menus.sql` | — | 四期菜单种子 |
| 12 | `migration_login_log.sql` | — | 登录日志表 |
| 13 | `migration_email_system.sql` | — | 邮件系统表（EmailConfig/Template/VerificationCode/Log） |
| 14 | `migration_background_task.sql` | — | 后台任务日志表 |
| 15 | `migration_member_kyc.sql` | — | 会员实名认证（KYC）表 |
| 16 | `migration_user_to_member.sql` | — | 旧 User 表迁移到 Member 表（历史记录，保留旧表名） |
| 17 | `migration_menu_quota_violation.sql` | — | 菜单配额违规记录表 |
| 18 | `migration_realtime_rename.sql` | 20260820_01 | OpenIM → Realtime 列重命名（OpenImUserId → RealtimeUserId 等） |
| 19 | `migration_phase_e_indexes.sql` | 20260820_01 | Phase E: 索引优化 + FileRecord 默认值修复 + 迁移版本表创建 |
| 20 | `migration_phase_e_comments.sql` | 20260820_02 | Phase E: 全部表和字段添加 COMMENT 注释（含历史表） |
| 21 | `migration_g_rename_saibo.sql` | G-saibo-rename | **Phase J1: saibo 平台表名迁移 `SB_*` → `sb_agenthub_*`（33 张表）** |

> **表名说明**（Phase J 后）：
> - 序号 1-15、17、19-21 的脚本已更新为新表名 `sb_agenthub_*`（适用于全新部署和重跑）
> - 序号 16 与根目录 `migration.sql` 为历史重命名记录，保留旧表名不动
> - 已有数据库升级路径：先停应用 → 执行 `migration_g_rename_saibo.sql` → 启动新版本应用
>
> 另有 `agenthub-api/migration.sql`（早期拼音→英文重命名，已归档）和 `agenthub-api/Sb.Apps.AgentHub.WebApi/SQL/Menu_ErrorLog.sql`（菜单错误日志修复）。

## 迁移版本表

```sql
CREATE TABLE `sb_agenthub_migration_version` (
    `Id` INT AUTO_INCREMENT PRIMARY KEY,
    `Version` VARCHAR(50) NOT NULL COMMENT '迁移版本号',
    `Description` VARCHAR(200) COMMENT '迁移描述',
    `AppliedAt` TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '执行时间',
    UNIQUE INDEX `ix_migrationversion_version` (`Version`)
);
```

## 新增迁移脚本要求

1. 文件名：`migration_{phase}_{feature}.sql`
2. 放在 `agenthub-api/SQL/` 目录
3. 末尾注册版本号：
   ```sql
   INSERT INTO `sb_agenthub_migration_version` (`Version`, `Description`) VALUES
       ('YYYYMMDD_NN', '描述');
   ```
4. 所有 `ALTER TABLE` / `CREATE TABLE` 必须包含 COMMENT（见 [开发规范.md](开发规范.md)）
5. 更新本索引文档
