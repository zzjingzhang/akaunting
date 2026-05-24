# Document Item 与 Banking Transaction 税费计算差异分析

代码参考：[CreateDocumentItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L29-L202)、[CreateTransactionTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L50-L155)、[document_item_taxes 迁移](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L187-L204)、[transaction_taxes 迁移](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2023_10_03_000000_core_v310.php#L34-L50)

---

## 1. 各税类型的税基与符号对比

### 变量定义与流转（Document Item 链路）

| 变量 | 初始值（第60行） | 被谁修改 | 被谁使用 |
|------|-----------------|----------|----------|
| `$item_discounted_amount` | `price × quantity` 减 line/total 折扣 | 仅初始化时 | inclusive 税基 |
| `$actual_price_item` | = `$item_discounted_amount` | inclusive 后：`= $item_discounted_amount - $item_tax_total`（第95行） | normal、withholding 税基；最终存为 `document_items.total` |
| `$item_amount` | = `$item_discounted_amount` | fixed/normal/withholding 各累加 `$tax_amount`（第109、124、139行）；compound **不累加** | compound 税基 |
| `$item_tax_total` | 0 | 所有税类型均累加 | 最终存为 `document_items.tax` |

**关键流转关系**：
1. `$item_amount` 初始值 = `$item_discounted_amount`，后者来自 `price × quantity`。如果 `price` 在业务含义上是价内税价格（即已含 inclusive 税），则 `$item_amount` 从一开始就嵌有价内税的货币成分——这是业务层面的隐含假设，代码层面不做强制区分。
2. inclusive **不直接**修改 `$item_amount`，只修改 `$actual_price_item`。
3. 但 normal/withholding 基于 `$actual_price_item`，且其税额会累加到 `$item_amount`。因此 **inclusive 通过 normal/withholding 间接影响 `$item_amount`，进而间接影响 compound 税基**。
4. compound 自身税额不累加到 `$item_amount`，因此多个 compound 之间不会彼此叠加。

### 1.1 inclusive（价内税）

| 链路 | 税基 | 符号 | 公式 |
|------|------|------|------|
| Document Item | `$item_discounted_amount` | 正 | `amount = base - (base / (1 + rate/100))` |
| Banking Transaction | `$transaction_amount` | 正 | `amount = base - (base / (1 + rate/100))` |

**Document 副作用**：inclusive 计算后，执行 `$actual_price_item = $item_discounted_amount - $item_tax_total`（第95行）。此操作降低了 `$actual_price_item`，使后续 normal/withholding 的税基变小，进而改变它们的税额，进而改变 `$item_amount`，最终间接影响 compound 的税基。

### 1.2 fixed（固定税）

| 链路 | 税基 | 符号 | 公式 |
|------|------|------|------|
| Document Item | `$tax->rate × quantity` | 正 | `amount = rate × quantity` |
| Banking Transaction | `$tax->rate × 1` | 正 | `amount = rate × 1` |

**差异**：Document 中 fixed 乘以数量，Transaction 中固定乘以 1。

**Document 副作用**：`$item_amount += $tax_amount`，fixed 税额进入后续 compound 的税基。

### 1.3 normal（正常税）

| 链路 | 税基 | 符号 | 公式 |
|------|------|------|------|
| Document Item | `$actual_price_item` | 正 | `amount = base × (rate / 100)` |
| Banking Transaction | `$transaction_amount` | 正 | `amount = base × (rate / 100)` |

**差异**：Document 中 normal 的税基是扣除所有 inclusive 税后的不含税价格；Transaction 直接使用原始金额。

**Document 副作用**：`$item_amount += $tax_amount`，normal 税额进入后续 compound 的税基。

### 1.4 withholding（预扣税）

| 链路 | 税基 | 符号 | 公式 |
|------|------|------|------|
| Document Item | `$actual_price_item` | 负 | `amount = -(base × (rate / 100))` |
| Banking Transaction | `$transaction_amount` | 负 | `amount = -(base × (rate / 100))` |

**差异**：同 normal。

**Document 副作用**：`$item_amount += $tax_amount`（负值使 `$item_amount` 变小），withholding 税额也进入后续 compound 的税基（以负值方式降低税基）。

### 1.5 compound（复合税）

| 链路 | 税基 | 符号 | 公式 |
|------|------|------|------|
| Document Item | `$item_amount` | 正 | `amount = (base / 100) × rate` |
| Banking Transaction | `$transaction_amount` | 正 | `amount = (base / 100) × rate` |

**差异**：Document 中 compound 基于 `$item_amount`。Transaction 中 compound 始终基于原始 `$transaction_amount`。

**Document compound 税基的精确构成**：

```
$item_amount = $item_discounted_amount           // 初始值（业务上 price 可能已嵌有价内税成分）
              + Σ fixed 税额                      // fixed 直接累加
              + Σ normal 税额（基于 $actual_price_item）  // normal 受 inclusive 间接影响
              + Σ withholding 税额（基于 $actual_price_item） // withholding 受 inclusive 间接影响
```

- **不含 inclusive 直接成分**：inclusive 的税额不会被直接累加到 `$item_amount`，inclusive 的计算步骤本身不改变 `$item_amount`。
- **可能含 inclusive 间接成分**：当存在 normal/withholding 时，inclusive 通过降低 `$actual_price_item`，使 normal/withholding 的税额变小，从而 `$item_amount` 变小，compound 税基变小。inclusive 对 compound 产生**间接负向影响**。
- **不含 compound 自身成分**：compound 仅将税额累加到 `$item_tax_total`，不累加到 `$item_amount`，因此多个 compound 之间互不叠加。

---

## 2. fixed 与 compound 的特殊行为

### 2.1 fixed tax 是否乘 quantity

- **Document Item**：**是**，[CreateDocumentItem.php:100](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L100)：`$tax_amount = $tax->rate * (double) $this->request['quantity']`
- **Banking Transaction**：**否**，[CreateTransactionTaxes.php:100](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L100)：`$tax_amount = $tax->rate * (double) 1`

### 2.2 compound tax 是否基于已含其他税后的金额

- **Document Item**：**部分是，且含间接影响**。compound 基于 `$item_amount`，其构成为：折扣金额 + fixed 税额 + normal 税额 + withholding 税额。其中 normal/withholding 的税额已受 inclusive 间接影响。compound 自身的税额不累加到 `$item_amount`。
- **Banking Transaction**：**否**。compound 始终基于原始 `$transaction_amount`，与其他税无关。

**Document 中多个 compound 不叠加**：每个 compound 都使用同一个 `$item_amount`（因为 compound 的税额不累加到 `$item_amount`），所以多个 compound 独立计算。

**inclusive 对 compound 的间接影响路径**：
```
inclusive 计算 → $actual_price_item 降低 → normal/withholding 税额变小 → $item_amount 变小 → compound 税基变小
```
在代码层面，inclusive 的计算步骤本身不改变 `$item_amount`，`$item_amount` 初始等于 `$item_discounted_amount`（业务上 price 可能已嵌有价内税成分）。inclusive 仅通过 normal/withholding 间接传导到 compound。若无 normal/withholding，inclusive 对 compound 无传导影响。

---

## 3. amount 保存时的 abs、round 与数据库精度

### 3.1 数据库字段精度

两个表的 `amount` 字段定义完全相同：

```sql
-- document_item_taxes（2019_11_16_000000_core_v2.php 第195行）
$table->double('amount', 15, 4)->default('0.0000');

-- transaction_taxes（2023_10_03_000000_core_v310.php 第41行）
$table->double('amount', 15, 4)->default('0.0000');
```

- 字段类型为 `DOUBLE(M,D)`，M=15 为总显示位数，D=4 为小数位数
- 整数部分为 M-D=11 位，最大可表示约 `±99,999,999,999.9999`（11 位整数 + 4 位小数）
- 注意：MySQL 的 DOUBLE 是浮点类型，`(M,D)` 参数更多是显示宽度/精度提示，不是硬性存储约束；数据库迁移中也未设置 `unsigned` 或 check 约束来限制正负

### 3.2 DocumentItemTax 保存逻辑

[CreateDocumentItem.php:191-197](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L191-L197)：

```php
foreach ($item_taxes as $item_tax) {
    $item_tax['document_item_id'] = $document_item->id;
    $item_tax['amount'] = round(abs($item_tax['amount']), $precision);
    // ...
    DocumentItemTax::create($item_tax);
}
```

- **CreateDocumentItem 链路的处理**：经此作业创建时，amount 经过 `round(abs(...), $precision)`，其中 `$precision` 来自 `currency($this->document->currency_code)->getPrecision()`（通常 2 位）。`abs()` 确保所有类型（包括 withholding）在保存前转为非负值。
- **数据库和模型层无强制约束**：迁移中 `double('amount', 15, 4)` 无 `unsigned` 约束，[DocumentItemTax 模型](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/DocumentItemTax.php#L17) 的 `$fillable` 也无任何校验。其他写入路径（如直接 DB 操作、UpdateDocumentItem 等）可能不经过 `abs()`。
- **withholding 符号处理**：withholding 计算时为负值（`$tax_amount = -($actual_price_item * ($tax->rate / 100))`），经 CreateDocumentItem 链路 `abs()` 后存为非负，符号信息在 `document_item_taxes.amount` 中丢失。但 `document_items.tax` 字段保存的是 `$item_tax_total`（第 173 行 `round()` 不含 `abs()`），withholding 仍以负值参与该字段的汇总。

### 3.3 TransactionTax 保存逻辑

[CreateTransactionTaxes.php:39-44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L39-L44)：

```php
\DB::transaction(function () {
    $transaction_taxes = $this->getTaxesCalculated();
    foreach ($transaction_taxes as $transaction_tax) {
        TransactionTax::create($transaction_tax);
    }
});
```

- **abs**：否，withholding 税以负值存入数据库
- **round**：否，直接保存原始计算值（可能有浮点精度残留）
- **withholding 符号保留**：withholding 计算时为负值，直接存为负

---

## 4. 未知 tax type 的处理

### 4.1 Document Item 链路

[CreateDocumentItem.php:69-156](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L69-L156)

- **无类型检查**：第 79 行直接 `${$tax->type . 's'}[] = $tax` 创建动态变量
- **后续仅 5 个 if 分支**：`inclusives`、`fixeds`、`normals`、`withholdings`、`compounds`
- **行为**：未知类型的税被静默忽略，不报错、不记录日志，也不产生对应 `DocumentItemTax` 记录

### 4.2 Banking Transaction 链路

[CreateTransactionTaxes.php:66-79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateTransactionTaxes.php#L66-L79)：

```php
$known_types = ['inclusive', 'fixed', 'normal', 'withholding', 'compound'];

if (! in_array($tax->type, $known_types)) {
    report(new \UnexpectedValueException("CreateTransactionTaxes: unknown tax type '{$tax->type}' for tax ID {$tax->id} — skipped."));
    continue;
}
```

- **有类型白名单检查**
- **行为**：未知类型通过 `report()` 记录错误日志，然后跳过

---

## 5. 计算结果不同的最小例子

### 场景 1：fixed 税（数量差异）

**条件**：
- 税率类型：fixed，rate = 10
- Document Item：price = 100，quantity = 2，无折扣
- Banking Transaction：amount = 200

**Document Item 计算**：
```
tax_amount = 10 × 2 = 20
$item_amount = 200 + 20 = 220
$item_tax_total = 20
最终保存 DocumentItemTax.amount = round(abs(20)) = 20
```

**Banking Transaction 计算**：
```
tax_amount = 10 × 1 = 10
最终保存 TransactionTax.amount = 10
```

**结果差异**：Document 税额 20，Transaction 税额 10，相差 10。

---

### 场景 2：inclusive + normal 组合（税基差异）

**条件**：
- Tax A：inclusive，rate = 10
- Tax B：normal，rate = 10
- Document Item：price = 110，quantity = 1，无折扣
- Banking Transaction：amount = 110

**Document Item 计算**：
```
$item_discounted_amount = 110 × 1 = 110
$actual_price_item = $item_amount = 110

// inclusive（基于 $item_discounted_amount）
tax_A = 110 - (110 / 1.1) = 110 - 100 = 10
$item_tax_total = 10
$actual_price_item = 110 - 10 = 100  （$item_amount 仍为 110）

// normal（基于降低后的 $actual_price_item）
tax_B = 100 × 10% = 10
$item_amount = 110 + 10 = 120
$item_tax_total = 20

最终保存：tax_A=10, tax_B=10（均 abs+round）
```

**Banking Transaction 计算**：
```
// inclusive
tax_A = 110 - (110 / 1.1) = 10

// normal（基于原始 transaction_amount）
tax_B = 110 × 10% = 11

最终保存：tax_A=10, tax_B=11（无 abs、无 round）
```

**结果差异**：normal 税额 Document=10，Transaction=11，相差 1。

---

### 场景 3：inclusive + normal + compound 组合（inclusive 通过 normal 间接影响 compound）

**条件**：
- Tax A：inclusive，rate = 10
- Tax B：normal，rate = 10
- Tax C：compound，rate = 5
- Document Item：price = 110，quantity = 1，无折扣
- Banking Transaction：amount = 110

**Document Item 计算**：
```
$item_discounted_amount = 110
$actual_price_item = $item_amount = 110

// inclusive
tax_A = 110 - (110 / 1.1) = 10
$item_tax_total = 10
$actual_price_item = 110 - 10 = 100  （$item_amount 仍为 110）

// normal（税基已因 inclusive 而降低）
tax_B = 100 × 10% = 10
$item_amount = 110 + 10 = 120  （normal 税额进入 $item_amount）
$item_tax_total = 20

// compound（基于 $item_amount = 120，其中含 normal 税额 10，normal 已受 inclusive 间接影响）
tax_C = (120 / 100) × 5 = 6.0
$item_tax_total = 26.0

最终保存：tax_A=10, tax_B=10, tax_C=6.0（均 abs+round）
```

**Banking Transaction 计算**：
```
// inclusive
tax_A = 110 - (110 / 1.1) = 10

// normal（基于原始金额，不受 inclusive 影响）
tax_B = 110 × 10% = 11

// compound（基于原始 transaction_amount = 110）
tax_C = (110 / 100) × 5 = 5.5

最终保存：tax_A=10, tax_B=11, tax_C=5.5（无处理）
```

**结果差异**：
- normal：Document=10，Transaction=11，差 1
- compound：Document=6.0，Transaction=5.5，差 0.5
- compound 的差异源于 Document 中 normal 税额进入 `$item_amount`，而 normal 本身又因 inclusive 而降低

---

### 场景 4：多个 compound 不互相叠加

**条件**：
- Tax A：compound，rate = 5
- Tax B：compound，rate = 5
- Document Item：price = 100，quantity = 1，无折扣
- Banking Transaction：amount = 100

**Document Item 计算**：
```
// 两个 compound 均基于同一个 $item_amount = 100
tax_A = (100 / 100) × 5 = 5
tax_B = (100 / 100) × 5 = 5
$item_tax_total = 10
```

**Banking Transaction 计算**：
```
// 两个 compound 均基于同一个 $transaction_amount = 100
tax_A = (100 / 100) × 5 = 5
tax_B = (100 / 100) × 5 = 5
```

**结果相同**：本例中两者一致。若叠加 normal 或 fixed 则产生差异。

---

### 场景 5：withholding 符号与 abs 差异

**条件**：
- Tax A：withholding，rate = 10
- Document Item：price = 100，quantity = 1，无折扣
- Banking Transaction：amount = 100

**Document Item 计算**：
```
tax_A = -(100 × 10%) = -10
最终保存（经 CreateDocumentItem 链路）= round(abs(-10)) = 10（非负）
```

**Banking Transaction 计算**：
```
tax_A = -(100 × 10%) = -10
最终保存 = -10（负值，无 abs）
```

**结果差异**：经 CreateDocumentItem 链路 Document 存非负 10，Transaction 存负 10，绝对值相同，符号相反。注意：其他写入路径不经过 abs()，数据库层不强制非负。

---

## 总结对比表

| 对比项 | Document Item | Banking Transaction |
|--------|--------------|---------------------|
| inclusive 税基 | `$item_discounted_amount`（折扣后金额） | `$transaction_amount`（原始金额） |
| inclusive 后副作用 | `$actual_price_item = base - tax_total`，降低后续 normal/withholding 的税基 | 无副作用，normal/withholding 仍用原始金额 |
| fixed 乘 quantity | 是 | 否（乘 1） |
| normal/withholding 税基 | `$actual_price_item`（扣除 inclusive 后） | `$transaction_amount`（原始金额） |
| compound 税基 | `$item_amount` = 折扣金额 + fixed + normal + withholding；normal/withholding 已受 inclusive 间接影响；不含 compound 自身 | `$transaction_amount`（原始金额） |
| inclusive 对 compound 的影响 | 通过 normal/withholding 间接传导（负向）；inclusive 计算步骤本身不改变 `$item_amount`，无 normal/withholding 时无传导影响 | 无影响 |
| 多个 compound 是否叠加 | 否（compound 税额不累加到 `$item_amount`） | 否（始终基于原始金额） |
| 保存时 abs | CreateDocumentItem 链路是；数据库/模型层无强制，其他路径可能绕过 | 否 |
| 保存时 round | CreateDocumentItem 链路是（按货币精度）；其他路径可能不 | 否 |
| 数据库 amount 精度 | `double(15,4)`（11 位整数 + 4 位小数，最大约 ±99,999,999,999.9999） | `double(15,4)`（11 位整数 + 4 位小数，最大约 ±99,999,999,999.9999） |
| 未知类型处理 | 静默忽略 | report 日志后跳过 |
