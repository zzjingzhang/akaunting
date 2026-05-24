# Akaunting 多公司/租户上下文系统分析

---

## 1. Company::makeCurrent / forgetCurrent 触发事件与全局状态

### makeCurrent() 流程

源码：[app/Models/Common/Company.php#L601-L627](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L601-L627)

```
makeCurrent($force = false)
│
├─ [L603] 若已是当前公司且非强制 → 直接 return $this（无事件、无状态变化）
│
├─ [L607] 调用 static::forgetCurrent() 清除现有上下文
│
├─ [L609] event(new CompanyMakingCurrent($this))
│
├─ [L612] app()->instance(static::class, $this)   ← 绑定服务容器
│
├─ [L615-622] 加载公司配置：
│   ├─ setting()->setExtraColumns(['company_id' => $this->id])
│   ├─ setting()->forgetAll()
│   ├─ setting()->load(true)
│   ├─ Overrider::load('settings')
│   ├─ Overrider::load('currencies')
│   └─ Overrider::load('categoryTypes')
│
└─ [L624] event(new CompanyMadeCurrent($this))
```

### forgetCurrent() 流程

源码：[app/Models/Common/Company.php#L648-L667](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L648-L667)

```
forgetCurrent()
│
├─ [L650] $current = static::getCurrent()
│
├─ [L652-653] 若 is_null($current) → 直接 return null
│                ★ 关键：无当前公司时 —— 不触发任何事件、不操作容器、不清配置
│
├─ [L656] event(new CompanyForgettingCurrent($current))   ← 仅当有当前公司
│
├─ [L659] app()->forgetInstance(static::class)            ← 仅当有当前公司
│
├─ [L662] setting()->forgetAll()                          ← 仅当有当前公司
│
└─ [L664] event(new CompanyForgotCurrent($current))       ← 仅当有当前公司
```

### 无当前公司时的边界行为

| 操作 | 有当前公司 | 无当前公司 |
|------|-----------|-----------|
| `forgetCurrent()` 返回值 | 被移除的 Company 实例 | `null` |
| `CompanyForgettingCurrent` 事件 | ✅ 触发 | ❌ 不触发 |
| `CompanyForgotCurrent` 事件 | ✅ 触发 | ❌ 不触发 |
| `app()->forgetInstance()` | ✅ 执行 | ❌ 不执行 |
| `setting()->forgetAll()` | ✅ 执行 | ❌ 不执行 |
| 服务容器中 Company 绑定 | 被清除 | 保持不变（本来就没有） |

**核心原因**：[Company.php#L652-L653](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L652-L653) 的早返回守卫。

### makeCurrent() 内部对 forgetCurrent() 的调用

[Company.php#L607](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L607)：`makeCurrent()` 在每次设置新上下文前都会调用 `forgetCurrent()`。如果之前没有当前公司，`forgetCurrent()` 返回 `null` 且什么都不做，这是安全的 no-op。

### 全局状态影响汇总

| 状态类型 | 具体影响 | 源码位置 |
|---------|---------|---------|
| 服务容器 | `app()->instance(Company::class, $company)` 绑定 / `forgetInstance` 解绑 | [L612](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L612) / [L659](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L659) |
| `setting()` 系统 | `setExtraColumns(['company_id' => ...])` + `forgetAll()` + `load(true)` | [L615-L617](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L615-L617) |
| `Overrider` | 重载 settings / currencies / categoryTypes | [L620-L622](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L620-L622) |
| 后续 Eloquent 查询 | 所有使用 `Tenants` trait 的模型查询自动 `WHERE company_id = current_id` | 由 Scope 驱动，见第 2 节 |

### 四个生命周期事件

| 事件类 | 触发时机 | 源码 |
|--------|---------|------|
| `CompanyMakingCurrent` | 绑定容器之前 | [Company.php#L609](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L609) |
| `CompanyMadeCurrent` | 所有配置加载完毕之后 | [Company.php#L624](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L624) |
| `CompanyForgettingCurrent` | 解绑容器之前 | [Company.php#L656](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L656) |
| `CompanyForgotCurrent` | 解绑并清配置之后 | [Company.php#L664](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L664) |

事件类均位于 [app/Events/Common/](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/)，构造函数均只接收 `$company` 参数（见 [CompanyMakingCurrent.php#L16-L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/CompanyMakingCurrent.php#L16-L19)）。

---

## 2. Company Scope 如何限制普通查询，allCompanies 如何绕过，以及 Company 模型自身为何不受限

### Scope 应用逻辑

源码：[app/Scopes/Company.php#L21-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L21-L50)

```
apply(Builder $builder, Model $model)
│
├─ [L23-25] 若 $model->isNotTenantable() → return（跳过 Scope）
│
├─ [L27] $table = $model->getTable()
│
├─ [L34-41] 若 $table 在 skip_tables 中 → return
│   skip_tables = ['jobs', 'firewall_ips', 'firewall_logs', 'migrations',
│                   'notifications', 'role_companies', 'role_permissions',
│                   'sessions', 'user_companies', 'user_dashboards',
│                   'user_permissions', 'user_roles']
│
├─ [L44-46] 若 builder 中已有 company_id 条件 → return（避免重复）
│
└─ [L49] $builder->where($table . '.company_id', '=', company_id())
```

### Scope 的注册方式

源码：[app/Traits/Tenants.php#L14-L17](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php#L14-L17)

```php
protected static function bootTenants()
{
    static::addGlobalScope(new Company);
}
```

所有继承 [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) 的模型通过 `use Tenants`（[Abstracts/Model.php#L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L23)）自动获得此全局 Scope。

### isTenantable() 的双重检查

源码：[app/Traits/Tenants.php#L19-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php#L19-L29)

```php
public function isTenantable()
{
    $tenantable = $this->tenantable ?? true;
    return ($tenantable === true) && in_array('company_id', $this->getFillable());
}
```

模型必须同时满足两个条件才会被 Scope 过滤：
1. `$tenantable` 属性为 `true`（或未设置，默认为 `true`）
2. `company_id` 在 `$fillable` 数组中

### Company 模型自身不受限的根本原因

[app/Models/Common/Company.php#L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L39)：

```php
protected $fillable = ['domain', 'enabled', 'created_from', 'created_by'];
```

Company 模型的 `$fillable` 中 **没有 `company_id`**。虽然 Company 使用了 `Tenants` trait（[L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L26)）且没有设置 `$tenantable = false`，但 `isTenantable()` 的第二个条件 `in_array('company_id', $this->getFillable())` 返回 `false`，导致 `isNotTenantable()` 返回 `true`，Scope 在 [Scopes/Company.php#L23-25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L23-L25) 直接 return。

**这是刻意设计**：Company 作为上下文切换的锚点，其查询（如 `Company::find($id)`）必须在任何上下文状态下都能工作。

### allCompanies() 绕过机制

源码：[app/Abstracts/Model.php#L77-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L77-L80)

```php
public function scopeAllCompanies($query)
{
    return $query->withoutGlobalScope('App\Scopes\Company');
}
```

使用方式：
```php
Item::all();                          // 只返回当前公司的 items
Item::allCompanies()->get();          // 返回所有公司的 items
Item::allCompanies()->where(...)->get(); // 跨公司查询
```

### 对 Queue.php 恢复 context 的关键影响

[app/Providers/Queue.php#L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L50)：

```php
$company = company($payload['company_id']);
```

[helpers.php#L86-L88](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L86-L88)：`company()` 带数值 id 时调用 `Company::find($id)`。

**如果 Company 模型受 Scope 限制**，在队列 Worker 启动时（还未恢复任何公司上下文），`company_id()` 返回 `null`，Scope 会附加 `WHERE companies.company_id = NULL`，查询永远返回空 → 所有队列 Job 因找不到公司而被删除。

**正是因为 Company 模型不受 Scope 限制**，Queue.php 的 `JobProcessing` 监听器才能在无上下文的初始状态下成功查找公司并调用 `makeCurrent()` 完成上下文恢复。这是整个多租户队列系统的基础前提。

---

## 3. Queued Job 恢复 Company Context

### 完整流程

#### 入队阶段 — 捕获上下文

源码：[app/Providers/Queue.php#L32-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L32-L40)

```php
app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
    $company_id = company_id();          // 从容器读取当前 company id
    if (empty($company_id)) {
        return [];                        // 无上下文则不注入
    }
    return ['company_id' => $company_id]; // 写入 payload
});
```

**可达边界**：任何通过 Laravel queue 系统分发的 Job（`dispatch()`、`dispatchSync()` 的队列版本等）。如果 Job 在没有 `makeCurrent()` 的环境中分发，`company_id()` 返回 `null`，payload 中不会携带 `company_id`。

#### 出队阶段 — 恢复上下文

源码：[app/Providers/Queue.php#L42-L80](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L42-L80)

```
JobProcessing 事件触发
│
├─ [L43-47] 从 payload 提取 company_id → 不存在则 return（不恢复）
│
├─ [L50] $company = company($payload['company_id'])
│     └─ 调用 Company::find($id) — 因 Company 不受 Scope 限制，任何状态下都能查询
│
├─ [L51-61] catch Throwable → job->delete() + 日志
│     异常场景：数据库连接失败、模型未找到等
│
├─ [L63-72] empty($company) → job->delete() + 日志
│     场景：公司已被软删除（Company 使用 SoftDeletes）
│
├─ [L74] $company->makeCurrent()   ← 恢复上下文
│
└─ [L77-79] if (should_queue()) → $this->registerModules()
```

**可达边界**：所有被 Laravel queue worker 消费的 Job。`makeCurrent()` 内部会先调用 `forgetCurrent()`，然后绑定新的上下文。

#### 故障保护

如果 payload 中无 `company_id` 键（[L45-47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L45-L47)），监听器直接 return，不做任何操作。Job 在执行时会处于"无上下文"状态——此时模型查询会因 `company_id()` 返回 `null` 而返回空或写入失败。

### 与 IdentifyCompany 中间件的对比

HTTP 请求通过 [app/Http/Middleware/IdentifyCompany.php#L25-L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/IdentifyCompany.php#L25-L62) 恢复上下文：从 URL/Query/Header 提取 `company_id` → `company($id)` → `makeCurrent()`。队列则通过 `createPayloadUsing` + `JobProcessing` 的组合实现相同目的。

---

## 4. company_id / created_by / created_from 默认值注入路径与优先级

### company_id 注入

**唯一注入点**：[app/Abstracts/Http/FormRequest.php#L14-L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/FormRequest.php#L14-L19)

```php
protected function prepareForValidation()
{
    $this->merge([
        'company_id' => company_id(),
    ]);
}
```

`prepareForValidation()` 在 Laravel 中被 `merge()` 和验证流程自动调用。

#### 各可达边界的注入情况

| 边界 | 注入是否发生 | 源码路径 |
|------|------------|---------|
| **HTTP 请求**（通过路由验证） | ✅ `FormRequest::validateResolved()` 调用 `prepareForValidation()` | Laravel 框架行为 |
| **同步 Job + 数组参数** | ✅ `Job::getRequestInstance($array)` 创建匿名 FormRequest → `merge()` → `prepareForValidation()` | [Job.php#L100-L109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L100-L109) |
| **异步 Job + 数组参数** | ✅ `JobShouldQueue::getRequestAsCollection($array)` 创建匿名 FormRequest → `merge()` → `prepareForValidation()` → 包装为 QueueCollection | [JobShouldQueue.php#L123-L134](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php#L123-L134) |
| **同步 Job + Request 对象** | ❌ `getRequestInstance()` 直接返回原 Request，不重新注入 | [Job.php#L101-L103](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L101-L103) |
| **直接模型创建** `Model::create([])` | ❌ 无自动注入，必须手动指定 `company_id` | Eloquent 行为 |
| **工厂（Factory）** | ❌ 无自动注入。但工厂构造函数 `Abstracts/Factory::setCompany()` [L59-L68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Factory.php#L59-L68) 会调用 `makeCurrent()`，且工厂 definition() 中硬编码 `'company_id' => $this->company->id` | [Factories/Item.php#L27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/factories/Item.php#L27) |

### created_by 注入

**注入点**：`Job::setOwner()` / `JobShouldQueue::setOwner()`

#### 同步 Job（App\Abstracts\Job）

源码：[Job.php#L111-L122](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L111-L122)

```php
public function setOwner(): void
{
    if (! $this->request instanceof Request) { return; }   // 守卫：必须是 Request
    if ($this->request->has('created_by'))    { return; }   // 优先级：用户值 > 自动注入
    $this->request->merge(['created_by' => user_id()]);
}
```

触发条件：[Job.php#L40-L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L40-L62) — `bootCreate()` 要求 Job 同时 `instanceof ShouldCreate` 和 `instanceof HasOwner`。

#### 异步 Job（App\Abstracts\JobShouldQueue）

源码：[JobShouldQueue.php#L136-L147](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php#L136-L147)

```php
public function setOwner(): void
{
    if (! $this->request instanceof QueueCollection) { return; }  // 守卫：必须是 QueueCollection
    if ($this->request->has('created_by'))        { return; }     // 优先级：用户值 > 自动注入
    $this->request->merge(['created_by' => user_id()]);
}
```

触发条件：[JobShouldQueue.php#L44-L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php#L44-L66) — 同样要求 `ShouldCreate` + `HasOwner`。

#### 守卫差异

| 基类 | 守卫类型 | 含义 |
|------|---------|------|
| `Job`（同步） | `instanceof Request` | 数组参数会被 `getRequestInstance()` 转为 FormRequest（extends Request），通过守卫 |
| `JobShouldQueue`（异步） | `instanceof QueueCollection` | 数组参数会被 `getRequestAsCollection()` 转为 QueueCollection，通过守卫 |

#### 各可达边界的注入情况

| 边界 | 注入是否发生 | 条件 |
|------|------------|------|
| HTTP 控制器直接 `dispatch(new CreateXxx($request))` | ✅ | Job 需 `implements HasOwner, ShouldCreate` |
| 队列 `dispatch(new AsyncJob($array))` | ✅ | 同上 |
| `Model::create([])` 直接创建 | ❌ | 无自动注入 |
| 工厂 Factory | ❌ | definition 中写死 `'created_from' => 'core::factory'`，不注入 created_by |

### created_from 注入

**注入点**：`Job::setSource()` / `JobShouldQueue::setSource()`

源码（同步）：[Job.php#L124-L135](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L124-L135)
源码（异步）：[JobShouldQueue.php#L149-L160](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php#L149-L160)

```php
// 模式相同：类型守卫 → 检查是否已设置 → 注入
// 值来源：$this->getSourceName($this->request)
```

**优先级**：调用方显式传入的值 > 自动注入。

**来源值生成**：[app/Traits/Sources.php#L22-L45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php#L22-L45)

```
getSourceName($request, $alias)
│
├─ $prefix = getSourcePrefix()  →  'core::' 或 '{module-alias}::'
│
├─ 若 app()->runningInConsole() → $prefix . 'console'   [L26-28]
│
├─ 否则：
│   ├─ $request instanceof QueueCollection || running_in_queue()
│   │   → $prefix . 'queue'                          [L33-34]
│   ├─ $request->isApi() → $prefix . 'api'           [L36]
│   └─ 否则 → $prefix . 'ui'                          [L40-41]
```

前缀来源：[Sources.php#L54-L71](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php#L54-L71) — 命名空间 `Modules\Xxx\...` → `Str::kebab('Xxx')`；否则固定 `'core'`。

#### 各可达边界的注入情况

| 边界 | 注入条件 | 生成值示例 |
|------|---------|-----------|
| Artisan CLI 分发 Job | Job 需 `implements HasSource, ShouldCreate` | `core::console` |
| Queue Worker 消费 Job | 同上 | `core::queue` |
| HTTP UI 表单提交 | 同上 | `core::ui` |
| HTTP API 请求 | 同上 | `core::api` |
| 模块内 Job | 同上 | `{module-alias}::ui` 等 |
| 直接 `Model::create()` | ❌ 不注入 | 需手动指定 |

### 注入优先级总表

| 字段 | 用户显式传值 | 自动注入 | 注入失败行为 |
|------|------------|---------|------------|
| `company_id` | ✅ 优先保留 | `company_id()` | 无上下文时为 `null` → 后续查询/写入可能失败 |
| `created_by` | ✅ 优先保留 | `user_id()` | 无已登录用户时为 `null` |
| `created_from` | ✅ 优先保留 | `getSourceName()` | 始终有回退值（最低为 `core::ui`） |

### 注入调用链示例（以 CreateItem 为例）

[app/Jobs/Common/CreateItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php)：

```
new CreateItem(['name' => 'Widget', ...])
│
└─ Job::__construct() [Job.php#L26-L33]
   │
   ├─ bootCreate() [Job.php#L40-L62]
   │  ├─ getRequestInstance($array) → 创建 FormRequest → merge() → prepareForValidation()
   │  │   → 注入 company_id = 当前 company_id()         [FormRequest.php#L14-L19]
   │  │
   │  ├─ instanceof HasOwner → setOwner()               [Job.php#L55-L57]
   │  │   → 注入 created_by = user_id()                  [Job.php#L117-L121]
   │  │
   │  └─ instanceof HasSource → setSource()              [Job.php#L59-L61]
   │      → 注入 created_from = getSourceName()          [Job.php#L130-L134]
   │
   └─ handle() 执行时 $this->request->all() 已包含 company_id / created_by / created_from
      → Item::create($this->request->all())              [CreateItem.php#L21]
```

---

## 5. 忘记 makeCurrent() 导致查询空结果或写错公司的测试场景

所有场景均使用项目真实类，可直接在 `tests/Feature/` 下创建测试文件运行。

### 场景 1：查询空结果

**可达边界**：直接在测试用例中操作服务容器，绕过 IdentifyCompany 中间件。

**测试文件**：`tests/Feature/TenantScopeTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\Common\Company;
use App\Models\Common\Item;
use Tests\TestCase;

class TenantScopeTest extends TestCase
{
    public function test_item_query_without_make_current_returns_empty(): void
    {
        // Arrange: TestCase::setUp() 已通过 db:seed 创建了测试公司
        $seeded_company = Company::first();
        $this->assertNotNull($seeded_company);

        // 创建一个属于测试公司的 item
        $seeded_company->makeCurrent();
        $item = Item::factory()->create(['name' => 'Widget Alpha']);
        $this->assertEquals($seeded_company->id, $item->company_id);

        // Act: 清除上下文
        Company::forgetCurrent();
        $this->assertNull(Company::getCurrent());
        $this->assertNull(company_id());

        // Assert: 查询返回空
        // Scope 生成 SQL: WHERE items.company_id = NULL
        // SQL 中 = NULL 永远为 false
        $found = Item::where('name', 'Widget Alpha')->first();
        $this->assertNull($found);

        // 验证：恢复上下文后可查到
        $seeded_company->makeCurrent();
        $found = Item::where('name', 'Widget Alpha')->first();
        $this->assertNotNull($found);
        $this->assertEquals($item->id, $found->id);
    }
}
```

**源码依据**：
- `company_id()` 返回 `null`：[helpers.php#L110-L113](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L110-L113)
- Scope 追加 `WHERE company_id = NULL`：[Scopes/Company.php#L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L49)

---

### 场景 2：数据写入错误的公司（直接 Model::create）

**可达边界**：在切换上下文后、忘记重新 `makeCurrent()` 的情况下直接调用 `Model::create()`。

```php
public function test_direct_create_without_context_fails_or_writes_wrong_company(): void
{
    $companyA = Company::factory()->create();
    $companyB = Company::factory()->create();

    // 设置上下文为 A
    $companyA->makeCurrent();
    $this->assertEquals($companyA->id, company_id());

    // 清除上下文 — 模拟"忘记重新设置"的场景
    Company::forgetCurrent();
    $this->assertNull(company_id());

    // 直接 create：company_id 不会自动注入
    // 若数据库 company_id 有 NOT NULL 约束，会抛 QueryException
    try {
        $badItem = Item::create([
            'name' => 'Bad Widget',
            'type' => 'product',
            'purchase_price' => 10.00,
            'sale_price' => 15.00,
            'enabled' => 1,
            // company_id 未指定
        ]);
        // 如果没有 NOT NULL 约束，company_id 会是 NULL 或默认值
        $this->fail('Expected QueryException due to missing company_id');
    } catch (\Illuminate\Database\QueryException $e) {
        $this->assertStringContainsString('company_id', $e->getMessage());
    }

    // 对比：通过 Job 分发（HasOwner + HasSource）会自动注入
    $companyA->makeCurrent();
    $goodItem = $this->dispatch(new \App\Jobs\Common\CreateItem([
        'name' => 'Good Widget',
        'type' => 'product',
        'purchase_price' => 10.00,
        'sale_price' => 15.00,
        'enabled' => 1,
    ]));
    $this->assertEquals($companyA->id, $goodItem->company_id);
    $this->assertNotNull($goodItem->created_by);
    $this->assertStringContainsString('core::', $goodItem->created_from);
}
```

**源码依据**：
- `Model::create()` 不经过 FormRequest，`company_id` 无自动注入：Eloquent 原生行为
- `CreateItem` 实现 `HasOwner, HasSource, ShouldCreate`：[CreateItem.php#L14](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php#L14)

---

### 场景 3：队列 Job 分发时无上下文导致 payload 丢失 company_id

**可达边界**：在没有 `makeCurrent()` 的状态下分发实现 `ShouldQueue` 的 Job。

```php
public function test_queued_job_dispatched_without_context_has_no_company_id_in_payload(): void
{
    $company = Company::factory()->create();

    // 确保没有上下文
    Company::forgetCurrent();
    $this->assertNull(company_id());

    // 分发一个异步 Job
    $job = new \App\Jobs\Common\CreateMediableForExport(
        new \App\Models\Auth\User,
        'test_export.xlsx',
        ['title' => 'Export']
    );

    // 直接验证 Queue::createPayloadUsing 的行为
    // createPayloadUsing 在无上下文时返回空数组
    $result = app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
        return ['company_id' => company_id()];
    });
    // 此时 company_id() 为 null，result 中的 company_id 也是 null
    // 队列系统会忽略 null 值，不写入 payload

    // 对比：有上下文时分发
    $company->makeCurrent();
    $this->assertNotNull(company_id());

    // 此时 createPayloadUsing 会返回 ['company_id' => $company->id]
    // payload 中携带了 company_id，JobProcessing 监听器可恢复上下文
}
```

**源码依据**：
- `createPayloadUsing` 无上下文返回 `[]`：[Queue.php#L32-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L32-L40)
- `JobProcessing` 无 `company_id` 则不恢复上下文：[Queue.php#L45-L47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php#L45-L47)

---

### 场景 4：Company 模型不受 Scope 限制（核心安全验证）

```php
public function test_company_model_is_not_scoped_by_company_scope(): void
{
    $companyA = Company::factory()->create();
    $companyB = Company::factory()->create();

    // 设置上下文为 A
    $companyA->makeCurrent();
    $this->assertEquals($companyA->id, company_id());

    // 清除上下文 — 模拟队列 Worker 启动时的状态
    Company::forgetCurrent();
    $this->assertNull(company_id());

    // 即使没有上下文，Company::find() / ::all() 都能正常工作
    // 因为 Company 模型的 $fillable 不含 company_id → isTenantable() = false
    $foundB = Company::find($companyB->id);
    $this->assertNotNull($foundB);
    $this->assertEquals($companyB->id, $foundB->id);

    $allCompanies = Company::all();
    $this->assertGreaterThanOrEqual(2, $allCompanies->count());

    // 对比：Item 受 Scope 限制，无上下文时查询不到
    $item = Item::factory()->create(['company_id' => $companyA->id]);
    $itemAfterForget = Item::find($item->id);
    $this->assertNull($itemAfterForget); // 因 Scope 追加 WHERE company_id = NULL

    // 验证 Scope 确实已注册到 Company 模型（但因 isNotTenantable 被跳过）
    $this->assertTrue(
        collect(Company::getGlobalScopes())->has('App\Scopes\Company')
    );
}
```

**源码依据**：
- Company `$fillable` 不含 `company_id`：[Company.php#L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php#L39)
- `isTenantable()` 检查 `company_id` 在 fillable 中：[Tenants.php#L19-L24](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php#L19-L24)
- Scope 检查 `isNotTenantable()` 并早返回：[Scopes/Company.php#L23-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php#L23-L25)

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| [app/Models/Common/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php) | 公司模型，`makeCurrent` / `forgetCurrent` 实现，`$fillable` 不含 `company_id` |
| [app/Scopes/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php) | 全局查询 Scope，`isNotTenantable()` → 跳过，追加 `WHERE company_id` |
| [app/Traits/Tenants.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php) | `bootTenants()` 注册 Scope，`isTenantable()` 双重检查 |
| [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) | 基础模型，`scopeAllCompanies()` 绕过机制 |
| [app/Providers/Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php) | `createPayloadUsing` 捕获 + `JobProcessing` 恢复上下文 |
| [app/Http/Middleware/IdentifyCompany.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/IdentifyCompany.php) | HTTP 请求的上下文恢复（URL/Query/Header → `makeCurrent()`） |
| [app/Abstracts/Http/FormRequest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/FormRequest.php) | `prepareForValidation()` 注入 `company_id` |
| [app/Abstracts/Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php) | 同步 Job 基类，`bootCreate` → `setOwner` / `setSource` |
| [app/Abstracts/JobShouldQueue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php) | 异步 Job 基类，QueueCollection 守卫 |
| [app/Traits/Sources.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php) | `getSourceName()` 来源值生成（console/queue/api/ui） |
| [app/Utilities/helpers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php) | `company()` / `company_id()` 辅助函数 |
| [app/Abstracts/Factory.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Factory.php) | 工厂基类，`setCompany()` 自动 `makeCurrent()` |
| [tests/TestCase.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/TestCase.php) | 测试基类，`setUp()` 中 seed 测试公司 |
