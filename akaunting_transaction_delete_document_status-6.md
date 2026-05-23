# Akaunting 删除 Banking Transaction 后 Invoice/Bill 与 Split 父交易更新分析

## 核心代码位置

主逻辑位于 [app/Observers/Transaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php) 的 `deleted()` 方法。

---

## 1. deleted observer 如何根据 transaction type 找到 invoice 或 bill

### 代码片段

```php
// [Transaction.php#L23-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L23-L29)
public function deleted(Model $transaction)
{
    if (! empty($transaction->document_id)) {
        $type = ($transaction->type == 'income') ? Document::INVOICE_TYPE : Document::BILL_TYPE;
        $this->updateDocument($transaction, $type);
    }
    // ...
}
```

### 查找逻辑

1. **判断关联存在**：首先检查 `$transaction->document_id` 是否非空，确定该交易关联了某个文档。
2. **类型映射**：
   - 若 `transaction->type == 'income'` → `$type = Document::INVOICE_TYPE`（值为 `'invoice'`）
   - 否则 → `$type = Document::BILL_TYPE`（值为 `'bill'`）
3. **动态属性关联**：在 `updateDocument()` 中通过 `$transaction->{$type}` 利用 Eloquent 动态属性获取关联文档：
   - `$transaction->invoice()` 定义于 [Transaction.php#L136-L139](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L136-L139)
   - `$transaction->bill()` 定义于 [Transaction.php#L111-L114](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Banking/Transaction.php#L111-L114)
   - 两者均通过 `document_id` 外键关联到 `App\Models\Document\Document`

---

## 2. transactions_count、TransactionsCounted 事件与 sent/received/partial 状态决策

### 代码片段

```php
// [Transaction.php#L36-L57](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L36-L57)
protected function updateDocument($transaction, $type)
{
    $document = $transaction->{$type};
    if (empty($document)) {
        return;
    }

    $document->transactions_count = $document->transactions->count();  // ①

    event(new TransactionsCounted($document));                         // ②

    $document->status = ($type == Document::INVOICE_TYPE) ? 'sent' : 'received';  // ③

    if ($document->transactions_count > 0) {                           // ④
        $document->status = 'partial';
    }

    unset($document->transactions_count);                              // ⑤
    $document->save();
    // ...
}
```

### 执行流程

| 步骤 | 操作 | 说明 |
|------|------|------|
| ① | 计算 `transactions_count` | 统计该文档当前剩余的关联交易数量（删除操作已完成，此 count 是删除后的结果） |
| ② | 触发 `TransactionsCounted` 事件 | 纯钩子事件，无默认监听器，留给扩展点使用 |
| ③ | 设置默认状态 | Invoice 默认 `'sent'`，Bill 默认 `'received'`（表示未付款） |
| ④ | 覆盖为 `'partial'` | 若仍有剩余交易，说明部分付款，覆盖为 `'partial'` |
| ⑤ | 清理临时属性 | `transactions_count` 不是数据库字段，unset 避免保存时报错 |

### 状态决策表

| 场景 | transactions_count | 最终 status |
|------|--------------------|-------------|
| 删除最后一笔付款 | 0 | Invoice: `sent` / Bill: `received` |
| 删除其中一笔付款，仍有剩余 | > 0 | `partial` |

---

## 3. 为什么不直接恢复 paid 状态，删除最后一笔与删除其中一笔的差异

### 设计意图（推测）

代码**只依据交易数量**决策状态，**完全不比较已付金额与总额**。原因可能包括：

1. **性能考量**：避免在 Observer 中进行金额计算（需遍历交易、处理汇率转换等）。
2. **简化逻辑**：认为「只要还有交易记录就视为部分付款」是保守策略。
3. **职责分离**：`paid` 状态通常在**新增/更新交易**时由付款逻辑判断，删除时只做「降级」处理。

### 删除最后一笔 vs 删除其中一笔

| 维度 | 删除最后一笔 | 删除其中一笔 |
|------|-------------|-------------|
| `transactions_count` | 0 | > 0 |
| 最终 status | `sent` / `received` | `partial` |
| `getPaidAttribute()` 返回值 | 0 | 剩余交易金额总和 |
| 业务含义 | 完全未付款 | 部分付款 |

### 关键缺陷

代码**永远不会把状态恢复为 `'paid'`**，即便剩余交易的金额总和恰好等于文档总额。例如：

- Invoice 总额 $100，两笔付款各 $50 → 状态 `paid`
- 删除其中一笔 $50 → 剩余 $50 → 状态变为 `partial`（正确）
- 但如果总额 $50，有两笔付款各 $50（重复付款），删除一笔 → 剩余 $50 等于总额 → 状态仍为 `partial`（本应是 `paid`，但逻辑不支持）

---

## 4. split_id 分支何时把父交易 type 改回子交易 type

### 代码片段

```php
// [Transaction.php#L68-L79](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L68-L79)
protected function updateTransaction($transaction)
{
    $splitted_transaction = Model::find($transaction->split_id);

    if ($splitted_transaction->splits->count() == 0) {
        $values = [
            'type' => $transaction->type,
        ];
        $this->dispatch(new UpdateTransaction($splitted_transaction, $values));
    }
}
```

### 触发条件

**当且仅当**被删除的 split 子交易是父交易的**最后一笔 split** 时：

1. 被删交易必须有 `split_id`（即它是某个 split 父交易的子交易）。
2. 父交易的 `splits` 关联查询结果数量为 0（即该子交易是最后一个，删除后没有其他 split 了）。
3. 将父交易的 `type` 字段**更新为当前被删除子交易的 `type`**。

### 设计意图

Split 父交易在拆分时 type 通常会被标记为特殊类型（如 `split_income` / `split_expense`）。当所有子 split 都被删除后，父交易恢复为普通的 `income` 或 `expense` 类型，使其可以再次被拆分或正常处理。

---

## 5. 删除后状态与剩余金额直觉不一致的边界场景

### 场景描述

**场景：零金额交易导致 partial 状态与实际已付金额不符**

| 项目 | 值 |
|------|----|
| Invoice 总额 | $100.00 |
| 交易 1（正常付款） | $100.00 |
| 交易 2（零金额，如错误录入、抵扣、或系统生成的调整记录） | $0.00 |
| 当前状态 | `paid`（因存在 $100 付款） |

### 操作

删除交易 1（$100.00 的付款）。

### 系统行为

1. `transactions_count` = 1（还剩 $0.00 的交易）
2. 因 `count > 0`，状态被设置为 `partial`
3. 但 `$document->paid`（通过 [getPaidAttribute()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L373-L407) 计算）实际返回 `0.00`

### 不一致性

- **状态显示**：`partial`（暗示部分付款）
- **实际已付**：$0.00（完全未付）
- **用户直觉**：看到 `partial` 会以为还有未结清的付款，实际已无有效付款

### 其他类似边界场景

1. **负金额交易**：存在 -$50 调整交易，删除 $100 付款后，剩余 -$50，状态仍为 `partial`，但实际已付为负。
2. **多笔小额交易总额为零**：三笔交易各 $0，删除其中两笔，剩余 $0，状态 `partial`。
3. **交易金额总和恰好等于总额**：总额 $100，三笔 $40、$30、$30，删除 $40 后剩余 $60 ≠ $100 → `partial`（合理），但如果总额 $60，同样场景下剩余 $60 仍为 `partial`（不合理）。

---

## 流程图

```
Transaction deleted
     │
     ├─► document_id 非空? ──┐
     │     Yes               │
     │                       ▼
     │          根据 type 动态获取 invoice/bill
     │                       │
     │                       ▼
     │          统计 transactions_count
     │                       │
     │                       ▼
     │          触发 TransactionsCounted 事件
     │                       │
     │                       ▼
     │          status = sent/received (默认)
     │                       │
     │                       ▼
     │          count > 0 ? ──► status = partial
     │                       │
     │                       ▼
     │          保存 document
     │
     └─► split_id 非空? ──┐
           Yes             │
                           ▼
                  找到父 split 交易
                           │
                           ▼
                  splits->count() == 0 ?
                           │
                           ▼
                  父交易 type = 子交易 type
```
