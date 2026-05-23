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

在 [CreateDocument.php::handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocument.php#L24-L26) 中：

```php
if (empty($this->request['amount'])) {
    $this->request['amount'] = 0;
}
```

**关键点**：无论请求中是否有 amount，在创建 Document 前都会被强制设为 0（如果为空的话）。

### 创建流程

```
CreateDocument::handle()
├─ 设置 amount = 0
├─ 创建 Document (amount = 0)
├─ 调度 CreateDocumentItemsAndTotals 子 Job (同步执行)
│  └─ 该子 Job 会修改 $this->request['amount']
└─ 更新 Document: $this->model->update($this->request->all())
   └─ 此时 $this->request['amount'] 已包含计算后的总金额
```

### 回写链路

在 [CreateDocumentItemsAndTotals.php::handle()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L29-L158) 中：

| 步骤 | 代码位置 | 操作 |
|------|---------|------|
| 1 | 第50行 | `$this->request['amount'] += $actual_total;` (初始 0 + actual_total) |
| 2 | 第109行 | `$this->request['amount'] += $tax['amount'];` (累加各税) |
| 3 | 第133-137行 | 处理 extra totals (addition/subtraction) |
| 4 | 第144行 | `$this->request['amount'] = round($this->request['amount'], $precision);` |

**关键机制**：子 Job 直接修改父 Job 的 `$this->request` 对象的 `amount` 属性，父 Job 在子 Job 完成后用修改后的 request 数据更新 Document 模型。这是一种"隐式回写"，没有显式的 return 或参数传递，依赖对象引用。

---

## 2. 折扣计算层级与税基影响

### 折扣分类

| 折扣类型 | 计算层级 | 是否影响税基 | 代码位置 |
|---------|---------|------------|---------|
| 行折扣 (百分比) | DocumentItem 层 | ✅ 影响 | [CreateDocumentItem.php:40-41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L40-L41) |
| 行折扣 (固定) | DocumentItem 层 | ✅ 影响 | [CreateDocumentItem.php:42-44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L42-L44) |
| 全局折扣 (百分比) | DocumentTotal 层 | ❌ 不影响 | [CreateDocumentItemsAndTotals.php:72-74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L72-L74) |
| 全局折扣 (固定) | DocumentItem 层 (分摊) | ✅ 影响 | [CreateDocumentItemsAndTotals.php:170-186](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L170-L186) |

### 行折扣计算

```php
// CreateDocumentItem.php:39-45
if (! empty($this->request['discount'])) {
    if ($this->request['discount_type'] === 'percentage') {
        $item_discounted_amount -= ($item_amount * ($item['discount'] / 100));
    } else {
        $item_discounted_amount -= $this->request['discount'];
    }
}
```

**注意**：行折扣直接作用于 `$item_discounted_amount`，这是后续税计算的基础，因此**会影响税基**。

### 全局折扣计算

#### 百分比全局折扣

```php
// CreateDocumentItemsAndTotals.php:72-74
if ($this->request['discount_type'] === 'percentage') {
    $discount_total = $sub_total * ($this->request['discount'] / 100);
}
```

**关键点**：
- 基于 `$sub_total`（所有行项目金额之和，已扣行折扣，但未扣全局折扣）
- 只在 DocumentTotal 层面体现，**不影响 item 级别的税基**
- 注释掉了 `($sub_total - $discount_amount_total)` 的版本，说明之前存在 bug

#### 固定全局折扣

固定全局折扣的处理比较特殊，分为两步：

**第一步：分摊到行项目**（[fixedDiscountCalculate()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L265-L289)）

```php
// 按行项目金额占比分摊全局折扣
$item['global_discount'] = ($for_fixed_discount[$key] / ($for_fixed_discount['total'] / 100)) * ($this->request['discount'] / 100);
```

**第二步：在 DocumentItem 层扣减**

```php
// CreateDocumentItem.php:48-56
if (! empty($this->request['global_discount'])) {
    if ($this->request['global_discount_type'] === 'percentage') {
        $global_discount = $item_discounted_amount * ($this->request['global_discount'] / 100);
    } else {
        $global_discount = $this->request['global_discount'];
    }
    $item_discounted_amount -= $global_discount;
}
```

**关键点**：固定全局折扣会被分摊到每个行项目的 `global_discount`，并在税前扣除，因此**会影响税基**。

---

## 3. 五类税的计算逻辑与体现

税的计算全部在 [CreateDocumentItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L69-L156) 中完成，执行顺序为：`inclusive → fixed → normal → withholding → compound`。

### 基础变量定义

| 变量 | 含义 |
|------|------|
| `$item_amount` | price × quantity，行项目原始金额 |
| `$item_discounted_amount` | 扣减行折扣和全局折扣后的金额 |
| `$actual_price_item` | 扣减 inclusive 税后的净额（税基） |
| `$item_tax_total` | 所有税的合计（含正负） |

### 3.1 inclusive (含税)

**计算**：
```php
$tax_amount = $item_discounted_amount - ($item_discounted_amount / (1 + $inclusive->rate / 100));
$actual_price_item = $item_discounted_amount - $item_tax_total;
```

**特点**：
- 从 `$item_discounted_amount` 中**分离**出税额（价格已含税）
- 降低 `$actual_price_item`（后续税的税基）
- **不增加** `$item_amount`
- DocumentItem.total = `$actual_price_item`（不含税金额）
- DocumentTotal.tax 中体现为正数

### 3.2 fixed (固定税)

**计算**：
```php
$tax_amount = $tax->rate * (double) $this->request['quantity'];
$item_amount += $tax_amount;
```

**特点**：
- 按数量计税，与金额无关
- **增加** `$item_amount`
- DocumentTotal.tax 中体现为正数

### 3.3 normal (普通税)

**计算**：
```php
$tax_amount = $actual_price_item * ($tax->rate / 100);
$item_amount += $tax_amount;
```

**特点**：
- 基于 `$actual_price_item`（已扣 inclusive 税）
- **增加** `$item_amount`
- DocumentTotal.tax 中体现为正数

### 3.4 withholding (预提税)

**计算**：
```php
$tax_amount = -($actual_price_item * ($tax->rate / 100));
$item_amount += $tax_amount;
```

**特点**：
- 基于 `$actual_price_item`（已扣 inclusive 税）
- **负值**，减少 `$item_amount`
- DocumentItemTax.amount 存储为 `abs($tax_amount)`
- DocumentTotal.tax 中体现为正数（取 abs）
- 但在计算 Document.amount 时使用原始负值（减少总金额）

### 3.5 compound (复合税)

**计算**：
```php
$tax_amount = ($item_amount / 100) * $compound->rate;
```

**特点**：
- 基于 `$item_amount`（此时已包含 fixed、normal、withholding 税）
- **不增加** `$item_amount`（代码中没有 `$item_amount += $tax_amount`）
- DocumentTotal.tax 中体现为正数

### 税在各层级的体现汇总

| 税类型 | DocumentItem.total | DocumentItem.tax | DocumentItemTax.amount | DocumentTotal.tax | Document.amount |
|--------|-------------------|-----------------|-----------------------|-------------------|----------------|
| inclusive | 不含税净额 | 包含 | 正数 | 正数 | +0（已含在净额） |
| fixed | 不含税净额 | 包含 | 正数 | 正数 | +税额 |
| normal | 不含税净额 | 包含 | 正数 | 正数 | +税额 |
| withholding | 不含税净额 | 包含（负） | 正数（abs） | 正数（abs） | -税额 |
| compound | 不含税净额 | 包含 | 正数 | 正数 | +0（未加） |

---

## 4. 反例：容易算错的场景

### 场景设定

创建一张 Invoice，包含：
- **商品 A**：价格 $100，数量 1，无行折扣
- **商品 B**：价格 $200，数量 1，无行折扣
- **全局折扣**：固定 $15
- **税 1**：VAT (inclusive)，税率 10%（应用于商品 A）
- **税 2**：WHT (withholding)，税率 5%（应用于商品 A）

### 期望的直观计算（易错版本）

```
商品 A: $100
商品 B: $200
小计: $300
全局折扣: -$15
折扣后小计: $285
VAT (含税 10%，基于 $285): $25.91
WHT (5%，基于 $285 - $25.91 = $259.09): -$12.95
总计: $285 - $12.95 = $272.05
```

### 实际计算流程（Akaunting 版本）

#### 第一步：固定全局折扣分摊

```
商品 A 占比: 100 / (100 + 200) = 33.33%
商品 B 占比: 200 / (100 + 200) = 66.67%
商品 A 分摊: $15 × 33.33% = $5
商品 B 分摊: $15 × 66.67% = $10
```

#### 第二步：商品 A 计算

```
item_amount = 100 × 1 = $100
global_discount = $5
item_discounted_amount = 100 - 5 = $95

// inclusive 税 (10%)
tax_amount = 95 - (95 / 1.1) = $8.64
actual_price_item = 95 - 8.64 = $86.36
item_tax_total = $8.64

// withholding 税 (5%)
tax_amount = -(86.36 × 5%) = -$4.32
item_tax_total = 8.64 - 4.32 = $4.32
item_amount = 95 - 4.32 = $90.68  // 注意: item_amount 被修改为 95 - 4.32

DocumentItem.total = $86.36
DocumentItem.tax = $4.32
```

#### 第三步：商品 B 计算

```
item_amount = 200 × 1 = $200
global_discount = $10
item_discounted_amount = 200 - 10 = $190

// 无税
actual_price_item = $190
item_tax_total = $0

DocumentItem.total = $190
DocumentItem.tax = $0
```

#### 第四步：DocumentTotal 汇总

```
sub_total = 86.36 + 190 = $276.36

// 行折扣合计: $0
// 全局折扣: $15 (单独创建 DocumentTotal)

// 税汇总:
// VAT: $8.64
// WHT: $4.32 (abs)
// 税合计: $12.96

// amount 计算:
// actual_total = 90.68 + 190 = $280.68
// 加税: 280.68 + (-4.32) = $276.36  // 注意: WHT 使用原始负值
// 最终 amount = $276.36
```

### 易错点总结

| 易错点 | 直观错误 | 实际行为 | 代码位置 |
|--------|---------|---------|---------|
| 全局固定折扣的税基影响 | 认为折扣在税后扣除 | 分摊到行项目，税前扣除，影响税基 | [CreateDocumentItemsAndTotals.php:170-186](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L170-L186) |
| inclusive 税的税基 | 认为基于折扣后总金额 | 基于**单个行项目**折扣后金额 | [CreateDocumentItem.php:83-95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L83-L95) |
| withholding 税的符号 | 认为总金额 += abs(税额) | 总金额 += 原始负值（减少总金额） | [CreateDocumentItem.php:128-140](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L128-L140) |
| compound 税的税基 | 认为基于不含税金额 | 基于已包含 fixed/normal/withholding 的金额 | [CreateDocumentItem.php:143-155](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L143-L155) |
| DocumentItem.total | 认为包含税 | 只包含 actual_price_item（不含税） | [CreateDocumentItem.php:176](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItem.php#L176) |
| 百分比全局折扣的税基 | 认为基于扣减行折扣后的金额 | 基于 sub_total（未扣行折扣？需看 sub_total 定义） | [CreateDocumentItemsAndTotals.php:74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php#L74) |

### 反例最终结果对比

| 项目 | 直观计算 (错误) | Akaunting 实际 | 差异 |
|------|---------------|---------------|------|
| 商品 A 税基 | $95 (100 - 5) | $95 | 一致 |
| VAT 税额 | $8.64 | $8.64 | 一致 |
| WHT 税额 | -$4.32 | -$4.32 | 一致 |
| 商品 A total | $86.36 | $86.36 | 一致 |
| 商品 B total | $190 | $190 | 一致 |
| sub_total | $276.36 | $276.36 | 一致 |
| 全局折扣 | $15 | $15 | 一致 |
| 税合计 | $4.32 | $12.96 (显示) / $4.32 (实际) | 显示差异 |
| 最终 amount | $272.05 | $276.36 | **+$4.31** |

**差异原因**：直观计算错误地在 discount 后再扣 WHT，而 Akaunting 中 WHT 已在 item 层面从 amount 中扣除，全局折扣是独立的 DocumentTotal 记录。
