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
1. 优先取请求参数：`from_account_rate` / `to_account_rate`
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

### 2.4 金额转换逻辑

方法：[convertBetween()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php#L43-L54)

```php
public function convertBetween($amount, $from_code, $from_rate, $to_code, $to_rate)
{
    $default_amount = $amount;

    if ($from_code != default_currency()) {
        $default_amount = $this->convertToDefault($amount, $from_code, $from_rate);
    }

    $converted_amount = $this->convertFromDefault($default_amount, $to_code, $to_rate);

    return $converted_amount;
}
```

转换路径：`源币种 → 本位币 → 目标币种`，使用各自的汇率进行转换。

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

**Transfer 表只保存关联关系**：
- `expense_transaction_id`：指向转出方 Transaction
- `income_transaction_id`：指向转入方 Transaction

所有业务字段（金额、日期、描述、支付方式等）都通过访问器从 expense_transaction 中动态获取。

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

假设默认货币为 CNY：

| 账户 | 币种 | 操作前余额 | 转账金额 | 操作后余额 |
|------|------|-----------|---------|-----------|
| 账户 A（转出） | CNY | 10,000 | -1,000 | 9,000 |
| 账户 B（转入） | CNY | 5,000 | +1,000 | 6,000 |

**落库数据**：
- expense_transaction: amount = 1000, currency_code = CNY, currency_rate = 1.0
- income_transaction: amount = 1000, currency_code = CNY, currency_rate = 1.0
- Transfer.amount（访问器）= 1000

### 5.2 异币种例子

假设默认货币为 CNY，USD 汇率 = 7.0：

| 账户 | 币种 | 操作前余额 | 转账金额 | 操作后余额 |
|------|------|-----------|---------|-----------|
| 账户 A（转出） | USD | 5,000 | -1,000 | 4,000 |
| 账户 B（转入） | CNY | 50,000 | +7,000 | 57,000 |

**落库数据**：
- expense_transaction: amount = 1000, currency_code = USD, currency_rate = 7.0
- income_transaction: amount = 7000, currency_code = CNY, currency_rate = 1.0
- Transfer.amount（访问器）= 1000（只取 expense 侧金额）

### 5.3 为什么余额分析不能只看 transfers 表

**核心原因**：Transfer 模型的所有业务字段都是通过访问器从 `expense_transaction` 中获取的，这意味着：

1. **Transfer 表本身不存 amount**：`$transfer->amount` 实际是 `$this->expense_transaction->amount`
2. **异币种时收入侧金额丢失**：Transfer 只能看到转出方金额，看不到转入方实际收到的金额（如上述例子中，Transfer 显示 1000，但实际 B 账户增加了 7000）
3. **余额分析需要查 transactions 表**：要准确计算每个账户的余额变动，必须直接查询 transactions 表，根据每笔交易的 `account_id` + `type` + `amount` 来汇总

**正确的余额分析方式**：
```sql
-- 统计每个账户的净余额变动
SELECT 
    account_id,
    SUM(CASE WHEN type LIKE 'income%' THEN amount ELSE -amount END) as net_change
FROM transactions
WHERE company_id = ?
GROUP BY account_id;
```

---

## 核心代码参考

| 组件 | 文件位置 |
|------|---------|
| Transfer 模型 | [Transfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transfer.php) |
| Transaction 模型 | [Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php) |
| 创建转账 Job | [CreateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransfer.php) |
| 更新转账 Job | [UpdateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransfer.php) |
| 转账控制器 | [Transfers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transfers.php) |
| 请求验证 | [Transfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Banking/Transfer.php) |
| 币种转换 Trait | [Currencies.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Currencies.php) |
