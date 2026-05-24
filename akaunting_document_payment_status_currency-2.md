# CreateBankingDocumentTransaction 支付流程分析（修订版）

## 核心文件

- 主流程: [CreateBankingDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php)
- 币种换算: [Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php)
- 单据模型: [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407)
- 历史记录: [CreateDocumentHistory.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentHistory.php)
- 外部入口: [PaymentController.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L170-L180)
- 事件监听器: [CreateDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/CreateDocumentTransaction.php)
- 通知监听器: [SendDocumentPaymentNotification.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/SendDocumentPaymentNotification.php)
- 事件注册: [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L73-L76)
- 测试用例: [DocumentTransactionsTest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Banking/DocumentTransactionsTest.php)

---

## 1. 请求字段默认值补齐逻辑

### `prepareRequest()` 方法 ([L51-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L51-L72))

字段缺失判断使用了三种不同的 PHP 判定方式，对默认值回退有微妙影响：

| 字段 | 判定方式 | 源码 | 缺失时的默认值 | 判定差异影响 |
|------|---------|------|---------------|-------------|
| **amount** | `!isset($this->request['amount'])` | [L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L53) | `$this->model->amount - $this->model->paid_amount` | `isset` 对 key 不存在或 `null` 返回 `false`；空字符串 `""`、`0`、`"0"` 均视为"已设置"，不回退 |
| **currency_code** | `!empty($this->request['currency_code'])` | [L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L60) | `$this->model->currency_code` | `empty` 对空字符串、`null`、`"0"`、`0`、`false` 均视为"空"，全部回退到默认 |
| **currency_rate** | `isset($this->request['currency_rate'])` | [L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L65) | `currency($currency_code)->getRate()` | 同 amount：key 存在且非 null 即视为已设置 |
| **account_id** | `isset($this->request['account_id'])` | [L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L66) | `setting('default.account')` | 同 amount |
| **paid_at** | `isset($this->request['paid_at'])` | [L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L64) | `Date::now()->toDateTimeString()` | 同 amount |
| **document_id** | `isset($this->request['document_id'])` | [L67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L67) | `$this->model->id` | 同 amount |
| **contact_id** | `isset($this->request['contact_id'])` | [L68](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L68) | `$this->model->contact_id` | 同 amount |
| **category_id** | `isset($this->request['category_id'])` | [L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L69) | `$this->model->category_id` | 同 amount |
| **payment_method** | `isset($this->request['payment_method'])` | [L70](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L70) | `setting('default.payment_method')` | 同 amount |
| **notify** | `isset($this->request['notify'])` | [L71](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L71) | `0` | 同 amount |
| **company_id** | 直接赋值 | [L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L62) | `$this->model->company_id` | 无条件覆盖 |

### amount 补齐的特殊流程

当 `amount` 缺失（key 不存在或为 `null`）时，会先触发 `PaidAmountCalculated` 事件：

```php
$this->model->paid_amount = $this->model->paid;  // 触发 getPaidAttribute()
event(new PaidAmountCalculated($this->model));
$this->request['amount'] = $this->model->amount - $this->model->paid_amount;
```

`getPaidAttribute()` ([Document.php L373-L407](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407)) 会遍历所有已关联的 transactions，将不同币种的交易金额通过 `convertBetween()` 换算为单据币种后累加。

---

## 2. 币种不同时 checkAmount 的超额支付判定与精度

### `checkAmount()` 方法 ([L74-L123](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123))

### 判定流程

```
交易币种精度 (transaction_precision)
    ↓
round(交易金额, transaction_precision)          ← L82: 按交易币种精度舍入
    ↓
币种不同? → 是 → convertBetween() 换算为单据币种
    ↓            ↓
    ↓         round(换算结果, document_precision)  ← L87: 按单据币种精度舍入
    ↓            ↓
    +------------+
    ↓
bccomp(amount, total_amount, document_precision)  ← L98: 以单据币种精度比较
```

### `convertBetween()` 实现 ([Currencies.php L43-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54))

```php
public function convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)
{
    $default_amount = $amount;
    if ($from_code != default_currency()) {
        $default_amount = $this->convertToDefault($amount, $from_code, $from_rate);  // divide
    }
    $converted_amount = $this->convertFromDefault($default_amount, $to_code, $to_rate);  // multiply
    return $converted_amount;
}
```

**关键确认**: `convertFromDefault()` ([Currencies.php L36-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L36-L41)) 使用 **`multiply`**，`convertToDefault()` ([Currencies.php L29-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L29-L34)) 使用 **`divide`**。

汇率约定：`currency_rate` 表示"1 单位默认货币 = X 单位该货币"。例如默认 USD，JPY rate = 149.50 表示 1 USD = 149.50 JPY。

### 关键精度点

1. **交易金额舍入** ([L82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L82)): `round($this->request['amount'], $transaction_precision)` — 按 **交易币种精度** 舍入
2. **币种换算**: `convertBetween()` 以默认币种为桥接：
   - 交易币种 → 默认币种: `convertToDefault()` = `divide` by 交易汇率
   - 默认币种 → 单据币种: `convertFromDefault()` = `multiply` by 单据汇率
3. **换算后舍入** ([L87](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L87)): `round($converted_amount, $document_precision)` — 按 **单据币种精度** 舍入
4. **应付金额计算** ([L93](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L93)): `round($this->model->amount - $this->model->paid_amount, $document_precision)`
5. **比较精度** ([L98](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L98)): `bccomp($amount, $total_amount, $document_precision)` — 以 **单据币种精度** 为比较精度

### 超额判定

当 `bccomp($amount, $total_amount, $document_precision) === 1` 时判定为超额支付。

错误消息中的金额会反向换算回交易币种并按交易精度舍入显示 ([L100-L115](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L100-L115))。

---

## 3. compare 三种状态的行为

`compare = bccomp($amount, $total_amount, $document_precision)` 于 [L98](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L98)

| compare 值 | 含义 | 状态设置 | 异常行为 |
|-----------|------|---------|---------|
| **1** | `amount > total_amount` (超额) | **不设置**，不修改 `$this->model->status` | **在 `DB::transaction()` 之前** 抛出 `\Exception`（消息为 `messages.error.over_payment`），流程终止。Transaction 和 History **均未创建**，无事务回滚需求 |
| **0** | `amount == total_amount` (全额) | `$this->model->status = 'paid'` ([L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119)) | 正常进入事务，创建 Transaction、保存 Document（状态 paid）、创建 History |
| **-1** | `amount < total_amount` (部分) | `$this->model->status = 'partial'` ([L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119)) | 正常进入事务，创建 Transaction、保存 Document（状态 partial）、创建 History |

### 代码片段

```php
$compare = bccomp($amount, $total_amount, $document_precision);  // L98

if ($compare === 1) {
    // ... 构造错误消息 ...
    throw new \Exception($message);  // L117: 在 DB::transaction() 之前抛出
} else {
    $this->model->status = ($compare === 0) ? 'paid' : 'partial';  // L119
}
```

---

## 4. 事务内外事件与 History 创建顺序

### 外层完整链路

```
PaymentController::dispatchPaidEvent()  [PaymentController.php L170-L180]
    ↓ 构造 request (amount=invoice->amount, type='income', payment_method=alias 等)
    ↓
event(new PaymentReceived($invoice, $request))  [PaymentReceived.php]
    ↓
    ├── Listener 1: CreateDocumentTransaction::handle()  [CreateDocumentTransaction.php L20-50]
    │     ↓ try {
    │       dispatch(new CreateBankingDocumentTransaction($document, $request))
    │         ↓
    │       CreateBankingDocumentTransaction::handle()  [主流程，见下文]
    │     ↓ } catch (\Exception $e) {
    │         portal/signed 用户: flash 错误消息，返回 redirect 响应
    │         admin 用户: 重新抛出 \Exception
    │       }
    │
    └── Listener 2: SendDocumentPaymentNotification::handle()  [SendDocumentPaymentNotification.php L16-42]
          ↓ 仅当 type === 'income'
          ↓ 读取 $document->transactions()->latest()->first()
          ↓ 通知 contact + company users
```

**监听器执行顺序**（由 [Event.php L73-L76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L73-L76) 注册）: `CreateDocumentTransaction` 先于 `SendDocumentPaymentNotification`。Laravel 按数组顺序同步执行。

**异常传播注意**: 如果 `CreateDocumentTransaction` listener 中发生超额支付异常且被 catch 后返回（portal/signed 用户），`SendDocumentPaymentNotification` 仍会执行，但 `$document->transactions()->latest()->first()` 会返回上一笔交易（或 null），可能导致通知与实际支付不匹配。

### CreateBankingDocumentTransaction::handle() 内部时序

```
handle() 入口  [L30-L49]
    ↓
[事务外] event(DocumentTransactionCreating)  ← L32: 所有操作之前
    ↓
prepareRequest()  ← L34: 补齐默认值
    ↓
checkAmount()  ← L36: 金额校验
    ↓  (如 compare=1，在此处抛出异常，后续代码不执行)
    ↓
\DB::transaction(function () {  ← L38: 事务开始
    ↓
    dispatch(new CreateTransaction($this->request))  ← L39
      └─ 内部嵌套事务/事件:
          event(TransactionCreating)    [CreateTransaction.php L18]
          Transaction::create()         [CreateTransaction.php L32]
          dispatch(CreateTransactionTaxes)  [CreateTransaction.php L43]
          createRecurring()             [CreateTransaction.php L46]
          event(TransactionCreated)     [CreateTransaction.php L49]
    ↓
    $this->model->save()  ← L41: 保存单据（status 已在 checkAmount 中设置）
    ↓
    dispatch(new CreateDocumentHistory(...))  ← L43: 创建历史记录
    ↓
})  ← 事务提交
    ↓
[事务外] event(DocumentTransactionCreated)  ← L46: 事务提交后
    ↓
return $this->transaction  ← L48
```

### 关键时序点

1. **`DocumentTransactionCreating`** ([L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L32)): 事务外、所有操作之前触发，可用于修改 request
2. **`TransactionCreated`** 时序 ([CreateTransaction.php L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransaction.php#L49)): 在 `CreateTransaction::handle()` 内部的 `\DB::transaction()` 结束后立即触发；但由于 `CreateTransaction` 是在 `CreateBankingDocumentTransaction` 外层 `\DB::transaction()` 中 dispatch 的，所以 `TransactionCreated` 仍发生在外层事务提交之前，并且早于 `$this->model->save()` 和 `CreateDocumentHistory`
3. **超额支付异常位置** ([L117](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L117)): `checkAmount()` 在 `DB::transaction()` **之前** 执行，因此超额支付不产生 Transaction 或 History，无事务回滚
4. **事务内顺序**: `CreateTransaction`（含其嵌套事务及 `TransactionCreated`）→ `model->save()` → `CreateDocumentHistory`
5. **`DocumentTransactionCreated`** ([L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L46)): 外层事务提交后触发，此时 Transaction 和 History 均已持久化

### CreateDocumentHistory 写入字段

`CreateDocumentHistory::handle()` ([CreateDocumentHistory.php L31-L47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentHistory.php#L31-L47)) 通过 `DocumentHistory::create()` 写入以下字段：

| 字段 | 值 | 来源 |
|------|-----|------|
| `company_id` | `$this->document->company_id` | 单据所属公司 |
| `type` | `$this->document->type` | 单据类型 (invoice/bill) |
| `document_id` | `$this->document->id` | 关联单据 ID |
| **`status`** | **`$this->document->status`** | **单据当前状态（`paid` 或 `partial`），在 checkAmount 中设置后保存** |
| `notify` | `$this->notify` | 构造参数传入，默认 `0` |
| **`description`** | **`money((double)$this->transaction->amount, (string)$this->transaction->currency_code)->format() . ' ' . 'payment'`** | **由 createHistory() ([L125-L130](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L125-L130)) 构造，格式为"交易币种格式化金额 + payments"** |
| `created_from` | `$this->getCustomSourceName()` | 来源标识 |
| `created_by` | `user_id()` | 当前用户 ID |

**关键说明**: History 记录的 `status` 字段捕获了单据在该次支付后的状态（`paid` 或 `partial`），而不仅仅是金额描述文本。这使得可以通过历史记录追踪每次支付后单据状态的变化。

---

## 5. rounding / bccomp 边界例子

### 问题背景

`bccomp` 是任意精度数字字符串比较（来自 bcmath 扩展），在本流程中调用 `bccomp($amount, $total_amount, $document_precision)` 按单据币种精度逐位比较两个操作数；而 `round()` 是浮点数舍入。当跨币种换算后出现无限循环小数时，舍入方向和精度位的微小差异可能导致模型判断失误。

### convertFromDefault() 行为确认

已确认 `convertFromDefault($amount, $to, $rate)` 实现为：

```php
// Currencies.php L36-L41
return $this->convert('multiply', $amount, $default_currency, $to, $rate);
```

即：**默认币种金额 × 目标币种汇率 = 目标币种金额**。汇率约定为"1 单位默认货币 = X 单位该货币"。

### 边界场景：跨币种精度不匹配导致无法精确支付

**场景**: 默认币种 USD，单据 JPY（精度 0），支付 EUR（精度 2）

- 单据金额: 100 JPY，汇率 149.50（1 USD = 149.50 JPY）
- 支付币种: EUR，汇率 0.9200（1 USD = 0.9200 EUR）

**换算公式**:

```
EUR → USD: divide by 0.9200   (convertToDefault)
USD → JPY: multiply by 149.50 (convertFromDefault)

amount_jpy = amount_eur / 0.9200 * 149.50
           = amount_eur * (149.50 / 0.9200)
           = amount_eur * 162.5
```

**寻找精确支付金额**:

目标: `amount_eur * 162.5 = 100` → `amount_eur = 100 / 162.5 = 0.615384615...`

由于 EUR 精度为 2，可用值只能是 `0.61` 或 `0.62`：

```
支付 0.61 EUR:
  0.61 / 0.9200 = 0.663043478... USD
  0.663043478 * 149.50 = 99.125543... JPY
  round(99.125543, 0) = 99 JPY
  bccomp("99", "100", 0) = -1  → status: partial

支付 0.62 EUR:
  0.62 / 0.9200 = 0.673913043... USD
  0.673913043 * 149.50 = 100.743478... JPY
  round(100.743478, 0) = 101 JPY
  bccomp("101", "100", 0) = 1  → 超额支付，抛出异常
```

**结论**: 不存在 EUR 金额（精度 2）能恰好支付 100 JPY 单据。0.61 EUR 是部分支付，0.62 EUR 超额。

### 模型易误判的陷阱

1. **直觉 vs 系统行为**: 模型可能认为"0.62 EUR 很接近 100 JPY，应该算部分支付"，但系统严格舍入后是 101 > 100，判定超额并抛出异常。

2. **忽略舍入步骤**: 模型可能直接比较浮点数 `100.743... > 100` 得出"超额"结论（这个碰巧对了），但如果遇到 `99.999...` 被舍入为 100 的情况，模型会判断错误。

3. **汇率方向混淆**: 模型可能错误地将 `convertFromDefault` 理解为 `divide` 而非 `multiply`，导致换算方向完全相反。例如将 `0.67 * 149.50` 算成 `0.67 / 149.50`，得出完全不同的金额。

4. **精度不匹配导致的"无法支付"**: 模型可能假设"总有一个金额能精确支付"，但由于不同币种精度不同（如 JPY 精度 0 vs EUR 精度 2），在特定汇率组合下可能不存在精确解。

### 另一个边界例子：舍入临界点

**场景**: 默认 USD，单据 JPY（精度 0），金额 100 JPY，汇率 150.00，支付 USD（精度 2）

```
支付 0.67 USD:
  convertFromDefault(0.67, JPY, 150.00) = 0.67 * 150.00 = 100.50 JPY
  round(100.50, 0) = 101  → bccomp("101", "100", 0) = 1  → 超额!

支付 0.66 USD:
  convertFromDefault(0.66, JPY, 150.00) = 0.66 * 150.00 = 99.00 JPY
  round(99.00, 0) = 99  → bccomp("99", "100", 0) = -1  → partial

支付 0.666 USD:  (不会被拒绝，而是先按交易币种精度舍入)
  checkAmount 中 round(0.666, 2) = 0.67  [CreateBankingDocumentTransaction.php L82]
  → 与 0.67 USD 等价，最终走到 over payment 分支，抛出异常
```

模型可能认为 0.67 USD ≈ 100.50 JPY "差不多就是 100"，但系统严格舍入为 101 并判定超额。

---

## 总结流程图

```
外部入口 (PaymentController / API 等)
  ↓
event(PaymentReceived)  [事件注册: Event.php L73-L76]
  ├─ CreateDocumentTransaction listener → dispatch(CreateBankingDocumentTransaction)
  │     ↓
  │   [事务外] event(DocumentTransactionCreating)  ← L32
  │     ↓
  │   prepareRequest()  ← L34
  │     ├─ isset/!empty 判定缺失字段
  │     ├─ amount: 单据总额 - 已付金额 (触发 PaidAmountCalculated)
  │     ├─ currency_code: 单据币种
  │     ├─ currency_rate: 系统当前汇率
  │     └─ account_id: 默认账户
  │     ↓
  │   checkAmount()  ← L36 (在 DB::transaction 之前)
  │     ├─ 按交易精度舍入交易金额
  │     ├─ convertBetween: divide(交易→默认) → multiply(默认→单据)
  │     ├─ 按单据精度舍入换算结果
  │     └─ bccomp(amount, total, 单据精度):
  │         1 → 抛 \Exception (无事务/记录产生)
  │         0 → status = 'paid'
  │        -1 → status = 'partial'
  │     ↓
  │   \DB::transaction()  ← L38
  │     ├─ CreateTransaction (嵌套事务 + TransactionCreating/Created)
  │     ├─ model->save()  (持久化 status 变更)
  │     └─ CreateDocumentHistory (写入 status + description)
  │     ↓
  │   [事务外] event(DocumentTransactionCreated)  ← L46
  │
  └─ SendDocumentPaymentNotification listener (仅 income 类型)
        → 通知 contact + company users
```
