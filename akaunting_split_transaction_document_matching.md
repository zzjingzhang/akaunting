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
        // throw exception
    }

    return true;
}
```

**比较逻辑：**
- 遍历 `items` 累加所有子项金额得到 `$total_amount`
- 获取父交易货币的精度：`currency($this->model->currency_code)->getPrecision()`
- 使用 `bccomp($total_amount, $this->model->amount, $precision)` 进行高精度比较
- 比较结果 `$compare !== 0` 时抛出 `messages.error.same_amount` 异常

**关键点：使用货币精度确保不同货币（如 USD 是 2 位小数，JPY 是 0 位小数）的金额比较正确性。

---

## 2. document_id 缺失或不存在时在哪一步失败

在 [SplitTransaction.php#L47-L58](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L47-L58) 的 `authorize()` 方法中进行校验：

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

**执行顺序：**
1. `handle()` 第一步调用 `checkAmount()` 校验金额
2. 第二步调用 `authorize()` 校验 document_id
3. 失败点：
   - `document_id` 为空时失败
   - `Document::find()` 找不到时失败
   - 抛出统一的错误消息：`messages.error.not_found`

---

## 3. 每个子交易复制了父交易哪些属性，又覆盖了哪些字段

**复制属性：**

通过 `$this->model->duplicate()` 使用 Bkwld\Cloner\Cloneable trait 复制：
- 所有 `fillable` 属性（company_id, type, number, account_id, paid_at, amount, currency_code, currency_rate, document_id, contact_id, description, category_id, payment_method, reference, parent_id, split_id, created_from, created_by）
- 关联关系：`recurring` 和 `taxes`（在 `$cloneable_relations` 中定义）

**onCloning 钩子自动修改：**
在 [Transaction.php#L317-L329](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L317-L329)：
```php
public function onCloning($src, $child = null)
{
    $this->number       = $this->getNextTransactionNumber($this->type, $suffix);
    $this->document_id  = null;
    $this->split_id     = null;
    unset($this->reconciled);
}
```

**显式覆盖字段（在 handle() 中）：

在 [SplitTransaction.php#L28-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L28-L31)：
```php
$transaction            = $this->model->duplicate();
$transaction->split_id  = $this->model->id;
$transaction->amount    = $item['amount'];
$transaction->save();
```

**MatchBankingDocumentTransaction 中覆盖：
在 [MatchBankingDocumentTransaction.php#L30-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php#L30-L33)：
```php
$this->transaction = $this->dispatch(new UpdateTransaction($this->transaction, [
    'document_id'   => $this->model->id,
    'type'          => $this->transaction->type,
]));
```

**总结覆盖的字段：**
| 字段 | 来源 | 说明 |
|------|------|------|
| `number` | onCloning | 生成新的交易编号 |
| `document_id` | onCloning + MatchBankingDocumentTransaction | 先设 null，后设为匹配的 document_id |
| `split_id` | onCloning + handle() | 先设 null，后设为父交易 id |
| `reconciled` | onCloning | 取消对账标记 |
| `amount` | handle() | 设置为子项金额 |

---

## 4. MatchBankingDocumentTransaction 对 document/status/transaction 可能有什么影响

在 [MatchBankingDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/MatchBankingDocumentTransaction.php) 中：

**对 Transaction 的影响：**
- 设置 `document_id` 关联到 Document
- 保存 Transaction（通过 UpdateTransaction job）

**对 Document 的影响：**
```php
protected function checkAmount(): bool
{
    // 计算 paid_amount
    $this->model->paid_amount = $this->model->paid;
    event(new PaidAmountCalculated($this->model));

    $total_amount = round($this->model->amount - $this->model->paid_amount, $document_precision);

    // 比较金额
    $compare = bccomp($amount, $total_amount, $document_precision);

    if ($compare === 1) {
        // 超额匹配，抛出异常
    } else {
        $this->model->status = ($compare === 0) ? 'paid' : 'partial';
    }
}
```

**状态变更逻辑：
- `$compare === 0`：金额完全匹配 → status = 'paid'
- `$compare === -1`：部分匹配 → status = 'partial'
- `$compare === 1`：超额匹配 → 抛出 `messages.error.over_match` 异常

**其他影响：
- 触发 `PaidAmountCalculated` 事件重新计算已付金额
- 创建 DocumentHistory 记录支付历史
- 保存 Document

---

## 5. 父交易 type 为什么最后会变成 split_type，删除子交易后可能如何恢复

**设置 split_type：

在 [SplitTransaction.php#L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/SplitTransaction.php#L38)：
```php
$this->model->type = config('type.transaction.' . $this->model->type . '.split_type');
```

根据 [type.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/type.php) 配置：
- `income` → `income-split`（Transaction::INCOME_SPLIT_TYPE
- `expense` → `expense-split`（Transaction::EXPENSE_SPLIT_TYPE

**目的：** 标记该交易已被拆分，便于查询和展示时区分普通交易和拆分父交易。

**删除子交易后的恢复：

在 [Transaction Observer](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L68-L79) 的 `deleted` 事件中：

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

**恢复逻辑：
1. 当删除子交易（有 split_id）
2. 查找父交易 `$splitted_transaction`
3. 检查父交易的 `splits` 关联计数为 0（所有子交易都已删除）
4. 将父交易的 `type` 恢复为被删除子交易的 `type`
5. 通过 UpdateTransaction job 保存

**注意：** 恢复的 type 来自最后一个被删除的子交易的 type，通常与原始父交易 type 一致。

---

## 整体流程

```
SplitTransaction.handle()
├─ checkAmount()          # 校验子项金额总和 = 父交易金额
├─ authorize()           # 校验 document_id 有效性
├─ TransactionSplitting  # 触发拆分前事件
├─ DB::transaction
│  └─ foreach items
│     ├─ duplicate()      # 复制父交易
│     ├─ 设置 split_id, amount
│     ├─ 保存子交易
│     ├─ 查找 Document
│     └─ dispatch MatchBankingDocumentTransaction
│        ├─ UpdateTransaction 设置 document_id
│        ├─ 更新 Document status (paid/partial)
│        └─ 创建支付历史
├─ 设置父交易 type 为 split_type
├─ 保存父交易
└─ TransactionSplitted  # 触发拆分后事件
```
