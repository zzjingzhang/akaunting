# 银行对账 transaction.reconciled 字段审计报告

## 1. started_at/ended_at 如何被规范到一天边界

在创建对账时，日期边界通过 `startOfDay()` 和 `endOfDay()` 方法规范化：

**[CreateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L18-L19)**
```php
$started_at = Date::parse($this->request->get('started_at'))->startOfDay();
$ended_at = Date::parse($this->request->get('ended_at'))->endOfDay();
```

- `started_at` 被设置为当天 00:00:00
- `ended_at` 被设置为当天 23:59:59

在控制器查询交易时也有类似处理：
**[Reconciliations.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Reconciliations.php#L186-L187)**
```php
$started = explode(' ', $started_at)[0] . ' 00:00:00';
$ended = explode(' ', $ended_at)[0] . ' 23:59:59';
```

---

## 2. request['transactions'] 的 key/value 怎样解析出交易 id

请求中的 transactions 是一个关联数组，key 格式为 `{type}_{transaction_id}`，value 为交易金额。通过 `explode('_', $key)` 解析：

**[CreateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L33-L42)**
```php
foreach ($transactions as $key => $value) {
    if (empty($value)) {
        continue;
    }

    $t = explode('_', $key);

    $transaction = Transaction::find($t[1]);
    $transaction->reconciled = 1;
    $transaction->save();
}
```

**[UpdateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L22-L35)**
```php
foreach ($transactions as $key => $value) {
    $transaction_reconcile = $reconcile;

    if (empty($value) || $value === 'false') {
        $transaction_reconcile = 0;
    }

    $t = explode('_', $key);

    $transaction = Transaction::find($t[1]);
    $transaction->reconciled = $transaction_reconcile;
    $transaction->save();
}
```

- key 示例：`income_123`、`expense_456`
- `$t[0]` = 交易类型（income/expense）
- `$t[1]` = 交易 ID

---

## 3. reconcile 为 0 或没有 transactions 时是否标记交易

### 创建对账时
**[CreateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L32-L44)**
```php
if ($reconcile && $transactions) {
    foreach ($transactions as $key => $value) {
        // 标记交易为已对账
    }
}
```
- **必须同时满足** `$reconcile = 1` **且** `$transactions` 非空，才会标记交易
- reconcile 为 0 时：不标记任何交易
- 没有 transactions 时：不标记任何交易

### 更新对账时
**[UpdateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L22-L36)**
```php
if ($transactions) {
    foreach ($transactions as $key => $value) {
        $transaction_reconcile = $reconcile;

        if (empty($value) || $value === 'false') {
            $transaction_reconcile = 0;
        }
        // 更新交易 reconciled 状态
    }
}
```
- 只要有 `$transactions` 就会处理，无论 reconcile 是 0 还是 1
- 当 `$value` 为空或等于 `'false'` 时，强制标记为 0
- 没有 transactions 时：不处理任何交易

---

## 4. 更新/删除 reconciliation 时旧交易标记如何处理

### 更新对账时
**[UpdateReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L22-L36)**

⚠️ **潜在问题**：更新时只处理请求中提交的 transactions，对于之前被标记但本次未提交的交易，**不会自动取消标记**。这可能导致孤儿标记。

### 删除对账时
**[DeleteReconciliation.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/DeleteReconciliation.php#L16-L21)**
```php
Transaction::where('account_id', $this->model->account_id)
    ->isReconciled()
    ->whereBetween('paid_at', [$this->model->started_at, $this->model->ended_at])
    ->each(function ($transaction) {
        $transaction->reconciled = 0;
        $transaction->save();
    });
```

删除对账时：
1. 按 `account_id` 筛选
2. 按 `isReconciled()`（即 `reconciled = 1`）筛选
3. 按 `paid_at` 在对账时间范围内筛选
4. 将所有符合条件的交易标记为 `reconciled = 0`

---

## 5. 为什么这会影响 DeleteDocument/CancelDocument 的授权判断

**[DeleteDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/DeleteDocument.php#L42-L49)**
**[CancelDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CancelDocument.php#L41-L49)**

两者的授权检查完全相同：
```php
public function authorize(): void
{
    if ($this->model->transactions()->isReconciled()->count()) {
        $type = Str::plural($this->model->type);
        $message = trans('messages.warning.reconciled_doc', ['type' => trans_choice("general.$type", 1)]);

        throw new \Exception($message);
    }
}
```

### 影响链

1. 当银行对账标记某交易为 `reconciled = 1`
2. 该交易关联到某个文档（发票/账单等）
3. 尝试删除或取消该文档时
4. 授权检查发现该文档存在已对账的交易
5. 抛出异常，阻止删除/取消操作

### 关键问题点

- **删除对账不彻底**：如果删除对账时因某些原因未能将所有关联交易的 `reconciled` 重置为 0，这些交易将无法再被删除
- **更新对账遗漏**：更新对账时未处理的旧交易可能残留 `reconciled = 1` 标记
- **跨对账周期**：如果一笔交易被多个对账涵盖，删除其中一个对账可能误重置其他对账的标记

### isReconciled 作用域定义

**[Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L307-L310)**
```php
public function scopeIsReconciled(Builder $query): Builder
{
    return $query->where('reconciled', 1);
}
```

---

## 总结

| 操作 | 交易标记行为 | 潜在风险 |
|------|-------------|----------|
| 创建对账 | reconcile=1 且有交易时，标记为 1 | 无 |
| 更新对账 | 只处理提交的交易，旧交易不处理 | 孤儿标记残留 |
| 删除对账 | 按账户+时间范围批量重置为 0 | 可能误重置其他对账的交易 |
| 删除/取消文档 | 检查关联交易是否有 reconciled=1，有则拒绝 | 残留标记导致无法删除文档 |
