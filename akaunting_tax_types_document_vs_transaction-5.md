# Document Item 与 Banking Transaction 税费计算差异分析

代码参考：[CreateDocumentItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php)、[CreateTransactionTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php)

---

## 1. 各税类型的税基与符号对比

### 1.1 inclusive（价内税）

| 链路 | 税基（tax base） | 符号 | 公式 |
|------|-----------------|------|------|
| Document Item | `$item_discounted_amount`（折扣后金额） | 正 | `amount = base - (base / (1 + rate/100))` |
| Banking Transaction | `$transaction_amount`（交易原始金额） | 正 | `amount = base - (base / (1 + rate/100))` |

**Document 额外处理**：inclusive 计算后，`$actual_price_item = $item_discounted_amount - $item_tax_total`，后续 normal/withholding/compound 均基于此不含税价格计算。

### 1.2 fixed（固定税）

| 链路 | 税基（tax base） | 符号 | 公式 |
|------|-----------------|------|------|
| Document Item | `$tax->rate * quantity` | 正 | `amount = rate * quantity` |
| Banking Transaction | `$tax->rate * 1` | 正 | `amount = rate * 1` |

**差异**：Document 中 fixed 税会乘以数量，Transaction 中固定乘以 1。

### 1.3 normal（正常税）

| 链路 | 税基（tax base） | 符号 | 公式 |
|------|-----------------|------|------|
| Document Item | `$actual_price_item`（inclusive 税后的不含税价） | 正 | `amount = base * (rate / 100)` |
| Banking Transaction | `$transaction_amount`（交易原始金额） | 正 | `amount = base * (rate / 100)` |

**差异**：Document 中 normal 基于 inclusive 处理后的不含税价格计算，Transaction 直接基于原始金额。

### 1.4 withholding（预扣税）

| 链路 | 税基（tax base） | 符号 | 公式 |
|------|-----------------|------|------|
| Document Item | `$actual_price_item`（inclusive 税后的不含税价） | 负 | `amount = -(base * (rate / 100))` |
| Banking Transaction | `$transaction_amount`（交易原始金额） | 负 | `amount = -(base * (rate / 100))` |

**差异**：同 normal，Document 的税基是扣除 inclusive 后的价格。

### 1.5 compound（复合税）

| 链路 | 税基（tax base） | 符号 | 公式 |
|------|-----------------|------|------|
| Document Item | `$item_amount`（已含 fixed/normal/withholding 的金额） | 正 | `amount = (base / 100) * rate` |
| Banking Transaction | `$transaction_amount`（交易原始金额） | 正 | `amount = (base / 100) * rate` |

**差异**：Document 中 compound 基于已包含其他税后的金额计算，Transaction 基于原始金额。

---

## 2. fixed 与 compound 的特殊行为

### 2.1 fixed tax 是否乘 quantity

- **Document Item**：**是**，`$tax_amount = $tax->rate * (double) $this->request['quantity']`（第100行）
- **Banking Transaction**：**否**，`$tax_amount = $tax->rate * (double) 1`（第100行）

### 2.2 compound tax 是否基于已含其他税后的金额

- **Document Item**：**是**。compound 在 fixed、normal、withholding 之后计算，使用的 `$item_amount` 已经累加了这些税的金额（第145行）
- **Banking Transaction**：**否**。compound 始终基于原始 `$transaction_amount` 计算，与其他税无关（第142行）

---

## 3. amount 保存时的绝对值与 round 处理

### 3.1 DocumentItemTax

```php
// CreateDocumentItem.php 第193行
$item_tax['amount'] = round(abs($item_tax['amount']), $precision);
```

- **取绝对值**：是，`abs()` 确保金额始终为正
- **round**：是，按货币精度 `$precision` 四舍五入

### 3.2 TransactionTax

```php
// CreateTransactionTaxes.php 无处理
TransactionTax::create($transaction_tax);
```

- **取绝对值**：否，withholding 税会保存为负值
- **round**：否，直接保存原始计算值，无精度处理

---

## 4. 未知 tax type 的处理

### 4.1 Document Item 链路

**代码位置**：CreateDocumentItem.php 第69-156行

- **无类型检查**：直接通过 `${$tax->type . 's'}[] = $tax` 创建变量
- **后续无匹配分支**：只有 `inclusives`、`fixeds`、`normals`、`withholdings`、`compounds` 五个 if 分支
- **行为**：未知类型的税会被静默忽略，不报错、不记录日志

### 4.2 Banking Transaction 链路

**代码位置**：CreateTransactionTaxes.php 第66-79行

```php
$known_types = ['inclusive', 'fixed', 'normal', 'withholding', 'compound'];

if (! in_array($tax->type, $known_types)) {
    report(new \UnexpectedValueException("CreateTransactionTaxes: unknown tax type '{$tax->type}' for tax ID {$tax->id} — skipped."));
    continue;
}
```

- **有类型白名单检查**
- **行为**：未知类型会被 `report()` 记录错误日志，然后跳过

---

## 5. 计算结果不同的最小例子

### 场景 1：fixed 税（数量差异）

**条件**：
- 税率类型：fixed，rate = 10
- Document Item：price = 100，quantity = 2
- Banking Transaction：amount = 200（与 Document 总价一致）

**Document Item 计算**：
```
tax_amount = 10 * 2 = 20
最终保存（abs + round）：20
```

**Banking Transaction 计算**：
```
tax_amount = 10 * 1 = 10
最终保存（无处理）：10
```

**结果差异**：Document 税额 20，Transaction 税额 10，相差 10。

---

### 场景 2：compound + normal 组合税

**条件**：
- Tax A：normal，rate = 10
- Tax B：compound，rate = 5
- Document Item：price = 100，quantity = 1，无折扣
- Banking Transaction：amount = 100

**Document Item 计算**：
```
// normal 税
actual_price_item = 100（无 inclusive）
tax_A = 100 * 10% = 10
item_amount = 100 + 10 = 110

// compound 税（基于已含税金额）
tax_B = (110 / 100) * 5 = 5.5

总税额 = 10 + 5.5 = 15.5
最终保存（abs + round）：10 和 5.5
```

**Banking Transaction 计算**：
```
// normal 税
tax_A = 100 * 10% = 10

// compound 税（基于原始金额）
tax_B = (100 / 100) * 5 = 5

总税额 = 10 + 5 = 15
最终保存（无处理）：10 和 5
```

**结果差异**：Document compound 税 5.5，Transaction compound 税 5，总税额相差 0.5。

---

### 场景 3：inclusive + normal 组合税

**条件**：
- Tax A：inclusive，rate = 10
- Tax B：normal，rate = 10
- Document Item：price = 110，quantity = 1
- Banking Transaction：amount = 110

**Document Item 计算**：
```
// inclusive 税
tax_A = 110 - (110 / 1.1) = 110 - 100 = 10
actual_price_item = 110 - 10 = 100

// normal 税（基于不含税价）
tax_B = 100 * 10% = 10

总税额 = 10 + 10 = 20
```

**Banking Transaction 计算**：
```
// inclusive 税
tax_A = 110 - (110 / 1.1) = 10

// normal 税（基于原始金额）
tax_B = 110 * 10% = 11

总税额 = 10 + 11 = 21
```

**结果差异**：Document 总税额 20，Transaction 总税额 21，相差 1。

---

## 总结对比表

| 对比项 | Document Item | Banking Transaction |
|--------|--------------|---------------------|
| fixed 乘 quantity | 是 | 否（乘 1） |
| compound 税基 | 已含其他税金额 | 原始金额 |
| normal/withholding 税基 | 扣除 inclusive 后 | 原始金额 |
| 保存时 abs | 是 | 否 |
| 保存时 round | 是 | 否 |
| 未知类型处理 | 静默忽略 | 报错并跳过 |
