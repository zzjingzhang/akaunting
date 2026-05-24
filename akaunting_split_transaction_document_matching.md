# SplitTransaction 银行交易拆分到多个 Document 分析

## 1. items 金额总和如何按 currency precision 与父交易比较

在 [SplitTransaction.php#L60-L84](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L60-L84) 的 `checkAmount()` 方法中实现金额校验：

```php
protected function checkAmount(): bool
{
    $total_amount = 0;

    foreach ($this->request->items as $item) {
        $total_amount += $item['amount'];
    }

    $precision = currency($this->model->currency_code)->getPrecision();

    $compare = bccomp($total_amount, $this->model->amount, $precision);

    if ($compare !== 0) {
        $error_amount = $this->model->amount;

        $message = trans('messages.error.same_amount', [
            'transaction' => ucfirst(trans_choice('general.' . Str::plural($this->model->type), 1)),
            'amount' => money($error_amount, $this->model->currency_code)
        ]);

        throw new \Exception($message);
    }

    return true;
}
```

**比较逻辑：**
- 遍历 `items` 累加所有子项金额得到 `$total_amount`
- 获取父交易货币的精度：`currency($this->model->currency_code)->getPrecision()`
- 使用 `bccomp($total_amount, $this->model->amount, $precision)` 进行高精度比较
- 比较结果 `$compare !== 0` 时抛出 `messages.error.same_amount` 异常

**关键点：** 使用货币精度确保不同货币（如 USD 是 2 位小数，JPY 是 0 位小数）的金额比较正确性。`checkAmount()` 是 `handle()` 中最先执行的步骤，早于 `authorize()`、`TransactionSplitting` 事件和 `DB::transaction`。

---

## 2. document_id 缺失或不存在时在哪一步失败

### 2.1 多层失败点定位

失败可能在三个层级发生，取决于调用入口：

**层级 A：正常 UI 入口（TransactionConnect FormRequest 校验）**

[TransactionConnect.php#L14-L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/TransactionConnect.php#L14-L21)：

```php
public function rules()
{
    return [
        'data' => 'required|array',
        'data.items' => 'required|array',
        'data.items.*.document_id' => 'required',
    ];
}
```

- 在 [Transactions::connect()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L432-L461) 进入 `SplitTransaction` 之前由 Laravel FormRequest 先行校验
- `document_id` 缺失/null → 立即返回 422 Validation 错误，根本不会 dispatch 任何 Job
- 发生时间：**远早于** `checkAmount()`、`authorize()`、`TransactionSplitting` 事件和 `DB::transaction`

**层级 B：SplitTransaction 直接 dispatch 时的 authorize 校验**

[SplitTransaction.php#L47-L58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L47-L58)：

```php
public function authorize(): void
{
    foreach ($this->request->items as $item) {
        if (empty($item['document_id'])) {
            throw new \Exception(trans('messages.error.not_found', ['type' => trans_choice('general.documents', 1)]));
        }

        if (! Document::find($item['document_id'])) {
            throw new \Exception(trans('messages.error.not_found', ['type' => trans_choice('general.documents', 1)]));
        }
    }
}
```

- `document_id` 为空或 `Document::find()` 返回 null → 抛出 `messages.error.not_found` 异常
- 发生时间：**在 `checkAmount()` 之后、`TransactionSplitting` 事件和 `DB::transaction` 之前**（见 `handle()` 顺序）
- 若失败，父交易未被修改，没有任何副作用

**层级 C：handle() 中 `Document::find($item['document_id'])` 失败的理论情况**

[SplitTransaction.php#L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L33)：

```php
$document = Document::find($item['document_id']);
$this->dispatch(new MatchBankingDocumentTransaction($document, $transaction));
```

- 由于 `authorize()` 已经对相同的 `document_id` 做过存在性校验，正常情况下此 `find()` 不会返回 null
- 但如果文档在 authorize 之后、dispatch 之前被并发删除，`$document` 可能为 null，随后 `MatchBankingDocumentTransaction` 会将 null 作为 `Document $model` 类型参数传入，导致框架层面的类型错误/异常
- 发生时间：**在 `DB::transaction` 内部**，会回滚已保存的子交易

### 2.2 handle() 执行顺序回顾

[SplitTransaction.php#L18-L45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L18-L45)：

```
handle()
  ├─ 1. checkAmount()          ← 金额不等时失败
  ├─ 2. authorize()            ← document_id 缺失/不存在时失败
  ├─ 3. TransactionSplitting   ← 事件
  ├─ 4. DB::transaction {
  │     foreach items {
  │       duplicate()+save()
  │       Document::find()     ← 理论上的并发失败点
  │       MatchBankingDocumentTransaction
  │     }
  │     父交易 type = split_type
  │  }
  └─ 5. TransactionSplitted
```

---

## 3. 每个子交易最终复制和覆盖字段

### 3.1 duplicate() 阶段的复制

通过 Bkwld\Cloner\Cloneable trait 的 `duplicate()`：

- 所有 `fillable` 属性（company_id, type, number, account_id, paid_at, amount, currency_code, currency_rate, document_id, contact_id, description, category_id, payment_method, reference, parent_id, split_id, created_from, created_by）
- 关联关系：`recurring` 和 `taxes`（见 [Transaction.php#L94](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L94) 的 `$cloneable_relations = ['recurring', 'taxes']`）

### 3.2 onCloning 钩子的自动修改

[Transaction.php#L317-L329](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L317-L329)：

```php
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

覆盖的字段：`number`（新编号）、`document_id`（null）、`split_id`（null）、`reconciled`（取消对账）。

### 3.3 SplitTransaction::handle() 中显式覆盖

[SplitTransaction.php#L28-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L28-L31)：

```php
$transaction            = $this->model->duplicate();
$transaction->split_id  = $this->model->id;
$transaction->amount    = $item['amount'];
$transaction->save();
```

覆盖：`split_id`（改为父交易 id）、`amount`（改为子项金额）。

### 3.4 MatchBankingDocumentTransaction::checkAmount() 对 amount 的舍入

[MatchBankingDocumentTransaction.php#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L51)：

```php
$amount = $this->transaction->amount = round($this->transaction->amount, $transaction_precision);
```

- `transaction.amount` 可能按交易币种精度被再次舍入（注意此处直接写回 `$this->transaction->amount`，不是 request）
- 该值在后续 `UpdateTransaction` 的 `model->update(request->all())` 中并不会被覆盖（request 只含 `document_id` 与 `type`），所以最终落库的是被舍入后的值

### 3.5 MatchBankingDocumentTransaction 中 dispatch UpdateTransaction 的实际覆盖

[MatchBankingDocumentTransaction.php#L29-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L29-L33)：

```php
$this->transaction = $this->dispatch(new UpdateTransaction($this->transaction, [
    'document_id'   => $this->model->id,
    'type'          => $this->transaction->type,
]));
```

request 中仅包含 `document_id` 与 `type`（sparse request）。

### 3.6 UpdateTransaction::handle() 对 taxes 与 recurring 的处理

[UpdateTransaction.php#L31-L55](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransaction.php#L31-L55)：

```php
\DB::transaction(function () {
    $this->model->update($this->request->all());

    // ... attachment 处理 ...

    $this->deleteRelationships($this->model, ['taxes'], true);

    $this->dispatch(new CreateTransactionTaxes($this->model, $this->request));

    // Recurring
    $this->model->updateRecurring($this->request->all());
});
```

**对 taxes 的影响：**

1. `deleteRelationships($this->model, ['taxes'], true)` → 由 [Relationships.php#L41-L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php#L41-L69) 对已克隆的 `taxes` 关联执行 `forceDelete()`，**父交易原本的 taxes 关联在 duplicate() 阶段复制来的子 TransactionTax 会被物理删除**
2. `CreateTransactionTaxes` → [CreateTransactionTaxes.php#L33-L37](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L33-L37)：

```php
public function handle()
{
    if (empty($this->request['tax_ids'])) {
        return false;
    }
    // ...
}
```

- 由于 request 中不含 `tax_ids`，**不会重新创建任何 TransactionTax**
- 结果：子交易最终 **没有 taxes 关联**（与 duplicate() 后的中间状态不同）

**对 recurring 的影响：**

`$this->model->updateRecurring($this->request->all())` → [Recurring.php#L45-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Recurring.php#L45-L51)：

```php
public function updateRecurring($request)
{
    if (empty($request['recurring_frequency']) || ($request['recurring_frequency'] == 'no')) {
        $this->recurring()->delete();

        return;
    }
    // ...
}
```

- 由于 request 不含 `recurring_frequency`（empty），`recurring()` 关系被删除
- 结果：子交易最终 **没有 recurring 关联**（与 duplicate() 后的中间状态不同）

### 3.7 字段状态总结

| 字段 | duplicate() 后初值 | onCloning | handle() | checkAmount | UpdateTransaction | 最终 |
|------|---------|-----------|----------|-------------|-------------------|------|
| `number` | 父交易号 | 新编号 | - | - | - | 新编号 |
| `document_id` | 父交易 id | null | - | - | 设为匹配 document id | 匹配的 document id |
| `split_id` | 父交易 id | null | 改为父交易 id | - | - | 父交易 id |
| `reconciled` | 父交易值 | unset | - | - | - | unset（默认 0） |
| `amount` | 父交易金额 | - | 子项金额 | 按币种精度舍入 | request 中无 amount，不会覆盖 | 舍入后的值 |
| `type` | 父交易 type | - | - | - | 设回自身 type（不变） | 父交易 type |
| `taxes` 关联 | 复制的 TransactionTax 集合 | - | - | - | 被 forceDelete + 无 tax_ids 不重建 | **空** |
| `recurring` 关联 | 复制的 Recurring 记录 | - | - | - | updateRecurring 无 recurring_frequency → delete | **空** |

---

## 4. MatchBankingDocumentTransaction 对 document/status/transaction 可能有什么影响

### 4.1 Document::paid 通过 transactions() 按 document_id 累计已有交易

[Document.php#L373-L403](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L403)：

```php
public function getPaidAttribute()
{
    if (empty($this->amount)) { return false; }
    if ($this->status == 'paid') { return $this->amount; }

    $paid = 0;
    $code = $this->currency_code;
    $rate = $this->currency_rate;
    $precision = currency($code)->getPrecision();

    if (!$this->relationLoaded('transactions')) {
        $this->load('transactions');
    }

    if ($this->transactions->count()) {
        foreach ($this->transactions as $transaction) {
            $amount = $transaction->amount;

            if ($code != $transaction->currency_code) {
                $amount = $this->convertBetween($amount, $transaction->currency_code, $transaction->currency_rate, $code, $rate);
            }

            $paid += $amount;
        }
    }
    // ...
}
```

- `transactions()` 关系：[Document.php#L176-L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L176-L179) `return $this->hasMany('App\Models\Banking\Transaction', 'document_id');`
- 累计同一 `document_id` 下所有已有交易的 `amount`（按 document 币种换算后累加）
- 若 document 已经是 `paid` 状态则直接返回 `$this->amount`（全额）

### 4.2 跨币种转换分支

[MatchBankingDocumentTransaction.php#L53-L57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L53-L57)：

```php
if ($this->model->currency_code != $code) {
    $converted_amount = $this->convertBetween($amount, $code, $rate, $this->model->currency_code, $this->model->currency_rate);

    $amount = round($converted_amount, $document_precision);
}
```

- 当 document 与 transaction 币种不一致时，按 `currency_rate` 转换后再按 document 精度舍入
- 后续 `bccomp` 比较与 `over_match` 错误金额都使用转换后的值

### 4.3 PaidAmountCalculated 只是事件扩展点

[PaidAmountCalculated.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Document/PaidAmountCalculated.php) 是一个纯粹的事件类：

```php
class PaidAmountCalculated extends Event
{
    public $model;

    public function __construct($model)
    {
        $this->model = $model;
    }
}
```

- 在 [Event.php#L14-L156](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L14-L156) 的 `$listen` / `$subscribe` 中 **没有任何监听器注册**
- 作用只是作为扩展点钩子（让模块/插件可订阅并修改 `$model->paid_amount`），对核心状态计算没有直接影响

### 4.4 状态判定与 over_match 异常

[MatchBankingDocumentTransaction.php#L59-L84](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L59-L84)：

```php
$this->model->paid_amount = $this->model->paid;
event(new PaidAmountCalculated($this->model));

$total_amount = round($this->model->amount - $this->model->paid_amount, $document_precision);

unset($this->model->reconciled);
unset($this->model->paid_amount);

$compare = bccomp($amount, $total_amount, $document_precision);

if ($compare === 1) {
    $error_amount = $total_amount;

    if ($this->model->currency_code != $code) {
        $converted_amount = $this->convertBetween($total_amount, $this->model->currency_code, $this->model->currency_rate, $code, $rate);

        $error_amount = round($converted_amount, $transaction_precision);
    }

    $message = trans('messages.error.over_match', ['type' => ucfirst($this->model->type), 'amount' => money($error_amount, $code)]);

    throw new \Exception($message);
} else {
    $this->model->status = ($compare === 0) ? 'paid' : 'partial';
}
```

- `$compare === 0`：本次金额 = 剩余未付 → status = `paid`
- `$compare === -1`：本次金额 < 剩余未付 → status = `partial`
- `$compare === 1`：本次金额 > 剩余未付 → 抛出 `messages.error.over_match` 异常

### 4.5 over_match 在 SplitTransaction 外层 DB::transaction 中的回滚影响

[SplitTransaction.php#L26-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L26-L40)：

```php
DB::transaction(function () {
    foreach ($this->request->items as $item) {
        $transaction            = $this->model->duplicate();
        $transaction->split_id  = $this->model->id;
        $transaction->amount    = $item['amount'];
        $transaction->save();

        $document = Document::find($item['document_id']);

        $this->dispatch(new MatchBankingDocumentTransaction($document, $transaction));
    }

    $this->model->type = config('type.transaction.' . $this->model->type . '.split_type');
    $this->model->save();
});
```

- `MatchBankingDocumentTransaction::handle()` 内部的 `\DB::transaction` 是**嵌套事务**（MySQL savepoint）
- 若 `over_match` 异常在 `MatchBankingDocumentTransaction` 中抛出，会冒泡到外层 `SplitTransaction` 的 `DB::transaction`
- 整个事务回滚，**已保存的子交易、Document 状态修改、DocumentHistory 都会被撤销**
- 父交易的 `type = split_type` 也不会被应用
- 注意：`TransactionSplitting` 事件是在 `DB::transaction` **之前**触发，已经发出；`TransactionSplitted` 事件不会触发

### 4.6 正常 UI 同币种过滤 vs 直接 job 可达性差异

**UI 入口（Transactions::dial + connect）：**

[Transactions::dial()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L392-L430)：

```php
$documents = $builder->notPaid()
                     ->where('currency_code', $transaction->currency_code)  // 同币种过滤
                     ->with(['media', 'totals', 'transactions'])
                     ->get()
                     ->toJson();
```

- 前端仅能选择与交易同币种、未付清、属于对应 contact 的 Document
- UI 路径几乎不可能触发跨币种路径

**直接 dispatch Job：**

- `SplitTransaction` / `MatchBankingDocumentTransaction` 可以通过代码/队列/测试直接调用
- 传入不同币种的 document_id 会走到跨币种分支，此时：
  - `amount` 会被转换并按 document 精度舍入
  - `over_match` 错误金额也会反向转换回交易币种展示
  - document 状态 `paid`/`partial` 基于转换后的金额与 document 未付金额比较

---

## 5. 父交易 type 为什么最后会变成 split_type，删除子交易后可能如何恢复

### 5.1 设置 split_type

[SplitTransaction.php#L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L38)：

```php
$this->model->type = config('type.transaction.' . $this->model->type . '.split_type');
```

根据 [type.php#L388-L389](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L388-L389) 和 [type.php#L515-L516](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php#L515-L516)：
- `income` / `income-transfer` → `income-split`（`Transaction::INCOME_SPLIT_TYPE`）
- `expense` / `expense-transfer` → `expense-split`（`Transaction::EXPENSE_SPLIT_TYPE`）

**目的：** 标记该交易已被拆分，配合 [Transaction.php#L254-L262](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L254-L262) 的 `scopeIsSplit` / `scopeIsNotSplit` 方便查询和展示。

### 5.2 删除子交易时 Observer::deleted() 的双路径处理

[Observers/Transaction.php#L23-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L23-L34)：

```php
public function deleted(Model $transaction)
{
    if (! empty($transaction->document_id)) {
        $type = ($transaction->type == 'income') ? Document::INVOICE_TYPE : Document::BILL_TYPE;

        $this->updateDocument($transaction, $type);
    }

    if (! empty($transaction->split_id)) {
        $this->updateTransaction($transaction);
    }
}
```

**路径 A：有 document_id（子交易已匹配）→ updateDocument**

[Observers/Transaction.php#L36-L59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L36-L59)：

```php
protected function updateDocument($transaction, $type)
{
    $document = $transaction->{$type};

    if (empty($document)) {
        return;
    }

    $document->transactions_count = $document->transactions->count();

    event(new TransactionsCounted($document));

    $document->status = ($type == Document::INVOICE_TYPE) ? 'sent' : 'received';

    if ($document->transactions_count > 0) {
        $document->status = 'partial';
    }

    unset($document->transactions_count);

    $document->save();

    $this->dispatch(new CreateDocumentHistory($document, 0, $this->getDescription($transaction)));
}
```

- 重新统计 document 剩余关联交易数
- 删除前还有其他支付 → status = `partial`
- 删除后没有任何关联交易 → 发票回 `sent`，账单回 `received`
- 写入 DocumentHistory 记录（金额+"payments deleted"描述）

**路径 B：有 split_id（是子交易）→ updateTransaction（父交易 type 恢复）**

[Observers/Transaction.php#L68-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L68-L79)：

```php
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

- 仅当 `$splitted_transaction->splits->count() == 0`（所有子交易都已删除）才触发恢复
- 恢复的 type 取自**最后一个被删除的子交易的 `type`**（通常与父交易原始 type 一致）
- 注意：`$splitted_transaction = Model::find($transaction->split_id)` 如果父交易已被删除会得到 null，但代码未做 null 检查，极端情况下可能出现调用 `null->splits` 的错误

### 5.3 UpdateTransaction::authorize() 对恢复的拦截

[UpdateTransaction.php#L65-L76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransaction.php#L65-L76)：

```php
public function authorize(): void
{
    if ($this->model->reconciled) {
        $message = trans('messages.warning.reconciled_tran');

        throw new \Exception($message);
    }

    if ($this->model->isTransferTransaction()) {
        throw new \Exception(trans('messages.error.transfer_transaction'));
    }
}
```

- 如果父交易处于 `reconciled` 状态 → 恢复会失败，type 不会被还原
- 父交易 type 已被设为 `income-split` 或 `expense-split`，不匹配 `-transfer` 后缀 → transfer 检查不会触发
- 恢复失败不会回滚子交易删除；父交易保持 `split_type`

### 5.4 稀疏 request 恢复 type 对父交易 taxes/recurring 的副作用

[UpdateTransaction.php#L49-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransaction.php#L49-L54)：

```php
$this->deleteRelationships($this->model, ['taxes'], true);

$this->dispatch(new CreateTransactionTaxes($this->model, $this->request));

// Recurring
$this->model->updateRecurring($this->request->all());
```

- request 只含 `type` → 不含 `tax_ids` 与 `recurring_frequency`
- `deleteRelationships` 强制删除父交易所有 `TransactionTax`，`CreateTransactionTaxes` 不会重建
- `updateRecurring` 因 `recurring_frequency` 为空，会 `$this->recurring()->delete()`
- **副作用：父交易原本的 taxes 和 recurring 关联也会被清空**

这意味着"恢复 type"操作实际上是一个破坏性的 UpdateTransaction：不仅还原 type，还可能 wipe 掉父交易的税项和循环配置。

---

## 整体流程

```
Transactions::connect()
├─ [FormRequest] TransactionConnect 校验 document_id 必填
├─ total_items == 1 → MatchBankingDocumentTransaction（不走 SplitTransaction）
└─ total_items > 1 → ajaxDispatch(SplitTransaction)
   └─ SplitTransaction::handle()
      ├─ checkAmount()                    # items 金额总和 == 父交易金额（按货币精度）
      ├─ authorize()                      # 每个 document_id 存在
      ├─ event TransactionSplitting
      └─ DB::transaction
         └─ foreach items
            ├─ duplicate()                # 复制 fillable + recurring + taxes
            │   └─ onCloning              # 新 number, document_id=null, split_id=null, reconciled unset
            ├─ split_id = parent_id
            ├─ amount = item.amount
            ├─ save()
            ├─ Document::find(document_id)
            └─ dispatch MatchBankingDocumentTransaction
               ├─ checkAmount()
               │   ├─ amount 按交易币种精度舍入
               │   ├─ 跨币种时转换为 document 币种
               │   ├─ Document::paid 累计同 document_id 的历史交易
               │   ├─ PaidAmountCalculated 事件（无默认监听）
               │   ├─ bccomp(amount, document未付金额)
               │   │   ├─ 0 → status=paid
               │   │   ├─ -1 → status=partial
               │   │   └─ 1 → 抛出 over_match 异常 → 整事务回滚
               │   └─ unset reconciled, paid_amount
               └─ DB::transaction
                  ├─ dispatch UpdateTransaction(document_id, type)  # sparse request
                  │   ├─ authorize()      # 未对账、非 transfer
                  │   ├─ model->update()  # 仅更新 document_id, type
                  │   ├─ deleteRelationships(taxes, force)  # 清掉 duplicate() 复制的税项
                  │   ├─ CreateTransactionTaxes # 无 tax_ids → 跳过
                  │   └─ updateRecurring()     # 无 recurring_frequency → 删 recurring
                  ├─ document->save()     # 保存新 status
                  └─ CreateDocumentHistory
         ├─ 父交易 type = split_type (income-split / expense-split)
         └─ 父交易 save()
      └─ event TransactionSplitted

当删除子交易时 Observer::deleted()
├─ 有 document_id
│   └─ updateDocument()
│      ├─ 重算 document->transactions_count
│      ├─ 剩余=0 → sent/received；剩余>0 → partial
│      ├─ save document
│      └─ CreateDocumentHistory
└─ 有 split_id
    └─ updateTransaction()
       ├─ splits->count()==0 → dispatch UpdateTransaction(parent, [type => child.type])
       │   ├─ authorize()  # 父交易 reconciled 则失败不恢复
       │   └─ 副作用：wipe 父交易的 taxes 与 recurring 关联
       └─ 否则不做任何恢复，父交易保持 split_type
```
