# Akaunting 应用/模块更新流程分析

## 1. 四个阶段成功/失败触发事件

### 1.1 下载阶段 (Download)

**成功触发事件：**
- `UpdateDownloaded` - 在 [Update.php#L125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L125-L125) 或 [Updates.php#L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L179-L179) 中触发
- 参数：`alias`, `new`, `old`

**失败触发事件：**
- `UpdateFailed` - 在 [Update.php#L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L131-L131) 中触发
- 参数：`alias`, `new`, `old`, `step='Download'`, `message`

### 1.2 解压阶段 (Unzip)

**成功触发事件：**
- `UpdateUnzipped` - 在 [Update.php#L146](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L146-L146) 或 [Updates.php#L215](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L215-L215) 中触发
- 参数：`alias`, `new`, `old`

**失败触发事件：**
- `UpdateFailed` - 在 [Update.php#L152](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L152-L152) 中触发
- 参数：`alias`, `new`, `old`, `step='Unzip'`, `message`

### 1.3 复制阶段 (Copy Files)

**成功触发事件：**
- `UpdateCopied` - 在 [Update.php#L167](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L167-L167) 或 [Updates.php#L251](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L251-L251) 中触发
- 参数：`alias`, `new`, `old`

**失败触发事件：**
- `UpdateFailed` - 在 [Update.php#L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L173-L173) 中触发
- 参数：`alias`, `new`, `old`, `step='Copy Files'`, `message`

### 1.4 完成阶段 (Finish)

**成功触发事件：**
- `UpdateFinished` - 在 [FinishUpdate.php#L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L66-L66) 中触发（注意：此事件在 `update:finish` 命令内触发，而非在主 Update 命令中）
- 参数：`alias`, `new`, `old`

**失败触发事件：**
- `UpdateFailed` - 在 [Update.php#L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192) 中触发
- 参数：`alias`, `new`, `old`, `step='Finish'`, `message`

### 全局失败处理
所有 `UpdateFailed` 事件都会被 [SendNotificationOnFailure](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Update/SendNotificationOnFailure.php) 监听器捕获，发送邮件/Slack通知。

---

## 2. UpdateFailed 事件定位问题

### 2.1 事件数据结构

[UpdateFailed](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Install/UpdateFailed.php) 事件携带以下字段：

| 字段 | 说明 |
|------|------|
| `alias` | 模块别名（如 'core'、'invoice' 等） |
| `old` | 当前安装的版本号 |
| `new` | 目标更新版本号 |
| `step` | 失败阶段标识 |
| `message` | 异常错误信息 |

### 2.2 Stage 字段可能的值

| Stage 值 | 对应阶段 | 触发位置 |
|----------|----------|----------|
| `Version` | 版本检查阶段 | [Update.php#L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L64-L64) |
| `Download` | 下载阶段 | [Update.php#L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L131-L131) |
| `Unzip` | 解压阶段 | [Update.php#L152](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L152-L152) |
| `Copy Files` | 文件复制阶段 | [Update.php#L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L173-L173) |
| `Finish` | 完成阶段 | [Update.php#L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192) |

### 2.3 定位问题的方法

1. **通过 stage 快速定位阶段**：检查 `step` 字段确定失败发生在哪个阶段
2. **结合 message 分析具体原因**：
   - Download 阶段：网络问题、权限不足、临时目录不可写
   - Unzip 阶段：ZIP 文件损坏、磁盘空间不足
   - Copy Files 阶段：目录权限不足、文件被占用
   - Finish 阶段：数据库迁移失败、监听器执行错误
3. **查看版本号**：`old` 和 `new` 字段可确认更新跨度，判断是否有特殊迁移逻辑

---

## 3. FinishUpdate Job 与 update:finish 命令的使用时机

### 3.1 FinishUpdate Job

**位置**：[Jobs/Install/FinishUpdate.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php)

**触发时机**：
- 在 CLI 更新流程中被 [Update.php#L186](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L186-L186) 调用
- 在 Web 更新流程中被 [Updates.php#L285](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L285-L285) 调用

**核心职责**：
1. 检查模块是否存在
2. 根据监听器类型决定需要更新哪些公司：
   - `none`：没有迁移监听器，只需清缓存
   - `one`：只需更新当前触发更新的公司
   - `all`：需要更新所有安装了该模块的公司（实现了 `ShouldUpdateAllCompanies` 接口）
3. 为每个公司调用 `update:finish` 命令

### 3.2 update:finish 命令

**位置**：[Console/Commands/FinishUpdate.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php)

**触发时机**：
- 被 [Jobs/Install/FinishUpdate.php#L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L54-L54) 通过 `Console::run()` 调用

**核心职责**：
1. 清除缓存
2. 验证文件版本是否正确（防止文件复制失败但继续执行）
3. 切换到目标公司上下文
4. 触发 `UpdateFinished` 事件，执行所有迁移监听器

### 3.3 两者关系与设计意图

```
FinishUpdate Job (协调层)
    ↓ 决定哪些公司需要更新
    ↓ 逐个调用
update:finish 命令 (执行层)
    ↓ 验证文件完整性
    ↓ 切换公司上下文
    ↓ 触发 UpdateFinished 事件
        ↓ 执行 CreateModuleUpdatedHistory
        ↓ 执行 UpdateExtraModules
        ↓ 执行各版本迁移监听器 (Version300, Version303, ...)
```

**设计意图**：
- Job 负责多公司协调，命令负责单公司实际更新
- 通过子进程执行命令，隔离每个公司的更新环境
- 每个公司独立失败不影响其他公司（但当前实现中如果一个失败会抛出异常终止全部）

---

## 4. UpdateFinished 监听器顺序对迁移副作用的影响

### 4.1 监听器注册顺序

在 [Event.php#L15-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L15-L36) 中定义：

```php
\App\Events\Install\UpdateFinished::class => [
    \App\Listeners\Update\CreateModuleUpdatedHistory::class,      // 1. 创建更新历史
    \App\Listeners\Module\UpdateExtraModules::class,              // 2. 更新依赖模块
    \App\Listeners\Update\V30\Version300::class,                  // 3. v3.0.0 迁移
    \App\Listeners\Update\V30\Version303::class,                  // 4. v3.0.3 迁移
    \App\Listeners\Update\V30\Version304::class,                  // 5. v3.0.4 迁移
    // ... 更多版本监听器
],
```

### 4.2 顺序影响分析

#### 问题 1：CreateModuleUpdatedHistory 排在首位

**潜在问题**：如果后续监听器执行失败，更新历史已经被创建，导致数据库记录显示更新成功但实际未完成。

#### 问题 2：UpdateExtraModules 排在版本迁移之前

**代码参考**：[UpdateExtraModules.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/UpdateExtraModules.php)

**副作用**：
- 如果依赖模块更新失败，该监听器返回 `false`，**终止事件传播**（第 64 行）
- 后续的版本迁移监听器将不会执行
- 但当前模块的文件已经复制完成，版本号已更新，导致代码与数据库不匹配

#### 问题 3：版本监听器按版本号顺序执行

**执行逻辑**：每个版本监听器通过 `Versions::shouldUpdate()` 判断是否需要执行
- 参见 [Versions.php#L282-L292](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Versions.php#L282-L292)
- 条件：`listener_version > old_version && listener_version <= new_version`

**顺序重要性**：
- 必须严格按版本号升序排列
- 如果顺序错误（如 Version307 排在 Version300 之前），可能导致：
  - 高版本迁移依赖低版本迁移的数据结构，但低版本还未执行
  - 数据迁移不完整或失败

### 4.3 监听器失败的影响范围

| 监听器类型 | 失败后果 | 回滚机制 |
|-----------|----------|----------|
| CreateModuleUpdatedHistory | 无更新历史记录 | 无，可事后手动补充 |
| UpdateExtraModules | 依赖模块未更新，但当前模块继续 | 无，需要手动更新依赖 |
| 版本迁移监听器 | 数据库结构/数据不完整 | 无，需要手动修复或重新执行 |

---

## 5. 复制成功但 Finish 失败的状态不一致场景

### 5.1 场景描述

**时间线**：

1. **阶段 1-3 成功**：
   - Download：下载成功，触发 `UpdateDownloaded`
   - Unzip：解压成功，触发 `UpdateUnzipped`
   - Copy Files：文件复制成功，触发 `UpdateCopied`
   - 此时：新版本代码文件已覆盖旧版本

2. **阶段 4 失败**：
   - FinishUpdate Job 执行 `update:finish` 命令
   - `update:finish` 中某个监听器执行失败（如数据库迁移出错）
   - 抛出异常，触发 `UpdateFailed`（step='Finish'）
   - 此时：文件是新版本，但数据库可能停留在旧版本

### 5.2 具体不一致表现

#### 场景 A：版本号检查通过但迁移失败

在 [FinishUpdate.php#L43-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L43-L48) 验证了文件版本与请求版本一致，这在复制成功后必然通过。

但后续在触发 `UpdateFinished` 事件时：
- 假设 `Version300` 监听器的 `updateDatabase()` 方法执行 `Artisan::call('migrate')` 失败
- 或者 `updateCompanies()` 中某个数据转换失败
- 异常抛出后，`UpdateFinished` 事件传播中断

**结果**：
- 代码文件：v3.0.0
- 数据库迁移：部分完成或完全未执行
- 模块版本记录：未创建（如果 CreateModuleUpdatedHistory 在失败前已执行则会有记录）
- 系统可能处于半更新状态，导致运行时错误

#### 场景 B：多公司更新部分失败

在 [FinishUpdate.php#L49-L59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L49-L59) 中循环更新所有公司：

```
公司 1 更新 → 成功
公司 2 更新 → 失败（抛出异常）
公司 3 及以后 → 未执行
```

**结果**：
- 公司 1：完全更新（代码 + 数据库）
- 公司 2：代码新，数据库旧（不一致）
- 公司 3+：代码新，数据库旧（不一致）

### 5.3 风险分析

| 风险点 | 影响 | 表现 |
|--------|------|------|
| 代码与数据库版本不匹配 | 高 | 页面报错、功能异常、数据丢失 |
| 部分公司更新部分未更新 | 中 | 多租户系统中不同公司体验不一致 |
| 更新历史记录缺失 | 低 | 审计困难，无法追溯更新状态 |
| 缓存未清理 | 中 | 旧配置/视图缓存导致异常 |

### 5.4 恢复建议

1. **识别失败阶段**：通过 `UpdateFailed` 事件的 `step` 和 `message` 定位
2. **检查文件完整性**：确认代码文件是否为目标版本
3. **数据库状态检查**：检查 migrations 表，确认哪些迁移已执行
4. **手动修复**：
   - 如果是迁移失败：手动执行失败的 SQL 或重新运行 `php artisan migrate`
   - 如果是数据迁移失败：根据监听器逻辑手动修复数据
5. **重试 Finish 阶段**：`php artisan update:finish {alias} {company_id} {new_version} {old_version}`

---

## 总结

更新流程采用"事件驱动 + 分阶段"设计，各阶段通过事件解耦，但缺乏完整的事务回滚机制。理解各阶段的事件触发、监听器顺序和失败处理逻辑，对于排查更新失败和处理状态不一致至关重要。
