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

| 属性 | 来源行号 | 说明 |
|------|----------|------|
| `$default_name` | [L15](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L15) | 报表名称：`reports.profit_loss` |
| `$category` | [L17](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L17) | 分类：`general.accounting` |
| `$icon` | [L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L19) | 图标：`favorite_border` |
| `$type` | [L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L21) | 类型：`detail` |
| `$chart` | [L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L23) | 是否显示图表：`false` |
| `$gross_profit` | [L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L25) | 毛利润数据数组（由 `setGrossProfit()` 填充） |
| `$net_profit` | [L27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L27) | 净利润数据数组（由 `setNetProfit()` 填充） |

### ProfitLoss 自身定义的方法

| 方法 | 来源行号 | 说明 |
|------|----------|------|
| `setViews()` | [L29-L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L29-L38) | 覆盖基类视图路径，重定向到 `reports.profit_loss.*` |
| `setTables()` | [L40-L47](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L40-L47) | 定义 income/cogs/expense 三大 table |
| `setData()` | [L49-L113](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L49-L113) | 核心数据计算（权责/收付两种 basis 分支） |
| `setGrossProfit()` | [L131-L139](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L131-L139) | 毛利润 = 收入合计 - COGS 合计 |
| `setNetProfit()` | [L141-L162](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L141-L162) | 净利润 = 收入合计 - 支出合计（不含 COGS） |
| `array()` | [L164-L180](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L164-L180) | 数组输出，追加 gross_profit/net_profit |
| `getFields()` | [L182-L190](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L182-L190) | 覆盖基类，额外追加 `getPercentageField()` |
| `getPercentageField()` | [L192-L208](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L192-L208) | 收入百分比筛选器（ProfitLoss 独有） |
| `getPercentageOfIncome()` | [L215-L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L215-L232) | 计算某单元格占收入合计的百分比 |
| `getDrillDownUrl()` | [L234-L251](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L234-L251) | 生成下钻链接 |

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

### ProfitLoss 视图覆盖与通用视图协作

**ProfitLoss::setViews()** 在 [L29-L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L29-L38) 只覆盖了基类 `setViews()` 中定义的 6 个视图路径：

```php
public function setViews()
{
    parent::setViews();  // 先调用基类加载全部默认视图
    $this->views['detail'] = 'reports.profit_loss.detail';
    $this->views['detail.content.header'] = 'reports.profit_loss.content.header';
    $this->views['detail.content.footer'] = 'reports.profit_loss.content.footer';
    $this->views['detail.table.header'] = 'reports.profit_loss.table.header';
    $this->views['detail.table.row'] = 'reports.profit_loss.table.row';
    $this->views['detail.table.footer'] = 'reports.profit_loss.table.footer';
}
```

**未被覆盖的视图**（使用基类默认的 `components.reports.*`）：

| 视图键 | 基类默认路径 | 作用 |
|--------|-------------|------|
| `show` | `components.reports.show` | 页面外壳 |
| `print` | `components.reports.print` | 打印外壳 |
| `filter` | `components.reports.filter` | 筛选器区域 |
| `detail.table` | `components.reports.detail.table` | 表格容器（包含 header/body/footer） |
| `detail.table.body` | `components.reports.detail.table.body` | **遍历 row_tree_nodes 渲染行** |

**通用 body 视图遍历 row_tree_nodes**：

基类 [body.blade.php#L1-L13](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/components/reports/detail/table/body.blade.php#L1-L13) 负责：

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

**视图协作流程**：
1. ProfitLoss 自定义的 `detail` 视图（[detail.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/detail.blade.php#L1-L38)）遍历三大 table
2. 每个 table 引入基类默认的 `detail.table` 视图（[table.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/components/reports/detail/table.blade.php#L1-L7)）
3. `detail.table` 内部引入基类默认的 `detail.table.body` 视图
4. `detail.table.body` 遍历 `row_tree_nodes` 并调用 ProfitLoss 自定义的 `detail.table.row` 视图渲染每一行
5. ProfitLoss 自定义的 row 视图（[row.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/table/row.blade.php#L1-L120)）处理树形递归、占比显示、drill-down 链接等

### 来自 Listener 的行/维度

| 监听器 | 注入内容 |
|--------|----------|
| [AddIncomeExpenseCategories](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L11-L122) | `category` 分组维度、所有收入/支出/COGS 类别的行数据、树形层级结构 |
| [AddPeriod](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L8-L39) | `period` 过滤器元数据（周/月/季/年选项） |
| [AddBasis](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddBasis.php#L8-L39) | `basis` 过滤器元数据（权责发生制/收付实现制选项） |
| [AddGroup](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddGroup.php#L8-L39) | `group` 过滤器元数据（分组选项） |
| [AddDate](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddDate.php) | 日期范围过滤器元数据 |
| [AddSearchString](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddSearchString.php) | 搜索字符串过滤功能 |

---

## 3. AddAccounts / AddIncomeExpenseCategories / AddPeriod 分别改了什么

### AddAccounts

**适用报表**：`IncomeSummary`, `ExpenseSummary`, `IncomeExpenseSummary`（**不适用 ProfitLoss**）

**修改内容**：

1. **FilterShowing 阶段**（[AddAccounts.php#L25-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L25-L35)）：
   - 添加 `accounts` 过滤器到 `$event->class->filters`
   - 设置过滤器的路由和多选属性

2. **GroupShowing 阶段**（[AddAccounts.php#L43-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L43-L50)）：
   - 添加 `account` 分组选项到 `$event->class->groups`

3. **GroupApplying 阶段**（[AddAccounts.php#L58-L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L58-L65)）：
   - 调用 `applyAccountGroup()` 注入 `account_id` 到模型属性
   - **关键限定**：`applyAccountGroup()` 在 [Report Listener#L213-L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Listeners/Report.php#L213-L232) 中只对 `documents` 表生效：
     ```php
     public function applyAccountGroup($event)
     {
         if ($event->model->getTable() != 'documents') {
             return;  // 非 documents 表直接跳过
         }
         // ...
     }
     ```

4. **RowsShowing 阶段**（[AddAccounts.php#L73-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L73-L92)）：
   - 当分组为 `account` 时，初始化所有账户的 `row_names` 和 `row_values`
   - **account_id 筛选分支**（[AddAccounts.php#L79-L90](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddAccounts.php#L79-L90)）：若 URL search string 中含 `account_id`，则只保留筛选出的账户行：
     ```php
     if ($account_ids = $this->getSearchStringValue('account_id')) {
         $accounts = explode(',', $account_ids);
         $rows = collect($all_accounts)->filter(function ($value, $key) use ($accounts) {
             return in_array($key, $accounts);
         });
     } else {
         $rows = $all_accounts;
     }
     ```

### AddIncomeExpenseCategories

**适用报表**：`DiscountSummary`, `IncomeExpenseSummary`, `ProfitLoss`

**修改内容**：

1. **FilterShowing 阶段**（[AddIncomeExpenseCategories.php#L25-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L25-L36)）：
   - 添加 `categories` 过滤器，包含收入、支出、COGS 三种类型的类别

2. **GroupShowing 阶段**（[AddIncomeExpenseCategories.php#L44-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L44-L51)）：
   - 添加 `category` 分组选项

3. **RowsShowing 阶段**（[AddIncomeExpenseCategories.php#L59-L111](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L59-L111)）：
   - 按 table 类型（income/cogs/expense）分配对应的类别行
   - 初始化 `row_names[$table_key][$id]` 和 `row_values[$table_key][$id][$date] = 0`
   - 构建 `row_tree_nodes` 树形层级结构（支持父子类别）

### AddPeriod

**适用报表**：`IncomeSummary`, `ExpenseSummary`, `IncomeExpenseSummary`, `ProfitLoss`, `TaxSummary`, `DiscountSummary`

**修改内容（仅添加筛选器元数据）**：

**FilterShowing 阶段**（[AddPeriod.php#L25-L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L25-L39)）：
```php
public function handleFilterShowing(FilterShowing $event)
{
    if ($this->skipThisClass($event)) { return; }

    $event->class->filters['period'] = $this->getPeriod();
    $event->class->filters['keys']['period'] = 'period';
    $event->class->filters['defaults']['period'] = $event->class->getSetting('period', 'quarterly');
    $event->class->filters['operators']['period'] = [
        'equal' => true, 'not_equal' => false, 'range' => false,
    ];
}
```
- **仅添加 `filters` 元数据**：4 个选项值、默认值、操作符配置
- **不生成时间列**：时间列由基类 `Report::setDates()` 生成

**period 时间列真正的生成路径**：

1. `Report::load()` 调用链在 [Report.php#L114-L127](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L114-L127)：`setYear()` → `setViews()` → `setTables()` → `setDates()` → ...

2. `Report::setDates()` 在 [Report.php#L407-L438](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L407-L438)：
   ```php
   public function setDates()
   {
       if (! $period = $this->getPeriod()) { return; }

       [$start, $end] = $this->getStartAndEndDates($this->year);

       $counter = match ($period) {
           'weekly'    => $end->diffInWeeks($start),
           'quarterly' => $end->diffInQuarters($start),
           'yearly'    => $end->diffInYears($start),
           default     => $end->diffInMonths($start),
       };

       for ($j = 0; $j <= $counter; $j++) {
           $date = $this->getPeriodicDate($start, $this->getPeriod(), $this->year);
           $this->dates[] = $date;
           // ...初始化 footer_totals
       }
   }
   ```

3. `Report::getPeriod()` 在 [Report.php#L612-L615](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L612-L615) 的默认值解析链：
   ```php
   public function getPeriod()
   {
       return $this->getSearchStringValue('period',
           $this->getSetting('period',
               $this->getDefaultFieldSelection($this->getPeriodField())
           )
       );
   }
   ```
   优先级：URL search string → 报表 model settings 中的 `period` → `getPeriodField()` 中硬编码的 `'quarterly'`

---

## 4. 导出报表时如何消费这些结构

ProfitLoss 有 **三种可达路径**，消费方式各不相同：

### 路径 A：Excel 导出（FromView 渲染）

**入口**：[Report::export()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L315-L318)
```php
public function export()
{
    return ExportHelper::toExcel(new Export($this->views[$this->type], $this), $this->model->name);
}
```

**中间类**：[Reports](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Exports/Common/Reports.php#L11-L34) 实现 `FromView`，通过 Blade 视图渲染导出：
```php
public function view(): View
{
    return view($this->view, ['class' => $this->class, 'print' => true]);
}
```

对于 ProfitLoss，`$this->views[$this->type]` = `reports.profit_loss.detail`，对应 [detail.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/detail.blade.php#L1-L38)。

**ProfitLoss 导出视图如何消费 gross_profit 和 net_profit**：

1. **gross_profit（毛利润）** 在 [detail.blade.php#L6-L35](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/detail.blade.php#L6-L35) 消费：
   - 遍历三大 table 后，在 `$table_key === 'cogs'` 且 `$class->gross_profit` 非空时输出
   - 直接读取 `$class->gross_profit` 数组中各周期的值，再计算合计

2. **net_profit（净利润）** 在 [content/footer.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/content/footer.blade.php#L1-L30) 消费：
   - 由 `detail.blade.php` 末尾 `@include($class->views['detail.content.footer'])` 引入
   - 直接遍历 `$class->net_profit` 数组输出

3. **通用行渲染** 在 [row.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/table/row.blade.php#L1-L120) 消费：
   - 读取 `$class->row_tree_nodes[$table_key]` 树形结构递归渲染
   - 叶子行读取 `$class->row_values[$table_key][$id]` 输出单元格值
   - 调用 `$class->getPercentageOfIncome($date, $cell_value)` 输出占比

4. **表头** 在 [content/header.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/content/header.blade.php#L1-L19) 消费 `$class->dates` 输出时间周期列

5. **各 table 合计** 在 [table/footer.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/reports/profit_loss/table/footer.blade.php#L1-L26) 消费 `$class->footer_totals[$table_key]`

### 路径 B：REST API 资源（JsonResource 序列化，不调用 array()）

**API 路由**：[routes/api.php#L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/api.php#L61)
```php
Route::resource('reports', 'Common\Reports');
```

**API Controller**：[Reports.php#L33-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Common/Reports.php#L33-L36)
```php
public function show(Report $report)
{
    return new Resource($report);
}
```

**API Resource**：[Report.php#L16-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Resources/Common/Report.php#L16-L31) 的 `toArray()` 将 `Report` Eloquent model 转为数组：
```php
public function toArray($request)
{
    return [
        'id' => $this->id,
        'class' => $this->class,
        'name' => $this->name,
        'settings' => $this->settings,
        'data' => $this->getReportData(),  // 关键：加载报表实例数据
        // ...
    ];
}
```

**getReportData()** 在 [Report.php#L33-L52](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Resources/Common/Report.php#L33-L52)：
```php
protected function getReportData()
{
    $report = Utility::getClassInstance($this);  // 实例化报表类
    // 只 unset model/views/loaded 等属性
    // 直接返回 $report 对象，由 Laravel JsonResource 自动序列化 public 属性
    return $report;
}
```

**Utility::getClassInstance()** 在 [Reports.php#L56-L73](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Utilities/Reports.php#L56-L73)：
```php
public static function getClassInstance($model, $load_data = true)
{
    // ...
    return new $class($model, $load_data);  // $load_data=true 触发构造函数中的 load()
}
```

**关键结论**：REST API **不调用** `ProfitLoss::array()`。API 返回的 `data` 字段是整个报表对象的 public 属性直接由 Laravel 序列化。其中：
- `gross_profit`、`net_profit`、`dates`、`row_names`、`row_values`、`row_tree_nodes`、`footer_totals`、`tables` 等全部作为顶级 key 出现
- 没有经过 `array()` 方法的 `[$group][$row_name]` 重新组织

### 路径 C：编程式 array() 方法（独立调用）

**入口**：[ProfitLoss::array()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Reports/ProfitLoss.php#L164-L180)
```php
public function array(): array
{
    $data = parent::array();  // 基类构建 income/cogs/expense 分组数据

    $net_profit = $this->net_profit;
    $gross_profit = $this->gross_profit;
    // ...money 格式化

    $data['gross_profit'] = $gross_profit;  // 追加毛利润
    $data['net_profit']   = $net_profit;    // 追加净利润

    return $data;
}
```

基类 [Report::array()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L264-L297) 消费 `row_values`、`row_names`、`footer_totals` 构建每个 table 的 `[$group][$row_name] = $values` 结构。

**与 API 路径的区别**：
- `array()` 输出经过重新组织，key 为 table 名称（如 `income`），内部按 group 类型分组（如 `categories`）
- API 输出直接平铺所有 public 属性（如 `row_names`、`row_values` 分别是独立的顶级 key）

输出结构示意：
```
[
  'income'  => ['categories' => ['收入类别A' => [...周期值...], ...], 'totals' => [...]],
  'cogs'    => ['categories' => ['COGS类别A' => [...周期值...], ...], 'totals' => [...]],
  'expense' => ['categories' => ['支出类别A' => [...周期值...], ...], 'totals' => [...]],
  'gross_profit' => [...各周期毛利润...],
  'net_profit'   => [...各周期净利润...],
]
```

**当前项目中 `array()` 没有被 API 或 UI 路径调用**，属于可被外部代码显式调用的独立编程接口。

### 路径 D：UI 查看（show() 方法）

**入口**：[Report::show()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Report.php#L254-L257)
```php
public function show()
{
    return view($this->views['show'], ['print' => false])->with('class', $this);
}
```

消费方式与 Excel 导出类似（同样走 Blade 视图），区别是 `print` 参数为 `false`，因此行数据中的 drill-down 链接正常渲染（`$is_print` 判断），而导出时 `$print` 为 `true` 则只输出纯文本金额。

### 四种可达路径对比总结

| 路径 | 入口 | 消费形式 | 是否调用 array() | gross_profit/net_profit |
|------|------|----------|-----------------|------------------------|
| Excel 导出 | `export()` → `FromView` | Blade 视图渲染 | 否 | detail/footer 视图直接消费 |
| REST API | `Resource::toArray()` → JsonSerialize | 对象属性序列化 | 否 | 作为顶级 key 出现在 data 字段中 |
| 编程 array() | 显式调用 `$report->array()` | PHP 数组 | 是（自身） | 作为额外 key 追加到父级输出 |
| UI 查看 | `show()` | Blade 视图渲染 | 否 | 同 Excel 导出，但 drill-down 链接可点击 |

---

## 5. 只读 ProfitLoss.php 会漏掉的行或分组

### 漏掉的分组维度

| 分组维度 | 来源 | 作用 |
|----------|------|------|
| `category` 分组 | [AddIncomeExpenseCategories#L44-L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L44-L51) | 按类别分组查看损益数据 |

### 漏掉的行数据初始化

**所有类别的行数据** 完全由 `AddIncomeExpenseCategories::handleRowsShowing()` 注入，包括：
- `row_names['income'][$category_id]` — 收入类别名称
- `row_names['cogs'][$category_id]` — COGS 类别名称
- `row_names['expense'][$category_id]` — 支出类别名称
- `row_values[$table][$category_id][$date] = 0` — 初始化为 0 的数值矩阵
- `row_tree_nodes[$table][$category_id]` — 树形层级结构

### 漏掉的过滤器元数据

| 过滤器 | 来源 |
|--------|------|
| `period`（周/月/季/年选项） | [AddPeriod#L25-L39](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddPeriod.php#L25-L39) |
| `basis`（权责发生制/收付实现制选项） | [AddBasis#L24-L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddBasis.php#L24-L38) |
| `group` 选项 | [AddGroup#L23-L38](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddGroup.php#L23-L38) |
| `categories` 类别筛选 | [AddIncomeExpenseCategories#L25-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddIncomeExpenseCategories.php#L25-L36) |
| `date_range` 日期范围 | [AddDate](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddDate.php) |
| 搜索字符串功能 | [AddSearchString](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Report/AddSearchString.php) |

### 核心结论

**只读 `ProfitLoss.php` 只能看到报表的「骨架」（三大 table 定义、计算逻辑、gross_profit/net_profit 计算、array() 输出扩展），但看不到「血肉」——所有具体的行数据（各类别）、分组维度选项、过滤选项元数据都是通过事件订阅器动态注入的。** 这种设计使得：
1. 不同报表可以复用相同的监听器（如 AddPeriod 被 6 个报表共用）
2. 新增分组维度不需要修改报表类本身
3. 模块可以通过订阅事件扩展报表功能
