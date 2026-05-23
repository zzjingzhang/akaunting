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
    
    // 用户ID：如果可以登录且邮箱存在，则关联对应用户
    $row['user_id'] = null;
    if (isset($row['can_login']) && isset($row['email'])) {
        $row['user_id'] = user_model_class()::where('email', $row['email'])->first()?->id ?? null;
    }

    return $row;
}
```

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

### 去重逻辑
通过 `$columns = ['type', 'name', 'email']` 进行重复检测，相同类型、名称和邮箱的记录不会重复导入。

### 重要说明
**Sales Customers 导入只创建 Contact 主记录，不创建 ContactPerson 记录。** 导入文件中没有 contact_persons 相关字段的映射。

---

## 4. InvalidEmailDetected listener 如何找到并禁用对象

### 事件触发流程
当邮件发送失败时，异常处理流程如下：

1. **异常捕获**：在 [Handler.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exceptions/Handler.php#L249-L269) 的 `handleMailerExceptions()` 中：
```php
protected function handleMailerExceptions(MailerHttpTransportException $exception): string
{
    // 从异常消息中提取邮箱地址
    preg_match("/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,4}/", $exception->getMessage(), $matches);

    if (empty($matches[0])) {
        return '';
    }

    $email = $matches[0];
    
    // 触发 InvalidEmailDetected 事件
    event(new InvalidEmailDetected($email, $exception->getMessage()));

    return $email;
}
```

2. **事件构造**：在 [InvalidEmailDetected.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Email/InvalidEmailDetected.php) 中查找对象：
```php
public function setContact()
{
    // 只在 Contact 表中查找 enabled 的记录
    $contact = Contact::email($this->email)->enabled()->first();

    if (empty($contact)) {
        return;
    }

    $this->contact = $contact;
}

public function setUser()
{
    // 只在 User 表中查找 enabled 的记录
    $user = user_model_class()::email($this->email)->enabled()->first();
    // ...
}
```

3. **监听器注册**：在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L128-L131) 中：
```php
\App\Events\Email\InvalidEmailDetected::class => [
    \App\Listeners\Email\DisablePersonDueToInvalidEmail::class,
    \App\Listeners\Email\SendInvalidEmailNotification::class,
],
```

4. **禁用逻辑**：在 [DisablePersonDueToInvalidEmail.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Email/DisablePersonDueToInvalidEmail.php) 中：
```php
public function disableContact(Event $event): void
{
    if (empty($event->contact)) {
        return;
    }

    // 将 Contact 的 enabled 设为 false
    $event->contact->enabled = false;
    $event->contact->save();
}
```

### 关键局限
**⚠️ 重要发现**：当前实现只在 `Contact` 表和 `User` 表中查找邮箱，**不会在 `ContactPerson` 表中查找**。因此，ContactPerson 的邮箱失效不会触发任何禁用操作。

---

## 5. 主联系人仍启用但某个 person 被禁用的场景

### 当前系统的实际情况
基于代码分析，当前系统存在以下限制：

1. **ContactPerson 没有 enabled 字段**（[迁移文件](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2023_10_03_000000_core_v310.php#L16-L32)），只有软删除 `deleted_at`
2. **InvalidEmailDetected 不检查 ContactPerson**，只检查 Contact 和 User 表

### 可实现的场景（手动操作）
虽然系统不会自动禁用 ContactPerson，但可以通过手动操作实现"主联系人启用、某个人员被移除"的状态：

#### 场景描述
**某公司客户 "ABC科技有限公司" 有3个联系人：**
- 主联系人：张总 `zhangzong@abctech.com`（有效，启用）
- 联系人人员1：李经理 `lijingli@abctech.com`（有效）
- 联系人人员2：王助理 `wangzhuli@abctech.com`（已离职，邮箱失效）

#### 操作步骤
1. **创建客户及联系人**（通过界面或API）：
```php
// CreateContact job 执行
$request = [
    'type' => 'customer',
    'name' => 'ABC科技有限公司',
    'email' => 'zhangzong@abctech.com',  // 主联系人邮箱
    'enabled' => 1,
    'contact_persons' => [
        ['name' => '李经理', 'email' => 'lijingli@abctech.com', 'phone' => '13800138001'],
        ['name' => '王助理', 'email' => 'wangzhuli@abctech.com', 'phone' => '13800138002'],
    ],
    // ... 其他字段
];
$contact = $this->dispatch(new CreateContact($request));
```

2. **发送邮件给王助理时失败**，触发 `InvalidEmailDetected` 事件：
   - 系统在 Contact 表查找 `wangzhuli@abctech.com` → 未找到（因为这是 ContactPerson 的邮箱）
   - 系统在 User 表查找 → 未找到
   - **结果：没有任何对象被禁用**

3. **管理员手动删除王助理**（通过 DeleteContactPerson job）：
```php
// 找到王助理的 ContactPerson 记录
$wangZhuli = $contact->contact_persons()->where('email', 'wangzhuli@abctech.com')->first();

// 软删除
$this->dispatch(new DeleteContactPerson($wangZhuli));
```

#### 最终状态
| 对象 | 邮箱 | 状态 |
|------|------|------|
| Contact (ABC科技有限公司) | zhangzong@abctech.com | enabled = 1 (启用) |
| ContactPerson (李经理) | lijingli@abctech.com | 存在 (未删除) |
| ContactPerson (王助理) | wangzhuli@abctech.com | deleted_at 有值 (已软删除) |

#### 系统行为
- 发送发票邮件时，`withPersons()` 只会返回张总和李经理（因为王助理已软删除，不会被查询到）
- 主联系人 ABC科技有限公司 仍然完全启用，可以正常创建发票、账单等

### 理想场景（系统增强建议）
如果希望系统自动检测并禁用 ContactPerson 的无效邮箱，需要：

1. 为 `contact_persons` 表添加 `enabled` 字段
2. 在 `InvalidEmailDetected` 事件中增加 ContactPerson 的查找逻辑
3. 在 `DisablePersonDueToInvalidEmail` 监听器中增加禁用 ContactPerson 的逻辑

```php
// 建议的增强代码
public function setContactPerson()
{
    $contactPerson = ContactPerson::email($this->email)->first();
    if ($contactPerson) {
        $this->contact_person = $contactPerson;
    }
}
```

---

## 总结

| 问题 | 关键结论 |
|------|---------|
| CreateContact 何时创建 ContactPerson | Contact 创建后，当 request 包含 contact_persons 数组且每条记录至少有 name/email/phone 之一时 |
| 主联系人 email 与 person email 区分 | Contact 有 enabled 字段，ContactPerson 没有；Contact 的 email 是主邮箱，ContactPerson 的 email 是人员独立邮箱；通过 withPersons() 合并用于邮件发送 |
| Sales Customers 导入映射 | 只创建 Contact，type 强制为 'customer'，不创建 ContactPerson |
| InvalidEmailDetected 查找逻辑 | 只在 Contact 和 User 表中查找 enabled 记录，**不查找 ContactPerson** |
| 主联系人启用但person禁用场景 | 当前需手动软删除 ContactPerson；系统不会自动检测和禁用 ContactPerson 的无效邮箱 |
