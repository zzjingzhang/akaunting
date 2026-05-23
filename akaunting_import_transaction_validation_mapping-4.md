# Akaunting Banking Transactions Excel 导入完整路径分析

## 一、整体流程概览

```
Excel 文件上传
    ↓
ImportController::create() 显示导入表单
    ↓
提交表单 → Import::fromExcel() 入口 [Utilities/Import.php]
    ↓
判断 should_queue() → 同步 or 队列
    ↓
Transactions 导入类处理 [Imports/Banking/Transactions.php]
    ├─ map() 阶段：字段映射与转换
    ├─ withValidator() 阶段：复用 FormRequest 校验
    └─ model() 阶段：创建 Model 实例
```

---

## 二、同步导入 vs 队列导入的差异

### 核心判断逻辑
[Utilities/Import.php#L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L28-L28)
```php
$should_queue = should_queue();  // config('queue.default') != 'sync'
```

### 1. 执行路径差异

| 维度 | 同步导入 | 队列导入 |
|------|---------|---------|
| 触发方式 | `$class->import($file)` 直接执行 | `$class->queue($file)->onQueue('imports')` 推送到队列 |
| 完成时机 | 同步等待，请求结束时完成 | 异步执行，通过通知告知完成 |
| 异常处理 | 同步捕获，直接 flash 到 session | 队列异常由队列系统处理 |

### 2. 成功消息差异

**同步导入成功消息**：
[Utilities/Import.php#L38-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L38-L41)
```php
$message = trans('messages.success.imported', ['type' => $translation]);
// 翻译后示例："Transactions imported successfully."
```

**队列导入成功消息**：
```php
$message = trans('messages.success.import_queued', ['type' => $translation]);
// 翻译后示例："Transactions queued for import."
```

### 3. 失败消息差异

两种模式共享同一失败处理方法 `flashFailures()`：
[Utilities/Import.php#L82-L99](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L82-L99)

```php
foreach ($e->failures() as $failure) {
    $message = trans('messages.error.import_column', [
        'message'   => collect($failure->errors())->first(),
        'column'    => $failure->attribute(),
        'line'      => $failure->row(),
    ]);
    flash($message)->error()->important();
}
```

**关键区别**：
- 同步导入：失败时立即在当前请求中 flash 错误消息
- 队列导入：队列任务失败不会触发此方法，错误由队列日志记录

### 4. 通知总行数计算（仅队列导入）

队列导入时，在推送任务前预先统计行数用于通知：
[Utilities/Import.php#L67-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php#L67-L79)

```php
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
```

**计算逻辑**：
1. 先调用 `toArray()` 将 Excel 转为数组
2. 单 sheet 时取 `$rows[0]` 的 count
3. 多 sheet 时取第一个 sheet 的行数
4. 此行数在 `ImportCompleted` 通知中展示给用户

通知内容示例：
[Notifications/Common/ImportCompleted.php#L69-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Common/ImportCompleted.php#L69-L72)
```php
'description' => trans('notifications.menu.import_completed.description', [
    'type'  => $this->translation,
    'count' => $this->total_rows,
]);
// 翻译后示例："5 Transactions imported successfully."
```

---

## 三、Map 阶段字段处理详解

Map 阶段分为两层：**父类通用处理** + **子类特定处理**

### 1. 父类 Import::map() 通用处理
[Abstracts/Import.php#L47-L84](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L47-L84)

#### company_id
```php
$row['company_id'] = company_id();  // 直接取当前公司 ID
```

#### created_by
```php
// created_by 导出时是 owner email，导入时需要反向解析
$row['created_by'] = $this->getCreatedById($row);
```
[Traits/Import.php#L151-L164](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L151-L164)
- 优先通过 email 查找用户 ID
- 查找失败或为空时使用当前登录用户 ID

#### created_from
```php
$row['created_from'] = $this->getSourcePrefix() . 'import';
```
- 格式：`{source_prefix}import`，如 `api_import`

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
1. 数字格式：使用 `ExcelDate::excelToDateTimeObject()` 解析 Excel 序列号
2. 字符串格式：使用 `Date::parseWithFallbackLocales()` 多 locale 尝试解析
3. 解析失败：仅 `Log::info()` 记录，**不抛出异常，继续处理**

---

### 2. 子类 Transactions::map() 特定处理
[Imports/Banking/Transactions.php#L31-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L31-L51)

#### type 字段
```php
if (!isset($row['type'])) {
    return [];  // ⚠️ type 缺失直接返回空数组，该行被跳过
}

$real_type = $this->getRealTypeTransaction($row['type']);
$contact_type = config('type.transaction.' . $real_type . '.contact_type', $real_type == 'income' ? 'customer' : 'vendor');
```

`getRealTypeTransaction` 处理复合类型：
[Traits/Transactions.php#L220-L227](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php#L220-L227)
```php
public function getRealTypeTransaction(string $type): string
{
    $type = $this->getRealTypeOfRecurringTransaction($type);  // 去掉 -recurring 后缀
    $type = $this->getRealTypeOfTransferTransaction($type);   // 去掉 -transfer 后缀
    $type = $this->getRealTypeOfSplitTransaction($type);      // 去掉 -split 后缀
    return $type;
}
```

#### account_id
```php
$row['account_id'] = $this->getAccountId($row);
```
[Traits/Import.php#L65-L82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L65-L82)
- 优先级：`account_id` → `account_name` → `account_number` → `currency_code`
- 不存在时自动创建 Account 记录

#### category_id
```php
$row['category_id'] = $this->getCategoryId($row, $real_type);
```
[Traits/Import.php#L84-L95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L84-L95)
- 优先级：`category_id` → `category_name`
- 不存在时自动创建 Category 记录

#### contact_id
```php
$row['contact_id'] = $this->getContactId($row, $contact_type);
```
[Traits/Import.php#L102-L117](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L102-L117)
- 优先级：`contact_id` → `contact_email` → `contact_name`
- 不存在时自动创建 Contact 记录

#### document_id
```php
$row['document_id'] = $this->getDocumentId($row);
```
[Traits/Import.php#L166-L191](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L166-L191)
- 优先级：`document_id` → `document_number` → `invoice_number`/`bill_number` → `invoice_bill_number`

#### payment_method
```php
$row['payment_method'] = $this->getPaymentMethod($row);
```
[Traits/Import.php#L216-L240](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L216-L240)
- 先检查是否为已注册的支付方式
- 不存在且 `offline-payments` 模块启用时自动创建离线支付方式

---

## 四、withValidator 复用 FormRequest rules 机制

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
        $request->initialize(request: $data);  // ⚠️ 用行数据初始化 FormRequest

        $rules = $this->prepareRules($request->rules());  // 获取并处理 rules

        try {
            Validator::make($data, $rules)->validate();  // 二次校验
        } catch (ValidationException $e) {
            // 错误收集与转换...
            throw new ValidationException($validator);
        }
    }
}
```

### 为什么 prepareRules 会覆盖 number/type

在 `Transactions` 导入类中：
[Imports/Banking/Transactions.php#L53-L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L53-L60)

```php
public function prepareRules($rules): array
{
    $rules['number'] = 'required|string';
    $rules['type'] = 'required|string';
    return $rules;
}
```

**原因分析**：

原始 FormRequest 中的 rules 依赖请求上下文：
[Http/Requests/Banking/Transaction.php#L16-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L16-L79)

```php
public function rules()
{
    $type = $this->request->get('type', Model::INCOME_TYPE);
    $type = config('type.transaction.' . $type . '.route.parameter');

    // PATCH/PUT 时获取 ID 用于 unique 规则
    if (in_array($this->getMethod(), ['PATCH', 'PUT'])) {
        $model = $this->isApi() ? 'transaction' : $type;
        $id = is_numeric($this->$model) ? $this->$model : $this->{$model}->getAttribute('id');
    } else {
        $id = null;
    }

    $company_id = (int) $this->request->get('company_id', company_id());

    $rules = [
        'number' => 'required|string|unique:transactions,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL',
        'type' => 'required|string',
        // ...
    ];

    return $rules;
}
```

**问题**：
1. FormRequest 的 `rules()` 依赖 `$this->getMethod()` 判断是创建还是更新
2. Import 场景下 `getMethod()` 可能不是 PATCH/PUT，导致 `$id = null`
3. unique 规则中的模型路由绑定（`$this->$model`）在导入场景下失效
4. 复杂的上下文依赖可能导致规则构建异常

**解决方案**：
`prepareRules` 简化了 `number` 和 `type` 的规则，移除了依赖上下文的 unique 规则，只保留最基本的必填和字符串校验。

---

## 五、unique number 的 company/deleted_at 限制

### 限制定义位置
FormRequest 中的完整 unique 规则：
[Http/Requests/Banking/Transaction.php#L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L42-L42)

```php
'number' => 'required|string|unique:transactions,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL',
```

### 规则解析（Laravel unique 规则语法）
```
unique:table,column,except,idColumn,extraWhere,extraValue,...

unique:transactions,NULL,'null',id,company_id,{company_id},deleted_at,NULL
```

**含义**：在 `transactions` 表中，`number` 字段必须唯一，排除条件：
- `id` 等于 `null`（创建时不排除任何记录）
- 附加 where 条件：
  - `company_id = {company_id}` - 仅限当前公司
  - `deleted_at IS NULL` - 排除已软删除的记录

### 在导入流程中如何生效

**注意**：由于 `prepareRules` 覆盖了 number 规则，导入时的 unique 校验**不在 withValidator 阶段进行**。

实际生效位置在 `model()` 方法的 `hasRow()` 检查：
[Imports/Banking/Transactions.php#L22-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L22-L29)

```php
public function model(array $row)
{
    if (self::hasRow($row)) {
        return;  // 已存在则返回 null，不创建
    }

    return new Model($row);
}
```

`hasRow()` 实现：
[Abstracts/Import.php#L234-L270](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L234-L270)

```php
public function hasRow($row)
{
    // 预加载所有记录到内存（性能优化，避免每行查询）
    if (! $this->has_row || ! $this->has_row instanceof $this->model) {
        $this->has_row = $this->model::withoutEvents(function () {
            if ($this->with_trashed) {
                return $this->model::withTrashed()->get($this->columns);
            } else {
                return $this->model::get($this->columns);  // 默认不含软删除
            }
        });
    }

    // 按配置的 columns 构建搜索条件
    $search_value = [];
    foreach ($this->columns as $key) {
        $search_value[$key] = isset($row[$key]) ? $row[$key] : null;
    }

    return in_array($search_value, $this->has_row->toArray());
}
```

Transactions 导入类配置的 unique 列：
[Imports/Banking/Transactions.php#L18-L20](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php#L18-L20)
```php
public $columns = [
    'number',
];
```

**限制生效机制**：
1. `$columns = ['number']` 定义了按 `number` 字段去重
2. `hasRow()` 预加载所有未软删除的 transaction number
3. 每行检查 number 是否已存在，存在则跳过创建
4. 默认 `$with_trashed = false`，所以已软删除的记录不会被检查到（可以重复导入已删除的 number）

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

**处理流程**：
1. 父类 `map()` 正常处理，添加 company_id、created_by 等
2. 进入子类 `map()`：
   ```php
   if (!isset($row['type'])) {
       return [];  // 返回空数组
   }
   ```
3. 返回空数组后，Maatwebsite Excel 会**跳过该行**
4. 不会进入 withValidator 校验
5. 不会创建 Model 实例
6. **不会产生任何错误消息**，静默跳过

**最终结果**：返回空行，静默跳过，无错误提示。

---

### 场景 2：日期格式不符合 locale

**测试行数据**（locale 为 zh-CN，日期格式错误）：
```php
[
    'number' => 'TXN-002',
    'type' => 'income',
    'paid_at' => '15/01/2024',  // 非标准格式，假设 locale 不支持
    'amount' => 100.00,
    'account_id' => 1,
    'category_id' => 1,
    'payment_method' => 'cash',
    'currency_code' => 'USD',
    'currency_rate' => 1,
]
```

**处理流程**：
1. 父类 `map()` 处理日期字段：
   ```php
   try {
       $row[$date_field] = Date::parseWithFallbackLocales($row[$date_field], null, [$this->preferredLocale()])
           ->format('Y-m-d H:i:s');
   } catch (InvalidFormatException | \Exception $e) {
       Log::info($e->getMessage());  // 仅记录日志
   }
   ```
2. `parseWithFallbackLocales` 会尝试多个 locale 解析：
   - 用户偏好 locale
   - App locale
   - Fallback locale
   - 'en'
3. 所有 locale 都解析失败时，`paid_at` 保持原始值（未转换）
4. 进入 withValidator 阶段，FormRequest 规则要求：
   ```php
   'paid_at' => 'required|date_format:Y-m-d H:i:s',
   ```
5. 校验失败，抛出 ValidationException
6. 错误被 `flashFailures()` 捕获，flash 错误消息

**最终结果**：校验失败，显示错误消息：
```
Line 3: The paid at does not match the format Y-m-d H:i:s.
```

---

## 七、关键代码文件索引

| 文件 | 作用 |
|------|------|
| [Utilities/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Import.php) | 导入入口，同步/队列分发 |
| [Imports/Banking/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Banking/Transactions.php) | Transactions 导入类 |
| [Abstracts/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php) | 导入抽象基类，map/withValidator 核心逻辑 |
| [Traits/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php) | 字段解析辅助方法 |
| [Http/Requests/Banking/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php) | FormRequest 校验规则 |
| [Traits/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php) | Transaction 类型处理 |
| [Notifications/Common/ImportCompleted.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Common/ImportCompleted.php) | 队列导入完成通知 |
