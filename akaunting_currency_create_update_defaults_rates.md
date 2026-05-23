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
| `default_currency` | 当前是默认货币 |

---

## 3. scopeCode 与业务链路使用

### 3.1 scopeCode 定义

[Currency.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Currency.php#L122-L125)

```php
public function scopeCode($query, $code)
{
    return $query->where($this->qualifyColumn('code'), $code);
}
```

### 3.2 业务链路中的使用

**在 SettingController 中设置默认货币：**

[SettingController.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/SettingController.php#L109-L119)

```php
if ($real_key == 'default.currency') {
    $currencies = Currency::enabled()->pluck('code')->toArray();

    if (!in_array($value, $currencies)) {
        continue;
    }

    $currency = Currency::code($value)->first();
    $currency->rate = '1';
    $currency->save();
}
```

### 3.3 Model 关联关系

Currency 通过 `code` 字段与多个业务模型关联：

| 模型 | 关联方法 | 外键 |
|------|----------|------|
| [Account](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Account.php#L49-L52) | `currency()` | `currency_code` |
| [Transaction](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L131-L134) | `currency()` | `currency_code` |
| [Document](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L128-L131) | `currency()` | `currency_code` |
| [Contact](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Contact.php#L138-L141) | `currency()` | `currency_code` |

### 3.4 Helper 函数

[helpers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L247-L255)

```php
function default_currency(): string
{
    return setting('default.currency');
}
```

---

## 4. rate 为 0 或禁用货币的潜在影响

### 4.1 Request 层防护

Request 中 `rate` 规则为 `gt:0`，**正常流程下 rate 不可能为 0**。

但如果绕过校验（如直接操作数据库、API 调用跳过 Request），可能引发以下问题。

### 4.2 对 CreateTransfer 的影响

[CreateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L124-L133)

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

**风险点：**

1. 转换时使用 `convertBetween` 进行汇率换算
2. [Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54) 中 `convertToDefault` 使用 `divide` 操作

```php
public function convert($method, $amount, $from, $to, $rate, $format = false)
{
    // ...
    try {
        $money = $money->$method((double) $rate);
    } catch (\Throwable $e) {
        report($e);
        return 0;
    }
}
```

**rate = 0 后果：**
- `divide(0)` 会抛出异常，被捕获后返回 `0`
- 转出金额变为 0，导致对账不一致
- 异常被 `report($e)` 记录但不中断流程

### 4.3 对 CreateBankingDocumentTransaction 的影响

[CreateBankingDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L65)

```php
$this->request['currency_rate'] = isset($this->request['currency_rate']) 
    ? $this->request['currency_rate'] 
    : currency($currency_code)->getRate();
```

**风险点：**

1. 从货币表获取汇率存入交易记录
2. [checkAmount()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123) 中使用 `convertBetween` 比较支付金额
3. rate = 0 会导致转换金额异常，可能触发 `over_payment` 错误或金额计算错误

### 4.4 禁用货币的影响

1. **创建/更新时保护**：存在关联记录的货币无法禁用
2. **已禁用货币**：`Currency::enabled()` scope 会过滤掉，但历史数据仍通过 `code` 关联
3. **新建交易**：禁用货币不会出现在下拉选项中，但 API 直接调用可能绕过
4. **报表统计**：货币禁用不影响已存交易的金额计算（交易记录中保存了当时的 `currency_rate`）

---

## 5. 汇率更新后旧交易不自动重算的验证思路

### 5.1 核心设计原理

**快照机制**：[Document](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L45) 和 [Transaction](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L46) 模型均有 `currency_rate` 字段，**创建时保存当时的汇率快照**。

### 5.2 验证方案

**步骤 1：准备测试数据**

```php
// 1. 创建货币 USD，rate = 7.0
$usd = Currency::create([
    'company_id' => 1,
    'name' => 'US Dollar',
    'code' => 'USD',
    'rate' => 7.0,
    'enabled' => 1,
]);

// 2. 创建一笔 USD 发票（金额 100）
$invoice = Document::create([
    'type' => 'invoice',
    'currency_code' => 'USD',
    'currency_rate' => 7.0,  // 快照保存
    'amount' => 100,
    // ...
]);

// 记录当时的默认货币等值: 100 * 7.0 = 700
```

**步骤 2：更新汇率**

```php
// 更新 USD 汇率为 7.2
$usd->update(['rate' => 7.2]);
```

**步骤 3：验证旧数据不变**

```php
$invoice = Document::find($invoice->id);

// 断言: 发票保存的汇率快照不变
$this->assertEquals(7.0, $invoice->currency_rate);

// 断言: 发票金额不变
$this->assertEquals(100, $invoice->amount);

// 断言: paid 计算使用保存的汇率
$paid = $invoice->paid; // 内部使用 $this->currency_rate 转换
```

**步骤 4：验证新数据使用新汇率**

```php
// 创建新发票
$newInvoice = Document::create([
    'type' => 'invoice',
    'currency_code' => 'USD',
    'currency_rate' => 7.2,  // 新汇率
    'amount' => 100,
]);

$this->assertEquals(7.2, $newInvoice->currency_rate);
```

### 5.3 关键验证点

| 验证场景 | 预期结果 | 代码位置 |
|----------|----------|----------|
| 发票 `paid` 属性计算 | 使用 `$this->currency_rate` 而非 `currency($code)->getRate()` | [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L385-L403) |
| 交易金额转换 | 使用 `$this->currency_rate` | [Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L354-L380) |
| 报表统计 | 不重新计算历史数据 | 需检查报表相关代码 |

### 5.4 反模式验证（确认不触发重算）

验证 `CurrencyUpdated` 事件没有监听器触发批量更新：

```php
// 搜索监听 CurrencyUpdated 的 Listener
// 结果: 无默认监听器，证明汇率更新不会触发交易重算
```

---

## 总结

1. **校验严格**：Request 层确保 `rate > 0`，Update 时保护有关联的货币不被禁用或改 code
2. **默认货币特殊处理**：设为默认时强制 `rate = 1`，并更新 `setting('default.currency')`
3. **快照设计**：交易/单据创建时保存 `currency_rate` 快照，后续汇率更新不影响历史数据
4. **转换逻辑**：`convertBetween` 通过默认货币作为中间货币进行双向转换
5. **潜在风险**：绕过 Request 校验导致 rate = 0 会引发除零异常，虽被捕获但返回 0 可能导致数据不一致
