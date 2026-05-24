# Akaunting recurring:check 命令深度分析

## 概述

`recurring:check` 是 Akaunting 中用于自动生成周期性发票/账单/收入/支出的核心调度命令。该命令通过遍历所有活跃的 Recurring 记录，根据调度规则生成真实的业务模型（Document 或 Transaction）。

核心代码位置：[RecurringCheck.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php)

---

## 1. Recurring 记录的删除、跳过、标记 Completed 条件

### 1.1 handle() 中每个 Recurring 记录的完整处理流程

[handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L40-L162) 对每个 `$recur` 执行以下检查，按顺序：

#### 检查 A：缺少公司 → 删除 Recurring

```php
// RecurringCheck.php#L62-L68
if (empty($recur->company)) {
    $recur->delete();
    continue;
}
```

条件：`$recur->company` 为 null。动作：直接 `$recur->delete()`，不删除 template。

#### 检查 B：公司已禁用 → 可能删除，也可能仅跳过

```php
// RecurringCheck.php#L77-L89
if (! $recur->company->enabled) {
    // 仅在 company.updated_at 晚于 now()-3个月 时才删除
    if (Date::parse($recur->company->updated_at)->format('Y-m-d') > Date::now()->subMonth(3)->format('Y-m-d')) {
        $recur->delete();
        if ($template) {
            $template->delete();
        }
    }
    // 无论是否删除，都 continue 跳过
    continue;
}
```

条件分支：
- `company.enabled = false` **且** `company.updated_at > now()->subMonth(3)`：删除 `$recur` 和 `$template`
- `company.enabled = false` **且** `company.updated_at <= now()->subMonth(3)`：仅 `continue`，不删除任何东西

#### 检查 C：公司无活跃用户 → 删除 Recurring 和 Template

```php
// RecurringCheck.php#L92-L112
$has_active_users = false;
foreach ($recur->company->users as $company_user) {
    if (Date::parse($company_user->last_logged_in_at)->format('Y-m-d') > Date::now()->subMonth(3)->format('Y-m-d')) {
        $has_active_users = true;
        break;
    }
}

if (! $has_active_users) {
    $recur->delete();
    if ($template) {
        $template->delete();
    }
    continue;
}
```

条件：所有用户的 `last_logged_in_at` 都早于 `now()->subMonth(3)`。动作：删除 `$recur` 和 `$template`。

#### 检查 D：缺少模板模型 → 仅删除 Recurring

```php
// RecurringCheck.php#L116-L122
if (! $template) {
    $recur->delete();
    continue;
}
```

条件：`$recur->recurable()->where('company_id', ...)->first()` 返回 null。动作：仅删除 `$recur`。

### 1.2 删除条件汇总表

| 场景 | 触发位置 | 条件 | 删除 Recurring | 删除 Template |
|------|---------|------|:---:|:---:|
| 缺少关联公司 | [L62-L68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L62-L68) | `empty($recur->company)` | ✅ | ❌ |
| 公司禁用 + 3个月内更新 | [L77-L89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L77-L89) | `!enabled` **且** `updated_at > now()-3月` | ✅ | ✅ |
| 公司禁用 + 3个月前更新 | [L77-L89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L77-L89) | `!enabled` **且** `updated_at <= now()-3月` | ❌（仅 skip） | ❌（仅 skip） |
| 公司无活跃用户 | [L92-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L92-L112) | 所有用户 `last_logged_in_at` 超过3个月 | ✅ | ✅ |
| 缺少模板模型 | [L116-L122](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L116-L122) | `$template` 为 null | ✅ | N/A |

### 1.3 跳过（continue）条件

| 场景 | 触发位置 | 条件 |
|------|---------|------|
| 所有调度已完成 | [L130-L136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L130-L136) | `getRemainingSchedules()` 返回空集合 |
| 今日无待执行调度 | [L141-L145](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L141-L145) | `endsBefore(tomorrow)` 过滤后为空 |
| 模型创建失败 | [L167-L169](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L167-L169) | `getModel()` 抛出异常返回 false |

### 1.4 标记 Completed 条件

| 场景 | 触发位置 | 条件 |
|------|---------|------|
| 所有调度已创建 | [L130-L136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L130-L136) | `getRemainingSchedules()` 返回空集合 → `$recur->update(['status' => Recurring::COMPLETE_STATUS])` |

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

    // 步骤1：查询已创建的子模型日期
    $created_schedules = DB::table($template->getTable())
                            ->where('type', $this->getRealType($template->type))
                            ->where('parent_id', $template->id)
                            ->get($date_field)
                            ->transform(function ($item, $key) use ($date_field) {
                                return Date::parse($item->$date_field)->format('Y-m-d');
                            })
                            ->toArray();

    // 步骤2：过滤掉已创建的调度
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

```php
protected function getDocumentModel(Document $template, Date $schedule_date): Document
{
    $template->cloneable_relations = ['items', 'totals'];

    $model = $template->duplicate();

    $diff_days = Date::parse($template->due_at)->diffInDays(Date::parse($template->issued_at));

    $model->type = $this->getRealType($template->type);
    $model->parent_id = $template->id;
    $model->issued_at = $schedule_date->format('Y-m-d');
    $model->due_at = $schedule_date->copy()->addDays($diff_days)->format('Y-m-d');
    $model->created_from = 'core::recurring';
    $model->save();

    $this->updateRelationTypes($model, $template->cloneable_relations);

    return $model;
}
```

| 字段 | 变化规则 | 示例 |
|-----|---------|------|
| `type` | 去除 `-recurring` 后缀 | `invoice-recurring` → `invoice` |
| `parent_id` | 设置为模板 ID | `$template->id` |
| `issued_at` | 设置为调度日期 | `$schedule_date->format('Y-m-d')` |
| `due_at` | 调度日期 + 模板的签发日到到期日的天数差 | `$schedule_date->copy()->addDays($diff_days)` |
| `created_from` | 固定为 `core::recurring` | `core::recurring` |

**关联关系处理：**
- `cloneable_relations = ['items', 'totals']`
- 复制后调用 `updateRelationTypes()` 更新 `items` 和 `totals` 的 `type` 字段
- `document_item_taxes` 由 Cloner 自动复制，其 `type` 字段**不**被 `updateRelationTypes()` 直接更新

### 3.2 Transaction 模型（income/expense）

生成逻辑：[getTransactionModel()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L226-L241)

```php
protected function getTransactionModel(Transaction $template, Date $schedule_date): Transaction
{
    $template->cloneable_relations = ['taxes'];

    $model = $template->duplicate();

    $model->type = $this->getRealType($template->type);
    $model->parent_id = $template->id;
    $model->paid_at = $schedule_date->format('Y-m-d');
    $model->created_from = 'core::recurring';
    $model->save();

    $this->updateRelationTypes($model, $template->cloneable_relations);

    return $model;
}
```

| 字段 | 变化规则 | 示例 |
|-----|---------|------|
| `type` | 去除 `-recurring` 后缀 | `expense-recurring` → `expense` |
| `parent_id` | 设置为模板 ID | `$template->id` |
| `paid_at` | 设置为调度日期 | `$schedule_date->format('Y-m-d')` |
| `created_from` | 固定为 `core::recurring` | `core::recurring` |

**关联关系处理：**
- `cloneable_relations = ['taxes']`
- 复制后调用 `updateRelationTypes()` 更新 TransactionTax 的 `type` 字段

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

该方法仅遍历传入的 `$relations` 数组，对每个关联执行 `UPDATE ... SET type = ?`。

### 3.5 DocumentItemTax 的 type 不被更新

[DocumentItemTax::onCloned()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/DocumentItemTax.php#L65-L70)：

```php
public function onCloned($src)
{
    $document_item = DocumentItem::find($this->document_item_id);
    $this->update(['document_id' => $document_item->document_id]);
}
```

`onCloned()` 是 Cloner 库的回调，仅更新 `document_id` 指向新复制的 DocumentItem 的所属文档。**`type` 字段保持为模板复制时的原始值**（即 `invoice-recurring` 或 `bill-recurring`），不会被 `updateRelationTypes()` 触及，因为 `cloneable_relations` 中不包含 `document_item_taxes`。

### 3.6 模板类型与生成类型对照表

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
| `DocumentCreated` | `CreateDocumentCreatedHistory` | 创建"已添加"历史记录 |
| `DocumentCreated` | `IncreaseNextDocumentNumber` | 增加文档编号 |
| `DocumentCreated` | `SettingFieldCreated` | 创建设置字段 |
| `DocumentRecurring` | `SendDocumentRecurringNotification` | 发送 recurring 通知 + 触发 sent/received |
| `DocumentSent` | `MarkDocumentSent` | 标记为已发送 + 创建"已发送"历史记录 |
| `DocumentReceived` | `MarkDocumentReceived` | 标记为已接收 + 创建"已接收"历史记录 |
| `TransactionCreated` | `IncreaseNextTransactionNumber` | 增加交易编号 |

### 4.3 自动发送邮件的完整条件

自动发送逻辑在 [SendDocumentRecurringNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L19-L57)：

```php
public function handle(Event $event)
{
    $document = $event->document;
    $config = config('type.document.' . $document->type . '.notification');

    // 条件1：notification config 存在且有 class
    if (empty($config) || empty($config['class'])) {
        return;
    }

    // 条件2：模板的 recurring.auto_send 不能为 false
    if ($document->parent?->recurring?->auto_send == false) {
        return;
    }

    // 条件3：canNotifyTheContactOfDocument 全量检查（见下）
    if ($this->canNotifyTheContactOfDocument($document)) {
        $document->contact->notify(new $notification($document, "{$document->type}_recur_customer", $attach_pdf));
    }

    // 触发标记事件（读取 invoice/bill 的 auto_send 配置，不是 -recurring 的）
    $sent = config('type.document.' . $document->type . '.auto_send', DocumentSent::class);
    event(new $sent($document));

    // 条件4：notify_user 为 true 时对公司用户发送通知
    if (! $config['notify_user']) {
        return;
    }
    foreach ($document->company->users as $user) {
        if ($user->cannot('read-notifications')) {
            continue;
        }
        $user->notify(new $notification($document, "{$document->type}_recur_admin"));
    }
}
```

#### canNotifyTheContactOfDocument 的完整条件

[canNotifyTheContactOfDocument()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L242-L269)：

```php
public function canNotifyTheContactOfDocument(Document $document): bool
{
    $config = config('type.document.' . $document->type . '.notification');

    // 条件 A：notify_contact 配置为 true
    if (! $config['notify_contact']) {
        return false;
    }

    // 条件 B：contact 存在且 enabled != 0
    if (! $document->contact || ($document->contact->enabled == 0)) {
        return false;
    }

    // 条件 C：contact_email 非空
    if (empty($document->contact_email)) {
        return false;
    }

    // 条件 D：邮箱通过 RFC + DNS 校验
    $validator = new EmailValidator();
    $validations = new MultipleValidationWithAnd([
        new RFCValidation(),
        new DNSCheckValidation(),
    ]);
    if (! $validator->isValid($document->contact_email, $validations)) {
        return false;
    }

    return true;
}
```

汇总：

| 编号 | 条件 | 位置 |
|-----|------|------|
| 1 | `config('type.document.{type}.notification.class')` 存在 | [SendDocumentRecurringNotification.php#L24](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L24) |
| 2 | `$document->parent->recurring->auto_send != false` | [SendDocumentRecurringNotification.php#L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L28) |
| 3A | `$config['notify_contact']` 为 true | [Documents.php#L246](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L246) |
| 3B | `$document->contact` 存在 **且** `contact->enabled != 0` | [Documents.php#L250](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L250) |
| 3C | `$document->contact_email` 非空 | [Documents.php#L254](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L254) |
| 3D | 邮箱通过 RFCValidation + DNSCheckValidation | [Documents.php#L259-L266](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L259-L266) |
| 4 | `$config['notify_user']` 为 true 时通知公司用户 | [SendDocumentRecurringNotification.php#L45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L45) |

### 4.4 auto_send 配置读取的是 invoice/bill，不是 invoice-recurring/bill-recurring

生成 Document 时 `type` 已从 `invoice-recurring` 变为 `invoice`，因此 [SendDocumentRecurringNotification.php#L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L40)：

```php
$sent = config('type.document.' . $document->type . '.auto_send', DocumentSent::class);
```

实际读取的是：
- invoice 类型 → `config('type.document.invoice.auto_send')` → [type.php#L164](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L164) → `App\Events\Document\DocumentSent`
- bill 类型 → `config('type.document.bill.auto_send')` → [type.php#L268](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L268) → `App\Events\Document\DocumentReceived`

`invoice-recurring` 和 `bill-recurring` 类型的 `auto_send` 配置不会被 recurring:check 生成的文档读取。

| 文档类型（生成后） | 读取的配置 key | 配置值 | 触发的事件 |
|---------|---------|---------|------|
| `invoice` | `type.document.invoice.auto_send` | `App\Events\Document\DocumentSent` | `DocumentSent` |
| `bill` | `type.document.bill.auto_send` | `App\Events\Document\DocumentReceived` | `DocumentReceived` |

### 4.5 自动标记 sent/received 的状态逻辑

**Invoice → DocumentSent**：[MarkDocumentSent.php#L14-L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentSent.php#L14-L28)

```php
public function handle(DocumentMarkedSent|DocumentSent $event): void
{
    if (! in_array($event->document->status, ['partial', 'paid'])) {
        $event->document->status = 'sent';
        if ($event->document->amount == 0) {
            $event->document->status = 'paid';
        }
        $event->document->save();
    }
    // 创建"已发送"历史记录
    $this->dispatch(new CreateDocumentHistory($event->document, 0, $this->getDescription($event)));
}
```

**Bill → DocumentReceived**：[MarkDocumentReceived.php#L19-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentReceived.php#L19-L49)

```php
public function handle(Event $event)
{
    if (! in_array($event->document->status, ['partial', 'paid'])) {
        $event->document->status = 'received';
        if ($event->document->amount == 0) {
            $event->document->status = 'paid';
        }
        $event->document->save();
    }
    // 创建"已接收"历史记录
    $this->dispatch(new CreateDocumentHistory($event->document, 0, trans('documents.messages.marked_received', ...)));
}
```

标记规则：
- 当前状态非 `partial`/`paid` → 设为 `sent`（invoice）或 `received`（bill）
- 金额为 0 → 直接设为 `paid`
- 已是 `partial`/`paid` → 不改状态，仅创建历史记录

### 4.6 历史记录创建顺序

一个 recurring 生成的 Document，在 `DB::transaction` 内按以下顺序产生 2~3 条历史记录：

```
1. event(DocumentCreated)
   └─ CreateDocumentCreatedHistory.handle()
      └─ dispatch(CreateDocumentHistory)  → 消息: "added {document_number}"

2. event(DocumentRecurring)
   └─ SendDocumentRecurringNotification.handle()
      ├─ 发送客户/用户通知（如果条件满足）
      └─ event(DocumentSent 或 DocumentReceived)
         └─ MarkDocumentSent/MarkDocumentReceived.handle()
            ├─ 更新 document.status = 'sent'/'received'
            └─ dispatch(CreateDocumentHistory)  → 消息: "email_sent" 或 "marked_received"
```

Transaction 没有历史记录事件链，仅 `IncreaseNextTransactionNumber`。

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
├─ 遍历 Recurring 记录（status=active）
│  ├─ 检查公司状态（enabled=true）→ 通过
│  ├─ 检查活跃用户（存在3个月内登录用户）→ 通过
│  ├─ 获取模板（invoice-recurring, id=T）
│  ├─ getRemainingSchedules(template, recur)
│  │  ├─ 查询 documents 表
│  │  │  └─ WHERE parent_id = T
│  │  │  └─ WHERE type = 'invoice'
│  │  │  └─ SELECT issued_at
│  │  │  └─ 假设返回: ['2026-05-15', '2026-05-16']
│  │  │     （17号失败未创建，18号也未创建）
│  │  ├─ getRecurringSchedule() 生成完整调度（到limit为止）
│  │  │  └─ 2026-05-15, 2026-05-16, 2026-05-17, 2026-05-18, 2026-05-19
│  │  └─ filter() 排除已存在日期
│  │     └─ 剩余: 2026-05-17, 2026-05-18, 2026-05-19
│  ├─ endsBefore(明天=2026-05-20) 过滤
│  │  └─ 保留: 2026-05-17, 2026-05-18, 2026-05-19
│  └─ 遍历剩余调度，逐个 recur()
│     ├─ 2026-05-17（补建历史）
│     │  └─ DB::transaction
│     │     ├─ getDocumentModel()
│     │     │  ├─ cloneable_relations = ['items', 'totals']
│     │     │  ├─ duplicate() → 复制 template + items + totals
│     │     │  │  └─ document_item_taxes 由 Cloner 自动复制
│     │     │  │     └─ onCloned() 仅更新 document_id，type 不变
│     │     │  ├─ type: 'invoice-recurring' → 'invoice'
│     │     │  ├─ parent_id = T
│     │     │  ├─ issued_at = '2026-05-17'
│     │     │  ├─ due_at = '2026-05-17' + diff_days
│     │     │  ├─ created_from = 'core::recurring'
│     │     │  ├─ save()
│     │     │  └─ updateRelationTypes(model, ['items', 'totals'])
│     │     │     ├─ items → UPDATE SET type='invoice'
│     │     │     └─ totals → UPDATE SET type='invoice'
│     │     │     └─ document_item_taxes 不受影响，type 仍为 'invoice-recurring'
│     │     ├─ event(DocumentCreated)
│     │     │  ├─ CreateDocumentCreatedHistory → dispatch(CreateDocumentHistory "added")
│     │     │  ├─ IncreaseNextDocumentNumber
│     │     │  └─ SettingFieldCreated
│     │     └─ event(DocumentRecurring)
│     │        └─ SendDocumentRecurringNotification
│     │           ├─ 检查 notification config/class → 通过
│     │           ├─ 检查 parent.recurring.auto_send != false → 通过
│     │           ├─ canNotifyTheContactOfDocument:
│     │           │  ├─ notify_contact=true
│     │           │  ├─ contact 存在且 enabled
│     │           │  ├─ contact_email 非空
│     │           │  └─ 邮箱 RFC+DNS 校验通过
│     │           │  → 发送客户通知
│     │           ├─ event(DocumentSent)  ← 读取 type.document.invoice.auto_send
│     │           │  └─ MarkDocumentSent
│     │           │     ├─ status = 'sent'
│     │           │     └─ dispatch(CreateDocumentHistory "email_sent")
│     │           └─ notify_user=true → 通知公司用户
│     ├─ 2026-05-18（补建历史）
│     │  └─ 同上流程
│     └─ 2026-05-19（今日）
│        └─ 同上流程
└─ 完成
```

### 5.3 关键机制

1. **失败恢复**：`getRemainingSchedules()` 通过查询已创建子模型的日期判断缺失，不依赖上次执行状态记录
2. **历史日期补建**：`endsBefore(tomorrow)` 包含所有早于明天的未执行调度
3. **去重保证**：每次执行都检查已存在的子模型，避免重复创建
4. **完整事件链**：补建的历史日期文档也触发 DocumentCreated → DocumentRecurring → DocumentSent/Received 的完整事件链

### 5.4 测试验证

测试用例：[RecurringCheckTest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Commands/RecurringCheckTest.php#L25-L38)

该测试启动日期设为 7 天前，`limit_count=20`，执行 `recurring:check` 后断言应创建 7 个 Document（含历史日期全部补建）：

```php
public function testItShouldCreateCorrectNumberOfRecurringInvoicesByCount(): void
{
    Notification::fake();
    $this->dispatch(new CreateDocument($this->getInvoiceRequest('count')));
    Date::setTestNow(Date::today());
    $this->artisan('recurring:check');
    $this->assertDatabaseCount('documents', $this->recurring_count + 1);
    Notification::assertSentToTimes($this->user, InvoiceNotification::class, $this->recurring_count);
}
```

---

## 总结

`recurring:check` 命令的设计体现了几个关键特性：

1. **健壮的错误恢复**：通过查询已创建的子模型日期来判断调度状态，自然支持失败恢复
2. **清晰的类型转换**：模板（`*-recurring`）与生成模型（`*`）通过 `type` 字段区分，`parent_id` 建立关联
3. **完整的事件驱动**：DocumentCreated → DocumentRecurring → DocumentSent/Received 三级事件链，分别创建不同的历史记录
4. **灵活的调度过滤**：`endsBefore(tomorrow)` 确保只创建今天及之前的调度
5. **条件严格的自动发送**：notification config、auto_send、notify_contact、contact 状态、邮箱有效性等多重校验
6. **关联同步的局限**：`updateRelationTypes()` 仅更新 items/totals 的 type，嵌套的 document_item_taxes.type 保持模板原值
