# Akaunting Search String 过滤机制深度分析

本文档分析列表页 `search` 查询参数如何影响 `invoices` 和 `transactions` 的查询构建过程。

---

## 1. search 参数如何解析 field:value 和多个值

### 1.1 解析核心：`SearchString` Trait

系统使用两层 SearchString 机制：

**自定义 Trait**：[SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php)

**第三方包**：`Lorisleiva\LaravelSearchString\Concerns\SearchString`（在 [Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L19) 中引入）

### 1.2 field:value 解析流程

`getSearchStringValue()` 方法 ([SearchString.php#L23-L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L23-L64)):

```php
// 步骤1：归一化空格形式 "age > 30" → "age>30"
$input = preg_replace('/\b(' . preg_quote($name, '/') . ')\s*(>=|<=|>|<|=|:)\s*/', '$1$2', $input);

// 步骤2：按空格分割成多个 token
$columns = explode(' ', $input);

// 步骤3：按操作符分割每个 token
$variable = preg_split('/(>=|<=|>|<|=|:)/', $column, 2);

// 步骤4：去除引号
$extracted = trim($variable[1], '"\'');
```

支持的操作符：`>=`, `<=`, `>`, `<`, `=`, `:`

### 1.3 多值处理

在配置文件 [search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) 中标记 `'multiple' => true` 的字段支持逗号分隔的多个值：

```php
// Transaction 配置 (第158-206行)
'type' => [
    'values' => [
        'income' => 'general.incomes',
        'expense' => 'general.expenses',
    ],
],
'account_id' => [
    'route' => ['accounts.index', 'search=enabled:1'],
    'multiple' => true,  // 支持多值
],

// Invoice 配置 (第429-483行)
'status' => [
    'values' => [
        'sent,viewed,partial' => 'documents.statuses.unpaid',  // 组合值作为 key
        'paid' => 'documents.statuses.paid',
        'partial' => 'documents.statuses.partial',
    ],
    'multiple' => true,  // 支持多值
],
```

**注意**：`status:sent,partial` 这种形式中，逗号分隔的值会被第三方包 `lorisleiva/laravel-search-string` 解析为 `WHERE IN ('sent', 'partial')`。

---

## 2. status:sent,partial 或 type:income 这类过滤如何进入 query

### 2.1 核心入口：`scopeCollect()`

在 [Model.php#L103-L127](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L103-L127) 中定义：

```php
public function scopeCollect($query, $sort = 'name')
{
    $request = request();
    
    // 关键：调用 usingSearchString() 处理搜索字符串
    $query->usingSearchString()->sortable($sort);
    
    $limit = (int) $request->get('limit', setting('default.list_limit', '25'));
    return $query->paginate($limit);
}
```

### 2.2 `usingSearchString()` 处理流程

[Model.php#L129-L140](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L129-L140):

```php
public function scopeUsingSearchString(Builder $query, string|null $string = null)
{
    event(new SearchStringApplying($query));

    $string = $string ?: request('search', '');

    // 步骤1：移除未知字段的 token
    $string = $this->stripUnknownSearchStringTokens($string, get_class($this));

    // 步骤2：调用第三方包解析并构建查询
    $this->getSearchStringManager()->updateBuilder($query, $string);

    event(new SearchStringApplied($query));
}
```

### 2.3 控制器调用示例

**Invoices 控制器** ([Invoices.php#L34-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L34-L51)):

```php
$invoices = Document::invoice()->with([...])->collect(['document_number'=> 'desc']);
```

**Transactions 控制器** ([Transactions.php#L42-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php#L42-L50)):

```php
$transactions = Transaction::with([...])->collect(['paid_at'=> 'desc']);
```

### 2.4 额外的 type 过滤（Transactions 特有）

Transactions 控制器在第85-86行还有一层特殊处理：

```php
$search_type = search_string_value('type');
$type = empty($search_type) ? 'transactions' : (($search_type == 'income') ? 'income' : 'expense');
```

`search_string_value()` 是 [helpers.php#L363-L368](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php#L363-L368) 中定义的辅助函数，直接调用 `SearchString` trait。

---

## 3. Controller、Model Scope、Builder 分别承担什么

### 3.1 职责分层

| 层级 | 职责 | 关键文件 |
|------|------|----------|
| **Controller** | 1. 组织查询的初始条件（如 `Document::invoice()`）<br>2. 预加载关联关系 (`with()`)<br>3. 调用 `collect()` 触发完整流程<br>4. 特殊业务逻辑（如 Transactions 中的 type 二次处理） | [Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php)<br>[Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php) |
| **Model Scope** | 1. 定义业务相关的查询片段（如 `scopeInvoice()`, `scopePaid()`）<br>2. `scopeCollect()` 作为核心编排方法<br>3. `scopeUsingSearchString()` 集成第三方搜索包<br>4. 定义类型过滤 scope（如 `scopeType()`, `scopeStatus()`） | [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php)<br>[Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php)<br>[Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) |
| **Builder / SearchString** | 1. `SearchString` trait 解析 field:value 语法<br>2. 第三方包 `lorisleiva/laravel-search-string` 实际构建 WHERE 条件<br>3. `stripUnknownSearchStringTokens()` 安全过滤未知字段<br>4. 配置驱动的字段白名单机制 | [SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php)<br>[search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) |
| **Global Scope** | 1. 自动排除 recurring 和 split 类型的记录<br>2. 可被显式条件绕过（通过 `scopeColumnExists` 检测） | [Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Document.php)<br>[Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Transaction.php)<br>[Scopes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php) |

### 3.2 关键 Model Scope 示例

**Transaction 的 scopeType()** ([Transaction.php#L189-L196](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L189-L196)):

```php
public function scopeType(Builder $query, $types): Builder
{
    if (empty($types)) {
        return $query;
    }
    // 支持数组，使用 whereIn
    return $query->whereIn($this->qualifyColumn('type'), (array) $types);
}
```

**Document 的 scopeStatus()** ([Document.php#L201-L204](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L201-L204)):

```php
public function scopeStatus(Builder $query, string $status): Builder
{
    return $query->where($this->qualifyColumn('status'), '=', $status);
}
```

**注意**：`Document::scopeStatus()` 只接受字符串，不直接支持数组！这是第5个问题的关键。

---

## 4. 空 search、未知字段、关系字段会怎样

### 4.1 空 search 参数

- `request('search', '')` 返回空字符串
- `stripUnknownSearchStringTokens()` 直接返回空
- `getSearchStringManager()->updateBuilder($query, '')` 不添加任何 WHERE 条件
- **结果**：返回所有符合 Global Scope 的记录

### 4.2 未知字段处理

`stripUnknownSearchStringTokens()` 方法 ([SearchString.php#L134-L195](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php#L134-L195)):

```php
public function stripUnknownSearchStringTokens(string $input, string $model_class): string
{
    // 读取该模型的配置
    $model_config_columns = config('search-string.' . $model_class . '.columns', null);
    
    // 合并合法字段：配置中的字段 + 默认关键词 + 默认字段
    $valid_columns = array_unique(array_merge(
        array_keys($model_config_columns),
        array_values(config('search-string.default.keywords', [])),
        array_keys(config('search-string.default.columns', [])),
    ));
    
    // 通过正则替换移除未知字段的 token
    $input = preg_replace_callback(
        pattern: '/\b(?:not\s+)?(\w+)\s*(?:>=|<=|>|<|=|:)\s*(?:"[^"]*"|\S+)/',
        callback: function (array $matches) use ($valid_columns, $key_patterns): string {
            $token = $matches[1];
            // 不在白名单中的 token 被替换为空字符串
            return in_array($token, $valid_columns, strict: true) ? $matches[0] : '';
        },
        subject: $input,
    );
    
    return trim(preg_replace('/\s+/', ' ', $input));
}
```

**结果**：未知字段的 token 被静默移除，不会报错（可通过 `config/search-string.php` 第18行的 `fail` 配置改变行为）。

### 4.3 关系字段处理

配置中标记 `'relationship' => true` 的字段会被特殊处理：

```php
// Transaction 配置
'recurring' => [
    'key' => 'recurring',
    'foreign_key' => '',
    'relationship' => true,  // 标记为关系字段
    'boolean' => true,
],
```

第三方包 `lorisleiva/laravel-search-string` 会自动：
1. 识别关系字段
2. 调用 `whereHas()` 进行关联查询
3. 对 `boolean` 类型的关系字段，`recurring:1` 会转换为 `WHERE EXISTS (SELECT * FROM recurring WHERE ...)`

---

## 5. 容易把逗号值当普通字符串处理的反例

### 5.1 问题场景

假设我们要查询状态为 "sent" 或 "partial" 的发票：

```
GET /invoices?search=status:sent,partial
```

### 5.2 Document Model 的问题

[Document.php#L201-L204](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L201-L204):

```php
public function scopeStatus(Builder $query, string $status): Builder
{
    return $query->where($this->qualifyColumn('status'), '=', $status);
}
```

**问题**：
1. 参数类型声明为 `string $status`，强制转换为字符串
2. 使用 `=` 而不是 `whereIn`
3. 当传入 `"sent,partial"` 时，会执行 `WHERE status = 'sent,partial'`，而不是 `WHERE status IN ('sent', 'partial')`

### 5.3 对比 Transaction 的正确实现

[Transaction.php#L189-L196](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L189-L196):

```php
public function scopeType(Builder $query, $types): Builder
{
    if (empty($types)) {
        return $query;
    }
    // ✅ 正确：不限制类型，使用 (array) 转换，使用 whereIn
    return $query->whereIn($this->qualifyColumn('type'), (array) $types);
}
```

### 5.4 为什么会出现这个问题

1. **类型声明过于严格**：`string $status` 导致数组被转换为字符串 `"Array"` 或逗号连接的字符串
2. **没有使用 `whereIn`**：对于 `multiple: true` 的字段，应该支持多值查询
3. **配置与实现不同步**：配置中 `status` 标记了 `'multiple' => true`，但代码实现没有支持

### 5.5 修复方案

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

## 附录：完整调用链

```
Controller::index()
  ↓
Model::invoice() / Model::with(...)
  ↓
Model::collect($sort)  [Abstract Model]
  ↓
Model::usingSearchString()  [Abstract Model]
  ↓
  ├─ event(SearchStringApplying)
  ├─ stripUnknownSearchStringTokens()  [自定义 SearchString Trait]
  └─ getSearchStringManager()->updateBuilder()  [第三方 lorisleiva/laravel-search-string]
      ├─ 解析 search 字符串为 AST
      ├─ 根据配置识别字段类型
      ├─ 对 multiple 字段拆分逗号值
      └─ 构建 WHERE 条件（自动调用 scope 方法）
  ↓
Model::sortable($sort)
  ↓
Model::paginate($limit)
  ↓
Controller 返回视图
```

---

## 关键文件速查

| 模块 | 文件路径 |
|------|----------|
| SearchString Trait | [app/Traits/SearchString.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/SearchString.php) |
| Abstract Model | [app/Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php) |
| Document Model | [app/Models/Document/Document.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php) |
| Transaction Model | [app/Models/Banking/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php) |
| Invoices Controller | [app/Http/Controllers/Sales/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php) |
| Transactions Controller | [app/Http/Controllers/Banking/Transactions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Banking/Transactions.php) |
| 搜索配置 | [config/search-string.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/search-string.php) |
| 辅助函数 | [app/Utilities/helpers.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/helpers.php) |
| Global Scopes Trait | [app/Traits/Scopes.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Scopes.php) |
