# Sales Invoices 多 Sheet Excel 导出分析

## 1. Workbook Sheet 结构与 ID 传递机制

### 1.1 包含的 Sheet

导出的 Workbook 包含以下 6 个 Sheet（在 [Invoices.php#L25-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Invoices.php#L25-L35) 的 `sheets()` 方法中定义）：

| Sheet 名称 | 对应类 | 数据模型 | 角色 |
|-----------|--------|----------|------|
| `invoices` | [Sheets/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php) | `Document` | 主发票表 |
| `invoice_items` | [Sheets/InvoiceItems.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItems.php) | `DocumentItem` | 发票行项目 |
| `invoice_item_taxes` | [Sheets/InvoiceItemTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php) | `DocumentItemTax` | 行项目税 |
| `invoice_histories` | [Sheets/InvoiceHistories.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceHistories.php) | `DocumentHistory` | 发票历史 |
| `invoice_totals` | [Sheets/InvoiceTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTotals.php) | `DocumentTotal` | 发票总计行 |
| `invoice_transactions` | [Sheets/InvoiceTransactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTransactions.php) | `Transaction` | 付款交易记录 |

### 1.2 ID 传递机制

#### 1.2.1 入口：`toExcel()` 的 ids 初始化

当调用 [Utilities/Export.php#L20-L56](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Export.php#L20-L56) 的 `toExcel()` 时：

```php
if (empty($class->ids) && method_exists($class, 'sheets') && is_array($sheets = $class->sheets())) {
    $class->ids = ! empty($ids = (new $sheets[array_key_first($sheets)])->collection()->pluck('id')->toArray()) ? $ids : [0];
}
```

- 如果 `$class->ids` 为空，取第一个 Sheet（即 `Invoices` 主表）的 collection，pluck 所有 `id` 作为 ids
- 这样子 Sheet 可以通过相同的 `$this->ids` 过滤关联数据

#### 1.2.2 `scopeCollectForExport` 的 id 过滤

[Model.php#L152-L172](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L152-L172) 中的 `collectForExport` 方法：

- **主表**：`collectForExport($this->ids, ['document_number' => 'desc'])` → 默认 `$id_field = 'id'`，用 `id` 过滤
- **子表**：`collectForExport($this->ids, null, 'document_id')` → `$id_field = 'document_id'`，用 `document_id` 过滤

关键代码：
```php
if (!empty($ids)) {
    $query->whereIn($id_field, (array) $ids);
}
```

#### 1.2.3 `WithParentSheet` 接口对分页的影响

[WithParentSheet.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Interfaces/Export/WithParentSheet.php) 是空标记接口，所有子 Sheet 都实现它。

在 `collectForExport` 中（[Model.php#L168-L170](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L168-L170)）：
```php
if (! $this instanceof WithParentSheet && (empty($ids) || count((array) $ids) > $limit)) {
    $query->offset($offset)->limit($limit);
}
```

- **主表**（不实现 `WithParentSheet`）：当 ids 为空或超过 limit 时应用分页
- **子表**（实现 `WithParentSheet`）：始终跳过分页，导出所有匹配的关联数据

#### 1.2.4 Sheet 间的连接字段

每个子 Sheet 在 `map()` 方法中都会添加 `invoice_number` 字段（即 `document->document_number`），用于在 Excel 中标识行的所属发票：

```
invoice_number (document_number) → 跨所有 Sheet 的关联键
```

## 2. 各 Sheet 数据来源与字段读取

### 2.1 主发票 (Invoices)
- **模型**：`App\Models\Document\Document`，带 `with('category')` 关联
- **Scope**：`invoice()` 过滤 `type = 'invoice'`
- **collection()**（[Invoices.php#L12-L15](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L12-L15)）：
  ```php
  return Model::with('category')->invoice()->collectForExport($this->ids, ['document_number' => 'desc']);
  ```
- **map() 映射处理**（[Invoices.php#L17-L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L17-L32)）：
  - `category_name` ← `category->name`
  - `invoice_number` ← `document_number`
  - `invoiced_at` ← `issued_at`
  - `contact_country` ← 国家代码转译后的名称（通过 `trans('countries.' . $model->contact_country)`）
  - `parent_number` ← `Model::invoiceRecurring()->find($model->parent_id)?->document_number`
- **导出字段**（[Invoices.php#L34-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L34-L65)）：
  `invoice_number`, `order_number`, `status`, `invoiced_at`, `due_at`, `amount`,
  `discount_type`, `discount_rate`, `currency_code`, `currency_rate`, `category_name`,
  `contact_name`, `contact_email`, `contact_tax_number`, `contact_phone`, `contact_address`,
  `contact_country`, `contact_state`, `contact_zip_code`, `contact_city`, `title`,
  `subheading`, `notes`, `footer`, `template`, `color`, `parent_number`

### 2.2 行项目 (InvoiceItems)
- **模型**：`App\Models\Document\DocumentItem`，带 `with('document', 'item')` 关联
- **Scope**：`invoice()` 过滤
- **collection()**（[InvoiceItems.php#L11-L14](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItems.php#L11-L14)）：
  ```php
  return Model::with('document', 'item')->invoice()->collectForExport($this->ids, null, 'document_id');
  ```
- **map() 映射处理**（[InvoiceItems.php#L16-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItems.php#L16-L30)）：
  - `invoice_number` ← `document->document_number`
  - `item_name` ← `item->name`
  - `item_description` ← `item->description`
  - `item_type` ← `item->type`
- **导出字段**：`invoice_number`, `item_name`, `item_description`, `item_type`, `quantity`, `discount_type`, `discount_rate`, `price`, `total`, `tax`

### 2.3 行税 (InvoiceItemTaxes)
- **模型**：`App\Models\Document\DocumentItemTax`，带 `with('document', 'item', 'tax')` 关联
- **Scope**：`invoice()` 过滤
- **collection()**（[InvoiceItemTaxes.php#L11-L14](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php#L11-L14)）：
  ```php
  return Model::with('document', 'item', 'tax')->invoice()->collectForExport($this->ids, null, 'document_id');
  ```
- **map() 映射处理**（[InvoiceItemTaxes.php#L16-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php#L16-L30)）：
  - `invoice_number` ← `document->document_number`
  - `item_name` ← `item->name`
  - `tax_name` ← `tax->name`
  - `tax_rate` ← `tax->rate`
- **导出字段**：`invoice_number`, `item_name`, `tax_name`, `tax_rate`, `amount`
- **注意**：导出字段中**缺少 `document_item_id`**，该字段是 `document_item_taxes` 表关联到 `document_items` 表的外键

### 2.4 交易记录 (InvoiceTransactions)
- **模型**：`App\Models\Banking\Transaction`，带 `with('account', 'category', 'contact', 'document')` 关联
- **Scope**：`income()->isDocument()` 过滤
- **collection()**（[InvoiceTransactions.php#L13-L16](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTransactions.php#L13-L16)）：
  ```php
  return Model::with('account', 'category', 'contact', 'document')->income()->isDocument()->collectForExport($this->ids, ['paid_at' => 'desc'], 'document_id');
  ```
- **map() 映射处理**（[InvoiceTransactions.php#L18-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTransactions.php#L18-L33)）：
  - `invoice_number` ← `document->document_number`
  - `account_name` ← `account->name`
  - `category_name` ← `category->name`
  - `contact_email` ← `contact->email`
  - `transaction_number` ← `number`
- **导出字段**：`invoice_number`, `transaction_number`, `paid_at`, `amount`, `currency_code`, `currency_rate`, `account_name`, `contact_email`, `category_name`, `description`, `payment_method`, `reference`, `reconciled`

### 2.5 其他 Sheet
- **InvoiceHistories**（[InvoiceHistories.php#L29-L37](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceHistories.php#L29-L37)）：字段为 `invoice_number`, `status`, `notify`, `description`
- **InvoiceTotals**（[InvoiceTotals.php#L30-L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceTotals.php#L30-L39)）：字段为 `invoice_number`, `code`, `name`, `amount`, `sort_order`

## 3. HeadingsPreparing / RowsPreparing 事件对输出的影响

### 3.1 事件触发位置

- **`HeadingsPreparing`**：在 [Export.php#L103-L108](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L103-L108) 的 `headings()` 方法中触发：
  ```php
  public function headings(): array
  {
      event(new HeadingsPreparing($this));
      return $this->fields;
  }
  ```

- **`RowsPreparing`**：在 [Export.php#L110-L115](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L110-L115) 的 `prepareRows()` 方法中触发：
  ```php
  public function prepareRows($rows)
  {
      event(new RowsPreparing($this, $rows));
      return $rows;
  }
  ```

### 3.2 HeadingsPreparing 能否改变输出

**可以改变**。事件接收 `$this`（Export 实例的引用），监听器可以修改 `$class->fields` 属性。`headings()` 方法直接返回 `$this->fields`，因此修改会生效。

### 3.3 RowsPreparing 能否改变输出

**不能通过替换事件属性直接改变返回行**。分析事件类 [RowsPreparing.php#L1-L24](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Export/RowsPreparing.php#L1-L24)：

```php
class RowsPreparing extends Event
{
    public $class;
    public $rows;

    public function __construct($class, $rows)
    {
        $this->class = $class;
        $this->rows = $rows;
    }
}
```

- `$rows` 是通过值传递给构造函数，事件保存的是值的副本
- `prepareRows()` 返回的是原始的 `$rows` 参数，而非 `$event->rows`
- 因此监听器修改 `$event->rows`（如 `$event->rows = []`）不会影响返回值
- 但监听器可以通过修改 `$event->class`（Export 实例）的属性来间接影响后续处理

**结论**：`HeadingsPreparing` 可以改变表头；`RowsPreparing` 不能通过替换事件属性直接改变行数据。

## 4. 导出字段能否无损导回导入器

**结论：不能直接无损导回**。导出字段经过了转换，导入时需要反向解析，存在信息损失。

### 4.1 差异 1：名称 vs ID

- **导出**：使用 `category_name`, `contact_name`, `item_name`, `account_name` 等显示名称
- **导入**：需要 `category_id`, `contact_id`, `item_id`, `account_id` 等外键 ID
- **导入处理**（如 [InvoiceItems.php#L48-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Invoices/Sheets/InvoiceItems.php#L48-L54)）：通过名称反查 ID
  ```php
  if (empty($row['item_id']) && ! empty($row['item_name'])) {
      $row['item_id'] = $this->getItemIdFromName($row);
      $row['name'] = $row['item_name'];
  }
  ```
- **风险**：如果存在重名或名称已修改，可能匹配错误或失败

### 4.2 差异 2：DocumentItemTax 缺少 document_item_id

- **导出**（[InvoiceItemTaxes.php#L32-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/InvoiceItemTaxes.php#L32-L41)）：导出字段中**没有 `document_item_id`**
- **数据库表结构**（[2019_11_16_000000_core_v2.php#L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L192)）：`document_item_taxes` 表有 `document_item_id` 字段
- **模型关系**（[DocumentItem.php#L83-L86](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/DocumentItem.php#L83-L86)）：`DocumentItem` 通过 `document_item_id` 关联 `DocumentItemTax`
- **导入处理**（[InvoiceItemTaxes.php#L53-L62](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Invoices/Sheets/InvoiceItemTaxes.php#L53-L62)）：通过 `item_name` 反查 `document_item_id`
  ```php
  if (empty($row['document_item_id']) && !empty($row['item_name'])) {
      $item_id = Item::name($row['item_name'])->whereIn('id', $document_items_ids)->pluck('id')->first();
      $row['document_item_id'] = DocumentItem::invoice()
          ->where('document_id', $row['document_id'])
          ->where('item_id', $item_id)
          ->pluck('id')
          ->first();
  }
  ```
- **风险**：如果同一商品在发票中有多行（相同 `item_id`），`pluck('id')->first()` 只会返回第一行的 ID，导致税行被错误地关联到同一行项目

### 4.3 差异 3：日期字段丢失时间部分

- **导出处理**（[Export.php#L69-L82](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Export.php#L69-L82)）：
  ```php
  $date_fields = ['paid_at', 'invoiced_at', 'billed_at', 'due_at', 'issued_at', 'transferred_at'];
  foreach ($this->fields as $field) {
      $value = $model->$field;
      if (in_array($field, $date_fields)) {
          $value = ExcelDate::PHPToExcel(Date::parse($value)->format('Y-m-d'));
      }
  }
  ```
- `Date::parse($value)->format('Y-m-d')` 只保留日期部分，**丢失了时间部分**（时、分、秒）
- 原始字段（如 `issued_at`, `paid_at`）在数据库中是 datetime 类型，包含完整时间戳

### 4.4 其他差异

- **国家字段双向翻译**：
  - 导出（[Invoices.php#L21-L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Sales/Invoices/Sheets/Invoices.php#L21-L23)）：`contact_country` 是翻译后的国家名称
  - 导入（[Invoices.php#L43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Invoices/Sheets/Invoices.php#L43)）：通过 `array_search($row['contact_country'], trans('countries'))` 反向查找国家代码
  - **风险**：翻译不一致可能导致反向查找失败

- **parent_number → parent_id**：导出的 `parent_number`（定期发票号）在导入时转换为 `parent_id`，依赖定期发票已存在

## 5. 无 Transactions 但有 Item Taxes 时各 Sheet 行数预期

假设场景：**1 张发票，包含 3 个行项目，其中 2 个行项目各有 1 条税，1 个行项目无税，无付款记录**

| Sheet | 预期行数 | 依据 |
|-------|----------|------|
| `invoices` | **1 行** | 每张发票 1 行，来自 `documents` 表 |
| `invoice_items` | **3 行** | 每个行项目 1 行，来自 `document_items` 表 |
| `invoice_item_taxes` | **2 行** | 2 个行项目各有 1 条税，来自 `document_item_taxes` 表 |
| `invoice_histories` | **取决于已有历史** | 见下文详细说明 |
| `invoice_totals` | **≈ 3-5 行** | 通常包含 `sub_total`, `tax`, `total`，可能有 `discount` 等 |
| `invoice_transactions` | **0 行** | 无付款交易记录 |

### invoice_histories 行数的可达性差异

`invoice_histories` 的行数取决于 `document_histories` 表中已有的记录，不同创建方式产生的历史记录不同：

1. **普通创建（通过 UI）**：通常有 1 条创建记录（`status = 'draft'` 或 `status = 'issued'`），状态变更会追加记录
2. **通过导入器导入**（[InvoiceHistories.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Sales/Invoices/Sheets/InvoiceHistories.php)）：导入时会创建历史记录，数量取决于导入文件中的历史行数
3. **直接模型或数据库写入**：如果绕过业务逻辑直接写入数据库，可能没有任何历史记录

**注**：`invoice_totals` 的具体行数取决于系统配置和发票是否有折扣、运费等其他总计项。
