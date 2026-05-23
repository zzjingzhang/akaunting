# Document Cancel / Delete / Restore 关系处理对比分析

## 核心代码位置

| 操作 | Job 类 | 监听器 |
|------|--------|--------|
| Cancel | [CancelDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/CancelDocument.php) | [MarkDocumentCancelled.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentCancelled.php) |
| Delete | [DeleteDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/DeleteDocument.php) | 无（直接触发事件） |
| Restore | [RestoreDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/RestoreDocument.php) | [RestoreDocument.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/RestoreDocument.php) |

---

## 1. 有 reconciled transactions 时 cancel/delete 都失败的原因

### 检查逻辑

两个 Job 的 `authorize()` 方法完全相同：

```php
// CancelDocument.php L41-L49
// DeleteDocument.php L42-L50
public function authorize(): void
{
    if ($this->model->transactions()->isReconciled()->count()) {
        $type = Str::plural($this->model->type);
        $message = trans('messages.warning.reconciled_doc', ['type' => trans_choice("general.$type", 1)]);
        throw new \Exception($message);
    }
}
```

### isReconciled Scope 定义

[Transaction.php L307-L310](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L307-L310)

```php
public function scopeIsReconciled(Builder $query): Builder
{
    return $query->where('reconciled', 1);
}
```

### 设计意图

CancelDocument 类注释 L36-L40 明确说明了原因：

```
* Mirrors the same guard in DeleteDocument to prevent cancelling a document
* that has reconciled transactions, which would silently destroy bank records.
```

**原因总结**：已对账（reconciled）的交易代表银行对账单已匹配确认。如果允许取消或删除关联的文档，会级联删除这些交易记录，导致银行对账记录被静默破坏，产生财务数据不一致。

---

## 2. 三者对关系的处理差异

### Document 模型定义的关系

[Document.php L113-L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L113-L179) 定义了以下关系：
- `items` - 行项目
- `item_taxes` - 行项目税
- `histories` - 历史记录
- `transactions` / `payments` - 付款交易
- `recurring` - 重复账单配置
- `totals` - 金额汇总
- `contact`, `category`, `currency` 等（belongsTo 关系，不级联删除）

### deleteRelationships 实现

[Relationships.php L41-L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php#L41-L69)

```php
public function deleteRelationships($model, $relationships, $permanently = false): void
{
    event(new RelationshipDeleting($record));
    foreach ((array) $record->relationships as $relationship) {
        // 遍历每个关系项，调用 delete() 或 forceDelete()
        foreach ((array) $items as $item) {
            $item->$function();
        }
    }
}
```

**关键点**：`deleteRelationships` 会逐个调用关系项的 `delete()` 方法，这意味着：
1. 如果模型使用了 SoftDeletes，会执行软删除
2. 每个关系项的删除会触发其自身的 Model 事件（如 deleting、deleted）
3. 触发 `RelationshipDeleting` 全局事件

### 三者关系删除对比

| 关系 | CancelDocument | DeleteDocument | RestoreDocument |
|------|---------------|----------------|-----------------|
| `transactions`（付款） | ✅ 删除 | ✅ 删除 | ❌ 不恢复 |
| `recurring`（重复） | ✅ 删除 | ✅ 删除 | ❌ 不恢复 |
| `items`（行项目） | ❌ 保留 | ✅ 删除 | ❌ 不恢复 |
| `item_taxes`（税） | ❌ 保留 | ✅ 删除 | ❌ 不恢复 |
| `histories`（历史） | ❌ 保留 | ✅ 删除 | ❌ 不恢复 |
| `totals`（汇总） | ❌ 保留 | ✅ 删除 | ❌ 不恢复 |
| `status` 状态 | 设为 `cancelled` | 文档软删除 | 设为 `draft` |

### RestoreDocument 的行为

[RestoreDocument.php L19-L27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/RestoreDocument.php#L19-L27)

```php
public function handle(): Document
{
    \DB::transaction(function () {
        $this->model->status = 'draft';
        $this->model->save();
    });
    return $this->model;
}
```

**重要结论**：RestoreDocument **仅修改 status 字段为 draft，不恢复任何被删除的关系数据**。

---

## 3. DeleteDocument 为什么要 mute Transaction observer

### Transaction Observer 的 deleted 处理

[Transaction.php L23-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L23-L34)

```php
public function deleted(Model $transaction)
{
    if (! empty($transaction->document_id)) {
        $type = ($transaction->type == 'income') ? Document::INVOICE_TYPE : Document::BILL_TYPE;
        $this->updateDocument($transaction, $type);
    }
    // ...
}

protected function updateDocument($transaction, $type)
{
    $document = $transaction->{$type};
    $document->transactions_count = $document->transactions->count();
    event(new TransactionsCounted($document));
    // 更新 document 的 status 为 sent/received/partial
    $document->save();
    $this->dispatch(new CreateDocumentHistory($document, ...));
}
```

### mute 的原因

[DeleteDocument.php L20-L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/DeleteDocument.php#L20-L32)

```php
\DB::transaction(function () {
    try {
        Transaction::mute();
        $this->deleteRelationships($this->model, [
            'items', 'item_taxes', 'histories', 'transactions', 'recurring', 'totals'
        ]);
        $this->model->delete();
    } finally {
        Transaction::unmute();
    }
});
```

**原因分析**：

1. **避免无效回调**：当删除整个 Document 时，Transaction 的 deleted 观察者会尝试回查并更新 Document 的状态和统计。但 Document 本身即将被删除，这种更新是无意义的。

2. **防止数据库死锁/异常**：在同一个数据库事务中，如果删除 Transaction 时触发 Observer 去查询和更新 Document，而 Document 又即将被删除，可能导致查询异常或死锁。

3. **性能优化**：跳过不必要的事件触发（TransactionsCounted）、状态计算和历史记录创建。

4. **避免状态竞争**：Observer 会根据剩余交易数量将 Document 状态改为 `sent` / `received` / `partial`，但 DeleteDocument 正要删除整个 Document，会导致状态冲突。

---

## 4. 三者触发的事件对比

### 事件注册

[Event.php L61-L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L61-L66)

```php
\App\Events\Document\DocumentCancelled::class => [
    \App\Listeners\Document\MarkDocumentCancelled::class,
],
\App\Events\Document\DocumentRestored::class => [
    \App\Listeners\Document\RestoreDocument::class,
],
```

### 事件触发流程对比

| 操作 | 触发方式 | 触发事件 | 备注 |
|------|---------|---------|------|
| **Cancel** | 控制器触发事件 → Listener dispatch Job | `DocumentCancelled` | Job 内部不触发事件 |
| **Delete** | Job 内部直接触发 | `DocumentDeleting` → `DocumentDeleted` | 两个事件都触发 |
| **Restore** | 控制器触发事件 → Listener dispatch Job | `DocumentRestored` | Job 内部不触发事件 |

### 详细流程

#### CancelDocument 流程
```
Controller::markCancelled()
    → event(DocumentCancelled)
        → MarkDocumentCancelled listener
            → dispatch(CancelDocument Job)
                → deleteRelationships(transactions, recurring)
                → 触发 RelationshipDeleting 事件
                → 每个 Transaction 删除时触发其自身的 deleted 事件
                → document.status = 'cancelled'
```

#### DeleteDocument 流程
```
直接调用 DeleteDocument Job
    → event(DocumentDeleting)
    → Transaction::mute()
    → deleteRelationships(items, item_taxes, histories, transactions, recurring, totals)
        → 触发 RelationshipDeleting 事件
        → Transaction 的 deleted observer 被 mute，不执行
    → document->delete()
    → event(DocumentDeleted)
```

#### RestoreDocument 流程
```
Controller 触发 event(DocumentRestored)
    → RestoreDocument listener
        → dispatch(RestoreDocument Job)
            → document.status = 'draft'
            → document.save()
```

---

## 5. 取消后再恢复仍丢失付款关系的反例

### 问题根源

CancelDocument 删除了 `transactions`（付款）关系，而 RestoreDocument **只恢复 status，不恢复任何关系数据**。

### 反例构造

```php
// 场景：一张已付款的发票，先取消再恢复

// 1. 创建一张 $1000 的发票并标记已付款
$invoice = Document::factory()->create([
    'type' => Document::INVOICE_TYPE,
    'status' => 'paid',
    'amount' => 1000,
]);

// 创建付款交易
$transaction = $invoice->transactions()->create([
    'company_id' => $invoice->company_id,
    'type' => 'income',
    'amount' => 1000,
    'account_id' => 1,
    'paid_at' => now(),
    'reconciled' => 0, // 未对账，允许取消
]);

// 此时：$invoice->transactions()->count() = 1
// 状态：paid

// 2. 取消发票
event(new DocumentCancelled($invoice));
// CancelDocument Job 执行：
// - deleteRelationships 删除 transactions 和 recurring
// - status 改为 cancelled

// 此时：$invoice->transactions()->count() = 0  ← 付款记录被删除
// 状态：cancelled

// 3. 恢复发票
event(new DocumentRestored($invoice));
// RestoreDocument Job 执行：
// - 仅将 status 改为 draft
// - 不恢复 transactions

// 此时：$invoice->transactions()->count() = 0  ← 付款记录永久丢失！
// 状态：draft
// 金额：仍为 1000，但已无对应的付款记录
```

### 数据不一致表现

| 阶段 | 状态 | transactions 数量 | 金额 | 财务一致性 |
|------|------|------------------|------|-----------|
| 初始 | paid | 1 | 1000 | ✅ 一致 |
| 取消后 | cancelled | 0 | 1000 | ⚠️ 付款被删除 |
| 恢复后 | draft | 0 | 1000 | ❌ 不一致：金额 1000 但无付款记录 |

### 业务影响

1. **财务数据丢失**：实际已收到的付款记录被永久删除，无法追溯
2. **需重新录入付款**：恢复后的发票金额仍在，但付款记录需要财务人员重新手工录入
3. **审计线索断裂**：原付款的支付方式、日期、凭证号等信息全部丢失
4. **可能重复收款**：恢复后的发票状态为 draft，可能被再次发送给客户请求付款

---

## 总结对比表

| 维度 | CancelDocument | DeleteDocument | RestoreDocument |
|------|---------------|----------------|-----------------|
| **reconciled 校验** | ✅ 检查，有则失败 | ✅ 检查，有则失败 | ❌ 不检查 |
| **删除的关系** | transactions, recurring | items, item_taxes, histories, transactions, recurring, totals | 无 |
| **恢复关系** | - | - | ❌ 不恢复任何关系 |
| **mute Observer** | ❌ 不 mute | ✅ mute Transaction | - |
| **Document 本身** | status = cancelled | 软删除 | status = draft |
| **触发事件** | DocumentCancelled（外部触发） | DocumentDeleting, DocumentDeleted | DocumentRestored（外部触发） |
| **可逆性** | 可通过 Restore 恢复状态 | 可软恢复但数据永久丢失 | 仅恢复状态字段 |
| **付款关系丢失风险** | ⚠️ 取消后无法恢复 | ⚠️ 永久删除 | ❌ 不主动删除但也不恢复 |
