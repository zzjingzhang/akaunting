# 邮件发送异常事件对系统状态的影响分析

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
- `user_id` (int): 触发邮件发送超限的用户ID

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
        $this->setContact();
        $this->setUser();
    }
}
```

**携带数据：**
- `email` (string): 无效的邮箱地址
- `error` (string): 邮件发送失败的错误信息
- `contact` (Contact|null): 关联的联系人实体（自动通过邮箱查找）
- `user` (User|null): 关联的用户实体（自动通过邮箱查找）

### 数据差异对比

| 数据字段 | TooManyEmailsSent | InvalidEmailDetected |
|---------|-------------------|----------------------|
| user_id | ✅ (int) | ❌ |
| email | ❌ | ✅ (string) |
| error | ❌ | ✅ (string) |
| contact | ❌ | ✅ (Contact/null) |
| user | ❌ | ✅ (User/null) |

**核心差异：**
- `TooManyEmailsSent` 是**速率限制事件**，仅需识别触发者（user_id）
- `InvalidEmailDetected` 是**业务异常事件**，需要携带邮箱、错误信息以及自动解析关联的业务实体（联系人和用户）

---

## 2. Event Provider 中事件与 Listener 绑定关系

[Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L124-L131)

```php
protected $listen = [
    \App\Events\Email\TooManyEmailsSent::class => [
        \App\Listeners\Email\ReportTooManyEmailsSent::class,
        \App\Listeners\Email\TellFirewallTooManyEmailsSent::class,
    ],
    \App\Events\Email\InvalidEmailDetected::class => [
        \App\Listeners\Email\DisablePersonDueToInvalidEmail::class,
        \App\Listeners\Email\SendInvalidEmailNotification::class,
    ],
];
```

### TooManyEmailsSent 事件绑定的 Listener（按顺序）
1. `ReportTooManyEmailsSent` - 上报异常日志
2. `TellFirewallTooManyEmailsSent` - 通知防火墙系统

### InvalidEmailDetected 事件绑定的 Listener（按顺序）
1. `DisablePersonDueToInvalidEmail` - 禁用关联的联系人和用户
2. `SendInvalidEmailNotification` - 向管理员发送通知

### 顺序影响分析

**TooManyEmailsSent 顺序影响：低**
- 两个 listener 都是通知类操作，互不依赖
- 先上报后通知防火墙符合常规流程，但调换顺序不会导致功能错误

**InvalidEmailDetected 顺序影响：高**
- `DisablePersonDueToInvalidEmail` 必须在 `SendInvalidEmailNotification` 之前执行
- 原因：通知需要基于最新的实体状态（已禁用）发送给管理员
- 如果顺序颠倒，通知可能在实体被禁用前发送，导致通知内容与实际状态不一致

---

## 3. Listener 行为分类

### 上报/通知类（仅触发外部通知，不修改业务状态）

| Listener | 所属事件 | 行为描述 |
|----------|---------|----------|
| [ReportTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/ReportTooManyEmailsSent.php) | TooManyEmailsSent | 调用 `report()` 上报异常到日志系统 |
| [TellFirewallTooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/TellFirewallTooManyEmailsSent.php) | TooManyEmailsSent | 触发防火墙 `AttackDetected` 事件，记录攻击日志 |
| [SendInvalidEmailNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/SendInvalidEmailNotification.php) | InvalidEmailDetected | 向具有 `read-notifications` 权限的管理员发送邮件通知 |

### 状态修改类（修改数据库中的业务实体状态）

| Listener | 所属事件 | 行为描述 |
|----------|---------|----------|
| [DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php) | InvalidEmailDetected | 将 `contact.enabled = false` 并保存；<br>将 `user.enabled = false` 并保存（最后一个用户除外） |

---

## 4. Recurring Invoice 自动发送的事务关系分析

### 执行流程

1. **调度触发**：`recurring:check` 命令在 [RecurringCheck.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php) 中执行

2. **事务创建**：在 `recur()` 方法中使用 `DB::transaction` 包裹
   ```php
   DB::transaction(function () use ($template, $schedule_date) {
       if (! $model = $this->getModel($template, $schedule_date)) {
           return;
       }
       event(new DocumentCreated($model, request()));
       event(new DocumentRecurring($model));
   });
   ```

3. **邮件发送**：在 [SendDocumentRecurringNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentRecurringNotification.php#L36-L42) 中触发邮件通知
   ```php
   if ($this->canNotifyTheContactOfDocument($document)) {
       $document->contact->notify(new $notification($document, "{$document->type}_recur_customer", $attach_pdf));
   }
   event(new $sent($document));  // DocumentSent 事件
   ```

4. **状态标记**：[MarkDocumentSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentSent.php) 在 `DocumentSent` 事件中更新单据状态
   ```php
   $event->document->status = 'sent';
   $event->document->save();
   ```

5. **无效邮箱捕获**：在 [Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L249-L269) 全局异常处理器中捕获
   ```php
   protected function handleMailerExceptions(MailerHttpTransportException $exception): string
   {
       preg_match("/.../", $exception->getMessage(), $matches);
       $email = $matches[0];
       event(new InvalidEmailDetected($email, $exception->getMessage()));
       return $email;
   }
   ```

### 事务关系结论

**状态标记 sent 与联系人禁用之间不存在直接事务关系**

原因：
1. `DocumentSent` 事件触发 `MarkDocumentSent` 修改单据状态为 `sent`
2. 邮件发送失败时，异常被全局处理器捕获，触发 `InvalidEmailDetected` 事件
3. `DisablePersonDueToInvalidEmail` 在全局异常处理流程中执行，不在 `recur()` 的数据库事务范围内
4. 两者是独立的异步操作链，没有事务包裹保证原子性

**风险点：**
- 单据可能被标记为 `sent`，但实际上邮件发送失败
- 联系人被禁用，但单据状态不会回滚
- 这是设计上的权衡：保证 recurring 单据创建成功，邮件失败作为单独的异常处理

---

## 5. 测试矩阵

### 场景定义

| 场景ID | 场景名称 | 触发条件 |
|--------|---------|----------|
| S1 | 过量发送 | 单位时间内发送邮件超过速率限制 |
| S2 | 无效邮箱 | 邮件发送时触发 `MailerHttpTransportException` |
| S3 | 缺失联系人 | 邮箱地址在系统中无对应的 contact 或 user |

### 测试矩阵

| 测试用例 | 场景 | 触发方式 | 预期结果 |
|---------|------|---------|---------|
| TC1-1 | S1 过量发送 | 快速连续调用 `sendEmail()` 超过月/分钟限制 | ✅ TooManyEmailsSent 事件触发<br>✅ ReportTooManyEmailsSent 上报日志<br>✅ TellFirewallTooManyEmailsSent 通知防火墙<br>✅ 返回 `['success' => false]` |
| TC1-2 | S1 + 白名单 | 白名单IP触发超限 | ✅ 事件触发<br>✅ TellFirewallTooManyEmailsSent 中 `skip()` 返回 true<br>✅ 防火墙不记录攻击 |
| TC2-1 | S2 无效邮箱（联系人存在） | 发送邮件到已存在 contact 的无效邮箱 | ✅ InvalidEmailDetected 事件触发<br>✅ `event->contact` 不为 null<br>✅ DisablePersonDueToInvalidEmail 禁用 contact（enabled=false）<br>✅ SendInvalidEmailNotification 向管理员发送通知<br>✅ 异常返回友好错误信息 |
| TC2-2 | S2 无效邮箱（用户存在） | 发送邮件到已存在 user 的无效邮箱 | ✅ InvalidEmailDetected 事件触发<br>✅ `event->user` 不为 null<br>✅ DisablePersonDueToInvalidEmail 禁用 user（enabled=false，非最后一个用户时）<br>✅ SendInvalidEmailNotification 向管理员发送通知（排除无效邮箱用户） |
| TC2-3 | S2 无效邮箱（最后一个用户） | 发送邮件到系统最后一个用户的无效邮箱 | ✅ InvalidEmailDetected 事件触发<br>✅ `event->user` 不为 null<br>✅ DisablePersonDueToInvalidEmail 不禁用 user（保护机制）<br>✅ 通知正常发送 |
| TC3-1 | S3 缺失联系人 | 发送邮件到系统中不存在的邮箱 | ✅ InvalidEmailDetected 事件触发<br>✅ `event->contact` 为 null<br>✅ `event->user` 为 null<br>✅ DisablePersonDueToInvalidEmail 不执行任何禁用操作<br>✅ SendInvalidEmailNotification 不发送任何通知（无关联实体） |
| TC3-2 | S2 + S3 无效邮箱（联系人已禁用） | 发送邮件到已禁用 contact 的邮箱 | ✅ InvalidEmailDetected 事件触发<br>✅ `event->contact` 为 null（因 `enabled()` 过滤）<br>✅ 行为同 S3 |
| TC4-1 | Recurring + S2 | Recurring invoice 自动发送到无效邮箱 | ✅ Document 创建成功（事务内）<br>✅ DocumentSent 事件触发，状态标记为 sent<br>✅ 邮件发送失败触发 InvalidEmailDetected<br>✅ Contact 被禁用<br>✅ 两个操作独立，无回滚<br>✅ 单据状态保持 sent |
| TC4-2 | Recurring + S3 | Recurring invoice 自动发送到不存在的邮箱 | ✅ Document 创建成功，状态标记为 sent<br>✅ InvalidEmailDetected 触发但无禁用操作<br>✅ 管理员不会收到通知 |
| TC5-1 | 顺序验证 - InvalidEmailDetected | 监听执行顺序验证 | ✅ DisablePersonDueToInvalidEmail 先执行<br>✅ SendInvalidEmailNotification 后执行<br>✅ 通知中能反映实体已被禁用的状态 |

### 验证点汇总

| 验证项 | S1 过量发送 | S2 无效邮箱 | S3 缺失联系人 |
|-------|------------|------------|--------------|
| 事件触发 | TooManyEmailsSent | InvalidEmailDetected | InvalidEmailDetected |
| 日志上报 | ✅ | ❌ | ❌ |
| 防火墙通知 | ✅ | ❌ | ❌ |
| 联系人禁用 | ❌ | ✅ (contact存在时) | ❌ |
| 用户禁用 | ❌ | ✅ (user存在且非最后一个) | ❌ |
| 管理员通知 | ❌ | ✅ (有实体时) | ❌ |
| 事务回滚 | N/A | ❌ (独立执行) | ❌ |
| 单据状态影响 | ❌ | ❌ (仍标记sent) | ❌ |

---

## 关键代码参考

- 事件定义：[TooManyEmailsSent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/TooManyEmailsSent.php), [InvalidEmailDetected.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php)
- 事件绑定：[Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L124-L131)
- 监听器：[DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php), [SendInvalidEmailNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/SendInvalidEmailNotification.php)
- 异常处理：[Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L249-L269)
- Recurring 调度：[RecurringCheck.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L164-L188)
