# Akaunting 客户/供应商联系人及联系人人员分析报告

## 1. CreateContact 何时创建 ContactPerson

### 创建流程
在 [CreateContact.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateContact.php) 的 `handle()` 方法中，创建 ContactPerson 的时机和条件如下：

```php
public function handle(): Contact
{
    event(new ContactCreating($this->request));

    \DB::transaction(function () {
        // ... 创建 Contact 主体
        $this->model = Contact::create($this->request->all());
        
        // ... 处理 logo 上传
        
        // 关键：在 Contact 创建后，dispatch CreateContactPersons job
        $this->dispatch(new CreateContactPersons($this->model, $this->request));
    });

    event(new ContactCreated($this->model, $this->request));

    return $this->model;
}
```

### CreateContactPersons 的创建条件
在 [CreateContactPersons.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateContactPersons.php) 中：

```php
public function handle()
{
    // 条件1：request 中必须存在 contact_persons 数组
    if (empty($this->request['contact_persons'])) {
        return false;
    }

    \DB::transaction(function () {
        foreach ($this->request['contact_persons'] as $person) {
            // 条件2：每个 person 至少有 name/email/phone 之一不为空
            if (empty($person['name']) && empty($person['email']) && empty($person['phone'])) {
                continue;
            }

            ContactPerson::create([
                'company_id' => $this->contact->company_id,
                'type' => $this->contact->type,
                'contact_id' => $this->contact->id,
                'name' => $person['name'] ?? null,
                'email' => $person['email'] ?? null,
                'phone' => $person['phone'] ?? null,
                'created_from' => $this->request['created_from'],
                'created_by' => $this->request['created_by'],
            ]);
        }
    });

    return $this->contact->contact_persons;
}
```

### 更新时的处理
在 [UpdateContact.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/UpdateContact.php#L44-L46) 中，更新时会先删除所有关联的 contact_persons，然后重新创建：

```php
$this->deleteRelationships($this->model, ['contact_persons']);
$this->dispatch(new CreateContactPersons($this->model, $this->request));
```

---

## 2. 主联系人 email 与 person email 在模型关系上如何区分

### 模型关系
```
Contact (1) ────> (*) ContactPerson
  |                     |
  ├─ email              ├─ email (人员独立邮箱)
  ├─ enabled            ├─ (无 enabled 字段)
  └─ name               └─ name
```

### 字段与表结构对比

| 特性 | Contact (主联系人) | ContactPerson (联系人人员) |
|------|-------------------|---------------------------|
| 表名 | `contacts` | `contact_persons` |
| email 字段 | 有 ([Contact.php#L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Contact.php#L53)) | 有 ([ContactPerson.php#L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/ContactPerson.php#L30)) |
| enabled 字段 | 有 | **无** ([迁移文件](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2023_10_03_000000_core_v310.php#L16-L32)) |
| 软删除 | 有 | 有 (`deleted_at`) |
| 关联关系 | `hasMany(ContactPerson::class)` | `belongsTo(Contact::class)` |
| 唯一约束 | `(company_id, type, email, deleted_at)` 联合唯一 | 无唯一约束 |

**核心区分**：Contact 有 `enabled` 字段可被禁用/启用，而 ContactPerson 在表结构中没有 `enabled` 字段，只有软删除 `deleted_at`。Contact 表在 [2019_11_16_000000_core_v2.php#L36-L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L36-L61) 中定义了 `(company_id, type, email, deleted_at)` 联合唯一索引，这意味着同公司、同类型、同邮箱、未软删除的 Contact 只能有一条；ContactPerson 无此约束。两者各自存储独立的 email，通过 `withPersons()` 方法合并用于邮件发送。

### 邮箱合并逻辑
在 [Contact.php#L168-L185](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Contact.php#L168-L185) 的 `withPersons()` 方法中：

```php
public function withPersons()
{
    $contacts = collect();

    // 1. 首先添加主联系人（如果有email）
    if (! empty($this->email)) {
        $contacts->push($this);
    }

    // 2. 然后添加所有有email的联系人人员
    $contact_persons = $this->contact_persons()->whereNotNull('email')->get();

    if ($contact_persons) {
        foreach ($contact_persons as $contact_person) {
            $contacts->push($contact_person);
        }
    }

    return $contacts;
}
```

该方法用于发送邮件时收集所有收件人，例如在 [SendDocumentAsCustomMail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/SendDocumentAsCustomMail.php#L42) 中使用。

---

## 3. Sales Customers 导入如何映射到 Contact

### 导入类结构
Sales Customers 导入由 [app/Imports/Sales/Customers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Customers.php) 处理，继承自 [Abstracts/Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php)。

### 映射逻辑
```php
public function map($row): array
{
    $row = parent::map($row);  // 调用父类通用映射

    // 国家代码转换
    $country = array_search($row['country'], trans('countries'));

    // 强制类型为 customer
    $row['type'] = 'customer';
    $row['country'] = !empty($country) ? $country : null;
    
    // 解析分类ID（收入分类）
    $row['category_id'] = $this->getCategoryId($row, 'income');
    
    // 解析货币代码
    $row['currency_code'] = $this->getCurrencyCode($row);
    
    // 用户ID：检查 can_login 字段是否存在（而非其真值），且 email 字段存在
    $row['user_id'] = null;
    if (isset($row['can_login']) && isset($row['email'])) {
        $row['user_id'] = user_model_class()::where('email', $row['email'])->first()?->id ?? null;
    }

    return $row;
}
```

**关键细节**：`can_login` 的检查使用的是 `isset($row['can_login'])`，即**只检查字段在行数据中是否存在，不检查其值的真假**。只要 Excel 行中包含 `can_login` 列（无论值是 `true`、`false`、`0`、`1` 还是空字符串），且同时存在 `email` 字段，就会尝试关联对应用户。

### 模型创建
```php
public function model(array $row)
{
    if (self::hasRow($row)) {  // 检查是否已存在（去重）
        return;
    }

    return new Model($row);  // Model = Contact::class
}
```

### 去重逻辑（hasRow() 实现细节）
去重由父类 [Abstracts/Import.php#L233-L270](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L233-L270) 的 `hasRow()` 方法实现：

```php
public function hasRow($row)
{
    if (empty($this->model) || empty($this->columns)) {
        return false;
    }

    // 首次调用时，一次性从数据库取出所有已存在记录（仅取 columns 字段），缓存到 $this->has_row
    if (! $this->has_row || ! $this->has_row instanceof $this->model) {
        $this->has_row = $this->model::withoutEvents(function () {
            return $this->model::get($this->columns)->each(function ($data) {
                $data->setAppends([]);
                $data->unsetRelations();
            });
        });
    }

    // 用当前行的 columns 字段组成搜索数组，与缓存做 in_array 比较
    $search_value = [];
    foreach ($this->columns as $key) {
        $search_value[$key] = isset($row[$key]) ? $row[$key] : null;
    }

    return in_array($search_value, $this->has_row->toArray());
}
```

**去重说明**：
- **基于数据库缓存**：导入开始时一次性从数据库查询所有已存在的 Contact（仅取 `type`、`name`、`email` 三个字段），缓存到 `$this->has_row`。后续每行都与该缓存做 `in_array` 比较。
- **不处理同一导入文件内部的重复行**：缓存只在首次调用时从数据库加载一次，**不会随着导入过程中新增的行而更新**。因此，如果同一个 Excel 文件内出现两行相同的 `type/name/email`，由于这两行在导入前数据库中都不存在，`hasRow()` 对两者都会返回 `false`，两行都会被写入模型（后续会由数据库的联合唯一索引 `(company_id, type, email, deleted_at)` 拦截第二行导致写入失败，或因导入器的批次处理策略跳过）。
- **跳过已存在行**：对于与数据库中已有记录 `type/name/email` 完全匹配的行，`hasRow()` 返回 `true`，`model()` 直接 `return;`，不会创建新模型。

### 重要说明
**Sales Customers 导入只创建 Contact 主记录，不创建 ContactPerson 记录。** 导入文件中没有 contact_persons 相关字段的映射。

---

## 4. InvalidEmailDetected listener 如何找到并禁用对象

### 事件触发入口（Handler 中两处，条件不同）

在 [Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php) 中，`InvalidEmailDetected` 事件有两个触发入口，两者都通过调用 `handleMailerExceptions()` 触发事件，但进入条件不同：

**入口1：`report()` 方法**（第79-90行）—— 任何环境下只要异常是 `MailerHttpTransportException` 就会处理：
```php
public function report(Throwable $exception)
{
    if ($exception instanceof MailerHttpTransportException) {
        $email = $this->handleMailerExceptions($exception);
        // handleMailerExceptions 内部触发 event(new InvalidEmailDetected(...))
        if (! empty($email)) {
            return;  // 成功提取邮箱后，不再走 parent::report()
        }
    }
    parent::report($exception);
}
```

**入口2：`render()` 方法**（第101-112行）—— 仅在 **非 API 请求 且 `app.debug = false`** 的 web 异常渲染路径才会进入 `handleWebExceptions()`，进而调用 `handleMailerExceptions()`：
```php
public function render($request, Throwable $exception)
{
    if (request_is_api($request)) {
        return $this->handleApiExceptions($request, $exception);
        // API 路径：不调用 handleMailerExceptions，不触发 InvalidEmailDetected
    }

    if (config('app.debug') === false) {
        return $this->handleWebExceptions($request, $exception);
        // handleWebExceptions 第211-227行：检查 MailerHttpTransportException
        // 调用 handleMailerExceptions 触发事件
    }

    return parent::render($request, $exception);
    // app.debug = true 时：也不调用 handleMailerExceptions
}
```

两处入口共同调用的 `handleMailerExceptions()`（第249-269行）：

```php
protected function handleMailerExceptions(MailerHttpTransportException $exception): string
{
    preg_match("/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,4}/", $exception->getMessage(), $matches);

    if (empty($matches[0])) {
        return '';
    }

    $email = $matches[0];
    event(new InvalidEmailDetected($email, $exception->getMessage()));
    return $email;
}
```

**触发条件汇总**：

| 入口 | 触发条件 | 是否调用 handleMailerExceptions |
|------|---------|-------------------------------|
| `report()` | 异常是 `MailerHttpTransportException`（任何环境） | 是 |
| `render()` | 非 API 请求 **且** `app.debug = false`（web 路径） | 是 |
| `render()` | API 请求 | 否 |
| `render()` | 非 API 但 `app.debug = true` | 否 |

### 事件查找对象
在 [InvalidEmailDetected.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php) 的构造函数中，**只在 Contact 表和 User 表中查找**：

```php
public function setContact()
{
    // 只在 Contact 表中查找 enabled 的记录
    $contact = Contact::email($this->email)->enabled()->first();
    if (empty($contact)) { return; }
    $this->contact = $contact;
}

public function setUser()
{
    // 只在 User 表中查找 enabled 的记录
    $user = user_model_class()::email($this->email)->enabled()->first();
    if (empty($user)) { return; }
    $this->user = $user;
}
```

**⚠️ 关于 ContactPerson**：事件**不会按 ContactPerson 本身查找或禁用**。但如果某个 ContactPerson 的 email 恰好与某个 Contact 或 User 的 email 相同，那么同邮箱的 Contact 或 User 会被找到并禁用——此时被禁用的是 Contact/User 对象，而非 ContactPerson 本身。

### 监听器注册与执行
在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L128-L131) 中：

```php
\App\Events\Email\InvalidEmailDetected::class => [
    \App\Listeners\Email\DisablePersonDueToInvalidEmail::class,
    \App\Listeners\Email\SendInvalidEmailNotification::class,
],
```

### 禁用逻辑（完整）
在 [DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php) 中，会依次尝试禁用 Contact 和 User：

**禁用 Contact**：
```php
public function disableContact(Event $event): void
{
    if (empty($event->contact)) {
        return;
    }
    $event->contact->enabled = false;
    $event->contact->save();
}
```

**禁用 User（含用户数量保护）**：
```php
public function disableUser(Event $event): void
{
    if (empty($event->user)) {
        return;
    }

    // 保护条件1：如果当前公司只剩下1个用户，不禁用
    $users = company()?->users;
    if ($users && $users->count() <= 1) {
        return;
    }

    // 保护条件2：遍历该用户所属的所有公司，
    // 如果任何一个公司只剩下1个用户，不禁用
    $companies = $event->user->companies;
    foreach ($companies as $company) {
        if ($company->users->count() <= 1) {
            return;
        }
    }

    // 通过所有保护条件后才禁用
    $event->user->enabled = false;
    $event->user->save();
}
```

### 查找与禁用范围总结

| 查找目标 | 表 | 条件 | 能否被禁用 |
|---------|---|------|-----------|
| Contact | `contacts` | `email = ? AND enabled = 1` | 能（直接设 enabled=false） |
| User | `users` | `email = ? AND enabled = 1` | 能（需通过用户数量保护） |
| ContactPerson | `contact_persons` | 不查找 | 不能（没有 enabled 字段，也不被查找） |

---

## 5. 主联系人仍启用但某个 ContactPerson 被软删除/移除的场景

### 源码层面的事实
- **ContactPerson 表不存在 `enabled` 字段**（见 [迁移文件](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2023_10_03_000000_core_v310.php#L16-L32)），只有 `deleted_at` 软删除字段
- **InvalidEmailDetected 不会自动禁用 ContactPerson**——既不查找 ContactPerson 表，ContactPerson 也没有 enabled 字段可供禁用
- 因此，"主联系人启用但某个 person 被禁用"的场景在当前源码中**不存在自动实现路径**

### 替代场景：主联系人启用但某个 ContactPerson 被软删除/移除（手动操作，非无效邮箱 listener 自动触发）

#### 场景描述
某公司客户 "ABC科技有限公司" 有3个联系人：
- 主联系人：张总 `zhangzong@abctech.com`（主 Contact，enabled=1）
- 联系人人员1：李经理 `lijingli@abctech.com`（ContactPerson，存在）
- 联系人人员2：王助理 `wangzhuli@abctech.com`（ContactPerson，已离职，需移除）

#### 操作步骤

1. **创建客户及联系人**（通过界面或API调用 CreateContact job，注意补充有效 `currency_code` 以通过 [Contact 请求验证](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/Contact.php#L60) 的 `required|string|currency` 规则）：
```php
$request = [
    'type' => 'customer',
    'name' => 'ABC科技有限公司',
    'email' => 'zhangzong@abctech.com',
    'currency_code' => 'CNY',    // 必须：Contact 请求验证要求 currency_code 为有效货币代码
    'enabled' => 1,
    'contact_persons' => [
        ['name' => '李经理', 'email' => 'lijingli@abctech.com', 'phone' => '13800138001'],
        ['name' => '王助理', 'email' => 'wangzhuli@abctech.com', 'phone' => '13800138002'],
    ],
];
$contact = $this->dispatch(new CreateContact($request));
```

2. **假设向王助理发送邮件失败**，触发 `InvalidEmailDetected` 事件：
   - 系统在 Contact 表查找 `wangzhuli@abctech.com` → 未找到（因为这是 ContactPerson 的 email，Contact 表的 email 是 `zhangzong@abctech.com`）
   - 系统在 User 表查找 → 未找到
   - **结果：没有任何对象被禁用**（ContactPerson 不被查找，也没有同邮箱的 Contact/User）

3. **管理员手动软删除王助理**（通过 DeleteContactPerson job）：
```php
$wangZhuli = $contact->contact_persons()
    ->where('email', 'wangzhuli@abctech.com')
    ->first();
$this->dispatch(new DeleteContactPerson($wangZhuli));
```

#### 最终状态
| 对象 | 邮箱 | 状态 |
|------|------|------|
| Contact (ABC科技有限公司) | zhangzong@abctech.com | enabled = 1（启用） |
| ContactPerson (李经理) | lijingli@abctech.com | 存在（未删除） |
| ContactPerson (王助理) | wangzhuli@abctech.com | deleted_at 有值（已软删除） |

#### 系统行为
- 发送发票邮件时，`withPersons()` 只会返回张总和李经理（王助理已软删除，不在查询结果中）
- 主联系人 ABC科技有限公司 仍然完全启用，可以正常创建发票、账单等
- 此场景是**管理员手动软删除**，不是 InvalidEmailDetected listener 自动触发的结果

### 同邮箱 Contact 被禁用的场景（ContactPerson 间接影响）

如果 ContactPerson 的 email 与某个 Contact 的 email 恰好相同，InvalidEmailDetected 会禁用那个 Contact：

- 主联系人（Contact）：`wangzhuli@abctech.com`，enabled=1
- ContactPerson（王助理）：`wangzhuli@abctech.com`（与主联系人同邮箱）
- 发送邮件失败 → 触发事件 → 在 Contact 表找到 `wangzhuli@abctech.com` → 禁用该 Contact
- 结果：主联系人被禁用（enabled=0），ContactPerson 不受影响（仍存在但无 enabled 概念）

---

## 总结

| 问题 | 关键结论 |
|------|---------|
| CreateContact 何时创建 ContactPerson | Contact 创建后，当 request 包含 contact_persons 数组且每条记录至少有 name/email/phone 之一不为空时 |
| 主联系人 email 与 person email 区分 | Contact 有 enabled 字段和 `(company_id,type,email,deleted_at)` 联合唯一索引；ContactPerson 没有 enabled 字段也没有唯一约束；两者各自存储独立 email，通过 withPersons() 合并用于邮件发送 |
| Sales Customers 导入映射 | 只创建 Contact，type 强制为 'customer'；can_login 检查用 isset（只判断字段存在，不检查真值）；不创建 ContactPerson |
| Sales Customers 去重 | hasRow() 基于导入前的数据库缓存（一次性加载 type/name/email）做 in_array 比对；同一导入文件内部重复行不会被 hasRow() 识别，需由数据库联合唯一索引拦截 |
| InvalidEmailDetected 触发入口 | report() 无条件处理 MailerHttpTransportException；render() 仅在非 API 且 app.debug=false 的 web 路径进入 handleWebExceptions() 后才调用 handleMailerExceptions() |
| InvalidEmailDetected 查找与禁用 | 只查找 Contact 和 User 表（enabled=1），禁用 Contact（直接）和 User（需通过用户数量保护）；不查找 ContactPerson；同邮箱的 Contact/User 会被禁用（而非 ContactPerson） |
| 主联系人启用但 person 被"禁用"的场景 | 源码不存在 ContactPerson.enabled；替代方案是手动软删除 ContactPerson，这不是 InvalidEmailDetected listener 自动触发的；若 ContactPerson email 与 Contact/User 同邮箱，会禁用 Contact/User（而非 ContactPerson） |
