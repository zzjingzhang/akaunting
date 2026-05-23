# Akaunting 删除分类时子分类处理机制分析

## 1. DeleteCategory 本体做了什么，事件在哪里触发

### 核心流程
[DeleteCategory.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php) 是删除分类的核心 Job 类，执行流程如下：

```php
public function handle(): bool
{
    $this->authorize();                     // 权限与引用检查

    event(new CategoryDeleting($this->model));  // 触发删除前事件

    \DB::transaction(function () {
        $this->deleteSubCategories($this->model);  // 递归删除子分类
        $this->model->delete();                    // 删除主分类
    });

    event(new CategoryDeleted($this->model));  // 触发删除后事件

    return true;
}
```

### 事件触发位置
- **CategoryDeleting**：[DeleteCategory.php:L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L21-L21) — 删除前事件
- **CategoryDeleted**：[DeleteCategory.php:L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L29-L29) — 删除后事件

### 事件注册
事件与监听器的映射在 [Event.php:L121-L123](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L121-L123) 中注册：
```php
\App\Events\Setting\CategoryDeleted::class => [
    \App\Listeners\Setting\DeleteCategoryDeletedSubCategories::class,
],
```

---

## 2. CategoryDeleted Listener 如何找到并删除子分类

[DeleteCategoryDeletedSubCategories.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Setting/DeleteCategoryDeletedSubCategories.php) 监听 CategoryDeleted 事件：

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

### 工作机制
1. 通过 Eloquent 关系 `$category->sub_categories` 获取直接子分类（定义在 [Category.php:L103-L106](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Setting/Category.php#L103-L106)）
2. 为每个子分类派发新的 `DeleteCategory` Job
3. 新的 Job 会重复完整的删除流程（授权检查 → 删除前事件 → 事务删除 → 删除后事件）

---

## 3. 多层子分类是否会递归触发

**答案：会递归触发，但存在**「双重删除」**的设计问题。**

### 两条独立的递归路径

#### 路径 A：DeleteCategory 内部递归（事务内）
[DeleteCategory.php:L34-L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L34-L41)
```php
public function deleteSubCategories($model)
{
    $model->delete();  // 先删除当前分类
    foreach ($model->sub_categories as $sub_category) {
        $this->deleteSubCategories($sub_category);  // 递归处理子分类
    }
}
```

#### 路径 B：Listener 事件驱动递归（事务外）
监听器收到 CategoryDeleted 事件后，为每个子分类派发新的 DeleteCategory Job，形成事件链：
```
删除 Category A → 触发 CategoryDeleted(A) → Listener 派发 DeleteCategory(B)
删除 Category B → 触发 CategoryDeleted(B) → Listener 派发 DeleteCategory(C)
删除 Category C → ...
```

### 关键问题：双重删除
在 `handle()` 方法中：
```php
\DB::transaction(function () {
    $this->deleteSubCategories($this->model);  // 路径A：已删除所有子分类 + 主分类
    $this->model->delete();                    // 主分类被第二次删除！
});

event(new CategoryDeleted($this->model));      // 触发路径B
```

**问题分析：**
1. `deleteSubCategories($this->model)` 首先删除主分类，然后递归删除所有子孙分类
2. `$this->model->delete()` 对已删除的主分类执行第二次删除（软删除下只是更新 deleted_at）
3. 事件触发时，所有子分类实际上已被路径A删除，监听器尝试获取 `$category->sub_categories` 可能返回空集合或已删除模型

---

## 4. 已有 item/transaction/document 引用时的保护机制

### 保护机制（存在于 authorize 阶段）

[DeleteCategory.php:L46-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L46-L73) 中的 `authorize()` 方法执行三层检查：

#### 第一层：特殊分类禁止删除
```php
if ($this->model->isTransferCategory()) {
    throw new \Exception($message);  // 转账分类不可删除
}
```

#### 第二层：最后一个父分类保护
```php
if (Category::where('type', $this->model->type)->count() == 1 
    && $this->model->parent_id === null) {
    throw new LastCategoryDelete($message);  // 同类型最后一个父分类不可删除
}
```

#### 第三层：关联引用检查（核心）
[getRelationships()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L75-L99) 检查四类关联：
```php
$rels = [
    'items' => 'items',              // 商品
    'invoices' => 'invoices',        // 发票
    'bills' => 'bills',              // 账单
    'transactions' => 'transactions',// 交易记录
];
$relationships = $this->countRelationships($model, $rels);
```

同时检查是否为默认分类：
```php
if ($model->id == setting('default.income_category')) {
    $relationships[] = 'income';
}
if ($model->id == setting('default.expense_category')) {
    $relationships[] = 'expense';
}
```

#### 第四层：递归检查子分类引用
[getSubCategoryRelationships()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Setting/DeleteCategory.php#L101-L112) 递归遍历所有子孙分类，执行同样的关联检查。

### 潜在缺失

1. **路径A 无保护**：`deleteSubCategories()` 方法直接调用 `$model->delete()`，不进行任何关联检查。虽然 `authorize()` 已在事务前检查，但如果在高并发下检查与删除之间有新关联写入，会造成数据不一致。

2. **Listener 路径重复检查**：监听器派发的 DeleteCategory Job 会再次执行 authorize 检查，但此时子分类可能已被路径A删除，导致检查逻辑对已删除模型执行。

3. **缺少数据库外键约束**：数据库层面没有 ON DELETE RESTRICT 约束，完全依赖应用层检查。

---

## 5. 只 mock DeleteCategory 而没触发事件会漏测的问题

### 漏测场景说明

现有测试 [CategoriesTest.php:L70-L87](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Settings/CategoriesTest.php#L70-L87) 通过 HTTP 请求调用删除流程，完整触发了事件链。但如果单元测试只直接 dispatch DeleteCategory Job 并 mock 掉事件系统：

```php
// 不良测试示例（漏测）
public function testDeleteCategory()
{
    Event::fake();  // 阻止事件触发
    
    $category = Category::factory()->hasSubCategories(2)->create();
    
    $this->dispatch(new DeleteCategory($category));
    
    // 断言主分类和子分类都被删除
    $this->assertSoftDeleted($category);
    foreach ($category->sub_categories as $sub) {
        $this->assertSoftDeleted($sub);  // 这个断言会通过，但掩盖了问题
    }
}
```

### 会漏测的问题

| 漏测问题 | 影响说明 |
|---------|---------|
| **Listener 本身的逻辑错误** | 无法发现监听器中可能存在的 bug，如关系加载失败、Job 派发失败等 |
| **事件参数传递问题** | CategoryDeleted 事件构造时如果参数错误（如传递了已删除模型），在单元测试中不会暴露 |
| **双重删除的副作用** | 无法发现路径A删除后，路径B尝试删除已删除模型时可能产生的异常 |
| **事件链完整性** | 无法验证多层子分类是否都能通过事件链正确触发删除 |
| **其他监听器遗漏** | 如果未来有其他监听器监听 CategoryDeleted 事件，mock 事件会导致这些监听器完全不被测试 |

### 推荐测试方式

**集成测试优先**：通过 HTTP 接口或完整 Job 派发（不 mock 事件）来测试删除流程，确保事件链完整执行。

```php
// 推荐测试方式
public function testDeleteCategoryWithSubcategories()
{
    // 创建三层分类结构
    $parent = Category::factory()->create();
    $child = Category::factory()->create(['parent_id' => $parent->id]);
    $grandchild = Category::factory()->create(['parent_id' => $child->id]);
    
    // 不 mock 事件，走完整流程
    $response = $this->loginAs()
        ->delete(route('categories.destroy', $parent->id));
    
    // 断言所有层级都被删除
    $this->assertSoftDeleted($parent);
    $this->assertSoftDeleted($child);
    $this->assertSoftDeleted($grandchild);
}
```

---

## 总结：架构问题与改进建议

### 核心问题
1. **双重删除设计冗余**：路径A（内部递归）和路径B（事件驱动）功能重叠，增加复杂度
2. **deleteSubCategories 逻辑顺序问题**：先删父分类再删子分类，不符合直觉且可能导致外键问题
3. **主分类重复删除**：`deleteSubCategories($this->model)` 已删除主分类，后续 `$this->model->delete()` 冗余

### 改进建议
方案一（保留事件驱动）：移除 Job 内部的 deleteSubCategories 递归，完全依赖监听器处理子分类
方案二（简化实现）：移除监听器，只保留 Job 内部递归，事件只用于通知审计日志等
方案三（修复现有逻辑）：修正 deleteSubCategories 顺序，先删子分类后删父分类，移除重复删除
