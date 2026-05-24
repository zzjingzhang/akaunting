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

### Document 模型定义的全部关系

[Document.php L113-L179](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L113-L179)

```php
// belongsTo（不级联）
category()   // L113
contact()    // L123
currency()   // L128
parent()     // L156 — belongsTo Document, filtered by isRecurring

// hasMany（可级联删除）
items()        // L133
item_taxes()   // L138
histories()    // L143
children()     // L118 — hasMany Document, 子文档
transactions() // L176 / payments() L162
totals()       // L172

// morphOne（可级联删除）
recurring() // L166

// media（通过 Media trait，morphToMany）
media() // app/Traits/Media.php L24
```

**三个操作的关系删除列表均不包含 `children`、`parent`、`media`**。

### deleteRelationships 实现与 SoftDeletes

[Relationships.php L41-L69](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Relationships.php#L41-L69)

```php
public function deleteRelationships($model, $relationships, $permanently = false): void
{
    event(new RelationshipDeleting($record));
    foreach ((array) $record->relationships as $relationship) {
        // ...
        $function = $permanently ? 'forceDelete' : 'delete';
        foreach ((array) $items as $item) {
            $item->$function();
        }
    }
}
```

**关键**：`$permanently` 默认为 `false`，因此调用的是 `$item->delete()`。

[Abstracts/Model.php L17-L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L17-L23)

```php
use Illuminate\Database\Eloquent\SoftDeletes;

abstract class Model extends Eloquent implements Ownable
{
    use ..., SoftDeletes, ...;
```

Document 和 Transaction 均继承自 `App\Abstracts\Model`，均具备 `SoftDeletes`。因此 `delete()` 执行的是**软删除**（设置 `deleted_at` 字段），普通查询（不含 `withTrashed`）中这些关系会消失。

### 测试验证：取消后 transaction 为软删除

[CancelDocumentTest.php L62-L74](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/tests/Feature/Document/CancelDocumentTest.php#L62-L74)

```php
public function testItShouldDeleteTransactionsOnCancelSuccess(): void
{
    $bill = $this->createBill();
    $transaction = Transaction::factory()->expense()->create([
        'document_id' => $bill->id,
        'reconciled'  => 0,
    ]);
    $this->dispatch(new CancelDocument($bill));
    $this->assertSoftDeleted('transactions', ['id' => $transaction->id]);
}
```

### 三者关系删除对比

| 关系 | CancelDocument | DeleteDocument | RestoreDocument |
|------|---------------|----------------|-----------------|
| `transactions`（付款） | ✅ 软删除 | ✅ 软删除 | ❌ 不恢复 |
| `recurring`（重复） | ✅ 软删除 | ✅ 软删除 | ❌ 不恢复 |
| `items`（行项目） | ❌ 保留 | ✅ 软删除 | ❌ 不恢复 |
| `item_taxes`（税） | ❌ 保留 | ✅ 软删除 | ❌ 不恢复 |
| `histories`（历史） | ❌ 保留 | ✅ 软删除 | ❌ 不恢复 |
| `totals`（汇总） | ❌ 保留 | ✅ 软删除 | ❌ 不恢复 |
| `children`（子文档） | ❌ 保留 | ❌ 保留 | ❌ 保留 |
| `parent`（父文档） | ❌ 保留 | ❌ 保留 | ❌ 保留 |
| `media`（附件） | ❌ 保留 | ❌ 保留 | ❌ 保留 |
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

**结论**：RestoreDocument 仅修改 status 字段为 draft，不调用 `restore()` 或 `withTrashed()`，不恢复任何被软删除的关系数据。

---

## 3. DeleteDocument 为什么要 mute Transaction observer

### Transaction Observer 的 deleted 处理

[Transaction.php L23-L59](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L23-L59)

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
    if (empty($document)) { return; }

    $document->transactions_count = $document->transactions->count();
    event(new TransactionsCounted($document));    // ① 触发事件
    $document->status = ($type == Document::INVOICE_TYPE) ? 'sent' : 'received'; // ② 重新计算状态
    if ($document->transactions_count > 0) {
        $document->status = 'partial';
    }
    $document->save();                            // ③ 保存即将删除的 document
    $this->dispatch(new CreateDocumentHistory($document, 0, $this->getDescription($transaction))); // ④ 创建历史
}
```

### mute 的源码可证原因

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

**源码可证的行为**：mute 避免了每个被删除的 transaction 触发以下无效操作：

1. **重新计算并保存即将删除的 document 状态** — Transaction observer 的 `updateDocument()` 会根据剩余交易数量将 document.status 设为 `sent` / `received` / `partial`，然后 `$document->save()`。但 DeleteDocument 紧接着要删除这个 document，这些计算和保存毫无意义。

2. **触发 `TransactionsCounted` 事件** — 每个被删除的 transaction 都会触发此事件，但 document 即将被删除，没有监听者需要此事件。

3. **创建付款删除历史记录** — Transaction observer 会 dispatch `CreateDocumentHistory` 为每个被删除的付款创建历史记录。但 DeleteDocument 同时在删除 `histories` 关系，这些历史记录随即被删除。

> **推测（源码未直接证明）**：mute 也可能防止同一事务内 Transaction observer 回查并更新即将删除的 Document 导致的数据库死锁或查询异常，但源码中没有注释或直接证据支持此结论。

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

### 事件分层说明

| 事件类型 | 说明 | 涉及操作 |
|---------|------|---------|
| **外部业务事件** | 由控制器或外部代码触发，用于启动操作流程 | `DocumentCancelled`, `DocumentRestored` |
| **Job 内部业务事件** | Job 在 handle() 内直接触发 | `DocumentDeleting`, `DocumentDeleted` |
| **关系删除事件** | `deleteRelationships()` 统一触发 | `RelationshipDeleting` |
| **模型 deleted observer** | 每个关系项 `$item->delete()` 触发的 Eloquent 模型事件 | Transaction observer 的 `deleted()` |
| **历史记录创建** | 监听器和 observer 副作用派发的 Job | `CreateDocumentHistory` |

### 事件触发流程对比

| 操作 | 触发方式 | 触发事件 | 备注 |
|------|---------|---------|------|
| **Cancel** | 控制器触发外部事件 → Listener dispatch Job | `DocumentCancelled`（外部） | Job 内部不触发业务事件 |
| **Delete** | Job 内部直接触发 | `DocumentDeleting` → `DocumentDeleted` | 两个事件都在 Job 内 |
| **Restore** | 控制器触发外部事件 → Listener dispatch Job | `DocumentRestored`（外部） | Job 内部不触发业务事件 |

### 详细流程

#### CancelDocument 流程（成功路径）
```
Controller::markCancelled()
    → event(DocumentCancelled)                    [外部业务事件]
        → MarkDocumentCancelled listener
            → dispatch(CancelDocument Job)
                → authorize() 成功
                → DB::transaction begin
                    → deleteRelationships(transactions, recurring)
                        → event(RelationshipDeleting)           [关系删除事件]
                        → 每个 transaction->delete()
                            → Transaction observer deleted()     [模型 observer]
                                → 重新计算 document.status
                                → event(TransactionsCounted)      [事件]
                                → $document->save()
                                → dispatch(CreateDocumentHistory) [历史记录]
                    → $document->status = 'cancelled'
                    → $document->save()
                → DB::transaction commit
            → dispatch(CreateDocumentHistory, 'marked_cancelled') [历史记录]
```

#### CancelDocument 流程（authorize 失败路径）
```
Controller::markCancelled()
    → event(DocumentCancelled)                    [外部业务事件已触发]
        → MarkDocumentCancelled listener
            → dispatch(CancelDocument Job)
                → authorize() 失败 → throw Exception
            → 不再执行后续 CreateDocumentHistory
    ⚠️ DocumentCancelled 事件已发出，但 CancelDocument 失败
```

#### 直接 dispatch CancelDocument（绕过控制器）
```
直接 dispatch(new CancelDocument($document))
    → authorize() 检查
    → 无外部 DocumentCancelled 事件
    → 成功或失败均不触发 DocumentCancelled
```

#### DeleteDocument 流程（成功路径）
```
直接调用 DeleteDocument Job
    → authorize() 成功
    → event(DocumentDeleting)                     [Job 内部业务事件]
    → DB::transaction begin
        → Transaction::mute()                      [mute observer]
        → deleteRelationships(items, item_taxes, histories, transactions, recurring, totals)
            → event(RelationshipDeleting)         [关系删除事件]
            → 每个关系项->delete()
            → Transaction observer 被 mute，不执行
        → $document->delete()                     [软删除]
    → Transaction::unmute()
    → event(DocumentDeleted)                      [Job 内部业务事件]
```

#### DeleteDocument 流程（authorize 失败路径）
```
直接调用 DeleteDocument Job
    → authorize() 失败 → throw Exception
    → 不触发 DocumentDeleting
    → 不触发 DocumentDeleted
    → 不触发 RelationshipDeleting
```

#### RestoreDocument 流程
```
Controller::restoreInvoice/restoreBill()
    → event(DocumentRestored)                     [外部业务事件]
        → RestoreDocument listener
            → dispatch(RestoreDocument Job)
                → DB::transaction
                    → $document->status = 'draft'
                    → $document->save()
                → 不恢复任何软删除关系
            → dispatch(CreateDocumentHistory, 'restored') [历史记录]
```

### 事件触发完整对比表

| 事件 | Cancel（成功） | Cancel（失败） | Cancel（直接dispatch） | Delete（成功） | Delete（失败） | Restore |
|------|--------------|--------------|---------------------|--------------|--------------|---------|
| `DocumentCancelled` | ✅ 外部触发 | ✅ 外部已触发 | ❌ | ❌ | ❌ | ❌ |
| `DocumentRestored` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ 外部触发 |
| `DocumentDeleting` | ❌ | ❌ | ❌ | ✅ Job 内 | ❌ | ❌ |
| `DocumentDeleted` | ❌ | ❌ | ❌ | ✅ Job 内 | ❌ | ❌ |
| `RelationshipDeleting` | ✅ | ❌ | ✅ 若authorize成功 | ✅ | ❌ | ❌ |
| Transaction observer `deleted` | ✅ 每个transaction | ❌ | ✅ 若authorize成功 | ❌ (muted) | ❌ | ❌ |
| `TransactionsCounted` | ✅ 每个transaction | ❌ | ✅ 若authorize成功 | ❌ (muted) | ❌ | ❌ |
| CreateDocumentHistory（付款删除） | ✅ 每个transaction | ❌ | ✅ 若authorize成功 | ❌ (muted) | ❌ | ❌ |
| CreateDocumentHistory（标记取消/恢复） | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 5. 取消后再恢复仍丢失付款关系的反例

### 问题根源

CancelDocument 通过 `deleteRelationships()` 调用 `$transaction->delete()` 执行软删除。由于 Document 和 Transaction 继承的 [Abstracts/Model.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Model.php#L17-L23) 使用了 `SoftDeletes`，交易记录被软删除（`deleted_at` 被设置），在普通查询中不可见。RestoreDocument 仅将 `document.status` 改为 `draft`，不调用 `restore()` 或 `withTrashed()`，因此不恢复任何软删除关系。

### 可达条件

1. 存在至少一条 `reconciled = 0` 的 transaction 关联到 document
2. 取消操作 authorize() 通过，`deleteRelationships` 软删除所有 transactions
3. 恢复操作仅修改 status，不恢复软删除的 transactions

### 反例构造

```php
// 场景：一张已付款的发票，先取消再恢复

// 1. 创建一张 $1000 的发票（未对账付款）
$invoice = Document::factory()->create([
    'type' => Document::INVOICE_TYPE,
    'status' => 'paid',
    'amount' => 1000,
]);

// 创建一条未对账的付款交易
$transaction = Transaction::factory()->income()->create([
    'document_id' => $invoice->id,
    'type' => 'income',
    'amount' => 1000,
    'account_id' => 1,
    'paid_at' => now(),
    'reconciled' => 0, // 未对账，允许取消
]);

// 此时：$invoice->transactions()->count() = 1
// 状态：paid

// 2. 取消发票（通过控制器路径）
event(new DocumentCancelled($invoice));
// CancelDocument Job 执行：
// - deleteRelationships(['transactions', 'recurring'])
// - 每个 $transaction->delete() → 软删除，deleted_at 被设置
// - $document->status = 'cancelled'

// 此时：$invoice->transactions()->count() = 0  ← 软删除，普通查询不可见
// $invoice->transactions()->withTrashed()->count() = 1  ← 数据库中仍存在
// 状态：cancelled

// 3. 恢复发票
event(new DocumentRestored($invoice));
// RestoreDocument Job 执行：
// - 仅将 $document->status = 'draft' 并保存
// - 不调用 restore() 或 withTrashed()
// - 不恢复 transactions

// 此时：$invoice->transactions()->count() = 0  ← 付款关系在普通查询中仍不可见
// $invoice->transactions()->withTrashed()->count() = 1  ← 数据仍在数据库中
// 状态：draft
// 金额：仍为 1000，但付款关系在普通查询中不可见
```

### 数据不一致表现

| 阶段 | 状态 | transactions() 数量 | withTrashed 数量 | 金额 | 财务一致性 |
|------|------|--------------------|-----------------|------|-----------|
| 初始 | paid | 1 | 1 | 1000 | ✅ 一致 |
| 取消后 | cancelled | 0 | 1 | 1000 | ⚠️ 付款被软删除 |
| 恢复后 | draft | 0 | 1 | 1000 | ❌ 不一致：金额 1000 但付款关系不可见 |

### 业务影响

1. **付款关系查询不可见**：恢复后的发票在业务界面上看不到付款记录，但数据实际仍在数据库中（`deleted_at` 非空）
2. **需重新录入或手动恢复**：财务人员需在付款模块中手动恢复软删除记录，或重新录入付款
3. **审计线索**：软删除保留了 `deleted_at` 时间戳，审计上可追溯，但业务层不可见
4. **可能重复收款**：恢复后的发票状态为 draft，可能被再次发送给客户，而客户实际上已付款

---

## 总结对比表

| 维度 | CancelDocument | DeleteDocument | RestoreDocument |
|------|---------------|----------------|-----------------|
| **reconciled 校验** | ✅ 检查，有则失败 | ✅ 检查，有则失败 | ❌ 不检查 |
| **删除的关系** | transactions, recurring | items, item_taxes, histories, transactions, recurring, totals | 无 |
| **删除方式** | 软删除（SoftDeletes） | 软删除（SoftDeletes） | 不删除 |
| **恢复关系** | - | - | ❌ 不恢复任何软删除关系 |
| **未处理的关系** | children, parent, media | children, parent, media | children, parent, media |
| **mute Observer** | ❌ 不 mute | ✅ mute Transaction | - |
| **Document 本身** | status = cancelled | 软删除 | status = draft |
| **外部业务事件** | DocumentCancelled（控制器触发） | 无 | DocumentRestored（控制器触发） |
| **Job 内部事件** | 无 | DocumentDeleting, DocumentDeleted | 无 |
| **关系删除事件** | RelationshipDeleting | RelationshipDeleting | 无 |
| **Transaction observer 副作用** | ✅ 触发（状态重算、TransactionsCounted、付款删除历史） | ❌ 被 mute | - |
| **监听器历史记录** | ✅ CreateDocumentHistory（标记取消） | 无 | ✅ CreateDocumentHistory（恢复） |
| **可逆性** | 可通过 Restore 恢复状态 | 可软恢复但关系数据不恢复 | 仅恢复 status 字段 |
| **付款关系丢失** | ⚠️ 取消后软删除，恢复后仍不可见 | ⚠️ 软删除 | ❌ 不主动删除但也不恢复 |
