# UpdateDocument 完整更新路径分析

## 核心代码位置

主更新逻辑位于 [UpdateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php)

---

## 完整更新路径与入口关系

### 并列入口关系图

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                      HTTP 请求层                        │
                    └─────────────────────────────────────────────────────────┘
                                              │
          ┌───────────────────────────────────┼───────────────────────────────────┐
          │                                   │                                   │
          ▼                                   ▼                                   ▼
┌───────────────────┐           ┌───────────────────────┐           ┌─────────────────────────┐
│  Web 普通单据控制器│           │  Web 周期单据控制器    │           │    API 控制器             │
│  Sales/Invoices   │           │  Sales/RecurringInvoices│          │  Api/Document/Documents  │
│  Purchases/Bills  │           │  Purchases/RecurringBills│          │                          │
└─────────┬─────────┘           └───────────┬───────────┘           └────────────┬────────────┘
          │                                 │                                    │
          │  直接调用                         │  merge issued_at 后调用              │  直接调用
          │  UpdateDocument Job               │  UpdateDocument Job                   │  UpdateDocument Job
          │                                 │                                    │
          ▼                                 ▼                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          UpdateDocument Job (统一处理逻辑)                                │
│  ├─ authorize() - 权限校验 (contact_id 修改限制)                                          │
│  ├─ DocumentUpdating event                                                                │
│  ├─ 附件处理 (Web/API 条件差异)                                                            │
│  ├─ 删除 items/item_taxes/totals                                                          │
│  ├─ CreateDocumentItemsAndTotals - 重建明细                                                │
│  ├─ 计算 paid_amount 并设置 status (paid/partial)                                          │
│  ├─ 更新主模型                                                                            │
│  ├─ 同步 transactions 的 contact_id (仅未对账)                                             │
│  ├─ updateRecurring - 更新 recurring 设置                                                 │
│  └─ DocumentUpdated event                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 入口控制器详细分析

#### 1. Web 普通单据控制器

**销售发票** - [Invoices.php#L189-L214](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L189-L214)：
```php
public function update(Document $invoice, Request $request)
{
    $response = $this->ajaxDispatch(new UpdateDocument($invoice, $request));
    // ... 处理响应
}
```

**采购账单** - [Bills.php#L165-L184](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/Bills.php#L165-L184)：
```php
public function update(Document $bill, Request $request)
{
    $response = $this->ajaxDispatch(new UpdateDocument($bill, $request));
    // ... 处理响应
}
```

**特点**：直接将请求传递给 UpdateDocument，不做额外处理。

#### 2. Web 周期单据控制器

**周期发票** - [RecurringInvoices.php#L169-L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L169-L192)：
```php
public function update(Document $recurring_invoice, Request $request)
{
    $issue_at = Date::parse($request->get('recurring_started_at'))->format('Y-m-d');
    $request->merge(['issued_at' => $issue_at]);  // ⚠️ 关键：合并 issued_at
    $response = $this->ajaxDispatch(new UpdateDocument($recurring_invoice, $request));
    // ... 处理响应
}
```

**周期账单** - [RecurringBills.php#L169-L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L169-L192)：
```php
public function update(Document $recurring_bill, Request $request)
{
    $issue_at = Date::parse($request->get('recurring_started_at'))->format('Y-m-d');
    $request->merge(['issued_at' => $issue_at]);  // ⚠️ 关键：合并 issued_at
    $response = $this->ajaxDispatch(new UpdateDocument($recurring_bill, $request));
    // ... 处理响应
}
```

**特点**：在调用 UpdateDocument 之前，将 `recurring_started_at` 合并为 `issued_at`。

#### 3. API 控制器

[Documents.php#L95-L100](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L95-L100)：
```php
public function update(Document $document, Request $request)
{
    $document = $this->dispatch(new UpdateDocument($document, $request));
    return new Resource($document->fresh());
}
```

**特点**：直接调用 UpdateDocument，不做额外处理。

---

## 问题解答

### 1. Web/API 删除或替换 attachment 的条件差异

#### 完整处理流程

**核心代码**：[UpdateDocument.php#L37-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L37-L49)

```php
if ($this->request->file('attachment')) {
    // 场景1: 有新文件上传 → 先删后加（替换模式）
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
    foreach ($this->request->file('attachment') as $attachment) {
        $media = $this->getMedia($attachment, Str::plural($this->model->type));
        $this->model->attachMedia($media, 'attachment');
    }
} elseif ($this->request->isNotApi() && ! $this->request->file('attachment') && $this->model->attachment) {
    // 场景2: Web 端无新文件 → 直接删除
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
} elseif ($this->request->isApi() && $this->request->has('remove_attachment') && $this->model->attachment) {
    // 场景3: API 端传 remove_attachment → 直接删除
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

#### uploaded_attachment 保留旧附件机制

**前端组件** - [AkauntingDropzoneFileUpload.vue#L204-L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/components/AkauntingDropzoneFileUpload.vue#L204-L232)：
```javascript
if (self.attachments.length) {
    setTimeout(() => {
        self.attachments.forEach(async (attachment) => {
            var mockFile = {
                id: attachment.id,
                name: attachment.name,
                size: attachment.size,
                type: attachment.type,
                download: attachment.downloadPath,
                dropzone: 'edit',  // ⚠️ 关键标记
            };
            dropzone.emit("addedfile", mockFile);
            // ...
        });
    }, 100);
}
```

**Dropzone 中间件** - [Dropzone.php#L17-L76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/Dropzone.php#L17-L76)：
```php
foreach ($value as $index => $parameter) {
    if (is_array($parameter)) {
        if (! Arr::has($parameter, 'dropzone')) {
            $files[] = $parameter;  // 新上传的文件
            continue;
        }
    }
    // ...
    $multiple = true;
    $uploaded[] = $parameter;  // 旧附件（带 dropzone 标记）
}

if ($multiple && $uploaded) {
    $request->request->set('uploaded_' . $key, $uploaded);  // ⚠️ 设置 uploaded_attachment
    $request->request->set($key, $files);  // ⚠️ 只保留新文件
}
```

**deleteMediaModel 实现** - [Uploads.php#L55-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L55-L92)：
```php
public function deleteMediaModel($model, $parameter, $request = null)
{
    $medias = $model->$parameter;
    if (! $medias) { return; }

    $already_uploaded = [];

    if ($request && isset($request['uploaded_' . $parameter])) {
        $uploaded = ($multiple) ? $request['uploaded_' . $parameter] : [$request['uploaded_' . $parameter]];

        // ⚠️ 关键：如果旧附件数量与 uploaded_attachment 数量相同，直接返回不删除
        if (count($medias) == count($uploaded)) {
            return;
        }

        foreach ($uploaded as $old_media) {
            $already_uploaded[] = $old_media['id'];  // 记录要保留的旧附件 ID
        }
    }

    foreach ((array) $medias as $media) {
        if (in_array($media->id, $already_uploaded)) {
            continue;  // ⚠️ 跳过要保留的旧附件
        }
        MediaModel::where('id', $media->id)->delete();
    }
}
```

#### 差异总结

| 场景 | Web 端 (isNotApi) | API 端 (isApi) |
|------|------------------|---------------|
| **上传新文件** | `file('attachment')` 不为空 → 先删后加，但 `uploaded_attachment` 会保留未修改的旧附件 | `file('attachment')` 不为空 → 先删后加，API 端通常不设置 `uploaded_attachment`，所以会删除所有旧附件 |
| **未上传文件** | `!file('attachment')` → 直接删除，但 `uploaded_attachment` 可能保留全部旧附件 | `!file('attachment')` → 不删除（除非显式传 `remove_attachment`） |
| **删除触发** | 隐式：表单未上传新文件时触发，但实际删除受 `uploaded_attachment` 控制 | 显式：必须传 `remove_attachment` 参数 |

#### 边界场景分析

| 可达方式 | 场景 | 实际行为 |
|---------|------|---------|
| **Web UI** | 编辑时不修改附件直接保存 | `uploaded_attachment` 包含所有旧附件 → `count($medias) == count($uploaded)` → 不删除任何附件 |
| **Web UI** | 编辑时删除一个旧附件，不上传新文件 | `uploaded_attachment` 少了一个 → 数量不等 → 删除不在列表中的旧附件 |
| **Web UI** | 编辑时上传新文件 | `file('attachment')` 不为空 → 删除不在 `uploaded_attachment` 中的旧附件 → 添加新文件 |
| **API** | 不包含 attachment 字段 | 不删除任何附件 |
| **API** | 包含 `remove_attachment: true` | 删除所有附件 |
| **API** | 包含新的 attachment 文件 | 删除所有旧附件（无 `uploaded_attachment`）→ 添加新文件 |
| **Job 直接调用** | 取决于传入的 request | 取决于 request 中的字段设置 |

---

### 2. items/item_taxes/totals 为什么先删再重建

**代码位置**：
- 删除：[UpdateDocument.php#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L51)
- 重建：[UpdateDocument.php#L53](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L53)

```php
$this->deleteRelationships($this->model, ['items', 'item_taxes', 'totals'], true);
$this->dispatch(new CreateDocumentItemsAndTotals($this->model, $this->request));
```

**删除实现** - [Relationships.php#L41-L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php#L41-L69)：
- 遍历每个关系，逐个调用 `forceDelete()`（第3个参数为 `true`）
- 会触发每个模型的删除事件和观察者

**重建实现** - [CreateDocumentItemsAndTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php)：
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

```php
$this->model->paid_amount = $this->model->paid;  // ⚠️ 从访问器获取已付款金额

event(new PaidAmountCalculated($this->model));

if ($this->model->paid_amount > 0) {
    if ($this->request['amount'] == $this->model->paid_amount) {
        $this->request['status'] = 'paid';      // 全额付款
    }

    if ($this->request['amount'] > $this->model->paid_amount) {
        $this->request['status'] = 'partial';   // 部分付款
    }
}

unset($this->model->reconciled);
unset($this->model->paid_amount);
$this->model->update($this->request->all());  // ⚠️ 用修改后的 request 更新模型
```

#### paid 访问器行为分析

**代码位置**：[Document.php#L373-L407](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407)

```php
public function getPaidAttribute()
{
    if (empty($this->amount)) {
        return false;
    }

    // ⚠️ 关键：如果原状态已经是 paid，直接返回 amount
    if ($this->status == 'paid') {
        return $this->amount;
    }

    // 否则遍历 transactions 计算实际付款
    $paid = 0;
    // ... 货币转换后求和
    return round($paid, $precision);
}
```

#### 边界场景分析

| 原状态 | 条件 | 结果 status | 说明 |
|-------|------|------------|------|
| draft/sent/... | `amount == paid_amount` | `paid` | 正常全额付款 |
| draft/sent/... | `amount > paid_amount` | `partial` | 部分付款 |
| draft/sent/... | `amount < paid_amount` | 不变 | ⚠️ **不修改 status**，但付款超过金额 |
| paid | `amount == paid_amount` | `paid` | 访问器直接返回 amount，所以必然相等 |
| paid | `amount != paid_amount` | `paid` | 访问器直接返回 amount，所以 `paid_amount == amount` |
| partial | `amount == paid_amount` | `paid` | 付款增加到全额 |
| partial | `amount > paid_amount` | `partial` | 仍然部分付款 |
| 任何状态 | `paid_amount == 0` | 不变 | ⚠️ **不进入 if 块**，不修改 status |

**关键注意**：
- 这里是直接修改 `$this->request['status']`，然后用 `$this->request->all()` 更新模型
- 优先级高于用户在 request 中传入的 status
- 只有 `paid_amount > 0` 时才会自动设置，付款为 0 时不修改 status
- 当原状态为 `paid` 时，`getPaidAttribute()` 直接返回 `$this->amount`，所以 `paid_amount` 必然等于 `amount`

---

### 4. 修改 contact_id 在哪些 status 或 reconciled 场景会被拒绝

**代码位置**：[UpdateDocument.php#L92-L112](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L92-L112)

```php
public function authorize(): void
{
    if (
        isset($this->request['contact_id']) &&
        (int) $this->request['contact_id'] !== (int) $this->model->contact_id
    ) {
        $lockedStatuses = ['sent', 'received', 'viewed', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled'];

        if (in_array($this->model->status, $lockedStatuses)) {
            // 场景1: 状态锁定
            throw new \Exception(trans('messages.warning.contact_change', ...));
        } else if ($this->model->transactions()->isReconciled()->exists()) {
            // 场景2: 存在已对账的交易
            throw new \Exception(trans('messages.warning.reconciled_doc', ...));
        }
    }
}
```

#### 可用状态列表

**代码位置**：[Documents.php#L69-L107](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php#L69-L107)

```php
public function getDocumentStatuses(string $type): Collection
{
    $list = [
        'invoice' => [
            'draft', 'sent', 'viewed', 'approved', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled',
        ],
        'bill'    => [
            'draft', 'received', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled',
        ],
    ];
    // ...
}
```

#### 拒绝场景总结

| 场景 | 条件 | 错误信息 |
|------|------|---------|
| **状态锁定 (Invoice)** | status ∈ `['sent', 'viewed', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled']` | "Cannot change contact of a document with status: xxx" |
| **状态锁定 (Bill)** | status ∈ `['received', 'partial', 'paid', 'overdue', 'unpaid', 'cancelled']` | "Cannot change contact of a document with status: xxx" |
| **交易已对账** | 任意 transaction 的 `reconciled = 1` | "Cannot change contact of a reconciled document" |

#### 允许修改的状态

| 单据类型 | 允许修改 contact_id 的状态 |
|---------|--------------------------|
| **Invoice** | `draft`, `approved` ⚠️ |
| **Bill** | `draft` |

**关于 approved 状态**：
- `approved` 在 `lockedStatuses` 中不存在，所以 Invoice 的 approved 状态允许修改 contact_id
- Bill 没有 approved 状态

#### isReconciled scope

**代码位置**：[Transaction.php#L307-L310](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L307-L310)

```php
public function scopeIsReconciled(Builder $query): Builder
{
    return $query->where('reconciled', 1);
}
```

#### 边界场景分析

| 可达方式 | 场景 | 结果 |
|---------|------|------|
| **Web UI** | draft 状态修改 contact | 允许（如果没有已对账交易） |
| **Web UI** | sent/status 修改 contact | 拒绝 |
| **Web UI** | draft 但有已对账交易 | 拒绝（第二个条件） |
| **API** | 同上述条件 | 同上述结果 |
| **Job 直接调用** | 同上述条件 | 同上述结果 |

**设计原因**：
- 已发送/查看/付款的单据在业务上已经生效，变更联系人会导致财务记录混乱
- 已对账的交易是已经和银行对账单核对过的历史记录，不允许修改

---

### 5. contact 改变后为什么只同步 reconciled=0 的 transactions

**代码位置**：[UpdateDocument.php#L75-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L75-L79)

```php
$originalContactId = $this->model->contact_id;

\DB::transaction(function () use ($originalContactId) {
    // ... 其他更新逻辑 ...

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

## 补充：Recurring 设置更新路径

### 完整更新流程

```
前端组件 (AkauntingRecurring.vue)
    │
    │ emit 事件: frequency, interval, custom_frequency, started, limit, limit_count, limit_date, send_email
    ▼
表单提交 (Blade 模板)
    │
    ▼
中间件验证 (Document.php FormRequest)
    │
    ├─ 验证 recurring_frequency, recurring_interval, recurring_custom_frequency
    ├─ 验证 recurring_started_at
    ├─ 验证 recurring_limit, recurring_limit_count, recurring_limit_date
    ▼
Web 控制器 (RecurringInvoices / RecurringBills)
    │
    │ merge(['issued_at' => Date::parse('recurring_started_at')->format('Y-m-d')])
    ▼
UpdateDocument Job
    │
    ▼
updateRecurring() (Recurring trait)
    ├─ frequency == 'no' 或为空 → delete()
    ├─ model 已存在 → update()
    └─ model 不存在 → create()
```

### 字段来源

**前端组件** - [AkauntingRecurring.vue](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/components/AkauntingRecurring.vue)：
- `frequency` - 频率选择（no/weekly/monthly/yearly/custom）
- `interval` - 自定义间隔（当 frequency == custom 时）
- `custom_frequency` - 自定义频率单位（daily/weekly/monthly/yearly）
- `started_at` - 开始日期
- `limit` - 限制类型（count/date）
- `limit_count` - 限制次数（当 limit == count 时）
- `limit_date` - 限制日期（当 limit == date 时）
- `send_email` - 是否自动发送邮件

### 表单验证

**代码位置**：[Document.php#L69-L88](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L69-L88)

```php
if ($this->request->has('recurring_frequency')) {
    if ($this->request->get('recurring_frequency') == 'custom') {
        $rules['recurring_interval'] = 'required|gte:1';
        $rules['recurring_custom_frequency'] = 'required|string|in:daily,weekly,monthly,yearly';
    }

    $rules['recurring_started_at'] = 'required|date_format:Y-m-d H:i:s';

    switch($this->request->get('recurring_limit')) {
        case 'date':
            $rules['recurring_limit_date'] = 'required|date_format:Y-m-d H:i:s|after_or_equal:recurring_started_at';
            break;
        case 'count':
            $rules['recurring_limit_count'] = 'required|gte:0';
            break;
    }
}
```

### issued_at 合并逻辑

**代码位置**：
- [RecurringInvoices.php#L171-L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L171-L173)
- [RecurringBills.php#L171-L173](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L171-L173)

```php
$issue_at = Date::parse($request->get('recurring_started_at'))->format('Y-m-d');
$request->merge(['issued_at' => $issue_at]);
```

**作用**：周期单据的 `issued_at` 由 `recurring_started_at` 派生，确保单据日期与周期开始日期一致。

### updateRecurring 实现

**代码位置**：[Recurring.php#L45-L91](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Recurring.php#L45-L91)

```php
public function updateRecurring($request)
{
    // 场景1: 关闭周期 → 删除 recurring 记录
    if (empty($request['recurring_frequency']) || ($request['recurring_frequency'] == 'no')) {
        $this->recurring()->delete();
        return;
    }

    // 计算字段值
    $frequency = ($request['recurring_frequency'] != 'custom') ? $request['recurring_frequency'] : $request['recurring_custom_frequency'];
    $interval = (($request['recurring_frequency'] != 'custom') || ($request['recurring_interval'] < 1)) ? 1 : (int) $request['recurring_interval'];
    $started_at = !empty($request['recurring_started_at']) ? $request['recurring_started_at'] : Date::now();
    $limit_by = !empty($request['recurring_limit']) ? $request['recurring_limit'] : 'count';
    $limit_count = isset($request['recurring_limit_count']) ? (int) $request['recurring_limit_count'] : 0;
    $limit_date = !empty($request['recurring_limit_date']) ? $request['recurring_limit_date'] : null;
    $auto_send = !empty($request['recurring_send_email']) ? $request['recurring_send_email'] : 0;

    $recurring = $this->recurring();
    $model_exists = $recurring->count();

    $data = [
        'company_id'    => $this->company_id,
        'frequency'     => $frequency,
        'interval'      => $interval,
        'started_at'    => $started_at,
        'limit_by'      => $limit_by,
        'limit_count'   => $limit_count,
        'limit_date'    => $limit_date,
        'auto_send'     => $auto_send,
    ];

    // ⚠️ 只有显式传入 recurring_status 时才更新状态
    if (! empty($request['recurring_status'])) {
        $data['status'] = $request['recurring_status'];
    }

    if ($model_exists) {
        $recurring->update($data);  // 场景2: 更新已存在的 recurring
    } else {
        // 场景3: 创建新的 recurring
        $recurring->create(array_merge($data, [
            'status'        => Model::ACTIVE_STATUS,  // 默认 active
            'created_from'  => $source,
            'created_by'    => $owner,
        ]));
    }
}
```

### end 动作

**代码位置**：
- [RecurringInvoices.php#L209-L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L209-L234)
- [RecurringBills.php#L209-L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L209-L234)

```php
public function end(Document $recurring_invoice)
{
    $response = $this->ajaxDispatch(new UpdateDocument($recurring_invoice, [
        'recurring_frequency' => $recurring_invoice->recurring->frequency,
        'recurring_interval' => $recurring_invoice->recurring->interval,
        'recurring_started_at' => $recurring_invoice->recurring->started_at,
        'recurring_limit' => $recurring_invoice->recurring->limit,
        'recurring_limit_count' => $recurring_invoice->recurring->limit_count,
        'recurring_limit_date' => $recurring_invoice->recurring->limit_date,
        'created_from' => $recurring_invoice->created_from,
        'created_by' => $recurring_invoice->created_by,
        'recurring_status' => Recurring::END_STATUS,  // ⚠️ 设置为 ended
    ]));
    // ...
}
```

**关键点**：end 动作不是删除 recurring，而是通过 `UpdateDocument` 更新 recurring 的 status 为 `ended`。

---

## 附件：完整调用链

### 入口控制器

| 类型 | 控制器 | 方法 | 代码位置 |
|-----|--------|------|---------|
| Web 销售发票 | Sales/Invoices | update | [L189-L214](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/Invoices.php#L189-L214) |
| Web 采购账单 | Purchases/Bills | update | [L165-L184](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/Bills.php#L165-L184) |
| Web 周期发票 | Sales/RecurringInvoices | update | [L169-L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L169-L192) |
| Web 周期账单 | Purchases/RecurringBills | update | [L169-L192](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L169-L192) |
| Web 周期发票 | Sales/RecurringInvoices | end | [L209-L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Sales/RecurringInvoices.php#L209-L234) |
| Web 周期账单 | Purchases/RecurringBills | end | [L209-L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Purchases/RecurringBills.php#L209-L234) |
| API | Api/Document/Documents | update | [L95-L100](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L95-L100) |

### 关联 Job

- [CreateDocumentItemsAndTotals.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CreateDocumentItemsAndTotals.php) - 创建明细和汇总
- [UpdateDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php) - 主更新 Job

### 关键 Trait

- [Relationships.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php) - 删除关联关系
- [Uploads.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php) - 处理附件上传/删除
- [Recurring.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Recurring.php) - 处理 recurring 设置
- [Documents.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Documents.php) - 文档状态定义等

### 关键中间件

- [Dropzone.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/Dropzone.php) - 处理附件上传请求

### 前端组件

- [AkauntingDropzoneFileUpload.vue](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/components/AkauntingDropzoneFileUpload.vue) - 附件上传组件
- [AkauntingRecurring.vue](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/components/AkauntingRecurring.vue) - 周期设置组件
