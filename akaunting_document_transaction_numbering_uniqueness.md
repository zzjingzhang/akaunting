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

### 递增机制对比

| 特性 | Document | Transaction |
|------|----------|-------------|
| Setting Key | `{type}.number_next`（支持 alias 解析） | `transaction{suffix}.number_next` |
| Recurring 后缀 | 无特殊处理（按 type 区分） | 自动追加 `-recurring` 后缀 |
| 原子性 | 读-改-写非原子（并发风险） | 读-改-写非原子（并发风险） |

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

### 唯一性规则对比

| 特性 | Document | Transaction |
|------|----------|-------------|
| 唯一键字段 | `document_number` | `number` |
| 公司隔离 | ✅ `company_id` | ✅ `company_id` |
| 软删除过滤 | ✅ `deleted_at IS NULL` | ✅ `deleted_at IS NULL` |
| 类型隔离 | ✅ `type` 维度 | ❌ 无类型维度 |
| 生效时机 | Request 验证阶段（表单提交） | Request 验证阶段（表单提交） |

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

**代码位置：** [SplitTransaction.php#L26-L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L26-L39)

```php
foreach ($this->request->items as $item) {
    $transaction = $this->model->duplicate();  // 复制原交易
    $transaction->split_id = $this->model->id;
    $transaction->amount = $item['amount'];
    $transaction->save();  // 保留原有 number
}
$this->model->type = config('type.transaction.' . $this->model->type . '.split_type');
```

- **序列：** 通过 `duplicate()` 复制，**保留原交易编号**
- **结论：** 不生成新编号，不涉及序列

### Recurring（定期）

**代码位置：** [IncreaseNextTransactionNumber.php#L20](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Banking/IncreaseNextTransactionNumber.php#L20)

```php
$suffix = $event->transaction->isRecurringTransaction() ? '-recurring' : '';
$this->increaseNextTransactionNumber($event->transaction->type, $suffix);
```

**Setting Key：** `transaction-recurring.number_next`

- **序列：** 使用独立的 `transaction-recurring.*` setting
- **结论：** 与普通交易 **使用独立编号序列**

### 编号序列总结

| 类型 | Setting Key | 序列归属 |
|------|-------------|----------|
| 普通 income/expense | `transaction.number_next` | 主序列 |
| Transfer（expense-transfer/income-transfer） | `transaction.number_next` | 与普通共享 |
| Split | 不生成新编号 | 复用原编号 |
| Recurring（income-recurring/expense-recurring） | `transaction-recurring.number_next` | 独立序列 |

> **注意：** Document 类型（invoice/bill/credit-note 等）各自有独立的 setting key（如 `invoice.number_next`, `bill.number_next`），通过 `resolveTypeAlias()` 解析。

---

## 5. 直接 Dispatch Job 导致编号重复风险的测试

### 风险场景描述

当绕过 Request 验证层，直接在代码中 dispatch CreateTransaction/CreateDocument Job 时，存在编号重复风险。原因：

1. **Transaction** 的 `getNextNumber()` [TransactionNumber.php#L12-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/TransactionNumber.php#L12-L36) 虽然有冲突检测（do-while 循环），但在高并发下仍可能出现竞态条件
2. **Document** 的 `getNextNumber()` [DocumentNumber.php#L10-L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/DocumentNumber.php#L10-L19) **没有任何冲突检测**，直接使用 setting 中的值

### 测试用例（Transaction 并发场景）

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

    public function test_concurrent_job_dispatch_causes_duplicate_number()
    {
        // 初始化 setting
        setting(['transaction.number_next' => 1])->save();

        // 模拟并发：同时 dispatch 10 个 CreateTransaction Job
        $promises = [];
        for ($i = 0; $i < 10; $i++) {
            $promises[] = \Illuminate\Support\Facades\Bus::dispatch(new CreateTransaction([
                'company_id' => 1,
                'type' => Transaction::INCOME_TYPE,
                'number' => app(\App\Interfaces\Utility\TransactionNumber::class)->getNextNumber(Transaction::INCOME_TYPE),
                'account_id' => 1,
                'paid_at' => now(),
                'currency_code' => 'USD',
                'currency_rate' => 1,
                'amount' => 100,
                'category_id' => 1,
                'payment_method' => 'cash',
            ]));
        }

        // 等待所有 Job 完成
        foreach ($promises as $promise) {
            $promise->wait();
        }

        // 检查是否有重复编号
        $duplicates = Transaction::query()
            ->select('number', DB::raw('COUNT(*) as count'))
            ->groupBy('number')
            ->having('count', '>', 1)
            ->get();

        $this->assertEmpty(
            $duplicates,
            'Found duplicate transaction numbers: ' . $duplicates->pluck('number')->implode(', ')
        );
    }

    public function test_bypass_request_validation_allows_duplicate()
    {
        // 直接创建两个相同编号的 transaction（绕过 Request 验证）
        $transaction1 = Transaction::create([
            'company_id' => 1,
            'type' => Transaction::INCOME_TYPE,
            'number' => 'INV-00001',
            'account_id' => 1,
            'paid_at' => now(),
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'amount' => 100,
            'category_id' => 1,
            'payment_method' => 'cash',
        ]);

        // 由于没有数据库层面的唯一索引，第二条可以成功插入
        $transaction2 = Transaction::create([
            'company_id' => 1,
            'type' => Transaction::INCOME_TYPE,
            'number' => 'INV-00001',  // 相同编号
            'account_id' => 1,
            'paid_at' => now(),
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'amount' => 200,
            'category_id' => 1,
            'payment_method' => 'cash',
        ]);

        $this->assertNotFalse($transaction2, 'Should allow duplicate when bypassing validation');
        $this->assertEquals($transaction1->number, $transaction2->number, 'Numbers should be duplicate');
    }
}
```

### 测试用例（Document 并发场景）

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

    public function test_document_getNextNumber_has_no_conflict_detection()
    {
        // 初始化 setting
        setting(['invoice.number_next' => 1])->save();

        $utility = app(\App\Interfaces\Utility\DocumentNumber::class);

        // 连续调用两次 getNextNumber，会得到相同编号
        $number1 = $utility->getNextNumber(Document::INVOICE_TYPE, null);
        $number2 = $utility->getNextNumber(Document::INVOICE_TYPE, null);

        $this->assertEquals(
            $number1,
            $number2,
            'DocumentNumber::getNextNumber() returns same number on multiple calls before increment'
        );
    }

    public function test_concurrent_document_creation_causes_duplicate()
    {
        // 模拟两个并发请求，同时获取编号后保存
        setting(['invoice.number_next' => 1])->save();

        $utility = app(\App\Interfaces\Utility\DocumentNumber::class);

        // 两个请求同时获取编号
        $number1 = $utility->getNextNumber(Document::INVOICE_TYPE, null);
        $number2 = $utility->getNextNumber(Document::INVOICE_TYPE, null);

        // 第一个请求保存并触发 listener 递增
        $doc1 = Document::create([
            'company_id' => 1,
            'type' => Document::INVOICE_TYPE,
            'document_number' => $number1,
            'status' => 'draft',
            'issued_at' => now(),
            'due_at' => now()->addDays(30),
            'amount' => 100,
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'contact_id' => 1,
            'contact_name' => 'Test Customer',
            'category_id' => 1,
        ]);
        event(new \App\Events\Document\DocumentCreated($doc1, request()));

        // 第二个请求也保存（使用相同编号）
        $doc2 = Document::create([
            'company_id' => 1,
            'type' => Document::INVOICE_TYPE,
            'document_number' => $number2,  // 与 doc1 相同
            'status' => 'draft',
            'issued_at' => now(),
            'due_at' => now()->addDays(30),
            'amount' => 200,
            'currency_code' => 'USD',
            'currency_rate' => 1,
            'contact_id' => 1,
            'contact_name' => 'Test Customer',
            'category_id' => 1,
        ]);

        $this->assertEquals($doc1->document_number, $doc2->document_number);
    }
}
```

### 根本原因分析

1. **Document 没有冲突检测：** `DocumentNumber::getNextNumber()` 只是读取 setting 并格式化，不检查编号是否已存在
2. **Transaction 冲突检测不足：** `TransactionNumber::getNextNumber()` 的 do-while 循环在并发下可能读到相同值
3. **Setting 读写非原子：** `setting()->save()` 不是原子操作，并发下会丢失更新
4. **无数据库唯一索引：** `documents` 和 `transactions` 表没有 `(company_id, number, deleted_at)` 组合唯一索引，Request 验证是唯一防线

### 建议修复方案

1. **数据库层面：** 添加唯一索引
   ```sql
   ALTER TABLE documents ADD UNIQUE KEY unique_doc_number (company_id, type, document_number, deleted_at);
   ALTER TABLE transactions ADD UNIQUE KEY unique_trans_number (company_id, number, deleted_at);
   ```

2. **应用层面：** 使用数据库锁或原子递增
   ```php
   // 使用查询构建器的原子递增
   DB::table('settings')
     ->where('key', 'invoice.number_next')
     ->increment('value');
   ```

3. **编号生成：** 在保存时再生成编号，而不是在表单渲染时预生成

---

## 总结对比表

| 对比项 | Document Number | Transaction Number |
|--------|-----------------|--------------------|
| 表单生成位置 | View Component | Controller |
| Job 保存方式 | 使用 Request 传入 | 普通用 Request，Transfer 自己生成 |
| 递增时机 | DocumentCreated 事件后 | TransactionCreated 事件后 |
| Recurring 序列 | 按 type 区分（如 invoice-recurring） | 独立 setting（-recurring 后缀） |
| Transfer 序列 | N/A | 与普通交易共享 |
| Split 序列 | N/A | 复用原编号 |
| 生成时冲突检测 | ❌ 无 | ✅ do-while 循环 |
| 唯一性校验 | Request 层 unique 规则 | Request 层 unique 规则 |
| 数据库唯一索引 | ❌ 无 | ❌ 无 |
| 并发安全 | ❌ 高危 | ⚠️ 中等风险 |
