# Akaunting 模块安装到启用控制流分析（修订版）

## 1. DownloadFile / UnzipFile / CopyFiles / InstallModule / EnableModule 职责边界

### DownloadFile（下载）
- 文件：[DownloadFile.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/DownloadFile.php)
- 职责：通过 HTTP 从 Akaunting 站点 API 下载模块 zip 包，写入 `storage/app/temp/{temp}/upload.zip`
- 不涉及文件系统模块目录，不涉及数据库

### UnzipFile（解压）
- 文件：[UnzipFile.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/UnzipFile.php)
- 职责：将 zip 解压到 `storage/app/temp/{temp}/`，解压后删除 zip 文件
- 含 ZipSlip 路径穿越防护
- 不涉及文件系统模块目录，不涉及数据库

### CopyFiles（复制）
- 文件：[CopyFiles.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/CopyFiles.php)
- 职责：将解压后的文件从 `storage/app/temp/{temp}/` 复制到 `config('module.paths.modules')/{StudlyAlias}/`
- 含路径穿越防护（realpath 检查）
- 完成后删除临时目录
- 不涉及数据库

### InstallModule（安装）
- 文件：[InstallModule.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/InstallModule.php)
- 职责：仅为 **Job 包装器**
  - `authorize()`：检查文件系统中模块目录是否存在（`moduleExists()`）、locale 合法性
  - `handle()`：通过 `Console::run()` 启动**子进程**执行 `php artisan module:install {alias} {company} {locale}`
  - 根据返回值判断是否抛异常
- 自身不写数据库、不触发事件

### EnableModule（启用）
- 文件：[EnableModule.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/EnableModule.php)
- 职责：仅为 **Job 包装器**
  - `authorize()`：检查文件系统中模块目录是否存在、locale 合法性
  - `handle()`：通过 `Console::run()` 启动**子进程**执行 `php artisan module:enable {alias} {company} {locale}`
  - 根据返回值判断是否抛异常
- 自身不写数据库、不触发事件

---

## 2. Item 控制器各 endpoint 职责

[Item.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Modules/Item.php) 中各方法是**独立的 HTTP endpoint**，由前端分步调用，不是一个方法串完整流程。

### steps()
- 路由：返回安装步骤列表
- 逻辑：根据 `moduleExists(alias)` 判断文件是否已存在
  - **已存在**：只返回 `install` 一个步骤
  - **不存在**：返回 `download → unzip → copy → install` 四个步骤

### download()
- 分发 `DownloadFile` Job
- 成功：返回 `{ path }`
- 失败：返回错误 JSON

### unzip()
- 分发 `UnzipFile` Job（依赖上一步返回的 path）
- 成功：返回 `{ path }`
- 失败：返回错误 JSON

### copy()
- 分发 `CopyFiles` Job（依赖上一步返回的 path）
- 成功后触发 `Copied` 事件
- 成功：返回 `{ alias }`
- 失败：返回错误 JSON

### install()
- 先触发 `Installing` 事件
- 分发 `InstallModule` Job
- 成功：flash 成功消息，可能按 `module.json` 的 `redirect_after_install` 重定向
- 失败：flash 错误消息

### enable()
- 分发 `EnableModule` Job
- 成功：flash 成功消息，重定向到模块详情页
- 失败：flash 错误消息

---

## 3. InstallModule / EnableModule 只是 Job 包装器，真正的状态变更在 Command 中

### InstallModule → module:install → InstallCommand
调用链：`InstallModule::handle()` → `Console::run('module:install ...')` → 子进程中执行 [InstallCommand.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/InstallCommand.php)

```
InstallCommand::handle() 实际操作：
1. prepare()         解析 alias / company_id / locale 参数
2. getModel()        查 modules 表是否已有记录（防重复安装）
3. changeRuntime()   切换当前公司、语言，关闭模型缓存
4. Model::create()   INSERT modules 表：company_id, alias, enabled='1', created_from, created_by
5. createHistory()   INSERT module_histories 表
6. event(Installed)  触发 Installed 事件（同步执行监听器）
7. revertRuntime()   恢复原公司
```

### EnableModule → module:enable → EnableCommand
调用链：`EnableModule::handle()` → `Console::run('module:enable ...')` → 子进程中执行 [EnableCommand.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php)

```
EnableCommand::handle() 实际操作：
1. prepare()         解析 alias / company_id / locale 参数
2. getModel()        查 modules 表是否存在（不存在则 info + return）
3. 检查 enabled       已启用则 comment + return
4. changeRuntime()   切换当前公司、语言，关闭模型缓存
5. $model->enabled = true; $model->save()   UPDATE modules 表
6. createHistory()   INSERT module_histories 表
7. event(Enabled)    触发 Enabled 事件（同步执行监听器）
8. revertRuntime()   恢复原公司
```

---

## 4. modules 表、module_histories 表、事件触发的精确位置

### modules 表写入
- **安装时 INSERT**：[InstallCommand.php#L43-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/InstallCommand.php#L43-L49)
  ```php
  $this->model = Model::create([
      'company_id' => $this->company_id,
      'alias' => $this->alias,
      'enabled' => '1',
      'created_from' => source_name(),
      'created_by' => user_id(),
  ]);
  ```
- **启用时 UPDATE**：[EnableCommand.php#L48-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php#L48-L49)
  ```php
  $this->model->enabled = true;
  $this->model->save();
  ```

### module_histories 表写入
- 方法定义：[Module.php#L58-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Commands/Module.php#L58-L72)
- **没有 action 字段**。`createHistory($action)` 中的 `$action` 参数仅用于 `trans('modules.' . $action)` 生成 `description` 文本。
- 实际写入字段：`company_id`, `module_id`, `version`, `description`, `created_from`, `created_by`
- 安装时调用：[InstallCommand.php#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/InstallCommand.php#L51) → `createHistory('installed')`
- 启用时调用：[EnableCommand.php#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php#L51) → `createHistory('enabled')`

### Installed / Enabled 事件触发位置
- `Installed` 事件：[InstallCommand.php#L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/InstallCommand.php#L53)
- `Enabled` 事件：[EnableCommand.php#L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php#L53)
- 两者均在**子进程**中触发，监听器在子进程内同步执行

---

## 5. Installed 事件监听器顺序

### 注册顺序（[Event.php#L111-L114](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L111-L114)）

```php
\App\Events\Module\Installed::class => [
    \App\Listeners\Module\InstallExtraModules::class,  // 第 1 位
    \App\Listeners\Module\FinishInstallation::class,    // 第 2 位
],
```

### return false 的影响
Laravel 事件系统约定：如果监听器 `handle()` 返回 `false`，则**停止后续监听器的执行**。

在 [InstallExtraModules.php#L55-L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/InstallExtraModules.php#L55-L60) 中：

```php
} catch (\Exception $e) {
    report($e);
    return false;  // 停止事件传播
}
```

如果 required extra module 安装失败，`FinishInstallation`（迁移 + 权限挂载）**不会被执行**。

---

## 6. InstallExtraModules 的具体风险

### 完整链路

```
父模块 InstallCommand:
  Model::create(enabled=1)     ← modules 记录已写入
  createHistory('installed')   ← history 已写入
  event(Installed)             ← 触发事件
    └─ InstallExtraModules::handle()
         └─ 遍历 extra-modules (level=required)
              ├─ moduleExists() → DownloadModule（如果不存在）
              └─ InstallModule
                   └─ 如果抛异常：
                      ├─ report($e)       ← 只记日志
                      └─ return false     ← 阻止 FinishInstallation
```

### 后果
1. **父模块**：`modules` 表记录已创建（enabled=1），`module_histories` 已写入
2. **FinishInstallation 不执行**：`module:migrate` 未运行，`attachDefaultModulePermissions` 未执行
3. **HTTP 层表现**：`InstallModule` Job 中 `Console::run()` 已成功返回（Command 子进程正常完成），HTTP 响应可能为 `success: true`
4. 模块处于"已安装、已启用，但数据库表未创建、权限未挂载"的不一致状态

---

## 7. moduleIsEnabled() 判断的不是"是否已安装"

[InstallExtraModules.php#L44-L47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/InstallExtraModules.php#L44-L47)：

```php
// Check if module is already installed
if ($this->moduleIsEnabled($alias)) {
    continue;
}
```

`moduleIsEnabled()` 定义在 [Modules.php#L415-L422](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Modules.php#L415-L422)：

```php
public function moduleIsEnabled($alias): bool
{
    if (! $this->moduleExists($alias)) {
        return false;
    }
    return module($alias)->enabled();
}
```

它判断的是"模块文件存在**且** enabled 状态为 true"。如果依赖模块已安装但被禁用（enabled=0）：
- `moduleIsEnabled()` 返回 `false`
- 代码进入安装流程，调用 `InstallModule`
- `InstallModule` 中 `module:install` Command 的 `getModel()` 查到已有记录 → "already installed" → `return`
- `Console::run()` 返回成功，不抛异常
- **依赖模块不会被启用**，但其安装被视为"已完成"

---

## 8. EnableModule 异常暴露边界

### 模块目录不存在
- 在 [EnableModule.php#L54-L58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Install/EnableModule.php#L54-L58) 的 `authorize()` 中检查
- `moduleExists($alias)` 返回 false → 抛出 `Exception("Module [{$alias}] not found.")`

### 文件存在但数据库无 modules 记录
- `authorize()` 通过（文件存在）
- `Console::run('module:enable ...')` 启动子进程
- 子进程中 [EnableCommand.php#L33-L37](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/overrides/akaunting/laravel-module/Commands/EnableCommand.php#L33-L37)：
  ```php
  if (! $this->getModel()) {
      $this->info("Module [{$this->alias}] not found.");
      return;   // 静默返回，不抛异常
  }
  ```
- 子进程 exit code 为 0，输出中不包含 `isValidOutput()` 检查的错误关键词
- `Console::run()` 返回 `true`
- **EnableModule Job 认为执行成功**，不抛异常
- 但实际上什么也没做：modules 表没有记录、没有触发 Enabled 事件

---

## 9. FinishMigration 返回码未检查

[FinishInstallation.php#L19-L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/FinishInstallation.php#L19-L26)：

```php
public function handle(Event $event)
{
    $module = module($event->alias);

    Artisan::call('module:migrate', ['alias' => $event->alias, '--force' => true]);

    $this->attachDefaultModulePermissions($module);
}
```

- `Artisan::call()` 的返回值（exit code）被完全忽略
- 如果 `module:migrate` 执行失败（如 SQL 错误），只要不抛出异常：
  - `attachDefaultModulePermissions()` 仍然会执行
  - 权限可能被挂载到一个数据库表不存在的模块上
  - 监听器不返回 false，整体流程仍视为成功

---

## 10. ClearCache 不监听 Installed

[ClearCache.php#L31-L44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Module/ClearCache.php#L31-L44)：

```php
public function subscribe($dispatcher)
{
    $events = [
        'App\Events\Install\UpdateCacheCleared',
        'App\Events\Module\Copied',      // ← 监听
        'App\Events\Module\Enabled',     // ← 监听
        'App\Events\Module\Disabled',    // ← 监听
        'App\Events\Module\Uninstalled', // ← 监听
    ];
    // 不监听 Installed！
}
```

### 影响
如果模块文件已存在于文件系统中：
- `steps()` 只返回 `install` 一个步骤
- 不经过 `download/unzip/copy`
- `Item::install()` 中不触发 `Copied` 事件
- `InstallModule` → `module:install` 完成后触发 `Installed` 事件
- `ClearCache` **不会被调用**
- 缓存中的已安装模块列表可能过期，直到下一次 `Enabled` / `Disabled` / `Uninstalled` 或 `Copied` 触发

---

## 11. 只看 Modules 控制器会漏掉的副作用

控制器 [Item.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Modules/Item.php) 只做了：
1. 分发 DownloadFile / UnzipFile / CopyFiles / InstallModule / EnableModule Job
2. 触发 Installing / Copied 事件
3. 处理 HTTP 响应和重定向

以下副作用在控制器中不可见：

### 额外模块自动安装
- 触发链路：`InstallCommand` → `event(Installed)` → `InstallExtraModules::handle()`
- 副作用：读取 `module.json` 中的 `extra-modules`，对 `level=required` 的模块自动执行 DownloadModule + InstallModule
- 可能触发递归安装（A 依赖 B，B 依赖 C...）

### 数据库迁移执行
- 触发链路：`InstallCommand` → `event(Installed)` → `FinishInstallation::handle()` → `Artisan::call('module:migrate')`
- 副作用：创建/修改数据库表结构

### 权限自动挂载
- 触发链路：`InstallCommand` → `event(Installed)` → `FinishInstallation::handle()` → `attachDefaultModulePermissions()`
- 副作用：为角色分配模块权限

### 缓存清除（部分路径下）
- 触发链路：`Item::copy()` → `event(Copied)` → `ClearCache::handle()`
- 或：`EnableCommand` → `event(Enabled)` → `ClearCache::handle()`
- 副作用：清除 module cache、apps notifications、apps suggestions、installed cache

### 子进程隔离执行
- 所有数据库写入（modules / module_histories）、事件触发、监听器执行均发生在 `Console::run()` 启动的**子进程**中
- 与 HTTP 请求进程的数据库事务不共享
- 异常通过 stdout 文本传递，不是异常对象

---

## 12. 真实控制流全景图

```
┌─ HTTP 进程（同步） ─────────────────────────────────────────────────────┐
│                                                                          │
│  [前端] 分步调用 steps → download → unzip → copy → install            │
│                                                                          │
│  Item::install()                                                         │
│    ├─ event(Installing)           仅触发事件，无监听器注册              │
│    └─ dispatch(InstallModule)     Job 同步执行（dispatchSync）           │
│         │                                                               │
│         └─ InstallModule::handle()                                      │
│              ├─ authorize()         moduleExists + locale 检查          │
│              └─ Console::run('module:install ...')                      │
│                   │                                                    │
│                   │ ┌─ 子进程（独立 PHP 进程） ───────────────────────┐ │
│                   │ │  php artisan module:install {alias} {company}    │ │
│                   │ │                                                  │ │
│                   │ │  InstallCommand::handle()                        │ │
│                   │ │    ├─ getModel() 检查重复 → return（静默）       │ │
│                   │ │    ├─ changeRuntime()                            │ │
│                   │ │    ├─ Model::create(enabled=1)  ── INSERT modules│ │
│                   │ │    ├─ createHistory('installed') ── INSERT history│ │
│                   │ │    ├─ event(Installed)  ── 同步触发监听器        │ │
│                   │ │    │    ├─ InstallExtraModules                   │ │
│                   │ │    │    │    ├─ 递归安装 required extra modules  │ │
│                   │ │    │    │    └─ return false → 停止后续监听器    │ │
│                   │ │    │    └─ FinishInstallation                    │ │
│                   │ │    │         ├─ Artisan::call('module:migrate')  │ │
│                   │ │    │         └─ attachDefaultModulePermissions() │ │
│                   │ │    └─ revertRuntime()                            │ │
│                   │ └──────────────────────────────────────────────────┘ │
│                   │                                                    │
│                   └─ 子进程 exit code 决定 $result === true             │
│                        ├─ true  → HTTP 返回 success:true               │
│                        └─ false → 抛 Exception → HTTP 返回 error       │
│                                                                          │
│  Item::enable()                                                          │
│    └─ dispatch(EnableModule)                                             │
│         └─ （同上，子进程执行 module:enable → UPDATE modules → event）  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**关键事实**：所有数据库写入和事件监听器均在子进程中完成。子进程退出后 HTTP 进程才拿到结果，不存在"控制器返回时监听器还在执行"的情况。
