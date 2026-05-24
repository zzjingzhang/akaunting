# Akaunting Search String 过滤机制深度分析（v1.3.0 源码核实版）

本文档基于 `lorisleiva/laravel-search-string v1.3.0` 源码与项目源码双重核实，分析列表页 `search` 查询参数如何影响 `invoices` 和 `transactions` 的查询构建过程。

---

## 1. search 参数如何解析 field:value 和多个值

### 1.1 语法定义来源：Grammar.pp

语法文件定义了完整的词法和语法规则（[Grammar.pp#L1-L74](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Compiler/Grammar.pp#L1-L74)）：

```
# 词法
T_ASSIGNMENT    :|=         # 赋值操作符
T_COMPARATOR    >=?|<=?    # 比较操作符
T_COMMA         ,           # 列表分隔符
T_NOT           (not|NOT)  # 否定

# 语法规则
QueryNode:
    <T_TERM> Operator() NullableScalar()   # field:value 或 field>=value

ListNode:
    <T_TERM> ::T_IN:: ::T_LPARENTHESIS:: ScalarList() ::T_RPARENTHESIS:: |
    <T_TERM> ::T_ASSIGNMENT:: ScalarList()                 # 逗号分隔的多值列表

ScalarList:
    Scalar() ( ::T_COMMA:: Scalar() )*     # 逗号分隔，允许多个 Scalar
```

### 1.2 解析流水线

完整的 visitor 链定义在 [SearchString.php#L22-L31](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Concerns/SearchString.php#L22-L31)：

```
AttachRulesVisitor → IdentifyRelationshipsFromRulesVisitor → ValidateRulesVisitor
  → RemoveNotSymbolVisitor → BuildKeywordsVisitor → RemoveKeywordsVisitor
  → OptimizeAstVisitor → BuildColumnsVisitor
```

- **AttachRulesVisitor**：将 `config/search-string.php` 中的列规则绑定到 AST 节点
- **BuildColumnsVisitor**：将 AST 节点转换为 Eloquent Builder 的 WHERE 条件

### 1.3 field:value 解析细节

在 [BuildColumnsVisitor.php#L125-L131](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L125-L131) 中：

```php
protected function buildBasicQuery(QuerySymbol $query, ColumnRule $rule)
{
    $column = $rule->qualifyColumn($this->builder);
    return $this->builder->where($column, $query->operator, $query->value, $this->boolean);
}
```

- 单个 `field:value` → `WHERE column = 'value'`
- `field>=value` → `WHERE column >= 'value'`
- 值通过 `mapValue()` 做映射（[BuildColumnsVisitor.php#L149-L161](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L149-L161)）

### 1.4 逗号多值列表：依赖 `multiple: true` 吗？

**结论：不依赖。**

逗号列表是 Grammar 层的**语法特性**，与配置无关。`ListNode` 规则在 [Grammar.pp#L51-L54](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Compiler/Grammar.pp#L51-L54)：

```
ListNode:
    <T_TERM> ::T_IN:: ::T_LPARENTHESIS:: ScalarList() ::T_RPARENTHESIS:: |
    <T_TERM> ::T_ASSIGNMENT:: ScalarList()
```

在 [BuildColumnsVisitor.php#L115-L122](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L115-L122)：

```php
protected function buildList(ListSymbol $list)
{
    if (! $rule = $list->rule) {
        return;
    }
    $column = $rule->qualifyColumn($this->builder);
    $list->values = $this->mapValue($list->values, $rule);
    return $this->builder->whereIn($column, $list->values, $this->boolean, $list->negated);
}
```

**行为**：
- 当 Parser 遇到 `status:sent,partial`，因为 `sent,partial` 包含逗号，会匹配 `ListNode` 而非 `QueryNode`
- `buildList()` 直接调用 `whereIn()`，不检查 `$rule` 是否有 `multiple` 标记
- `multiple: true` 的唯一作用是：前端 [SearchString.php#L81](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/SearchString.php#L81) 渲染过滤器时决定是否允许多选

**可达性**：
- 对任何在 `search-string.php` 中注册的列——**URL 手写可达**（即使 `multiple` 为 false）
- 只有配置了 `multiple: true` 的列——**UI 可达**（通过多选下拉）

---

## 2. status:sent,partial 或 type:income 这类过滤如何进入 query

### 2.1 核心入口：scopeCollect()

[Model.php#L103-L127](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L103-L127)：

```php
public function scopeCollect($query, $sort = 'name')
{
    $request = request();
    $query->usingSearchString()->sortable($sort);    // ← 关键调用
    $limit = (int) $request->get('limit', setting('default.list_limit', '25'));
    return $query->paginate($limit);
}
```

### 2.2 usingSearchString() 处理流程

[Model.php#L129-L140](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L129-L140)：

```php
public function scopeUsingSearchString(Builder $query, string|null $string = null)
{
    event(new SearchStringApplying($query));
    $string = $string ?: request('search', '');

    // 步骤1：移除未知字段 token
    $string = $this->stripUnknownSearchStringTokens($string, get_class($this));

    // 步骤2：调用第三方包解析并构建查询
    $this->getSearchStringManager()->updateBuilder($query, $string);

    event(new SearchStringApplied($query));
}
```

### 2.3 是否调用 Model Scope 方法？

**结论：不直接调用。**

`BuildColumnsVisitor` 使用 Eloquent Builder 的原生方法：

| AST 节点 | Builder 方法 | 是否调用 scope |
|----------|-------------|---------------|
| `QuerySymbol` (status:sent) | `->where('status', '=', 'sent')` | ❌ 不调用 scopeStatus() |
| `ListSymbol` (status:sent,partial) | `->whereIn('status', ['sent','partial'])` | ❌ 不调用 scopeStatus() |
| `SoloSymbol` (纯文本搜索) | `->where('...', 'like', '%...%')` | ❌ 不调用 |
| `RelationshipSymbol` | `->whereHas('relation', ...)` | ❌ 不调用 |

**关键源码**：
- [BuildColumnsVisitor.php#L125-L131](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L125-L131) — `buildBasicQuery` 使用 `$this->builder->where()`
- [BuildColumnsVisitor.php#L115-L122](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L115-L122) — `buildList` 使用 `$this->builder->whereIn()`

**重要推论**：`Document::scopeStatus()` 和 `Transaction::scopeType()` 这些 scope 方法永远不会被 `usingSearchString()` 调用。它们只在手动调用时生效（如 `Document::status('paid')`）。

### 2.4 type:income 进入 Transaction 查询的完整路径

1. **URL 层**：`GET /transactions?search=type:income` — **URL 手写可达**
2. **控制器层**：[Transactions.php#L42-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L42-L50) 调用 `Transaction::with(...)->collect(['paid_at'=> 'desc'])` — **后端主查询可达**
3. **Global Scope 层**：[Scopes.php#L11-L20](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php#L11-L20) 的 `applyNotRecurringScope` 会检查是否已有 `type` 条件——**如果 search 已含 `type:`，则跳过 `isNotRecurring()`**
4. **Builder 层**：`BuildColumnsVisitor::buildBasicQuery` 追加 `WHERE type = 'income'`

### 2.5 status:sent,partial 进入 Invoice 查询的完整路径

1. **URL 层**：`GET /invoices?search=status:sent,partial` — **URL 手写可达**
2. **控制器层**：[Invoices.php#L34-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L34-L51) 调用 `Document::invoice()->with(...)->collect(['document_number'=> 'desc'])`
3. **Document::invoice()** 先追加 `WHERE type = 'invoice'`
4. **Builder 层**：`BuildColumnsVisitor::buildList` 追加 `WHERE status IN ('sent','partial')`

---

## 3. Controller、Model Scope、Builder 分别承担什么

### 3.1 职责分层

| 层级 | 职责 | 关键文件 | 触发行号 |
|------|------|----------|----------|
| **Controller** | 1. 设置 Active Tab（注入默认 search）<br>2. 组织初始 scope（如 `Document::invoice()`）<br>3. 预加载关联 (`with()`)<br>4. 调用 `collect()` 触发流程<br>5. 计算汇总数据（Transactions 特有） | [Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L30-L55)<br>[Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L38-L94) | L34-L51<br>L42-L50 |
| **Controller (Tab 逻辑)** | 从 setting 读取 pin tab，若无 search 则注入默认 search（如 `status:sent,viewed,partial`），仅作用于 request 对象 | [Controller.php#L104-L154](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php#L104-L154) | L140-L143 |
| **Model (Scope)** | `scopeCollect()` 编排 usingSearchString + sortable + paginate<br>`scopeUsingSearchString()` 集成第三方包<br>`scopeInvoice()`/`scopeType()` 等业务 scope（手动调用时生效） | [Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L103-L140)<br>[Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L201-L246)<br>[Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L184-L196) | L103-L140<br>L231-L234<br>L189-L196 |
| **Global Scope** | 自动排除 recurring/split 记录；可被显式 type 条件绕过 | [Document.php (Scope)](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Document.php#L21-L24)<br>[Transaction.php (Scope)](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php#L21-L26)<br>[Scopes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php#L11-L31) | L21-L24<br>L21-L26 |
| **Builder (lorisleiva)** | 解析 search 字符串为 AST<br>AttachRulesVisitor 绑定配置规则<br>BuildColumnsVisitor 构建 WHERE 条件 | Grammar.pp<br>BuildColumnsVisitor.php<br>AttachRulesVisitor.php | 见 1.1/2.3 |
| **Builder (Akaunting 自定义)** | `Category` 自定义 Builder：`get()` 会递归展开子分类树；`paginate()` 用 `getWithoutChildren()` 避免分页膨胀 | [Builder/Category.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Builders/Category.php#L16-L73) | L16-L35<br>L59-L73 |
| **SearchString Trait** | `stripUnknownSearchStringTokens()` 白名单过滤<br>`getSearchStringValue()`/`getSearchStringOperator()` 手动提取（供 Controller/View 使用） | [SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L23-L195) | L23-L64<br>L134-L195 |
| **配置层** | 列白名单、列类型（date/boolean/relationship/searchable）、值映射（values）、路由（route）、多选（multiple） | [search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) | 全文 |

### 3.2 Transactions 控制器的 Tab 状态逻辑 vs Query 过滤

[Controller.php#L156-L206](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php#L156-L206) 中 `setActiveTabForTransactions()` 的行为：

```php
// 当无 search 也无 list_records 时：
if (! request()->has('list_records') && ! request()->has('search')) {
    // 读取用户 pin 的 tab，注入对应 search
    request()->merge($data);  // 如 type:income
}

// 当仍然没有 type 过滤时：
if (empty($type)) {
    // 注入默认 income search
    request()->offsetSet('search', $search);   // 如 'type:income'
    request()->offsetSet('programmatic', '1'); // 标记为系统注入
}
```

**关键区分**：
- **Tab 逻辑**：修改 `request()->search`，作用于后续 `usingSearchString()` 的输入
- **Query 过滤**：`usingSearchString()` 读取 `request('search')`，构建 WHERE 条件
- **`programmatic=1`**：标记此 search 是系统注入的，前端 UI 用它来决定显示"所有记录"tab 还是某个特定 tab

### 3.3 Document 后端查询配置 vs App\Models\Sale\Invoice 搜索组件配置

配置文件 [search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) 中有两个相关条目：

| 配置键 | 使用者 | 差异 |
|--------|--------|------|
| `App\Models\Document\Document::class` (L331-L372) | `Invoices` 控制器 `Document::invoice()->collect()` | `status` 仅有 `'multiple' => true`，**无** `values` 映射 |
| `'App\Models\Sale\Invoice'` (L429-L483) | 前端 `x-index.search` 组件 [search.blade.php#L5](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/components/index/search.blade.php#L5) | `status` 有 `values` 映射（如 `sent,viewed,partial` → unpaid） |

**来源追溯**：
- 前端 View 通过 `x-index.search search-string="..."` 传入模型类名
- [SearchString.php#L46-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/SearchString.php#L46-L51) 读取对应配置渲染过滤器 UI
- `'App\Models\Sale\Invoice'` 是**纯配置键**，对应类不存在，但 `config('search-string')` 可以直接按键取值

**实际效果**：
- 后端查询时，`Document::collect()` 使用 `App\Models\Document\Document::class` 配置——`status:sent,partial` 直接传原值到 `whereIn`
- 前端过滤器使用 `'App\Models\Sale\Invoice'` 配置——显示"未付款"选项但发送 `status:sent,viewed,partial` 搜索串

---

## 4. 空 search、未知字段、关系字段会怎样

### 4.1 空 search 参数

**行为**：
1. `request('search', '')` 返回 `''`
2. `stripUnknownSearchStringTokens('', ...)` 返回 `''`（[SearchString.php#L138-L142](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L138-L142)）
3. `updateBuilder($query, '')` 解析为空 AST，不添加 WHERE 条件（[SearchStringManager.php](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/SearchStringManager.php) 的 `build()` 方法）
4. Controller 的 Tab 逻辑会注入默认 search（如 `status:sent,viewed,partial`）

**可达性**：
- **UI 可达**：点击"所有记录" tab 可设置 `list_records=all`，跳过默认注入
- **URL 手写可达**：`?search=` 或完全省略 search 参数

### 4.2 未知字段处理

`stripUnknownSearchStringTokens()` ([SearchString.php#L134-L195](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L134-L195))：

```php
// 白名单来源：
$valid_columns = array_unique(array_merge(
    array_keys($model_config_columns),          // 模型配置的列
    array_values(config('search-string.default.keywords', [])),  // sort/fields/limit/from
    array_keys(config('search-string.default.columns', [])),     // created_at/updated_at
));

// 正则移除未知 token：
$input = preg_replace_callback(
    pattern: '/\b(?:not\s+)?(\w+)\s*(?:>=|<=|>|<|=|:)\s*(?:"[^"]*"|\S+)/',
    callback: fn($m) => in_array($m[1], $valid_columns, true) ? $m[0] : '',
    subject: $input,
);
```

**行为**：
- 不在白名单中的 `field:value` 被静默移除
- 如果移除后 search 为空，则无过滤条件

**可达性**：
- **URL 手写可达**：`?search=unknown_field:value` 被移除，但不会报错
- 默认配置 `fail => 'all-results'` 确保不抛异常

### 4.3 关系字段处理

配置标记 `'relationship' => true` 的列：

```php
// search-string.php Transaction 配置
'recurring' => [
    'key' => 'recurring',
    'foreign_key' => '',
    'relationship' => true,   // ← 标记为关系
    'boolean' => true,
],
```

**处理流程**：

1. **AttachRulesVisitor** 将 `ColumnRule` 绑定到 AST 节点（[AttachRulesVisitor.php#L38-L49](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/AttachRulesVisitor.php#L38-L49)）

2. **IdentifyRelationshipsFromRulesVisitor** 将 `SoloSymbol` 转为 `RelationshipSymbol`（[IdentifyRelationshipsFromRulesVisitor.php#L16-L24](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/IdentifyRelationshipsFromRulesVisitor.php#L16-L24)）：

```php
public function visitSolo(SoloSymbol $solo)
{
    if (! $this->isRelationship($solo->rule)) {
        return $solo;
    }
    return (new RelationshipSymbol($solo->content, new EmptySymbol()))
        ->attachRule($solo->rule);
}
```

3. **BuildColumnsVisitor** 调用 `has()` 方法（[BuildColumnsVisitor.php#L77-L82](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L77-L82)）：

```php
protected function buildRelationship(RelationshipSymbol $relationship)
{
    // ...
    return $this->builder->has($rule->column, $operator, $count, $this->boolean, $callback);
}
```

**行为**：
- `recurring:1` → `WHERE EXISTS (SELECT * FROM recurring WHERE ...)`
- 关系模型必须也使用 `SearchString` trait（[ColumnRule.php#L62-L68](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Options/ColumnRule.php#L62-L68) 中的 `LogicException`）

**可达性**：
- **URL 手写可达**：`?search=recurring:1`
- **UI 可达**：配置了 `relationship + boolean` 的列会渲染为布尔开关

### 4.4 recurring 相关的 Global Scope 交互

[Transaction.php#L101-L104](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L101-L104) 注册了 Global Scope：

```php
protected static function booted()
{
    static::addGlobalScope(new Scope);  // App\Scopes\Transaction
}
```

[Scopes/Transaction.php#L21-L26](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php#L21-L26)：

```php
public function apply(Builder $builder, Model $model)
{
    $this->applyNotRecurringScope($builder, $model);
    $this->applyNotSplitScope($builder, $model);
}
```

[Scopes.php#L11-L20](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php#L11-L20)：

```php
public function applyNotRecurringScope(Builder $builder, Model $model): void
{
    // 如果已有 type 条件，跳过
    if ($this->scopeColumnExists($builder, $model->getTable(), 'type')) {
        return;
    }
    $builder->isNotRecurring();  // WHERE type NOT LIKE '%-recurring'
}
```

**边界情况**：
- 当 search 包含 `type:income` → `scopeColumnExists` 检测到 `type` 列 → 跳过 Global Scope → 可能返回 recurring 记录
- 当 search 包含 `recurring:0` → `buildRelationship` 追加 `WHERE NOT EXISTS (SELECT * FROM recurring ...)`
- 当 search 同时含 `type:income` 和 `recurring:0` → 显式关系条件限制，Global Scope 已跳过

### 4.5 Category 自定义 Builder 的边界

[Category.php#L64-L67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L64-L67)：

```php
public function newEloquentBuilder($query)
{
    return new Builder($query);  // 自定义 App\Builders\Category
}
```

[Builders/Category.php#L16-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Builders/Category.php#L16-L35)：

```php
public function get($columns = ['*'])
{
    $collection = parent::get($columns);

    return $collection->withChildren('sub_categories', function ($list, $parent, ...) {
        // 递归展开子分类树，push 到同一个 collection
        $list->push($parent);
        foreach ($parent->$relation as $item) {
            $addChildren($list, $item, $relation, $level + 1, $addChildren);
        }
    });
}
```

**边界**：
- `get()` 返回展开的分类树（父分类 + 所有子分类），可能远多于 SQL 返回的行数
- `paginate()` 调用 `getWithoutChildren()` 避免分页膨胀（[Builders/Category.php#L59-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Builders/Category.php#L59-L73)）
- `scopeCollect()` 使用 `paginate()` 而非 `get()`，所以列表页不会返回递归展开的结果
- `scopeCollectForExport()` 调用 `$query->withSubCategory()` → `get()` → 返回展开的树结构（[Category.php#L268-L291](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L268-L291)）

---

## 5. 容易把逗号值当普通字符串处理的反例

### 5.1 反例：搜索 status:sent,partial 后调用 scopeStatus()

假设开发者写出以下代码（在 Controller 或 Job 中）：

```php
$status = search_string_value('status');
// 当 search=status:sent,partial 时
// $status = 'sent,partial'（字符串，非数组）

$invoices = Document::invoice()
    ->status($status)   // ← 调用 scopeStatus(string $status)
    ->get();
```

[Document.php#L201-L204](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L201-L204)：

```php
public function scopeStatus(Builder $query, string $status): Builder
{
    return $query->where($this->qualifyColumn('status'), '=', $status);
    //                                                      ↑  精确匹配 'sent,partial'
}
```

**生成的 SQL**：
```sql
WHERE status = 'sent,partial'  -- 永远为空结果
```

**而 `usingSearchString()` 会生成**：
```sql
WHERE status IN ('sent', 'partial')  -- 正确
```

### 5.2 为什么会答错

1. `getSearchStringValue()` 返回 `string` 类型（[SearchString.php#L27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L27)），不会自动拆分数组
2. `Document::scopeStatus()` 参数类型声明为 `string`，强化了"这是单值"的错觉
3. 配置中 `status` 的 `values` 映射键本身就是逗号字符串（如 `'sent,viewed,partial'`），容易让人以为 scope 内部会处理

### 5.3 对比：usingSearchString() 不会调用 scope

[BuildColumnsVisitor.php#L115-L122](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php#L115-L122)：

```php
protected function buildList(ListSymbol $list)
{
    if (! $rule = $list->rule) {
        return;
    }
    $column = $rule->qualifyColumn($this->builder);
    $list->values = $this->mapValue($list->values, $rule);
    return $this->builder->whereIn($column, $list->values, $this->boolean, $list->negated);
    //                       ↑ 直接使用 Eloquent Builder 的 whereIn，不经过 scope
}
```

### 5.4 修复方案

如果必须手动调用 scope：

```php
public function scopeStatus(Builder $query, $status): Builder
{
    if (empty($status)) {
        return $query;
    }

    $statuses = is_array($status) ? $status : explode(',', $status);

    return $query->whereIn($this->qualifyColumn('status'), $statuses);
}
```

---

## 附录 A：完整调用链

```
用户 URL: /invoices?search=status:sent,partial
  ↓
[Router] → Invoices::index()
  ↓
[Controller L32] setActiveTabForDocuments()
  ├─ 有 search → 跳过默认注入
  └─ status 值为 sent,partial → 不等于 unpaid/draft → list_records=all
  ↓
[Controller L34] Document::invoice()
  └─ scopeInvoice() → WHERE type = 'invoice'  [Document.php L231-L234]
  ↓
[Controller L34] ->with([...])->collect(['document_number'=> 'desc'])
  ↓
[Model L103] scopeCollect()
  ↓
[Model L115] usingSearchString()
  ├─ stripUnknownSearchStringTokens("status:sent,partial", "App\Models\Document\Document")
  │   └─ status 在白名单 → 保留
  └─ SearchStringManager::updateBuilder()
      ├─ Parser: ListNode("status", ["sent","partial"])
      ├─ AttachRulesVisitor: 绑定 ColumnRule(status)
      ├─ IdentifyRelationshipsFromRulesVisitor: 非关系 → 不变
      └─ BuildColumnsVisitor::buildList()
          └─ $builder->whereIn("status", ["sent","partial"])
  ↓
[Global Scope] App\Scopes\Document
  └─ applyNotRecurringScope()
      ├─ scopeColumnExists 检测到 type 列 → 跳过
      └─ 不追加 NOT LIKE '%-recurring'
  ↓
[Model L124] paginate($limit)
  ↓
[Controller L54] return view('sales.invoices.index', compact('invoices', 'total_invoices'))
```

---

## 附录 B：可达性速查表

| 行为 | UI 可达 | URL 手写可达 | 后端主查询可达 |
|------|---------|-------------|---------------|
| `status:sent,partial` 多值 | ✅ `multiple: true` 时渲染多选 | ✅ Grammar 层直接支持 | ✅ buildList → whereIn |
| `type:income` 单值 | ✅ Tab 点击 | ✅ | ✅ buildBasicQuery → where |
| `unknown_field:value` | ❌ 不出现在过滤器 UI | ✅ 可写入 URL | ⚠️ 被 strip 移除，无错误 |
| `recurring:1` 关系 | ✅ 布尔开关 | ✅ | ✅ buildRelationship → has() |
| `type:income,expense` 多值 | ⚠️ 配置无 `multiple: true` | ✅ | ✅ buildList → whereIn |
| 空 search | ✅ "所有" tab | ✅ | ⚠️ Tab 逻辑注入默认值 |
| Tab 注入默认 search | ✅ | ❌ URL 不触发 | ✅ 修改 request 对象 |

---

## 关键文件速查

| 模块 | 文件路径 | 关键行号 |
|------|----------|----------|
| SearchString Trait (Akaunting) | [app/Traits/SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php) | L23-L64 (getValue), L87-L124 (getOperator), L134-L195 (stripUnknown) |
| Abstract Model | [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) | L103-L127 (scopeCollect), L129-L140 (scopeUsingSearchString) |
| Document Model | [app/Models/Document/Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php) | L201-L204 (scopeStatus), L231-L234 (scopeInvoice) |
| Transaction Model | [app/Models/Banking/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php) | L189-L196 (scopeType) |
| Category Model | [app/Models/Setting/Category.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php) | L64-L67 (newEloquentBuilder) |
| Category Builder | [app/Builders/Category.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Builders/Category.php) | L16-L35 (get 递归展开), L59-L73 (paginate) |
| Invoices Controller | [app/Http/Controllers/Sales/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php) | L30-L55 |
| Transactions Controller | [app/Http/Controllers/Banking/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php) | L38-L94 |
| Controller (Tab 逻辑) | [app/Abstracts/Http/Controller.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/Controller.php) | L104-L154 (Documents), L156-L206 (Transactions) |
| Global Scopes | [app/Scopes/Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Document.php), [app/Scopes/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php) | L21-L26 |
| Scopes Trait | [app/Traits/Scopes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php) | L11-L31 |
| 搜索组件 | [app/View/Components/SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/View/Components/SearchString.php) | L46-L84 |
| 配置 | [config/search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) | L158-L206 (Transaction), L331-L372 (Document), L429-L483 (Sale Invoice) |
| Grammar.pp (v1.3.0) | [GitHub](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Compiler/Grammar.pp) | L51-L54 (ListNode), L61-L63 (QueryNode) |
| BuildColumnsVisitor (v1.3.0) | [GitHub](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Visitors/BuildColumnsVisitor.php) | L77-L82 (buildRelationship), L115-L122 (buildList), L125-L131 (buildBasicQuery) |
| SearchStringOptions (v1.3.0) | [GitHub](https://github.com/lorisleiva/laravel-search-string/blob/3842079d0fe6315a3a24dbc30c15ed590c635357/src/Options/SearchStringOptions.php) | L38-L47 (generateOptions), L74-L86 (parseColumns) |
