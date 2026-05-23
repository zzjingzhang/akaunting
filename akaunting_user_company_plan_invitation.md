# CreateUser & UpdateUser 深度分析

## 核心文件
- [CreateUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php)
- [UpdateUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php)
- [CreateInvitation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateInvitation.php)
- [Plans.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php)
- [UserSeed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UserSeed.php)
- [CreateCompany.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateCompany.php)

---

## 1. 创建用户时各字段处理

### password（密码）
**代码位置**: [CreateUser.php:26-28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L26-L28)
```php
if (empty($this->request->get('password', false))) {
    $this->request->merge(['password' => Str::random(40)]);
}
```
**处理逻辑**:
- 如果请求中 password 为空（或不存在），自动生成 40 位随机字符串作为密码
- 该随机密码主要用于邀请场景，用户会通过邀请邮件重置密码

### picture（头像）
**代码位置**: [CreateUser.php:32-37](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L32-L37)
```php
if ($this->request->file('picture')) {
    $media = $this->getMedia($this->request->file('picture'), 'users');
    $this->model->attachMedia($media, 'picture');
}
```
**处理逻辑**:
- 仅当请求中有上传的 picture 文件时才处理
- 通过 `getMedia()` 处理上传并关联到用户模型
- 没有上传文件时不做任何处理（保持默认头像）

### dashboards（仪表盘）
**代码位置**: [CreateUser.php:39-41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L39-L41)
```php
if ($this->request->has('dashboards')) {
    $this->model->dashboards()->attach($this->request->get('dashboards'));
}
```
**处理逻辑**:
- 请求中包含 dashboards 参数时，通过 `attach()` 关联到用户
- 使用 `attach()` 而非 `sync()`，不会删除已存在的关联

### permissions（权限）
**代码位置**: [CreateUser.php:43-45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L43-L45)
```php
if ($this->request->has('permissions')) {
    $this->model->permissions()->attach($this->request->get('permissions'));
}
```
**处理逻辑**:
- 请求中包含 permissions 参数时，通过 `attach()` 关联到用户
- 直接附加，不移除已有权限

### roles（角色）
**代码位置**: [CreateUser.php:47-49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L47-L49)
```php
if ($this->request->has('roles')) {
    $this->model->roles()->attach($this->request->get('roles'));
}
```
**处理逻辑**:
- 请求中包含 roles 参数时，通过 `attach()` 关联到用户
- 直接附加，不移除已有角色

### companies（公司）
**代码位置**: [CreateUser.php:51-65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L51-L65)
```php
if ($this->request->has('companies')) {
    if (app()->runningInConsole() || request()->isInstall()) {
        $this->model->companies()->attach($this->request->get('companies'));
    } else {
        $user = user();
        $companies = $user->withoutEvents(function () use ($user) {
            return $user->companies()->whereIn('id', $this->request->get('companies'))->pluck('id');
        });
        if ($companies->isNotEmpty()) {
            $this->model->companies()->attach($companies->toArray());
        }
    }
}
```
**处理逻辑**:
- console/install 环境：直接 `attach()` 所有传入的公司 ID
- 普通请求：先过滤出当前登录用户有权访问的公司，再 `attach()`
- 安全机制：普通用户只能将自己有权限的公司分配给新用户

---

## 2. console/install 与普通请求在 companies attach/sync 上的差异

### CreateUser 中的差异

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| **console/install** | 直接 `attach($request->get('companies'))`，所有传入的公司 ID 都会被关联 | [CreateUser.php:52-53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L52-L53) |
| **普通请求** | 先通过 `user()->companies()->whereIn('id', $companies)` 过滤，只关联当前用户有权访问的公司 | [CreateUser.php:55-63](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L55-L63) |

### UpdateUser 中的差异

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| **console/install** | 直接 `sync($request->get('companies'))`，全量同步（附加新的、移除不存在的） | [UpdateUser.php:48-49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L48-L49) |
| **普通请求** | 先过滤出当前用户有权访问的公司，再 `sync()` | [UpdateUser.php:50-58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L50-L58) |

### 关键差异总结

| 维度 | console/install | 普通请求 |
|------|----------------|---------|
| **权限检查** | 跳过（信任命令行/安装流程） | 严格检查当前用户对公司的访问权限 |
| **操作方法** | attach (CreateUser) / sync (UpdateUser) | attach (CreateUser) / sync (UpdateUser) |
| **公司范围** | 可关联任意公司 ID | 只能关联当前用户已有权限的公司 |
| **使用场景** | 安装初始化、CLI 脚本 | Web UI、API 调用 |

> **设计意图**：console/install 环境通常是管理员或系统初始化场景，信任度高，因此跳过权限检查；普通请求需遵循最小权限原则，防止越权。

---

## 3. user:seed 调用时机

`user:seed` 命令定义在 [UserSeed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UserSeed.php)，用于为指定用户在指定公司中播种初始数据。

### 调用位置汇总

| 调用位置 | 触发时机 | 代码 |
|---------|---------|------|
| **CreateUser** | 创建用户后，遍历所有关联公司调用 | [CreateUser.php:71-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L71-L76) |
| **UpdateUser** | 更新用户时，仅对**新附加**的公司调用（sync 返回的 attached 数组） | [UpdateUser.php:67-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L67-L76) |
| **CreateCompany** | 创建公司后，对当前登录用户调用 | [CreateCompany.php:86-89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateCompany.php#L86-L89) |

### 调用逻辑详解

**CreateUser 中**:
```php
foreach ($this->model->companies as $company) {
    Artisan::call('user:seed', [
        'user' => $this->model->id,
        'company' => $company->id,
    ]);
}
```
- 创建用户后，如果用户有关联公司，对每个公司都执行一次 seed

**UpdateUser 中**:
```php
if (isset($sync) && !empty($sync['attached'])) {
    foreach ($sync['attached'] as $id) {
        $company = Company::find($id);
        Artisan::call('user:seed', [
            'user' => $this->model->id,
            'company' => $company->id,
        ]);
    }
}
```
- 仅针对 `sync()` 返回的 `attached` 数组中的新公司
- 已存在的关联公司不会重复 seed

**CreateCompany 中**:
```php
Artisan::call('user:seed', [
    'user' => $user->id,
    'company' => $this->model->id,
]);
```
- 创建新公司后，为当前登录用户在新公司中 seed

---

## 4. 邀请发送的环境条件

邀请逻辑在 `shouldSendInvitation()` 方法中定义 [CreateUser.php:101-116](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L101-L116)

```php
protected function shouldSendInvitation()
{
    if (app()->runningUnitTests()) {
        return true;
    }

    if (app()->runningInConsole()) {
        return false;
    }

    if (request()->isInstall()) {
        return false;
    }

    return true;
}
```

### 发送邀请（返回 true）
| 环境 | 条件 | 说明 |
|-----|------|------|
| **单元测试** | `runningUnitTests()` | 测试环境始终发送，便于测试邀请流程 |
| **普通 Web/API 请求** | 非 console、非 install、非测试 | 正常业务场景发送邀请邮件 |

### 不发送邀请（返回 false）
| 环境 | 条件 | 说明 |
|-----|------|------|
| **Console 命令行** | `runningInConsole()` | CLI 脚本、Artisan 命令不发送邮件 |
| **安装流程** | `request()->isInstall()` | 安装路径（`install/*`）不发送，因为是初始化管理员账号 |

### 邀请创建流程
邀请创建在 [CreateInvitation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateInvitation.php) 中：
1. 删除该用户已存在的所有邀请（确保只有一个有效邀请）
2. 创建新的 `UserInvitation` 记录，生成 UUID token
3. 通过 `NotifyUser` Job 发送邀请邮件

---

## 5. 更新时的权限限制

UpdateUser 的 `authorize()` 方法定义了两项关键限制 [UpdateUser.php:87-121](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L87-L121)

### 限制 1：不能禁用自己
```php
if (($this->request->get('enabled', 1) == 0) && ($this->model->id == user()->id)) {
    $message = trans('auth.error.self_disable');
    throw new \Exception($message);
}
```
**原因**:
- 防止管理员误操作将自己禁用，导致无法登录系统
- 保持系统至少有一个可用的管理员账号

### 限制 2：不能移除某公司最后一个用户
```php
if ($this->request->has('companies')) {
    $companies = (array) $this->request->get('companies', []);
    $user_companies = $this->model->companies()->pluck('id')->toArray();
    $company_diff = array_diff($user_companies, $companies);

    if ($company_diff) {
        foreach ($company_diff as $company_id) {
            $company = Company::withCount('users')->find($company_id);
            if ($company->users_count < 2) {
                $errors[] = trans('auth.error.unassigned', ['company' => $company->name]);
            }
        }
        if ($errors) {
            throw new \Exception(implode('\n', $errors));
        }
    }
}
```
**原因**:
- 计算当前用户拥有的公司与请求中公司的差集（即要移除的公司）
- 检查要移除的公司用户数是否少于 2
- 如果公司只有 1 个用户，移除后该公司将没有任何用户可访问，造成"孤儿公司"
- 保证每个公司至少有一个管理员用户

---

## 计划限制检查

CreateUser 在创建前会检查计划限制 [CreateUser.php:93-99](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L93-L99)

```php
public function authorize(): void
{
    $limit = $this->getAnyActionLimitOfPlan();
    if (! $limit->action_status) {
        throw new \Exception($limit->message);
    }
}
```

`getAnyActionLimitOfPlan()` 在 [Plans.php:33-57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L33-L57) 中依次检查：
1. 用户数量限制 (`getUserLimitOfPlan`)
2. 公司数量限制 (`getCompanyLimitOfPlan`)
3. 发票数量限制 (`getInvoiceLimitOfPlan`)

任何一项限制触发都会阻止用户创建。

> **例外**：未安装或测试环境下跳过计划限制检查 [Plans.php:61-69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L61-L69)

---

## 总结：CreateUser 完整流程图

```
请求到达
  ↓
authorize() → 检查计划限制（用户/公司/发票数量）
  ↓
UserCreating 事件
  ↓
DB 事务开始
  ├─ password 为空 → 生成 40 位随机密码
  ├─ 创建 User 模型
  ├─ 有 picture 上传 → 处理并关联
  ├─ 有 dashboards → attach
  ├─ 有 permissions → attach
  ├─ 有 roles → attach
  ├─ 有 companies → 
  │   ├─ console/install → 直接 attach
  │   └─ 普通请求 → 过滤当前用户有权限的公司后 attach
  ├─ 有关联公司 → 对每个公司调用 user:seed
  └─ shouldSendInvitation() → 发送邀请（非 console、非 install）
  ↓
DB 事务结束
  ↓
清除计划缓存
  ↓
UserCreated 事件
  ↓
返回 User 模型
```
