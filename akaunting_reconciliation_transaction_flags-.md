# 银行对账 transaction.reconciled 字段审计报告

---

## 1. started_at/ended_at 如何被规范到一天边界

### 创建对账

**[CreateReconciliation.php#L18-L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L18-L28)**

```php
$started_at = Date::parse($this->request->get('started_at'))->startOfDay();
$ended_at = Date::parse($this->request->get('ended_at'))->endOfDay();

$this->request->merge([
    'started_at' => $started_at,
    'ended_at'   => $ended_at,
    'reconciled' => $reconcile,
]);

$this->model = Reconciliation::create($this->request->all());
```

- 规范化：`startOfDay()` → `00:00:00`，`endOfDay()` → `23:59:59`
- 持久化：**是**。规范化后的值通过 `merge()` 写回 request，再通过 `create()` 持久化到 `reconciliations` 表。

### 更新对账

**[UpdateReconciliation.php#L14-L20](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L14-L20)**

```php
$reconcile = (int) $this->request->get('reconcile');
$transactions = $this->request->get('transactions');

$this->model->transactions = $transactions;
$this->model->reconciled = $reconcile;
$this->model->save();
```

- 规范化：**不执行**。Update Job 不读取也不修改 `started_at`/`ended_at`。
- 持久化：`reconciliations` 表中的 `started_at`/`ended_at` 保持创建时写入的值不变。

### 删除对账

**[DeleteReconciliation.php#L16-L18](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/DeleteReconciliation.php#L16-L18)**

```php
Transaction::where('account_id', $this->model->account_id)
    ->isReconciled()
    ->whereBetween('paid_at', [$this->model->started_at, $this->model->ended_at])
```

- 规范化：**不执行**。直接使用从数据库读取的 `$this->model->started_at` 和 `$this->model->ended_at`。
- 持久化：删除操作不涉及写入 `reconciliations` 表。

### getTransactions 查询

**[Reconciliations.php#L186-L189](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Reconciliations.php#L186-L189)**

```php
$started = explode(' ', $started_at)[0] . ' 00:00:00';
$ended = explode(' ', $ended_at)[0] . ' 23:59:59';

$transactions = Transaction::with('account', 'contact')
    ->where('account_id', $account->id)
    ->whereBetween('paid_at', [$started, $ended])
    ->get();
```

- 规范化：**是**（仅查询层面）。通过字符串拼接将日期补全到 `00:00:00` 和 `23:59:59`。
- 持久化：**否**。此规范化仅用于 SQL 查询，不写回任何模型或表。

### 汇总

| 场景 | 是否规范化 | 是否持久化到 reconciliations 表 |
|------|-----------|-------------------------------|
| 创建对账 | 是（startOfDay/endOfDay） | 是 |
| 更新对账 | 否 | 否（字段不被修改） |
| 删除对账 | 否（用 DB 原值） | 否 |
| getTransactions 查询 | 是（字符串拼接） | 否 |

---

## 2. request['transactions'] 的 key/value 怎样解析出交易 id

### key 格式来源

**[create.blade.php#L150-L155](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/reconciliations/create.blade.php#L150-L155)**

```blade
@php $type = $item->isIncome() ? 'income' : 'expense'; @endphp

<x-form.input.checkbox name="{{ $type . '_' . $item->id }}"
    :value="! empty($item->amount_for_account) ? $item->amount_for_account : '0.00'"
```

**[edit.blade.php#L86-L87](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/reconciliations/edit.blade.php#L86-L87)**

```blade
$type = $item->isIncome() ? 'income' : 'expense';
$name = $type . '_' . $item->id;
```

- key 格式：`{type}_{transaction_id}`，如 `income_123`、`expense_456`
- type 值：`'income'` 或 `'expense'`（由 `$item->isIncome()` 决定）
- value 值：`$item->amount_for_account`（账户本位金额），为空时 fallback 为 `'0.00'`

### 前端 JS 中的解析

**[reconciliations.js#L82-L93](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/banking/reconciliations.js#L82-L93)**

```js
Object.keys(transactions).forEach(function(transaction) {
    if (! transactions[transaction]) {
        return;
    }

    let type = transaction.split('_');

    if (type[0] == 'income') {
        income_total += parseFloat(document.getElementById('transaction-' + type[1] + '-' + type[0]).value);
    } else {
        expense_total += parseFloat(document.getElementById('transaction-' + type[1] + '-' + type[0]).value);
    }
});
```

前端用 `split('_')` 解析，`type[0]` 是类型（income/expense），`type[1]` 是交易 ID。

### Job 中的解析与校验

**[CreateReconciliation.php#L33-L42](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L33-L42)**

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

**[UpdateReconciliation.php#L22-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L22-L35)**

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

**校验边界结论**：

- `$t[0]`（type 部分）：**不被校验**。两个 Job 都不验证 key 中的 type 是否与交易实际类型一致。
- `$t[1]`（id 部分）：**不被校验归属**。`Transaction::find($t[1])` 直接按主键查找，不验证该交易是否属于当前 `account_id`，也不验证 `paid_at` 是否在对账日期范围内。
- 如果 `Transaction::find()` 返回 `null`，后续 `$transaction->reconciled = 1` 将对 null 赋属性，导致 Job 失败。

### value 对 reconciled 的影响

#### CreateReconciliation（创建）

依据 [CreateReconciliation.php#L33-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L33-L36)：
```php
if (empty($value)) {
    continue;
}
```

PHP 语义：`empty('0.00')` → `false`（非空字符串不为空），`empty('false')` → `false`，`empty('')` → `true`。Create Job 不检查 `'false'` 字符串。

| value 值 | `empty($value)` 结果 | 行为 | transaction.reconciled 结果 |
|----------|---------------------|------|---------------------------|
| `'0.00'` | `false` | 继续执行 | 设为 `1` |
| `'false'` | `false` | 继续执行（Create 无 `'false'` 判断） | 设为 `1` |
| 空字符串 `''` | `true` | `continue` 跳过 | 不变（保持原值） |
| 非空金额字符串（如 `'100.00'`） | `false` | 继续执行 | 设为 `1` |
| key 不存在于 transactions 数组 | — | 不会被遍历到 | 不变 |

#### UpdateReconciliation（更新）

依据 [UpdateReconciliation.php#L24-L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L24-L28)：
```php
if (empty($value) || $value === 'false') {
    $transaction_reconcile = 0;
}
```

| value 值 | 条件判断 | 行为 | transaction.reconciled 结果 |
|----------|----------|------|---------------------------|
| `'0.00'` | 两个条件均为 `false` | 正常处理 | 跟随 `$reconcile` 值（0 或 1） |
| `'false'` | `$value === 'false'` 为 `true` | 强制置 0 | 设为 `0` |
| 空字符串 `''` | `empty($value)` 为 `true` | 强制置 0 | 设为 `0` |
| 非空金额字符串（如 `'100.00'`） | 两个条件均为 `false` | 正常处理 | 跟随 `$reconcile` 值（0 或 1） |
| key 不存在于 transactions 数组 | — | 不会被遍历到 | 不变（保持原值） |

**Create 与 Update 的关键差异**：
- Create：`'0.00'` 和 `'false'` 均被视为非空，**标记为 1**；空字符串才跳过
- Update：`'0.00'` 跟随 `$reconcile`；`'false'` 和空字符串**强制设为 0**
- Create Job 不判断 `'false'` 字符串，Update Job 才判断

---

## 3. reconcile 为 0 或没有 transactions 时是否标记交易，及交易 ID 校验边界

### 创建对账

**[CreateReconciliation.php#L21-L44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateReconciliation.php#L21-L44)**

```php
$reconcile = (int) $this->request->get('reconcile');
$transactions = $this->request->get('transactions');

$this->request->merge([
    'started_at' => $started_at,
    'ended_at' => $ended_at,
    'reconciled' => $reconcile,
]);

$this->model = Reconciliation::create($this->request->all());

if ($reconcile && $transactions) {
    foreach ($transactions as $key => $value) {
        // 仅当 reconcile=1 且 transactions 非空时才进入
    }
}
```

- `reconcile = 0`：**不标记任何交易**。`if ($reconcile && $transactions)` 条件短路。
- `reconcile = 1` 且 `transactions = null`：**不标记任何交易**。
- `reconcile = 1` 且 `transactions` 非空：按 value 非空原则标记，见第 2 节。

### 更新对账

**[UpdateReconciliation.php#L15-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L15-L36)**

```php
$reconcile = (int) $this->request->get('reconcile');
$transactions = $this->request->get('transactions');

$this->model->transactions = $transactions;
$this->model->reconciled = $reconcile;
$this->model->save();

if ($transactions) {
    foreach ($transactions as $key => $value) {
        // 只要 transactions 非空就进入，无论 reconcile 是 0 还是 1
    }
}
```

- `transactions = null` 或空数组：`if ($transactions)` 为 `false`，**不处理任何交易**。
- `transactions` 非空：进入循环，按 value 规则处理，见第 2 节。

### 交易 ID 校验边界 — 三种路径

**Request 校验类**：[Reconciliation.php#L14-L22](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Reconciliation.php#L14-L22)

```php
public function rules()
{
    return [
        'account_id'      => 'required|integer',
        'started_at'      => 'required|date_format:Y-m-d H:i:s|before_or_equal:ended_at',
        'ended_at'        => 'required|date_format:Y-m-d H:i:s|after_or_equal:started_at',
        'closing_balance' => 'required',
    ];
}
```

**对 `transactions` 字段没有任何校验规则**。三种路径共用此 Request 类：

| 路径 | 交易 ID 是否属于当前 account_id | 交易 ID 是否在日期范围内 | 依据 |
|------|------------------------------|------------------------|------|
| 正常 UI 提交 | 是 | 是 | `getTransactions()` 按 `account_id` + `paid_at between` 查询，页面只渲染这些交易的 checkbox，用户无法提交不在列表中的交易 ID |
| API 提交 | 否（不校验） | 否（不校验） | [Api/Reconciliations.php#L44-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Banking/Reconciliations.php#L44-L48) 的 `store()` 直接 dispatch `CreateReconciliation`，不做额外校验 |
| 手工构造请求 | 否（不校验） | 否（不校验） | 同上，Request 类不校验 transactions，Job 中 `Transaction::find()` 直接按 ID 查找 |

**UI 路径的隐式约束**：
- [create.blade.php#L117-L162](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/reconciliations/create.blade.php#L117-L162)：`@foreach($transactions as $item)` 遍历 `getTransactions()` 返回的结果
- [edit.blade.php#L51-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/reconciliations/edit.blade.php#L51-L112)：同样遍历 `getTransactions()` 返回的结果
- UI 用户只能勾选页面上已有的交易，无法提交不在该范围内的交易 ID

---

## 4. 更新 reconciliation 时旧交易标记如何处理

### UpdateReconciliation 的处理逻辑

**[UpdateReconciliation.php#L18-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L18-L36)**

```php
$this->model->transactions = $transactions;   // 整字段替换
$this->model->reconciled = $reconcile;
$this->model->save();

if ($transactions) {
    foreach ($transactions as $key => $value) {
        // 仅处理本次请求中的 key
    }
}
```

当旧 key 未出现在本次请求的 `transactions` 中时：

| 字段 | 结果 | 依据 |
|------|------|------|
| `transaction.reconciled` | **保持原值不变** | 旧 key 不被遍历，`$transaction->reconciled` 不会被赋值或 `save()` |
| `reconciliation.transactions` | **旧 key 被移除** | 第 18 行 `$this->model->transactions = $transactions` 整体替换为本次请求的 transactions 数组 |

### 三种路径的可达性

#### 正常 UI 编辑页

[edit.blade.php#L51-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/reconciliations/edit.blade.php#L51-L112) 渲染的交易来自 [Reconciliations.php#L184-L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Reconciliations.php#L184-L192) 的 `getTransactions()`，该方法按 `account_id` + `paid_at between` 查询。页面上所有渲染出来的交易 checkbox 都会被提交，包括未勾选的（value 为 `'false'` 或空值）。

**正常 UI 路径不会主动生成范围外 key，但可能遗漏此前已存在的范围外 key**：如果 `reconciliation->transactions` 中已经存在不在当前 `getTransactions()` 查询结果里的旧 key（例如此前由 API 或手工请求写入的、不属于当前 account_id 或 paid_at 不在范围内的 key），编辑页不会渲染这些 key，提交时也不会包含它们。[UpdateReconciliation.php#L18](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L18) 用本次请求整体覆盖 `$this->model->transactions`，且 [L22-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateReconciliation.php#L22-L35) 只处理本次请求中的 key，因此这些范围外旧 key 对应的交易 `reconciled` 将保持原值（可能为 1），同时这些旧 key 会从 `reconciliation->transactions` 字段中被移除。

对于 `getTransactions()` 查询结果范围内的交易，页面会渲染并提交全部 key，因此正常 UI 编辑不会遗漏范围内的旧 key。

#### API 路径

[Api/Reconciliations.php#L58-L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Banking/Reconciliations.php#L58-L62)

```php
public function update(Reconciliation $reconciliation, Request $request)
{
    $reconciliation = $this->dispatch(new UpdateReconciliation($reconciliation, $request));
    return new Resource($reconciliation->fresh());
}
```

API 调用者可以只提交部分 transactions 数组，遗漏的旧 key 对应的交易 `reconciled` 将保持原值（可能为 1）。

#### 手工请求

与 API 路径同理，可构造任意 transactions 数组，遗漏旧 key。

### 删除对账的处理

**[DeleteReconciliation.php#L14-L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/DeleteReconciliation.php#L14-L21)**

```php
$this->model->delete();

Transaction::where('account_id', $this->model->account_id)
    ->isReconciled()
    ->whereBetween('paid_at', [$this->model->started_at, $this->model->ended_at])
    ->each(function ($transaction) {
        $transaction->reconciled = 0;
        $transaction->save();
    });
```

删除对账时，按以下条件批量重置：
1. `account_id` 匹配
2. `reconciled = 1`（`isReconciled()` scope）
3. `paid_at` 在 `started_at` ~ `ended_at` 之间

所有符合条件的交易被设为 `reconciled = 0`。

---

## 5. 这些如何影响 DeleteDocument / CancelDocument 的授权判断

### 授权检查代码

**[DeleteDocument.php#L42-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/DeleteDocument.php#L42-L49)**

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

**[CancelDocument.php#L41-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CancelDocument.php#L41-L48)**

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

两者的授权检查完全一致。

### 关联关系定义

**[Document.php#L176-L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L176-L179)**

```php
public function transactions()
{
    return $this->hasMany('App\Models\Banking\Transaction', 'document_id');
}
```

**[Transaction.php#L307-L309](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L307-L309)**

```php
public function scopeIsReconciled(Builder $query): Builder
{
    return $query->where('reconciled', 1);
}
```

### 影响链

1. `$document->transactions()` → `SELECT * FROM transactions WHERE document_id = ?`
2. 追加 `->isReconciled()` → `AND reconciled = 1`
3. `->count()` > 0 → 抛出异常，阻止删除/取消

**等价 SQL**：
```sql
SELECT COUNT(*) FROM transactions
WHERE document_id = {document.id} AND reconciled = 1;
```

### 残留/误标记 transaction.reconciled 的具体影响场景

| 场景 | 残留标记如何产生 | 对文档操作的影响 |
|------|-----------------|-----------------|
| API 更新对账只提交部分 transactions | 旧交易 key 不在本次请求中，`transaction.reconciled` 保持为 1（见第 4 节） | 若该交易关联了文档（`document_id` 非空），该文档的 `DeleteDocument` / `CancelDocument` 将被拒绝 |
| 手工请求更新对账遗漏交易 | 同上 | 同上 |
| 同一账户有多个对账覆盖同一交易 | 删除其中一个对账时，`whereBetween('paid_at', ...)` 会把该交易的 `reconciled` 重置为 0，即使另一个对账仍包含它 | 该交易的对账状态被错误清除（反向影响，不阻止文档删除，但导致对账数据不一致） |
| 跨账户误标记 | API/手工请求提交不属于当前 account 的交易 ID，将其 `reconciled` 设为 1 | 若该交易关联文档，文档删除/取消被拒绝 |

**正常 UI 编辑不会遗漏 `getTransactions()` 范围内的旧 key**：页面会渲染并提交该范围内所有交易的 key（即使未勾选也会提交 `'false'`/空值），Update Job 会将这些交易的 `reconciled` 正确处理。但如果 `reconciliation->transactions` 中已存在范围外的旧 key（此前由 API 或手工请求写入），UI 不会渲染也不会提交它们，这些旧 key 对应的交易 `reconciled` 将保持原值。

---

## 汇总

| 操作 | started_at/ended_at 规范化 | 交易标记行为 | 产生残留标记的路径 |
|------|--------------------------|-------------|------------------|
| 创建对账 | 是，持久化 | reconcile=1 + 有 transactions + value 非空 → 标记 1 | API/手工请求可提交任意交易 ID |
| 更新对账 | 否（不修改该字段） | 遍历本次 transactions：value 空/'false' → 0，否则跟随 reconcile | **API/手工请求遗漏旧 key** |
| 删除对账 | 否（用 DB 原值） | 按 account_id + 日期范围批量重置为 0 | 可能误重置其他对账的交易 |
| 删除/取消文档 | 不适用 | 检查 `transactions WHERE document_id=? AND reconciled=1` | 残留标记导致拒绝 |
