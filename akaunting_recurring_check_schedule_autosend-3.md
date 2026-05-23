# Akaunting recurring:check 命令深度分析

## 概述

`recurring:check` 是 Akaunting 中用于自动生成周期性发票/账单/收入/支出的核心调度命令。该命令通过遍历所有活跃的 Recurring 记录，根据调度规则生成真实的业务模型（Document 或 Transaction）。

核心代码位置：[RecurringCheck.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php)

---

## 1. Recurring 记录的删除、跳过、标记 Completed 条件

### 1.1 删除条件

Recurring 记录在以下情况下会被删除：

| 场景 | 触发位置 | 条件 |
|------|---------|------|
| 缺少关联公司 | [RecurringCheck.php#L62-L68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L62-L68) | `empty($recur->company)` 为 true |
| 公司已禁用且近期更新 | [RecurringCheck.php#L77-L89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L77-L89) | 公司 `enabled = false` 且 `updated_at` 在 3 个月内 |
| 公司无活跃用户 | [RecurringCheck.php#L92-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L92-L112) | 所有用户 `last_logged_in_at` 都超过 3 个月 |
| 缺少模板模型 | [RecurringCheck.php#L116-L122](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L116-L122) | `$recur->recurable()` 查询返回 null |

**注意**：公司禁用或无活跃用户时，除了删除 Recurring 记录，还会同时删除关联的模板模型（`$template->delete()`）。

### 1.2 跳过条件

| 场景 | 触发位置 | 条件 |
|------|---------|------|
| 所有调度已完成 | [RecurringCheck.php#L130-L136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L130-L136) | `getRemainingSchedules()` 返回空集合 |
| 今日无待执行调度 | [RecurringCheck.php#L141-L145](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L141-L145) | `endsBefore(tomorrow)` 过滤后为空 |
| 模型创建失败 | [RecurringCheck.php#L167-L169](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L167-L169) | `getModel()` 抛出异常返回 false |

### 1.3 标记 Completed 条件

| 场景 | 触发位置 | 条件 |
|------|---------|------|
| 所有调度已创建 | [RecurringCheck.php#L130-L136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L130-L136) | `getRemainingSchedules()` 返回空集合，设置 `status = 'completed'` |

常量定义：[Recurring.php#L15](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Recurring.php#L15)

```php
public const COMPLETE_STATUS = 'completed';
```

---

## 2. getRemainingSchedules 去重机制与日期字段差异

### 2.1 避免重复创建的机制

[getRemainingSchedules()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L243-L262) 通过以下步骤避免重复创建：

```php
protected function getRemainingSchedules(Document|Transaction $template, Recurring $recur): RecurrenceCollection
{
    $date_field = $this->getDateField($template);

    // 查询已创建的子模型日期
    $created_schedules = DB::table($template->getTable())
                            ->where('type', $this->getRealType($template->type))
                            ->where('parent_id', $template->id)
                            ->get($date_field)
                            ->transform(function ($item, $key) use ($date_field) {
                                return Date::parse($item->$date_field)->format('Y-m-d');
                            })
                            ->toArray();

    // 过滤掉已创建的调度
    $schedules = $recur->getRecurringSchedule()->filter(function ($recurrence) use ($created_schedules) {
        return ! in_array($recurrence->getStart()->format('Y-m-d'), $created_schedules);
    });

    return $schedules;
}
```

**核心去重逻辑：
1. 查询所有 `parent_id = 模板ID` 且 `type = 真实类型` 的子模型
2. 提取这些子模型的日期字段值（`Y-m-d` 格式）
3. 从完整调度集合中过滤掉日期已存在的调度

### 2.2 Document 与 Transaction 使用不同日期字段的原因

| 模型类型 | 日期字段 | 触发位置 | 原因 |
|---------|---------|---------|------|
| Document | `issued_at` | [RecurringCheck.php#L266](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L266) | 发票/账单的核心日期是"签发日期"，代表业务发生时间 |
| Transaction | `paid_at` | [RecurringCheck.php#L266](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L266) | 收入/支出的核心日期是"支付日期"，代表资金变动时间 |

```php
protected function getDateField(Document|Transaction $template): string
{
    return ($template instanceof Transaction) ? 'paid_at' : 'issued_at';
}
```

---

## 3. 生成模型时的字段与关联变化

### 3.1 Document 模型（invoice/bill）

生成逻辑：[getDocumentModel()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L205-L224)

| 字段 | 变化规则 | 示例 |
|-----|---------|------|
| `type` | 去除 `-recurring` 后缀 | `invoice-recurring` → `invoice` |
| `parent_id` | 设置为模板 ID | `$template->id` |
| `issued_at` | 设置为调度日期 | `$schedule_date->format('Y-m-d')` |
| `due_at` | 调度日期 + 模板的签发日到到期日的天数差 | `$schedule_date->copy()->addDays($diff_days)` |
| `created_from` | 固定为 `core::recurring` | `core::recurring` |

**关联关系处理：**
- `cloneable_relations = ['items', 'totals']`
- 复制后调用 `updateRelationTypes()` 更新关联模型的 `type` 字段

### 3.2 Transaction 模型（income/expense）

生成逻辑：[getTransactionModel()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L226-L241)

| 字段 | 变化规则 | 示例 |
|-----|---------|------|
| `type` | 去除 `-recurring` 后缀 | `expense-recurring` → `expense` |
| `parent_id` | 设置为模板 ID | `$template->id` |
| `paid_at` | 设置为调度日期 | `$schedule_date->format('Y-m-d')` |
| `created_from` | 固定为 `core::recurring` | `core::recurring` |

**关联关系处理：**
- `cloneable_relations = ['taxes']`
- 复制后调用 `updateRelationTypes()` 更新关联模型的 `type` 字段

### 3.3 类型转换方法

[getRealType()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L269-L272)：

```php
public function getRealType(string $recurring_type): string
{
    return Str::replace('-recurring', '', $recurring_type);
}
```

### 3.4 关联类型更新

[updateRelationTypes()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L274-L283)：

```php
public function updateRelationTypes($model, $relations)
{
    foreach ($relations as $relation) {
        if (! method_exists($model, $relation)) {
            continue;
        }

        $model->$relation()->update(['type' => $model->type]);
    }
}
```

**关联模型的 `type` 会同步更新为新的真实类型（如 `invoice-recurring` → `invoice`）。

### 3.5 模板类型与生成类型对照表

| 模板类型 | 生成类型 | 模型 |
|---------|---------|------|
| `invoice-recurring` | `invoice` | Document |
| `bill-recurring` | `bill` | Document |
| `income-recurring` | `income` | Transaction |
| `expense-recurring` | `expense` | Transaction |

---

## 4. 事件触发与自动发送/标记逻辑

### 4.1 事件触发位置

所有事件在 [recur()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L164-L188) 方法中触发：

```php
protected function recur(Document|Transaction $template, Date $schedule_date): void
{
    DB::transaction(function () use ($template, $schedule_date) {
        if (! $model = $this->getModel($template, $schedule_date)) {
            return;
        }

        switch ($template::class) {
            case Document::class:
                event(new DocumentCreated($model, request()));
                event(new DocumentRecurring($model));
                break;
            case Transaction::class:
                event(new TransactionCreated($model));
                event(new TransactionRecurring($model));
                break;
        }
    });
}
```

### 4.2 事件监听器注册

事件监听器在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php) 中注册：

| 事件 | 监听器 | 作用 |
|------|---------|------|
| `DocumentCreated` | `CreateDocumentCreatedHistory` | 创建历史记录 |
| `DocumentCreated` | `IncreaseNextDocumentNumber` | 增加文档编号 |
| `DocumentCreated` | `SettingFieldCreated` | 创建设置字段 |
| `DocumentRecurring` | `SendDocumentRecurringNotification` | 发送 recurring 通知 |
| `DocumentSent` | `MarkDocumentSent` | 标记为已发送 |
| `DocumentReceived` | `MarkDocumentReceived` | 标记为已接收 |
| `TransactionCreated` | `IncreaseNextTransactionNumber` | 增加交易编号 |

### 4.3 自动发送邮件条件

自动发送逻辑在 [SendDocumentRecurringNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php) 中：

```php
public function handle(Event $event)
{
    $document = $event->document;
    $config = config('type.document.' . $document->type . '.notification');

    // 条件1: 必须有通知配置
    if (empty($config) || empty($config['class'])) {
        return;
    }

    // 条件2: 模板的 recurring.auto_send 不能为 false
    if ($document->parent?->recurring?->auto_send == false) {
        return;
    }

    // 发送客户通知
    if ($this->canNotifyTheContactOfDocument($document)) {
        $document->contact->notify(new $notification($document, "{$document->type}_recur_customer", $attach_pdf));
    }

    // 触发标记事件
    $sent = config('type.document.' . $document->type . '.auto_send', DocumentSent::class);
    event(new $sent($document));

    // 发送用户通知
    if ($config['notify_user']) {
        foreach ($document->company->users as $user) {
            $user->notify(new $notification($document, "{$document->type}_recur_admin"));
        }
    }
}
```

### 4.4 自动标记 sent/received 的条件

标记逻辑：

| 文档类型 | 自动标记事件 | 配置位置 |
|---------|-------------|---------|
| invoice | `DocumentSent` | [type.php#L164](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L164) |
| invoice-recurring | `DocumentSent` | [type.php#L214](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L214) |
| bill | `DocumentReceived` | [type.php#L268](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L268) |
| bill-recurring | `DocumentReceived` | [type.php#L317](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L317) |

**标记状态逻辑（以 DocumentSent 为例）：
[MarkDocumentSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentSent.php#L14-L25)

```php
public function handle(DocumentMarkedSent|DocumentSent $event): void
{
    // 仅在非 partial/paid 状态下标记
    if (! in_array($event->document->status, ['partial', 'paid'])) {
        $event->document->status = 'sent';

        // 金额为0直接标记为paid
        if ($event->document->amount == 0) {
            $event->document->status = 'paid';
        }

        $event->document->save();
    }
}
```

**Transaction 没有自动标记逻辑，只有编号自增逻辑。

---

## 5. "上次任务失败后今天补建历史日期"执行路径

### 5.1 场景描述

假设：
- 有一个每日 recurring 发票，启动日期为 2026-05-15
- 2026-05-17 的任务执行失败（如服务器宕机）
- 今天是 2026-05-19，任务重新执行

### 5.2 具体执行路径

```
2026-05-19 执行 recurring:check
├─ 遍历 Recurring 记录（status=active
│  ├─ 检查公司状态（enabled=true）
│  ├─ 检查活跃用户（存在3个月内登录用户）
│  ├─ 获取模板（invoice-recurring）
│  ├─ getRemainingSchedules() 调用
│  │  ├─ 查询已创建的子模型
│  │  │  └─ WHERE parent_id = 模板ID
│  │  │  └─ WHERE type = 'invoice'
│  │  │  └─ 假设只查到 2026-05-15, 2026-05-16（17号失败未创建）
│  │  │  └─ 返回已创建日期数组: ['2026-05-15', '2026-05-16']
│  │  ├─ 获取完整调度集合（假设到今天为止应有:
│  │  │  2026-05-15, 2026-05-16, 2026-05-17, 2026-05-18, 2026-05-19
│  │  └─ filter() 过滤已创建日期
│  │     └─ 剩余: 2026-05-17, 2026-05-18, 2026-05-19
│  ├─ endsBefore(明天) 过滤（明天是2026-05-20）
│  │  └─ 保留: 2026-05-17, 2026-05-18, 2026-05-19
│  └─ 遍历剩余调度
│     ├─ 2026-05-17（补建历史）
│     │  ├─ getDocumentModel()
│     │  │  ├─ duplicate() 复制模板
│     │  │  ├─ type: invoice-recurring → invoice
│     │  │  ├─ parent_id = 模板ID
│     │  │  ├─ issued_at = 2026-05-17
│     │  │  ├─ due_at = 2026-05-17 + diff_days
│     │  │  ├─ created_from = 'core::recurring'
│     │  │  └─ updateRelationTypes() 更新 items/totals 的 type
│     │  ├─ event(DocumentCreated)
│     │  └─ event(DocumentRecurring)
│     │     └─ SendDocumentRecurringNotification
│     │        ├─ 检查 auto_send（true）
│     │        ├─ 发送客户通知
│     │        └─ event(DocumentSent)
│     │        └─ MarkDocumentSent
│     │           └─ status = 'sent'
│     ├─ 2026-05-18（补建历史）
│     │  └─ 同上流程
│     └─ 2026-05-19（今日）
│        └─ 同上流程
└─ 完成
```

### 5.3 关键代码验证：

1. **失败恢复机制**：`getRemainingSchedules() 通过查询已创建的子模型来判断哪些调度需要补建，不依赖上次执行状态。

2. **历史日期补建：只要 `endsBefore(tomorrow) 会包含所有早于明天的未执行调度。

3. **去重保证：每次执行都会检查已存在的子模型，避免重复创建。

4. **自动发送：补建的历史日期文档也会触发完整的事件链，包括邮件通知和状态标记。

### 5.4 测试验证

测试用例：[RecurringCheckTest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Commands/RecurringCheckTest.php)

```php
// 测试场景：启动日期为7天前，设置limit_count=20
// 执行 recurring:check 后应创建7个发票（包括历史日期全部补建）
public function testItShouldCreateCorrectNumberOfRecurringInvoicesByCount(): void
{
    $this->dispatch(new CreateDocument($this->getInvoiceRequest('count')));
    Date::setTestNow(Date::today()));
    $this->artisan('recurring:check'));
    $this->assertDatabaseCount('documents', $this->recurring_count + 1)); // 7 + 1模板
}
```

---

## 总结

`recurring:check 命令的设计体现了几个关键特性：

1. **健壮的错误恢复**：通过查询已创建的子模型而非记录来判断调度状态，实现了自然的失败恢复机制
2. **清晰的类型转换**：模板与生成模型通过 `type` 字段区分，`-recurring` 后缀实现模板与真实模型的隔离
3. **完整的事件驱动**：通过事件系统实现自动发送、状态标记等附加功能
4. **灵活的调度过滤**：`endsBefore(tomorrow) 确保只创建今天及之前的调度，避免提前创建
5. **关联同步**：自动同步关联模型的 `type` 字段保证数据一致性
