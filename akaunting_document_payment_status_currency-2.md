# CreateBankingDocumentTransaction 支付流程分析

## 核心文件

- 主流程: [CreateBankingDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php)
- 币种换算: [Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)
- 单据模型: [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407)
- 历史记录: [CreateDocumentHistory.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentHistory.php)
- 测试用例: [DocumentTransactionsTest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Banking/DocumentTransactionsTest.php)

---

## 1. 请求字段默认值补齐逻辑

### `prepareRequest()` 方法 ([L51-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L51-L72))

| 字段 | 缺失时的默认值 | 来源 |
|------|---------------|------|
| **amount** | `$this->model->amount - $this->model->paid_amount` | 通过 `getPaidAttribute()` 计算已支付金额，再用单据总额减去已付金额得到剩余应付 |
| **currency_code** | `$this->model->currency_code` | 直接使用单据本身的币种 |
| **currency_rate** | `currency($currency_code)->getRate()` | 从系统中读取该币种当前汇率 |
| **account_id** | `setting('default.account')` | 系统设置中的默认账户 |
| **paid_at** | `Date::now()->toDateTimeString()` | 当前时间 |
| **document_id** | `$this->model->id` | 关联的单据 ID |
| **contact_id** | `$this->model->contact_id` | 单据的往来单位 |
| **category_id** | `$this->model->category_id` | 单据的分类 |
| **payment_method** | `setting('default.payment_method')` | 系统设置中的默认支付方式 |
| **notify** | `0` | 不发送通知 |
| **company_id** | `$this->model->company_id` | 单据所属公司 |

### amount 补齐的特殊流程

当 `amount` 缺失时，会先触发 [PaidAmountCalculated](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Document/PaidAmountCalculated.php) 事件：

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
round(交易金额, transaction_precision)
    ↓
币种不同? → 是 → convertBetween() 换算为单据币种
    ↓            ↓
    ↓         round(换算结果, document_precision)
    ↓            ↓
    +------------+
    ↓
bccomp(amount, total_amount, document_precision)
```

### 关键精度点

1. **交易金额舍入**: 先按 **交易币种精度** 舍入 (`line 82`)
2. **币种换算**: `convertBetween()` 通过默认币种作为中间桥接 ([Currencies.php L43-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)):
   - 交易币种 → 默认币种 (divide by 交易汇率)
   - 默认币种 → 单据币种 (multiply by 单据汇率)
3. **换算后舍入**: 按 **单据币种精度** 舍入 (`line 87`)
4. **应付金额计算**: `round($this->model->amount - $this->model->paid_amount, document_precision)` (`line 93`)
5. **比较精度**: `bccomp($amount, $total_amount, $document_precision)` (`line 98`) —— **以单据币种精度为比较精度**

### 超额判定

当 `bccomp(amount, total_amount, document_precision) === 1` 时判定为超额支付。

**注意**: 错误消息中的金额会反向换算回交易币种并按交易精度舍入显示 (`line 103-113`)。

---

## 3. compare 三种状态的行为

`compare = bccomp($amount, $total_amount, $document_precision)`

| compare 值 | 含义 | 状态设置 | 异常行为 |
|-----------|------|---------|---------|
| **1** | `amount > total_amount` | 不设置 | **抛出 `\Exception`**，消息为 `messages.error.over_payment`，流程终止，事务回滚 |
| **0** | `amount == total_amount` | `$this->model->status = 'paid'` | 正常继续，事务内保存单据、创建交易记录和历史 |
| **-1** | `amount < total_amount` | `$this->model->status = 'partial'` | 正常继续，事务内保存单据、创建交易记录和历史 |

### 代码片段

```php
if ($compare === 1) {
    // 抛出异常，终止整个流程
    throw new \Exception($message);
} else {
    // compare 为 0 或 -1 都进入此分支
    $this->model->status = ($compare === 0) ? 'paid' : 'partial';
}
```

---

## 4. 事务内外事件与 History 创建顺序

### 整体时序

```
handle() 入口
    ↓
[事务外] event(DocumentTransactionCreating)  ← 事件1: 创建前
    ↓
prepareRequest()
    ↓
checkAmount()  → 异常时直接抛出，不进入事务
    ↓
\DB::transaction(function () {  ← 事务开始
    ↓
    dispatch(CreateTransaction)  → 内部有自己的嵌套事务/事件:
        event(TransactionCreating)
        Transaction::create()
        dispatch(CreateTransactionTaxes)
        createRecurring()
        event(TransactionCreated)
    ↓
    $this->model->save()  ← 保存单据（状态已在 checkAmount 中设置）
    ↓
    dispatch(CreateDocumentHistory)  ← 历史记录在事务内创建
    ↓
})  ← 事务提交
    ↓
[事务外] event(DocumentTransactionCreated)  ← 事件2: 创建后
    ↓
return $this->transaction
```

### 关键点

1. **`DocumentTransactionCreating`**: 事务外、所有操作之前触发，可用于修改 request
2. **事务内顺序**: `CreateTransaction` → `model->save()` → `CreateDocumentHistory`
3. **`DocumentTransactionCreated`**: 事务提交后触发，此时交易和历史都已持久化
4. **历史记录内容**: 由 `createHistory()` ([L125-L130](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L125-L130)) 使用 `$this->transaction->amount`（交易币种金额）格式化描述

---

## 5. rounding / bccomp 边界例子

### 问题背景

`bccomp` 是二进制字符串比较，要求两个操作数在指定精度下完全一致；而 `round()` 是浮点数舍入。当跨币种换算后出现无限循环小数时，舍入方向和精度位的微小差异可能导致模型判断失误。

### 边界场景

**场景**: 单据币种 JPY（精度 0），交易币种 USD（精度 2），默认币种 USD。

- 单据金额: 100 JPY，汇率 0.0067114（1 JPY = 0.0067114 USD）
- 支付金额: 0.67 USD，汇率 1.0（USD 对 USD）

**换算过程**:

```
交易金额 (USD): 0.67 (精度 2，round 后仍是 0.67)
    ↓
convertBetween(0.67, USD, 1.0, JPY, 0.0067114)
    ↓
convertToDefault(0.67, USD, 1.0) = 0.67 (币种相同不换算)
    ↓
convertFromDefault(0.67, JPY, 0.0067114)
    = 0.67 / 0.0067114 ≈ 99.83013976...
    ↓
round(99.83013976, 0) = 100  ← 四舍五入到整数
    ↓
应付金额: round(100 - 0, 0) = 100
    ↓
bccomp("100", "100", 0) = 0  → 状态: paid ✓
```

**但如果支付金额是 0.66 USD**:

```
convertFromDefault(0.66, JPY, 0.0067114)
    = 0.66 / 0.0067114 ≈ 98.34013797...
    ↓
round(98.34013797, 0) = 98
    ↓
bccomp("98", "100", 0) = -1  → 状态: partial
```

**模型易误判的陷阱**:

模型可能会简单地认为 `0.67 / 0.0067114 ≈ 99.83` 应该是 `partial`，但由于 `round(99.83, 0) = 100`，实际结果是 `paid`。反之，如果换算结果是 `99.4999999`，round 后是 `99`，会被判为 partial，但实际金额非常接近全额。

### 更隐蔽的浮点精度问题

```php
// 单据: USD (精度 2), 金额 100.00
// 交易: EUR (精度 2), 金额 88.50, 汇率 0.8850 (1 EUR = 0.8850 USD)

// convertBetween(88.50, EUR, 0.8850, USD, 1.0)
// = (88.50 / 0.8850) * 1.0 = 100.0
// 但由于浮点运算:
88.50 / 0.8850 = 99.99999999999999...  ← 浮点精度损失

round(99.99999999999999, 2) = 100.00  ← PHP round 处理正确
bccomp("100.00", "100.00", 2) = 0  → paid ✓
```

但如果 `round()` 行为在某些 PHP 版本/环境下不同，或者 `convertBetween` 返回的是字符串而不是浮点数，可能导致比较结果不符合预期。

---

## 总结流程图

```
请求进入
  ├─ 缺失字段补齐 (prepareRequest)
  │    ├─ amount = 单据总额 - 已付金额 (触发 PaidAmountCalculated)
  │    ├─ currency_code = 单据币种
  │    ├─ currency_rate = 系统当前汇率
  │    └─ account_id = 默认账户
  ├─ 金额校验 (checkAmount)
  │    ├─ 按交易精度舍入交易金额
  │    ├─ 跨币种换算为单据币种
  │    ├─ 按单据精度舍入换算结果
  │    └─ bccomp 比较: 1→抛异常, 0→paid, -1→partial
  ├─ 数据库事务
  │    ├─ 创建 Transaction (内部触发 TransactionCreating/Created)
  │    ├─ 保存 Document (更新 status)
  │    └─ 创建 DocumentHistory
  └─ 触发 DocumentTransactionCreated
```
