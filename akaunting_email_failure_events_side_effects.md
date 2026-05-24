# 邮件发送异常事件对系统状态的影响分析（修正版）

## 1. TooManyEmailsSent 与 InvalidEmailDetected 携带的数据差异

### TooManyEmailsSent 事件
[TooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/TooManyEmailsSent.php)

```php
class TooManyEmailsSent extends Event
{
    public $user_id;

    public function __construct(int $user_id)
    {
        $this->user_id = $user_id;
    }
}
```

**携带数据：**
- `user_id` (int): 触发邮件发送超限的用户 ID

### InvalidEmailDetected 事件
[InvalidEmailDetected.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php)

```php
class InvalidEmailDetected extends Event
{
    public $email;
    public $error;
    public $contact = null;
    public $user = null;

    public function __construct(string $email, string $error)
    {
        $this->email = $email;
        $this->error = $error;
        $this->setContact();  // 查询 Contact 模型，不查询 ContactPerson
        $this->setUser();
    }

    public function setContact()
    {
        // 只查询 Contact 表，使用 email() + enabled() 过滤
        $contact = Contact::email($this->email)->enabled()->first();
        // ...
    }

    public function setUser()
    {
        // 只查询 User 表，使用 email() + enabled() 过滤
        $user = user_model_class()::email($this->email)->enabled()->first();
        // ...
    }
}
```

**携带数据：**
- `email` (string): 无效的邮箱地址
- `error` (string): 邮件发送失败的错误信息（来自 `MailerHttpTransportException::getMessage()`）
- `contact` (Contact|null): 关联的 **Contact** 实体（仅查询 `contacts` 表，**不查询 `contact_persons` 表**）
- `user` (User|null): 关联的 User 实体

### 数据差异对比

| 数据字段 | TooManyEmailsSent | InvalidEmailDetected |
|---------|-------------------|----------------------|
| user_id | ✅ (int) | ❌ |
| email | ❌ | ✅ (string) |
| error | ❌ | ✅ (string) |
| contact | ❌ | ✅ (Contact/null，仅查 contacts 表) |
| user | ❌ | ✅ (User/null) |

**核心差异：**
- `TooManyEmailsSent` 是**速率限制事件**，仅需识别触发者（user_id）
- `InvalidEmailDetected` 是**业务异常事件**，需要携带邮箱、错误信息以及自动解析关联的业务实体

---

## 2. Event Provider 中事件与 Listener 绑定关系及顺序影响

[Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L124-L131)

```php
protected $listen = [
    \App\Events\Email\TooManyEmailsSent::class => [
        \App\Listeners\Email\ReportTooManyEmailsSent::class,
        \App\Listeners\Email\TellFirewallTooManyEmailsSent::class,
    ],
    \App\Events\Email\InvalidEmailDetected::class => [
        \App\Listeners\Email\DisablePersonDueToInvalidEmail::class,     // L1
        \App\Listeners\Email\SendInvalidEmailNotification::class,        // L2
    ],
];
```

### Listener 顺序是否影响通知内容？—— **不影响**

经代码核对，两个 Listener 之间不存在数据依赖导致的顺序敏感：

**L1 [DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php) 做什么：**
- 修改 `$event->contact->enabled = false`
- 修改 `$event->user->enabled = false`（有保护）
- 仅修改 `enabled` 字段

**L2 [SendInvalidEmailNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/SendInvalidEmailNotification.php) 读什么：**
- `$event->contact` → 仅使用 `isCustomer()`/`isVendor()`/`isEmployee()`/`$event->contact->type`（依赖 `type` 字段，与 `enabled` 无关）
- `$event->user` → 仅判断 `if (empty($event->user))` 是否存在
- `$event->email` → 邮箱地址字符串
- `$event->error` → 错误信息字符串
- `$users = company()?->users` → 重新查询当前公司用户列表（不依赖 `$event->user->enabled`）

**通知内容 [InvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Email/InvalidEmail.php#L52-L64)：**
```php
public function toMail($notifiable): MailMessage
{
    return (new MailMessage)
        ->subject(trans('notifications.email.invalid.title', ['type' => $this->type]))
        ->line(trans('notifications.email.invalid.description', ['email' => $this->email]))
        ->line(new HtmlString('<i>' . $this->error . '</i>'))
        ->action(trans_choice('general.dashboards', 1), $dashboard_url);
}
```
- `$this->type` = 联系人类型名称（如 "Customer"），来自 `$event->contact->type`，不受 `enabled` 影响
- `$this->email` = 邮箱字符串，不受 `enabled` 影响
- `$this->error` = 错误信息，不受 `enabled` 影响

**结论：无论 L1 先于 L2 执行还是后于 L2 执行，通知内容（subject/description/error）完全相同。** L1 修改 `enabled` 字段不改变 L2 读取的任何数据字段。`company()?->users` 是 L2 内部重新查询的，不受 `$event` 对象状态影响。

---

## 3. Listener 行为分类（修正版）

### 上报/通知类（仅触发外部通知，不修改业务实体状态）

| Listener | 所属事件 | 行为描述 |
|----------|---------|----------|
| [ReportTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/ReportTooManyEmailsSent.php) | TooManyEmailsSent | 调用 `report(new TooManyEmailsSent('...'))` 上报异常到 Laravel 日志/错误追踪 |
| [SendInvalidEmailNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/SendInvalidEmailNotification.php) | InvalidEmailDetected | 向具有 `read-notifications` 权限的管理员发送 `InvalidEmail` 通知（通过 `mail` 和 `database` 通道） |

### 防火墙/安全类（不修改联系人/用户状态，但写入安全日志并可能封禁 IP）

| Listener | 所属事件 | 行为描述 |
|----------|---------|----------|
| [TellFirewallTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/TellFirewallTooManyEmailsSent.php) | TooManyEmailsSent | 1) 调用 `$this->log()` 写入 **防火墙日志表**（middleware = `too_many_emails_sent`，含 user_id）<br>2) 触发 `event(new AttackDetected($log))` → 可能导致 **IP 封禁**<br>3) 不修改任何 contact/user 实体 |

### 状态修改类（修改数据库中的业务实体状态）

| Listener | 所属事件 | 行为描述 |
|----------|---------|----------|
| [DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php) | InvalidEmailDetected | 将 `contact.enabled = false` 并 save()；<br>将 `user.enabled = false` 并 save()（**双重保护**：当前公司和用户所属所有公司均不能只剩 1 个用户） |

---

## 4. Recurring Invoice 自动发送的事务关系分析（代码级证据）

### 完整执行时序

```
RecurringCheck::recur()                    ← DB::transaction 开始
│
├─ 克隆模板创建子 Document                  ← 事务内 save()
│
├─ event(DocumentCreated)                   ← 事务内，同步 listener
│
└─ event(DocumentRecurring)                 ← 事务内，同步 listener
   │
   └─ SendDocumentRecurringNotification::handle()
      │
      ├─ 检查 auto_send == false → return   ← 若为 false，后续全部跳过
      │
      ├─ canNotifyTheContactOfDocument()    ← 前置校验（见第 5 节）
      │  └─ true → $document->contact->notify(new Notification)
      │              │
      │              └─ Abstracts\Notification implements ShouldQueue
      │                 → dispatch 到 notifications 队列
      │                 → queue.after_commit = true
      │                 → 【事务未提交，队列任务尚未执行】
      │
      ├─ event(new DocumentSent)            ← 事务内，同步 listener
      │  └─ MarkDocumentSent::handle()
      │     ├─ $document->status = 'sent'
      │     ├─ $document->save()            ← 【事务内写入 sent 状态】
      │     └─ dispatch(CreateDocumentHistory)
      │
      └─ (可选) 通知管理员用户               ← 同样 dispatch 到队列

DB::transaction 提交 ← 事务结束
│
├─ sync 连接: 队列任务立即执行
│  database/redis/sqs 连接: worker 异步消费
│
└─ Notification 的 toMail() 执行
   │
   └─ 若 SMTP 失败 → 抛出 MailerHttpTransportException
      │
      └─ Handler::report()                  ← 【事务外】
         │
         └─ handleMailerExceptions()
            └─ event(new InvalidEmailDetected($email, $error))
               ├─ DisablePersonDueToInvalidEmail  ← contact/user 禁用
               └─ SendInvalidEmailNotification   ← 管理员通知
```

### 关键代码证据

**1. sent 标记在事务内：**

[RecurringCheck.php#L166-L187](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L166-L187)
```php
protected function recur(Document|Transaction $template, Date $schedule_date): void
{
    DB::transaction(function () use ($template, $schedule_date) {
        // ...
        event(new DocumentRecurring($model));  // 触发 SendDocumentRecurringNotification
    });
}
```

[SendDocumentRecurringNotification.php#L40-L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L40-L42)
```php
$sent = config('type.document.' . $document->type . '.auto_send', DocumentSent::class);
event(new $sent($document));  // 触发 MarkDocumentSent，同步执行
```

[MarkDocumentSent.php#L14-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentSent.php#L14-L25)
```php
public function handle(DocumentMarkedSent|DocumentSent $event): void
{
    if (! in_array($event->document->status, ['partial', 'paid'])) {
        $event->document->status = 'sent';
        // ...
        $event->document->save();  // ← 事务内同步 save
    }
}
```

**2. 邮件通知走队列，事务提交后才执行：**

[queue.php#L33-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/queue.php#L33-L36)
```php
'sync' => [
    'driver' => 'sync',
    'after_commit' => env('QUEUE_AFTER_COMMIT', true),  // ← 事务提交后才执行
],
```

[Abstracts/Notification.php#L11-L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Notification.php#L11-L28)
```php
abstract class Notification extends BaseNotification implements ShouldQueue
{
    use Queueable;

    public function __construct()
    {
        $this->onQueue('notifications');  // ← 所有通知类走 notifications 队列
    }
}
```

**3. InvalidEmailDetected 在事务外触发：**

[Handler.php#L79-L90](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L79-L90)
```php
public function report(Throwable $exception)
{
    if ($exception instanceof MailerHttpTransportException) {
        $email = $this->handleMailerExceptions($exception);
        if (! empty($email)) {
            return;  // ← 不调用 parent::report，避免重复日志
        }
    }
    parent::report($exception);
}
```

[Handler.php#L249-L269](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L249-L269)
```php
protected function handleMailerExceptions(MailerHttpTransportException $exception): string
{
    preg_match("/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,4}/",
               $exception->getMessage(), $matches);
    if (empty($matches[0])) { return ''; }
    $email = $matches[0];
    event(new InvalidEmailDetected($email, $exception->getMessage()));  // ← 事务外
    return $email;
}
```

### 事务关系结论

**状态标记 sent 与联系人/用户禁用之间不存在直接事务关系。**

| 操作 | 执行阶段 | 事务范围 |
|------|---------|---------|
| Document 创建 + save | 事务内 | ✅ `DB::transaction` |
| DocumentSent → MarkDocumentSent → status='sent' + save | 事务内（同步 event） | ✅ `DB::transaction` |
| Notification dispatch 到队列 | 事务内（排队） | ✅ `DB::transaction` |
| Queue worker 消费 → SMTP 发送 | **事务提交后** | ❌ 独立执行 |
| MailerHttpTransportException → Handler::report() | **事务提交后** | ❌ 独立执行 |
| InvalidEmailDetected → DisablePersonDueToInvalidEmail | **事务提交后** | ❌ 独立执行 |

**风险点：**
- 单据被标记为 `sent`，但邮件实际发送失败
- 联系人被禁用，但单据状态不会回滚
- 设计权衡：保证 recurring 单据创建成功，邮件失败作为独立异常处理

---

## 5. InvalidEmailDetected 触发边界条件分析

### 触发路径唯一入口

`InvalidEmailDetected` 仅在 [Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L266) 的 `handleMailerExceptions()` 中触发：
```php
event(new InvalidEmailDetected($email, $exception->getMessage()));
```

该方法仅在捕获 `MailerHttpTransportException` 时调用。因此，`InvalidEmailDetected` 的触发必须满足：
1. 邮件通知已被 dispatch（通过 `notify()` 调用）
2. SMTP 传输层实际发生了异常（非队列消费失败、非模板渲染错误）
3. 异常消息中能正则匹配出邮箱地址

### 各边界场景下的触发判定

| 边界场景 | 代码位置 | 是否触发 InvalidEmailDetected | 说明 |
|----------|---------|------|------|
| **auto_send = false** | [SendDocumentRecurringNotification.php#L28-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L28-L30) | ❌ **不触发** | `handle()` 直接 return，`notify()` 从未调用 |
| **notify_contact = false** | [Documents.php#L246-L248](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L246-L248) | ❌ **不触发** | `canNotifyTheContactOfDocument()` 返回 false，`notify()` 跳过 |
| **contact 为 null** | [Documents.php#L250](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L250) | ❌ **不触发** | `!$document->contact` → 返回 false |
| **contact 已禁用** | [Documents.php#L250](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L250) | ❌ **不触发** | `$document->contact->enabled == 0` → 返回 false |
| **contact_email 为空** | [Documents.php#L254-L256](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L254-L256) | ❌ **不触发** | `empty($document->contact_email)` → 返回 false |
| **RFC 格式校验失败** | [Documents.php#L259-L266](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L259-L266) | ❌ **不触发** | `$validator->isValid(...)` 返回 false → 跳过 notify |
| **DNS 校验失败** | [Documents.php#L259-L266](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L259-L266) | ❌ **不触发** | 同上，在 notify 调用前被拦截 |
| **SMTP 实际传输失败** | Handler.php#L249-L269 | ✅ **触发** | `MailerHttpTransportException` 被捕获，正则提取邮箱成功 |
| **正则无法提取邮箱** | Handler.php#L258-L262 | ❌ **不触发** | `$matches[0]` 为空 → 返回空字符串 → report 中 `!empty($email)` 为 false → 继续 `parent::report`，事件不触发 |

### canNotifyTheContactOfDocument() 完整校验逻辑

[Documents.php#L242-L269](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L242-L269)

```php
public function canNotifyTheContactOfDocument(Document $document): bool
{
    $config = config('type.document.' . $document->type . '.notification');

    // 条件1: 配置允许通知联系人
    if (! $config['notify_contact']) {
        return false;
    }

    // 条件2: 联系人存在且已启用
    if (! $document->contact || ($document->contact->enabled == 0)) {
        return false;
    }

    // 条件3: 联系人邮箱非空
    if (empty($document->contact_email)) {
        return false;
    }

    // 条件4: RFC 格式 + DNS MX 校验通过
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

**关键洞察：所有前置校验（auto_send、RFC/DNS 等）在 `notify()` 调用之前执行。只有通过所有前置校验且 SMTP 实际传输失败时，才会触发 `InvalidEmailDetected`。**

---

## 6. Handler.php 双入口重复触发风险

### 代码路径分析

[Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php) 中有两个独立路径可以调用 `handleMailerExceptions()`：

**路径 A —— report()（第 79-90 行）：**
```php
public function report(Throwable $exception)
{
    if ($exception instanceof MailerHttpTransportException) {
        $email = $this->handleMailerExceptions($exception);  // 触发 InvalidEmailDetected
        if (! empty($email)) {
            return;  // 阻止 parent::report
        }
    }
    parent::report($exception);
}
```

**路径 B —— render() → handleWebExceptions()（第 101-230 行）：**
```php
public function render($request, Throwable $exception)
{
    // ...
    if (config('app.debug') === false) {
        return $this->handleWebExceptions($request, $exception);
    }
    // ...
}

protected function handleWebExceptions($request, $exception)
{
    // ...
    if ($exception instanceof MailerHttpTransportException) {
        $email = $this->handleMailerExceptions($exception);  // 再次触发 InvalidEmailDetected!
        if (! empty($email)) {
            // 渲染 403 响应
        }
    }
    // ...
}
```

### 各请求场景下的触发次数

| 场景 | report() 被调用 | render() 被调用 | handleMailerExceptions() 调用次数 | InvalidEmailDetected 触发次数 |
|------|:---:|:---:|:---:|:---:|
| **Web 请求（debug=false）** | ✅ | ✅ → handleWebExceptions | **2 次** | **2 次** |
| **Web 请求（debug=true）** | ✅ | ❌（parent::render） | **1 次** | **1 次** |
| **API 请求** | ✅ | ✅ → handleApiExceptions（不处理 Mailer 异常） | **1 次** | **1 次** |
| **Queue worker（sync）** | ✅ | ❌（无 HTTP 渲染） | **1 次** | **1 次** |
| **Queue worker（async）** | ✅ | ❌（无 HTTP 渲染） | **1 次** | **1 次** |

### 重复触发的副作用

Web 请求 `debug=false` 场景下：

1. **DisablePersonDueToInvalidEmail 执行 2 次**：
   - 第 1 次：`contact.enabled = false` → save() → 成功
   - 第 2 次：`contact.enabled = false` → save() → 无实际变化（幂等）
   - **但** `setContact()` 中使用 `enabled()->first()` 查询，第 2 次时 contact 已被禁用，查询结果为 null → `$event->contact` 为 null → 第 2 次 disableContact 直接 return
   - **实际上只有第 1 次真正禁用了 contact**

2. **SendInvalidEmailNotification 执行 2 次**：
   - 管理员会收到 **2 封相同的 InvalidEmail 通知邮件**
   - 数据库中会插入 **2 条相同的通知记录**

3. **TellFirewallTooManyEmailsSent 不涉及此问题**（仅在 TooManyEmailsSent 事件中触发，不在 Handler 中）

---

## 7. ContactPerson 邮箱不会被 InvalidEmailDetected 解析或禁用

### 证据

[InvalidEmailDetected.php#L29-L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php#L29-L38)

```php
public function setContact()
{
    // 仅查询 Contact 模型（对应 contacts 表）
    $contact = Contact::email($this->email)->enabled()->first();
    // 不查询 ContactPerson 模型（对应 contact_persons 表）
    // ...
}
```

[ContactPerson.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/ContactPerson.php) 虽然有 `email` 字段和 `scopeEmail()` 方法，但 `InvalidEmailDetected` 的 `setContact()` 明确使用 `Contact::email()` 而非 `ContactPerson::email()`。

### 影响

| 模型 | 表 | 被 setContact() 解析 | 被 DisablePersonDueToInvalidEmail 禁用 |
|------|---|:---:|:---:|
| Contact | `contacts` | ✅ 是 | ✅ 是（`contact.enabled = false`） |
| ContactPerson | `contact_persons` | ❌ 否 | ❌ 否（从不被查询） |
| User | `users` | ✅ 是（通过 setUser()） | ✅ 是（`user.enabled = false`，有保护） |

---

## 8. 用户禁用保护机制（双重校验确认）

[DisablePersonDueToInvalidEmail.php#L26-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php#L26-L49)

```php
public function disableUser(Event $event): void
{
    if (empty($event->user)) {
        return;
    }

    // 保护 1: 检查【当前公司】的用户总数
    $users = company()?->users;
    if ($users && $users->count() <= 1) {
        return;
    }

    // 保护 2: 检查【用户所属所有公司】的用户总数
    $companies = $event->user->companies;
    foreach ($companies as $company) {
        if ($company->users->count() <= 1) {
            return;
        }
    }

    // 只有当所有公司都有 2+ 用户时才禁用
    $event->user->enabled = false;
    $event->user->save();
}
```

**保护逻辑：当前公司和用户所属的每一个公司，都必须有至少 2 个用户，才允许禁用。**

注意：`$users = company()?->users` 不区分启用/禁用状态，统计的是该公司下的**所有用户**（包含已禁用的）。这意味着如果公司有 1 个启用用户 + 5 个禁用用户，`count() = 6`，保护不会触发，启用用户可能被禁用。

---

## 9. 测试矩阵（完整版，含可达路径与预期副作用）

### 场景定义

| 场景 ID | 场景名称 | 触发条件 |
|---------|---------|----------|
| S1 | 过量发送 | 单位时间内发送邮件超过速率限制（月/分钟） |
| S2 | 无效邮箱（SMTP 传输失败） | 邮件通知被 dispatch，SMTP 层抛出 `MailerHttpTransportException` |
| S3 | 缺失联系人 | 邮箱地址在 `contacts` 表和 `users` 表中均无匹配 |
| S4 | 边界条件（邮件发送前置拦截） | auto_send=false / 空邮箱 / 禁用联系人 / RFC/DNS 校验失败 |

### 测试矩阵

| 测试用例 | 场景 | 可达路径 | 预期副作用 |
|---------|------|---------|-----------|
| **TC1-1** | S1 过量发送（超月限制） | `Emails::sendEmail()` → `RateLimiter::attempt` 失败 → `event(TooManyEmailsSent)` | ✅ `ReportTooManyEmailsSent` 上报日志<br>✅ `TellFirewallTooManyEmailsSent` 写防火墙日志 + 触发 `AttackDetected`<br>✅ 返回 `['success' => false]` |
| **TC1-2** | S1 过量发送（超分钟限制） | 同上，per-minute 限制触发 | 同 TC1-1 |
| **TC1-3** | S1 + 白名单 IP | 同上 + `TellFirewallTooManyEmailsSent::skip()` 中 `isWhitelist()` 返回 true | ✅ 事件触发<br>✅ `ReportTooManyEmailsSent` 上报日志<br>❌ `TellFirewallTooManyEmailsSent` 跳过（不写日志，不触发 AttackDetected） |
| **TC1-4** | S1 + 防火墙已禁用 | 同上 + `isDisabled()` 返回 true | 同 TC1-3 |
| **TC2-1** | S2 无效邮箱（contact 存在且启用） | `notify()` → SMTP 失败 → `Handler::report()` → `handleMailerExceptions()` → `event(InvalidEmailDetected)` | ✅ `$event->contact` 非 null<br>✅ `DisablePersonDueToInvalidEmail` 禁用 contact（`enabled=false`）<br>✅ `SendInvalidEmailNotification` 向管理员发通知 |
| **TC2-2** | S2 无效邮箱（contact 存在但已禁用） | 同上 + `setContact()` 中 `enabled()->first()` 返回 null | ✅ `$event->contact` 为 null<br>✅ `DisablePersonDueToInvalidEmail::disableContact()` 直接 return<br>✅ contact 状态不变（已是 disabled）<br>✅ 通知正常发送 |
| **TC2-3** | S2 无效邮箱（user 存在，非最后一个用户） | 同上 + `setUser()` 返回 user + 各公司用户数 ≥ 2 | ✅ `$event->user` 非 null<br>✅ `DisablePersonDueToInvalidEmail` 禁用 user（`enabled=false`）<br>✅ 通知向其他管理员发送（排除该 user 自身） |
| **TC2-4** | S2 无效邮箱（user 存在，当前公司只剩 1 个用户） | 同上 + `company()?->users->count() <= 1` | ✅ 保护 1 触发<br>✅ user **不被禁用**<br>✅ 通知正常发送 |
| **TC2-5** | S2 无效邮箱（user 存在，某所属公司只剩 1 个用户） | 同上 + 某 company 的 `users->count() <= 1` | ✅ 保护 2 触发<br>✅ user **不被禁用**<br>✅ 通知正常发送 |
| **TC2-6** | S2 无效邮箱（Web + debug=false） | 同上 + 请求来自 Web | ✅ `report()` 路径触发 1 次<br>✅ `handleWebExceptions()` 路径触发 1 次<br>✅ **共触发 2 次 InvalidEmailDetected**<br>✅ 管理员收到 **2 封** 通知邮件<br>✅ contact 仅第 1 次被禁用（第 2 次查不到） |
| **TC2-7** | S2 无效邮箱（API 请求） | 同上 + 请求来自 API | ✅ `report()` 路径触发 1 次<br>✅ `handleApiExceptions()` 不处理 Mailer 异常<br>✅ 共触发 **1 次** |
| **TC2-8** | S2 无效邮箱（Queue worker） | 同上 + 队列消费时异常 | ✅ `report()` 路径触发 1 次<br>✅ 无 HTTP render 路径<br>✅ 共触发 **1 次** |
| **TC3-1** | S3 缺失联系人（邮箱不存在） | SMTP 失败 + `setContact()`/`setUser()` 均返回 null | ✅ `$event->contact` = null, `$event->user` = null<br>✅ `DisablePersonDueToInvalidEmail` 不执行任何操作<br>✅ `SendInvalidEmailNotification` 两个方法均 return（无实体可通知）<br>✅ 无管理员通知发出 |
| **TC3-2** | S3 + ContactPerson 邮箱 | 邮箱匹配 `contact_persons` 表但不匹配 `contacts` 表 | ✅ `setContact()` 仅查 `contacts` 表，不查 `contact_persons`<br>✅ `$event->contact` = null<br>✅ ContactPerson **不被禁用**<br>✅ 同 TC3-1 |
| **TC4-1** | S4 auto_send = false | `recurring.auto_send == 0` / `recurring_send_email` 未勾选 | ✅ `SendDocumentRecurringNotification::handle()` L28 return<br>✅ `notify()` **从未调用**<br>✅ InvalidEmailDetected **不可能触发**<br>✅ DocumentSent 事件**仍触发**，sent 状态正常标记 |
| **TC4-2** | S4 空邮箱（contact_email = null） | `canNotifyTheContactOfDocument()` L254 check fails | ✅ `notify()` 跳过<br>✅ InvalidEmailDetected **不可能触发**<br>✅ DocumentSent 事件仍触发 |
| **TC4-3** | S4 联系人已禁用 | `contact.enabled = 0`，canNotifyTheContactOfDocument L250 fails | ✅ 同上 |
| **TC4-4** | S4 RFC 格式校验失败 | 邮箱格式不合法，canNotifyTheContactOfDocument L264 fails | ✅ 同上 |
| **TC4-5** | S4 DNS MX 校验失败 | 域名无 MX 记录，canNotifyTheContactOfDocument L264 fails | ✅ 同上 |
| **TC5-1** | Recurring + S2（完整链路） | `RecurringCheck::recur()` → Document 创建 → DocumentRecurring → notify() → SMTP 失败 | ✅ Document 创建成功（事务内）<br>✅ DocumentSent 触发 → `status='sent'`（事务内 save）<br>✅ 队列任务在事务提交后执行<br>✅ SMTP 失败触发 InvalidEmailDetected（**事务外**）<br>✅ Contact 被禁用<br>✅ 单据状态 **保持 sent，无回滚**<br>✅ 两个操作完全独立 |
| **TC5-2** | Recurring + S3 | 同上 + 邮箱在系统中不存在 | ✅ Document 创建成功 + sent 状态正常<br>✅ InvalidEmailDetected 触发但无禁用操作<br>✅ 管理员不会收到通知 |
| **TC5-3** | Recurring + S4（auto_send=false） | recurring auto_send 关闭 | ✅ Document 创建成功 + sent 状态正常<br>✅ notify() 未调用<br>✅ InvalidEmailDetected 不可能触发 |
| **TC5-4** | Recurring + S2（Web + debug=false） | Recurring 任务通过 Web 界面手动触发 + SMTP 失败 | ✅ 同 TC2-6：InvalidEmailDetected 触发 **2 次**<br>✅ 管理员收到 2 封通知 |
| **TC6-1** | 顺序验证 - InvalidEmailDetected Listener | 验证 DisablePersonDueToInvalidEmail 先于 SendInvalidEmailNotification 执行 | ✅ 顺序为 [L1, L2]<br>✅ L2 读取 `$event->contact->type` 等字段不受 L1 修改 `enabled` 影响<br>✅ **通知内容与执行顺序无关** |
| **TC7-1** | 正则提取失败 | 异常消息中不含合法邮箱格式 | ✅ `handleMailerExceptions()` 返回 `''`<br>✅ `report()` 中 `!empty($email)` 为 false → `parent::report($exception)` 被调用<br>✅ InvalidEmailDetected **不触发** |

### 验证点汇总

| 验证项 | S1 过量发送 | S2 无效邮箱（SMTP 失败） | S3 缺失联系人 | S4 边界条件（前置拦截） |
|-------|:---------:|:----------------------:|:----------:|:-----------------:|
| 事件触发 | `TooManyEmailsSent` | `InvalidEmailDetected` | `InvalidEmailDetected` | ❌ 不触发任何邮件异常事件 |
| 日志上报 | ✅ | ❌（handler 中 return，不走 parent::report） | ❌ | N/A |
| 防火墙日志 | ✅（可能封禁 IP） | ❌ | ❌ | N/A |
| Contact 禁用 | ❌ | ✅（contact 存在且启用时） | ❌ | N/A |
| User 禁用 | ❌ | ✅（双重保护通过时） | ❌ | N/A |
| 管理员通知 | ❌ | ✅（有实体时） | ❌ | N/A |
| ContactPerson 影响 | ❌ | ❌（不查询该表） | ❌ | N/A |
| 重复触发（Web debug=false） | N/A | ✅（2 次） | ✅（2 次） | N/A |
| 单据状态影响 | ❌ | ❌（仍标记 sent） | ❌（仍标记 sent） | ❌（仍标记 sent） |
| 事务原子性 | N/A | ❌（独立执行） | ❌ | N/A |

---

## 关键代码参考

- 事件定义：[TooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/TooManyEmailsSent.php), [InvalidEmailDetected.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php)
- 事件绑定：[Event.php#L124-L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L124-L131)
- Listener - 状态修改：[DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php)
- Listener - 通知：[SendInvalidEmailNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/SendInvalidEmailNotification.php)
- Listener - 上报：[ReportTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/ReportTooManyEmailsSent.php)
- Listener - 防火墙：[TellFirewallTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/TellFirewallTooManyEmailsSent.php)
- 异常处理（双入口）：[Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php)
- 通知实现：[InvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Notifications/Email/InvalidEmail.php)
- 通知基类：[Abstracts/Notification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Notification.php)
- Recurring 调度：[RecurringCheck.php#L164-L188](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L164-L188)
- Recurring 通知发送：[SendDocumentRecurringNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php)
- Sent 标记：[MarkDocumentSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentSent.php)
- 邮件发送前置校验：[Documents.php#L242-L269](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L242-L269)
- 队列配置：[queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/queue.php)
- ContactPerson 模型：[ContactPerson.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/ContactPerson.php)
- 速率限制：[Emails.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Emails.php)
