# Akaunting 删除 Banking Transaction 后 Invoice/Bill 与 Split 父交易更新分析（二次修订版）

## 核心代码位置

主逻辑位于 [app/Observers/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php) 的 `deleted()` 方法。

---

## 1. deleted observer 如何根据 transaction type 找到 invoice 或 bill

### 代码片段

```php
// [Transaction.php#L23-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L23-L29)
public function deleted(Model $transaction)
{
    if (! empty($transaction->document_id)) {
        $type = ($transaction->type == 'income') ? Document::INVOICE_TYPE : Document::BILL_TYPE;
        $this->updateDocument($transaction, $type);
    }
    // ...
}
```

### 查找逻辑

1. **判断关联存在**：首先检查 `$transaction->document_id` 是否非空，确定该交易关联了某个文档。
2. **类型映射（基于 transaction type）**：
   - 若 `transaction->type == 'income'` → `$type = Document::INVOICE_TYPE`（值为 `'invoice'`）
   - 否则 → `$type = Document::BILL_TYPE`（值为 `'bill'`）
3. **动态属性关联**：在 `updateDocument()` 中通过 `$transaction->{$type}` 利用 Eloquent 动态属性获取关联文档。

### invoice() 和 bill() 不按 document type 过滤

查看 [Transaction.php#L111-L143](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L111-L143)：

```php
public function bill()
{
    return $this->belongsTo('App\Models\Document\Document', 'document_id')
        ->withoutGlobalScope('App\Scopes\Document');
}

public function invoice()
{
    return $this->belongsTo('App\Models\Document\Document', 'document_id')
        ->withoutGlobalScope('App\Scopes\Document');
}
```

**两者完全相同**：都只是通过 `document_id` 外键关联到 `App\Models\Document\Document`，**不做任何 type 过滤**。`withoutGlobalScope('App\Scopes\Document')` 仅用于移除默认的 `notRecurring` scope（参见 [app/Scopes/Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Document.php)），与 invoice/bill 类型区分无关。

**实际含义**：`$transaction->invoice` 和 `$transaction->bill` 返回的是同一个 Document 记录（通过同一个 `document_id`）。Observer 根据 transaction 的 `type`（income/expense）选择调用哪个方法，但这只是语义上的区分。

**潜在风险**：如果一个 expense 类型的 transaction 被错误地关联到了一个 invoice（`document_id` 指向 type=invoice 的记录），`$transaction->bill` 仍然能查到这条 invoice 记录，因为没有 type 过滤。

---

## 2. transactions_count、TransactionsCounted 事件与 sent/received/partial 状态决策

### 代码片段

```php
// [Transaction.php#L36-L59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L36-L59)
protected function updateDocument($transaction, $type)
{
    $document = $transaction->{$type};
    if (empty($document)) {
        return;
    }

    $document->transactions_count = $document->transactions->count();  // ①

    event(new TransactionsCounted($document));                         // ②

    $document->status = ($type == Document::INVOICE_TYPE) ? 'sent' : 'received';  // ③

    if ($document->transactions_count > 0) {                           // ④
        $document->status = 'partial';
    }

    unset($document->transactions_count);                              // ⑤
    $document->save();
    // ...
}
```

### 执行流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| ① | 计算 `transactions_count` | 在 `$document` 模型实例上设置临时属性，值为当前剩余关联交易数量 |
| ② | 触发 `TransactionsCounted` 事件 | 将 `$document`（含临时属性）传给事件，事件在状态判断**之前**触发 |
| ③ | 设置默认状态 | Invoice 默认 `'sent'`，Bill 默认 `'received'` |
| ④ | 覆盖为 `'partial'` | 若 `transactions_count > 0`，覆盖为 `'partial'` |
| ⑤ | 清理临时属性 | `transactions_count` 不是数据库字段，unset 避免保存时报错 |

### TransactionsCounted 事件的影响

`TransactionsCounted` 事件类位于 [app/Events/Document/TransactionsCounted.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Document/TransactionsCounted.php)，接收 `$document` 模型作为公共属性。

在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L14-L132) 的 `$listen` 数组中，**没有注册任何默认监听器**。但该事件在状态判断步骤 ③ 和 ④ **之前**触发，且携带的 `$document` 对象包含步骤 ① 设置的 `transactions_count` 临时属性。

**扩展模块可通过注册 listener 读取或修改 `$document->transactions_count`**，从而影响后续的 `if ($document->transactions_count > 0)` 判断。因此不能说它"完全不会影响"——虽然核心代码没有 listener，但事件的设计意图就是作为扩展钩子。

### 状态决策表

| 场景 | transactions_count | 最终 status |
|------|--------------------|-------------|
| 删除最后一笔付款 | 0 | Invoice: `sent` / Bill: `received` |
| 删除其中一笔付款，仍有剩余 | > 0 | `partial` |

---

## 3. 删除逻辑不恢复 paid 状态 —— 与创建/更新逻辑的源码对比

### 创建付款的状态判断

[CreateBankingDocumentTransaction.php#L74-L123](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123) 中 `checkAmount()` 方法：

```php
// paid_amount = 已付金额（遍历所有 transactions 求和）
$this->model->paid_amount = $this->model->paid;
event(new PaidAmountCalculated($this->model));

// total_amount = 文档总额 - 已付金额 = 剩余待付金额
$total_amount = round($this->model->amount - $this->model->paid_amount, $document_precision);

// 比较本次付款金额与剩余待付金额
$compare = bccomp($amount, $total_amount, $document_precision);

if ($compare === 1) {
    // 超额付款 → 抛出 over_payment 异常
    throw new \Exception(trans('messages.error.over_payment', ...));
} else {
    // compare === 0: 恰好付清 → paid
    // compare === -1: 部分付款 → partial
    $this->model->status = ($compare === 0) ? 'paid' : 'partial';
}
```

**判断依据**：比较 `本次付款金额` 与 `剩余待付金额`，精确到货币精度。

### 更新付款的状态判断

[UpdateBankingDocumentTransaction.php#L75-L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateBankingDocumentTransaction.php#L75-L119) 逻辑相同，但计算 `paid_amount` 时会先减去当前编辑交易的金额：

```php
$this->model->paid_amount = ($this->model->paid - $this->transaction->amount_for_document);
```

### 匹配交易的状态判断

[MatchBankingDocumentTransaction.php#L43-L86](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L43-L86) 逻辑相同。

### 删除付款的状态判断

[Transaction.php#L44-L56](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L44-L56)：

```php
$document->transactions_count = $document->transactions->count();
event(new TransactionsCounted($document));
$document->status = ($type == Document::INVOICE_TYPE) ? 'sent' : 'received';
if ($document->transactions_count > 0) {
    $document->status = 'partial';
}
```

**判断依据**：仅统计剩余交易**数量**，不比较金额。

### 对比总结

| 维度 | 创建/更新/匹配付款 | 删除付款 |
|------|-------------------|---------|
| 判断依据 | `bccomp(amount, total_amount, precision)` 比较金额 | `transactions->count() > 0` 比较数量 |
| 可设置状态 | `paid` / `partial` | `sent` / `received` / `partial` |
| 能否设置 `paid` | ✅ 当剩余金额恰好等于本次付款金额 | ❌ 永远不会 |
| 超额处理 | 抛出 `over_payment` 异常 | 不检查 |

### 删除最后一笔 vs 删除其中一笔

| 维度 | 删除最后一笔 | 删除其中一笔 |
|------|-------------|-------------|
| `transactions_count` | 0 | > 0 |
| 最终 status | `sent` / `received` | `partial` |
| `getPaidAttribute()` 返回值 | 0 | 剩余交易金额总和 |

### 关键缺陷

代码永远不会把状态恢复为 `'paid'`。如果出现以下情形（非正常入口可达，见第 5 节）：

- Invoice 总额 $50，存在一笔 $50 付款，状态为 `paid`
- 通过直接构造模型绕过校验，插入一笔 $0 交易（`document_id` 指向该 invoice）
- 删除 $50 付款 → 剩余 $0 交易 → count=1 → 状态 `partial`
- `getPaidAttribute()` 实际返回 $0.00，但状态显示 `partial`

---

## 4. split_id 分支何时把父交易 type 改回子交易 type

### 代码片段

```php
// [Transaction.php#L68-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L68-L79)
protected function updateTransaction($transaction)
{
    $splitted_transaction = Model::find($transaction->split_id);

    if ($splitted_transaction->splits->count() == 0) {
        $values = [
            'type' => $transaction->type,
        ];

        $this->dispatch(new UpdateTransaction($splitted_transaction, $values));
    }
}
```

### Observer 触发条件

Observer 在以下条件**同时满足**时**尝试**恢复父交易 type：

1. 被删交易有 `split_id`（是某个 split 父交易的子交易）。
2. 父交易的 `splits` 关联查询结果数量为 0（该子交易是最后一个，删除后没有其他 split 了）。

### 实际恢复成功的额外条件

Observer 通过 `dispatch(new UpdateTransaction(...))` 同步执行更新。`UpdateTransaction` 在 [UpdateTransaction.php#L65-L76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransaction.php#L65-L76) 中有 `authorize()` 检查：

```php
public function authorize(): void
{
    if ($this->model->reconciled) {
        throw new \Exception(trans('messages.warning.reconciled_tran'));
    }

    if ($this->model->isTransferTransaction()) {
        throw new \Exception(trans('messages.error.transfer_transaction'));
    }
}
```

**因此父交易 type 实际恢复成功需要额外满足：**

- 父交易不能是 `reconciled`（已对账）
- 父交易不能是 transfer 类型

### 异常传播的影响

`DeleteTransaction` job 在 [DeleteTransaction.php#L18-L22](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/DeleteTransaction.php#L18-L22) 中使用 DB 事务：

```php
\DB::transaction(function () {
    $this->deleteRelationships($this->model, ['recurring', 'taxes']);
    $this->model->delete();  // Observer 的 deleted() 在此触发
});
```

Observer 的 `deleted()` 在事务内同步执行。如果 `UpdateTransaction::authorize()` 抛出异常，异常会传播到 DB 事务，导致**事务回滚**——即被删的子交易也会被回滚（恢复）。

### split 父交易实际 type 名称

在 [SplitTransaction.php#L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L38) 中：

```php
$this->model->type = config('type.transaction.' . $this->model->type . '.split_type');
```

根据 [config/type.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php) 配置和 [Transaction.php#L22-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L22-L29) 常量：

- `income` → `income-split`（`Transaction::INCOME_SPLIT_TYPE`）
- `expense` → `expense-split`（`Transaction::EXPENSE_SPLIT_TYPE`）

当所有子 split 都被删除且 `UpdateTransaction` 成功执行后，父交易的 type 从 `income-split`/`expense-split` 恢复为子交易的原始 type（通常是 `income` 或 `expense`）。

---

## 5. 删除后状态与剩余金额直觉不一致的边界场景

### 场景：零金额交易导致 partial 状态与实际已付金额不符

| 项目 | 值 |
|------|----|
| Invoice 总额 | $100.00 |
| 交易 1（正常付款） | $100.00 |
| 交易 2（零金额交易） | $0.00 |
| 当前状态 | `paid` |

删除交易 1（$100.00 的付款）后：

1. `transactions_count` = 1（还剩 $0.00 的交易）
2. 因 `count > 0`，状态被设置为 `partial`
3. `$document->paid`（通过 [getPaidAttribute()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407) 计算）实际返回 `0.00`

- **状态显示**：`partial`（暗示部分付款）
- **实际已付**：$0.00（完全未付）
- **用户直觉**：看到 `partial` 会以为还有未结清的付款

### 零金额交易在各入口的可达性

#### 入口 1：UI 模态框路径（DocumentTransactions::store）

在 [DocumentTransactions.php#L43-L117](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Modals/DocumentTransactions.php#L43-L117) 的 `create()` 方法中：

```php
$paid = $document->paid;
// ...
$d_total = $document?->total ?? 0;
$document->grand_total = money($total, $currency->code, false)->getAmount();
if (! empty($paid)) {
    $document->grand_total = round($d_total - $paid, $currency->precision);
}
$amount = $document->grand_total;
```

模态框预填充的金额 = `总额 - 已付金额`。如果已完全付清，预填充金额为 $0.00。

然后 [CreateBankingDocumentTransaction.php#L52-L58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L52-L58) 的 `prepareRequest()`：

```php
if (!isset($this->request['amount'])) {
    $this->request['amount'] = $this->model->amount - $this->model->paid_amount;
}
```

UI 模态框路径会提交 `amount` 字段（预填充的值），所以 `isset` 为 true，使用用户提交的值。

在 [Transaction.php#L45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L45) 中，校验规则为：

```php
'amount' => 'required|amount:0',
```

[Validation.php#L69-L81](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Validation.php#L69-L81) 中 `amount` 规则：

```php
if ($value > 0 || in_array($value, $parameters)) {
    $status = true;
}
```

规则 `amount:0` 表示 `$parameters = [0]`，条件 `$value > 0 || $value == 0` 允许 $0 通过。

**但 `checkAmount()` 会进一步检查**：如果已完全付清，`total_amount = 0`，提交 `amount = 0` 时 `bccomp(0, 0) === 0` → 状态设为 `paid`。此时创建的 $0 交易不会导致 `partial` 状态（因为 document 状态会被设为 `paid`）。

**因此在 UI 路径下，零金额交易不会产生不一致的场景**——因为创建时 `checkAmount` 会把 document 状态设为 `paid`，后续删除 $100 付款后剩余 $0 交易才会出现不一致。但用户无法在 UI 上对已 `paid` 的 document 再添加付款（blade 模板中 `$document->status != 'paid'` 条件阻止）。

#### 入口 2：API 路径

[Api/Document/DocumentTransactions.php#L66-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/DocumentTransactions.php#L66-L73) 使用相同的 `CreateBankingDocumentTransaction` job 和相同的校验规则。行为与 UI 路径一致。

#### 入口 3：直接构造 Transaction 模型（绕过 Job）

如果通过 `Transaction::create()` 或 `$transaction->save()` 直接写入数据库，绕过 `CreateBankingDocumentTransaction` 的 `checkAmount()`，可以创建任意金额的交易（包括 $0 或超额付款），并关联到任意 document。

**此入口下可达不一致场景**：直接创建 $0 交易关联到已 `paid` 的 invoice，然后删除 $100 付款 → 状态 `partial`，实际已付 $0.00。

#### 入口 4：数据库直接操作

直接 SQL 插入也是可能的，但不在应用层代码范围内。

### 边界场景可达性总结

| 入口 | 零金额交易可达？ | 不一致场景可达？ | 原因 |
|------|-----------------|-----------------|------|
| UI 模态框 | ✅ 可达（`amount:0` 允许） | ❌ 不可达 | 已 `paid` 的 document 无法再添加付款；未 `paid` 时创建 $0 交易会被 `checkAmount` 设为 `paid` |
| API 路径 | ✅ 可达 | ❌ 不可达 | 同 UI 路径逻辑 |
| 直接构造模型 | ✅ 可达 | ✅ 可达 | 绕过 `checkAmount` 校验 |
| SQL 直接操作 | ✅ 可达 | ✅ 可达 | 绕过所有应用层校验 |

### "重复付款"场景不可达

文档上一版中"总额 $50，两笔各 $50"的示例在正常入口下不可达：

- UI 路径：`create()` 预填充金额 = `总额 - 已付`，第二笔会被预填充为 $0.00（而非 $50）
- API 路径：手动传 `amount=50` 时，`checkAmount` 中 `bccomp(50, 0) === 1` → 抛出 `over_payment` 异常
- 直接构造模型：可达但非正常入口

---

## 流程图

```
DeleteTransaction::handle()
│
└─► DB::transaction
     │
     ├─► deleteRelationships (recurring, taxes)
     │
     └─► model->delete()  ──► Observer::deleted()
          │
          ├─ document_id 非空?
          │    │
          │    ├─Yes─► type = income? → invoice, 否则 → bill
          │    │       │
          │    │       ▼
          │    │    $transaction->{$type}  (document_id 关联，不按文档类型过滤)
          │    │       │
          │    │       ▼
          │    │    document 存在? ──No──► return
          │    │       │
          │    │       ▼
          │    │    transactions_count = transactions->count()
          │    │       │
          │    │       ▼
          │    │    event(TransactionsCounted)  [扩展 listener 可修改临时属性]
          │    │       │
          │    │       ▼
          │    │    status = sent/received (默认)
          │    │       │
          │    │       ▼
          │    │    count > 0 ? ──► status = partial
          │    │       │
          │    │       ▼
          │    │    unset transactions_count
          │    │       │
          │    │       ▼
          │    │    保存 document + 创建历史记录
          │    │
          │    └─No─► 跳过 document 更新
          │
          ├─ split_id 非空?
          │    │
          │    ├─Yes─► 查找父交易 (income-split / expense-split)
          │    │       │
          │    │       ▼
          │    │    splits->count() == 0 ?
          │    │       │
          │    │       ▼
          │    │    dispatch UpdateTransaction
          │    │       ├─ authorize(): reconciled? ──Yes──► 抛异常 → DB 事务回滚
          │    │       ├─ authorize(): transfer? ──Yes──► 抛异常 → DB 事务回滚
          │    │       └─ 成功 → 父交易 type 恢复为 income/expense
          │    │
          │    └─No─► 跳过 split 父交易更新
          │
          └─► [Observer 执行完毕]
               │
               ▼
          event(TransactionDeleted)
```
