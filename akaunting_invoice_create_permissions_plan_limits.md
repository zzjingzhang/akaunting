# Web 创建 Invoice 与 API 创建 Document 的权限和计划限制路径分析

## 1. 两个入口各自经过的 Route/Controller/Job 层

### 1.1 Web 创建 Invoice 路径

**Route 层**
- 路由定义: [admin.php:83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php#L83-L83)
  ```php
  Route::resource('invoices', 'Sales\Invoices', ['middleware' => ['date.format', 'money', 'dropzone']]);
  ```
- 完整路径: `GET/POST /{company_id}/invoices` → 路由到 `Sales\Invoices@store`
- 路由注册: 通过 `mapAdminRoutes()` [Route.php:228-234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Route.php#L228-L234) 添加 `{company_id}` 前缀
- **admin 中间件组** [Kernel.php:76-88](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L76-L88):
  ```php
  'admin' => [
      'web',
      'auth',
      'auth.disabled',
      'company.identify',
      'bindings',
      'read.only',
      'wizard.redirect',
      'menu.admin',
      'permission:read-admin-panel',  // 基础管理面板权限
      'plan.limits',                   // RedirectIfHitPlanLimits
      'module.subscription',
  ],
  ```

**Controller 层**
- 控制器: [Sales/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php)
- 继承链: `Invoices` → `Controller` (抽象基类)
- 构造函数: [Controller.php:29-32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php#L29-L32) 调用 `assignPermissionsToController()`
- Store 方法: [Invoices.php:100-125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L100-L125)
  ```php
  public function store(Request $request)
  {
      $response = $this->ajaxDispatch(new CreateDocument($request));
      // ... 处理响应
  }
  ```
- 使用 `ajaxDispatch()` [Jobs.php:55-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Jobs.php#L55-L76) 调度 Job，**会捕获异常**

**Job 层**
- Job 类: [Jobs/Document/CreateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php)
- Handle 方法: [CreateDocument.php:20-57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L20-L57)
  - 第一步调用 `$this->authorize()` [CreateDocument.php:62-67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L62-L67) 进行计划限制检查
  - 然后在数据库事务中创建 Document、处理附件、创建条目和总计、处理递归

### 1.2 API 创建 Document 路径

**Route 层**
- 路由定义: [api.php:43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/api.php#L43-L43)
  ```php
  Route::apiResource('documents', 'Document\Documents', ['middleware' => ['date.format', 'money', 'dropzone']]);
  ```
- 完整路径: `POST /api/documents` → 路由到 `Api\Document\Documents@store`
- 路由注册: 通过 `mapApiRoutes()` [Route.php:168-175](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Route.php#L168-L175) 添加 `api` 前缀
- **api 中间件组** [Kernel.php:50-60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L50-L60):
  ```php
  'api' => [
      'auth.basic.once',
      'auth.disabled',
      'throttle:api',
      'permission:read-api',          // API 访问权限
      'company.identify',
      'bindings',
      'read.only',
      'language',
      'firewall.all',
  ],
  ```

**Controller 层**
- 控制器: [Api/Document/Documents.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php)
- 继承链: `Documents` → `ApiController` (抽象基类)
- 构造函数: [ApiController.php:24-27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/ApiController.php#L24-L27) 调用 `assignPermissionsToController()`
- **关键发现**: API 的 Documents 控制器在 `assignPermissionsToController()` 中**只从 search 参数读取 type**，而非 POST body
  - [Permissions.php:435-457](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L435-L457):
    ```php
    if (in_array($table, ['contacts', 'documents'])) {
        $type = $this->getSearchStringValue('type');  // 仅从 search 参数获取
        // ...
    }
    ```
- Store 方法: [Documents.php:80-85](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L80-L85)
  ```php
  public function store(Request $request)
  {
      $document = $this->dispatch(new CreateDocument($request));
      return $this->created(route('api.documents.show', $document->id), new Resource($document));
  }
  ```
- 使用 `$this->dispatch()` [Jobs.php:17-22](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Jobs.php#L17-L22) 调度 Job，**不捕获异常**

**Job 层**
- 与 Web 入口共用同一个 Job: [Jobs/Document/CreateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php)

---

## 2. 权限校验和 Plan Limit 校验的层次及失败表现

### 2.1 权限校验 (Permission Check)

**发生层次**: Controller 层构造函数中，通过中间件实现

**实现机制**:
- 基类构造函数调用 `assignPermissionsToController()` [Permissions.php:425-500](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L425-L500)
- 对于 Web 入口 (`Sales\Invoices`):
  - 根据控制器路径解析: `sales-invoices`
  - 自动分配中间件: `permission:create-sales-invoices` 作用于 `store` 方法
- 对于 API 入口 (`Api\Document\Documents`):
  - **仅从 `search` 查询参数解析 type**，而非 POST body
  - 如 `GET /api/documents?search=type:invoice` 或 `POST /api/documents?search=type:invoice`
  - 如果 `type=invoice`，需要 `create-sales-invoices` 权限
  - 如果 `type=bill`，需要 `create-purchases-bills` 权限

**失败表现**:
- 权限中间件 [Kernel.php:192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L192) 使用 `LaratrustPermission::class`
- 失败时抛出 `Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException`
- Web 入口: 返回 403 页面或重定向
- API 入口: 返回 403 Forbidden JSON 响应

### 2.2 Plan Limit 校验 (计划限制检查)

**Plan Limit 校验存在三种层次和三种失败表现**:

#### 2.2.1 Web GET /create 页面的重定向检查

**发生位置**: `plan.limits` 中间件 [RedirectIfHitPlanLimits.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/RedirectIfHitPlanLimits.php)

**触发条件**:
```php
// [RedirectIfHitPlanLimits.php:24-26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/RedirectIfHitPlanLimits.php#L24-L26)
if (! $request->isMethod(strtolower('GET')) || ! in_array($last_segment, ['create'])) {
    return $next($request);
}
```
- 仅对 GET 请求生效
- 仅对最后一个 URL segment 为 `create` 的路由生效

**检查逻辑**:
```php
// [RedirectIfHitPlanLimits.php:36-46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/RedirectIfHitPlanLimits.php#L36-L46)
if (! $this->getUserLimitOfPlan()->action_status) {
    return redirect()->route('users.index');
}

if (! $this->getCompanyLimitOfPlan()->action_status) {
    return redirect()->route('companies.index');
}

if (! $this->getInvoiceLimitOfPlan()->action_status) {
    return redirect()->route('invoices.index');
}
```

**失败表现**: 直接重定向到对应列表页，无错误消息提示

#### 2.2.2 Web POST /store 的 AJAX 异常捕获

**发生位置**: Job 层 `CreateDocument::authorize()` → `ajaxDispatch()` 捕获

**检查逻辑**:
```php
// [CreateDocument.php:62-67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L62-L67)
public function authorize(): void
{
    $limit = $this->getAnyActionLimitOfPlan();
    if (! $limit->action_status && ! empty($this->request['type']) && ($this->request['type'] == 'invoice')) {
        throw new \Exception($limit->message);
    }
}
```

**失败表现**:
```php
// [Jobs.php:55-76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Jobs.php#L55-L76)
public function ajaxDispatch($job)
{
    try {
        $data = $this->dispatch($job);
        $response = ['success' => true, ...];
    } catch (Exception | Throwable $e) {
        $response = [
            'success' => false,
            'error' => true,
            'data' => null,
            'message' => $e->getMessage(),  // 错误消息包含在响应中
        ];
    }
    return $response;
}
```
- 返回 JSON: `{"success": false, "message": "限制信息"}`
- 前端显示 error flash 消息，停留在创建页面

#### 2.2.3 API POST /documents 的异常处理

**发生位置**: Job 层 `CreateDocument::authorize()` → 未被捕获，进入全局异常处理器

**失败表现**:
- API 控制器使用 `dispatch()` 而非 `ajaxDispatch()`
- 异常直接抛出，由 Laravel 异常处理器转为 500 Internal Server Error JSON 响应

---

## 3. CreateDocument::authorize 只在 type == invoice 时抛出计划限制异常

### 3.1 代码分析

[CreateDocument.php:62-67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L62-L67):

```php
public function authorize(): void
{
    $limit = $this->getAnyActionLimitOfPlan();
    if (! $limit->action_status && ! empty($this->request['type']) && ($this->request['type'] == 'invoice')) {
        throw new \Exception($limit->message);
    }
}
```

### 3.2 源码证据

**Plan 限制检查逻辑** [Plans.php:33-57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L33-L57):
```php
public function getAnyActionLimitOfPlan(): object
{
    $user_limit = $this->getUserLimitOfPlan();
    $company_limit = $this->getCompanyLimitOfPlan();
    $invoice_limit = $this->getInvoiceLimitOfPlan();

    if (! $user_limit->action_status) {
        return $user_limit;
    }

    if (! $company_limit->action_status) {
        return $company_limit;
    }

    if (! $invoice_limit->action_status) {
        return $invoice_limit;
    }
    // ... 返回成功状态
}
```

**关键发现**:
1. `getAnyActionLimitOfPlan()` 检查三种限制: user、company、invoice
2. 但 `CreateDocument::authorize()` 只有在 `type == 'invoice'` 时才抛出异常
3. 当 `type != 'invoice'` 时，即使 `getAnyActionLimitOfPlan()` 返回失败状态，也不会抛出异常

### 3.3 设计缺陷确认

这是一个明确的代码缺陷，因为:
- `getAnyActionLimitOfPlan()` 返回失败状态可能是因为 user 或 company 限制触发
- 但由于 `type != 'invoice'` 的条件，bill、invoice-recurring 等类型都会绕过检查
- 这意味着即使公司已达计划上限，仍然可以创建 bill 和 invoice-recurring

---

## 4. bill、invoice-recurring、异常 type 的保护机制分析

### 4.1 Bill 类型 (`type == 'bill'`)

#### Web 创建页 (GET /{company_id}/bills/create)
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-admin-panel` | ✅ | admin 中间件组强制检查 |
| `permission:create-purchases-bills` | ✅ | 控制器构造函数自动分配 |
| `plan.limits` 中间件 | ✅ | 检查 user/company/invoice 限制，失败时重定向 |
| FormRequest 验证 | ✅ | [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php) |

#### Web POST /{company_id}/bills
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:create-purchases-bills` | ✅ | 中间件检查 |
| FormRequest 验证 | ✅ | 数据验证 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |
| `ajaxDispatch()` 异常捕获 | ✅ | 仅捕获其他异常（如数据库错误） |

#### API 请求 (POST /api/documents)
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-api` | ✅ | api 中间件组强制检查 |
| `permission:create-purchases-bills` | ⚠️ | **仅从 search 参数获取 type，POST body 中的 type 不影响权限检查** |
| FormRequest 验证 | ✅ | 数据验证 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |
| 异常处理 | ✅ | 异常直接抛出，返回 500 |

#### 直接 dispatch Job
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| 权限检查 | ❌ | **完全跳过** |
| FormRequest 验证 | ❌ | **完全跳过** |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |

### 4.2 Invoice-Recurring 类型 (`type == 'invoice-recurring'`)

#### Web 创建页 (GET /{company_id}/recurring-invoices/create)
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-admin-panel` | ✅ | admin 中间件组 |
| `permission:create-sales-invoices` | ✅ | [RecurringInvoices.php:33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L33) 显式定义 |
| `plan.limits` 中间件 | ✅ | 检查限制 |
| FormRequest 验证 | ✅ | 数据验证 |

#### Web POST /{company_id}/recurring-invoices
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:create-sales-invoices` | ✅ | 中间件检查 |
| FormRequest 验证 | ✅ | 数据验证 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |
| `ajaxDispatch()` 异常捕获 | ✅ | 捕获其他异常 |

#### API 请求 (POST /api/documents?search=type:invoice-recurring)
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-api` | ✅ | api 中间件组 |
| `permission:create-sales-invoices` | ⚠️ | 依赖 search 参数中的 type |
| FormRequest 验证 | ✅ | 数据验证 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |

#### 直接 dispatch Job
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| 权限检查 | ❌ | **完全跳过** |
| FormRequest 验证 | ❌ | **完全跳过** |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |

### 4.3 异常 type（如 `type == 'fake'`）

#### Web 创建页
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-admin-panel` | ✅ | admin 中间件组 |
| 控制器权限 | ✅ | 基于控制器路径，与 type 无关 |
| `plan.limits` 中间件 | ✅ | 不依赖 type |

#### Web POST
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| 权限检查 | ✅ | 基于控制器路径 |
| FormRequest 验证 | ⚠️ | `type` 字段仅验证 `required|string`，不验证是否为合法类型 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |
| 数据库写入 | ⚠️ | 可能写入脏数据或触发错误 |

#### API 请求
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| `permission:read-api` | ✅ | api 中间件组 |
| 动态权限解析 | ❌ | `config('type.document.fake')` 返回 null，权限解析失败 |
| FormRequest 验证 | ⚠️ | 仅验证 type 为字符串 |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |

#### 直接 dispatch Job
| 保护机制 | 状态 | 说明 |
|---------|------|------|
| 权限检查 | ❌ | **完全跳过** |
| FormRequest 验证 | ❌ | **完全跳过** |
| `CreateDocument::authorize()` | ❌ | **type != 'invoice'，跳过检查** |
| 数据库写入 | ⚠️ | 可能写入脏数据 |

---

## 5. 绕过控制器直接 dispatch Job 的测试思路

### 5.1 测试目的

验证当绕过 Controller 层后，Job 层的校验行为，特别是:
1. Plan Limit 检查对不同 type 的实际效果
2. 权限检查是否完全依赖 Controller 层
3. 如何稳定模拟 `getAnyActionLimitOfPlan()` 失败（避免 `running_in_test()` 影响）

### 5.2 关键技术点

**`running_in_test()` 问题**:
```php
// [Plans.php:59-69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L59-L69)
public function getPlanLimitByType($type): object
{
    if (! config('app.installed') || running_in_test()) {
        $limit = new \stdClass();
        $limit->action_status = true;  // 测试环境直接返回成功
        $limit->view_status = true;
        $limit->message = "Success";
        return $limit;
    }
    // ...
}
```

**解决方案**: 使用 Mockery 或 Reflection 绕过此检查

### 5.3 测试代码

#### 测试 1: 验证 Plan Limit 对不同 type 的检查效果

```php
public function test_create_document_job_plan_limit_check()
{
    // 1. 准备失败的 limit 对象
    $failedLimit = new \stdClass();
    $failedLimit->action_status = false;
    $failedLimit->view_status = false;
    $failedLimit->message = 'Plan limit exceeded';

    // 2. Mock Plans trait 的 getAnyActionLimitOfPlan 方法
    $job = \Mockery::mock(\App\Jobs\Document\CreateDocument::class, [$this->createRequest('invoice')])
        ->makePartial();
    $job->shouldReceive('getAnyActionLimitOfPlan')
        ->andReturn($failedLimit);

    // 3. 测试 type = invoice（应该抛出异常）
    $this->expectException(\Exception::class);
    $this->expectExceptionMessage('Plan limit exceeded');
    $job->handle();
}

public function test_create_document_job_plan_limit_ignores_bill()
{
    // 1. 准备失败的 limit 对象
    $failedLimit = new \stdClass();
    $failedLimit->action_status = false;
    $failedLimit->message = 'Plan limit exceeded';

    // 2. Mock getAnyActionLimitOfPlan 返回失败状态
    $job = \Mockery::mock(\App\Jobs\Document\CreateDocument::class, [$this->createRequest('bill')])
        ->makePartial();
    $job->shouldReceive('getAnyActionLimitOfPlan')
        ->andReturn($failedLimit);

    // 3. 测试 type = bill（不应该抛出异常）
    // 这证明了 bill 类型绕过了 plan limit 检查
    try {
        $result = $job->handle();
        // 如果执行到这里，说明 bill 确实绕过了检查
        $this->assertNotNull($result);
    } catch (\Exception $e) {
        $this->fail('Bill should not be restricted by plan limit check');
    }
}

public function test_create_document_job_plan_limit_ignores_invoice_recurring()
{
    // 1. 准备失败的 limit 对象
    $failedLimit = new \stdClass();
    $failedLimit->action_status = false;
    $failedLimit->message = 'Plan limit exceeded';

    // 2. Mock getAnyActionLimitOfPlan 返回失败状态
    $job = \Mockery::mock(\App\Jobs\Document\CreateDocument::class, [$this->createRequest('invoice-recurring')])
        ->makePartial();
    $job->shouldReceive('getAnyActionLimitOfPlan')
        ->andReturn($failedLimit);

    // 3. 测试 type = invoice-recurring（不应该抛出异常）
    try {
        $result = $job->handle();
        $this->assertNotNull($result);
    } catch (\Exception $e) {
        $this->fail('Invoice-recurring should not be restricted by plan limit check');
    }
}

protected function createRequest(string $type): \Illuminate\Http\Request
{
    $request = new \Illuminate\Http\Request();
    $request->merge([
        'type' => $type,
        'document_number' => 'TEST-' . strtoupper($type) . '-001',
        'status' => 'draft',
        'issued_at' => now()->format('Y-m-d H:i:s'),
        'due_at' => now()->addDays(30)->format('Y-m-d H:i:s'),
        'amount' => 100,
        'currency_code' => 'USD',
        'currency_rate' => 1,
        'contact_id' => 1,
        'contact_name' => 'Test Contact',
        'category_id' => 1,
        'items' => [[
            'name' => 'Test Item',
            'quantity' => 1,
            'price' => 100,
        ]],
    ]);
    return $request;
}
```

#### 测试 2: 验证权限检查完全依赖 Controller 层

```php
public function test_job_has_no_permission_check()
{
    // 1. 创建无权限用户
    $user = \App\Models\Auth\User::factory()->create();
    $user->detachAllPermissions();
    $this->actingAs($user);

    // 2. 准备成功的 plan limit（避免 plan limit 干扰）
    $successLimit = new \stdClass();
    $successLimit->action_status = true;
    $successLimit->message = 'Success';

    // 3. 创建 Job（绕过 Controller 层）
    $job = \Mockery::mock(\App\Jobs\Document\CreateDocument::class, [$this->createRequest('invoice')])
        ->makePartial();
    $job->shouldReceive('getAnyActionLimitOfPlan')
        ->andReturn($successLimit);

    // 4. 执行 Job（应该成功，证明无权限检查）
    $result = $job->handle();
    
    $this->assertInstanceOf(\App\Models\Document\Document::class, $result);
    $this->assertEquals('invoice', $result->type);
    // 结论：Job 层完全没有权限检查，依赖 Controller 层中间件
}
```

#### 测试 3: 使用 Reflection 绕过 running_in_test()

```php
public function test_plan_limit_check_with_reflection()
{
    // 1. 使用 Reflection 修改 running_in_test() 返回值
    // 或者直接 Mock Plans trait 的方法
    
    $request = $this->createRequest('invoice');
    $job = new \App\Jobs\Document\CreateDocument($request);

    // 2. 使用 Reflection 获取 Plans trait 的方法
    $reflection = new \ReflectionClass($job);
    $method = $reflection->getMethod('getPlanLimitByType');
    $method->setAccessible(true);

    // 3. 验证测试环境下的行为
    $limit = $method->invoke($job, 'invoice');
    $this->assertTrue($limit->action_status);  // 测试环境默认返回成功

    // 4. 模拟生产环境：修改 config 和 env
    config(['app.installed' => true]);
    putenv('APP_ENV=production');
    
    // 此时 getPlanLimitByType 会调用 getPlanLimits() 获取真实限制
    // 需要 Mock Cache 来返回失败状态
    
    \Illuminate\Support\Facades\Cache::shouldReceive('remember')
        ->once()
        ->andReturn((object)[
            'user' => (object)['action_status' => false, 'message' => 'User limit exceeded'],
            'company' => (object)['action_status' => true],
            'invoice' => (object)['action_status' => true],
        ]);

    // 5. 重新执行测试
    $limit = $method->invoke($job, 'invoice');
    $this->assertFalse($limit->action_status);
}
```

### 5.4 测试预期结论

| 测试场景 | 预期结果 | 源码依据 |
|---------|---------|---------|
| invoice + plan limit 失败 | 抛出异常 | [CreateDocument.php:62-67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L62-L67) |
| bill + plan limit 失败 | 不抛出异常 | `type != 'invoice'` 条件不满足 |
| invoice-recurring + plan limit 失败 | 不抛出异常 | `type != 'invoice'` 条件不满足 |
| 无权限用户直接 dispatch | 成功创建 | Job 层无权限检查逻辑 |
| 测试环境直接执行 | plan limit 跳过 | [Plans.php:59-69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L59-L69) |

---

## 总结

### 校验层次总结

| 校验类型 | Web Invoice | Web Bill | API Document | 直接 Dispatch |
|---------|------------|----------|--------------|---------------|
| `permission:read-admin-panel` | ✅ 中间件 | ✅ 中间件 | ❌ | ❌ |
| `permission:read-api` | ❌ | ❌ | ✅ 中间件 | ❌ |
| 动态权限检查 | ✅ Controller | ✅ Controller | ⚠️ 仅从 search 参数 | ❌ |
| `plan.limits` 中间件 (GET) | ✅ 重定向 | ✅ 重定向 | ❌ | ❌ |
| `CreateDocument::authorize()` | ✅ 仅 invoice | ❌ | ✅ 仅 invoice | ✅ 仅 invoice |
| FormRequest 验证 | ✅ | ✅ | ✅ | ❌ |

### 核心问题

1. **Plan Limit 检查不完整**: `CreateDocument::authorize()` 只检查 `type == 'invoice'`，bill 和 invoice-recurring 可绕过
2. **API 权限解析缺陷**: 仅从 `search` 参数获取 type，POST body 中的 type 不影响权限检查
3. **Job 层无权限保护**: 绕过 Controller 可直接创建任意文档
4. **测试环境绕过**: `running_in_test()` 导致 plan limit 检查在测试中始终返回成功

### 代码优化建议

针对 `CreateDocument::authorize()` 的修复建议：

```php
public function authorize(): void
{
    $limit = $this->getAnyActionLimitOfPlan();
    
    if (! $limit->action_status && ! empty($this->request['type'])) {
        // 对所有类型都进行检查，而不仅仅是 invoice
        throw new \Exception($limit->message);
    }
}
```

或更精细的控制：

```php
public function authorize(): void
{
    $limit = $this->getAnyActionLimitOfPlan();
    
    if (! $limit->action_status && ! empty($this->request['type'])) {
        $type = $this->request['type'];
        
        // invoice 和 invoice-recurring 都应该受发票限制约束
        if (in_array($type, ['invoice', 'invoice-recurring'])) {
            throw new \Exception($limit->message);
        }
        
        // user 和 company 限制对所有类型生效
        if (!$this->getUserLimitOfPlan()->action_status || !$this->getCompanyLimitOfPlan()->action_status) {
            throw new \Exception($limit->message);
        }
    }
}
```
