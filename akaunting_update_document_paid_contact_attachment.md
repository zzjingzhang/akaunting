# UpdateDocument 完整更新路径分析

## 核心代码位置

主更新逻辑位于 [UpdateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php)

## 完整更新路径

```
Web 控制器 (Invoices.php / Bills.php)
        ↓
API 控制器 (Documents.php)
        ↓
UpdateDocument Job
        ├─ authorize() - 权限校验
        ├─ DocumentUpdating event
        ├─ 附件处理 (删除/替换)
        ├─ 删除 items/item_taxes/totals
        ├─ CreateDocumentItemsAndTotals - 重建明细
        ├─ 计算 paid_amount 并设置 status (paid/partial)
        ├─ 更新主模型
        ├─ 同步 transactions 的 contact_id
        ├─ updateRecurring - 更新 recurring 设置
        └─ DocumentUpdated event
```

---

## 问题解答

### 1. Web/API 删除或替换 attachment 的条件差异

**代码位置**：[UpdateDocument.php#L37-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L37-L49)

**核心逻辑**：

```php
// 场景1: 上传了新文件 → 先删后加（替换）
if ($this->request->file('attachment')) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
    foreach ($this->request->file('attachment') as $attachment) {
        $media = $this->getMedia($attachment, Str::plural($this->model->type));
        $this->model->attachMedia($media, 'attachment');
    }
}
// 场景2: Web端未上传文件 → 直接删除
elseif ($this->request->isNotApi() && ! $this->request->file('attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
// 场景3: API端传 remove_attachment → 直接删除
elseif ($this->request->isApi() && $this->request->has('remove_attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

**差异总结**：

| 场景 | Web 端 (isNotApi) | API 端 (isApi) |
|------|------------------|---------------|
| **上传新文件** | 先删除所有旧附件，再添加新附件 | 先删除所有旧附件，再添加新附件 |
| **未上传文件** | 直接删除现有附件（表单提交时若文件字段为空视为删除） | 不删除，除非显式传 `remove_attachment` 参数 |
| **删除触发** | 隐式：表单未上传文件即删除 | 显式：必须传 `remove_attachment` 参数 |

**设计原因**：Web 端表单提交时，如果用户清空了文件选择框，后端无法区分是"清空"还是"未修改"，所以默认视为删除；而 API 端需要明确的语义，避免误删。

---

### 2. items/item_taxes/totals 为什么先删再重建

**代码位置**：
- 删除：[UpdateDocument.php#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L51)
- 重建：[UpdateDocument.php#L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L53)

**核心逻辑**：

```php
$this->deleteRelationships($this->model, ['items', 'item_taxes', 'totals'], true);
$this->dispatch(new CreateDocumentItemsAndTotals($this->model, $this->request));
```

**删除实现**（[Relationships.php#L41-L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php#L41-L69)）：
- 遍历每个关系，逐个调用 `forceDelete()`（第3个参数为 `true`）
- 会触发每个模型的删除事件和观察者

**重建实现**（[CreateDocumentItemsAndTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)）：
- 创建 items（包含每个 item 的 item_taxes）
- 计算 sub_total、discount、taxes
- 创建 totals 记录（sub_total, discount, tax, extra, total）

**设计原因**：

1. **数据复杂性**：items 与 item_taxes 是层级关系，totals 依赖 items 的计算结果，增量更新需要处理新增、修改、删除三种情况，逻辑复杂
2. **避免数据不一致**：删除重建可以确保所有关联数据完全符合当前 request 的状态，不会有残留的旧数据
3. **代码复用**：与创建时复用同一套 `CreateDocumentItemsAndTotals` 逻辑，保证计算规则一致
4. **事件触发**：每个模型的删除/创建事件会被正确触发，保证审计日志和缓存等同步更新

---

### 3. 已有付款时 request['status'] 如何变成 paid 或 partial

**代码位置**：[UpdateDocument.php#L55-L67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L55-L67)

**核心逻辑**：

```php
// 1. 从访问器获取已付款金额（遍历 transactions 求和）
$this->model->paid_amount = $this->model->paid;

event(new PaidAmountCalculated($this->model));

// 2. 根据已付款金额自动设置状态
if ($this->model->paid_amount > 0) {
    // 全额付款 → paid
    if ($this->request['amount'] == $this->model->paid_amount) {
        $this->request['status'] = 'paid';
    }
    // 部分付款 → partial
    if ($this->request['amount'] > $this->model->paid_amount) {
        $this->request['status'] = 'partial';
    }
}

// 3. 清除临时属性后更新模型
unset($this->model->reconciled);
unset($this->model->paid_amount);
$this->model->update($this->request->all());
```

**`paid` 访问器实现**（[Document.php#L373-L407](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407)）：
- 遍历所有关联的 transactions
- 处理货币转换（如果交易货币与单据货币不同）
- 求和并按货币精度四舍五入
- 如果 status 已经是 'paid'，直接返回 amount（性能优化）

**关键注意**：
- 这里是直接修改 `$this->request['status']`，然后用 `$this->request->all()` 更新模型
- 优先级高于用户在 request 中传入的 status
- 只有 `paid_amount > 0` 时才会自动设置，付款为 0 时不修改 status

---

### 4. 修改 contact_id 在哪些 status 或 reconciled 场景会被拒绝

**代码位置**：[UpdateDocument.php#L92-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L92-L112)

**核心逻辑**：

```php
public function authorize(): void
{
    if (
        isset($this->request['contact_id']) &&
        (int) $this->request['contact_id'] !== (int) $this->model->contact_id
    ) {
        // 场景1: 状态在锁定列表中
        $lockedStatuses = ['sent', 'received', 'viewed', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled'];
        if (in_array($this->model->status, $lockedStatuses)) {
            throw new \Exception(trans('messages.warning.contact_change', ...));
        }
        // 场景2: 存在已对账的交易
        else if ($this->model->transactions()->isReconciled()->exists()) {
            throw new \Exception(trans('messages.warning.reconciled_doc', ...));
        }
    }
}
```

**`isReconciled` scope**（[Transaction.php#L307-L310](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L307-L310)）：
```php
public function scopeIsReconciled(Builder $query): Builder
{
    return $query->where('reconciled', 1);
}
```

**拒绝场景总结**：

| 场景 | 条件 | 错误信息 |
|------|------|---------|
| **状态锁定** | status ∈ `['sent', 'received', 'viewed', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled']` | "Cannot change contact of a document with status: xxx" |
| **交易已对账** | 任意 transaction 的 `reconciled = 1` | "Cannot change contact of a reconciled document" |

**允许修改的状态**：只有 `draft` 状态下可以修改联系人，且没有已对账的交易记录。

**设计原因**：
- 已发送/查看/付款的单据在业务上已经生效，变更联系人会导致财务记录混乱
- 已对账的交易是已经和银行对账单核对过的历史记录，不允许修改

---

### 5. contact 改变后为什么只同步 reconciled=0 的 transactions

**代码位置**：[UpdateDocument.php#L75-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L75-L79)

**核心逻辑**：

```php
// 记录原始 contact_id
$originalContactId = $this->model->contact_id;

\DB::transaction(function () use ($originalContactId) {
    // ... 其他更新逻辑 ...

    // 只更新未对账的交易
    if (isset($this->request['contact_id']) && $originalContactId != $this->request['contact_id']) {
        $this->model->transactions()->where('reconciled', 0)->update([
            'contact_id' => $this->request['contact_id'],
        ]);
    }
});
```

**设计原因**：

1. **财务数据完整性**：已对账（`reconciled = 1`）的交易代表已经与银行对账单核对无误，属于"已确认"的历史财务记录。修改这些记录会破坏审计追踪。

2. **对账一致性**：银行对账单上的交易对手（contact）是固定的历史事实，如果修改已对账交易的 contact，会导致系统记录与银行记录不一致。

3. **实际业务场景**：只有未对账的交易可能存在录入错误需要修正，已对账的交易理论上不应该再修改。

4. **与 authorize 校验的关系**：authorize 已经确保只要有任何一笔交易已对账，就不允许修改 contact_id。所以这里的 `where('reconciled', 0)` 是双重保护：
   - 第一层：authorize 阻止"存在已对账交易"时修改 contact
   - 第二层：即使通过了 authorize（没有已对账交易），更新时也只更新未对账的（全部都是未对账的）

---

## 附件：完整调用链

### 入口控制器

- **Web 端（销售发票）**：[Invoices.php@update](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L189-L214)
- **Web 端（采购账单）**：[Bills.php@update](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/Bills.php#L165-L184)
- **API 端**：[Documents.php@update](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L95-L100)

### 关联 Job

- [CreateDocumentItemsAndTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) - 创建明细和汇总
- [UpdateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php) - 主更新 Job

### 关键 Trait

- [Relationships.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php) - 删除关联关系
- [Uploads.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php) - 处理附件上传/删除
- [Recurring.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Recurring.php) - 处理 recurring 设置
