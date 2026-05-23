# InvoiceReminder vs BillReminder 命令分析

## 1. 两个命令筛选 document type/status/due_at/company 的差异

### Document Type
| 命令 | Scope | 筛选条件 |
|------|-------|----------|
| InvoiceReminder | `invoice()` | `type = 'invoice'` |
| BillReminder | `bill()` | `type = 'bill'` |

### Status 筛选
两个命令使用相同的状态筛选逻辑：
- `accrued()`: 排除 `draft` 和 `cancelled` 状态
- `notPaid()`: 排除 `paid` 状态

```php
// Document.php 中的 scope 定义
public function scopeAccrued(Builder $query): Builder
{
    return $query->whereNotIn($this->qualifyColumn('status'), ['draft', 'cancelled']);
}

public function scopeNotPaid(Builder $query): Builder
{
    return $query->where($this->qualifyColumn('status'), '<>', 'paid');
}
```

### Due Date 范围（公司筛选阶段）
| 命令 | 起始日期 | 结束日期 | 时间窗口 |
|------|---------|---------|---------|
| InvoiceReminder | `today - 1 month` | `today + 1 week` | 过去1个月到未来1周 |
| BillReminder | `today - 1 week` | `today + 1 month` | 过去1周到未来1个月 |

```php
// InvoiceReminder.php handle()
$start_date = $today->copy()->subMonth()->toDateString() . ' 00:00:00';
$end_date = $today->copy()->addWeek()->toDateString() . ' 23:59:59';

// BillReminder.php handle()
$start_date = $today->copy()->subWeek()->toDateString() . ' 00:00:00';
$end_date = $today->copy()->addMonth()->toDateString() . ' 23:59:59';
```

### Due Date 精确匹配（remind 方法）
| 命令 | 日期计算 | 含义 | 类型 |
|------|---------|------|------|
| InvoiceReminder | `today - $day` | 到期日在过去N天 | 逾期提醒 |
| BillReminder | `today + $day` | 到期日在未来N天 | 提前提醒 |

```php
// InvoiceReminder.php remind()
$date = Date::today()->subDays($day)->toDateString();
$invoices = Document::with('contact')->invoice()->accrued()->notPaid()->due($date)->cursor();

// BillReminder.php remind()
$date = Date::today()->addDays($day)->toDateString();
$bills = Document::with('contact')->bill()->accrued()->notPaid()->due($date)->cursor();
```

`scopeDue` 实现精确日期匹配：
```php
public function scopeDue(Builder $query, $date): Builder
{
    return $query->whereDate('due_at', '=', $date);
}
```

### Company 配置
| 命令 | 开关配置 | 天数配置 |
|------|---------|---------|
| InvoiceReminder | `schedule.send_invoice_reminder` | `schedule.invoice_days` |
| BillReminder | `schedule.send_bill_reminder` | `schedule.bill_days` |

---

## 2. DocumentReminded 事件携带的 notification class

[DocumentReminded.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Document/DocumentReminded.php) 事件接收两个参数：

```php
public function __construct(Document $document, string $notification)
{
    $this->document = $document;
    $this->notification = $notification;
}
```

| 命令 | Notification Class |
|------|-------------------|
| InvoiceReminder | `App\Notifications\Sale\Invoice::class` |
| BillReminder | `App\Notifications\Purchase\Bill::class` |

事件触发代码：
```php
// InvoiceReminder.php
event(new DocumentReminded($invoice, Notification::class));

// BillReminder.php
event(new DocumentReminded($bill, Notification::class));
```

---

## 3. Listener 如何决定是否通知 contact 和 company users

[SendDocumentReminderNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentReminderNotification.php) 的 `handle` 方法：

### 通知 Contact
使用 `canNotifyTheContactOfDocument()` 方法进行四重检查：

```php
public function canNotifyTheContactOfDocument(Document $document): bool
{
    // 1. 检查配置是否允许通知 contact
    $config = config('type.document.' . $document->type . '.notification');
    if (! $config['notify_contact']) {
        return false;
    }

    // 2. 检查 contact 存在且已启用
    if (! $document->contact || ($document->contact->enabled == 0)) {
        return false;
    }

    // 3. 检查 contact_email 非空
    if (empty($document->contact_email)) {
        return false;
    }

    // 4. 检查邮箱格式有效性（RFC + DNS MX 记录）
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

通知模板别名：
- Invoice: `invoice_remind_customer`
- Bill: `bill_remind_customer`

### 通知 Company Users
遍历公司所有用户，检查 `read-notifications` 权限：

```php
foreach ($document->company->users as $user) {
    if ($user->cannot('read-notifications')) {
        continue;
    }

    $user->notify(new $notification($document, "{$document->type}_remind_admin"));
}
```

通知模板别名：
- Invoice: `invoice_remind_admin`
- Bill: `bill_remind_admin`

---

## 4. Notification config 缺失或 contact 不可通知时结果

### 配置缺失或 notify_contact = false
- `canNotifyTheContactOfDocument()` 返回 `false`
- **Contact 不会收到通知**
- Company users 通知不受影响，正常发送
- 不会抛出异常，静默跳过

### Contact 不可通知的场景
| 场景 | 结果 |
|------|------|
| `$document->contact` 为 null | 不通知 contact |
| `$document->contact->enabled == 0` | 不通知 contact |
| `$document->contact_email` 为空 | 不通知 contact |
| 邮箱格式不符合 RFC 标准 | 不通知 contact |
| 域名无 MX 记录（DNSCheck 失败） | 不通知 contact |

**所有这些情况都只是跳过 contact 通知，company users 通知仍然正常执行，不会中断或抛出异常。**

---

## 5. 到期日边界导致今天不提醒的例子

### 例子 1：Invoice 逾期提醒边界（天数 = 1）
假设今天是 `2026-05-19`，`schedule.invoice_days = 1`

| 发票到期日 | 计算目标日期 `today - 1` | 是否匹配 | 结果 |
|-----------|------------------------|---------|------|
| 2026-05-18 | 2026-05-18 | ✅ 匹配 | **今天提醒** |
| 2026-05-19 | 2026-05-18 | ❌ 不匹配 | **今天不提醒** |
| 2026-05-17 | 2026-05-18 | ❌ 不匹配 | **今天不提醒** |

**说明**：到期日恰好是今天（2026-05-19）的发票，由于 `subDays(1) = 2026-05-18`，而 `due_at = 2026-05-19` 不等于目标日期，因此今天不会提醒。需要等到明天（2026-05-20），当 `subDays(1) = 2026-05-19` 时才会触发提醒。

### 例子 2：Bill 提前提醒边界（天数 = 1）
假设今天是 `2026-05-19`，`schedule.bill_days = 1`

| 账单到期日 | 计算目标日期 `today + 1` | 是否匹配 | 结果 |
|-----------|------------------------|---------|------|
| 2026-05-20 | 2026-05-20 | ✅ 匹配 | **今天提醒** |
| 2026-05-19 | 2026-05-20 | ❌ 不匹配 | **今天不提醒** |
| 2026-05-21 | 2026-05-20 | ❌ 不匹配 | **今天不提醒** |

**说明**：到期日恰好是今天（2026-05-19）的账单，由于 `addDays(1) = 2026-05-20`，而 `due_at = 2026-05-19` 不等于目标日期，因此今天不会提醒。昨天（2026-05-18）当 `addDays(1) = 2026-05-19` 时已经提醒过了。

### 例子 3：公司筛选阶段的边界排除
假设今天是 `2026-05-19`：

- **Invoice 到期日 = 2026-04-18**：`today - 1 month = 2026-04-19`，由于 2026-04-18 < 2026-04-19，**不在时间窗口内，公司直接被跳过**
- **Bill 到期日 = 2026-06-20**：`today + 1 month = 2026-06-19`，由于 2026-06-20 > 2026-06-19，**不在时间窗口内，公司直接被跳过**

**关键结论**：`scopeDue()` 使用 `whereDate('due_at', '=', $date)` 进行**精确日期匹配**，而非范围匹配。只有到期日与计算出的目标日期完全相等的单据才会被筛选出来。
