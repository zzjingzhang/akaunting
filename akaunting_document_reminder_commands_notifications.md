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
// app/Models/Document/Document.php scope 定义
public function scopeAccrued(Builder $query): Builder
{
    return $query->whereNotIn($this->qualifyColumn('status'), ['draft', 'cancelled']);
}

public function scopeNotPaid(Builder $query): Builder
{
    return $query->where($this->qualifyColumn('status'), '<>', 'paid');
}
```

### Company 筛选链路详解

两个命令共享相同的公司筛选结构，但时间窗口不同：

```
Company::whereHas('invoices'|'bills', fn ($q) =>
    $q->allCompanies()
      ->whereBetween('due_at', [$start, $end])
      ->accrued()
      ->notPaid()
)
->enabled()
->cursor()
```

#### (a) `allCompanies()` — 移除当前公司全局 scope

[Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L77-L80)：

```php
public function scopeAllCompanies($query)
{
    return $query->withoutGlobalScope('App\Scopes\Company');
}
```

在公司筛选阶段，`$query->allCompanies()` 被调用在 `whereHas` 闭包内部（即对 `invoices`/`bills` 关联查询），目的是**跨所有公司**查找是否有到期单据。否则 Company 全局 scope 会按 `company_id()` 过滤，导致只能查到"当前公司"的单据，而此时尚未设置当前公司（CLI 命令启动时默认无当前公司），查询会得到空结果。

Company 全局 scope 定义见 [app/Scopes/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L21-L50)，默认注入 `where company_id = company_id()`。

#### (b) `enabled()` — 只对 Company 模型自身有效

[Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L394-L397)：

```php
public function scopeEnabled($query, $value = 1)
{
    return $query->where('enabled', $value);
}
```

`enabled()` 是链式调用在 `Company::whereHas(...)->enabled()` 上，作用于 Company 模型自身：只选出 `enabled = 1` 的公司。disabled 的公司即使有到期单据也被跳过。

> 注意：Company 模型**重写**了 `Abstracts\Model` 中的 `scopeEnabled`，签名不同（Company 版带 `$value = 1` 参数，Model 版无参数）。命令中调用的是 Company 模型自己的版本。

#### (c) `makeCurrent()` — 设置当前公司、加载该公司 settings

[Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L601-L627)：

```php
public function makeCurrent($force = false)
{
    // ...
    app()->instance(static::class, $this);
    setting()->setExtraColumns(['company_id' => $this->id]);
    setting()->forgetAll();
    setting()->load(true);
    Overrider::load('settings');
    Overrider::load('currencies');
    Overrider::load('categoryTypes');
    // ...
}
```

在 `foreach ($companies as $company)` 循环内调用 `$company->makeCurrent()`，这会：
1. 将该公司绑定到容器，成为"当前公司"
2. 加载该公司的 settings（`schedule.send_invoice_reminder` / `schedule.send_bill_reminder` / `schedule.invoice_days` / `schedule.bill_days` 等配置生效）
3. 覆盖 currencies、categoryTypes

因此紧接着的 `setting('schedule.send_invoice_reminder')` 读取的就是**该公司自己的配置**，而不是全局默认值。

#### (d) Company scope 对 `remind()` 查询的影响

`remind()` 内部执行：

```php
$invoices = Document::with('contact')
    ->invoice()
    ->accrued()
    ->notPaid()
    ->due($date)
    ->cursor();
```

此时由于 `makeCurrent()` 已将 `company_id()` 指向当前公司，而 Document 模型默认应用 Company 全局 scope（`$tenantable = true`），所以这个查询**只会返回当前公司的 invoices**。不需要显式 `allCompanies()`，因为我们此时只关心当前公司的单据。

**潜在坑点**：如果 `makeCurrent()` 未成功调用（比如 `setting()` 抛异常被外层吞掉），那么 `company_id()` 可能返回 `null`，Company scope 会注入 `where company_id = null`，导致查不到任何单据。但 `makeCurrent()` 本身没有 try/catch，异常会直接向上冒泡到命令外层，由 `foreach` 循环外的代码接管（命令没有总 try/catch），可能中断整个命令。

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

事件触发代码（注意 try/catch 结构）：
```php
// InvoiceReminder.php / BillReminder.php remind()
try {
    event(new DocumentReminded($invoice, Notification::class));
} catch (\Throwable $e) {
    $this->error($e->getMessage());
    report($e);
}
```

**try/catch 范围**：只包住 `event(...)` 调用。只有 listener 在**同步执行**中抛出的 Throwable 才会被捕获（详见 §4 场景 D）。

---

## 3. Listener 如何决定是否通知 contact 和 company users

[SendDocumentReminderNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentReminderNotification.php) 的 `handle` 方法：

```php
public function handle(Event $event)
{
    $document = $event->document;
    $notification = $event->notification;

    // Notify the customer
    if ($this->canNotifyTheContactOfDocument($document)) {
        $document->contact->notify(new $notification($document, "{$document->type}_remind_customer"));
    }

    // Notify all users assigned to this company
    foreach ($document->company->users as $user) {
        if ($user->cannot('read-notifications')) {
            continue;
        }

        $user->notify(new $notification($document, "{$document->type}_remind_admin"));
    }
}
```

### 通知 Contact

Listener 首先调用 `canNotifyTheContactOfDocument()` 进行**四重检查**：

```php
// app/Traits/Documents.php canNotifyTheContactOfDocument()
public function canNotifyTheContactOfDocument(Document $document): bool
{
    // 检查 1: 配置是否允许通知 contact
    $config = config('type.document.' . $document->type . '.notification');
    if (! $config['notify_contact']) {
        return false;
    }

    // 检查 2: contact 存在且已启用
    if (! $document->contact || ($document->contact->enabled == 0)) {
        return false;
    }

    // 检查 3: contact_email 非空
    if (empty($document->contact_email)) {
        return false;
    }

    // 检查 4: 邮箱格式有效性（RFC + DNS MX 记录）
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

#### Bill 默认 `notify_contact = false`

[config/type.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php) 中定义：

```php
// Invoice (line 159-163)
'notification' => [
    'class'                 => 'App\Notifications\Sale\Invoice',
    'notify_contact'        => true,
    'notify_user'           => true,
],

// Bill (line 263-267)
'notification' => [
    'class'                 => 'App\Notifications\Purchase\Bill',
    'notify_contact'        => false,
    'notify_user'           => true,
],
```

**关键结论**：Bill 默认 `notify_contact = false`。因此在正常配置下，`canNotifyTheContactOfDocument()` 对 Bill 直接返回 false，BillReminder **永远不会通知 vendor/contact**。即使配置被改成 true，还需要通过后续 contact 存在、邮箱有效等检查。

#### 默认没有 `bill_remind_customer` 模板

[database/seeds/EmailTemplates.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/EmailTemplates.php) 中只 seed 了以下 reminder 模板：

| alias | class | 说明 |
|-------|-------|------|
| `invoice_remind_customer` | Invoice | ✅ 存在 |
| `invoice_remind_admin` | Invoice | ✅ 存在 |
| `bill_remind_admin` | Bill | ✅ 存在 |
| **`bill_remind_customer`** | — | ❌ **不存在** |

若非正常地把 Bill 的 `notify_contact` 改为 true，listener 仍会尝试用别名 `bill_remind_customer` 构造 `new Bill($document, 'bill_remind_customer')`。由于该模板未 seed，`EmailTemplate::alias('bill_remind_customer')->first()` 返回 null，`$this->template` 为 null。

[app/Notifications/Purchase/Bill.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Purchase/Bill.php) 的构造函数：

```php
public function __construct(Document $bill = null, string $template_alias = null)
{
    parent::__construct();
    $this->bill = $bill;
    $this->template = EmailTemplate::alias($template_alias)->first(); // 返回 null
}
```

后续当 queue worker 执行 `toMail()` 时，基类 `Abstracts\Notification` 的 `initMailMessage()` → `getSubject()` → `getBody()` → `getFooter()` 都会访问 `$this->template->subject`、`$this->template->body`、`$this->template->alias`：

```php
// app/Abstracts/Notification.php
public function getSubject(): string
{
    return !empty($this->custom_mail['subject'])
            ? $this->custom_mail['subject']
            : $this->replaceTags($this->template->subject);  // null->subject = Error
}

public function getBody()
{
    $body = !empty($this->custom_mail['body']) ? $this->custom_mail['body'] : $this->replaceTags($this->template->body);
    return $body . $this->getFooter();
}

public function getFooter()
{
    $url = 'https://akaunting.com/...' . $this->template->alias;  // null->alias = Error
    // ...
}
```

对 null 访问属性会产生 `Error`（"Attempt to read property on null"），这是 Throwable。但由于 Notification 实现了 `ShouldQueue`，这个错误发生在 queue worker 中（异步），**不会被命令层 try/catch 捕获**。

**这种场景只在配置被改坏（把 `notify_contact` 从 false 改为 true）时可达**。正常部署下 BillReminder 不会尝试通知 contact。

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
- Invoice: `invoice_remind_admin`（seed 存在）
- Bill: `bill_remind_admin`（seed 存在）

**Company users 通知在 contact 通知之后执行**，如果 contact 通知的同步阶段抛出异常并中断了 listener，company users 也不会被通知（详见 §4 场景 D）。

---

## 4. Notification config 缺失 / notify_contact=false / contact 不可通知时的结果

### 场景 A：`notification` 配置数组缺失（config 被改坏）

源码（[app/Traits/Documents.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L242-L248)）：
```php
$config = config('type.document.' . $document->type . '.notification');
if (! $config['notify_contact']) {
    return false;
}
```

如果 `config('type.document.invoice.notification')` 返回 `null`（配置被改坏或未定义），访问 `$config['notify_contact']` 在 PHP 8+ 中会触发 `TypeError`："Cannot access offset of type string on null"，这是 **Throwable**。

- **Contact**：不会通知
- **Company users**：取决于异常是否发生。`canNotifyTheContactOfDocument()` 在 company users 循环**之前**调用，如果此处抛 Throwable，`handle()` 方法立即中断，**company users 也不会被通知**
- **是否被命令层 try/catch 捕获**：`TypeError` 是 Throwable，listener 同步执行时抛出，**会被命令层 `catch (\Throwable $e)` 捕获**，命令继续处理下一张单据
- **可达性**：需要 `config/type.php` 被改坏或运行时被清除，非正常部署场景

### 场景 B：`notify_contact = false`（Bill 默认配置）

- **Contact**：不会通知（`canNotifyTheContactOfDocument()` 在第 1 步就返回 false）
- **Company users**：继续通知（不受影响）
- **是否被外层 try/catch 捕获**：不会，正常的条件分支返回 false，无异常
- **可达性**：Bill 默认场景，**正常可达**

### 场景 C：`notify_contact = true` 但 contact 不可通知

| 子场景 | Contact 是否通知 | Company users | 是否抛异常 | 是否被 try/catch 捕获 |
|--------|:---:|:---:|:---:|:---:|
| `$document->contact` 不可用（详见下方说明） | ❌ | ✅ 继续 | 否 | — |
| `$document->contact->enabled == 0` | ❌ | ✅ 继续 | 否 | — |
| `$document->contact_email` 为空 | ❌ | ✅ 继续 | 否 | — |
| 邮箱格式不符合 RFC | ❌ | ✅ 继续 | 否 | — |
| 域名无 MX 记录（DNSCheck 失败） | ❌ | ✅ 继续 | 否 | — |

所有子场景都是 `canNotifyTheContactOfDocument()` 内正常的条件判断，**返回 false 而不抛异常**，不会被命令层 try/catch 捕获。Company users 通知照常执行。

#### 关于 `$document->contact` 为 null 的可达性

[Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L123-L126)：

```php
public function contact()
{
    return $this->belongsTo('App\Models\Common\Contact')->withDefault(['name' => trans('general.na')]);
}
```

`contact()` 关系定义了 `withDefault()`，正常的 Eloquent 关系访问（`$document->contact`）会返回一个默认 Contact 对象（`name = 'N/A'`），**不会是 null**。只有在以下非正常场景下才可能为 null：
- 直接构造 Document 模型时手动设置 `contact` 为 null
- 通过 `setRelation('contact', null)` 覆盖 relation
- 数据库中 `contact_id` 指向不存在的记录（虽然 `withDefault` 通常对此也返回默认对象）
- 外部代码直接触发 DocumentReminded 事件且传入的 Document 没有正确加载 relation

**正常命令流程下，`! $document->contact` 为 true 的情况不可达**。

### 场景 D：contact 可通知但通知执行异常

需区分两个阶段：

#### (i) Listener 同步阶段异常

`new $notification($document, "{$document->type}_remind_customer")` 构造函数中，`EmailTemplate::alias(...)` 查询可能抛 DB 异常（连接断开等），或构造函数本身抛异常。这些发生在 `event()` 同步调用内：

- **Contact**：通知失败
- **Company users**：`handle()` 方法被中断，**company users 也不会被通知**
- **是否被命令层 try/catch 捕获**：**会被捕获**，因为异常在 `event()` 同步调用内抛出

#### (ii) Queued notification 处理阶段异常

[app/Abstracts/Notification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Notification.php#L11-L12)：

```php
abstract class Notification extends BaseNotification implements ShouldQueue
{
    use Queueable;
    // ...
    public function __construct()
    {
        $this->onQueue('notifications');
    }
}
```

`Notification` 实现了 `ShouldQueue`，所以 `toMail()` / `toArray()` 中的模板访问、邮件发送等操作在 queue worker 中**异步**执行：

- **Contact**：通知失败（mail/send 抛异常）
- **Company users**：不受影响，因为 queue worker 失败和 listener 同步执行无关
- **是否被命令层 try/catch 捕获**：**不会被捕获**，异常发生在 queue worker 进程中，不在命令 `event()` 同步调用范围内
- **可达性**：Mail 配置错误、网络故障、模板被误删等情况下可达

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
