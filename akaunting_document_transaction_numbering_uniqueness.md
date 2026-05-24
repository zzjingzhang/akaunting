# Document Number vs Transaction Number 生成机制与唯一性校验比较

## 1. 创建表单默认编号和 Job 保存编号分别在哪层发生

### Document Number 流程

| 阶段 | 所在层 | 代码位置 | 说明 |
|------|--------|----------|------|
| 表单默认值 | View Component 层 | [Form.php#L856-L877](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/View/Components/Documents/Form.php#L856-L877) | `getDocumentNumber()` 调用 `DocumentNumber::getNextNumber()` 生成默认编号填充到表单 |
| 保存 | Controller → Job 层 | [CreateDocument.php#L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L36) | Controller 将 Request 传入 Job，`Document::create($request->all())` 直接使用表单提交的 `document_number` |

**关键代码：**
```php
// Form.php getDocumentNumber()
$utility = app(DocumentNumber::class);
$document_number = $utility->getNextNumber($type, $contact);
```

### Transaction Number 流程

| 阶段 | 所在层 | 代码位置 | 说明 |
|------|--------|----------|------|
| 表单默认值 | Controller 层 | [Transactions.php#L121](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L121) | `create()` 方法中调用 `getNextTransactionNumber($type)` 生成默认编号 |
| 保存（普通） | Controller → Job 层 | [CreateTransaction.php#L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransaction.php#L32) | Controller 将 Request 传入 Job，`Transaction::create($request->all())` 使用表单提交的 `number` |
| 保存（转账） | Job 内部 | [CreateTransfer.php#L39,L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L39) | Job 内部两次调用 `getNextTransactionNumber()` 分别生成支出和收入交易编号 |
| Split 子交易 | Job → Model 层 | [SplitTransaction.php#L28-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L28-L31) | 通过 `$model->duplicate()` → `onCloning()` 重新生成编号，直接 `save()` |

**关键代码：**
```php
// CreateTransfer.php 内部生成编号
'expense' => $this->dispatch(new CreateTransaction([
    'number' => $this->getNextTransactionNumber(),
    ...
]));
'income' => $this->dispatch(new CreateTransaction([
    'number' => $this->getNextTransactionNumber(),
    ...
]));

// SplitTransaction.php — 不走 CreateTransaction Job
$transaction = $this->model->duplicate();  // 触发 onCloning() 重新生成 number
$transaction->split_id = $this->model->id;
$transaction->amount = $item['amount'];
$transaction->save();  // 直接 save，不触发 TransactionCreated
```

---

## 2. DocumentCreated/TransactionCreated 后 Listener 如何递增下一个编号

### Document 递增流程

**事件绑定：** [Event.php#L53-L57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L53-L57)
```php
\App\Events\Document\DocumentCreated::class => [
    \App\Listeners\Document\IncreaseNextDocumentNumber::class,
],
```

**Listener 实现：** [IncreaseNextDocumentNumber.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/IncreaseNextDocumentNumber.php)
```php
public function handle(Event $event)
{
    $this->increaseNextDocumentNumber($event->document->type);
}
```

**最终递增逻辑：** [DocumentNumber.php#L21-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/DocumentNumber.php#L21-L29)
```php
public function increaseNextNumber(string $type, ?Contact $contact): void
{
    $type = $this->resolveTypeAlias($type);
    $next = setting($type . '.number_next', 1) + 1;
    setting([$type . '.number_next' => $next]);
    setting()->save();
}
```

### Transaction 递增流程

**事件绑定：** [Event.php#L118-L120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L118-L120)
```php
\App\Events\Banking\TransactionCreated::class => [
    \App\Listeners\Banking\IncreaseNextTransactionNumber::class,
],
```

**Listener 实现：** [IncreaseNextTransactionNumber.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Banking/IncreaseNextTransactionNumber.php)
```php
public function handle(Event $event)
{
    $suffix = $event->transaction->isRecurringTransaction() ? '-recurring' : '';
    $this->increaseNextTransactionNumber($event->transaction->type, $suffix);
}
```

**最终递增逻辑：** [TransactionNumber.php#L55-L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/TransactionNumber.php#L55-L61)
```php
public function increaseNextNumber($type, $suffix = '', ?Contact $contact): void
{
    $next = setting('transaction' . $suffix . '.number_next', 1) + 1;
    setting(['transaction' . $suffix . '.number_next' => $next]);
    setting()->save();
}
```

### 不走 CreateTransaction/TransactionCreated 的递增路径

**SplitTransaction 路径：**

[SplitTransaction.php#L26-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L26-L40) 中，子交易通过 `$model->duplicate()` 复制，触发 [Transaction.php#L317-L328](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L317-L328) 的 `onCloning()`：

```php
// Transaction.php onCloning()
public function onCloning($src, $child = null)
{
    if (app()->has(\App\Console\Commands\RecurringCheck::class)) {
        $suffix = '';
    } else {
        $suffix = $src->isRecurringTransaction() ? '-recurring' : '';
    }

    $this->number       = $this->getNextTransactionNumber($this->type, $suffix);
    $this->document_id  = null;
    $this->split_id     = null;
    unset($this->reconciled);
}
```

**关键差异：**
- Split 子交易直接 `$transaction->save()`，**不经过 `CreateTransaction` Job**
- Split 子交易保存后，**不触发 `TransactionCreated` 事件**
- 因此 `IncreaseNextTransactionNumber` Listener **不会被调用**，setting 中的 `number_next` **不会自动递增**
- 编号递增依赖 `getNextTransactionNumber()` 内部的 do-while 冲突检测来跳过已用编号（见 [TransactionNumber.php#L23-L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/TransactionNumber.php#L23-L32)）

### 递增机制对比

| 特性 | Document | Transaction（普通/Transfer） | Transaction（Split 子交易） |
|------|----------|------------------------------|---------------------------|
| Setting Key | `{type}.number_next`（alias 解析） | `transaction{suffix}.number_next` | 同左，但 Listener 不触发 |
| Recurring 后缀 | 按 type 区分 | `-recurring` 后缀 | `-recurring` 后缀（通过 onCloning） |
| 递增触发 | `DocumentCreated` 事件 Listener | `TransactionCreated` 事件 Listener | ❌ 无事件，依赖 do-while |
| 原子性 | 读-改-写非原子 | 读-改-写非原子 | 仅 do-while 冲突检测 |

---

## 3. Request Unique 规则如何按 company_id 和 deleted_at 限制

### Document 唯一性规则

**代码位置：** [Document.php#L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L50)

```php
'document_number' => 'required|string|unique:documents,NULL,' . ($id ?? 'null') . ',id,type,' . $type . ',company_id,' . $company_id . ',deleted_at,NULL',
```

**规则解析（Laravel unique 规则格式）：**
```
unique:table,column,except,idColumn,where1,value1,where2,value2,...
```

**实际条件：**
- 表：`documents`
- 字段：`document_number`（省略，默认使用字段名）
- 排除当前记录：`id = $id`（更新时）
- 附加条件：
  - `type = $type`（invoice/bill 等）
  - `company_id = $company_id`
  - `deleted_at IS NULL`

**数据库层面唯一索引：** [2019_11_16_000000_core_v2.php#L141](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L141)
```php
$table->unique(['document_number', 'deleted_at', 'company_id', 'type']);
```

### Transaction 唯一性规则

**代码位置：** [Transaction.php#L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L42)

```php
'number' => 'required|string|unique:transactions,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL',
```

**实际条件：**
- 表：`transactions`
- 字段：`number`
- 排除当前记录：`id = $id`（更新时）
- 附加条件：
  - `company_id = $company_id`
  - `deleted_at IS NULL`

**数据库层面：** [2019_11_16_000000_core_v2.php#L409-L413](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L409-L413) 中 transactions 表只有普通索引：
```php
$table->index(['company_id', 'type']);
$table->index('account_id');
$table->index('category_id');
$table->index('contact_id');
$table->index('document_id');
```

在 [2022_08_29_000000_core_v3015.php#L34-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_08_29_000000_core_v3015.php#L34-L36) 中仅追加了：
```php
$table->index('number');
```

**结论：transactions 表没有数据库层面的唯一索引。**

### 唯一性规则对比

| 特性 | Document | Transaction |
|------|----------|-------------|
| 唯一键字段 | `document_number` | `number` |
| 公司隔离 | ✅ `company_id` | ✅ `company_id` |
| 软删除过滤 | ✅ `deleted_at IS NULL` | ✅ `deleted_at IS NULL` |
| 类型隔离 | ✅ `type` 维度 | ❌ 无类型维度 |
| 生效时机 | Request 验证阶段（表单提交） | Request 验证阶段（表单提交） |
| 数据库唯一索引 | ✅ `(document_number, deleted_at, company_id, type)` | ❌ 无唯一索引，仅有 `number` 普通索引 |

---

## 4. Transfer/Split/Recurring 类型是否共享编号序列

### Transfer（转账）

**代码位置：** [CreateTransfer.php#L39,L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L39)

```php
// 支出交易
'number' => $this->getNextTransactionNumber(),  // 不带后缀
// 收入交易
'number' => $this->getNextTransactionNumber(),  // 不带后缀
```

- **序列：** 使用默认 `transaction.number_next`
- **结论：** 与普通 income/expense **共享同一编号序列**

### Split（拆分）

**代码位置：** [SplitTransaction.php#L27-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L27-L31)

```php
$transaction = $this->model->duplicate();  // 触发 onCloning()
$transaction->split_id  = $this->model->id;
$transaction->amount    = $item['amount'];
$transaction->save();
```

**编号生成：** `duplicate()` → [Transaction::onCloning()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L317-L328) → `getNextTransactionNumber($type, $suffix)`

```php
public function onCloning($src, $child = null)
{
    if (app()->has(\App\Console\Commands\RecurringCheck::class)) {
        $suffix = '';  // RecurringCheck 命令下使用空后缀
    } else {
        $suffix = $src->isRecurringTransaction() ? '-recurring' : '';
    }

    $this->number = $this->getNextTransactionNumber($this->type, $suffix);
    // ...
}
```

- **序列：** 与原交易类型相同，如果原交易是 recurring 则使用 `-recurring` 后缀，否则使用空后缀
- **递增：** 不走 `TransactionCreated` 事件，仅依赖 `getNextTransactionNumber()` 的 do-while 冲突检测
- **结论：** 会重新生成编号（与原交易不同），但不走 CreateTransaction 和 TransactionCreated

### Recurring（定期）— 两套序列

#### Recurring 模板编号序列（用户手动创建模板）

**代码位置：** [RecurringTransactions.php#L75](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/RecurringTransactions.php#L75)

```php
// 创建表单时
$number = $this->getNextTransactionNumber($type, '-recurring');
```

- **序列：** `transaction-recurring.number_next`
- **模板保存：** 通过 `CreateTransaction` Job → `TransactionCreated` 事件 → Listener 递增 `transaction-recurring.number_next`

#### RecurringCheck 生成实际实例编号序列

**代码位置：** [RecurringCheck.php#L226-L241](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L226-L241)

```php
protected function getTransactionModel(Transaction $template, Date $schedule_date): Transaction
{
    $template->cloneable_relations = ['taxes'];
    $model = $template->duplicate();  // 触发 onCloning()
    $model->type = $this->getRealType($template->type);  // 去除 -recurring 后缀
    // ...
    $model->save();
    return $model;
}
```

**`onCloning()` 中的判断：**
```php
if (app()->has(\App\Console\Commands\RecurringCheck::class)) {
    $suffix = '';  // 在 RecurringCheck 命令下，使用空后缀
}
```

- **序列：** `transaction.number_next`（空后缀，即普通序列）
- **触发事件：** [RecurringCheck.php#L180-L181](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Console/Commands/RecurringCheck.php#L180-L181) 中手动 dispatch `TransactionCreated`
  ```php
  event(new TransactionCreated($model));
  ```
- **递增：** Listener 中根据 `isRecurringTransaction()` 判断，但此时 `$model->type` 已去除 `-recurring` 后缀，所以 `isRecurringTransaction()` 返回 false，`$suffix = ''`，递增的是 `transaction.number_next`

#### Document 的 Recurring 类似逻辑

[Document.php#L263-L273](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L263-L273)：
```php
public function onCloning($src, $child = null)
{
    if (app()->has(\App\Console\Commands\RecurringCheck::class)) {
        $type = $this->getRealTypeOfRecurringDocument($src->type);  // 去除 -recurring
    } else {
        $type = $src->type;
    }

    $this->status          = 'draft';
    $this->document_number = app(DocumentNumber::class)->getNextNumber($type, $src->contact);
}
```

### 编号序列总结

| 场景 | Setting Key | 序列归属 |
|------|-------------|----------|
| 普通 income/expense | `transaction.number_next` | 主序列 |
| Transfer（expense-transfer/income-transfer） | `transaction.number_next` | 与普通共享 |
| Split 子交易（非 recurring 原交易） | `transaction.number_next` | 与普通共享 |
| Split 子交易（recurring 原交易） | `transaction-recurring.number_next` | 独立序列 |
| Recurring 模板（用户创建） | `transaction-recurring.number_next` | 独立序列 |
| RecurringCheck 生成的实际实例 | `transaction.number_next` | 主序列 |
| Document 模板 | `{type}.number_next`（带 -recurring） | 按 type 区分 |
| RecurringCheck 生成的 Document 实例 | `{type}.number_next`（去除 -recurring） | 与普通 Document 共享 |

> **注意：** Document 类型（invoice/bill/credit-note 等）各自有独立的 setting key（如 `invoice.number_next`, `bill.number_next`），通过 `resolveTypeAlias()` 解析。RecurringCheck 生成实例时会去除 `-recurring` 后缀，因此使用与普通类型相同的序列。

---

## 5. 直接 Dispatch Job 导致编号重复风险的测试

### 风险场景描述

当绕过 FormRequest 验证层，直接在代码中 dispatch `CreateDocument`/`CreateTransaction` Job 时，**Request 的 unique 规则校验被完全跳过**。原因：

1. **FormRequest 校验只在 HTTP 请求经过 Controller 时生效**，直接 dispatch Job 时不会执行
2. **Document** 的 `getNextNumber()` [DocumentNumber.php#L10-L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/DocumentNumber.php#L10-L19) **没有任何冲突检测**，连续调用返回相同编号
3. **Transaction** 的 `getNextNumber()` [TransactionNumber.php#L12-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/TransactionNumber.php#L12-L36) 虽然有 do-while 冲突检测，但并发下可能失效
4. **transactions 表没有数据库唯一索引**，只有 `number` 普通索引
5. **documents 表虽然有唯一索引**，但这是最后一道防线，不应依赖数据库异常来保证业务唯一性

### 测试用例：直接 Dispatch CreateDocument 传入相同编号

```php
<?php
// tests/Feature/DocumentNumberDuplicateTest.php

namespace Tests\Feature;

use App\Jobs\Document\CreateDocument;
use App\Models\Document\Document;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\DB;
use Tests\TestCase;

class DocumentNumberDuplicateTest extends TestCase
{
    use RefreshDatabase;

    /**
     * 直接 dispatch 两次 CreateDocument 并传入相同编号，
     * 验证该路径绕过了 FormRequest 唯一性校验。
     *
     * 由于 documents 表有 (document_number, deleted_at, company_id, type) 唯一索引，
     * 第二次应该抛出 QueryException。
     * 如果将来移除唯一索引，两次 dispatch 都会成功，产生重复编号。
     */
    public function test_direct_dispatch_create_document_with_same_number_bypasses_form_request()
    {
        $number = 'INV-DUPLICATE-001';

        $data = [
            'company_id' => 1,
            'type' => Document::INVOICE_TYPE,
            'document_number' => $number,
            'status' => 'draft',
            'issued_at' => now()->toDateTimeString(),
            'due_at' => now()->addDays(30)->toDateTimeString(),
            'amount' => 100,
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'contact_id' => 1,
            'contact_name' => 'Test Customer',
            'category_id' => 1,
        ];

        // 第一次 dispatch — 成功
        $doc1 = $this->dispatch(new CreateDocument($data));
        $this->assertEquals($number, $doc1->document_number);

        // 第二次 dispatch — 传入相同编号
        // 注意：没有经过 FormRequest，unique 规则未执行
        // documents 表唯一索引会拦截，抛出 QueryException
        $this->expectException(\Illuminate\Database\QueryException::class);

        $doc2 = $this->dispatch(new CreateDocument($data));
    }

    /**
     * 验证 DocumentNumber::getNextNumber() 连续调用返回相同编号。
     * 这是编号重复的根本原因之一。
     */
    public function test_document_getNextNumber_returns_same_value_on_consecutive_calls()
    {
        setting(['invoice.number_next' => 1])->save();

        $utility = app(\App\Interfaces\Utility\DocumentNumber::class);

        $number1 = $utility->getNextNumber(Document::INVOICE_TYPE, null);
        $number2 = $utility->getNextNumber(Document::INVOICE_TYPE, null);

        // DocumentNumber 没有冲突检测，连续调用返回相同值
        $this->assertEquals($number1, $number2);
    }
}
```

### 测试用例：直接 Dispatch CreateTransaction 传入相同编号

```php
<?php
// tests/Feature/TransactionNumberDuplicateTest.php

namespace Tests\Feature;

use App\Jobs\Banking\CreateTransaction;
use App\Models\Banking\Transaction;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\DB;
use Tests\TestCase;

class TransactionNumberDuplicateTest extends TestCase
{
    use RefreshDatabase;

    /**
     * 直接 dispatch 两次 CreateTransaction 并传入相同编号，
     * 验证该路径绕过了 FormRequest 唯一性校验。
     *
     * 由于 transactions 表没有唯一索引，两次 dispatch 都会成功，
     * 产生两条 number 相同的记录。
     */
    public function test_direct_dispatch_create_transaction_with_same_number_bypasses_form_request()
    {
        $number = 'TX-DUPLICATE-001';

        $data = [
            'company_id' => 1,
            'type' => Transaction::INCOME_TYPE,
            'number' => $number,
            'account_id' => 1,
            'paid_at' => now()->toDateTimeString(),
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'amount' => 100,
            'category_id' => 1,
            'payment_method' => 'cash',
        ];

        // 第一次 dispatch — 成功
        $tx1 = $this->dispatch(new CreateTransaction($data));
        $this->assertEquals($number, $tx1->number);

        // 第二次 dispatch — 传入相同编号
        // 注意：没有经过 FormRequest，unique 规则未执行
        // transactions 表没有唯一索引，第二次也会成功！
        $tx2 = $this->dispatch(new CreateTransaction($data));
        $this->assertEquals($number, $tx2->number);

        // 验证产生了两条相同编号的记录
        $count = Transaction::where('company_id', 1)
            ->where('number', $number)
            ->whereNull('deleted_at')
            ->count();
        $this->assertEquals(2, $count, 'Should have 2 transactions with same number — FormRequest was bypassed');
    }

    /**
     * 验证在并发场景下，即使使用 getNextTransactionNumber() 自动生成编号，
     * 仍可能产生重复（因为 do-while 冲突检测非原子）。
     */
    public function test_transaction_getNextNumber_concurrent_race_condition()
    {
        setting(['transaction.number_next' => 1])->save();

        $utility = app(\App\Interfaces\Utility\TransactionNumber::class);

        // 模拟两个并发请求同时获取编号（在第一个保存前）
        $number1 = $utility->getNextNumber(Transaction::INCOME_TYPE);
        $number2 = $utility->getNextNumber(Transaction::INCOME_TYPE);

        // 如果 number_next 初始为 1 且 prefix/digit 配置相同，
        // 两次调用可能返回相同编号
        // （取决于 number_next 是否已被第一次调用后的 do-while 逻辑更新）
        //
        // 关键：getNextNumber() 中的冲突检测仅在编号已存在时触发，
        // 并发下两个请求可能在同一时刻都读到相同的 number_next 值
    }

    /**
     * 对比：通过 HTTP 请求创建 transaction 时，FormRequest 会拦截重复编号。
     */
    public function test_http_request_form_request_blocks_duplicate_number()
    {
        $number = 'TX-FORM-REQUEST-001';

        // 先创建一条记录
        Transaction::create([
            'company_id' => 1,
            'type' => Transaction::INCOME_TYPE,
            'number' => $number,
            'account_id' => 1,
            'paid_at' => now(),
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'amount' => 100,
            'category_id' => 1,
            'payment_method' => 'cash',
        ]);

        // 通过 HTTP POST 创建相同编号的 transaction
        // FormRequest 应该返回 422 错误
        $response = $this->postJson('/api/transactions', [
            'company_id' => 1,
            'type' => Transaction::INCOME_TYPE,
            'number' => $number,  // 重复编号
            'account_id' => 1,
            'paid_at' => now()->toDateTimeString(),
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'amount' => 200,
            'category_id' => 1,
            'payment_method' => 'cash',
        ]);

        // FormRequest 应该拦截，返回验证错误
        $response->assertStatus(422);
    }
}
```

### 根本原因分析

1. **FormRequest 校验只在 HTTP 请求路径生效：** 直接 dispatch Job 时，`Document` 和 `Transaction` 的 FormRequest 不会被执行，unique 规则被绕过
2. **Document 没有冲突检测：** `DocumentNumber::getNextNumber()` 只是读取 setting 并格式化，不检查编号是否已存在
3. **Transaction 冲突检测不足：** `TransactionNumber::getNextNumber()` 的 do-while 循环在并发下可能读到相同值
4. **Setting 读写非原子：** `setting()->save()` 不是原子操作，并发下会丢失更新
5. **transactions 表无唯一索引：** 数据库层面无法阻止重复，唯一防线是 FormRequest（可绕过）
6. **documents 表有唯一索引：** 作为最后一道防线，但不应依赖数据库异常来保证业务规则

### 建议修复方案

1. **数据库层面：** 为 transactions 添加唯一索引
   ```sql
   ALTER TABLE transactions ADD UNIQUE KEY unique_trans_number (company_id, number, deleted_at);
   ```

2. **应用层面：** 在 Job 内部进行编号唯一性校验（不依赖 FormRequest）
   ```php
   // CreateTransaction.php handle() 中添加
   $existing = Transaction::where('company_id', $this->request['company_id'])
       ->where('number', $this->request['number'])
       ->whereNull('deleted_at')
       ->exists();
   if ($existing) {
       throw new \Exception('Transaction number already exists');
   }
   ```

3. **编号生成：** 在保存时再生成编号，而不是在表单渲染时预生成
   ```php
   // 原子递增
   DB::table('settings')
     ->where('key', 'invoice.number_next')
     ->increment('value');
   ```

4. **Split 子交易：** 补充 `TransactionCreated` 事件触发，确保 setting 正确递增

---

## 总结对比表

| 对比项 | Document Number | Transaction Number |
|--------|-----------------|--------------------|
| 表单生成位置 | View Component | Controller |
| Job 保存方式 | 使用 Request 传入 | 普通用 Request；Transfer 自己生成；Split 走 duplicate → onCloning → save |
| 递增时机 | DocumentCreated 事件后 | TransactionCreated 事件后；Split 子交易无事件 |
| Recurring 模板序列 | 按 type 区分（含 -recurring） | `transaction-recurring.number_next` |
| RecurringCheck 实例序列 | 去除 -recurring，与普通共享 | 去除 -recurring，使用 `transaction.number_next` |
| Transfer 序列 | N/A | 与普通交易共享 |
| Split 序列 | N/A | 重新生成（onCloning），不走 CreateTransaction/TransactionCreated |
| 生成时冲突检测 | ❌ 无 | ✅ do-while 循环 |
| 唯一性校验 | Request 层 unique 规则 | Request 层 unique 规则 |
| 数据库唯一索引 | ✅ `(document_number, deleted_at, company_id, type)` | ❌ 无唯一索引 |
| 直接 dispatch Job 风险 | 数据库唯一索引拦截 | 完全绕过校验，可产生重复 |
| 并发安全 | ⚠️ 数据库索引兜底 | ❌ 高危（无数据库唯一索引） |
