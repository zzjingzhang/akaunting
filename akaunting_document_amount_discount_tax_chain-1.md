# Akaunting Document 金额、折扣、税费计算链路分析

## 核心代码位置引用

- 创建入口: [Invoices.php::store()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L100-L125)
- 主 Job: [CreateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php)
- 子 Job (金额计算): [CreateDocumentItemsAndTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)
- 行项目计算: [CreateDocumentItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php)
- 税类型定义: [Tax.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Tax.php)

---

## 1. Document amount 回写机制

### 初始值设置

在 [CreateDocument.php::handle() 第24-26行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L24-L26)：

```php
if (empty($this->request['amount'])) {
    $this->request['amount'] = 0;
}
```

创建 Document 前，若请求中 amount 为空则强制设为 0。

### 创建流程

```
CreateDocument::handle()
├─ 设置 amount = 0
├─ 创建 Document (amount = 0)                              ← 数据库中 amount 暂时为 0
├─ 调度 CreateDocumentItemsAndTotals 子 Job (同步执行)
│  └─ 该子 Job 修改 $this->request['amount']                ← 隐式回写
└─ 更新 Document: $this->model->update($this->request->all()) ← 用修改后的 amount 更新
```

### 回写链路

在 [CreateDocumentItemsAndTotals.php::handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158) 中，`$this->request['amount']` 的值经以下步骤被修改：

| 步骤 | 行号 | 代码 | 说明 |
|------|------|------|------|
| 1 | 第33行 | `list(...) = $this->createItems()` | 计算出 `$actual_total`（已扣全局折扣） |
| 2 | 第50行 | `$this->request['amount'] += $actual_total;` | 初始 0 + actual_total |
| 3 | 第96-113行 | 遍历 `$taxes`，`$this->request['amount'] += $tax['amount'];` | 累加各税的**原始值**（含正负） |
| 4 | 第133-137行 | 处理 extra totals | addition/subtraction |
| 5 | 第144行 | `$this->request['amount'] = round($this->request['amount'], $precision);` | 四舍五入 |

**关键机制**：子 Job 通过修改父 Job 的 `$this->request` 对象属性实现回写，无显式 return，依赖对象引用。

---

## 2. 折扣计算层级与税基影响

### 折扣分类

| 折扣类型 | 计算层级 | 是否影响税基 | 代码位置 |
|---------|---------|------------|---------|
| 行折扣 (百分比) | DocumentItem 层 | ✅ 影响 | [CreateDocumentItem.php:39-45](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L39-L45) |
| 行折扣 (固定) | DocumentItem 层 | ✅ 影响 | [CreateDocumentItem.php:42-44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L42-L44) |
| 全局折扣 (百分比) | DocumentItem 层 → DocumentTotal 层 | ✅ **影响** | [CreateDocumentItem.php:48-56](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L48-L56) |
| 全局折扣 (固定) | DocumentItem 层 (分摊) → DocumentTotal 层 | ✅ 影响 | [CreateDocumentItemsAndTotals.php:170-186](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L170-L186) |

### 行折扣计算

[CreateDocumentItem.php 第39-45行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L39-L45)：

```php
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        $item_discounted_amount -= ($item_amount * ($this->request['discount'] / 100));
    } else {
        $item_discounted_amount -= $this->request['discount'];
    }
}
```

行折扣直接从 `$item_discounted_amount` 中扣除，该变量随后作为税计算的基础，因此影响税基。

### 全局折扣计算——双阶段机制

全局折扣无论是百分比还是固定，都经历"**税前扣除参与税基 → 保存前加回 → 汇总层统一扣除**"的三阶段流程。

#### 阶段一：税前扣除（影响税基）

[CreateDocumentItem.php 第48-56行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L48-L56)：

```php
if (! empty($this->request['global_discount'])) {
    if ($this->request['global_discount_type'] === 'percentage') {
        $global_discount = $item_discounted_amount * ($this->request['global_discount'] / 100);
    } else {
        $global_discount = $this->request['global_discount'];
    }
    $item_discounted_amount -= $global_discount;
}
```

- 局部变量 `$global_discount` 保存**该行**实际扣减的金额
- 从 `$item_discounted_amount` 中扣除，该变量在第60行赋值给 `$actual_price_item` 和 `$item_amount`，作为后续所有税的计算基础
- **全局百分比折扣也会影响税基**，因为它同样在税前扣除

#### 阶段二：保存前加回

[CreateDocumentItem.php 第158-162行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L158-L162)：

```php
if (! empty($global_discount)) {
    $actual_price_item += $global_discount;
    $item_amount += $global_discount; 
    $item_discounted_amount += $global_discount;
}
```

税计算完成后，若存在 `$global_discount`（局部变量，即该行实际分摊的折扣金额），将其加回到三个变量：
- `$actual_price_item`：影响 DocumentItem.total
- `$item_amount`：此处也会加回 global_discount，但 compound 税已在前面计算完成，因此这次加回不会再影响 compound 税基
- `$item_discounted_amount`：恢复原始值

**关键**：DocumentItem.total 包含 global_discount 加回后的值，因此**不反映全局折扣的扣除**。

#### 阶段三：汇总层统一扣除

[CreateDocumentItemsAndTotals.php 第254-260行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L254-L260)：

```php
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        $actual_total -= ($sub_total * ($this->request['discount'] / 100));
    } else {
        $actual_total -= $this->request['discount'];
    }
}
```

在 `createItems()` 返回前，从 `$actual_total` 中统一扣除全局折扣。然后 `$actual_total` 被加到 `$this->request['amount']`（第50行），最终通过 `update()` 回写到 Document。

#### 固定全局折扣的分摊

[CreateDocumentItemsAndTotals.php 第170-186行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L170-L186)：

```php
if (! empty($this->request['discount']) && $this->request['discount_type'] !== 'percentage') {
    $for_fixed_discount = $this->fixedDiscountCalculate();
}

// 对每个 item:
if (isset($for_fixed_discount)) {
    $item['global_discount'] = ($for_fixed_discount[$key] / ($for_fixed_discount['total'] / 100)) * ($this->request['discount'] / 100);
} else {
    // 百分比全局折扣：直接传入百分比值
    $item['global_discount'] = $this->request['discount'];
}
```

- 固定全局折扣：按行项目金额占比分摊到每行，存入 `$item['global_discount']`
- 百分比全局折扣：直接传入百分比值（如 `10`），由 CreateDocumentItem 按行换算成金额

---

## 3. 五类税的计算逻辑与体现

税的计算在 [CreateDocumentItem.php 第69-156行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L69-L156) 中完成，执行顺序为：`inclusive → fixed → normal → withholding → compound`。

### 基础变量（第34-60行）

| 变量 | 赋值时机 | 含义 |
|------|---------|------|
| `$item_amount` | 第34行 | price × quantity，行项目原始金额 |
| `$item_discounted_amount` | 第36-55行 | 扣减行折扣和 global_discount 后金额 |
| `$actual_price_item` | 第60行 | 初始等于 `$item_discounted_amount`，后被 inclusive 税修改 |
| `$item_amount`（重赋值） | 第60行 | 被赋值为 `$item_discounted_amount`，后续被 fixed/normal/withholding 修改 |

### 3.1 inclusive (含税)

[第82-96行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L82-L96)：

```php
$tax_amount = $item_discounted_amount - ($item_discounted_amount / (1 + $inclusive->rate / 100));
// ...
$actual_price_item = $item_discounted_amount - $item_tax_total;
```

- 从含税金额 `$item_discounted_amount` 中**分离**出税额
- `$actual_price_item` = 含税价 - inclusive 税合计 → 不含税净额（后续税的税基）
- **不修改** `$item_amount`
- 税金额存入 `$item_taxes[]`，保留原始正值

### 3.2 fixed (固定税)

[第98-111行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L98-L111)：

```php
$tax_amount = $tax->rate * (double) $this->request['quantity'];
$item_amount += $tax_amount;
```

- 按数量计税，与金额无关
- **增加** `$item_amount`（影响后续 compound 税的税基）
- 税金额存入 `$item_taxes[]`，保留原始正值

### 3.3 normal (普通税)

[第113-126行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L113-L126)：

```php
$tax_amount = $actual_price_item * ($tax->rate / 100);
$item_amount += $tax_amount;
```

- 基于 `$actual_price_item`（已扣 inclusive 税的不含税净额）
- **增加** `$item_amount`（影响后续 compound 税的税基）
- 税金额存入 `$item_taxes[]`，保留原始正值

### 3.4 withholding (预提税)

[第128-141行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L128-L141)：

```php
$tax_amount = -($actual_price_item * ($tax->rate / 100));
$item_amount += $tax_amount;  // 负值，实际减少
```

- 基于 `$actual_price_item`（已扣 inclusive 税的不含税净额）
- **负值**，减少 `$item_amount`（影响后续 compound 税的税基）
- 税金额存入 `$item_taxes[]`，保留原始**负值**

### 3.5 compound (复合税)

[第143-155行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L143-L155)：

```php
$tax_amount = ($item_amount / 100) * $compound->rate;
```

- 基于 `$item_amount`（此时已包含 fixed、normal、withholding 税对它的加减）
- **不修改** `$item_amount`（没有 `$item_amount += $tax_amount`）
- 税金额存入 `$item_taxes[]`，保留原始正值

### 税在各层级的体现

| 层级 | 数据来源 | 含税/不含税 | 符号处理 |
|------|---------|------------|---------|
| DocumentItem.total | `round($actual_price_item, $precision)` (第176行) | 不含税净额 + global_discount 加回 | 无符号问题 |
| DocumentItem.tax | `round($item_tax_total, $precision)` (第173行) | 所有税的代数和（含正负） | 保留原始正负 |
| DocumentItemTax.amount | `round(abs($item_tax['amount']), $precision)` (第193行) | 单个税的绝对值 | **强制取 abs** |
| DocumentTotal.tax | `round(abs($tax['amount']), $precision)` (第103行) | 汇总后单个税的绝对值 | **强制取 abs** |
| Document.amount | `$actual_total + Σ(tax['amount']) + extra_totals` (第50+109行) | actual_total 加所有税的**原始值** | **保留原始正负** |

### 五类税在各层级的表现

| 税类型 | DocumentItem.total | DocumentItem.tax | DocumentItemTax.amount | DocumentTotal.tax | Document.amount |
|--------|-------------------|-----------------|-----------------------|-------------------|----------------|
| inclusive | 不含税净额(含global_discount加回) | 含（正） | 正数（abs） | 正数（abs） | **+税额** (原始正值累加) |
| fixed | 不含税净额(含global_discount加回) | 含（正） | 正数（abs） | 正数（abs） | +税额 (原始正值累加) |
| normal | 不含税净额(含global_discount加回) | 含（正） | 正数（abs） | 正数（abs） | +税额 (原始正值累加) |
| withholding | 不含税净额(含global_discount加回) | 含（负） | 正数（abs） | 正数（abs） | **-税额** (原始负值累加) |
| compound | 不含税净额(含global_discount加回) | 含（正） | 正数（abs） | 正数（abs） | **+税额** (原始正值累加) |

**重要澄清**：
- inclusive 税通过 `$taxes` 数组传入 handle()，其 `$tax['amount']` 是原始正值，在第109行被累加到 `$this->request['amount']`，所以 inclusive 税**会增加** Document.amount
- compound 税同理，其 `$tax['amount']` 是原始正值，也会增加 Document.amount
- withholding 税的 `$tax['amount']` 是原始负值，会**减少** Document.amount

---

## 4. 反例：完整计算流程

### 场景设定

创建一张 Invoice：
- **商品 A**：价格 $100，数量 1，无行折扣
- **商品 B**：价格 $200，数量 1，无行折扣
- **全局折扣**：固定 $15（`discount=15`, `discount_type=fixed`）
- **税 1**：VAT (inclusive)，税率 10%（应用于商品 A）
- **税 2**：WHT (withholding)，税率 5%（应用于商品 A）

### 第一步：固定全局折扣分摊

[fixedDiscountCalculate()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L265-L289)：

```
商品 A: $100 × 1 = $100 (无行折扣)
商品 B: $200 × 1 = $200 (无行折扣)
合计: $300
```

分摊到每个 item 的 global_discount：

```
商品 A: ($100 / (300 / 100)) × (15 / 100) = (100 / 3) × 0.15 = 33.33... × 0.15 = $5.00
商品 B: ($200 / (300 / 100)) × (15 / 100) = (200 / 3) × 0.15 = 66.66... × 0.15 = $10.00
```

商品 A: `global_discount=5.00`, `global_discount_type=fixed`
商品 B: `global_discount=10.00`, `global_discount_type=fixed`

### 第二步：商品 A — CreateDocumentItem

[CreateDocumentItem.php handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L29-L202)：

```
// 第34行: 原始金额
item_amount = 100 × 1 = 100

// 第36行: 初始化折扣后金额
item_discounted_amount = 100

// 第39-45行: 无行折扣

// 第48-56行: 全局折扣扣除
global_discount = 5.00 (局部变量)
item_discounted_amount = 100 - 5 = 95

// 第60行: 初始化
actual_price_item = item_amount = item_discounted_amount = 95

// === 税计算 ===

// 第82-96行: inclusive 税 (VAT 10%)
tax_amount = 95 - (95 / 1.1) = 95 - 86.363636... = 8.636363...
item_tax_total = 8.636363...
actual_price_item = 95 - 8.636363... = 86.363636...
// item_amount 不变 = 95

// 第128-141行: withholding 税 (WHT 5%)
tax_amount = -(86.363636... × 5%) = -4.318181...
item_tax_total = 8.636363... - 4.318181... = 4.318181...
item_amount = 95 + (-4.318181...) = 90.681818...

// 第158-162行: global_discount 加回
actual_price_item = 86.363636... + 5 = 91.363636...
item_amount = 90.681818... + 5 = 95.681818...
item_discounted_amount = 95 + 5 = 100

// 第173-176行: 最终赋值
DocumentItem.tax = round(4.318181..., 2) = 4.32
DocumentItem.total = round(91.363636..., 2) = 91.36

// 第186-199行: DocumentItemTax
// $item_taxes 数组中保留原始值:
//   VAT: amount = 8.636363... (正)
//   WHT: amount = -4.318181... (负)
// 存储时强制 abs:
//   VAT: abs(8.636363...) = 8.64
//   WHT: abs(-4.318181...) = 4.32
```

### 第三步：商品 B — CreateDocumentItem

```
item_amount = 200 × 1 = 200
item_discounted_amount = 200

// 无行折扣

// 全局折扣扣除
global_discount = 10.00
item_discounted_amount = 200 - 10 = 190

// 初始化
actual_price_item = item_amount = item_discounted_amount = 190

// 无税

// global_discount 加回
actual_price_item = 190 + 10 = 200
item_amount = 190 + 10 = 200
item_discounted_amount = 190 + 10 = 200

// 最终
DocumentItem.tax = 0
DocumentItem.total = 200
```

### 第四步：createItems() 汇总

[CreateDocumentItemsAndTotals.php::createItems()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L160-L263)：

```
// 第231行: sub_total 累加 DocumentItem.total
sub_total = 91.36 + 200 = 291.36

// 第232行: actual_total 累加 DocumentItem.total
actual_total = 91.36 + 200 = 291.36

// 第241-249行: 税汇总 (从 $document_item->item_taxes 中取原始值)
// $item_taxes 数组中 amount 保留原始正负:
//   VAT_id => {name: 'VAT', amount: 8.636363...}
//   WHT_id => {name: 'WHT', amount: -4.318181...}
// 汇总后 $taxes:
//   VAT_id => {name: 'VAT', amount: 8.64}  (round)
//   WHT_id => {name: 'WHT', amount: -4.32} (round, 保留负值)

// 第254-260行: actual_total 统一扣全局折扣
actual_total -= 15 = 291.36 - 15 = 276.36

// 返回: [sub_total=291.36, actual_total=276.36, discount_amount_total=0, taxes]
```

### 第五步：handle() 汇总

[CreateDocumentItemsAndTotals.php::handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158)：

```
// 第50行: amount 初始化为 actual_total
request['amount'] = 0 + 276.36 = 276.36

// DocumentTotal 记录 (按 sort_order):

// sort_order 1: sub_total = 291.36
// sort_order 2: discount = 15.00 (全局折扣)

// 第96-113行: 遍历 $taxes
//   VAT: DocumentTotal.tax = abs(8.64) = 8.64
//        request['amount'] += 8.64 = 285.00
//   WHT: DocumentTotal.tax = abs(-4.32) = 4.32
//        request['amount'] += (-4.32) = 280.68

// 第144行: 四舍五入
request['amount'] = round(280.68, 2) = 280.68

// sort_order 5: DocumentTotal.total = 280.68
```

### 最终结果

| 模型 | 字段 | 值 |
|------|------|-----|
| Document | amount | **$280.68** |
| DocumentItem (商品 A) | total | $91.36 |
| DocumentItem (商品 A) | tax | $4.32 |
| DocumentItem (商品 B) | total | $200.00 |
| DocumentItem (商品 B) | tax | $0.00 |
| DocumentItemTax (VAT) | amount | $8.64 |
| DocumentItemTax (WHT) | amount | $4.32 |
| DocumentTotal (sub_total) | amount | $291.36 |
| DocumentTotal (discount) | amount | $15.00 |
| DocumentTotal (tax - VAT) | amount | $8.64 |
| DocumentTotal (tax - WHT) | amount | $4.32 |
| DocumentTotal (total) | amount | $280.68 |

### 关键易错点

| 易错点 | 直觉错误 | 源码实际行为 | 代码位置 |
|--------|---------|------------|---------|
| 全局百分比折扣不影响税基 | 认为只在汇总层扣除 | 在 DocumentItem 层税前扣除，影响税基 | [CreateDocumentItem.php:48-56](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L48-L56) |
| 固定全局折扣一次性扣除 | 认为在汇总层直接减 | 先分摊到行项目税前扣除，再加回到 DocumentItem.total，最后在汇总层统一扣 | [CreateDocumentItem.php:158-162](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L158-L162) |
| inclusive 税不影响 amount | 认为已含在净额中 | 其分离出的税额通过 `$taxes` 数组累加至 Document.amount | [CreateDocumentItemsAndTotals.php:109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L109) |
| compound 税不影响 amount | 认为无影响 | 其计算出的税额通过 `$taxes` 数组累加至 Document.amount | [CreateDocumentItemsAndTotals.php:109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L109) |
| withholding 税加 abs 值 | 认为总金额 += abs(税额) | Document.amount 使用原始负值，实际减少总金额 | [CreateDocumentItemsAndTotals.php:109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L109) |
| DocumentItem.total 不含全局折扣 | 认为已扣除 | global_discount 在保存前加回，DocumentItem.total 包含折扣加回后的值 | [CreateDocumentItem.php:158-162](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L158-L162) |
| DocumentTotal.tax 与 Document.amount 一致 | 认为税显示值等于 amount 增加额 | DocumentTotal 显示 abs 值，Document.amount 使用原始正负值 | [CreateDocumentItemsAndTotals.php:103-109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L103-L109) |
