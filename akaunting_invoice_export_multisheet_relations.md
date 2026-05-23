# Sales Invoices 多 Sheet Excel 导出分析

## 1. Workbook Sheet 结构与关联方式

### 1.1 包含的 Sheet

导出的 Workbook 包含以下 6 个 Sheet（在 [Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Invoices.php#L25-L35) 中定义）：

| Sheet 名称 | 对应类 | 数据模型 | 角色 |
|-----------|--------|----------|------|
| `invoices` | [Sheets/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php) | `Document` | 主发票表 |
| `invoice_items` | [Sheets/InvoiceItems.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItems.php) | `DocumentItem` | 发票行项目 |
| `invoice_item_taxes` | [Sheets/InvoiceItemTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php) | `DocumentItemTax` | 行项目税 |
| `invoice_histories` | [Sheets/InvoiceHistories.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceHistories.php) | `DocumentHistory` | 发票历史 |
| `invoice_totals` | [Sheets/InvoiceTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTotals.php) | `DocumentTotal` | 发票总计行 |
| `invoice_transactions` | [Sheets/InvoiceTransactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTransactions.php) | `Transaction` | 付款交易记录 |

### 1.2 关联机制

#### 导出时的关联
- **主键**：主发票使用 `id`（Document 表主键）
- **外键**：所有子表通过 `document_id` 关联到主发票
- **`collectForExport` 方法**（[Model.php#L152-L172](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L152-L172)）：
  - 主表：`collectForExport($this->ids, ['document_number' => 'desc'])` 使用 `id` 过滤
  - 子表：`collectForExport($this->ids, null, 'document_id')` 使用 `document_id` 过滤
- **`WithParentSheet` 接口**（[WithParentSheet.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Interfaces/Export/WithParentSheet.php)）：
  - 所有子 Sheet 都实现此接口（空标记接口）
  - 作用：在 `collectForExport` 中跳过分页限制（[Model.php#L168-L170](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L168-L170)），确保导出所有关联数据

#### Sheet 间的连接字段
每个子 Sheet 在 `map()` 方法中都会添加 `invoice_number` 字段（即 `document->document_number`），用于在 Excel 中标识行的所属发票：

```
invoice_number (document_number) → 跨所有 Sheet 的关联键
```

## 2. 各 Sheet 数据来源与字段读取

### 2.1 主发票 (Invoices)
- **模型**：`App\Models\Document\Document`，带 `with('category')` 关联
- **Scope**：`invoice()` 过滤 `type = 'invoice'`
- **映射处理**（[Invoices.php#L17-L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L17-L32)）：
  - `category_name` ← `category->name`
  - `invoice_number` ← `document_number`
  - `invoiced_at` ← `issued_at`
  - `contact_country` ← 国家代码转译后的名称
  - `parent_number` ← 关联的定期发票 `document_number`
- **导出字段**（[Invoices.php#L34-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L34-L65)）：
  `invoice_number`, `order_number`, `status`, `invoiced_at`, `due_at`, `amount`,
  `discount_type`, `discount_rate`, `currency_code`, `currency_rate`, `category_name`,
  `contact_name`, `contact_email`, `contact_tax_number`, `contact_phone`, `contact_address`,
  `contact_country`, `contact_state`, `contact_zip_code`, `contact_city`, `title`,
  `subheading`, `notes`, `footer`, `template`, `color`, `parent_number`

### 2.2 行项目 (InvoiceItems)
- **模型**：`App\Models\Document\DocumentItem`，带 `with('document', 'item')` 关联
- **Scope**：`invoice()` 过滤
- **映射处理**（[InvoiceItems.php#L16-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItems.php#L16-L30)）：
  - `invoice_number` ← `document->document_number`
  - `item_name` ← `item->name`
  - `item_description` ← `item->description`
  - `item_type` ← `item->type`
- **导出字段**：`invoice_number`, `item_name`, `item_description`, `item_type`, `quantity`, `discount_type`, `discount_rate`, `price`, `total`, `tax`

### 2.3 行税 (InvoiceItemTaxes)
- **模型**：`App\Models\Document\DocumentItemTax`，带 `with('document', 'item', 'tax')` 关联
- **Scope**：`invoice()` 过滤
- **映射处理**（[InvoiceItemTaxes.php#L16-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php#L16-L30)）：
  - `invoice_number` ← `document->document_number`
  - `item_name` ← `item->name`
  - `tax_name` ← `tax->name`
  - `tax_rate` ← `tax->rate`
- **导出字段**：`invoice_number`, `item_name`, `tax_name`, `tax_rate`, `amount`

### 2.4 交易记录 (InvoiceTransactions)
- **模型**：`App\Models\Banking\Transaction`，带 `with('account', 'category', 'contact', 'document')` 关联
- **Scope**：`income()->isDocument()` 过滤
- **映射处理**（[InvoiceTransactions.php#L18-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTransactions.php#L18-L33)）：
  - `invoice_number` ← `document->document_number`
  - `account_name` ← `account->name`
  - `category_name` ← `category->name`
  - `contact_email` ← `contact->email`
  - `transaction_number` ← `number`
- **导出字段**：`invoice_number`, `transaction_number`, `paid_at`, `amount`, `currency_code`, `currency_rate`, `account_name`, `contact_email`, `category_name`, `description`, `payment_method`, `reference`, `reconciled`

### 2.5 其他 Sheet
- **InvoiceHistories**：字段为 `invoice_number`, `status`, `notify`, `description`
- **InvoiceTotals**：字段为 `invoice_number`, `code`, `name`, `amount`, `sort_order`

## 3. HeadingsPreparing / RowsPreparing 事件对输出的影响

### 3.1 事件触发位置
- **`HeadingsPreparing`**：在 [Export.php#L103-L108](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L103-L108) 的 `headings()` 方法中触发
  ```php
  public function headings(): array
  {
      event(new HeadingsPreparing($this));
      return $this->fields;
  }
  ```

- **`RowsPreparing`**：在 [Export.php#L110-L115](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L110-L115) 的 `prepareRows()` 方法中触发
  ```php
  public function prepareRows($rows)
  {
      event(new RowsPreparing($this, $rows));
      return $rows;
  }
  ```

### 3.2 事件能否改变最终输出
- **`HeadingsPreparing`**：**可以改变**。事件接收 `$this`（Export 实例），监听器可以修改 `$class->fields` 属性，而 `headings()` 方法直接返回 `$this->fields`。
- **`RowsPreparing`**：**可以改变**。事件接收 `$this` 和 `$rows`，虽然方法返回 `$rows`，但监听器可以通过引用修改 `$rows` 或修改 Export 实例的属性影响后续处理。

**结论**：两个事件都可能改变最终的导出输出，属于扩展点。

## 4. 导出字段能否无损导回导入器

**结论：不能直接无损导回**。导出字段经过了"显示友好"的转换，导入时需要反向解析，存在信息损失风险。

### 需要验证的字段差异

#### 差异 1：名称 vs ID
- **导出**：使用 `category_name`, `contact_name`, `item_name`, `account_name` 等**显示名称**
- **导入**：需要 `category_id`, `contact_id`, `item_id`, `account_id` 等**外键 ID**
- **风险**：导入时通过名称反查 ID（如 `getCategoryId()`, `getContactId()`），如果存在重名或名称已修改，可能匹配错误或失败

#### 差异 2：国家字段的双向翻译
- **导出**：`contact_country` 是**翻译后的国家名称**（如 "China"），代码见 [Invoices.php#L21-L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L21-L23)
- **导入**：需要将国家名称**反向翻译为国家代码**（如 "CN"），代码见 [Invoices.php#L43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Invoices/Sheets/Invoices.php#L43)
- **风险**：翻译不一致、多语言环境差异可能导致反向查找失败

#### 其他潜在差异
- `invoiced_at` 导出时转换为 Excel 日期格式（[Export.php#L81-L83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L81-L83)），导入时需正确解析
- `created_by` 导出为 `owner->email`，导入时无对应处理
- 导出的 `parent_number`（定期发票号）在导入时转换为 `parent_id`，依赖定期发票已存在

## 5. 无 Transactions 但有 Item Taxes 时各 Sheet 行数预期

假设场景：**1 张发票，包含 3 个行项目，其中 2 个行项目各有 1 条税，1 个行项目无税，无付款记录**

| Sheet | 预期行数 | 说明 |
|-------|----------|------|
| `invoices` | **1 行** | 主发票数据，1 张发票 1 行 |
| `invoice_items` | **3 行** | 每个行项目 1 行 |
| `invoice_item_taxes` | **2 行** | 2 个行项目各有 1 条税，共 2 行 |
| `invoice_histories` | **≥ 1 行** | 至少有 1 条创建历史，状态变更可能增加 |
| `invoice_totals` | **≈ 3-5 行** | 通常包含 `sub_total`, `tax`, `total`, 可能有 `discount` 等 |
| `invoice_transactions` | **0 行** | 无付款交易记录 |

**注**：`invoice_totals` 的具体行数取决于系统配置和发票是否有折扣、运费等其他总计项。
