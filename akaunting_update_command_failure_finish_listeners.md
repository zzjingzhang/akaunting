# Akaunting 应用/模块更新流程分析（第三轮修正版）

> 所有结论均附源码行号引用，可点击跳转验证。

---

## 1. UpdateFailed 事件构造函数参数与调用一致性

### 1.1 构造函数参数顺序

[UpdateFailed.php#L27-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Install/UpdateFailed.php#L27-L34)：

```php
public function __construct($alias, $old, $new, $step, $message = '')
{
    $this->alias = $alias;
    $this->old   = $old;     // 第 2 个参数：形参语义是"当前已安装版本"
    $this->new   = $new;     // 第 3 个参数：形参语义是"目标版本"
    $this->step  = $step;
    $this->message = $message;
}
```

### 1.2 Update.php 中的实际传参与值

[Update.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php) 中 `handle()` 的关键赋值顺序：

| 行号 | 操作 | 产生的结果 |
|---|---|---|
| [第 59 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L59-L59) | `$this->new = $this->getNewVersion()` | `$this->new` = 目标版本（或 `false`） |
| [第 64 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L64-L64) | `event(new UpdateFailed($this->alias, $this->new, $this->old, 'Version', ...))` | 仅在 `$this->new === false` 时到达 |
| [第 69 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L69-L69) | `$this->old = $this->getOldVersion()` | `$this->old` 才被赋值为当前版本 |

因此，所有 5 处 `event(new UpdateFailed(...))` 调用均将 `$this->new` 作为第 2 参数、`$this->old` 作为第 3 参数传入，与构造函数形参顺序 `($alias, $old, $new, ...)` 相反：

| 调用 | 代码 | `$this->new` 实际值 | `$this->old` 实际值 | 事件 `old` 字段 | 事件 `new` 字段 |
|---|---|---|---|---|---|
| Version 失败 ([L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L64-L64)) | `UpdateFailed($this->alias, $this->new, $this->old, 'Version', ...)` | `false`（第 59 行条件为 false 才进来） | `null`（第 69 行还没执行） | `false` | `null` |
| Download 失败 ([L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L131-L131)) | `UpdateFailed($this->alias, $this->new, $this->old, 'Download', ...)` | 目标版本 | 当前版本 | 目标版本 | 当前版本 |
| Unzip 失败 ([L152](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L152-L152)) | `UpdateFailed($this->alias, $this->new, $this->old, 'Unzip', ...)` | 目标版本 | 当前版本 | 目标版本 | 当前版本 |
| Copy Files 失败 ([L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L173-L173)) | `UpdateFailed($this->alias, $this->new, $this->old, 'Copy Files', ...)` | 目标版本 | 当前版本 | 目标版本 | 当前版本 |
| Finish 失败 ([L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192)) | `UpdateFailed($this->alias, $this->new, $this->old, 'Finish', ...)` | 目标版本 | 当前版本 | 目标版本 | 当前版本 |

**结论**：
- **Download/Unzip/Copy Files/Finish 四个阶段**：事件对象 `old` 字段 = 目标版本，`new` 字段 = 当前版本，与字段字面含义相反。
- **Version 阶段**：事件对象 `old = false`，`new = null`——两者都不是版本号，依赖这两个字段做版本比较的下游代码会得到无意义数据。

### 1.3 Stage 字段完整取值

| stage 值 | 触发行 |
|---|---|
| `Version` | [Update.php#L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L64-L64) |
| `Download` | [Update.php#L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L131-L131) |
| `Unzip` | [Update.php#L152](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L152-L152) |
| `Copy Files` | [Update.php#L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L173-L173) |
| `Finish` | [Update.php#L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192) |

---

## 2. CLI 入口与 Web 入口的失败处理差异

### 2.1 CLI 入口（Update 命令）

[Update.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php) 中四个阶段：

| 阶段 | 成功事件 | 失败事件 |
|---|---|---|
| 下载 | [L125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L125-L125)：`UpdateDownloaded` | [L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L131-L131)：`UpdateFailed` |
| 解压 | [L146](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L146-L146)：`UpdateUnzipped` | [L152](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L152-L152)：`UpdateFailed` |
| 复制 | [L167](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L167-L167)：`UpdateCopied` | [L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L173-L173)：`UpdateFailed` |
| 完成 | （无直接事件，由 `update:finish` 命令内部触发 `UpdateFinished`） | [L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192)：`UpdateFailed` |

CLI 入口在所有阶段失败时**均触发 `UpdateFailed` 事件**，该事件被 [SendNotificationOnFailure](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Update/SendNotificationOnFailure.php) 捕获并发送邮件/Slack 通知（参见 [Event.php#L90-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L90-L92)）。

### 2.2 Web 入口（Updates 控制器）

[Updates.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php) 中四个阶段：

| 阶段 | 成功事件 | 失败处理 |
|---|---|---|
| 下载 | [L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L179-L179)：`UpdateDownloaded` | [L189-L196](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L189-L196)：仅返回 JSON 错误，**不触发** `UpdateFailed` |
| 解压 | [L215](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L215-L215)：`UpdateUnzipped` | [L226-L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L226-L232)：仅返回 JSON 错误，**不触发** `UpdateFailed` |
| 复制 | [L251](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L251-L251)：`UpdateCopied` | [L262-L268](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L262-L268)：仅返回 JSON 错误，**不触发** `UpdateFailed` |
| 完成 | （无直接事件，由 `FinishUpdate` Job 内部调用 `update:finish` 触发 `UpdateFinished`） | [L293-L299](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Install/Updates.php#L293-L299)：仅返回 JSON 错误，**不触发** `UpdateFailed` |

**关键差异**：Web 入口的 `catch` 块仅组装 JSON 响应（`success: false, error: true, message: $e->getMessage()`），**从不调用 `event(new UpdateFailed(...))`**。这意味着：

- Web 更新失败时**不会发送通知邮件/Slack**
- Web 更新失败时**没有事件记录可供排查**
- 前端通过 `response.data.error` 和 `response.data.message` 显示错误（见 [update.js](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/install/update.js)）

---

## 3. 完成阶段：FinishUpdate Job / update:finish / update:db 三者关系

### 3.1 调用链总览

```
CLI:  Update::finish()  ──dispatch──►  FinishUpdate Job
Web:  Updates::finish() ──dispatch──►  FinishUpdate Job
                                       │
                                       ├── authorize()  检查模块是否存在
                                       ├── getCompanies()  决定需要更新哪些公司
                                       │   ├── listener=none → Artisan::call('cache:clear'), 返回 []
                                       │   ├── listener=one  → 返回 [$company_id]
                                       │   └── listener=all  → 返回所有安装了该模块的公司
                                       │
                                       └── 逐公司 Console::run("update:finish {alias} {cid} {new} {old}")
                                                           │
                                                           ├── $this->call('cache:clear')
                                                           ├── 验证文件版本 == 请求版本
                                                           ├── company($cid) 为空 → return 提前退出
                                                           ├── $company->makeCurrent()
                                                           ├── 设置模块 locale
                                                           ├── 禁用 model cache
                                                           └── event(new UpdateFinished(...))
```

### 3.2 FinishUpdate Job 的三种分支

[Jobs/Install/FinishUpdate.php#L72-L97](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L72-L97)：

- **alias = 'core'**：直接返回 `[$this->company_id]`，只更新触发更新的公司
- **alias ≠ 'core'**：调用 `getCompaniesOfModule()`，根据监听器类型决定：
  - `none`：模块目录下没有 `Listeners/Update` 文件夹（[L108-L109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L108-L109)），或文件夹存在但所有监听器版本都不在 `(old, new]` 区间内（[L118-L120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L118-L120)）。此时 **只清缓存不触发 `UpdateFinished`**（[L86](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L86-L86) `Artisan::call('cache:clear')`，返回空数组，foreach 不执行）
  - `one`：存在迁移监听器但都**不实现** `ShouldUpdateAllCompanies`（[L127-L129](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L127-L129)），只更新当前公司
  - `all`：存在实现了 `ShouldUpdateAllCompanies` 的监听器（[L122-L125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L122-L125)），查询所有安装公司并逐个更新

**关键**：当 `listener = 'none'` 时，**不触发 `UpdateFinished` 事件**，只清缓存即视为完成。

### 3.3 update:finish 命令的提前退出

[Console/Commands/FinishUpdate.php#L50-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L50-L54)：

```php
$company = company($company_id);
if (empty($company)) {
    return;  // 公司不存在，静默退出，不触发 UpdateFinished
}
```

此分支发生时：
- 文件已经复制完成（版本号已是新版本）
- 缓存已清除（[L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L35-L35)）
- 但**没有触发 `UpdateFinished`**，也**没有抛异常**
- FinishUpdate Job 的 `Console::run()` 收到的是命令正常退出（exit code 0），不抛异常
- 外层 Update.php 的 finish() 认为成功

### 3.4 update:db 命令的用途差异

[Console/Commands/UpdateDb.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UpdateDb.php) 是一个**纯手动触发**命令：

| 对比项 | update:finish | update:db |
|---|---|---|
| 文件版本检查 | ✅ 有（[L43-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L43-L48)） | ❌ 无 |
| 缓存清除 | ✅ 有（[L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L35-L35)） | ❌ 无 |
| 公司存在检查 | ✅ 有（[L50-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L50-L54)） | ❌ 无（直接 `company($company_id)->makeCurrent()`） |
| locale 设置 | ✅ 有（[L59-L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L59-L61)） | ❌ 无 |
| 触发事件 | `UpdateFinished` | `UpdateFinished` |
| 使用场景 | 更新流程中自动调用 | 手动重新执行迁移监听器（文件已到位，只需跑数据库迁移） |

---

## 4. UpdateExtraModules 的 return false 行为分析

### 4.1 项目源码行为

[Listeners/Module/UpdateExtraModules.php#L58-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/UpdateExtraModules.php#L58-L65)：

```php
if (true !== $result = Console::run($command)) {
    $message = !empty($result) ? $result : trans('modules.errors.finish', ['module' => $alias]);
    report($message);                              // 写入日志，不抛异常
    return false;                                  // 终止事件传播
}
```

- **不抛异常**：使用 `report()` 而非 `throw`
- **返回 false**：依赖 Laravel 事件分发器对 `false` 返回值的处理
- **外层命令正常退出**：`update:finish` 命令不以异常结束，`Console::run()` 向上返回 `true`，FinishUpdate Job 不抛异常，外层 finish() 认为成功

### 4.2 Laravel 事件分发器对 `return false` 的行为

项目 composer.lock 显示 laravel/framework 版本为 **10.50.2**（见 [composer.lock#L4904-L4905](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/composer.lock#L4904-L4905)）。

Laravel 10 的事件分发 `Dispatcher::invokeListeners` 位于 `vendor/laravel/framework/src/Illuminate/Events/Dispatcher.php`，其实现（引用自 [laravel/framework@v10.50.2 — Dispatcher.php#L193-L223](https://github.com/laravel/framework/blob/v10.50.2/src/Illuminate/Events/Dispatcher.php#L193-L223)）：

```php
protected function invokeListeners($listeners, $event, $payload)
{
    $halted = false;
    foreach ($listeners as $listener) {
        if ($halted) {
            break;
        }
        $response = $listener($event, $payload);
        if ($response === false) {
            $halted = true;
        }
    }
    return $halted ? false : null;
}
```

**证据说明**：上述代码段引自 Laravel 10.50.2 官方源码（对应项目 composer.lock 指定版本）。如果本地 `vendor/` 目录不存在，可通过 GitHub 链接 [Dispatcher.php#L193-L223](https://github.com/laravel/framework/blob/v10.50.2/src/Illuminate/Events/Dispatcher.php#L193-L223) 验证。

**语义**：当某个监听器返回 `false` 时，`$halted` 被置为 `true`，下一轮循环 `break`，**所有后续注册的监听器都不会被调用**。

### 4.3 对事件传播链的影响

[Event.php#L15-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L15-L36) 中 `UpdateFinished` 监听器的注册顺序：

```
1. CreateModuleUpdatedHistory     ← 注册在第 1 位，始终先执行
2. UpdateExtraModules             ← 注册在第 2 位，可能 return false
3. Version300                     ← 注册在第 3 位，若第 2 位 return false 则被跳过
4. Version303                     ← 注册在第 4 位，同上
5. ... (所有后续版本监听器)         ← 全部被跳过
```

**场景**：模块 A 有 `extra-modules` 配置，其中 B 为 `required`，B 有新版本待更新。

1. 模块 A 的文件复制成功
2. `update:finish` 触发 `event(new UpdateFinished('A', new, old))`
3. Laravel 按注册顺序逐监听器调用
4. 第 1 位 `CreateModuleUpdatedHistory::handle()` 执行完毕，未返回 false → 继续
5. 第 2 位 `UpdateExtraModules::handle()` 尝试 `Console::run("update B {cid} {latest}")`，B 更新失败，`Console::run()` 返回错误字符串，进入 if 分支，`report($message)` 写日志，**`return false`**
6. `Dispatcher::invokeListeners` 在第 2 位监听器上收到 `$response === false`，设置 `$halted = true`，下一轮循环 `break`
7. **第 3 位及之后的 Version300、Version303 等所有版本迁移监听器均不被调用**
8. `update:finish` 的 artisan 命令正常退出（exit 0），`Console::run()` 返回 `true`，FinishUpdate Job 不抛异常
9. 外层 `Update::finish()` 中 `$this->dispatch(new FinishUpdate(...))` 正常返回，finish() 返回 `true`
10. 整个更新被标记为**成功**

**结果**：
- 模块 A 历史可能已写入（取决于 `modules` 表中是否有对应 alias 记录）
- 模块 A 文件是新版本
- 依赖 B 未更新
- 模块 A 的数据库迁移未执行
- 无 `UpdateFailed` 通知（因为 UpdateExtraModules 没抛异常）

### 4.4 对 core 更新的影响

`UpdateExtraModules` 在 [L24-L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/UpdateExtraModules.php#L24-L26) 对 `alias == 'core'` 直接 `return`，所以 core 更新不受此监听器影响。

---

## 5. UpdateFailed message 字段的实际定位能力

### 5.1 各 Job 抛出的异常消息

**DownloadFile Job**（[Jobs/Install/DownloadFile.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php)）：

| 行号 | 异常消息 | 来源 |
|---|---|---|
| [L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php#L38-L38) | `trans('modules.errors.download')` → "Not able to download :module" | 通用翻译 |
| [L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php#L46-L46) | `$json->message` | 远端 API 返回的具体错误（唯一的具体消息来源） |
| [L63](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php#L63-L63) | `trans('modules.errors.download')` | 通用翻译 |

**UnzipFile Job**（[Jobs/Install/UnzipFile.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php)）：

| 行号 | 异常消息 | 来源 |
|---|---|---|
| [L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php#L35-L35) | `trans('modules.errors.unzip')` → "Not able to unzip :module" | 通用翻译 |
| [L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php#L46-L46) | `trans('modules.errors.unzip')` | 通用翻译 |
| [L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php#L64-L64) | `trans('modules.errors.unzip')` | 通用翻译 |
| [L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php#L69-L69) | `trans('modules.errors.unzip')` | 通用翻译 |

**CopyFiles Job**（[Jobs/Install/CopyFiles.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php)）：

| 行号 | 异常消息 | 来源 |
|---|---|---|
| [L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L35-L35) | `trans('modules.errors.file_copy')` → "Not able to copy :module files" | 通用翻译 |
| [L45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L45-L45) | `trans('modules.errors.file_copy')` | 通用翻译 |
| [L52](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L52-L52) | `trans('modules.errors.file_copy')` | 通用翻译 |
| [L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L66-L66) | `trans('modules.errors.file_copy')` | 通用翻译 |

**FinishUpdate Job**（[Jobs/Install/FinishUpdate.php#L54-L57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L54-L57)）：

```php
if (true !== $result = Console::run($command)) {
    $message = !empty($result) ? $result : trans('modules.errors.finish', ['module' => $this->alias]);
    throw new \Exception($message);
}
```

其 `$result` 来自 `Console::run()`。查看 [Utilities/Console.php#L12-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Console.php#L12-L36)：

- 正常退出且输出通过 `isValidOutput` 检查 → 返回 `true`
- 否则捕获 `Throwable`，取 `$e->getMessage()`，经 `formatOutput()` 清洗后返回

因此 FinishUpdate 的 message 来源优先级：
1. **子进程异常消息**：当 `update:finish` 命令以非零状态退出时，`Process::mustRun()` 抛出 `ProcessFailedException`，其 `getMessage()` 包含子命令输出的错误信息
2. **通用翻译消息**：仅当 `$result` 为空字符串时，才回退到 `trans('modules.errors.finish')`

### 5.2 实际定位能力评估

- **Download 阶段**：只有当远端 API 返回 JSON 错误对象时（[L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php#L46-L46)），message 才是具体的 API 错误消息。其他情况一律是 "Not able to download :module"，**无法区分是网络超时、权限不足、临时目录不可写还是远端服务错误**
- **Unzip 阶段**：所有异常一律是 "Not able to unzip :module"，**无法区分是 ZIP 损坏、磁盘空间不足、文件权限问题、还是 ZipSlip 路径穿越被拦截**（[L57-L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php#L57-L66) 的 ZipSlip 检查失败也抛出同一条通用消息）
- **Copy Files 阶段**：所有异常一律是 "Not able to copy :module files"，**无法区分路径穿越拦截、`copyDirectory` 失败、`module.json` 缺失等具体原因**
- **Finish 阶段**：message 优先使用子命令的错误输出，其中可能包含监听器抛出的具体异常消息；如果子命令无任何输出，才回退到 "Not able to finalize :module installation"

**结论**：`message` 字段在 Download/Unzip/Copy Files 三个阶段几乎都是翻译后的通用提示，**不能直接定位到网络/权限/ZIP 损坏/磁盘空间等具体原因**。排查时需要结合 `step` 字段缩小阶段范围，然后查看 Laravel 日志获取完整堆栈。Finish 阶段的 message 可能包含子命令的具体错误。

---

## 6. 复制成功但 Finish 失败的状态不一致场景

### 6.1 模块更新与 Core 更新的可达路径

**模块更新可达路径**：`DownloadFile → UnzipFile → CopyFiles → FinishUpdate Job → update:finish`

**Core 更新可达路径**：`DownloadFile → UnzipFile → CopyFiles → FinishUpdate Job → update:finish`

两者在复制阶段的差异在于目标路径：
- 模块：`modules/{StudlyAlias}/`（[CopyFiles.php#L76-L83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L76-L83)）
- Core：`base_path()`（[CopyFiles.php#L61-L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php#L61-L62)）

### 6.2 模块历史写入的真实条件

[CreateModuleUpdatedHistory.php#L17-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php#L17-L41)：

```php
public function handle(Event $event)
{
    $model = Module::where('alias', $event->alias)->first();  // 查 modules 表
    if (empty($model)) {
        return;                                                // 找不到记录 → 不写历史
    }
    $module = module($event->alias);                           // 从文件系统加载模块实例
    if (empty($module)) {
        return;                                                // 实例为空 → 不写历史
    }
    ModuleHistory::create([...]);                              // 写入历史
}
```

[Module.php#L7-L16](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Module/Module.php#L7-L16) 显示 `Module` 模型映射 `modules` 数据库表，`scopeAlias` 只是 `where('alias', ...)`。

**结论**：文件复制完成（`modules/{StudlyAlias}/` 目录有文件）**不等同于** `modules` 表中有对应 `alias` 记录。`Module::where('alias', ...)->first()` 能否找到记录取决于：
- 当前公司作用域下 `modules` 表中是否已存在该 alias 的行（由安装流程在安装时写入）
- 与文件是否到位无直接因果关系

因此，即使文件复制成功、`module()` 能从文件系统加载实例，`CreateModuleUpdatedHistory` 仍可能因为 `modules` 表中没有对应行而**跳过历史写入**。

### 6.3 模块更新：FinishUpdate Job 抛异常的两类场景

FinishUpdate Job 在 [L52-L58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/FinishUpdate.php#L52-L58) 统一处理 `Console::run()` 的返回，但失败可能发生在 `update:finish` 内部两个不同阶段：

#### 场景 A1：update:finish 在触发 UpdateFinished 之前失败

触发点：[FinishUpdate.php#L42-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L42-L48) 文件版本检查失败。

```
update:finish handle()
├── L35: $this->call('cache:clear')                       ← 已执行
├── L43: $version = module($alias)->get('version')        ← 读文件版本
├── L44: if ($version != $new)                            ← 检查失败
│   ├── L45: logger(...)                                   ← 写日志
│   └── L47: throw new \Exception(trans('modules.errors.finish'))  ← 抛异常
│
└── 以下都未执行：
    ├── L50-L54: company() 检查
    ├── L56: makeCurrent()
    ├── L66: event(new UpdateFinished(...))  ← 未触发
```

状态：
- 文件：已复制为新版本（CopyFiles 成功）
- 缓存：已清除（`$this->call('cache:clear')` 已执行）
- `UpdateFinished`：**未触发** → `CreateModuleUpdatedHistory` **未执行** → 历史未写入
- `UpdateFailed`：**已触发**（由 Update.php 的 catch 块在 [L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/Update.php#L192-L192) 触发）
- `message` 来源：`trans('modules.errors.finish')` → "Not able to finalize :module installation"（因为子命令在触发 UpdateFinished 前就抛异常，Console::run 捕获到的是该异常消息）

#### 场景 A2：update:finish 在 UpdateFinished 监听器执行中失败

触发点：`UpdateFinished` 事件传播时某个监听器抛异常，例如 `Version300` 的迁移失败。

```
update:finish handle()
├── L35:  cache:clear                                      ← 已执行
├── L43-L48: 版本检查通过                                   ← 继续
├── L50-L54: company 存在                                   ← 继续
├── L56: makeCurrent()                                     ← 已切换公司
├── L64: 禁用 model cache                                   ← 已执行
└── L66: event(new UpdateFinished(...))                     ← 触发事件
    │
    ├── [Event.php#L16] CreateModuleUpdatedHistory          ← 排在第 1 位，已执行
    │   └── 可能已写入历史（取决于 modules 表是否有记录）
    │
    ├── [Event.php#L17] UpdateExtraModules                  ← 排在第 2 位，已执行
    │   └── 若依赖模块更新失败 return false → 终止传播（见第 4 节）
    │
    └── [Event.php#L18] Version300                          ← 排在第 3 位，执行
        └── 抛异常 → update:finish 命令以非零退出
```

状态：
- 文件：已复制为新版本
- 缓存：已清除
- `CreateModuleUpdatedHistory`：**可能已执行并写入历史**（在 [Event.php#L16](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L16-L16) 第 1 位注册，早于后续版本监听器），但仅当 `modules` 表有对应 alias 记录时才实际写入
- `UpdateExtraModules`：若没有依赖模块则直接 return；若有依赖模块但依赖更新成功则不影响
- 版本迁移监听器：在抛出异常的那个监听器**之前**的可能已完成，**之后**的未执行
- `UpdateFailed`：**已触发**（由 Update.php catch 块触发）
- `message` 来源：优先使用子命令错误输出（`ProcessFailedException` 的消息），其中包含具体监听器的异常内容；若子命令无输出才回退到 `trans('modules.errors.finish')`

### 6.4 模块更新：update:finish 静默退出场景

触发点：[FinishUpdate.php#L50-L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/FinishUpdate.php#L50-L53) `company($company_id)` 返回空。

状态：
- 文件：新版本
- 缓存：已清除
- `UpdateFinished`：**未触发** → 历史未写入，迁移未执行
- `UpdateFailed`：**未触发**（命令正常 exit 0）
- 整体被标记为成功，但什么也没完成

### 6.5 模块更新：UpdateExtraModules 终止传播场景

触发点：[UpdateExtraModules.php#L58-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/UpdateExtraModules.php#L58-L65) 依赖模块更新失败，`report()` + `return false`。

状态：
- 文件：新版本
- 缓存：已清除
- `CreateModuleUpdatedHistory`：排在第 1 位（[Event.php#L16](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L16-L16)），已执行；写入与否取决于 `modules` 表记录存在
- `UpdateExtraModules`：排在第 2 位，依赖更新失败，`return false` 终止传播
- 版本迁移监听器（Version300 等）：**被跳过**（Laravel Dispatcher `break`）
- `UpdateFailed`：**未触发**（没有异常）
- 整体被标记为成功，但数据库迁移未执行

### 6.6 Core 更新：版本迁移监听器失败场景

Core 更新不经过 `UpdateExtraModules`（在 [L24-L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/UpdateExtraModules.php#L24-L26) 直接 return）。

```
UpdateFinished 触发
├── [Event.php#L16] CreateModuleUpdatedHistory
│   └── Module::where('alias', 'core')->first()
│       └── 若无 core 行 → return，不写历史
│
├── [Event.php#L17] UpdateExtraModules → alias=core → return
│
└── [Event.php#L18] Version300 → 执行 migrate → 失败抛异常
    └── update:finish 以非零退出
```

状态：
- 文件：core 新版本已覆盖 `base_path()`
- `CreateModuleUpdatedHistory`：通常**不写入**（`modules` 表中通常没有 alias 为 `'core'` 的记录，[L19-L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Update/CreateModuleUpdatedHistory.php#L19-L23) return）
- 迁移：部分执行或完全未执行
- `UpdateFailed`：**已触发**（由 Update.php catch 块）
- `message` 来源：优先使用 `Console::run()` 返回的子命令错误字符串（包含 `ProcessFailedException` 的消息，其中有具体迁移错误）；仅当子命令输出为空时才回退到 `trans('modules.errors.finish')` → "Not able to finalize core installation"

### 6.7 状态不一致汇总对比

| 维度 | 模块更新 | Core 更新 |
|---|---|---|
| 复制目标 | `modules/{StudlyAlias}/` | `base_path()` |
| 历史写入条件 | `modules` 表有 alias 行 + 文件系统能加载模块 | `modules` 表有 `'core'` 行（通常不存在） |
| UpdateExtraModules 干扰 | **有**（可能 return false 终止传播） | **无**（alias=core 直接 return） |
| 多公司更新 | 取决于监听器类型（none/one/all） | 只更新触发公司 |
| 清理难度 | 低（可删模块目录重装） | 高（core 文件已覆盖，无法简单回滚） |
| Finish 失败 message 来源 | 子命令错误优先，回退翻译消息 | 同左（不固定为 core 专用消息） |

---

## 总结

更新流程采用"事件驱动 + 分阶段"设计，各阶段通过事件解耦，修正后的关键特征如下：

1. **UpdateFailed 的 old/new 字段值与含义相反**——Download/Unzip/Copy Files/Finish 阶段事件对象 `old` = 目标版本、`new` = 当前版本；Version 阶段 `old = false`、`new = null`
2. **Web 入口不触发 UpdateFailed**——四个阶段失败仅返回 JSON，不发通知
3. **FinishUpdate Job 的 listener=none 分支只清缓存不触发事件**——静默完成
4. **UpdateExtraModules 的 return false 会终止事件传播但不抛异常**——依赖 Laravel `Dispatcher::invokeListeners` 中 `$response === false → $halted = true → break` 的机制，导致后续版本迁移监听器被跳过而外层仍认为成功
5. **message 字段多为翻译后的通用语句**——Download/Unzip/Copy Files 三阶段几乎都是通用翻译，Finish 阶段优先使用子命令错误输出
6. **Finish 阶段失败分两类**——UpdateFinished 触发前失败（历史未写入）与监听器执行中失败（历史可能已写入但迁移部分完成或被跳过）
7. **模块历史写入取决于 modules 表记录**——文件就位与数据库记录无直接因果关系
8. **Finish 失败的 message 不固定为"core installation"**——优先使用 `Console::run()` 返回的错误字符串，仅空值才回退到翻译消息
