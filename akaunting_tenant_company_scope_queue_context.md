# Akaunting 多公司/租户上下文系统分析

## 1. Company::makeCurrent/forgetCurrent 触发的事件与全局状态影响

### 事件触发流程

在 [app/Models/Common/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php) 中定义了完整的上下文切换生命周期事件：

**makeCurrent() 流程（第 601-627 行）：**
```
调用 makeCurrent()
    ↓
1.  如果已是当前公司，直接返回
    ↓
2.  调用 forgetCurrent() 清除现有上下文
    ↓
3.  触发事件：CompanyMakingCurrent($this)  [第 609 行]
    ↓
4.  绑定到容器：app()->instance(static::class, $this)  [第 612 行]
    ↓
5.  设置/加载公司配置
    - setting()->setExtraColumns(['company_id' => $this->id])
    - setting()->forgetAll()
    - setting()->load(true)
    - Overrider::load('settings')
    - Overrider::load('currencies')
    - Overrider::load('categoryTypes')
    ↓
6.  触发事件：CompanyMadeCurrent($this)  [第 624 行]
```

**forgetCurrent() 流程（第 648-667 行）：**
```
调用 forgetCurrent()
    ↓
1.  获取当前公司实例
    ↓
2.  触发事件：CompanyForgettingCurrent($current)  [第 656 行]
    ↓
3.  从容器移除：app()->forgetInstance(static::class)  [第 659 行]
    ↓
4.  清除配置：setting()->forgetAll()  [第 662 行]
    ↓
5.  触发事件：CompanyForgotCurrent($current)  [第 664 行]
```

### 影响的全局状态

| 状态类型 | 影响内容 |
|---------|---------|
| **服务容器** | Company 实例被绑定/解除绑定到 Laravel 服务容器 |
| **配置系统** | setting() 系统的 company_id 额外列被设置，配置被重新加载 |
| **覆盖器** | 公司设置、货币、分类类型被 Overrider 重新加载 |
| **后续查询** | 所有使用 Company Scope 的模型查询将自动过滤到当前公司 |

### 事件类定义

四个事件类均位于 [app/Events/Common/](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/) 目录：
- [CompanyMakingCurrent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/CompanyMakingCurrent.php)
- [CompanyMadeCurrent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/CompanyMadeCurrent.php)
- [CompanyForgettingCurrent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/CompanyForgettingCurrent.php)
- [CompanyForgotCurrent.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Common/CompanyForgotCurrent.php)

---

## 2. Company Scope 查询限制与 allCompanies 绕过机制

### Company Scope 工作原理

在 [app/Scopes/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php) 中定义了全局查询范围：

**应用条件（第 21-50 行）：**
```php
public function apply(Builder $builder, Model $model)
{
    // 1. 检查模型是否可租户化
    if (method_exists($model, 'isNotTenantable') && $model->isNotTenantable()) {
        return;
    }

    // 2. 跳过特定表（跨公司表）
    $skip_tables = [
        'jobs', 'firewall_ips', 'firewall_logs', 'migrations', 'notifications', 
        'role_companies', 'role_permissions', 'sessions', 'user_companies', 
        'user_dashboards', 'user_permissions', 'user_roles',
    ];

    // 3. 如果已存在 company_id 条件则跳过（避免重复）
    if ($this->scopeColumnExists($builder, '', 'company_id')) {
        return;
    }

    // 4. 应用公司过滤
    $builder->where($table . '.company_id', '=', company_id());
}
```

### 模型启用 Scope 的方式

通过 [app/Traits/Tenants.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php) 的 `bootTenants()` 方法（第 14-17 行）自动添加全局 Scope：

```php
protected static function bootTenants()
{
    static::addGlobalScope(new Company);
}
```

所有继承自 [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) 的模型都会自动使用 `Tenants` trait（第 23 行）。

### allCompanies 绕过机制

在 [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) 第 77-80 行定义了 `allCompanies()` 本地 scope：

```php
public function scopeAllCompanies($query)
{
    return $query->withoutGlobalScope('App\Scopes\Company');
}
```

**使用示例：**
```php
// 普通查询 - 自动过滤到当前公司
$invoices = Invoice::all(); // 只返回当前公司的发票

// 绕过 scope - 查询所有公司
$allInvoices = Invoice::allCompanies()->get(); // 返回所有公司的发票

// 链式调用
$paidInvoices = Invoice::allCompanies()->where('status', 'paid')->get();
```

### 租户化检查

模型可以通过设置 `$tenantable = false` 来禁用租户过滤（[Tenants.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php) 第 19-29 行）：

```php
public function isTenantable()
{
    $tenantable = $this->tenantable ?? true;
    return ($tenantable === true) && in_array('company_id', $this->getFillable());
}
```

---

## 3. Queued Job 恢复 Company Context 机制

在 [app/Providers/Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php) 中实现了队列 Job 的公司上下文自动恢复。

### 上下文存储（Job 创建时）

第 32-40 行，在创建 Job payload 时注入当前 company_id：

```php
app('queue')->createPayloadUsing(function ($connection, $queue, $payload) {
    $company_id = company_id();
    
    if (empty($company_id)) {
        return [];
    }
    
    return ['company_id' => $company_id];
});
```

### 上下文恢复（Job 执行时）

第 42-80 行，监听 `JobProcessing` 事件恢复上下文：

```php
app('events')->listen(JobProcessing::class, function ($event) {
    $payload = $event->job->payload();
    
    // 1. 检查 payload 中是否有 company_id
    if (! array_key_exists('company_id', $payload)) {
        return;
    }
    
    try {
        // 2. 查找公司
        $company = company($payload['company_id']);
    } catch (\Throwable $e) {
        // 3. 公司不存在或异常，删除 Job 并记录日志
        $event->job->delete();
        logger()->warning('Company could not be resolved...');
        return;
    }
    
    if (empty($company)) {
        $event->job->delete();
        logger()->warning('Company not found, job deleted...');
        return;
    }
    
    // 4. 恢复公司上下文
    $company->makeCurrent();
    
    // 5. 注册模块（如果启用队列）
    if (should_queue()) {
        $this->registerModules();
    }
});
```

### 恢复流程总结

```
Job 入队时
    ↓
createPayloadUsing 捕获当前 company_id
    ↓
company_id 被序列化到 payload 中
    ↓
========================================
    ↓
Job 出队执行时
    ↓
JobProcessing 事件触发
    ↓
从 payload 提取 company_id
    ↓
查询 Company 模型
    ↓
调用 $company->makeCurrent() 恢复上下文
    ↓
Job 业务逻辑在正确的公司上下文中执行
```

---

## 4. created_from/created_by/company_id 默认值注入机制

### company_id 注入

**位置：** [app/Abstracts/Http/FormRequest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/FormRequest.php) 第 14-19 行

```php
protected function prepareForValidation()
{
    $this->merge([
        'company_id' => company_id(),
    ]);
}
```

所有继承 `FormRequest` 的请求类在验证前会自动注入当前 `company_id`。

### created_by 注入（创建者）

**位置：** [app/Abstracts/Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php) 第 111-122 行（同步 Job）和 [app/Abstracts/JobShouldQueue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php) 第 136-147 行（异步 Job）

```php
public function setOwner(): void
{
    if (! $this->request instanceof Request) {
        return;
    }
    
    if ($this->request->has('created_by')) {
        return; // 已手动设置则跳过
    }
    
    $this->request->merge(['created_by' => user_id()]);
}
```

**触发条件：** Job 类需要实现 `HasOwner` 接口，且在 `bootCreate()` 中自动调用（[Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php) 第 55-57 行）。

### created_from 注入（来源）

**位置：** [app/Abstracts/Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php) 第 124-135 行（同步 Job）和 [app/Abstracts/JobShouldQueue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php) 第 149-160 行（异步 Job）

```php
public function setSource(): void
{
    if (! $this->request instanceof Request) {
        return;
    }
    
    if ($this->request->has('created_from')) {
        return; // 已手动设置则跳过
    }
    
    $this->request->merge(['created_from' => $this->getSourceName($this->request)]);
}
```

**触发条件：** Job 类需要实现 `HasSource` 接口。

**来源值生成规则**（[app/Traits/Sources.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php) 第 22-45 行）：
- CLI 环境：`core::console`
- 队列环境：`core::queue`
- API 请求：`core::api`
- Web UI：`core::ui`
- 模块来源：`{module-alias}::{source}`

### 注入时机总结

| 字段 | 注入位置 | 触发时机 |
|------|---------|---------|
| `company_id` | FormRequest::prepareForValidation() | 请求验证前 |
| `created_by` | Job::setOwner() | Job 构造时（bootCreate），需实现 HasOwner |
| `created_from` | Job::setSource() | Job 构造时（bootCreate），需实现 HasSource |

---

## 5. 忘记 makeCurrent 导致查询空结果或写错公司的测试场景

### 场景 1：查询空结果

**问题描述：** 在没有设置当前公司上下文的情况下查询租户模型，返回空结果。

**测试代码：**
```php
<?php

namespace Tests\Feature;

use App\Models\Common\Company;
use App\Models\Setting\Category;
use Tests\TestCase;

class TenantContextTest extends TestCase
{
    public function test_query_without_make_current_returns_empty()
    {
        // 1. 创建公司和分类
        $company = Company::factory()->create();
        $category = Category::factory()->create([
            'company_id' => $company->id,
            'name' => 'Test Category',
        ]);
        
        // 2. 确保没有当前公司上下文
        Company::forgetCurrent();
        $this->assertNull(Company::getCurrent());
        $this->assertNull(company_id());
        
        // 3. 尝试查询 - 这将返回空！
        // 因为 scope 添加 WHERE company_id = NULL 条件
        $found = Category::where('name', 'Test Category')->first();
        $this->assertNull($found);
        
        // 4. 验证：设置上下文后才能查询到
        $company->makeCurrent();
        $found = Category::where('name', 'Test Category')->first();
        $this->assertNotNull($found);
        $this->assertEquals($category->id, $found->id);
    }
}
```

**原因分析：**
- `company_id()` 返回 `null`
- Company Scope 生成的 SQL: `WHERE categories.company_id = NULL`
- SQL 中 `= NULL` 永远为 false，所以查询结果为空

---

### 场景 2：数据写入错误的公司

**问题描述：** 在切换公司后忘记重新设置上下文，导致后续数据写入错误的公司。

**测试代码：**
```php
public function test_data_written_to_wrong_company_after_switch()
{
    // 1. 创建两个公司
    $companyA = Company::factory()->create();
    $companyB = Company::factory()->create();
    
    // 2. 先在公司 A 上下文中
    $companyA->makeCurrent();
    $this->assertEquals($companyA->id, company_id());
    
    // 3. 创建分类 - 应该属于公司 A
    $categoryA = Category::factory()->create(['name' => 'Category A']);
    $this->assertEquals($companyA->id, $categoryA->company_id);
    
    // 4. 切换到公司 B，但后续操作忘记...
    $companyB->makeCurrent();
    
    // 5. 模拟一个"忘记上下文"的场景（比如 forgetCurrent 后忘记重新设置）
    Company::forgetCurrent();
    
    // 6. 错误：直接使用之前的模型创建关联数据
    // 注意：这是一个反例演示
    try {
        // 在没有上下文的情况下创建
        $badCategory = Category::create([
            'name' => 'Bad Category',
            'type' => 'item',
            'color' => 'blue',
            // company_id 会被 FormRequest 注入，但如果是直接创建...
        ]);
    } catch (\Exception $e) {
        // 可能触发 NOT NULL 约束错误
        $this->assertStringContainsString('company_id', $e->getMessage());
    }
    
    // 7. 正确做法：始终确保上下文已设置
    $companyA->makeCurrent();
    $goodCategory = Category::factory()->create(['name' => 'Good Category']);
    $this->assertEquals($companyA->id, $goodCategory->company_id);
}
```

---

### 场景 3：队列 Job 丢失上下文

**问题描述：** 在没有上下文的情况下分发 Job，导致 Job 执行时无法恢复正确的公司。

**测试代码：**
```php
public function test_queued_job_without_company_context()
{
    $company = Company::factory()->create();
    
    // 1. 确保没有当前公司
    Company::forgetCurrent();
    
    // 2. 分发 Job - 此时 company_id() 返回 null
    // createPayloadUsing 不会注入 company_id
    $job = new SomeImportJob($data);
    dispatch($job);
    
    // 3. Job 执行时：
    // - payload 中没有 company_id
    // - JobProcessing 监听器跳过上下文恢复
    // - Job 内部查询将返回空或写入错误
    
    // 4. 正确做法：分发前确保上下文已设置
    $company->makeCurrent();
    $job = new SomeImportJob($data);
    dispatch($job); // payload 包含 company_id = $company->id
}
```

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| [app/Models/Common/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Company.php) | 公司模型，makeCurrent/forgetCurrent 实现 |
| [app/Scopes/Company.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Company.php) | 全局查询 Scope，自动添加 company_id 过滤 |
| [app/Traits/Tenants.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Tenants.php) | 租户 Trait，自动注册 Scope |
| [app/Providers/Queue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Queue.php) | 队列上下文恢复 |
| [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) | 基础模型，allCompanies scope |
| [app/Abstracts/Http/FormRequest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/FormRequest.php) | 请求基类，注入 company_id |
| [app/Abstracts/Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php) | 同步 Job 基类，注入 created_by/created_from |
| [app/Abstracts/JobShouldQueue.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/JobShouldQueue.php) | 异步 Job 基类 |
| [app/Traits/Sources.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php) | 来源 Trait，getSourceName |
| [app/Utilities/helpers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php) | 辅助函数 company(), company_id() |
| [tests/TestCase.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/TestCase.php) | 测试基类 |
