# Akaunting Portal Invoice View & Payment 控制流分析

## 1. Portal Invoice Show/Payment 使用的 Route 和 Request

### 路由定义

**Portal 路由** ([routes/portal.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/portal.php)):
```php
Route::get('invoices/{invoice}/print', 'Portal\Invoices@printInvoice')->name('invoices.print');
Route::get('invoices/{invoice}/pdf', 'Portal\Invoices@pdfInvoice')->name('invoices.pdf');
Route::get('invoices/{invoice}/finish', 'Portal\Invoices@finish')->name('invoices.finish');
Route::resource('invoices', 'Portal\Invoices');
```

**Signed 路由** ([routes/signed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/signed.php)):
```php
Route::get('invoices/{invoice}', 'Portal\Invoices@signed')->name('signed.invoices.show');
Route::post('invoices/{invoice}/payment', 'Portal\Invoices@payment')->name('signed.invoices.payment');
Route::post('invoices/{invoice}/confirm', 'Portal\Invoices@confirm')->name('signed.invoices.confirm');
```

### 核心控制器方法

**Portal\Invoices 控制器** ([app/Http/Controllers/Portal/Invoices.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php)):

| 方法 | 路由 | 说明 |
|------|------|------|
| `index()` | `GET /portal/invoices` | 发票列表 |
| `show()` | `GET /portal/invoices/{invoice}` | 查看发票详情 |
| `signed()` | `GET /invoices/{invoice}` (signed) | 签名链接查看发票 |
| `finish()` | `GET /portal/invoices/{invoice}/finish` | 支付完成页面 |

### Request 类

- `App\Http\Requests\Portal\InvoiceShow` - 用于 show, signed, finish 等方法

---

## 2. DocumentViewed 触发时机与 MarkDocumentViewed 状态更新

### DocumentViewed 事件触发位置

**触发点1: Portal show 方法** ([app/Http/Controllers/Portal/Invoices.php#L63](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php#L63)):
```php
public function show(Document $invoice, Request $request)
{
    // ... 加载关联关系 ...
    event(new \App\Events\Document\DocumentViewed($invoice));
    // ... 返回视图 ...
}
```

**触发点2: Signed 方法** ([app/Http/Controllers/Portal/Invoices.php#L187-L189](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php#L187-L189)):
```php
// Guest or Invoice contact user track the invoice viewed.
if (empty(user()) || user()->id == $invoice->contact->user_id) {
    event(new \App\Events\Document\DocumentViewed($invoice));
}
```

### MarkDocumentViewed 监听器逻辑

**监听器** ([app/Listeners/Document/MarkDocumentViewed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentViewed.php)):

```php
public function handle(Event $event)
{
    $document = $event->document;

    // 关键检查：只有状态为 'sent' 时才更新为 'viewed'
    if ($document->status != 'sent') {
        return;  // 直接返回，不做任何修改
    }

    unset($document->paid);

    $document->status = 'viewed';
    $document->save();

    // 创建历史记录
    $this->dispatch(new CreateDocumentHistory(...));
}
```

**事件注册** ([app/Providers/Event.php#L86-L89](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L86-L89)):
```php
\App\Events\Document\DocumentViewed::class => [
    \App\Listeners\Document\MarkDocumentViewed::class,
    \App\Listeners\Document\SendDocumentViewNotification::class,
],
```

---

## 3. Viewed、Partial、Paid 状态之间的覆盖关系

### 状态转换约束分析

**MarkDocumentViewed 的状态保护** ([app/Listeners/Document/MarkDocumentViewed.php#L23-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentViewed.php#L23-L25)):
```php
if ($document->status != 'sent') {
    return;
}
```

**关键结论**：
- `MarkDocumentViewed` **只会**在状态为 `sent` 时将其改为 `viewed`
- 如果当前状态是 `partial`、`paid`、`viewed`、`cancelled` 等，都会直接 return，**不会覆盖**
- 因此：`viewed` 不会覆盖 `partial` 或 `paid`

### Payment 时的状态转换

**CreateBankingDocumentTransaction** ([app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119)):
```php
$this->model->status = ($compare === 0) ? 'paid' : 'partial';
```

**UpdateDocument** ([app/Jobs/Document/UpdateDocument.php#L59-L67](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L59-L67)):
```php
if ($this->model->paid_amount > 0) {
    if ($this->request['amount'] == $this->model->paid_amount) {
        $this->request['status'] = 'paid';
    }
    if ($this->request['amount'] > $this->model->paid_amount) {
        $this->request['status'] = 'partial';
    }
}
```

**状态覆盖优先级**：
- `paid` (最高) > `partial` > `viewed` > `sent` (最低)
- Payment 操作可以将 `sent`、`viewed` 转为 `partial` 或 `paid`
- Payment 操作可以将 `partial` 转为 `paid`（当剩余金额付清时）
- 但 `viewed` 事件**不会**反向覆盖 `partial` 或 `paid`

---

## 4. Payment 提交后进入 Document Transaction 或 Payment Flow

### 支付流程控制流

```
PaymentController::confirm() (各支付模块实现)
    ↓
PaymentController::finish() ([app/Abstracts/Http/PaymentController.php#L95](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L95))
    ↓
dispatchPaidEvent() ([app/Abstracts/Http/PaymentController.php#L170](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L170))
    ↓
event(new PaymentReceived($invoice, $request))
    ↓
├─ CreateDocumentTransaction ([app/Listeners/Document/CreateDocumentTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/CreateDocumentTransaction.php))
│   ↓
│   dispatch(new CreateBankingDocumentTransaction($document, $request))
│       ↓
│       CreateTransaction (创建银行交易记录)
│       ↓
│       更新 document.status = 'paid' or 'partial'
│       ↓
│       创建 DocumentHistory
│
└─ SendDocumentPaymentNotification (发送支付通知)
```

### PaymentReceived 事件监听器

**事件注册** ([app/Providers/Event.php#L73-L76](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L73-L76)):
```php
\App\Events\Document\PaymentReceived::class => [
    \App\Listeners\Document\CreateDocumentTransaction::class,
    \App\Listeners\Document\SendDocumentPaymentNotification::class,
],
```

### CreateBankingDocumentTransaction 详细流程

1. **准备请求数据** ([app/Jobs/Banking/CreateBankingDocumentTransaction.php#L51-L72](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L51-L72)):
   - 计算应付金额
   - 设置公司ID、货币、支付日期、汇率等

2. **金额校验** ([app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L74-L123)):
   - 检查是否超额支付
   - 更新文档状态：`paid`（全额）或 `partial`（部分支付）

3. **创建交易** ([app/Jobs/Banking/CreateBankingDocumentTransaction.php#L38-L44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L38-L44)):
   - `dispatch(new CreateTransaction($this->request))`
   - 保存文档
   - 创建历史记录

---

## 5. 已 Paid 发票被查看时不应退回 Viewed 的例子

### 场景描述

假设一个发票经历了以下状态流转：
1. 发票创建并发送给客户 → 状态: `sent`
2. 客户支付了全额款项 → 状态: `paid`
3. 客户在支付完成后，再次打开发票查看详情

### 代码执行路径

```php
// 客户访问 portal/invoices/{invoice}
public function show(Document $invoice, Request $request)
{
    // 发票当前状态: 'paid'
    
    event(new DocumentViewed($invoice));
    // ↓
    // MarkDocumentViewed::handle() 被触发
    // ↓
    // if ($document->status != 'sent') {
    //     return;  // 因为 status 是 'paid'，直接返回！
    // }
    // ↓
    // 状态保持为 'paid'，不被修改为 'viewed'
}
```

### 预期行为 vs 实际行为

| 项目 | 状态 |
|------|------|
| 支付完成后的状态 | `paid` |
| 再次查看触发 DocumentViewed | ✓ 触发 |
| MarkDocumentViewed 执行 | ✓ 执行 |
| 状态检查 `status != 'sent'` | `true` (当前是 'paid') |
| 执行 return | ✓ 直接返回 |
| 状态是否被修改为 'viewed' | ✗ 不修改 |
| 最终状态 | `paid` (保持不变) |

### 为什么这是正确的设计

1. **语义正确性**：`paid` 表示交易已完成，比 `viewed` 具有更高的状态优先级
2. **业务准确性**：已经支付的发票不应该因为被再次查看而退回到"已查看"状态
3. **数据一致性**：财务记录应该反映最终的交易状态，而不是查看行为
4. **状态机设计**：遵循了单向状态流转原则，高级状态不会被低级状态覆盖

---

## 总结

### 关键状态转换表

| 当前状态 | 触发事件 | 目标状态 | 允许 |
|----------|----------|----------|------|
| `sent` | DocumentViewed | `viewed` | ✓ |
| `sent` | Payment (全额) | `paid` | ✓ |
| `sent` | Payment (部分) | `partial` | ✓ |
| `viewed` | Payment (全额) | `paid` | ✓ |
| `viewed` | Payment (部分) | `partial` | ✓ |
| `viewed` | DocumentViewed | `viewed` | ✗ (无变化) |
| `partial` | Payment (剩余) | `paid` | ✓ |
| `partial` | DocumentViewed | `viewed` | ✗ (被拦截) |
| `paid` | DocumentViewed | `viewed` | ✗ (被拦截) |
| `paid` | Payment | - | ✗ (超额支付检查) |

### 核心保护机制

[MarkDocumentViewed.php#L23-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentViewed.php#L23-L25) 的状态检查是整个保护机制的关键：
```php
if ($document->status != 'sent') {
    return;
}
```
