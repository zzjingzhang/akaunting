# 队列任务公司上下文与导入失败通知分析

## 1. payload 中 company_id 何时注入，空 company_id 会怎样

### 注入时机

在 [Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L32-L40) 的 `boot()` 方法中，通过 `createPayloadUsing` 钩子在队列任务创建 payload 时注入：

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
- 在后续 `JobProcessing` 事件处理中，会检查 payload 是否存在 `company_id`，如果不存在则直接返回，不进行公司上下文设置
- 这意味着：在没有公司上下文的环境中（如某些命令行任务、安装流程等）分发的队列任务，不会携带公司信息，也不会尝试设置公司上下文

## 2. JobProcessing 如何解析 company、删除不可解析 job、makeCurrent 和 registerModules

在 [Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L42-L80) 中，`JobProcessing` 事件监听器的处理流程：

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

1. **解析异常**：调用 `company($id)` 时抛出异常（如数据库连接问题、模型查询异常等）
2. **公司不存在**：`company($id)` 返回 `null` 或空值

```php
// 解析异常时删除
$event->job->delete();
logger()->warning('Company could not be resolved for queued job, job deleted.', [...]);

// 公司为空时删除
$event->job->delete();
logger()->warning('Company not found, job deleted.', [...]);
```

### makeCurrent

成功解析到公司后，调用 `$company->makeCurrent()` 将该公司设置为当前上下文，使后续任务执行时 `company()` 和 `company_id()` 等辅助函数能正确返回该公司信息。

### registerModules

```php
if (should_queue()) {
    $this->registerModules();
}
```

当队列配置不是 `sync` 驱动时（即真正的异步队列环境），注册模块，确保模块的服务提供者、路由、视图等资源被正确加载。

## 3. Import::fromExcel 何时 queue 到 imports

在 [Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L23-L80) 中：

### 队列触发条件

```php
$should_queue = should_queue();  // config('queue.default') != 'sync'

if ($should_queue) {
    self::importQueue($class, $file, $translation);
} else {
    $class->import($file);  // 同步执行
}
```

### 队列到 imports

```php
protected static function importQueue($class, $file, string $translation): void
{
    $rows = $class->toArray($file);
    
    // 统计行数用于完成通知...
    
    $class->queue($file)->onQueue('imports')->chain([
        new NotifyUser(user(), new ImportCompleted($translation, $total_rows))
    ]);
}
```

**关键点**：
- 当 `should_queue()` 返回 `true`（即队列驱动不是 `sync`）时，导入任务会被队列化
- 通过 `onQueue('imports')` 指定发送到 `imports` 队列
- 使用 `chain()` 链式调用，在导入完成后发送 `ImportCompleted` 通知

## 4. JobFailed 只处理哪类异常和哪类 Excel job

在 [Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L82-L127) 中，`JobFailed` 事件监听器有严格的过滤条件：

### 只处理的异常类型

```php
if (!$event->exception instanceof \Maatwebsite\Excel\Validators\ValidationException) {
    return;
}
```

**仅处理** `Maatwebsite\Excel\Validators\ValidationException`（Excel 数据校验异常），其他所有异常（如数据库异常、PHP 错误、逻辑异常等）都会被直接忽略。

### 只处理的 Excel job 类型

```php
$excel_job = unserialize($payload->data->command);
if (!$excel_job instanceof \Maatwebsite\Excel\Jobs\ReadChunk) {
    return;
}
```

**仅处理** `Maatwebsite\Excel\Jobs\ReadChunk` 类型的 job（即 Laravel-Excel 库内部的分块读取任务）。

此外，还要求 import 类必须是以下两种抽象类的实例：
- `\App\Abstracts\Import`
- `\App\Abstracts\ImportMultipleSheets`

## 5. 不会发送 ImportFailed 通知的失败场景

基于上述过滤条件，以下场景不会发送 `ImportFailed` 通知：

### 场景1：非 ValidationException 异常

导入过程中发生任何非校验类异常时，不会触发通知：
- 数据库连接失败、查询异常
- 内存溢出、超时错误
- 文件读写错误
- 代码逻辑异常（如空指针、数组越界等）
- 第三方服务调用失败

### 场景2：非 ReadChunk 类型的 job 失败

只有 `ReadChunk` 类型的 job 失败才会被处理，其他 job 类型失败不会通知：
- 导入前的预处理 job 失败
- 自定义队列 job 失败
- 链式任务中的后续 job 失败

### 场景3：非系统抽象类的导入类

如果导入类不继承自 `App\Abstracts\Import` 或 `App\Abstracts\ImportMultipleSheets`，即使是 `ValidationException` 也不会发送通知。

### 场景4：JobProcessing 阶段被删除的 job

队列任务在 `JobProcessing` 阶段因为 company_id 无法解析而被删除时，任务不会执行，自然也不会触发 `JobFailed` 事件和通知。

### 场景5：同步导入模式下的失败

当 `should_queue()` 返回 `false`（使用 sync 驱动）时，导入同步执行，异常在请求周期内直接抛出，通过 `flashFailures()` 在页面显示错误信息，不会走队列失败通知流程。
