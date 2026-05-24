# 队列任务公司上下文与导入失败通知分析

## 1. payload 中 company_id 何时注入，空 company_id 会怎样

### 注入时机

在 [Queue.php#L32-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L32-L40) 的 `boot()` 方法中，通过 `createPayloadUsing` 钩子在队列任务创建 payload 时注入：

```php
app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
    $company_id = company_id();

    if (empty($company_id)) {
        return [];
    }

    return ['company_id' => $company_id];
});
```

### 空 company_id 的后果

- 当 `company_id()` 返回空时（即当前没有选中的公司），`createPayloadUsing` 返回空数组，payload 中 **不会包含** `company_id` 字段
- 在后续 `JobProcessing` 事件处理中（[Queue.php#L45-L47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L45-L47)），会检查 payload 是否存在 `company_id`，如果不存在则直接 return，不进行公司上下文设置
- 这意味着：在没有公司上下文的环境中（如某些命令行任务、安装流程等）分发的队列任务，不会携带公司信息，也不会尝试设置公司上下文

---

## 2. JobProcessing 如何解析 company、删除不可解析 job、makeCurrent 和 registerModules

在 [Queue.php#L42-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L42-L80) 中，`JobProcessing` 事件监听器的处理流程：

### 解析 company

```php
$payload = $event->job->payload();

if (! array_key_exists('company_id', $payload)) {
    return;
}

try {
    $company = company($payload['company_id']);
} catch (\Throwable $e) {
    // ... 删除 job
}
```

### 删除不可解析 job

两种情况会删除 job 并记录警告日志：

1. **解析异常**（[Queue.php#L49-L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L49-L61)）：调用 `company($id)` 时抛出异常（如数据库连接问题、模型查询异常等）
2. **公司不存在**（[Queue.php#L63-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L63-L72)）：`company($id)` 返回 `null` 或空值

```php
// 解析异常时删除
$event->job->delete();
logger()->warning('Company could not be resolved for queued job, job deleted.', [...]);

// 公司为空时删除
$event->job->delete();
logger()->warning('Company not found, job deleted.', [...]);
```

### makeCurrent

成功解析到公司后，调用 `$company->makeCurrent()`（[Queue.php#L74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L74)）将该公司设置为当前上下文，使后续任务执行时 `company()` 和 `company_id()` 等辅助函数能正确返回该公司信息。

### registerModules

在 [Queue.php#L77-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L77-L79)：

```php
if (should_queue()) {
    $this->registerModules();
}
```

`registerModules()` 定义在 [Modules.php#L655-L658](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Modules.php#L655-L658)：

```php
public function registerModules(): void
{
    app(\Akaunting\Module\Contracts\ActivatorInterface::class)->register();
}
```

当 `should_queue()` 为 true 时（即队列驱动不是 `sync`），调用 `$this->registerModules()`，该方法委托给 `ActivatorInterface::register()` 执行注册。

---

## 3. Import::fromExcel 何时 queue 到 imports

在 [Import.php#L23-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L23-L80) 中：

### 队列触发条件

```php
$should_queue = should_queue();  // config('queue.default') != 'sync'

if ($should_queue) {
    self::importQueue($class, $file, $translation);
} else {
    $class->import($file);  // 同步执行
}
```

### queue 到 imports 的完整流程

`importQueue()` 方法（[Import.php#L65-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L65-L80)）**不是立即 queue**，而是按以下顺序执行：

```php
protected static function importQueue($class, $file, string $translation): void
{
    // 第1步：先同步执行 toArray()，将 Excel 转为数组
    $rows = $class->toArray($file);

    // 第2步：统计总行数
    $total_rows = 0;
    if (! empty($rows[0])) {
        $total_rows = count($rows[0]);
    } else if (! empty($sheets = $class->sheets())) {
        $total_rows = count($rows[array_keys($sheets)[0]]);
    }

    // 第3步：上述步骤成功后，才调用 queue()
    $class->queue($file)->onQueue('imports')->chain([
        new NotifyUser(user(), new ImportCompleted($translation, $total_rows))
    ]);
}
```

**关键边界**：如果 `toArray($file)` 或行数统计阶段抛出异常，异常会被 `fromExcel()` 的 `catch(Throwable $e)` 捕获，**不会创建 imports 队列 job**。

---

## 4. JobFailed 只处理哪类异常和哪类 Excel job，以及如何通知用户

在 [Queue.php#L82-L127](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L82-L127) 中，`JobFailed` 事件监听器有严格的过滤条件和通知流程：

### 只处理的异常类型

[Queue.php#L83-L85](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L83-L85)：

```php
if (!$event->exception instanceof \Maatwebsite\Excel\Validators\ValidationException) {
    return;
}
```

**仅处理** `Maatwebsite\Excel\Validators\ValidationException`（Excel 数据校验异常），其他所有异常都会被直接忽略。

### 只处理的 Excel job 类型

[Queue.php#L97-L100](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L97-L100)：

```php
$excel_job = unserialize($payload->data->command);
if (!$excel_job instanceof \Maatwebsite\Excel\Jobs\ReadChunk) {
    return;
}
```

**仅处理** `Maatwebsite\Excel\Jobs\ReadChunk` 类型的 job。

此外，还要求 import 类必须是以下两种抽象类之一的实例（[Queue.php#L108-L110](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L108-L110)）：
- `\App\Abstracts\Import`
- `\App\Abstracts\ImportMultipleSheets`

### failures 收集与通知流程

通过上述全部过滤后，进入通知逻辑（[Queue.php#L112-L126](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L112-L126)）：

```php
$errors = [];

// 遍历 ValidationException 的 failures()，生成错误消息
foreach ($event->exception->failures() as $failure) {
    $message = trans('messages.error.import_column', [
        'message'   => collect($failure->errors())->first(),
        'column'    => $failure->attribute(),
        'line'      => $failure->row(),
    ]);
    $errors[] = $message;
}

// errors 非空时，通过 import 实例的 user 属性通知用户
if (!empty($errors)) {
    $class->user->notify(new ImportFailed($errors));
}
```

通知对象来自 import 实例的 `$class->user` 属性（由 `App\Abstracts\Import` 或 `App\Abstracts\ImportMultipleSheets` 提供），通过 `notify()` 发送 `ImportFailed` 通知。

---

## 5. 不会发送 ImportFailed 通知的失败场景

基于上述全部代码路径，以下是 **不会发送 `ImportFailed` 通知** 的失败场景，每个场景明确标注属于哪个阶段：

### 场景1：预入队阶段失败 — toArray() 抛出异常

**阶段**：`fromExcel()` → `importQueue()` 中的 `toArray($file)` 调用

当 `should_queue()` 为 true，`importQueue()` 中 `$class->toArray($file)` 抛出异常时：
- 异常被 `fromExcel()` 的 `catch(Throwable $e)` 捕获
- 尚未执行到 `$class->queue($file)->onQueue('imports')`，所以 **imports 队列 job 不会被创建**
- 既然队列 job 不存在，不会有 `JobFailed` 事件，自然不会发送 `ImportFailed` 通知

### 场景2：同步导入失败 — fromExcel() 捕获异常

**阶段**：`fromExcel()` 的同步执行路径（`should_queue()` 为 false）

当使用 `sync` 驱动时（`should_queue()` 返回 false），`fromExcel()` 直接调用 `$class->import($file)`。如果导入过程中抛出异常：
- 异常被 `fromExcel()` 的 `catch(Throwable $e)` 捕获
- 调用 `self::flashFailures($e)`，如果是 `ValidationException` 则通过 `flash()` 在页面显示错误
- 返回 `['success' => false, 'error' => true, 'message' => ...]`
- 由于是同步模式，**不经过队列的 `JobFailed` 监听器**，所以不会发送 `ImportFailed` 通知

### 场景3：JobProcessing 阶段被删除的 job

**阶段**：`JobProcessing` 事件处理

队列任务在 `JobProcessing` 阶段因以下原因被删除时：
- `company($payload['company_id'])` 抛出异常
- `company($payload['company_id'])` 返回空（公司不存在）

job 被 `$event->job->delete()` 删除，任务不会执行，**不会触发 `JobFailed` 事件**，因此不会发送 `ImportFailed` 通知。

### 场景4：非 ValidationException 异常导致的队列 job 失败

**阶段**：队列 job 执行中（`JobFailed` 事件触发，但被过滤掉）

导入队列 job 执行过程中发生任何非 `ValidationException` 异常时：
- 如数据库连接失败、PHP 内存溢出、代码逻辑异常等
- `JobFailed` 监听器在第一个 `if` 判断中直接 `return`
- **不会发送 `ImportFailed` 通知**

### 场景5：非 ReadChunk 类型的队列 job 失败

**阶段**：队列 job 执行中（`JobFailed` 事件触发，但被过滤掉）

即使异常是 `ValidationException`，如果失败的 job 不是 `Maatwebsite\Excel\Jobs\ReadChunk` 类型：
- `JobFailed` 监听器在 `instanceof ReadChunk` 判断中直接 `return`
- **不会发送 `ImportFailed` 通知**

### 场景6：导入类不继承自系统抽象类

**阶段**：队列 job 执行中（`JobFailed` 事件触发，但被过滤掉）

即使通过了前两个过滤，如果反序列化出的 import 类不是 `App\Abstracts\Import` 或 `App\Abstracts\ImportMultipleSheets` 的实例：
- `JobFailed` 监听器在抽象类判断中直接 `return`
- **不会发送 `ImportFailed` 通知**
