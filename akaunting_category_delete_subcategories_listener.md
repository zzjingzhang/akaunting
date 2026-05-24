# Akaunting 删除分类时子分类处理机制分析

## 1. DeleteCategory 本体做了什么，事件在哪里触发

### handle() 完整执行流程

[DeleteCategory.php:L17-L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L17-L32) 的 `handle()` 方法按以下顺序执行：

```php
public function handle(): bool
{
    $this->authorize();                              // 步骤1：授权与引用检查

    event(new CategoryDeleting($this->model));        // 步骤2：触发删除前事件

    \DB::transaction(function () {
        $this->deleteSubCategories($this->model);     // 步骤3：事务内递归软删除
        $this->model->delete();                       // 步骤4：主分类重复删除
    });

    event(new CategoryDeleted($this->model));         // 步骤5：触发删除后事件

    return true;
}
```

### deleteSubCategories() 实际行为

[DeleteCategory.php:L34-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L34-L41)：

```php
public function deleteSubCategories($model)
{
    $model->delete();                                 // 先软删除当前节点
    foreach ($model->sub_categories as $sub_category) {
        $this->deleteSubCategories($sub_category);    // 递归处理直接子分类
    }
}
```

**执行顺序**：调用 `deleteSubCategories($this->model)` 时，传入的是主分类，因此：
1. 主分类首先被 `$model->delete()` 软删除
2. 然后遍历主分类的 `sub_categories`，对每个子分类递归调用 `deleteSubCategories()`
3. 每个子分类同样先被软删除，再遍历其子分类

**注意**：`$this->model->delete()` 在第 26 行再次调用时，主分类已被 `deleteSubCategories()` 软删除，这是一次冗余调用（软删除下只是更新 `deleted_at` 时间戳）。

### 事件触发位置与注册

- **CategoryDeleting**（删除前）：[DeleteCategory.php:L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L21-L21)
- **CategoryDeleted**（删除后）：[DeleteCategory.php:L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L29-L29)

事件-监听器映射在 [Event.php:L121-L123](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L121-L123) 注册：

```php
\App\Events\Setting\CategoryDeleted::class => [
    \App\Listeners\Setting\DeleteCategoryDeletedSubCategories::class,
],
```

---

## 2. CategoryDeleted Listener 如何找到并删除子分类

### Listener 源码

[DeleteCategoryDeletedSubCategories.php:L19-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Setting/DeleteCategoryDeletedSubCategories.php#L19-L30)：

```php
public function handle(Event $event)
{
    $category = $event->category;

    if (empty($category->sub_categories)) {
        return;
    }

    foreach ($category->sub_categories as $sub_category) {
        $this->dispatch(new DeleteCategory($sub_category));
    }
}
```

### sub_categories 关系定义

该关系定义在 [Category.php:L103-L106](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L103-L106)：

```php
public function sub_categories()
{
    return $this->hasMany(Category::class, 'parent_id')
        ->withSubCategory()
        ->with('categories')
        ->orderBy('name');
}
```

### withSubCategory() 与 SoftDeletes 的关系

[Category.php:L242-L245](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L242-L245)：

```php
public function scopeWithSubCategory($query)
{
    return $query->withoutGlobalScope(new Scope);
}
```

`Scope` 指的是 [app/Scopes/Category.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Scopes/Category.php#L18-L25)，它的 `apply()` 方法仅添加 `$builder->whereNull('parent_id')` 过滤。

[app/Abstracts/Model.php:L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L23-L23) 中 `SoftDeletes` 是由 Eloquent 基类 `SoftDeletingScope` 全局 Scope 提供的。

**关键澄清**：`withSubCategory()` 只移除 `App\Scopes\Category` 这一个全局 Scope（即 `parent_id IS NULL` 过滤），**不影响 SoftDeletes 的 `deleted_at IS NULL` 过滤**。因此即使在 `sub_categories()` 关系中使用了 `withSubCategory()`，软删除的子分类在通过关系查询时仍会被 SoftDeletes Scope 过滤掉。

### Listener 读取子分类的真实机制

Listener 能否读到子分类，**不依赖查询 scope 或 SoftDeletes 过滤，而是依赖此前执行过程中是否已将关系加载到模型实例的内存中**。

执行时序中 `sub_categories` 的加载来源：

1. **authorize() 第 70 行**：`foreach ($this->model->sub_categories as $sub_category)` 以属性方式访问 `$this->model->sub_categories`，触发 Eloquent 懒加载，将子分类集合缓存到 `$this->model->relations['sub_categories']`。
2. **getSubCategoryRelationships() 第 109 行**：`foreach ($model->sub_categories as $sub_category)` 对每个子分类再次以属性方式访问，将孙分类缓存到子模型的 `relations`。
3. **deleteSubCategories() 第 38 行**：`foreach ($model->sub_categories as $sub_category)` 访问的是**已缓存的集合**（Eloquent 对已加载的关系不会重复查询）。

因此当第 29 行 `event(new CategoryDeleted($this->model))` 触发时，`$this->model` 的 `sub_categories` 关系**已在步骤 1 被加载到内存**。即使数据库中这些子分类已被路径 A 软删除，Listener 第 23 行 `$category->sub_categories` 仍会返回内存中已缓存的集合。

### 事件触发时子分类的状态

1. 事务内 `deleteSubCategories($this->model)` 先软删除主分类，再遍历**已缓存**的 `sub_categories` 集合递归软删除所有子孙分类
2. 事务提交后，所有子孙分类的 `deleted_at` 在数据库中已非空
3. `event(new CategoryDeleted($this->model))` 触发
4. Listener 第 23 行 `$category->sub_categories` 返回**内存中已缓存的集合**（包含所有已软删除的子分类模型）
5. Listener 为每个子分类派发新的 `DeleteCategory` Job
6. 新 Job 的 `authorize()` 会对已删除模型执行关联检查

---

## 3. 多层子分类是否会递归触发

### 主删除路径：DeleteCategory 内部递归（路径 A）

[DeleteCategory.php:L34-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L34-L41) 中的 `deleteSubCategories()` 在事务内递归：

```
deleteSubCategories(父类A)
  → 软删除 A
  → 遍历 A.relations['sub_categories']（已缓存）
       → deleteSubCategories(子类B)
            → 软删除 B
            → 遍历 B.relations['sub_categories']（已缓存）
                 → deleteSubCategories(孙类C)
                      → 软删除 C
```

**这条路径是实际完成删除的路径。**

### 事件驱动路径：Listener 派发（路径 B）

监听器收到 CategoryDeleted 事件后，为每个子分类派发新的 `DeleteCategory` Job，形成事件链：

```
CategoryDeleted(A) → dispatch DeleteCategory(B)
  → DeleteCategory(B).handle()
    → authorize()（对已删除模型检查）
    → deleteSubCategories(B)（对已删除模型再次软删除）
    → CategoryDeleted(B) → dispatch DeleteCategory(C)
      → 同上...
```

### 路径 A 与路径 B 的关系

| 维度 | 路径 A（内部递归） | 路径 B（事件驱动） |
|------|-------------------|-------------------|
| 执行时机 | 事务内，先删父节点 | 事务提交后，删除后 |
| 子分类状态 | 未删除 | 已被路径 A 软删除 |
| 实际效果 | 首次软删除 | 对已删除模型重复操作 |
| 子分类来源 | 内存缓存的关系集合 | 内存缓存的关系集合 |
| 授权检查 | 不在节点内执行 | 对已删除模型执行 |

**路径 B 不是独立的删除路径**，它是在路径 A 已完成删除后，对已删除模型触发的重复操作。

### 路径 A 中授权检查的实际执行范围

需要区分两个阶段：

**阶段 1：handle() 进入事务前的 authorize()**

[DeleteCategory.php:L46-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L46-L73) 中的 `authorize()` 在进入事务前执行一次。它：

- 第 64 行：对主分类调用 `getRelationships()` 检查引用
- 第 70-72 行：遍历主分类的 `sub_categories`，对每个子分类调用 `getSubCategoryRelationships()`
- [getSubCategoryRelationships()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L101-L112) 递归调用自身，对所有子孙分类调用 `getRelationships()`

因此 **authorize() 在事务开始前已递归检查了整棵分类树中每个节点的 items / invoices / bills / transactions 引用**，以及默认收入/支出分类判断。如果任一节点被引用，会抛出异常阻止删除。

**阶段 2：deleteSubCategories() 内部不再执行 authorize()**

[DeleteCategory.php:L34-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L34-L41) 的 `deleteSubCategories()` 直接调用 `$model->delete()`，**不会对每个节点再次调用 authorize() 或 getRelationships()**。

**影响**：阶段 1 的 authorize() 已覆盖了对 items / transactions / documents 引用的保护，因此路径 A 的 deleteSubCategories() 不需要再次检查。但路径 B 中监听器派发的新 `DeleteCategory` Job 会再次调用 `authorize()`，此时模型已被软删除，`getRelationships()` 中的 `countRelationships()` 仍会查询数据库中该 category_id 的关联记录，可能因子分类已删除而无法正确判断。

---

## 4. 已有 item/transaction/document/document_item/contact 引用时的保护机制

### authorize() 中的四层检查

[DeleteCategory.php:L46-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L46-L73)

#### 第一层：转账分类禁止删除

```php
if ($this->model->isTransferCategory()) {
    throw new \Exception($message);
}
```

#### 第二层：同类型最后一个父分类保护

```php
if (Category::where('type', $this->model->type)->count() == 1 
    && $this->model->parent_id === null) {
    throw new LastCategoryDelete($message);
}
```

#### 第三层：关联引用检查（核心）

[getRelationships()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L75-L99) 检查四类关联：

```php
$rels = [
    'items' => 'items',
    'invoices' => 'invoices',
    'bills' => 'bills',
    'transactions' => 'transactions',
];
$relationships = $this->countRelationships($model, $rels);
```

同时检查默认分类：

```php
if ($model->id == setting('default.income_category')) {
    $relationships[] = 'income';
}
if ($model->id == setting('default.expense_category')) {
    $relationships[] = 'expense';
}
```

#### 第四层：递归检查子分类引用

[getSubCategoryRelationships()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L101-L112) 递归遍历所有子孙分类，调用 `getRelationships()` 执行同样的检查。

### 各引用类型的数据库约束、模型关系、应用层检查对照表

| 引用类型 | 数据表 | 字段 | Category 模型 hasMany 关系 | getRelationships() 检查 | 数据库外键约束 |
|---------|--------|------|--------------------------|------------------------|---------------|
| Item | items | category_id | `items()` ([Category.php:L133-L136](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L133-L136)) | ✅ `'items' => 'items'` | ❌ 无外键 |
| Transaction | transactions | category_id | `transactions()` ([Category.php:L138-L141](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L138-L141)) | ✅ `'transactions' => 'transactions'` | ❌ 无外键，仅索引（[v2.php:L411](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L411-L411)） |
| Invoice Document | documents | category_id | `invoices()` ([Category.php:L128-L131](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L128-L131)) | ✅ `'invoices' => 'invoices'` | ❌ 无外键（[v2.php:L120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L120-L120)） |
| Bill Document | documents | category_id | `bills()` ([Category.php:L113-L116](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L113-L116)) | ✅ `'bills' => 'bills'` | ❌ 无外键（[v2.php:L120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L120-L120)） |
| Document Item | document_items | category_id | **无 hasMany 关系** | ❌ **未进入 $rels** | ❌ 无外键，仅索引（[v3122.php:L21-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2026_02_17_000000_core_v3122.php#L21-L25)） |
| Contact | contacts | category_id | **无 hasMany 关系** | ❌ **未进入 $rels** | ❌ 无外键，仅索引（[v3122.php:L28-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2026_02_17_000000_core_v3122.php#L28-L30)） |
| Category 自引用 | categories | parent_id | `sub_categories()` ([Category.php:L103-L106](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L103-L106)) | N/A | ✅ 外键（[v300.php:L32-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_05_10_000000_core_v300.php#L32-L35)） |

### document_items.category_id 与 contacts.category_id 引用的缺失

[v3122 迁移](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2026_02_17_000000_core_v3122.php#L21-L30) 分别为 `document_items` 和 `contacts` 添加了 `category_id` 字段并创建索引：

```php
Schema::table('document_items', function (Blueprint $table) {
    $table->unsignedInteger('category_id')->nullable()->after('item_id');
    $table->index('category_id');
});

Schema::table('contacts', function (Blueprint $table) {
    $table->unsignedInteger('category_id')->nullable()->after('user_id');
    $table->index('category_id');
});
```

[DocumentItem 模型](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/DocumentItem.php#L78-L81) 和 [Contact 模型](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Contact.php#L143-L146) 都定义了反向 belongsTo 关系：

```php
// DocumentItem
public function category()
{
    return $this->belongsTo('App\Models\Setting\Category');
}

// Contact
public function category()
{
    return $this->belongsTo(Category::class)
        ->withoutGlobalScope('App\Scopes\Category')
        ->withDefault(['name' => trans('general.na')]);
}
```

**但 Category 模型中没有定义 `document_items()` 或 `contacts()` 的 hasMany 关系**，因此 `getRelationships()` 中的 `$rels` 数组无法检查这两类引用。如果有 document_item 或 contact 引用了某个 category，删除保护不会拦截。

### 数据库约束总结

| 表 | 字段 | 约束类型 | 迁移来源 |
|----|------|---------|---------|
| categories | parent_id | ✅ FOREIGN KEY (parent_id) REFERENCES categories(id) | [2022_05_10_000000_core_v300.php:L32-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2022_05_10_000000_core_v300.php#L32-L35) |
| categories | parent_id | ✅ 同迁移定义 | v300.php |
| transactions | category_id | ❌ 仅 INDEX | [2019_11_16_000000_core_v2.php:L411](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L411-L411) |
| documents | category_id | ❌ 无约束 | [2019_11_16_000000_core_v2.php:L120](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2019_11_16_000000_core_v2.php#L120-L120) |
| items | category_id | ❌ 无约束 | 2017_09_14_000000_core_v1.php |
| document_items | category_id | ❌ 仅 INDEX | [2026_02_17_000000_core_v3122.php:L22-L24](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2026_02_17_000000_core_v3122.php#L22-L24) |
| contacts | category_id | ❌ 仅 INDEX | [2026_02_17_000000_core_v3122.php:L29-L30](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2026_02_17_000000_core_v3122.php#L29-L30) |

**只有 `categories.parent_id` 有自引用外键约束**，其他表的 `category_id` 字段均无外键约束：
- `items.category_id` 和 `documents.category_id`：无数据库层约束，完全依赖 `getRelationships()` 应用层检查
- `transactions.category_id`：仅有 INDEX，无 FOREIGN KEY，依赖应用层检查
- `document_items.category_id` 和 `contacts.category_id`：仅有 INDEX，既无 FOREIGN KEY 也未进入 `getRelationships()` 的 `$rels` 数组，完全无保护

---

## 5. 只 mock DeleteCategory 而没触发事件会漏测的问题

### 现有测试覆盖分析

[CategoriesTest.php:L70-L87](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Settings/CategoriesTest.php#L70-L87) 中的 `testItShouldDeleteCategory()`：

```php
public function testItShouldDeleteCategory()
{
    $request = $this->getRequest();

    $this->dispatch(new CreateCategory(array_merge($request, [
        'name' => $this->faker->text(15),
    ])));

    $category = $this->dispatch(new CreateCategory($request));

    $this->loginAs()
        ->delete(route('categories.destroy', $category->id))
        ->assertStatus(200);

    $this->assertFlashLevel('success');

    $this->assertSoftDeleted('categories', $request);
}
```

**实际覆盖**：
- ✅ HTTP DELETE 路由可达
- ✅ 单个无子分类 category 的软删除（`parent_id` 为 null）
- ✅ 成功 flash 消息

**未覆盖**：
- ❌ 创建带 `parent_id` 的子分类，未验证子分类级联删除
- ❌ 未断言 CategoryDeleted 事件是否触发
- ❌ 未验证 DeleteCategoryDeletedSubCategories 监听器是否执行
- ❌ 未验证多层（父→子→孙）子分类的级联删除
- ❌ 未验证有引用时的阻止逻辑

### Mock DeleteCategory 导致漏测的场景

当测试场景是验证控制器或调用方是否正确派发了 `DeleteCategory` Job，但通过 mock 阻止 Job 的 `handle()` 实际执行（导致事件不触发、监听器不运行）：

```php
// 漏测场景：验证控制器派发了 DeleteCategory
public function testControllerDispatchesDeleteCategory()
{
    Queue::fake();  // 或 Bus::fake()

    $category = Category::factory()->create();

    $this->loginAs()
        ->delete(route('categories.destroy', $category->id));

    // 仅断言 DeleteCategory 被派发
    Queue::assertPushed(DeleteCategory::class);

    // 但 DeleteCategory 的 handle() 从未执行
    // CategoryDeleted 事件从未触发
    // DeleteCategoryDeletedSubCategories 监听器从未运行
}
```

### 会漏掉的具体断言或行为

| 漏掉的内容 | 具体说明 |
|-----------|---------|
| **子分类未被软删除** | mock 后 `handle()` 不执行，`deleteSubCategories()` 不运行，子分类仍在数据库中。如果测试仅断言父分类被删除（或断言 Job 被派发），不会发现子分类未被处理。 |
| **CategoryDeleted 事件未触发** | 无法验证 `handle()` 第 29 行的 `event(new CategoryDeleted(...))` 被调用，以及传入的 `$this->model` 是预期的已删除模型实例。 |
| **DeleteCategoryDeletedSubCategories 监听器未执行** | 无法发现监听器中 `$category->sub_categories` 的空判断分支是否正确、`$this->dispatch(new DeleteCategory($sub_category))` 是否成功派发。 |
| **路径 B 对已删除模型的重复操作** | 无法发现监听器派发的 `DeleteCategory` Job 在 `authorize()` 中对已删除模型执行 `getRelationships()` 时可能产生的问题（如 `countRelationships()` 对已删除模型调用 `$model->items()->count()`）。 |
| **多层级联的完整性** | 对于三层分类结构（父→子→孙），mock 后无法验证中间层和叶子层是否均被 `deleteSubCategories()` 递归处理，以及监听器是否为每一层派发 Job。 |

### 正确的验证方式

不 mock Job 或事件，通过集成测试验证完整流程：

```php
public function testDeleteCategoryWithSubcategories()
{
    // 创建三层分类结构
    $parent = Category::factory()->create();
    $child = Category::factory()->create(['parent_id' => $parent->id]);
    $grandchild = Category::factory()->create(['parent_id' => $child->id]);

    // 不 mock，走完整流程
    $this->loginAs()
        ->delete(route('categories.destroy', $parent->id));

    // 断言所有层级均被软删除
    $this->assertSoftDeleted('categories', ['id' => $parent->id]);
    $this->assertSoftDeleted('categories', ['id' => $child->id]);
    $this->assertSoftDeleted('categories', ['id' => $grandchild->id]);
}
```

---

## 总结

### 核心设计特征

1. **路径 A（事务内递归）是实际删除路径**：`deleteSubCategories()` 在事务内递归软删除所有层级，先删父节点再删子节点，子分类来源是 authorize() 阶段已加载到内存的关系集合
2. **路径 B（事件驱动）是重复操作**：`CategoryDeleted` 事件触发时子分类已被路径 A 软删除，Listener 通过内存中缓存的 `sub_categories` 集合为子分类派发新的 `DeleteCategory` Job
3. **主分类被重复删除**：`deleteSubCategories($this->model)` 已删除主分类，第 26 行 `$this->model->delete()` 是冗余调用
4. **withSubCategory() 不移除 SoftDeletes 过滤**：它仅移除 `parent_id IS NULL` 的 Scope，不影响软删除记录的可见性

### 引用保护覆盖

- ✅ `items.category_id`：Category 有 `items()` hasMany，getRelationships() 检查 `'items'`
- ✅ `transactions.category_id`：Category 有 `transactions()` hasMany，getRelationships() 检查 `'transactions'`
- ✅ `documents.category_id`（invoice）：Category 有 `invoices()` hasMany，getRelationships() 检查 `'invoices'`
- ✅ `documents.category_id`（bill）：Category 有 `bills()` hasMany，getRelationships() 检查 `'bills'`
- ❌ `document_items.category_id`：Category 无 `document_items()` hasMany，未进入 `$rels`
- ❌ `contacts.category_id`：Category 无 `contacts()` hasMany，未进入 `$rels`

### 数据库约束

- ✅ 仅 `categories.parent_id` 有自引用 FOREIGN KEY（v300 迁移）
- ❌ `items.category_id`、`documents.category_id`：无约束，完全依赖应用层
- ❌ `transactions.category_id`、`document_items.category_id`、`contacts.category_id`：仅 INDEX

### 测试缺口

现有 [testItShouldDeleteCategory()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Settings/CategoriesTest.php#L70-L87) 仅覆盖单个无子分类 category 的删除，未创建 `parent_id` 子分类，未断言子分类级联删除、事件触发和监听器执行。
