# Web 创建 Invoice 与 API 创建 Document 的权限和计划限制路径分析

## 1. 两个入口各自经过的 Route/Controller/Job 层

### 1.1 Web 创建 Invoice 路径

**Route 层**
- 路由定义: [admin.php:83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/admin.php#L83-L83)
  ```php
  Route::resource('invoices', 'Sales\Invoices', ['middleware' => ['date.format', 'money', 'dropzone']]);
  ```
- POST `/invoices` → 路由到 `Sales\Invoices@store`
- 路由中间件: `date.format`, `money`, `dropzone`

**Controller 层**
- 控制器: [Sales/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php)
- 继承链: `Invoices` → `Controller` (抽象基类)
- 构造函数: 继承自 [Controller.php:29-32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php#L29-L32)，调用 `assignPermissionsToController()` 自动分配权限中间件
- Store 方法: [Invoices.php:100-125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L100-L125)
  ```php
  public function store(Request $request)
  {
      $response = $this->ajaxDispatch(new CreateDocument($request));
      // ... 处理响应
  }
  ```
- 使用 `ajaxDispatch()` 调度 Job，会捕获异常并返回 JSON 响应

**Job 层**
- Job 类: [Jobs/Document/CreateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php)
- Handle 方法: [CreateDocument.php:20-57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L20-L57)
  - 第一步调用 `$this->authorize()` 进行计划限制检查
  - 然后在数据库事务中创建 Document、处理附件、创建条目和总计、处理递归

### 1.2 API 创建 Document 路径

**Route 层**
- 路由定义: [api.php:43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/api.php#L43-L43)
  ```php
  Route::apiResource('documents', 'Document\Documents', ['middleware' => ['date.format', 'money', 'dropzone']]);
  ```
- POST `/api/documents` → 路由到 `Api\Document\Documents@store`
- 路由中间件: `date.format`, `money`, `dropzone`

**Controller 层**
- 控制器: [Api/Document/Documents.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php)
- 继承链: `Documents` → `ApiController` (抽象基类)
- 构造函数: 继承自 [ApiController.php:24-27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/ApiController.php#L24-L27)，调用 `assignPermissionsToController()` 自动分配权限中间件
- Store 方法: [Documents.php:80-85](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L80-L85)
  ```php
  public function store(Request $request)
  {
      $document = $this->dispatch(new CreateDocument($request));
      return $this->created(route('api.documents.show', $document->id), new Resource($document));
  }
  ```
- 使用 `$this->dispatch()` 直接调度 Job，**不捕获异常**，异常会直接抛出

**Job 层**
- 与 Web 入口共用同一个 Job: [Jobs/Document/CreateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php)
- 执行逻辑完全相同

---

## 2. 权限校验和 Plan Limit 校验分别在哪一层发生，失败时表现有什么不同

### 2.1 权限校验 (Permission Check)

**发生层次**: Controller 层构造函数中，通过中间件实现

**实现机制**:
- 基类 `Controller` 和 `ApiController` 的构造函数调用 `assignPermissionsToController()`
- 该方法定义在 [Permissions.php:425-500](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L425-L500)
- 对于 Web 入口 (`Sales\Invoices` 控制器):
  - 根据控制器路径解析出权限前缀: `sales-invoices`
  - 自动分配中间件: `permission:create-sales-invoices` 作用于 `store` 方法
- 对于 API 入口 (`Api\Document\Documents` 控制器):
  - 属于通用 API 端点，会根据请求参数 `type` 动态解析权限
  - 如果 `type=invoice`，则需要 `create-sales-invoices` 权限
  - 如果 `type=bill`，则需要 `create-purchases-bills` 权限

**失败表现**:
- 权限中间件校验失败时，抛出 `Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException`
- Web 入口: 通常重定向到登录页或显示 403 页面
- API 入口: 返回 403 Forbidden JSON 响应

### 2.2 Plan Limit 校验 (计划限制检查)

**发生层次**: Job 层的 `authorize()` 方法

**实现机制**:
- 在 [CreateDocument.php:59-68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L59-L68) 中定义
  ```php
  public function authorize(): void
  {
      $limit = $this->getAnyActionLimitOfPlan();
      if (! $limit->action_status && ! empty($this->request['type']) && ($this->request['type'] == 'invoice')) {
          throw new \Exception($limit->message);
      }
  }
  ```
- 调用 [Plans.php:33-57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php#L33-L57) 的 `getAnyActionLimitOfPlan()`，依次检查:
  1. 用户限制 (`getUserLimitOfPlan()`)
  2. 公司限制 (`getCompanyLimitOfPlan()`)
  3. 发票限制 (`getInvoiceLimitOfPlan()`)

**失败表现差异**:

| 入口 | 调度方式 | 异常捕获 | 失败表现 |
|------|---------|---------|---------|
| Web | `ajaxDispatch()` | **是** | 返回 JSON: `{"success": false, "message": "限制信息"}`，前端显示错误 flash 消息，停留在创建页面 |
| API | `dispatch()` | **否** | 直接抛出 `Exception`，由 Laravel 异常处理器转为 500 Internal Server Error JSON 响应 |

---

## 3. CreateDocument::authorize 为什么只在 type == invoice 时抛出计划限制异常

### 3.1 代码逻辑分析

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

### 3.2 原因分析

**设计意图推测**:
1. **SaaS 订阅模型差异**: Akaunting 采用的定价计划中，`invoice`（销售发票）是核心付费指标，而 `bill`（采购账单）可能不被计入限制。这是常见的 SaaS 商业模式——向收款方收费，而非付款方。

2. **历史设计原因**: 该方法名为 `authorize`，但实际只做计划限制检查，且早期可能只有 invoice 需要计划限制。

3. **类型检查不完整**: 代码只检查了 `type == 'invoice'`，但实际上还有 `invoice-recurring`（递归发票）类型，这是一个潜在的设计漏洞。

4. **Plan Limit 类型**: 从 [Plans.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Plans.php) 可以看到有独立的 `getInvoiceLimitOfPlan()` 方法，说明系统设计上确实只针对 invoice 有专门的数量限制。

### 3.3 代码缺陷

该实现存在明显的不完整性：
- `getAnyActionLimitOfPlan()` 检查了 user、company、invoice 三种限制
- 但抛出异常的条件却只针对 `type == 'invoice'`
- 这意味着当 user 或 company 限制触发时，如果创建的是 bill，即使 `getAnyActionLimitOfPlan()` 返回了失败状态，也**不会**抛出异常

---

## 4. bill、invoice-recurring 或 API 传入异常 type 时哪些保护仍然有效、哪些可能依赖上游校验

### 4.1 Bill 类型 (`type == 'bill'`)

**仍然有效的保护**:
1. **权限校验**: Controller 层的权限中间件仍然有效
   - Web 入口: 需要 `create-purchases-bills` 权限
   - API 入口: 根据 type 动态解析，需要 `create-purchases-bills` 权限

2. **表单验证**: [Document.php:19-122](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L19-L122) 的验证规则仍然生效

**可能失效/依赖上游的保护**:
1. **Plan Limit 校验**: **完全失效**
   - `CreateDocument::authorize()` 中 `type == 'invoice'` 条件不满足
   - 即使 `getAnyActionLimitOfPlan()` 返回失败状态（user/company/invoice 任意限制），也不会抛出异常
   - Bill 可以无限制创建，不受计划限制约束

### 4.2 Invoice-Recurring 类型 (`type == 'invoice-recurring'`)

**仍然有效的保护**:
1. **权限校验**: Controller 层的权限中间件有效
   - Web 入口 ([RecurringInvoices.php:33-36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L33-L36)): 显式定义了 `create-sales-invoices` 权限检查
   - API 入口: 根据 type 动态解析权限

2. **表单验证**: 同样生效

**可能失效/依赖上游的保护**:
1. **Plan Limit 校验**: **完全失效**
   - `type == 'invoice-recurring'` 不等于 `'invoice'`
   - 即使计划已达到发票数量上限，仍然可以创建递归发票
   - 这是一个**严重的设计缺陷**，因为递归发票会自动生成子发票

### 4.3 API 传入异常 type（如 `type == 'fake'` 或空值）

**仍然有效的保护**:
1. **表单验证**: [Document.php:49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L49-L49) 中 `type` 字段有 `required|string` 规则，但没有验证是否为合法类型

2. **数据库层面**: `Document::create()` 可能会因未知 type 触发错误或产生脏数据

**可能失效/依赖上游的保护**:
1. **权限校验**: **可能失效**
   - [Permissions.php:435-457](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L435-L457) 中，API 入口的权限解析依赖 `type` 参数从 config 中查找配置
   - 如果 type 不合法，`config('type.document.' . $type . '.xxx')` 返回 `null`
   - 最终生成的权限名称可能不完整，导致权限检查逻辑异常

2. **Plan Limit 校验**: **失效**
   - 只要 type 不等于 `'invoice'`，就不会触发异常

---

## 5. 绕过控制器直接 dispatch Job 的测试思路

### 5.1 测试目的

验证当绕过 Controller 层的权限中间件后，Job 层是否能独立承担某些校验职责，以及哪些校验会缺失。

### 5.2 测试思路

#### 测试 1: 验证 Job 层的 Plan Limit 对不同 type 的实际保护效果

```php
// 在测试或 tinker 中执行
public function test_job_plan_limit_check_by_type()
{
    // 1. Mock Plans::getAnyActionLimitOfPlan() 返回失败状态
    $mockLimit = new \stdClass();
    $mockLimit->action_status = false;
    $mockLimit->message = 'Plan limit exceeded';
    
    // 2. 测试 type = invoice（应该抛出异常）
    try {
        $request = new \Illuminate\Http\Request();
        $request->merge([
            'type' => 'invoice',
            'document_number' => 'INV-TEST-001',
            'status' => 'draft',
            'issued_at' => now()->format('Y-m-d H:i:s'),
            'due_at' => now()->addDays(30)->format('Y-m-d H:i:s'),
            'amount' => 100,
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'contact_id' => 1,
            'contact_name' => 'Test Customer',
            'category_id' => 1,
            'items' => [[
                'name' => 'Test Item',
                'quantity' => 1,
                'price' => 100,
            ]],
        ]);
        
        $job = new \App\Jobs\Document\CreateDocument($request);
        $job->handle();
        
        $this->fail('Expected exception was not thrown for invoice type');
    } catch (\Exception $e) {
        $this->assertEquals('Plan limit exceeded', $e->getMessage());
    }
    
    // 3. 测试 type = bill（不应该抛出异常 - 验证漏洞）
    try {
        $request = new \Illuminate\Http\Request();
        $request->merge([
            'type' => 'bill',
            // ... 其他必要字段
        ]);
        
        $job = new \App\Jobs\Document\CreateDocument($request);
        $result = $job->handle();
        
        // 如果执行到这里，说明 bill 确实绕过了 plan limit 检查
        $this->assertInstanceOf(\App\Models\Document\Document::class, $result);
        $this->assertEquals('bill', $result->type);
    } catch (\Exception $e) {
        $this->fail('Bill should not be restricted by plan limit');
    }
    
    // 4. 测试 type = invoice-recurring（不应该抛出异常 - 验证漏洞）
    try {
        $request = new \Illuminate\Http\Request();
        $request->merge([
            'type' => 'invoice-recurring',
            'recurring_frequency' => 'monthly',
            'recurring_interval' => 1,
            'recurring_started_at' => now()->format('Y-m-d H:i:s'),
            // ... 其他必要字段
        ]);
        
        $job = new \App\Jobs\Document\CreateDocument($request);
        $result = $job->handle();
        
        $this->assertInstanceOf(\App\Models\Document\Document::class, $result);
        $this->assertEquals('invoice-recurring', $result->type);
    } catch (\Exception $e) {
        $this->fail('Invoice-recurring should not be restricted by plan limit');
    }
}
```

#### 测试 2: 验证权限校验是否完全依赖 Controller 层

```php
public function test_job_has_no_permission_check()
{
    // 1. 使用无权限用户登录
    $user = \App\Models\Auth\User::factory()->create();
    $user->detachAllPermissions();
    $this->actingAs($user);
    
    // 2. 直接 dispatch Job（绕过 Controller 权限中间件）
    $request = new \Illuminate\Http\Request();
    $request->merge([
        'type' => 'invoice',
        'document_number' => 'INV-NO-PERM-001',
        'status' => 'draft',
        'issued_at' => now()->format('Y-m-d H:i:s'),
        'due_at' => now()->addDays(30)->format('Y-m-d H:i:s'),
        'amount' => 100,
        'currency_code' => 'USD',
        'currency_rate' => 1,
        'contact_id' => 1,
        'contact_name' => 'Test Customer',
        'category_id' => 1,
        'items' => [[
            'name' => 'Test Item',
            'quantity' => 1,
            'price' => 100,
        ]],
    ]);
    
    // 3. 确保计划限制不触发
    \Illuminate\Support\Facades\Cache::shouldReceive('remember')
        ->andReturn((object)[
            'user' => (object)['action_status' => true],
            'company' => (object)['action_status' => true],
            'invoice' => (object)['action_status' => true],
        ]);
    
    // 4. Job 应该能成功执行（证明 Job 层不做权限检查）
    $job = new \App\Jobs\Document\CreateDocument($request);
    $result = $job->handle();
    
    $this->assertInstanceOf(\App\Models\Document\Document::class, $result);
    $this->assertEquals('invoice', $result->type);
    // 说明：权限检查完全依赖 Controller 层中间件，绕过 Controller 即可创建
}
```

#### 测试 3: 验证异常 type 的行为

```php
public function test_job_with_invalid_type()
{
    // 1. 测试 type 为空
    try {
        $request = new \Illuminate\Http\Request();
        $request->merge([
            'type' => '',
            // ... 其他字段
        ]);
        
        $job = new \App\Jobs\Document\CreateDocument($request);
        $job->authorize(); // 直接调用 authorize
        
        // 不应该抛出 plan limit 异常
        $this->assertTrue(true);
    } catch (\Exception $e) {
        // 只在 type == 'invoice' 时抛出，其他情况（包括空）不抛出
        $this->assertStringNotContainsString('Plan', $e->getMessage());
    }
    
    // 2. 测试 type 为未知值
    $request = new \Illuminate\Http\Request();
    $request->merge([
        'type' => 'fake-type',
        // ... 其他字段
    ]);
    
    $job = new \App\Jobs\Document\CreateDocument($request);
    $job->authorize(); // 不会抛出 plan limit 异常
    
    $this->assertTrue(true);
}
```

### 5.3 测试预期结论

通过以上测试可以证明：

1. **Plan Limit 检查**：
   - ✅ 对 `type == 'invoice'` 有效
   - ❌ 对 `type == 'bill'` 无效
   - ❌ 对 `type == 'invoice-recurring'` 无效
   - ❌ 对异常 type 无效

2. **权限检查**：
   - ❌ Job 层**完全没有**权限检查
   - ✅ 完全依赖 Controller 层的中间件
   - ⚠️ 绕过 Controller 直接 dispatch Job 可以绕开所有权限控制

3. **表单验证**：
   - ⚠️ 如果绕过 Controller 层的 FormRequest，Job 层也没有额外的数据校验
   - ⚠️ 可能导致脏数据写入

---

## 总结

| 校验类型 | Web Invoice | API Document (type=invoice) | API Document (type=bill) | API Document (type=invoice-recurring) | 绕过 Controller 直接 Dispatch |
|---------|------------|----------------------------|-------------------------|--------------------------------------|-------------------------------|
| 权限检查 | ✅ 中间件 | ✅ 中间件（动态） | ✅ 中间件（动态） | ✅ 中间件（动态） | ❌ 完全失效 |
| Plan Limit 检查 | ✅ Job 层 | ✅ Job 层 | ❌ 失效 | ❌ 失效 | ⚠️ 仅 invoice 有效 |
| 表单验证 | ✅ FormRequest | ✅ FormRequest | ✅ FormRequest | ✅ FormRequest | ❌ 失效 |

**核心问题**:
1. `CreateDocument::authorize()` 的 plan limit 检查只针对 `type == 'invoice'`，存在设计漏洞
2. 权限检查完全依赖 Controller 层中间件，Job 层没有冗余校验
3. 异常 type 可能导致权限解析逻辑异常
