# Akaunting Banking Transactions Excel 导入完整路径分析

## 一、整体调用链路

```
POST /{company_id}/banking/transactions/import
    ↓
Route::prefix('{company_id}') → mapAdminRoutes() [Providers/Route.php#L228-L234]
    ↓
routes/admin.php L148 (L139 是重复定义，L148 额外带 middleware('import'))
    Route::post('transactions/import', 'Banking\Transactions@import')
    → middleware('import') = throttle:import
    ↓
ImportRequest 校验 [Http/Requests/Common/Import.php]
    'import' => 'required|file|extension:...'
    ↓
Banking\Transactions@import() [Controllers/Banking/Transactions.php#L195-L210]
    ↓
$this->importExcel(new Import, $request, trans_choice('general.transactions', 2))
    ↓
Abstracts\Http\Controller::importExcel() [Controller.php#L85-L88]
    ↓
Utilities\Import::fromExcel($class, $request, $translation) [Utilities/Import.php#L23-L58]
    ├─ should_queue() == false → $class->import($file)  [同步]
    └─ should_queue() == true  → importQueue()  [队列]
    ↓
Imports\Banking\Transactions [implements ToModel, WithMapping, WithValidation...]
    ├─ withValidator() 阶段：复用 FormRequest rules 二次校验  ← 先于 map 执行
    ├─ map() 阶段：父类 + 子类字段映射
    └─ model() 阶段：hasRow() 去重 → new Model($row)
```

### Maatwebsite Excel 3.1.67 执行顺序

Maatwebsite Excel 中 `import()` 方法的执行管线由 `ModelManager` / `ModelImporter` 驱动，核心顺序为：

1. `Reader` 读取 Excel 文件，逐 sheet 逐行迭代
2. `WithHeadingRow` 将首行转为关联数组键名
3. **`withValidator`**（如有 `WithValidation`）— 对**原始行数据**做批量校验
4. **`map()`**（如有 `WithMapping`）— 对每行做字段映射与转换
5. `SkipsEmptyRows` 检查（如 `map()` 返回空数组则跳过该行）
6. **`model()`**（如有 `ToModel`）— 创建 Model 实例

`toArray()` 方法走的是 `Importable` trait 的独立管线，直接通过 `Reader→Sheet→toArray()` 读取原始单元格数据，**不经过** ModelImporter 的 map/validator/model 管线，但会应用 `WithHeadingRow`。

### 真实导入路由（带 company_id 前缀）

[Providers/Route.php#L228-L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Route.php#L228-L234)
```php
protected function mapAdminRoutes()
{
    Facade::prefix('{company_id}')
        ->middleware('admin')
        ->namespace($this->namespace)
        ->group(base_path('routes/admin.php'));
}
```

所有 admin 路由自动加 `{company_id}` 前缀，所以真实 URL 为：
```
/{company_id}/banking/transactions/import
```

routes/admin.php 中有两次定义：
- [L139](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php#L139-L139)：`Route::post('transactions/import', 'Banking\Transactions@import')->name('transactions.import');`
- [L148](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php#L148-L148)：`Route::post('transactions/import', 'Banking\Transactions@import')->middleware('import')->name('transactions.import');`

L148 额外带了 `middleware('import')`（即 `throttle:import`），[Kernel.php#L136-L138](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L136-L138)：
```php
'import' => [
    'throttle:import',
],
```

速率限制定义在 [Providers/Route.php#L306-L308](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Route.php#L306-L308)：
```php
RateLimiter::for('import', function (Request $request) {
    return Limit::perMinute(config('app.throttles.import'));
});
```

### ImportRequest 文件校验

[Http/Requests/Common/Import.php#L14-L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/Import.php#L14-L19)
```php
public function rules()
{
    return [
        'import' => 'required|file|extension:' . config('excel.imports.extensions'),
    ];
}
```

---

## 二、同步导入 vs 队列导入的差异

### 核心判断
[Utilities/helpers.php#L140-L143](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L140-L143)
```php
function should_queue(): bool
{
    return config('queue.default') != 'sync';
}
```

### 1. 成功消息

| 模式 | 翻译键 | en-GB 实际值 |
|------|--------|-------------|
| 同步 | `messages.success.imported` | `:type imported!` |
| 队列 | `messages.success.import_queued` | `:type import has been scheduled! You will receive an email when it is finished.` |

[resources/lang/en-GB/messages.php#L11-L12](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/lang/en-GB/messages.php#L11-L12)
```php
'imported'          => ':type imported!',
'import_queued'     => ':type import has been scheduled! You will receive an email when it is finished.',
```

[Utilities/Import.php#L38-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L38-L41)
```php
$message = trans(
    'messages.success.' . ($should_queue ? 'import_queued' : 'imported'),
    ['type' => $translation]
);
```

**关键区别**：两种模式的成功消息都在 `fromExcel()` 中同步生成并返回。队列模式下 `import_queued` 告知用户已排队，真正完成后通过 `ImportCompleted` 通知（邮件+数据库）告知。

### 2. 失败消息

**同步导入失败**：
[Utilities/Import.php#L42-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L42-L49)
```php
} catch (Throwable $e) {
    if (! $e instanceof SheetNotFoundException) {
        report($e);
    }

    $message = self::flashFailures($e);
    $success = false;
}
```

**SheetNotFoundException 的特殊处理**：`SheetNotFoundException` 不会被 `report()`（即不写入 Laravel 日志/异常报告），直接传递给 `flashFailures()`。

[Utilities/Import.php#L82-L99](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L82-L99)
```php
protected static function flashFailures(Throwable $e): string
{
    if (! $e instanceof ValidationException) {
        return $e->getMessage();
    }

    foreach ($e->failures() as $failure) {
        $message = trans('messages.error.import_column', [
            'message'   => collect($failure->errors())->first(),
            'column'    => $failure->attribute(),
            'line'      => $failure->row(),
        ]);
        flash($message)->error()->important();
    }

    return '';
}
```
- `ValidationException`：逐条 flash 到 session，用户立即看到
- 其他 Throwable（包括 `SheetNotFoundException`）：直接返回 `$e->getMessage()` 作为错误消息
- en-GB 实际值（ValidationException 路径）：`Error: :message Column name: :column. Line number: :line.`

**队列导入失败通知的限定范围**：

[Providers/Queue.php#L82-L127](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L82-L127)
```php
app('events')->listen(JobFailed::class, function ($event) {
    // 条件 1：只处理 ValidationException
    if (!$event->exception instanceof \Maatwebsite\Excel\Validators\ValidationException) {
        return;
    }

    // 条件 2：从队列任务原始 payload 中提取
    $excel_job = unserialize($payload->data->command);
    // 条件 3：只处理 ReadChunk 类型的队列任务
    if (!$excel_job instanceof \Maatwebsite\Excel\Jobs\ReadChunk) {
        return;
    }

    // 条件 4：import 类必须是 Abstracts\Import 或 Abstracts\ImportMultipleSheets
    if (!$class instanceof \App\Abstracts\Import && !$class instanceof \App\Abstracts\ImportMultipleSheets) {
        return;
    }

    // 收集错误并发送通知
    $class->user->notify(new ImportFailed($errors));
});
```

**队列失败通知生效的三重限制**：
1. 异常必须是 `Maatwebsite\Excel\Validators\ValidationException`（其他 Throwable 被忽略，不会通知用户）
2. 失败的队列任务必须是 `ReadChunk`（即 Excel 分块读取任务），其他队列任务类型不触发
3. import 类必须是 `Abstracts\Import` 或 `Abstracts\ImportMultipleSheets` 的实例

**不触发通知的场景**：
- 非 ValidationException 的 Throwable（如 `SheetNotFoundException`）— 同步模式下不 report 但会 flash `$e->getMessage()`；队列模式下静默，无通知
- 非 `ReadChunk` 的队列任务失败
- 非上述两种抽象类的 import 实现

### 3. 通知总行数计算（仅队列导入）

[Utilities/Import.php#L65-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L65-L80)
```php
protected static function importQueue($class, $file, string $translation): void
{
    $rows = $class->toArray($file);

    $total_rows = 0;

    if (! empty($rows[0])) {
        $total_rows = count($rows[0]);
    } else if (! empty($sheets = $class->sheets())) {
        $total_rows = count($rows[array_keys($sheets)[0]]);
    }

    $class->queue($file)->onQueue('imports')->chain([
        new NotifyUser(user(), new ImportCompleted($translation, $total_rows))
    ]);
}
```

**计算逻辑**：
1. 调用 `toArray()` 将 Excel 转为数组。此操作走 `Importable` trait 的 Reader 管线，读取原始单元格数据（应用 `WithHeadingRow`），**不经过** ModelImporter 的 map/validator/model 管线
2. 单 sheet 文件：取 `$rows[0]`（第一个 sheet）的行数
3. 多 sheet 文件：取第一个 sheet 的行数
4. 此 `$total_rows` 在 `ImportCompleted` 通知中展示

### 4. 总结对比表

| 维度 | 同步导入 | 队列导入 |
|------|---------|---------|
| 执行方式 | `$class->import($file)` 直接执行 | `$class->queue($file)->onQueue('imports')` 入队 |
| 成功消息（请求内） | `imported` → `:type imported!` | `import_queued` → `:type import has been scheduled! ...` |
| 成功消息（完成时） | （请求内已完成） | `ImportCompleted` 邮件+数据库通知 |
| 失败消息 | `flashFailures()` 立即 flash；SheetNotFoundException 不 report 但显示消息 | `Queue::boot()` 监听 `JobFailed` → `ImportFailed` 通知（有限定范围） |
| 行数统计 | 无 | `toArray()` 预处理 → `ImportCompleted` 通知 |
| 异常可见性 | 当前请求可见（含 SheetNotFoundException） | 异步邮件/数据库通知（仅限 ValidationException + ReadChunk + 正确 import 类） |

---

## 三、Map 阶段字段处理详解

根据 Maatwebsite Excel 执行顺序，`withValidator` 先于 `map` 执行。但 map 阶段处理的字段会影响后续 `model()` 阶段的行为。

Map 阶段执行顺序：**父类 `Import::map()`** → **子类 `Transactions::map()`**

### 1. 父类 Import::map() 通用处理
[Abstracts/Import.php#L47-L84](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L47-L84)

#### company_id
```php
$row['company_id'] = company_id();  // 直接取当前登录用户所属公司 ID
```

#### created_by
```php
$row['created_by'] = $this->getCreatedById($row);
```
[Traits/Import.php#L151-L164](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L151-L164)
- 优先通过 email 查找用户 ID（导出时 created_by 是 owner email）
- 查找失败或为空时，使用当前登录用户 ID `$this->user?->id`

#### created_from
```php
$row['created_from'] = $this->getSourcePrefix() . 'import';
```

[Traits/Sources.php#L47-L52](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php#L47-L52)
```php
public function getSourcePrefix($alias = null)
{
    $alias = is_null($alias) ? $this->getSourceAlias() : $alias;
    return $alias . '::';
}

public function getSourceAlias()
{
    $namespaces = explode('\\', get_class($this));
    if ($namespaces[0] != 'Modules') {
        return 'core';
    }
    return Str::kebab($namespaces[1]);
}
```
**真实格式**：对于 core 导入类，`created_from` = `"core::import"`

#### 日期字段处理
```php
$date_fields = ['paid_at', 'invoiced_at', 'billed_at', 'due_at', 'issued_at', 'transferred_at'];
foreach ($date_fields as $date_field) {
    if (!isset($row[$date_field])) continue;

    try {
        $row[$date_field] = is_numeric($row[$date_field]) 
            ? Date::parse(ExcelDate::excelToDateTimeObject($row[$date_field]))->format('Y-m-d H:i:s')
            : Date::parseWithFallbackLocales($row[$date_field], null, [$this->preferredLocale()])->format('Y-m-d H:i:s');
    } catch (InvalidFormatException | \Exception $e) {
        Log::info($e->getMessage());  // 仅记录日志，不中断
    }
}
```

**日期解析逻辑**：
1. 数字格式（Excel 序列号）：`ExcelDate::excelToDateTimeObject()` → `Date::parse()` → `Y-m-d H:i:s`
2. 字符串格式：`Date::parseWithFallbackLocales()` 多 locale 尝试
   - 优先使用用户偏好 locale（`$this->preferredLocale()` = `$this->user->locale`）
   - 回退链：用户 locale → App locale → Fallback locale → `'en'`
3. 全部解析失败：仅 `Log::info()` 记录，**不抛出异常**，`$row[$date_field]` 保留原始字符串

---

### 2. 子类 Transactions::map() 特定处理
[Imports/Banking/Transactions.php#L31-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L31-L51)

#### type 字段
```php
if (!isset($row['type'])) {
    return [];  // type 缺失返回空数组
}

$real_type = $this->getRealTypeTransaction($row['type']);
$contact_type = config('type.transaction.' . $real_type . '.contact_type', $real_type == 'income' ? 'customer' : 'vendor');
```

`getRealTypeTransaction` 处理复合类型后缀：
[Traits/Transactions.php#L220-L227](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php#L220-L227)
```php
public function getRealTypeTransaction(string $type): string
{
    $type = $this->getRealTypeOfRecurringTransaction($type);  // 去掉 -recurring
    $type = $this->getRealTypeOfTransferTransaction($type);   // 去掉 -transfer
    $type = $this->getRealTypeOfSplitTransaction($type);      // 去掉 -split
    return $type;
}
```

**注意**：`$row['type']` 在 map 阶段**不被修改**，`getRealTypeTransaction` 的结果仅用于推导 `$real_type` 和 `$contact_type`，后续的 `getDocumentId()` 使用的仍是原始 `$row['type']`。

#### account_id
```php
$row['account_id'] = $this->getAccountId($row);
```
[Traits/Import.php#L65-L82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L65-L82)

**查找条件**（优先级递减）：
1. `account_id` 存在时直接使用
2. `account_name` 存在时按名称查找 → `getAccountIdFromName()`
3. `account_number` 存在时按账号查找 → `getAccountIdFromNumber()`
4. `currency_code` 存在时按货币查找 → `getAccountIdFromCurrency()`

**自动创建条件**（精确）：
- **`getAccountIdFromName()`**：`Account::where('name', $row['account_name'])` 无结果 → 自动创建，必填：`company_id`, `name`；可选：`account_type`（默认 `bank`）、`account_number`、`currency_code`（默认 `default_currency()`）、`opening_balance`
- **`getAccountIdFromNumber()`**：`Account::where('account_number', $row['account_number'])` 无结果 → 自动创建，必填：`company_id`, `account_number`, `name`（缺则用 account_number）；可选同上
- **`getAccountIdFromCurrency()`**：`Account::where('currency_code', $row['currency_code'])` 无结果 → 自动创建，必填：`company_id`, `currency_code`, `name`（缺则用 currency_code）；可选同上

**不自动创建的情况**：如果只提供了 `account_id` 但该 ID 不存在，`getAccountId()` 直接返回 `null`，**不会触发自动创建**。

#### category_id
```php
$row['category_id'] = $this->getCategoryId($row, $real_type);
```
[Traits/Import.php#L84-L95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L84-L95)

**查找条件**：
1. `category_id` 存在时直接使用
2. `category_name` 存在时按名称查找（限定 `$real_type` 类型）→ `getCategoryIdFromName()`

**自动创建条件**（精确）：
- **`getCategoryIdFromName()`**：`Category::type($type)->withSubCategory()->where('name', $row['category_name'])` 无结果 → 自动创建，必填：`company_id`, `name`, `type`（由 `$real_type` 传入，无效类型回退到 `Category::OTHER_TYPE`）；可选：`category_color`

**不自动创建的情况**：只提供 `category_id` 但 ID 不存在时，返回 `null`，不创建。

#### contact_id
```php
$row['contact_id'] = $this->getContactId($row, $contact_type);
```
[Traits/Import.php#L102-L117](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L102-L117)

**查找条件**：
1. `contact_id` 存在时直接使用
2. `contact_email` 存在时按邮箱查找（限定 `$contact_type`）→ `getContactIdFromEmail()`
3. `contact_name` 存在时按名称查找（限定 `$contact_type`）→ `getContactIdFromName()`

**自动创建条件**（精确）：
- **`getContactIdFromEmail()`**：`Contact::type($type)->where('email', $row['contact_email'])` 无结果 → 自动创建，必填：`company_id`, `email`, `type`, `name`（缺则用 email）；可选：`contact_currency`
- **`getContactIdFromName()`**：`Contact::type($type)->where('name', $row['contact_name'])` 无结果 → 自动创建，必填：`company_id`, `name`, `type`；可选：`email`、`contact_currency`

**不自动创建的情况**：只提供 `contact_id` 但 ID 不存在时，返回 `null`，不创建。

#### document_id（只查找，不创建）
[Traits/Import.php#L166-L191](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L166-L191)

```php
public function getDocumentId($row)
{
    $id = isset($row['document_id']) ? $row['document_id'] : null;

    if (empty($id) && !empty($row['document_number'])) {
        $id = Document::number($row['document_number'])->pluck('id')->first();
    }
    if (empty($id) && !empty($row['invoice_number'])) {
        $id = Document::invoice()->number($row['invoice_number'])->pluck('id')->first();
    }
    if (empty($id) && !empty($row['bill_number'])) {
        $id = Document::bill()->number($row['bill_number'])->pluck('id')->first();
    }
    if (empty($id) && !empty($row['invoice_bill_number'])) {
        if ($row['type'] == Transaction::INCOME_TYPE) {
            $id = Document::invoice()->number($row['invoice_bill_number'])->pluck('id')->first();
        } else {
            $id = Document::bill()->number($row['invoice_bill_number'])->pluck('id')->first();
        }
    }

    return is_null($id) ? $id : (int) $id;
}
```

[Models/Banking/Transaction.php#L22](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L22-L22)
```php
public const INCOME_TYPE = 'income';
```

**查找条件**（优先级递减）：
1. `document_id` 直接使用
2. `document_number` → 不限类型搜索
3. `invoice_number` → 限定 invoice 类型搜索
4. `bill_number` → 限定 bill 类型搜索
5. `invoice_bill_number` → 根据原始 `$row['type']` 精确匹配 `Transaction::INCOME_TYPE`（`'income'`）决定搜索 invoice 或 bill

**关键细节**：`invoice_bill_number` 分支比较的是 `$row['type'] == 'income'`（精确匹配）。当 Excel 中 type 为 `income-recurring`、`income-transfer`、`income-split` 等带后缀类型时，会落入 else 分支按 bill 搜索，而非 invoice。

**document_id 永不自动创建**：所有分支均为纯查询，找不到返回 `null`，不会创建 Document 记录。

#### parent_id
```php
$row['parent_id'] = $this->getParentId($row) ?? 0;
```
[Traits/Import.php#L193-L214](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L193-L214)
- 只查找，不创建。查不到时返回 `null`，被 `?? 0` 转为 `0`

#### payment_method
```php
$row['payment_method'] = $this->getPaymentMethod($row);
```
[Traits/Import.php#L216-L240](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L216-L240)

**查找条件**：
1. `payment_method` 存在时检查是否为已注册的支付方式 code（`Modules::getPaymentMethods('all')`）
2. 已注册 → 直接返回 code
3. 未注册 → 检查 `offline-payments` 模块是否启用

**自动创建条件**（精确）：
- 未注册 + `module_is_enabled('offline-payments')` → 通过 `OfflinePayments\Jobs\CreatePaymentMethod` 创建
- 未注册 + 模块未启用 → 返回原始值（可能为未注册 code 或 null）

---

## 四、withValidator 复用 FormRequest rules 机制

### 执行顺序注意

**重要**：在 Maatwebsite Excel 3.1.67 中，`withValidator` **先于** `map()` 执行。`withValidator` 接收的是原始 Excel 行数据（经 `WithHeadingRow` 处理后），尚未经过 `map()` 的字段转换。

### 核心代码
[Abstracts/Import.php#L105-L143](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L105-L143)

```php
public function withValidator($validator)
{
    // 条件：request_class 存在且是 FormRequest 实例
    $condition = class_exists($this->request_class)
                ? ! ($request = new $this->request_class) instanceof FormRequest
                : true;

    if ($condition) {
        return;  // 不满足条件则跳过
    }

    foreach ($validator->getData() as $row => $data) {
        $request->initialize(request: $data);  // 用原始行数据初始化 FormRequest

        $rules = $this->prepareRules($request->rules());  // 获取 FormRequest rules 并加工

        try {
            Validator::make($data, $rules)->validate();  // 二次校验
        } catch (ValidationException $e) {
            // 将二次校验的错误合并到主 validator
            foreach ($e->validator->failed() as $attribute => $value) {
                foreach ($value as $rule => $params) {
                    if ($rule === 'In' && !empty($params)) {
                        $actual_value = $data[$attribute] ?? 'null';
                        $expected_values = implode(', ', $params);
                        $custom_message = trans('validation.in_detailed', [
                            'attribute' => $attribute,
                            'value' => $actual_value,
                            'values' => $expected_values
                        ]);
                        $validator->errors()->add($row . '.' . $attribute, $custom_message);
                    } else {
                        $validator->addFailure($row . '.' . $attribute, $rule, $params);
                    }
                }
            }

            throw new ValidationException($validator);
        }
    }
}
```

**工作原理**：
1. 遍历 batch 中的每行数据（原始数据）
2. 用行数据初始化 FormRequest 实例（覆盖请求上下文）
3. 调用 `$request->rules()` 获取完整 rules
4. 通过 `prepareRules()` 对 rules 做导入场景适配
5. 用新 rules 对行数据二次校验
6. 校验失败时将错误合并到主 validator

### prepareRules 覆盖 number/type 的代码事实

[Imports/Banking/Transactions.php#L53-L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L53-L60)
```php
public function prepareRules($rules): array
{
    $rules['number'] = 'required|string';
    $rules['type'] = 'required|string';
    return $rules;
}
```

原始 FormRequest 中：
[Http/Requests/Banking/Transaction.php#L40-L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L40-L42)
```php
'type' => 'required|string',
'number' => 'required|string|unique:transactions,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL',
```

**代码层面发生的变化**：

| 字段 | FormRequest 原始 rules | prepareRules 后 |
|------|----------------------|-----------------|
| `number` | `required\|string\|unique:transactions,NULL,null,id,company_id,{cid},deleted_at,NULL` | `required\|string` |
| `type` | `required\|string` | `required\|string` |

`number` 字段的 `unique` 约束被移除，`type` 字段的值不变。`prepareRules` 仅对这两个字段赋值，其余所有 rules（`account_id`, `paid_at`, `amount`, `currency_code`, `currency_rate`, `document_id`, `contact_id`, `category_id`, `payment_method` 等）保持不变。

---

## 五、unique number 在导入中如何通过 Company Scope、Soft Delete 和 Transaction Scope 生效

### FormRequest 中的 unique 规则参数展开

[Http/Requests/Banking/Transaction.php#L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L42-L42)
```php
'number' => 'required|string|unique:transactions,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL',
```

Laravel `unique` 规则语法：`unique:table,column,except,idColumn,whereColumn,whereValue,...`

参数展开（注意 `column` 位置是 `NULL`，Laravel 会自动从正在验证的字段名推断为 `number`）：
```
unique:transactions,NULL,null,id,company_id,{company_id},deleted_at,NULL
         ────────── ──── ──── ── ────────── ──────────── ────────── ────
         table      col  exc  id  whereCol1   whereVal1   whereCol2  whereVal2
```

含义：
- `table` = `transactions`
- `column` = `NULL`（Laravel 自动从验证字段名 `number` 推断）
- `except` = `null`（配合 `idColumn = id`，不排除任何记录，因为 id 不可能为 null）
- `idColumn` = `id`
- `whereColumn1` = `company_id`, `whereValue1` = `{company_id}` — 仅限当前公司
- `whereColumn2` = `deleted_at`, `whereValue2` = `NULL` — 排除已软删除的记录（即 `deleted_at IS NULL`）

### 导入中实际生效机制

由于 `prepareRules` 移除了 `unique` 约束，导入中 unique 检查**通过 `hasRow()` 实现**：

#### 步骤 1：预加载当前公司的 transaction number
[Abstracts/Import.php#L245-L260](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L245-L260)
```php
$this->has_row = $this->model::withoutEvents(function () {
    if ($this->with_trashed) {
        return $this->model::withTrashed()->get($this->columns)->each(...);
    } else {
        return $this->model::get($this->columns)->each(...);
    }
});
```

`$this->model` = `Transaction::class`，`$this->columns` = `['number']`，`$this->with_trashed` = `false`。

#### 步骤 2：Company 全局作用域自动过滤
[Scopes/Company.php#L21-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L21-L50)
```php
public function apply(Builder $builder, Model $model)
{
    $builder->where($table . '.company_id', '=', company_id());
}
```
`Abstracts\Model` 的 `$tenantable = true`（默认），所以 Company 全局作用域**自动生效**。查询结果只包含 `company_id = 当前公司` 的记录。

#### 步骤 3：SoftDeletes 自动过滤
`Abstracts\Model` 使用了 `SoftDeletes` trait，所有 Eloquent 查询自动排除 `deleted_at IS NOT NULL` 的记录。

#### 步骤 4：Transaction 全局作用域自动过滤

[Models/Banking/Transaction.php#L101-L104](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L101-L104)
```php
protected static function booted()
{
    static::addGlobalScope(new Scope);  // App\Scopes\Transaction
}
```

[Scopes/Transaction.php#L21-L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php#L21-L26)
```php
public function apply(Builder $builder, Model $model)
{
    $this->applyNotRecurringScope($builder, $model);
    $this->applyNotSplitScope($builder, $model);
}
```

[Traits/Scopes.php#L11-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php#L11-L31)
```php
public function applyNotRecurringScope(Builder $builder, Model $model): void
{
    if ($this->scopeColumnExists($builder, $model->getTable(), 'type')) {
        return;
    }
    $builder->isNotRecurring();  // type not like '%-recurring'
}

public function applyNotSplitScope(Builder $builder, Model $model): void
{
    if ($this->scopeColumnExists($builder, $model->getTable(), 'type')) {
        return;
    }
    $builder->isNotSplit();  // type not like '%-split'
}
```

**关键影响**：`hasRow()` 查询 `Transaction::get(['number'])` 时，Transaction 全局作用域自动生效，只查询 `type` 不含 `-recurring` 和 `-split` 后缀的记录。这意味着：
- recurring transaction 的 number **不会被** hasRow 用来去重
- split transaction 的 number **不会被** hasRow 用来去重
- 导入 recurring/split 类型的 transaction 时，hasRow 不会阻止重复

#### 步骤 5：按 columns 匹配
[Abstracts/Import.php#L262-L269](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L262-L269)
```php
$search_value = [];
foreach ($this->columns as $key) {
    $search_value[$key] = isset($row[$key]) ? $row[$key] : null;
}
return in_array($search_value, $this->has_row->toArray());
```

#### hasRow 边界：同一 Excel 内重复 number

`hasRow()` 的预加载在 `model()` 首次被调用时执行一次（`$this->has_row` 为空时），后续行复用内存中的集合。如果 Excel 中有两行相同的 `number`：
- 第一行：`hasRow()` 返回 false → 创建 Model 实例 → 写入 DB
- 第二行：`hasRow()` 仍返回 false（预加载集合未更新）→ 再次创建 Model 实例 → 写入 DB

由于 **transactions 表没有 database-level 的 unique 约束**（仅有普通索引），两行相同 number 会成功写入数据库，不会触发数据库异常。

**数据库迁移确认**：
- [2019_11_16_000000_core_v2.php#L387-L414](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L387-L414)：初始建表，只有 `$table->index(['company_id', 'type'])` 等普通索引，无 unique 约束
- [2022_05_10_000000_core_v300.php#L62-L68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_05_10_000000_core_v300.php#L62-L68)：添加 `number` 列，仅 `$table->string('number')`
- [2022_08_29_000000_core_v3015.php#L34-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_08_29_000000_core_v3015.php#L34-L36)：添加 `$table->index('number')`，仍是普通索引

三处迁移均未添加 `unique` 约束。

### 生效链路总结

```
hasRow() 被调用
    ↓
$this->model::get(['number'])
    ↓
Company 全局作用域 → WHERE company_id = 当前公司
    ↓
Transaction 全局作用域 → WHERE type NOT LIKE '%-recurring' AND type NOT LIKE '%-split'
    ↓
SoftDeletes → WHERE deleted_at IS NULL
    ↓
内存中 in_array() 匹配 → 已存在则跳过创建
```

**与 FormRequest unique 规则的对比**：

| 维度 | FormRequest unique | hasRow() |
|------|-------------------|----------|
| Company 限制 | 规则中显式 `company_id,{id}` | Company 全局作用域自动过滤 |
| Soft delete 限制 | 规则中显式 `deleted_at,NULL` | SoftDeletes trait 自动过滤 |
| 类型过滤 | 无（所有类型） | 自动排除 `-recurring` 和 `-split` 类型 |
| 同 Excel 重复 | 由 validator 逐行校验（unique 在 DB 层查询） | 不检测（预加载在导入开始时完成，集合不更新） |
| 数据库层约束 | 依赖 DB unique 约束 | **无 DB unique 约束**，重复写入不会异常 |
| 性能 | 每行一次 DB 查询 | 一次预加载，内存匹配 |

---

## 六、异常场景分析

### 场景 1：type 字段缺失

**测试行数据**：
```php
[
    'number' => 'TXN-001',
    'paid_at' => '2024-01-15',
    'amount' => 100.00,
    // 'type' 字段缺失
]
```

**处理流程**（Maatwebsite Excel 执行顺序：withValidator → map → model）：

1. **withValidator 阶段**：FormRequest 的 rules 要求 `type` 为 `required|string`。`prepareRules` 也将 `type` 设为 `required|string`。原始数据中缺少 `type` 字段 → **校验失败** → 抛出 `ValidationException`

2. 如果 somehow 通过了 withValidator（理论上不可能），进入 **map 阶段**：
   ```php
   if (!isset($row['type'])) {
       return [];  // 返回空数组
   }
   ```
3. `SkipsEmptyRows` 检查：map 返回 `[]` → 该行被跳过，不进入 `model()`

**最终结果**：withValidator 阶段校验失败，产生错误消息（同步模式 flash，队列模式走 ImportFailed 通知）。map 返回空数组是兜底保护。

---

### 场景 2：日期格式不符合 locale

**测试行数据**（用户 locale 为 `zh_CN`，Excel 中日期为 `15/01/2024`）：
```php
[
    'number' => 'TXN-002',
    'type' => 'income',
    'paid_at' => '15/01/2024',
    'amount' => 100.00,
    'account_id' => 1,
    'category_id' => 1,
    'payment_method' => 'cash',
    'currency_code' => 'USD',
    'currency_rate' => 1,
]
```

**处理流程**：

1. **withValidator 阶段**：FormRequest 中 `paid_at` 规则为 `required|date_format:Y-m-d H:i:s`（原始数据中的 `paid_at` 为 `'15/01/2024'`）→ 不匹配 → **校验失败**

2. 即使 somehow 通过了 withValidator，进入 **map 阶段**，父类处理日期：
   ```php
   Date::parseWithFallbackLocales('15/01/2024', null, ['zh_CN'])
   ```
   尝试 locale 链：`zh_CN` → `zh` → App locale → fallback locale → `en`
   全部失败 → `Log::info($e->getMessage())` 记录日志，`$row['paid_at']` 保留原始值

**最终结果**：withValidator 阶段校验失败。

**同步模式**：`flashFailures()` flash 错误消息：
```
Error: The paid at does not match the format Y-m-d H:i:s. Column name: paid_at. Line number: 2.
```

**队列模式**：如果失败的是 `ReadChunk` 任务且异常是 `ValidationException`，通过 `ImportFailed` 邮件+数据库通知用户。

---

## 七、关键代码文件索引

| 文件 | 作用 |
|------|------|
| [routes/admin.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php) | L139/L148 定义 transactions 导入路由 |
| [Providers/Route.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Route.php) | L228-L234 mapAdminRoutes() 加 `{company_id}` 前缀；L306-L308 import 速率限制 |
| [Controllers/Banking/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php) | L195-L210 `import()` 方法 |
| [Abstracts/Http/Controller.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php) | L85-L88 `importExcel()` 委托 |
| [Utilities/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php) | 导入入口，L42-L45 SheetNotFoundException 不 report；同步/队列分发，成功/失败消息处理 |
| [Imports/Banking/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php) | Transactions 导入类，map/model/prepareRules |
| [Abstracts/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php) | 导入抽象基类，map/withValidator/hasRow |
| [Http/Requests/Banking/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php) | L40-L42 定义 type 和 number 校验规则 |
| [Http/Requests/Common/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/Import.php) | ImportRequest 文件校验 |
| [Traits/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php) | 字段解析辅助方法（L65-L82 account, L84-L95 category, L102-L117 contact, L166-L191 document 只查不建, L216-L240 payment_method） |
| [Traits/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php) | Transaction 类型处理 |
| [Traits/Sources.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php) | created_from 格式（getSourcePrefix） |
| [Traits/Scopes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php) | NotRecurring/NotSplit 全局作用域 |
| [Models/Banking/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php) | L101-L104 添加 Transaction 全局作用域 |
| [Scopes/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php) | 应用 NotRecurring + NotSplit 作用域 |
| [Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) | 模型基类，SoftDeletes + Company scope |
| [Scopes/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php) | Company 全局作用域 |
| [Providers/Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php) | L82-L127 监听 JobFailed → ImportFailed 通知（ValidationException + ReadChunk + 正确 import 类） |
| [Notifications/Common/ImportCompleted.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Common/ImportCompleted.php) | 队列导入完成通知 |
| [Notifications/Common/ImportFailed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Common/ImportFailed.php) | 队列导入失败通知 |
| [Http/Kernel.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php) | L136-L138 `import` 中间件 = `throttle:import` |
| [resources/lang/en-GB/messages.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/lang/en-GB/messages.php) | L11-L12 成功消息；L35 失败消息 |
| [2019_11_16_000000_core_v2.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php) | L387-L414 transactions 表初始结构，无 unique 约束 |
| [2022_05_10_000000_core_v300.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_05_10_000000_core_v300.php) | L62-L68 添加 number 列，无 unique 约束 |
| [2022_08_29_000000_core_v3015.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_08_29_000000_core_v3015.php) | L34-L36 添加 number 普通索引，无 unique 约束 |
