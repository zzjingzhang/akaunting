# Akaunting 报表事件订阅器组装机制分析

## 1. Event Provider 中 Report Subscribers 如何注册

### 注册位置
在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L139-L156) 的 `$subscribe` 属性中统一注册：

```php
protected $subscribe = [
    \App\Listeners\Report\AddAccounts::class,
    \App\Listeners\Report\AddContacts::class,
    \App\Listeners\Report\AddCustomers::class,
    \App\Listeners\Report\AddVendors::class,
    \App\Listeners\Report\AddExpenseCategories::class,
    \App\Listeners\Report\AddIncomeCategories::class,
    \App\Listeners\Report\AddIncomeExpenseCategories::class,
    \App\Listeners\Report\AddSearchString::class,
    \App\Listeners\Report\AddRowsToTax::class,
    \App\Listeners\Report\AddGroup::class,
    \App\Listeners\Report\AddBasis::class,
    \App\Listeners\Report\AddPeriod::class,
    \App\Listeners\Report\AddDate::class,
    \App\Listeners\Report\AddDiscount::class,
];
```

### 订阅器工作原理

每个 Report 监听器继承自 [Report Listener 基类](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Listeners/Report.php#L13-L363)，基类中的 `subscribe()` 方法实现自动事件绑定：

```php
public function subscribe($events)
{
    $class = get_class($this);

    foreach ($this->events as $event) {
        $method = 'handle' . (new \ReflectionClass($event))->getShortName();

        if (!method_exists($class, $method)) {
            continue;
        }

        $events->listen(
            $event,
            $class . '@' . $method
        );
    }
}
```

基类预定义了监听的事件类型：
```php
protected $events = [
    \App\Events\Report\FilterShowing::class,
    \App\Events\Report\FilterApplying::class,
    \App\Events\Report\GroupShowing::class,
    \App\Events\Report\GroupApplying::class,
    \App\Events\Report\RowsShowing::class,
];
```

### 类过滤机制

每个监听器通过 `$classes` 属性指定生效的报表类，在 `skipThisClass()` 方法中进行过滤：
```php
protected $classes = [
    \App\Reports\ProfitLoss::class,
];

public function skipThisClass($event)
{
    return (empty($event->class) || !in_array(get_class($event->class), $this->classes));
}
```

---

## 2. ProfitLoss 自身定义的基础属性 vs Listener 注入的行/维度

### ProfitLoss 自身定义的基础属性

在 [ProfitLoss.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L13-L301) 中定义：

| 属性 | 说明 |
|------|------|
| `$default_name` | 报表名称：`reports.profit_loss` |
| `$category` | 分类：`general.accounting` |
| `$icon` | 图标：`favorite_border` |
| `$type` | 类型：`detail` |
| `$chart` | 是否显示图表：`false` |
| `$gross_profit` | 毛利润数据数组 |
| `$net_profit` | 净利润数据数组 |

### ProfitLoss 自身定义的 Tables（三大分组）

```php
public function setTables()
{
    $this->tables = [
        Category::INCOME_TYPE  => trans_choice('general.incomes', 1),
        Category::COGS_TYPE    => trans_choice('general.cogs', 2),
        Category::EXPENSE_TYPE => trans_choice('general.expenses', 2),
    ];
}
```

### 来自 Listener 的行/维度

| 监听器 | 注入内容 |
|--------|----------|
| [AddIncomeExpenseCategories](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L11-L122) | `category` 分组维度、所有收入/支出/COGS 类别的行数据、树形层级结构 |
| [AddPeriod](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L8-L39) | `period` 过滤器（周/月/季/年） |
| [AddBasis](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddBasis.php#L8-L39) | `basis` 过滤器（权责发生制/收付实现制） |
| [AddGroup](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddGroup.php#L8-L39) | `group` 过滤器（分组选项） |
| [AddDate](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddDate.php) | 日期范围过滤器 |
| [AddSearchString](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddSearchString.php) | 搜索字符串过滤功能 |

---

## 3. AddAccounts / AddIncomeExpenseCategories / AddPeriod 分别改了什么

### AddAccounts

**适用报表**：`IncomeSummary`, `ExpenseSummary`, `IncomeExpenseSummary`

**修改内容**：

1. **FilterShowing 阶段**（[L25-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L25-L35)）：
   - 添加 `accounts` 过滤器到 `$event->class->filters`
   - 设置过滤器的路由和多选属性

2. **GroupShowing 阶段**（[L43-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L43-L50)）：
   - 添加 `account` 分组选项到 `$event->class->groups`

3. **GroupApplying 阶段**（[L58-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L58-L65)）：
   - 调用 `applyAccountGroup()` 将 `account_id` 注入到模型属性

4. **RowsShowing 阶段**（[L73-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L73-L92)）：
   - 当分组为 `account` 时，初始化所有账户的 `row_names` 和 `row_values`

### AddIncomeExpenseCategories

**适用报表**：`DiscountSummary`, `IncomeExpenseSummary`, `ProfitLoss`

**修改内容**：

1. **FilterShowing 阶段**（[L25-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L25-L36)）：
   - 添加 `categories` 过滤器，包含收入、支出、COGS 三种类型的类别

2. **GroupShowing 阶段**（[L44-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L44-L51)）：
   - 添加 `category` 分组选项

3. **RowsShowing 阶段**（[L59-L111](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L59-L111)）：
   - 按 table 类型（income/cogs/expense）分配对应的类别行
   - 初始化 `row_names[$table_key][$id]` 和 `row_values[$table_key][$id][$date] = 0`
   - 构建 `row_tree_nodes` 树形层级结构（支持父子类别）

### AddPeriod

**适用报表**：`IncomeSummary`, `ExpenseSummary`, `IncomeExpenseSummary`, `ProfitLoss`, `TaxSummary`, `DiscountSummary`

**修改内容**：

**FilterShowing 阶段**（[L25-L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L25-L39)）：
- 添加 `period` 过滤器，选项包括：`weekly`/`monthly`/`quarterly`/`yearly`
- 设置默认值为 `quarterly`
- 配置过滤器操作符

---

## 4. 导出报表时如何消费这些结构

### 导出入口

在 [Report 基类](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L315-L318) 的 `export()` 方法：

```php
public function export()
{
    return ExportHelper::toExcel(new Export($this->views[$this->type], $this), $this->model->name);
}
```

### Reports 导出类

[Reports.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Common/Reports.php#L11-L34) 通过视图渲染导出：

```php
public function view(): View
{
    return view($this->view, ['class' => $this->class, 'print' => true]);
}
```

### 视图层消费结构

以 [body.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/components/reports/detail/table/body.blade.php#L1-L13) 为例：

```blade
<tbody>
    @if (!empty($class->row_values[$table_key]))
        @foreach($class->row_tree_nodes[$table_key] as $id => $node)
            @include($class->views['detail.table.row'], ['tree_level' => 0])
        @endforeach
    @else
        <tr>
            <td colspan="{{ count($class->dates) + 2 }}">
                <div class="text-muted pl-0">{{ trans('general.no_records') }}</div>
            </td>
        </tr>
    @endif
</tbody>
```

### 核心数据结构消费

| 结构 | 用途 |
|------|------|
| `$class->dates` | 表头的时间周期列（如各月/各季） |
| `$class->row_names[$table_key][$id]` | 行名称（如类别名称） |
| `$class->row_values[$table_key][$id][$date]` | 单元格数值（某类别在某周期的金额） |
| `$class->row_tree_nodes[$table_key]` | 树形结构，用于渲染层级缩进 |
| `$class->footer_totals[$table_key][$date]` | 各周期的合计值 |
| `$class->tables` | 三大分组（收入/COGS/支出）的标题 |

### 数组导出

[Report::array()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L264-L297) 方法将报表数据结构化输出：

```php
public function array(): array
{
    $data = [];
    $group = Str::plural($this->group ?? $this->getGroup());

    foreach ($this->tables as $table_key => $table_name) {
        foreach ($this->row_values[$table_key] as $key => $values) {
            $data[$table_key][$group][$this->row_names[$table_key][$key]] = $values;
        }
        $data[$table_key]['totals'] = $this->footer_totals[$table_key];
    }

    return $data;
}
```

---

## 5. 只读 ProfitLoss.php 会漏掉的行或分组

### 漏掉的分组维度

| 分组维度 | 来源 | 作用 |
|----------|------|------|
| `category` 分组 | [AddIncomeExpenseCategories](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L44-L51) | 按类别分组查看损益数据 |

### 漏掉的行数据初始化

**所有类别的行数据** 完全由 `AddIncomeExpenseCategories::handleRowsShowing()` 注入，包括：
- `row_names['income'][$category_id]` - 收入类别名称
- `row_names['cogs'][$category_id]` - COGS 类别名称  
- `row_names['expense'][$category_id]` - 支出类别名称
- `row_values[$table][$category_id][$date] = 0` - 初始化为 0 的数值矩阵
- `row_tree_nodes[$table][$category_id]` - 树形层级结构

### 漏掉的过滤器

| 过滤器 | 来源 |
|--------|------|
| `period`（周/月/季/年） | [AddPeriod](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L25-L39) |
| `basis`（权责发生制/收付实现制） | [AddBasis](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddBasis.php#L24-L38) |
| `group` 选项 | [AddGroup](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddGroup.php#L23-L38) |
| `categories` 类别筛选 | [AddIncomeExpenseCategories](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L25-L36) |
| `date_range` 日期范围 | [AddDate](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddDate.php) |
| 搜索字符串功能 | [AddSearchString](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddSearchString.php) |

### 核心结论

**只读 `ProfitLoss.php` 只能看到报表的「骨架」（三大 table 定义、计算逻辑），但看不到「血肉」——所有具体的行数据（各类别）、分组维度选项、过滤选项都是通过事件订阅器动态注入的。** 这种设计使得：
1. 不同报表可以复用相同的监听器（如 AddPeriod 被 6 个报表共用）
2. 新增分组维度不需要修改报表类本身
3. 模块可以通过订阅事件扩展报表功能
