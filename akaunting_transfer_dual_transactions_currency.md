# Akaunting 银行转账落库机制分析

## 核心架构概览

创建银行转账时，系统会在一个数据库事务中依次创建三笔记录：
1. 一笔 `expense-transfer` 类型的 Transaction（转出方）
2. 一笔 `income-transfer` 类型的 Transaction（转入方）
3. 一笔 Transfer 记录，关联上述两笔 Transaction

---

## 1. from/to 账户不存在时在哪一层报错

**答案：在 Job 层的 `authorize()` 方法中报错。**

具体位置：
- [CreateTransfer.php:102-111](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L102-L111)
- [UpdateTransfer.php:99-108](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransfer.php#L99-L108)

```php
public function authorize(): void
{
    foreach (['from', 'to'] as $type) {
        $account_id = $this->request->get($type . '_account_id');

        if (empty($account_id) || ! Account::find($account_id)) {
            throw new \Exception(trans('messages.error.not_found', ['type' => trans_choice('general.accounts', 1)]));
        }
    }
}
```

**注意**：虽然 [Transfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transfer.php) 请求验证层也对 `from_account_id` 和 `to_account_id` 做了 `required|integer` 验证，但**不验证账户是否真实存在**。存在性检查发生在 Job 层。

---

## 2. 两侧 currency_code 与 currency_rate 如何取值，跨币种时只改哪一侧 amount

### 2.1 currency_code 取值逻辑

方法：[getCurrencyCode()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L113-L122)

```php
protected function getCurrencyCode($type)
{
    $currency_code = $this->request->get($type . '_account_currency_code');

    if (empty($currency_code)) {
        $currency_code = Account::where('id', $this->request->get($type . '_account_id'))->pluck('currency_code')->first();
    }

    return $currency_code;
}
```

**取值优先级**：
1. 优先取请求参数：`from_account_currency_code` / `to_account_currency_code`
2. 若请求未传，则从 Account 表中查询对应账户的 `currency_code`

**UI 字段名与 Job 读取名不匹配**：

查看 [create.blade.php:26-34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/create.blade.php#L26-L34)，普通 Web UI 的隐藏字段是：

```blade
<x-form.input.hidden name="from_currency_code" v-model="form.from_currency_code" />
<x-form.input.hidden name="to_currency_code" v-model="form.to_currency_code" />
```

即 UI 提交的参数名是 `from_currency_code` / `to_currency_code`（无 `_account_` 前缀），与 Job 中读取的 `from_account_currency_code` / `to_account_currency_code`（有 `_account_` 前缀）**不一致**。

再看前端 JS [transfers.js](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/banking/transfers.js)：

[onChangeFromAccount()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/banking/transfers.js#L51-L84) 中，L73-L74 把接口返回值写入：
```js
this.form.from_currency_code = response.data.currency_code;   // L73
this.form.from_account_rate = response.data.currency_rate;      // L74
```

[onChangeToAccount()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/banking/transfers.js#L86-L114) 中，L103-L104 把接口返回值写入：
```js
this.form.to_currency_code = response.data.currency_code;      // L103
this.form.to_account_rate = response.data.currency_rate;         // L104
```

JS 写入的字段名与 Blade 隐藏字段的 `name` 属性完全一致（`from_currency_code`/`from_account_rate`/`to_currency_code`/`to_account_rate`），但 `getCurrencyCode()` 读取的 key 是 `$type . '_account_currency_code'`，即 `from_account_currency_code`/`to_account_currency_code`，**多了 `_account_` 前缀**，导致从 UI 提交时第一分支永远取不到值。

因此：
- **正常 UI 创建时**：由于字段名不匹配，`getCurrencyCode()` 第一分支取不到值，回退到从 Account 表查询 `currency_code`
- **API 或直接构造 Job 时**：如果调用方传入 `from_account_currency_code` / `to_account_currency_code`，则可以覆盖默认行为

对比 `getCurrencyRate()`：[CreateTransfer.php:124-133](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L124-L133) 读取的是 `$type . '_account_rate'`，而 UI 中 [create.blade.php:28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/create.blade.php#L28) 和 [create.blade.php:34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/create.blade.php#L34) 的字段名也是 `from_account_rate` / `to_account_rate`，**字段名是匹配的**。

### 2.2 currency_rate 取值逻辑

方法：[getCurrencyRate()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L124-L133)

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

**取值优先级**：
1. 优先取请求参数：`from_account_rate` / `to_account_rate`（与 UI 字段名匹配）
2. 若请求未传，则调用 `currency($code)->getRate()` 获取该币种当前汇率

### 2.3 跨币种时只改哪一侧 amount

**答案：只改 income 侧（转入方）的 amount。**

代码位置：[CreateTransfer.php:54-59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L54-L59)

```php
$amount = $this->request->get('amount');

// Convert amount if not same currency
if ($expense_currency_code != $income_currency_code) {
    $amount = $this->convertBetween($amount, $expense_currency_code, $expense_currency_rate, $income_currency_code, $income_currency_rate);
}
```

- **expense 侧（转出方）**：amount = 用户输入的原始金额（始终不变）
- **income 侧（转入方）**：同币种时 = 原始金额；异币种时 = 转换后的金额

### 2.4 金额转换逻辑——currency_rate 在代码中如何用于除以/乘以

核心方法：[convert()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L9-L27)

```php
public function convert($method, $amount, $from, $to, $rate, $format = false)
{
    $money = Money::$to($amount, $format);   // ① 先以目标币种构造 Money 对象，金额为原始 amount

    if ($from == $to) {
        return $format ? $money->format() : $money->getAmount();
    }

    try {
        $money = $money->$method((double) $rate);  // ② 对目标币种金额执行 method(rate)
    } catch (\Throwable $e) {
        report($e);
        return 0;
    }

    return $format ? $money->format() : $money->getAmount();
}
```

**关键**：第①步 `Money::$to($amount)` 用**目标币种**构造，金额取原始 `$amount`。第②步对这个目标币种金额做 `divide(rate)` 或 `multiply(rate)`。

两个衍生方法：

[convertToDefault()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L29-L34)（源币种 → 默认币种，用 **divide**）：
```php
return $this->convert('divide', $amount, $from, $default_currency, $rate, $format);
// 展开：Money::default($amount)->divide($rate)
// 语义：default_amount = amount / rate
```

[convertFromDefault()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L36-L41)（默认币种 → 目标币种，用 **multiply**）：
```php
return $this->convert('multiply', $amount, $default_currency, $to, $rate, $format);
// 展开：Money::to($amount)->multiply($rate)
// 语义：converted_amount = amount * rate
```

[convertBetween()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)（源币种 → 默认币种 → 目标币种）：
```php
public function convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)
{
    $default_amount = $amount;
    if ($from_code != default_currency()) {
        $default_amount = $this->convertToDefault($amount, $from_code, $from_rate);  // amount / from_rate
    }
    $converted_amount = $this->convertFromDefault($default_amount, $to_code, $to_rate); // default_amount * to_rate
    return $converted_amount;
}
```

**currency_rate 的语义**：对于非默认货币 X，其 `currency_rate = R` 表示 **1 单位默认货币 = R 单位 X**。因此：
- X 金额转默认货币：`default_amount = x_amount / R`（divide）
- 默认货币金额转 X：`x_amount = default_amount * R`（multiply）

---

## 3. 两笔 transaction 的 type、number、category_id、contact_id 分别如何设置

| 字段 | expense-transfer（转出） | income-transfer（转入） |
|------|-------------------------|------------------------|
| **type** | `expense-transfer` | `income-transfer` |
| **number** | 各自独立生成的流水号 | 各自独立生成的流水号 |
| **category_id** | 转账分类 ID（Other） | 转账分类 ID（Other） |
| **contact_id** | `0` | `0` |

### 3.1 type 常量定义

[Transaction.php:22-29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L22-L29)

```php
public const INCOME_TRANSFER_TYPE = 'income-transfer';
public const EXPENSE_TRANSFER_TYPE = 'expense-transfer';
```

### 3.2 number 生成

调用 [getNextTransactionNumber()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php#L253-L256) 生成流水号，两笔交易调用两次，生成不同的编号。

### 3.3 category_id 取值

[getTransferCategoryId()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Categories.php#L293-L299)

```php
public function getTransferCategoryId(): mixed
{
    return Cache::remember('transferCategoryId.' . company_id(), 60, function () {
        return Category::other()->pluck('id')->first();
    });
}
```

两笔交易使用**同一个**转账分类 ID。

### 3.4 contact_id

两笔交易的 `contact_id` 均硬编码为 `0`，表示没有关联的联系人。

---

## 4. Transfer 表保存什么关系，附件挂到哪里

### 4.1 Transfer 表结构

[Transfer.php:39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php#L39)

```php
protected $fillable = ['company_id', 'expense_transaction_id', 'income_transaction_id', 'created_from', 'created_by'];
```

**Transfer 表只保存关联关系和元数据**：
- `expense_transaction_id`：指向转出方 Transaction
- `income_transaction_id`：指向转入方 Transaction
- `company_id`、`created_from`、`created_by`：元数据

所有业务字段（金额、日期、描述、支付方式等）都通过 Eloquent 访问器动态获取，来源如下：

[Transfer.php:139-242](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php#L139-L242)

| 访问器属性 | 数据源 |
|-----------|--------|
| `from_account_id` | `expense_transaction->account_id` |
| `from_currency_code` | `expense_transaction->currency_code` |
| `from_account_rate` | `expense_transaction->currency_rate` |
| `to_account_id` | `income_transaction->account_id` |
| `to_currency_code` | `income_transaction->currency_code` |
| `to_account_rate` | `income_transaction->currency_rate` |
| `amount` | `expense_transaction->amount` |
| `transferred_at` | `expense_transaction->paid_at` |
| `description` | `expense_transaction->description` |
| `payment_method` | `expense_transaction->payment_method` |
| `reference` | `expense_transaction->reference` |

规律：**`from_` 前缀从 expense_transaction 取，`to_` 前缀从 income_transaction 取**。

此外，[Transfer.php:84-89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php#L84-L89) 的 `income_transaction()` 关系允许通过 `$transfer->income_transaction->amount` 直接获取收入侧金额。

### 4.2 附件挂载位置

**附件挂到 Transfer 模型上**，而不是挂到 Transaction 上。

代码位置：[CreateTransfer.php:88-94](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php#L88-L94)

```php
// Upload attachment
if ($this->request->file('attachment')) {
    foreach ($this->request->file('attachment') as $attachment) {
        $media = $this->getMedia($attachment, 'transfers');

        $this->model->attachMedia($media, 'attachment');
    }
}
```

Media collection 名称为 `'attachment'`，存储在 `transfers` 目录下。

---

## 5. 同币种和异币种例子，余额分析时不能只看 transfers 表的原因

### 5.1 同币种例子

假设默认货币为 USD，两账户币种均为 USD：

| 账户 | 币种 | rate | 操作前余额 | 转账金额 | 操作后余额 |
|------|------|------|-----------|---------|-----------|
| 账户 A（转出） | USD | 1.0 | 10,000 | -1,000 | 9,000 |
| 账户 B（转入） | USD | 1.0 | 5,000 | +1,000 | 6,000 |

**落库数据**：
- expense_transaction: amount = 1000, currency_code = USD, currency_rate = 1.0
- income_transaction: amount = 1000, currency_code = USD, currency_rate = 1.0
- $transfer->amount（访问器）= 1000
- $transfer->income_transaction->amount = 1000（通过关系直接读取）

### 5.2 异币种例子（与源码换算方向一致）

假设默认货币为 USD，CNY 的 currency_rate = 7.0（即 1 USD = 7 CNY）：

| 账户 | 币种 | rate | 操作前余额 | 转账金额 | 操作后余额 |
|------|------|------|-----------|---------|-----------|
| 账户 A（转出） | USD | 1.0 | 10,000 | -1,000 | 9,000 |
| 账户 B（转入） | CNY | 7.0 | 50,000 | +7,000 | 57,000 |

**换算过程**（追踪 [convertBetween()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)）：

```
输入：amount=1000, from_code=USD, from_rate=1.0, to_code=CNY, to_rate=7.0

Step 1: from_code(USD) == default_currency(USD)，不转换
        $default_amount = 1000

Step 2: convertFromDefault(1000, 'CNY', 7.0)
        = Money::CNY(1000)->multiply(7.0)   // 目标币种 CNY，乘以 to_rate
        = 1000 * 7.0 = 7000

输出：7000 CNY
```

**落库数据**：
- expense_transaction: amount = 1000, currency_code = USD, currency_rate = 1.0
- income_transaction: amount = 7000, currency_code = CNY, currency_rate = 7.0
- $transfer->amount（访问器）= 1000（从 expense_transaction 取）
- $transfer->income_transaction->amount = 7000（通过 income_transaction 关系可取）
- $transfer->to_currency_code = CNY（从 income_transaction 取）

**反向例子**：默认货币 USD，转出 7000 CNY 到 USD（CNY rate = 7.0）：

```
输入：amount=7000, from_code=CNY, from_rate=7.0, to_code=USD, to_rate=1.0

Step 1: from_code(CNY) != default_currency(USD)
        convertToDefault(7000, 'CNY', 7.0)
        = Money::USD(7000)->divide(7.0)    // 目标币种 USD，除以 from_rate
        = 7000 / 7.0 = 1000

Step 2: convertFromDefault(1000, 'USD', 1.0)
        = Money::USD(1000)->multiply(1.0)  // 目标币种 USD，乘以 to_rate
        = 1000 * 1.0 = 1000

输出：1000 USD
```

**落库数据**：
- expense_transaction: amount = 7000, currency_code = CNY, currency_rate = 7.0
- income_transaction: amount = 1000, currency_code = USD, currency_rate = 1.0
- $transfer->amount（访问器）= 7000（从 expense_transaction 取）
- $transfer->income_transaction->amount = 1000（通过关系可取）

### 5.3 为什么余额分析不能只看 transfers 表

**核心原因**：Transfer 表本身不存储金额数据，真实的账户余额变动记录在两笔独立的 transactions 记录中。

具体说明：

1. **Transfer 表只存关联关系**：[Transfer.php:39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php#L39) 的 `$fillable` 只有 `company_id`、`expense_transaction_id`、`income_transaction_id`、`created_from`、`created_by`，不包含任何业务金额字段。

2. **`$transfer->amount` 访问器只返回支出侧金额**：[Transfer.php:219-222](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php#L219-L222)，但收入侧金额并非"看不到"——通过 `$transfer->income_transaction->amount` 仍可获取。列表页 [index.blade.php:142-147](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/index.blade.php#L142-L147) 也是分别展示两侧金额。

3. **余额分析必须查 transactions 表**：每个账户的真实余额变动由其关联的 transactions 汇总决定。一笔转账产生两条 transactions 记录，分别作用于两个不同账户。如果只看 transfers 表（即使通过关系关联也能拿到两侧数据），也无法高效地汇总单个账户的所有收支变动——因为需要遍历所有 transfers 再分别定位两侧。直接查询 transactions 表按 `account_id` 分组汇总是标准做法。

**源码中账户余额的计算方式**：

[Account::getBalanceAttribute()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Account.php#L123-L135)：

```php
public function getBalanceAttribute()
{
    $total = $this->opening_balance;

    // Sum Incomes
    $total += $this->income_transactions->sum('amount');

    // Subtract Expenses
    $total -= $this->expense_transactions->sum('amount');

    return $total;
}
```

其中 `income_transactions` 和 `expense_transactions` 关系在 [Account.php:54-62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Account.php#L54-L62) 中通过 `whereIn('type', getIncomeTypes()) 和 `whereIn('type', getExpenseTypes()) 过滤：

```php
public function expense_transactions()
{
    return $this->transactions()->whereIn('transactions.type', (array) $this->getExpenseTypes());
}

public function income_transactions()
{
    return $this->transactions()->whereIn('transactions.type', (array) $this->getIncomeTypes());
}
```

[getIncomeTypes()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php#L84-L87) 和 [getExpenseTypes()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Transactions.php#L89-L92) 从配置读取类型集合配置：

```php
public function getIncomeTypes(string $return = 'array'): string|array
{
    return $this->getTransactionTypes(Transaction::INCOME_TYPE, $return);
}

public function getExpenseTypes(string $return = 'array'): string|array
{
    return $this->getTransactionTypes(Transaction::EXPENSE_TYPE, $return);
}
```

默认配置在 [setting.php:192-195](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/setting.php#L192-L195)：

```php
'type' => [
    'income'  => env('SETTING_FALLBACK_TRANSACTION_TYPE_INCOME',
        Transaction::INCOME_TYPE . ',' . Transaction::INCOME_TRANSFER_TYPE),
    // => 'income,income-transfer'
    'expense' => env('SETTING_FALLBACK_TRANSACTION_TYPE_EXPENSE',
        Transaction::EXPENSE_TYPE . ',' . Transaction::EXPENSE_TRANSFER_TYPE),
    // => 'expense,expense-transfer'
],
```

因此，源码中账户余额汇总不是简单的 `type LIKE 'income%'` 判断，而是**按配置的收入/支出类型集合过滤**，以 `whereIn` 精确匹配。一笔转账产生的 `expense-transfer` 和 `income-transfer` 类型均被各自的收入/支出关系正确收录，余额变动会自动纳入统计

---

## 核心代码参考

| 组件 | 文件位置 |
|------|---------|
| Transfer 模型 | [Transfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php) |
| Transaction 模型 | [Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php) |
| 创建转账 Job | [CreateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php) |
| 更新转账 Job | [UpdateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransfer.php) |
| 币种转换 Trait | [Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php) |
| 转账控制器 | [Transfers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transfers.php) |
| 请求验证 | [Transfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transfer.php) |
| 创建视图 | [create.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/create.blade.php) |
| 列表视图 | [index.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/banking/transfers/index.blade.php) |
| 前端 JS | [transfers.js](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/banking/transfers.js) |
