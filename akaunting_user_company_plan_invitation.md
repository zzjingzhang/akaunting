# CreateUser & UpdateUser 深度分析

## 核心文件
- [CreateUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php)
- [UpdateUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php)
- [CreateInvitation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateInvitation.php)
- [NotifyUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/NotifyUser.php)
- [Users.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Users.php) (Controller)
- [Register.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Register.php)
- [User.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Auth/User.php) (FormRequest)
- [Plans.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php)
- [UserSeed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UserSeed.php)
- [database/seeds/User.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/User.php)
- [database/seeds/Dashboards.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/Dashboards.php)
- [CreateDashboard.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateDashboard.php)
- [CreateCompany.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateCompany.php)
- [admin.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php) (路由)

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
- 该随机密码不是用户最终密码，用户通过邀请链接注册时会在 `Register::registered()` 中用 `forceFill()` 设置真正的密码

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
- 如果请求传入的公司 ID 当前用户都没有权限（`$companies->isEmpty()`），则新用户不会被关联到任何公司

---

## 2. console/install 与普通请求在 companies attach/sync 上的差异

### CreateUser 中的差异

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| **console/install** | 直接 `attach($request->get('companies'))`，所有传入的公司 ID 都会被关联 | [CreateUser.php:52-53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L52-L53) |
| **普通请求** | 先通过 `user()->companies()->whereIn('id', $companies)` 过滤，只关联当前用户有权访问的公司 | [CreateUser.php:55-63](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L55-L63) |

### UpdateUser 中的差异

**代码位置**: [UpdateUser.php:47-60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L47-L60)

| 场景 | 处理方式 | 代码位置 |
|------|---------|---------|
| **console/install** | 直接 `sync($request->get('companies'))`，全量同步（附加新的、移除不存在的） | [UpdateUser.php:48-49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L48-L49) |
| **普通请求** | 先过滤出当前用户有权访问的公司，再 `sync()`；如果过滤结果为空，`sync()` 不执行，`$sync` 变量未设置 | [UpdateUser.php:50-58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L50-L58) |

### 普通请求过滤结果为空时的行为

**代码位置**: [UpdateUser.php:50-59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L50-L59)

```php
} else {
    $user = user();
    $companies = $user->withoutEvents(function () use ($user) {
        return $user->companies()->whereIn('id', $this->request->get('companies'))->pluck('id');
    });
    if ($companies->isNotEmpty()) {
        $sync = $this->model->companies()->sync($companies->toArray());
    }
}
```

- 如果 `$companies` 为空（当前用户对请求传入的公司都没有权限），不进入 `if` 块
- `$sync` 变量不会被设置
- 后续 [UpdateUser.php:67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L67) 的 `if (isset($sync) && !empty($sync['attached']))` 不成立
- **不会触发 `user:seed`**，用户的 companies 关联完全不会被修改

### 关键差异总结

| 维度 | console/install | 普通请求 |
|------|----------------|---------|
| **权限检查** | 跳过（信任命令行/安装流程） | 严格检查当前用户对公司的访问权限 |
| **操作方法** | attach (CreateUser) / sync (UpdateUser) | attach (CreateUser) / sync (UpdateUser) |
| **公司范围** | 可关联任意公司 ID | 只能关联当前用户已有权限的公司 |
| **使用场景** | 安装初始化、CLI 脚本 | Web UI、API 调用 |

> **设计意图**：console/install 环境通常是管理员或系统初始化场景，信任度高，因此跳过权限检查；普通请求需遵循最小权限原则，防止越权。

---

## 3. user:seed 调用时机与实际作用

`user:seed` 命令定义在 [UserSeed.php:14](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UserSeed.php#L14)，签名为 `user:seed {user} {company}`。

### 调用位置汇总

| 调用位置 | 触发时机 | 代码位置 |
|---------|---------|---------|
| **CreateUser** | 创建用户后，遍历所有关联公司调用 | [CreateUser.php:71-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L71-L76) |
| **UpdateUser** | 更新用户时，仅对 `sync` 返回的 `attached` 中新附加的公司调用；若 `sync` 未执行则不调用 | [UpdateUser.php:67-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L67-L76) |
| **CreateCompany** | 创建公司后，对当前登录用户调用；若 `user()` 为空则跳过 | [CreateCompany.php:77-89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateCompany.php#L77-L89) |

### 实际作用：创建仪表盘

调用链为：
1. `UserSeed::handle()` → 实例化 `Database\Seeds\User` 并调用 `__invoke()` [UserSeed.php:28-33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/UserSeed.php#L28-L33)
2. `Database\Seeds\User::run()` → 调用 `Dashboards::class` [database/seeds/User.php:14-17](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/User.php#L14-L17)
3. `Database\Seeds\Dashboards::create()` → 读取命令行参数 `user` 和 `company`，dispatch `CreateDashboard` [database/seeds/Dashboards.php:28-48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/Dashboards.php#L28-L48)

**实际播种的内容**：
- 创建一个 Dashboard 记录，关联指定 user 和 company
- 创建 7 个默认 Widgets：Receivables、Payables、CashFlow、ProfitLoss、ExpensesByCategory、AccountBalance、BankFeeds

### 跳过条件

#### 1. CreateCompany 中 user() 为空时跳过

**代码位置**: [CreateCompany.php:77-80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateCompany.php#L77-L80)

```php
if (!$user = user()) {
    report(new \UnexpectedValueException('CreateCompany: user() returned null — user:seed and company attachment skipped for company ID ' . $this->model->id));
    return;
}
```

- 当 `user()` 为空时，`company:seed` 仍然执行，但 `user:seed` 和 `companies()->attach()` 被跳过
- 这种情况可能发生在 console 命令行创建公司时没有已登录用户

#### 2. CreateDashboard 中用户无 read-admin-panel 权限时跳过

**代码位置**: [CreateDashboard.php:78-90](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateDashboard.php#L78-L90)

```php
protected function shouldCreateDashboardFor($user): bool
{
    if (empty($user)) {
        return false;
    }
    if ($user->cannot('read-admin-panel')) {
        return false;
    }
    return true;
}
```

- 如果用户没有 `read-admin-panel` 权限（如客户用户），不会为其创建仪表盘
- `getUsers()` 中会对每个用户调用 `shouldCreateDashboardFor()`，不符合条件的用户 ID 不会被加入列表

---

## 4. 邀请发送路径与环境条件

### 两条邀请路径

#### 路径 A：CreateUser 自动邀请

**代码位置**: [CreateUser.php:78-80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L78-L80)

```php
if ($this->shouldSendInvitation()) {
    $this->dispatch(new CreateInvitation($this->model));
}
```

此路径经过 `shouldSendInvitation()` 环境判断。

#### 路径 B：后台 users.invite 手动重邀

**路由定义**: [admin.php:66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php#L66)
```php
Route::get('users/{user}/invite', 'Auth\Users@invite')->name('users.invite');
```

**控制器方法**: [Users.php:399-416](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Users.php#L399-L416)

```php
public function invite($user_id)
{
    $user = user_model_class()::find($user_id);
    $response = $this->ajaxDispatch(new CreateInvitation($user, company()));
    // ...
}
```

此路径**直接调用 `CreateInvitation`**，不经过 `shouldSendInvitation()` 环境判断，只要能访问该路由即可发送邀请。

### shouldSendInvitation() 环境判断

**代码位置**: [CreateUser.php:101-116](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L101-L116)

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

| 环境 | 返回值 | 说明 |
|------|--------|------|
| **单元测试** | `true` | 测试环境始终发送，便于测试邀请流程 |
| **Console 命令行** | `false` | CLI 脚本、Artisan 命令不发送邮件 |
| **安装流程** (`install/*`) | `false` | 初始化管理员账号不发送邀请 |
| **普通 Web/API 请求** | `true` | 正常业务场景发送邀请邮件 |

> 注：`isInstall()` 判断当前请求路径是否匹配 `install/*`，定义在 [Macro.php:46-48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Macro.php#L46-L48)

### 邀请创建与发送流程

**CreateInvitation::handle()**: [CreateInvitation.php:26-54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateInvitation.php#L26-L54)

1. 删除该用户已存在的所有邀请记录（确保只有一个有效邀请）
2. 创建新的 `UserInvitation` 记录，生成 UUID token，记录 `created_by` 和 `created_from`
3. dispatch `NotifyUser` Job 发送邀请邮件

**NotifyUser::handle()**: [NotifyUser.php:32-35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/NotifyUser.php#L32-L35)

```php
public function handle()
{
    $this->user->notify($this->notification);
}
```

- `NotifyUser` 继承自 `JobShouldQueue`，在 `jobs` 队列上异步执行 [NotifyUser.php:7](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/NotifyUser.php#L7)

### 邀请后的密码设置流程

用户收到邀请邮件后，点击邮件中的链接，流程如下：

1. **Register::create($token)** — 展示注册表单 [Register.php:34-43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Register.php#L34-L43)
2. **Register::store()** — 处理表单提交 [Register.php:45-62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Register.php#L45-L62)
   - 根据 token 查找 `UserInvitation` 记录
   - dispatch `DeleteInvitation` 删除邀请记录
   - 触发 `Registered` 事件
3. **Register::registered()** — 设置密码并登录 [Register.php:71-87](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Register.php#L71-L87)
```php
$user->forceFill([
    'password' => $request->password,
    'remember_token' => Str::random(60),
])->save();
$this->guard()->login($user);
```

- 这是用户**第一次设置密码**，不是重置密码
- 邀请记录在设置密码之前已被 `DeleteInvitation` 删除

---

## 5. UpdateUser 的权限限制

UpdateUser 的 `authorize()` 方法定义了两项限制 [UpdateUser.php:87-121](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L87-L121)

### 限制 1：不能禁用自己

**代码位置**: [UpdateUser.php:89-94](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L89-L94)

```php
if (($this->request->get('enabled', 1) == 0) && ($this->model->id == user()->id)) {
    $message = trans('auth.error.self_disable');
    throw new \Exception($message);
}
```

**源码检查逻辑**：
- 条件仅为 `enabled == 0 && 操作对象是自己`
- 不检查是否有其他管理员，不检查系统是否还有其他启用用户
- 仅禁止当前用户把自己的 enabled 设为 0

**实际效果**：防止用户将自己锁定。

### 限制 2：不能移除公司最后一个用户

**代码位置**: [UpdateUser.php:96-120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L96-L120)

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

**源码检查逻辑**：
- 计算当前用户的公司与请求中公司的差集（即要移除的公司列表）
- 对每个要移除的公司，查询该公司的用户总数 `users_count`
- 如果 `users_count < 2`（即当前用户是该公司唯一的用户），报错
- 不检查用户角色，不判断是否有管理员

**实际效果**：防止移除后公司没有任何用户（0 个用户）。不涉及"管理员"概念的检查。

---

## 6. UpdateUser 与计划限制的关系

### UpdateUser 不使用 Plans trait

**代码位置**: [UpdateUser.php:1-10](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L1-L10)

```php
use App\Abstracts\Job;
use App\Events\Auth\UserUpdated;
use App\Events\Auth\UserUpdating;
use App\Interfaces\Job\ShouldUpdate;
use App\Models\Common\Company;
use Illuminate\Support\Facades\Artisan;

class UpdateUser extends Job implements ShouldUpdate
```

- 没有 `use Plans;`，没有导入 Plans trait
- 不调用 `getAnyActionLimitOfPlan()` 或任何计划限制检查方法

### UpdateUser 的 authorize() 不检查计划限制

**代码位置**: [UpdateUser.php:87-121](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L87-L121)

- 只检查"不能禁用自己"和"不能移除公司最后一个用户"
- 没有任何计划限制（用户数量、公司数量、发票数量）的检查

### UpdateUser 不清理计划缓存

**代码位置**: [UpdateUser.php:78-82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L78-L82)

```php
event(new UserUpdated($this->model, $this->request));
return $this->model;
```

- handle() 结束时没有调用 `$this->clearPlansCache()`
- 而 CreateUser 在 [CreateUser.php:83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L83) 明确调用了 `$this->clearPlansCache()`

### 对比总结

| 维度 | CreateUser | UpdateUser |
|------|-----------|-----------|
| **使用 Plans trait** | ✅ `use Plans;` [L17](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L17) | ❌ 不使用 |
| **检查计划限制** | ✅ `getAnyActionLimitOfPlan()` [L95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L95) | ❌ 不检查 |
| **清理计划缓存** | ✅ `clearPlansCache()` [L83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L83) | ❌ 不清理 |

---

## 7. UpdateUser 对角色权限字段的处理

### roles（角色）

**代码位置**: [UpdateUser.php:43-44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L43-L44)

```php
if ($this->request->has('roles')) {
    $this->model->roles()->sync($this->request->get('roles'));
}
```

- 请求包含 `roles` 时，使用 `sync()` 全量同步角色
- 与 CreateUser 的 `attach()` 不同：UpdateUser 会移除不在请求中的角色，CreateUser 只追加不移除

### permissions（权限）

**源码中 UpdateUser 没有任何 permissions 相关代码**
- 查看 [UpdateUser.php:14-82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L14-L82)，没有对 `permissions` 的任何处理
- 如果要修改用户权限，需要通过其他途径（如角色继承或直接操作中间表）

### dashboards（仪表盘）

**源码中 UpdateUser 没有任何 dashboards 相关代码**
- 查看 [UpdateUser.php:14-82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L14-L82)，没有对 `dashboards` 的任何处理
- 用户的仪表盘关联在创建时通过 `attach()` 设置，或由 `user:seed` 自动创建，UpdateUser 不管理

### 总结

| 字段 | CreateUser | UpdateUser |
|------|-----------|-----------|
| **roles** | `attach()`（追加） | `sync()`（全量同步） |
| **permissions** | `attach()`（追加） | ❌ 不处理 |
| **dashboards** | `attach()`（追加） | ❌ 不处理 |
| **companies** | `attach()`（追加，console/install 直接，普通请求过滤） | `sync()`（全量同步，console/install 直接，普通请求过滤） |

---

## 8. 表单验证规则

`App\Http\Requests\Auth\User` 定义了创建和更新用户的验证规则 [User.php:25-82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Auth/User.php#L25-L82)

### password / current_password 的条件拼接

**代码位置**: [User.php:67-71](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Auth/User.php#L67-L71)

```php
$change_password = $this->request->get('change_password') == true || $this->request->get('change_password') != null;

$current_password = $change_password ? '|current_password' : '';
$password = $change_password ? '|confirmed' : '';
```

```php
'current_password'  => 'required_if:change_password,true' . $current_password,
'password'          => 'required_if:change_password,true' . $password,
```

**规则说明**：
- 基础规则始终为 `required_if:change_password,true`
- 仅当请求中 `change_password` 为 true 或非 null 时，才追加 `current_password` 校验和 `confirmed` 校验
- 因此：创建用户时若请求不带 `change_password`，这两个字段均不做必填/确认校验，空 password 能够通过验证并进入 `CreateUser` 生成 40 位随机密码的逻辑

### companies / roles 必填规则的主体

**代码位置**: [User.php:48-49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Auth/User.php#L48-L49)

```php
$companies = $this->user->can('read-common-companies') ? 'required' : '';
$roles = $this->user->can('read-auth-roles') ? 'required|string' : '';
```

**主体说明**：
- 更新分支使用 `$this->user`（即被更新的用户模型）的 `can()` 方法判断是否必填
- 这与当前操作者的权限无关，是由被更新用户自身的权限决定的
- 不要与 `Users` 控制器中间件中的 `permission:update-auth-users` 等操作者权限混为一谈（[Users.php:24-28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Users.php#L24-L28)）

### 完整规则表

| 字段 | 创建时 | 更新时 |
|------|--------|--------|
| **name** | `required\|string` | `required\|string` |
| **email** | `required\|email\|unique:users` | `required\|email\|unique:users`（排除自身） |
| **companies** | `required` | 被更新用户 `can('read-common-companies')` 时 `required`，否则为空 |
| **roles** | `required\|string` | 被更新用户 `can('read-auth-roles')` 时 `required\|string`，否则为空 |
| **picture** | `nullable\|mimes\|between\|dimensions` | `nullable\|mimes\|between\|dimensions` |
| **landing_page** | `required\|string` | `required\|string` |
| **password** | `required_if:change_password,true`；带 `change_password` 时追加 `\|confirmed` | `required_if:change_password,true`；带 `change_password` 时追加 `\|confirmed` |
| **current_password** | `required_if:change_password,true`；带 `change_password` 时追加 `\|current_password` | `required_if:change_password,true`；带 `change_password` 时追加 `\|current_password` |

---

## 总结：CreateUser 完整流程图

```
请求到达
  ↓
authorize() → 检查计划限制（用户/公司/发票数量）[Plans trait]
  ↓
UserCreating 事件
  ↓
DB 事务开始
  ├─ password 为空 → 生成 40 位随机字符串
  ├─ 创建 User 模型
  ├─ 有 picture 上传 → 处理并关联
  ├─ 有 dashboards → attach
  ├─ 有 permissions → attach
  ├─ 有 roles → attach
  ├─ 有 companies →
  │   ├─ console/install → 直接 attach
  │   └─ 普通请求 → 过滤当前用户有权限的公司后 attach
  │       └─ 过滤为空 → 不关联任何公司
  ├─ 有关联公司 → 对每个公司调用 user:seed（创建仪表盘+widgets）
  │   └─ 用户无 read-admin-panel 权限 → CreateDashboard 跳过
  └─ shouldSendInvitation() →
      ├─ 单元测试/普通请求 → dispatch CreateInvitation → NotifyUser（异步邮件）
      └─ console/install → 不发送
  ↓
DB 事务结束
  ↓
clearPlansCache()
  ↓
UserCreated 事件
  ↓
返回 User 模型
```

## 总结：UpdateUser 完整流程图

```
请求到达
  ↓
authorize() →
  ├─ 检查：不能禁用自己（enabled=0 且操作对象是自己）
  └─ 检查：不能移除公司最后一个用户（users_count < 2）
  ↓
password 为空 → unset password/current_password/password_confirmation
  ↓
UserUpdating 事件
  ↓
DB 事务开始
  ├─ $model->update($request->input())
  ├─ 处理 picture（上传/删除）
  ├─ 有 roles → sync（全量同步）
  ├─ 有 companies →
  │   ├─ console/install → 直接 sync → $sync 被设置
  │   └─ 普通请求 → 过滤当前用户有权限的公司
  │       ├─ 过滤不为空 → sync → $sync 被设置
  │       └─ 过滤为空 → 不执行 sync → $sync 未设置
  ├─ 有 contact → 更新 contact 信息
  └─ $sync 被设置 且 attached 不为空 → 对新附加公司调用 user:seed
  ↓
UserUpdated 事件
  ↓
返回 User 模型
```
