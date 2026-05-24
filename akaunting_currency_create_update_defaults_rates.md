# Akaunting Currency 创建与更新机制分析

## 1. Currency Request 校验规则

[Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Setting/Currency.php#L14-L36)

### 核心字段校验

| 字段 | 校验规则 | 说明 |
|------|----------|------|
| `code` | `required`, `string`, `currency_code`, `unique:currencies` | 货币代码唯一，公司范围内不重复 |
| `rate` | `required`, `gt:0` | **汇率必须大于 0**，不允许为 0 或负值 |
| `enabled` | `integer`, `boolean` | 启用状态，0 或 1 |
| `default_currency` | `nullable`, `boolean` | 是否设为默认货币 |

### 唯一性校验细节

```php
'code' => 'required|string|currency_code|unique:currencies,NULL,' . ($id ?? 'null') . ',id,company_id,' . $company_id . ',deleted_at,NULL'
```

- 作用域：同一 `company_id` 下，排除已软删除的记录
- 更新时排除当前货币 ID，避免自身冲突

---

## 2. Create/Update Job 事件与字段保存

### 2.1 CreateCurrency Job

[CreateCurrency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/CreateCurrency.php#L15-L37)

**事件触发顺序：**

1. `CurrencyCreating($request)` - 创建前事件
2. 数据库事务内执行创建
3. `CurrencyCreated($model, $request)` - 创建后事件

**保存字段（来自 `$fillable`）：**

[Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Currency.php#L22-L35)

```php
'company_id', 'name', 'code', 'rate', 'enabled', 'precision',
'symbol', 'symbol_first', 'decimal_mark', 'thousands_separator',
'created_from', 'created_by'
```

**特殊处理：**

```php
// 设为默认货币时强制汇率为 1
if ($this->request->get('default_currency')) {
    $this->request['rate'] = '1';
    setting()->set('default.currency', $this->request->get('code'));
    setting()->save();
}
```

### 2.2 UpdateCurrency Job

[UpdateCurrency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/UpdateCurrency.php#L13-L78)

**事件触发顺序：**

1. `authorize()` - 权限与关联检查
2. `CurrencyUpdating($model, $request)` - 更新前事件
3. 数据库事务内执行更新
4. `CurrencyUpdated($model, $request)` - 更新后事件

**授权检查逻辑：**

```php
// 存在关联记录时不允许修改 code
if ($this->model->code != $this->request->get('code')) {
    throw new \Exception($message);
}

// 存在关联记录时不允许禁用
if ($this->request->has('enabled') && !$this->request->get('enabled')) {
    throw new \Exception($message);
}
```

**关联检查范围（`getRelationships`）：**

| 关联类型 | 说明 |
|----------|------|
| `accounts` | 银行账户 |
| `customers` | 客户 |
| `invoices` | 发票 |
| `bills` | 账单 |
| `transactions` | 交易记录 |
| `default_currency` | 当前是默认货币（通过 `$this->model->code == default_currency()` 判断） |

---

## 3. Currency Model 方法与业务链路

### 3.1 scopeCode 定义与使用

[Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Currency.php#L122-L125)

```php
public function scopeCode($query, $code)
{
    return $query->where($this->qualifyColumn('code'), $code);
}
```

**业务链路使用：**

- [SettingController](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/SettingController.php#L116)：通过 `Currency::code($value)->first()` 获取待设为默认的货币实例

### 3.2 enabled() Scope（继承自 Abstract Model）

[Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L181-L184)

```php
public function scopeEnabled($query)
{
    return $query->where($this->qualifyColumn('enabled'), 1);
}
```

**业务链路使用：**

| 位置 | 代码 | 作用 |
|------|------|------|
| [Form/Group/Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/Form/Group/Currency.php#L36) | `Model::enabled()->orderBy('name')->pluck('name', 'code')` | UI 货币下拉选项只展示启用的货币 |
| [Form/Group/Account.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/Form/Group/Account.php#L71) | `Model::with('currency')->enabled()->orderBy('name')->get()` | UI 账户下拉只展示启用的账户 |
| [SettingController](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/SettingController.php#L110) | `Currency::enabled()->pluck('code')->toArray()` | 检查待设为默认的货币是否在启用列表中 |

### 3.3 default() 方法

[Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Currency.php#L110-L113)

```php
public function default()
{
    return $this->code(default_currency())->first();
}
```

- 返回默认货币的模型实例
- 内部调用 `scopeCode`，无需传入参数即可使用：`Currency::default()`

### 3.4 Model 关联关系

Currency 通过 `code` 字段与多个业务模型建立 `hasMany`/`belongsTo` 关联：

| 模型 | 关联方法 | 外键 | 方向 |
|------|----------|------|------|
| [Account](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Account.php#L49-L52) | `currency()` | `currency_code` | belongsTo |
| [Transaction](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L131-L134) | `currency()` | `currency_code` | belongsTo |
| [Document](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L128-L131) | `currency()` | `currency_code` | belongsTo |
| [Contact](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Contact.php#L138-L141) | `currency()` | `currency_code` | belongsTo |

### 3.5 Helper 函数

[helpers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L247-L255)

```php
function default_currency(): string
{
    return setting('default.currency');
}
```

- 返回默认货币代码字符串（如 `'USD'`），而非模型实例
- 在 `Currencies` trait 的 `convertBetween` 中被广泛使用

---

## 4. rate 为 0 或禁用货币的潜在影响

### 4.1 可达性分析（rate = 0 如何进入系统）

| 路径 | 是否可达 | 防护机制 |
|------|----------|----------|
| **Web/API (Currency CRUD)** | 不可达 | [Currency Request](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Setting/Currency.php#L29): `rate: required|gt:0` |
| **Web/API (Transaction 直接创建)** | 不可达 | [Transaction Request](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transaction.php#L47): `currency_rate: required|gt:0` |
| **Web/API (Transfer 创建)** | **可达（间接）** | [Transfer Request](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transfer.php#L14-L23) 不校验 `from_account_rate`/`to_account_rate`；若数据库中货币 rate=0，则 `getCurrencyRate()` fallback 返回 0 |
| **直接 dispatch Job** | 可达 | 绕过 Request 校验，直接传入任意 rate 值 |
| **事件监听器** | 核心代码不可达 | 核心代码中 `CurrencyCreated`/`CurrencyUpdated` 无注册监听器，无法修改 rate；但扩展模块可自行添加 |
| **直接数据库写入** | 可达 | `UPDATE currencies SET rate = 0` 绕过所有应用层校验 |

### 4.2 对 CreateTransfer 的影响

[CreateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L23-L100)

**汇率获取逻辑：**

```php
protected function getCurrencyRate($type)
{
    $currency_rate = $this->request->get($type . '_account_rate');

    if (empty($currency_rate)) {
        $currency_rate = currency($this->getCurrencyCode($type))->getRate();
    }

    return $currency_rate;
}
```

- `from_account_rate`/`to_account_rate` 为可选参数
- 若 Request 中未提供，则从 `currencies` 表读取当前 rate
- Transfer Request **不校验** 这两个字段的有效性

**expense 交易与 income 交易的差异：**

| 交易 | amount 来源 | rate=0 影响 |
|------|-------------|--------------|
| **expense_transaction**（转出） | `$this->request->get('amount')`（原始金额） | **不受影响**，amount 保持原始值，仅 `currency_rate` 字段存为 0 |
| **income_transaction**（转入） | 当 `expense_currency_code != income_currency_code` 时，amount = `convertBetween(...)` 结果 | **受影响**，`convertBetween` 返回 0，导致转入金额为 0 |

**convertBetween 在 rate=0 时的行为：**

[Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)

```php
public function convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)
{
    $default_amount = $amount;

    if ($from_code != default_currency()) {
        $default_amount = $this->convertToDefault($amount, $from_code, $from_rate);
        // convert('divide', amount, from, default, rate=0) → catch → return 0
    }

    $converted_amount = $this->convertFromDefault($default_amount, $to_code, $to_rate);
    // convert('multiply', 0, default, to, rate) → return 0

    return $converted_amount;
}
```

- `from_rate=0`：`convertToDefault` 中 `divide(amount, 0)` 抛异常 → 捕获返回 0 → 最终结果为 0
- `to_rate=0` 且 `from_rate≠0`：`convertToDefault` 成功 → `convertFromDefault` 中 `multiply(default_amount, 0)` = 0 → 最终结果为 0
- `from_rate=0` 且 `from_code == default_currency()`：跳过 `convertToDefault` → `convertFromDefault` 中 `multiply(amount, 0)` = 0

**rate=0 后果：**
- 转出方账户余额扣减正确（原始金额），但转入方入账为 0
- 造成账不平，两个账户余额不匹配
- `convert` 中异常被 `report($e)` 记录但不中断流程

### 4.3 对 CreateBankingDocumentTransaction 的影响

[CreateBankingDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L51-L123)

**汇率获取逻辑：**

```php
$this->request['currency_rate'] = isset($this->request['currency_rate']) 
    ? $this->request['currency_rate'] 
    : currency($currency_code)->getRate();
```

- 优先使用 Request 传入值，fallback 到 `currencies` 表
- 直接 dispatch Job 时可传入任意值

**checkAmount() 中 rate=0 的真实分支：**

```php
$amount = $this->request['amount'] = round($this->request['amount'], $transaction_precision);

if ($this->model->currency_code != $code) {
    $converted_amount = $this->convertBetween($amount, $code, $rate, $this->model->currency_code, $this->model->currency_rate);
    $amount = round($converted_amount, $document_precision);
    // rate=0 → $converted_amount=0 → $amount=0
}

$total_amount = round($this->model->amount - $this->model->paid_amount, $document_precision);
$compare = bccomp($amount, $total_amount, $document_precision);
```

| 场景 | $amount | $total_amount | bccomp 结果 | 状态 |
|------|---------|---------------|-------------|------|
| 货币不同 + rate=0 | 0 | 通常 > 0 | -1 | `partial` |
| 货币不同 + rate=0 | 0 | 0 | 0 | `paid` |
| 货币相同 + rate=0 | 原始金额 | model.amount - paid | 取决于原始金额 | 正常逻辑 |

**关键发现：货币不同且 rate=0 时，`$amount` 始终为 0，导致 `bccomp(0, total_amount) = -1`，状态强制为 `partial`。此场景下**永远不会触发 `over_payment` 异常**，即使原始支付金额远超应付金额。**

### 4.4 禁用货币的多层检查差异

| 层级 | 检查位置 | 检查逻辑 | 效果 |
|------|----------|----------|------|
| **UI - 货币下拉** | [Form/Group/Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/Form/Group/Currency.php#L36) | `Model::enabled()->pluck('name', 'code')` | 禁用货币不显示在下拉中，但已选择的禁用货币通过 `old()` 保留 |
| **UI - 账户下拉** | [Form/Group/Account.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/Form/Group/Account.php#L71) | `Model::with('currency')->enabled()->orderBy('name')->get()` | 只过滤禁用的账户，不过滤账户关联的禁用货币 |
| **Job - UpdateCurrency** | [UpdateCurrency.php authorize()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/UpdateCurrency.php#L42-L58) | 关联记录存在时禁止禁用 | 有关联（账户/客户/发票/账单/交易）的货币无法禁用；无关联时可禁用 |
| **Setting - 设为默认** | [SettingController](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/SettingController.php#L110-L111) | `in_array($value, Currency::enabled()->pluck('code'))` | 禁用货币不能被设为默认 |

**潜在风险：**
- 已禁用货币仍可能通过已关联的启用账户被间接使用（因为账户下拉不检查货币状态）
- 直接 dispatch Job 或 API 调用可绕过 UI 层过滤
- 历史交易/单据的 `currency_code` 仍指向已禁用货币，不影响金额计算（因为快照中保存了 `currency_rate`）

---

## 5. 汇率更新后旧交易不自动重算的验证思路

### 5.1 核心设计原理

**快照机制**：[Document](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L45) 和 [Transaction](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L46) 模型均有 `currency_rate` 字段，**创建时保存当时的汇率快照**。后续读取时，`paid` 属性和金额转换均使用保存的快照值，而非重新查询 `currencies` 表。

### 5.2 核心默认代码与扩展监听器的分离分析

**核心默认代码（不会触发重算）：**

- `EventServiceProvider` 中未注册 `CurrencyCreated`/`CurrencyUpdated` 的监听器
- `app/Listeners/` 目录下无监听这两个事件的类
- `Currency::update()` 仅更新 `currencies` 表的 `rate` 字段，不触发级联更新
- 搜索结果：仅在 [CreateCurrency](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/CreateCurrency.php#L17) 和 [UpdateCurrency](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/UpdateCurrency.php#L17) Job 中 fire 事件，无对应 listener

**扩展监听器（可能触发重算）：**

- 第三方模块可通过自身 ServiceProvider 注册 `CurrencyUpdated` 事件监听器
- 监听器中可执行批量更新 `documents.currency_rate` 或 `transactions.currency_rate`
- 核心代码不提供此类监听器，需自行评估模块代码

### 5.3 验证方案（基于 currency_rate 快照断言）

**步骤 1：创建货币与发票**

```php
$usd = Currency::create([
    'company_id' => 1,
    'name' => 'US Dollar',
    'code' => 'USD',
    'rate' => 7.0,
    'enabled' => 1,
]);

$invoice = Document::create([
    'type' => Document::INVOICE_TYPE,
    'currency_code' => 'USD',
    'currency_rate' => 7.0,
    'amount' => 100,
    // ...
]);
```

**步骤 2：更新货币汇率**

```php
$usd->update(['rate' => 7.2]);
```

**步骤 3：验证旧发票快照不变**

```php
$invoice->refresh();

// 关键断言：发票的 currency_rate 快照不受货币汇率更新影响
$this->assertEquals(7.0, $invoice->currency_rate);
$this->assertEquals(100, $invoice->amount);
```

**步骤 4：验证 paid 使用快照汇率**

[Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L385-L386)

```php
// $code 和 $rate 均来自 $this（Document 实例自身），而非 currency() 辅助函数
$code = $this->currency_code;
$rate = $this->currency_rate;  // 使用保存的快照
```

**步骤 5：验证交易使用快照汇率**

[Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L359-L380)

```php
// getAmountForDocumentAttribute 使用 $this->document->currency_rate
$to_rate = $this->document->currency_rate;  // 而非 currency($code)->getRate()
// convertBetween 使用 $this->currency_rate 和 $this->document->currency_rate
```

### 5.4 关键验证点

| 验证场景 | 预期结果 | 代码位置 |
|----------|----------|----------|
| `documents.currency_rate` 字段值 | 保持创建时的值（快照不变） | [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L385-L403) |
| `transactions.currency_rate` 字段值 | 保持创建时的值（快照不变） | [Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L354-L380) |
| `Document::paid` 计算 | 使用 `$this->currency_rate` 快照 | 同上 |
| `Transaction::getAmountForDocumentAttribute` | 使用 `$this->currency_rate` 和 `$this->document->currency_rate` | 同上 |
| 核心代码无 CurrencyUpdated 监听器 | `app/Listeners/` 和 `EventServiceProvider` 中无注册 | 通过代码搜索确认 |

---

## 总结

1. **校验严格**：Currency Request 确保 `rate > 0`；Transaction Request 确保 `currency_rate > 0`；Transfer Request 不校验汇率字段，但 CreateTransfer 可通过 fallback 从货币表获取
2. **默认货币特殊处理**：设为默认时强制 `rate = 1`，并更新 `setting('default.currency')`；SettingController 确保默认货币在启用列表中
3. **快照设计**：交易/单据创建时保存 `currency_rate` 快照，后续汇率更新不影响历史数据；`paid` 计算和金额转换均使用快照值
4. **转换逻辑**：`convertBetween` 通过默认货币作为中间货币进行双向转换；rate=0 时除零异常被捕获返回 0，在 CreateTransfer 中导致 income 交易金额为 0，在 CreateBankingDocumentTransaction 中强制状态为 `partial`
5. **禁用货币多层防护**：UI 层过滤显示、Job 层阻止有关联记录的禁用、Setting 层阻止设为默认；但账户组件不检查货币状态，存在间接使用禁用货币的可能
6. **扩展监听器风险**：核心代码不监听 CurrencyUpdated 进行级联更新，但第三方模块可能添加此类监听器
