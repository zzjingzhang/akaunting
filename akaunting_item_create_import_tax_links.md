# Item 创建与导入时税关联建立机制分析

---

## 1. CreateItem 事务流程：图片与 taxes 处理位置

### 1.1 事务执行顺序

**代码位置**：[CreateItem.php#L16-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php#L16-L36)

```php
public function handle(): Item
{
    event(new ItemCreating($this->request));

    \DB::transaction(function () {
        // 第一步：创建 Item 主记录
        $this->model = Item::create($this->request->all());

        // 第二步：处理图片上传
        if ($this->request->file('picture')) {
            $media = $this->getMedia($this->request->file('picture'), 'items');
            $this->model->attachMedia($media, 'picture');
        }

        // 第三步：处理税关联
        $this->dispatch(new CreateItemTaxes($this->model, $this->request));
    });

    event(new ItemCreated($this->model, $this->request));

    return $this->model;
}
```

### 1.2 处理位置总结

| 操作 | 位置 | 说明 |
|------|------|------|
| **Item 主记录创建** | [CreateItem.php#L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php#L21) | 事务内第一步，`Item::create($this->request->all())` |
| **图片处理** | [CreateItem.php#L23-L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php#L23-L28) | 事务内第二步，在 Item 创建后、税关联前处理 |
| **税关联处理** | [CreateItem.php#L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php#L30) | 事务内第三步，通过 `CreateItemTaxes` Job 处理 |

---

## 2. tax_id 与 tax_ids 兼容逻辑及边界行为

### 2.1 兼容逻辑代码证据

**代码位置**：[CreateItemTaxes.php#L32-L56](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItemTaxes.php#L32-L56)

```php
public function handle()
{
    // BC for 2.0 version — 向后兼容，单一 tax_id 转为数组
    if (!empty($this->request['tax_id'])) {
        $this->request['tax_ids'][] = $this->request['tax_id'];
    }

    if (empty($this->request['tax_ids'])) {
        return false;
    }

    \DB::transaction(function () {
        foreach ($this->request['tax_ids'] as $tax_id) {
            ItemTax::create([
                'company_id' => $this->item->company_id,
                'item_id' => $this->item->id,
                'tax_id' => $tax_id,
                'created_from' => $this->request['created_from'],
                'created_by' => $this->request['created_by'],
            ]);
        }
    });

    return $this->item->taxes;
}
```

**Item 请求校验规则**：[Item.php#L43-L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/Item.php#L43-L53)

```php
return [
    'type'              => 'required|string|in:product,service',
    'name'              => 'required|string',
    'sale_price'        => $sale_price . '|regex:/^(?=.*?[0-9])[0-9.,]+$/',
    'purchase_price'    => $purchase_price . '|regex:/^(?=.*?[0-9])[0-9.,]+$/',
    'tax_ids'           => 'nullable|array',    // ← 仅校验是数组，不校验元素合法性
    'category_id'       => 'nullable|integer',
    'enabled'           => 'integer|boolean',
    'picture'           => $picture,
];
```

**item_taxes 表结构**：[2019_11_16_000000_core_v2.php#L296-L307](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L296-L307)

```php
Schema::create('item_taxes', function (Blueprint $table) {
    $table->increments('id');
    $table->integer('company_id');
    $table->integer('item_id');
    $table->integer('tax_id')->nullable();        // ← 可空，无外键约束
    $table->string('created_from', 100)->nullable();
    $table->unsignedInteger('created_by')->nullable();
    $table->timestamps();
    $table->softDeletes();

    $table->index(['company_id', 'item_id']);     // ← 仅普通索引，无 UNIQUE
});
```

### 2.2 边界行为详细分析

| 场景 | 可达条件 | 代码路径 | 结果 |
|------|----------|----------|------|
| **仅传 tax_id** | `!empty($request['tax_id'])` 为 true | L35-L37: 追加到 tax_ids | 税关联正常创建，兼容旧版 |
| **仅传 tax_ids** | `$request['tax_id']` 为空，`$request['tax_ids']` 非空 | 跳过 L35-L37，直接进入 foreach | 正常创建多个税关联 |
| **同时传 tax_id + tax_ids** | 两者均非空 | L35-L37: tax_id **追加**到 tax_ids 末尾 | **可能产生重复** — 若 tax_ids 中已包含该 tax_id，同一 (item_id, tax_id) 会被插入两次（表无 UNIQUE 约束） |
| **两者都不传 或 tax_ids 为空数组** | `empty($request['tax_ids'])` 为 true | L39-L41: return false | 不创建任何 ItemTax，Item 无税关联 |
| **tax_ids 含无效/不存在的 tax_id** | `$tax_id` 是整数但 taxes 表无对应记录 | L44-L51: 直接 `ItemTax::create()`，无存在性校验 | **脏数据写入** — item_taxes 表记录了指向不存在 tax 的 ID，后续 `$item->taxes->tax` 关系返回默认值（name='N/A', rate=0） |
| **tax_ids 含 null/空字符串元素** | 数组中包含 `null` 或 `''` | L44-L51: foreach 遍历每个元素，`ItemTax::create(['tax_id' => null])` | **可写入空 tax_id** — 因 `tax_id` 列为 `nullable()`，无 NOT NULL 约束，也无外键约束 |
| **tax_ids 含重复元素** | `[1, 1, 2]` 等 | L44-L51: 逐条插入，无去重 | **多条重复记录** — 同一 item 对同一 tax 有多条 ItemTax，后续 `$item->taxes` 返回重复集合 |

**关键校验限制总结**：
1. `tax_ids` 仅校验 `nullable\|array` — 不校验元素类型（integer vs string），不校验存在性，不校验唯一性
2. `item_taxes` 表无 `UNIQUE(item_id, tax_id)` 约束 — 允许重复
3. `item_taxes.tax_id` 为 `nullable()` 且无外键 — 允许 null 和无效 ID
4. 兼容逻辑 L35-L37 不做去重判断 — 盲目追加可能导致重复

---

## 3. 导入 Items 与 ItemTaxes 两个 sheet 的对应关系

### 3.1 导入架构

**代码位置**：[Items.php (Import)](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Items.php#L9-L18)

```php
class Items extends ImportMultipleSheets
{
    public function sheets(): array
    {
        return [
            'items' => new Base(),        // Items sheet 处理器
            'item_taxes' => new ItemTaxes(), // ItemTaxes sheet 处理器
        ];
    }
}
```

### 3.2 Items sheet 处理逻辑

**代码位置**：[Sheets/Items.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Sheets/Items.php#L9-L43)

```php
class Items extends Import
{
    public $columns = [
        'type',
        'name',
        'sale_price',
        'purchase_price',
    ];

    public function model(array $row)
    {
        if (self::hasRow($row)) {   // ← 按 type+name+sale_price+purchase_price 查重
            return;
        }

        return new Model($row);
    }

    public function map($row): array
    {
        $row = parent::map($row);   // ← 注入 company_id, created_by, created_from

        $row['sale_information'] = isset($row['sale_price']) ?? false;
        $row['purchase_information'] = isset($row['purchase_price']) ?? false;
        $row['category_id'] = $this->getCategoryId($row, 'item');

        return $row;
    }
}
```

- 直接导入 Item 主表数据，不处理任何税关联
- 通过 `parent::map()` 注入 `company_id`、`created_by`、`created_from`
- `hasRow()` 查重字段为 `[type, name, sale_price, purchase_price]` — 若四字段完全一致的 Item 已存在，跳过

### 3.3 ItemTaxes sheet 处理逻辑

**代码位置**：[Sheets/ItemTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Sheets/ItemTaxes.php#L9-L43)

```php
class ItemTaxes extends Import
{
    public $columns = [
        'item_id',
        'tax_id'
    ];

    public function model(array $row)
    {
        if (self::hasRow($row)) {   // ← 按 item_id+tax_id 查重
            return;
        }

        return new Model($row);
    }

    public function map($row): array
    {
        if ($this->isEmpty($row, 'item_name')) {   // ← item_name 为空则整行跳过
            return [];
        }

        $row = parent::map($row);                  // ← 注入 company_id, created_by, created_from

        $row['item_id'] = $this->getItemId($row);  // ← 解析 item_id
        $row['tax_id'] = $this->getTaxId($row);    // ← 解析 tax_id

        return $row;
    }
}
```

### 3.4 getItemId() 解析链路

**代码位置**：[Import.php Trait#L242-L253](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L242-L253)

```php
public function getItemId($row, $type = null)
{
    $id = isset($row['item_id']) ? $row['item_id'] : null;          // ① 先用 item_id

    $type = !empty($type) ? $type : (!empty($row['item_type']) ? $row['item_type'] : 'product');  // ② 默认为 'product'

    if (empty($id) && !empty($row['item_name'])) {                  // ③ item_id 为空且 item_name 非空
        $id = $this->getItemIdFromName($row, $type);                //    按 item_name + type 查找/创建
    }

    return is_null($id) ? $id : (int) $id;
}
```

**getItemIdFromName() 查找/创建逻辑**：[Import.php Trait#L428-L455](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L428-L455)

```php
public function getItemIdFromName($row, $type = null)
{
    $type = !empty($type) ? $type : (!empty($row['item_type']) ? $row['item_type'] : 'product');

    $item_id = Item::type($type)->where('name', $row['item_name'])->pluck('id')->first();

    if (!empty($item_id)) {
        return $item_id;                                            // ← 找到则直接返回
    }

    // 找不到 → 创建新 Item
    $data = [
        'company_id'        => company_id(),
        'type'              => $type,                               // ← 默认为 'product'
        'name'              => $row['item_name'],
        'description'       => !empty($row['item_description']) ? $row['item_description'] : null,
        'sale_price'        => !empty($row['sale_price']) ? $row['sale_price'] : (!empty($row['price']) ? $row['price'] : 0),
        'purchase_price'    => !empty($row['purchase_price']) ? $row['purchase_price'] : (!empty($row['price']) ? $row['price'] : 0),
        'enabled'           => 1,
        'created_from'      => !empty($row['created_from']) ? $row['created_from'] : $this->getSourcePrefix() . 'import',
        'created_by'        => !empty($row['created_by']) ? $row['created_by'] : user()?->id,
    ];

    Validator::validate($data, (new ItemRequest())->rules());

    $item = $this->dispatch(new CreateItem($data));                 // ← 完整走 CreateItem 事务

    return $item->id;
}
```

### 3.5 getTaxId() 解析链路

**代码位置**：[Import.php Trait#L255-L268](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L255-L268)

```php
public function getTaxId($row)
{
    $id = isset($row['tax_id']) ? $row['tax_id'] : null;           // ① 先用 tax_id

    if (empty($id) && !empty($row['tax_name'])) {                  // ② tax_id 为空且 tax_name 非空
        $id = Tax::name($row['tax_name'])->pluck('id')->first();   //    按 tax_name 查找
    }

    if (empty($id) && !empty($row['tax_rate'])) {                  // ③ tax_id 和 tax_name 都为空但 tax_rate 非空
        $id = $this->getTaxIdFromRate($row);                       //    按 tax_rate 查找/创建
    }

    return is_null($id) ? $id : (int) $id;
}
```

**getTaxIdFromRate() 查找/创建逻辑**：[Import.php Trait#L457-L480](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php#L457-L480)

```php
public function getTaxIdFromRate($row, $type = 'normal')         // ← 固定默认 type='normal'
{
    $tax_id = Tax::type($type)->where('rate', $row['tax_rate'])->pluck('id')->first();

    if (!empty($tax_id)) {
        return $tax_id;                                            // ← 找到则直接返回
    }

    // 找不到 → 创建新 Tax
    $data = [
        'company_id'        => company_id(),
        'rate'              => $row['tax_rate'],
        'type'              => $type,                              // ← 固定 'normal'
        'name'              => !empty($row['tax_name']) ? $row['tax_name'] : (string) $row['tax_rate'],
        'enabled'           => 1,
        'created_from'      => !empty($row['created_from']) ? $row['created_from'] : $this->getSourcePrefix() . 'import',
        'created_by'        => !empty($row['created_by']) ? $row['created_by'] : user()?->id,
    ];

    Validator::validate($data, (new TaxRequest())->rules());

    $tax = $this->dispatch(new CreateTax($data));

    return $tax->id;
}
```

### 3.6 对应关系建立机制

```
ItemTaxes sheet 每行的 map() 执行顺序：

1. isEmpty(item_name) → 为空则跳过整行
2. parent::map()     → 注入 company_id, created_by, created_from
3. getItemId($row)   → 解析 item_id：
   ├─ ① $row['item_id'] 非空 → 直接使用
   ├─ ② $row['item_id'] 空 + $row['item_name'] 非空 → getItemIdFromName()
   │   ├─ 按 type(默认'product') + item_name 查 items 表
   │   ├─ 找到 → 返回 id
   │   └─ 找不到 → CreateItem() 创建新 Item → 返回新 id
   └─ ③ 都为空 → 返回 null（ItemTax 校验会失败）
4. getTaxId($row)    → 解析 tax_id：
   ├─ ① $row['tax_id'] 非空 → 直接使用
   ├─ ② $row['tax_id'] 空 + $row['tax_name'] 非空 → Tax::name() 查
   ├─ ③ $row['tax_id']+tax_name 空 + $row['tax_rate'] 非空 → getTaxIdFromRate()
   │   ├─ 按 type(固定'normal') + rate 查 taxes 表
   │   ├─ 找到 → 返回 id
   │   └─ 找不到 → CreateTax() 创建新 Tax → 返回新 id
   └─ ④ 都为空 → 返回 null（ItemTax 校验会失败）
5. model($row)       → hasRow([item_id, tax_id]) 查重 → 不存在则 new ItemTax
```

**关键对应字段总结**：

| 字段 | 解析优先级 | 兜底行为 | 前置门槛 |
|------|-----------|----------|----------|
| item_id | ① `item_id` → ② `item_name`(按 type=product 查) → ③ 创建新 Item | 找不到即创建 | `ItemTaxes::map()` 首先要求 `item_name` **非空**（[L31-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Sheets/ItemTaxes.php#L31-L33)），**仅填 `item_id` 但缺少 `item_name` 的行会被直接跳过**，不会进入 `getItemId()` 解析逻辑 |
| tax_id | ① `tax_id` → ② `tax_name` 查 → ③ `tax_rate`(按 type=normal 查) → ④ 创建新 Tax | 找不到即创建 | 需先通过 `item_name` 非空门槛，无其他前置限制 |

**仅填 `item_id` 但 `item_name` 为空的场景**：
- `ItemTaxes::map()` 中 `isEmpty($row, 'item_name')` 返回 `true`，直接 `return []`
- 该行被静默跳过，既不创建 ItemTax，也不触发 `getItemId()` 的兜底创建逻辑
- 若意图关联已有的 Item（已知其 item_id），必须同时提供 `item_name`

---

## 4. created_from / created_by / company_id 来源与一致性保证

### 4.1 字段来源分析

| 字段 | 创建时来源 | 导入时来源 | 保证一致性的机制 |
|------|-----------|-----------|----------------|
| **company_id** | `$this->request['company_id']`，由 `HasOwner` 接口在 Job 基类中注入 | `company_id()` 助手函数，来自当前登录用户的公司上下文 | 统一从全局上下文获取，创建和导入都使用当前公司 |
| **created_by** | `$this->request['created_by']`，Job 基类通过 `HasOwner` 接口注入 | 优先使用 Excel 中的 `created_by` (email 转 id)，否则使用当前用户 `$this->user->id` | [Import.php#L52](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L52) 的 `getCreatedById()` 统一处理 |
| **created_from** | `$this->request['created_from']`，Job 基类通过 `HasSource` 接口注入 | `$this->getSourcePrefix() . 'import'`，固定为 `core::import` | [Import.php#L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L54) 统一设置 |

### 4.2 一致性保证机制

1. **创建时一致性**：
   - `CreateItem` 和 `CreateItemTaxes` 都实现了 `HasOwner` 和 `HasSource` 接口
   - Job 基类自动注入 `created_by` 和 `created_from`
   - `CreateItemTaxes` 使用 `$this->item->company_id` 确保与 Item 一致（L46）

2. **导入时一致性**：
   - 两个 sheet 都继承自 `Import` 抽象类
   - `Import::map()` 统一设置 `company_id`、`created_by`、`created_from`
   - `ItemTaxes` 的 `company_id` 由 `parent::map()` 注入，与 `Items` sheet 使用相同的 `company_id()`
   - `getItemIdFromName()` 中创建新 Item 时也使用 `company_id()` 和相同的 `created_from`/`created_by` 来源
   - `getTaxIdFromRate()` 中创建新 Tax 时同理

3. **关键代码**：
   - [Import.php#L49-L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L49-L54) — 注入三字段
   - [CreateItemTaxes.php#L45-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItemTaxes.php#L45-L51) — 创建 ItemTax 时引用 item 的 company_id

---

## 5. 只导入 item 主表但缺少 tax sheet 的结果

### 5.1 技术原理

**代码位置**：[ImportMultipleSheets.php#L11-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/ImportMultipleSheets.php#L11-L31)

```php
abstract class ImportMultipleSheets implements ShouldQueue, WithChunkReading, WithMultipleSheets, SkipsUnknownSheets
{
    public function onUnknownSheet($sheetName)
    {
        // 空实现 — 静默跳过未知/缺失的 sheet
    }
}
```

当 Excel 文件缺少 `item_taxes` sheet 时，`SkipsUnknownSheets` 的 `onUnknownSheet()` 被调用但不做任何处理，导入流程继续，仅处理存在的 `items` sheet。

### 5.2 Items sheet 自身的 hasRow() 跳过行为

**代码位置**：[Import.php#L233-L270](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php#L233-L270)

```php
public function hasRow($row)
{
    if (empty($this->model) || empty($this->columns)) {
        return false;
    }

    // 一次性加载所有现有记录（仅 columns 指定的字段）到内存
    if (! $this->has_row || ! $this->has_row instanceof $this->model) {
        $this->has_row = $this->model::withoutEvents(function () {
            return $this->model::get($this->columns)->each(function ($data) {
                $data->setAppends([]);
                $data->unsetRelations();
            });
        });
    }

    // 构造当前行的搜索值
    $search_value = [];
    foreach ($this->columns as $key) {
        $search_value[$key] = isset($row[$key]) ? $row[$key] : null;
    }

    return in_array($search_value, $this->has_row->toArray());
}
```

Items sheet 的 `$columns = ['type', 'name', 'sale_price', 'purchase_price']`，因此：
- 当 Excel 中某行的 `(type, name, sale_price, purchase_price)` 与数据库中已有 Item **完全一致**时，`hasRow()` 返回 `true`，`model()` 返回 `null`，该行被**静默跳过**
- 不比较 `sku`、`description`、`category_id` 等其他字段 — 仅这四个字段一致即跳过
- 跳过的行不会触发任何提示或日志

### 5.3 完整结果分析

| 方面 | 结果 |
|------|------|
| **导入是否成功** | ✅ 成功，不会报错 — `SkipsUnknownSheets` 静默跳过缺失的 `item_taxes` sheet |
| **Items 主表** | ✅ 正常处理，但部分行可能因 `hasRow()` 匹配已有 (type, name, sale_price, purchase_price) 而被跳过 |
| **ItemTaxes 关联** | ❌ **完全缺失** — 没有任何新 ItemTax 记录产生（即使 Items 中有新创建的 Item） |
| **已存在的税关联** | ⚠️ 不受影响（导入不会删除旧的 ItemTax 记录） |
| **错误/警告** | ❌ 无任何错误或警告，用户无法感知税关联缺失 |
| **ItemTaxes 中自动创建 Item 的路径** | ❌ 不会触发 — 因 `item_taxes` sheet 不存在，`getItemIdFromName()` 和 `getTaxIdFromRate()` 的兜底创建逻辑完全不执行 |

### 5.4 数据状态示例

假设 Excel 只有 `items` sheet，5 行数据，其中 2 行 `(type, name, sale_price, purchase_price)` 与已有 Item 重复：

| 表 | 操作 | 结果数据 |
|----|------|----------|
| `items` | INSERT | 3 条新 Item（2 条因 hasRow() 匹配跳过） |
| `item_taxes` | — | 0 条新记录 |
| 关联关系 | — | 3 个新 Item 都没有税关联 |
| 已有 Item 的税关联 | — | 不受影响 |

**业务影响**：导入成功但新导入的商品在创建发票/账单时不会自动应用税率，需手动编辑补充。且因无任何提示，操作者可能误以为税关联已随 items sheet 一并导入。

---

## 代码参考汇总

| 模块 | 文件路径 |
|------|---------|
| CreateItem Job | [CreateItem.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItem.php) |
| CreateItemTaxes Job | [CreateItemTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/CreateItemTaxes.php) |
| Items Import (多 sheet 入口) | [Items.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Items.php) |
| Items Sheet 处理器 | [Sheets/Items.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Sheets/Items.php) |
| ItemTaxes Sheet 处理器 | [Sheets/ItemTaxes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Imports/Common/Sheets/ItemTaxes.php) |
| Import 抽象类 (含 hasRow) | [Import.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Import.php) |
| ImportMultipleSheets 抽象类 | [ImportMultipleSheets.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/ImportMultipleSheets.php) |
| Import Trait (getItemId, getTaxId 等) | [Import.php (Trait)](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Import.php) |
| Sources Trait | [Sources.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Sources.php) |
| Item 请求校验 | [Item.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/Item.php) |
| ItemTax 请求校验 | [ItemTax.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Common/ItemTax.php) |
| FormRequest 基类 (company_id 注入) | [FormRequest.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/FormRequest.php#L14-L19) |
| Job 基类 (setOwner / setSource) | [Job.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Job.php#L111-L132) |
| HasOwner 接口 | [HasOwner.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Interfaces/Job/HasOwner.php) |
| item_taxes 表迁移 | [2019_11_16_000000_core_v2.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L296-L307) |
| ItemTax 模型 | [ItemTax.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/ItemTax.php) |
