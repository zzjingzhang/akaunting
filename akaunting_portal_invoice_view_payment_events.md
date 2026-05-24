# Akaunting Portal Invoice View & Payment 控制流分析

## 1. Route 与 Request 精确区分

### 1.1 三类路由体系

**A. Core Portal 查看路由** ([routes/portal.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/portal.php)):

```php
Route::get('invoices/{invoice}/print', 'Portal\Invoices@printInvoice')->name('invoices.print');
Route::get('invoices/{invoice}/pdf', 'Portal\Invoices@pdfInvoice')->name('invoices.pdf');
Route::get('invoices/{invoice}/finish', 'Portal\Invoices@finish')->name('invoices.finish');
Route::resource('invoices', 'Portal\Invoices');
// 展开后含: GET portal.invoices.index, GET portal.invoices.show
```

**B. Core Signed 查看路由 + 占位支付路由** ([routes/signed.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/signed.php)):

```php
Route::get('invoices/{invoice}', 'Portal\Invoices@signed')->name('signed.invoices.show');
Route::get('invoices/{invoice}/print', 'Portal\Invoices@printInvoice')->name('signed.invoices.print');
Route::get('invoices/{invoice}/pdf', 'Portal\Invoices@pdfInvoice')->name('signed.invoices.pdf');
Route::post('invoices/{invoice}/payment', 'Portal\Invoices@payment')->name('signed.invoices.payment');
Route::post('invoices/{invoice}/confirm', 'Portal\Invoices@confirm')->name('signed.invoices.confirm');
Route::get('invoices/{invoice}/finish', 'Portal\Invoices@finish')->name('signed.invoices.finish');
```

> **关键发现**：`signed.invoices.payment` → `Portal\Invoices@payment` 和 `signed.invoices.confirm` → `Portal\Invoices@confirm` 在 [Portal\Invoices 控制器](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php) 中**没有对应方法**。控制器实际方法为：`index()`、`show()`、`finish()`、`printInvoice()`、`pdfInvoice()`、`preview()`、`signed()`。这两个 POST 路由是**占位路由**，支付模块通过注册自己的同名路由来覆盖（module 的 RouteServiceProvider 在 core 之后加载）。

**C. 模块支付路由**（由支付模块自行注册，命名约定示例）:

```
signed.{alias}.invoices.show    → 模块控制器的 show() 方法
signed.{alias}.invoices.confirm → 模块控制器的 confirm() 方法
portal.{alias}.invoices.show    → 模块控制器的 show() 方法
portal.{alias}.invoices.confirm → 模块控制器的 confirm() 方法
```

这些路由由 `PaymentController::getModuleUrl()` 动态构建：
```php
// app/Abstracts/Http/PaymentController.php#L155-L160
public function getModuleUrl($invoice, $suffix)
{
    return request()->isPortal($invoice->company_id)
        ? route('portal.' . $this->alias . '.invoices.' . $suffix, $invoice->id)
        : URL::signedRoute('signed.' . $this->alias . '.invoices.' . $suffix, [$invoice->id]);
}
```

### 1.2 Request 类对比

| Request 类 | 被使用位置 | 验证规则 |
|------------|-----------|----------|
| [InvoiceShow](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Portal/InvoiceShow.php) | `Portal\Invoices::show()`, `finish()`, `printInvoice()`, `pdfInvoice()` | 空 rules，仅 authorize 检查 contact_id 匹配 |
| [InvoicePayment](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Portal/InvoicePayment.php) | `PaymentController::show()`, `signed()` | 空 rules，仅 FormRequest 基础 |
| [InvoiceConfirm](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Portal/InvoiceConfirm.php) | **当前源码中未被任何核心代码直接 use 或 type-hint** | 空 rules，仅 FormRequest 基础 |

> `InvoiceConfirm` 存在于 `app/Http/Requests/Portal/` 下，但搜索整个 `app/` 目录无任何控制器或方法对其进行类型提示或引用。它是为支付模块扩展预留的。

---

## 2. DocumentViewed 触发与 MarkDocumentViewed 状态更新

### 2.1 触发点

**Portal show（已认证用户）** — [Portal\Invoices.php#L63](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php#L63)：

```php
public function show(Document $invoice, Request $request)
{
    $invoice->load([...]);
    $payment_methods = Modules::getPaymentMethods();
    event(new \App\Events\Document\DocumentViewed($invoice)); // 无条件触发
    return view('portal.invoices.show', ...);
}
```

**Signed show（访客/签名链接）** — [Portal\Invoices.php#L187-L189](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php#L187-L189)：

```php
// Guest or Invoice contact user track the invoice viewed.
if (empty(user()) || user()->id == $invoice->contact->user_id) {
    event(new \App\Events\Document\DocumentViewed($invoice)); // 条件触发
}
```

### 2.2 MarkDocumentViewed 逻辑

[MarkDocumentViewed.php#L19-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentViewed.php#L19-L49)：

```php
public function handle(Event $event)
{
    $document = $event->document;

    // 仅当状态严格等于 'sent' 时，改为 'viewed'
    if ($document->status != 'sent') {
        return;
    }

    unset($document->paid);
    $document->status = 'viewed';
    $document->save();

    $this->dispatch(new CreateDocumentHistory(
        $event->document, 0,
        trans('documents.messages.marked_viewed', ['type' => $type])
    ));
}
```

**精确结论**：`MarkDocumentViewed` 的状态转换只有 `sent → viewed` 一种路径。对 `viewed`、`partial`、`paid`、`cancelled`、`draft` 等所有其他状态均直接 return，不产生任何副作用。

---

## 3. 状态覆盖关系（仅基于代码可证）

本分析**不**使用全局优先级概念。以下每条结论均对应具体代码行：

### 3.1 各代码路径对状态的写入

| 代码位置 | 写入条件 | 写入结果 |
|----------|----------|----------|
| [MarkDocumentViewed#L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/MarkDocumentViewed.php#L29) | `$document->status == 'sent'` | `status = 'viewed'` |
| [CreateBankingDocumentTransaction#L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119) | `bccomp($amount, $total_amount, $precision) === 0` | `status = 'paid'` |
| [CreateBankingDocumentTransaction#L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/CreateBankingDocumentTransaction.php#L119) | `bccomp($amount, $total_amount, $precision) === -1` | `status = 'partial'` |
| [UpdateDocument#L61](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L61) | `$paid_amount > 0 && $request['amount'] == $paid_amount` | `status = 'paid'` |
| [UpdateDocument#L65](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L65) | `$paid_amount > 0 && $request['amount'] > $paid_amount` | `status = 'partial'` |
| [Transaction Observer#L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L48) | 交易被删除（invoice） | `status = 'sent'` |
| [Transaction Observer#L51](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Observers/Transaction.php#L51) | 交易被删除后仍有剩余交易 | `status = 'partial'` |

### 3.2 具体可观察到的覆盖关系

**能被证明的覆盖（写入操作明确发生）**：

| 从 | 到 | 由谁执行 | 条件 |
|----|----|----------|------|
| `sent` | `viewed` | MarkDocumentViewed | 仅 `sent` 可触发 |
| `sent` | `paid` | CreateBankingDocumentTransaction | `amount == remaining` |
| `sent` | `partial` | CreateBankingDocumentTransaction | `amount < remaining` |
| `viewed` | `paid` | CreateBankingDocumentTransaction | `amount == remaining` |
| `viewed` | `partial` | CreateBankingDocumentTransaction | `amount < remaining` |
| `partial` | `paid` | CreateBankingDocumentTransaction | `amount == remaining` |
| `paid` | `partial` | Transaction Observer (deleted) | 删除部分交易后仍有剩余 |
| `paid` | `sent` | Transaction Observer (deleted) | 删除全部交易后无剩余 |
| `partial` | `sent` | Transaction Observer (deleted) | 删除全部交易后无剩余 |

**能被证明的不覆盖（guard 明确阻止）**：

| 尝试写入 | 被哪个 guard 阻止 | 代码位置 |
|----------|------------------|----------|
| `partial` → `viewed` | `$document->status != 'sent'` return | MarkDocumentViewed#L23 |
| `paid` → `viewed` | `$document->status != 'sent'` return | MarkDocumentViewed#L23 |

**不能确定或不存在的关系**：
- `viewed` 不会被自身覆盖（guard 阻止），但也无任何代码会从 `viewed` 回退到 `sent`
- `paid` → `partial` 仅在 Transaction Observer 的 `deleted` 事件中发生，CreateBankingDocumentTransaction 不会把 `paid` 改回 `partial`（因为全额支付后再次支付会触发 over_payment 异常）

---

## 4. 前端如何 GET 模块支付 show 并 POST confirm_url

### 4.1 Portal（已认证）前端流程

**视图** [show.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/portal/invoices/show.blade.php#L23-L86)：

```blade
@if (! empty($payment_methods) && ! in_array($invoice->status, ['paid', 'cancelled']))
    <div class="tabs" x-data="{ active: '{{ reset($payment_methods) }}' }">
        @foreach ($payment_methods as $key => $name)
            <div @click="onChangePaymentMethod('{{ $key }}')" ...>
                {{ $name }}
            </div>
        @endforeach

        @foreach ($payment_methods as $key => $name)
            <div id="tabs-payment-method-{{ $key }}">
                <component v-bind:is="method_show_html" @interface="onRedirectConfirm"></component>
            </div>
        @endforeach

        <x-form id="portal">
            <x-form.input.hidden name="document_id" :value="$invoice->id" v-model="form.document_id" />
        </x-form>
    </div>
@endif
```

**JS 逻辑** [apps.js#L118-L191](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/portal/apps.js#L118-L191)：

```javascript
onChangePaymentMethod(payment_method) {
    let method = payment_method.split('.');
    // 构造模块路由: /portal/{alias}/invoices/{document_id}
    let path = url + '/portal/' + method[0] + '/invoices/' + this.form.document_id;

    // GET 请求获取支付表单 HTML
    axios.get(path, {
        params: { payment_method: payment_method }
    })
    .then(response => {
        // 动态渲染模块返回的 HTML
        this.method_show_html = Vue.component('payment-method-confirm', ...);
        // 渲染后的 HTML 中包含 confirm 表单
        // 表单 action 由 PaymentController::getConfirmUrl() 生成
    });
},

onRedirectConfirm() {
    this.redirectForm = new Form('redirect-form');
    // POST 到模块生成的 confirm_url
    axios.post(this.redirectForm.action, this.redirectForm.data())
    .then(response => {
        if (response.data.redirect) {
            location = response.data.redirect; // 跳转至 finish 页面
        }
        if (response.data.success) {
            location.reload();
        }
    });
}
```

### 4.2 Signed（未认证）前端流程

**视图** [signed.blade.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/portal/invoices/signed.blade.php#L153-L156)：

```blade
@push('scripts_start')
    <script type="text/javascript">
        var payment_action_path = {!! json_encode($payment_actions) !!};
    </script>
@endpush
```

`$payment_actions` 在控制器中构建 ([Portal\Invoices.php#L171-L181](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Portal/Invoices.php#L171-L181))：

```php
foreach ($payment_methods as $payment_method_key => $payment_method_value) {
    $codes = explode('.', $payment_method_key);
    if (!isset($payment_actions[$codes[0]])) {
        $payment_actions[$codes[0]] = URL::signedRoute('signed.' . $codes[0] . '.invoices.show', [$invoice->id]);
    }
}
```

**JS 逻辑** [apps.js#L211-L286](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/assets/js/views/portal/apps.js#L211-L286)：

```javascript
onChangePaymentMethodSigned(payment_method) {
    let method = payment_method.split('.');
    // 直接使用预渲染的 payment_action_path（signed URL）
    let payment_action = payment_action_path[method[0]];

    // GET 到模块的 signed show 路由
    axios.get(payment_action, {
        params: { payment_method: payment_method }
    })
    .then(response => {
        this.method_show_html = Vue.component('payment-method-confirm', ...);
        // 渲染后的 HTML 中 confirm_url 由 PaymentController::getConfirmUrl() 生成
    });
}

// onRedirectConfirm 逻辑同上
```

### 4.3 模块 show 返回内容与 confirm_url 生成

[PaymentController::show()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L45-L66)：

```php
public function show(Document $invoice, PaymentRequest $request, $cards = [])
{
    $this->setContactFirstLastName($invoice);
    $confirm_url = $this->getConfirmUrl($invoice);  // 生成模块 confirm URL

    $html = view('components.payment_method.' . $this->type, [
        'setting'     => $this->setting,
        'invoice'     => $invoice,
        'confirm_url' => $confirm_url,  // 注入到模板中
        // ...
    ])->render();

    return response()->json([
        'code' => $this->setting['code'],
        'html' => $html,  // 前端渲染此 HTML，其中表单 action = $confirm_url
    ]);
}
```

`getConfirmUrl()` ([PaymentController#L135-L138](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L135-L138)) 调用 `getModuleUrl('confirm')`，根据认证状态生成 `portal.{alias}.invoices.confirm` 或 `signed.{alias}.invoices.confirm`。

### 4.4 完整前端-后端交互时序

```
用户点击支付方式 Tab
    ↓
onChangePaymentMethod('offline-payments.offline')  [Portal]
  或 onChangePaymentMethodSigned('offline-payments.offline')  [Signed]
    ↓
axios GET /portal/offline-payments/invoices/{id}    [Portal]
  或 axios GET {signed_url_of_module_show}           [Signed]
    ↓
模块 PaymentController::show() (extends PaymentController)
    ↓
生成 confirm_url = route('portal.offline-payments.invoices.confirm', id)
  或 confirm_url = URL::signedRoute('signed.offline-payments.invoices.confirm', [id])
    ↓
返回 JSON { html: '<form action="{confirm_url}" ...>...</form>' }
    ↓
前端动态渲染 HTML（包含支付表单）
    ↓
用户填写/点击表单提交
    ↓
axios POST {confirm_url} (即模块 confirm 路由)
    ↓
模块 PaymentController::confirm() 实现
    ↓
处理支付网关响应 → 调用 $this->finish()
    ↓
PaymentController::finish() → dispatchPaidEvent()
```

---

## 5. dispatchPaidEvent() 全额 amount 对 partial invoice 的影响

### 5.1 dispatchPaidEvent 的实现

[PaymentController.php#L170-L180](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Abstracts/Http/PaymentController.php#L170-L180)：

```php
public function dispatchPaidEvent($invoice, $request)
{
    $request['company_id']     = $invoice->company_id;
    $request['account_id']     = setting($this->alias . '.account_id', setting('default.account'));
    $request['amount']         = $invoice->amount;  // ← 始终设置为发票全额
    $request['payment_method'] = $this->alias;
    $request['reference']      = $this->getReference($invoice);
    $request['type']           = 'income';

    event(new PaymentReceived($invoice, $request));
}
```

**关键行为**：`$request['amount'] = $invoice->amount` 无条件覆盖传入的 `$request` 中的 amount 字段，始终等于发票的总金额（`grand_total`）。

### 5.2 对 partial invoice 的影响链

**场景**：一张 $100 的 invoice 已有 $30 线下支付（状态 `partial`），客户通过在线支付模块支付剩余 $70。

1. 模块 `confirm()` 调用 `$this->finish($invoice, $request)`
2. `finish()` 调用 `dispatchPaidEvent($invoice, $request)`
3. `dispatchPaidEvent` 设置 `$request['amount'] = 100.00`（全额）
4. `PaymentReceived` 事件触发
5. `CreateDocumentTransaction` 监听器分发 `CreateBankingDocumentTransaction`
6. `CreateBankingDocumentTransaction::prepareRequest()` 检查：
   ```php
   if (!isset($this->request['amount'])) {
       // 不会进入，因为 amount 已被设置
   }
   ```
7. `checkAmount()` 执行：
   ```php
   $amount = 100.00;  // 来自 dispatchPaidEvent
   $this->model->paid_amount = 30.00;
   $total_amount = round(100.00 - 30.00, 2) = 70.00;
   $compare = bccomp(100.00, 70.00, 2);  // = 1
   ```
8. `$compare === 1` → 抛出 `over_payment` 异常："Cannot overpay. The amount should be $70.00 or less."
9. 异常被 `CreateDocumentTransaction::handle()` 的 `catch` 块捕获
10. **但 `finish()` 不读取事件监听器返回值**，继续返回成功响应

**结论**：对 `partial` 状态的 invoice 通过在线支付模块支付时，`dispatchPaidEvent()` 硬编码全额 amount 会导致 `CreateBankingDocumentTransaction::checkAmount()` 抛出 over_payment 异常。该异常被监听器捕获，但 `finish()` 无法感知，仍向用户返回"支付成功"。

**例外**：若这是该 invoice 的**首次支付**（`paid_amount = 0`），则 `total_amount = invoice->amount`，`bccomp` 返回 0，状态正确设为 `paid`，流程正常。

---

## 6. Online Payment 流与 Direct Document Transaction 流的对比

### 6.1 Online Payment 流（Portal 客户通过支付网关）

```
模块 PaymentController::confirm()    [模块实现]
    ↓
模块完成支付网关回调/验证
    ↓
$this->finish($invoice, $request)   [PaymentController#L95-L119]
    ↓
$this->dispatchPaidEvent($invoice, $request)  [PaymentController#L170]
    ↓
event(new PaymentReceived($invoice, $request))
    ↓
├─ CreateDocumentTransaction::handle()  [Listener]
│   └─ try {
│        $this->dispatch(new CreateBankingDocumentTransaction(...));
│      } catch (\Exception $e) {
│        flash($message)->error();
│        return $this->getResponse(...);  // 返回值被事件系统丢弃
│      }
│
└─ SendDocumentPaymentNotification::handle()  [Listener, 仅 income]
    ↓
PaymentController::finish() 继续执行（不读取事件返回值）
    ↓
flash('Payment added successfully')->success();  ← 即使交易创建失败
    ↓
return response()->json(['success' => true, 'redirect' => $finish_url, ...]);
```

**关键特性**：
- 交易创建通过 **事件-监听器** 机制异步解耦
- `CreateDocumentTransaction` 内部的异常被 catch，但 **事件系统不向调用者传播返回值**
- `finish()` 无法感知交易创建是否成功，始终返回成功
- 这是一个**控制流盲区**：网关已扣款但本地交易创建失败时，用户看到"成功"但无交易记录

### 6.2 Direct Document Transaction 流（后台通过模态框手动记录支付）

[DocumentTransactions::store()](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Modals/DocumentTransactions.php#L128-L143)：

```php
public function store(Document $document, Request $request)
{
    $response = $this->ajaxDispatch(new CreateBankingDocumentTransaction($document, $request));

    if ($response['success']) {
        $response['redirect'] = $this->getRedirectUrl($document, $request);
        $message = trans('messages.success.added', ['type' => trans_choice('general.payments', 1)]);
        flash($message)->success();
    } else {
        $response['redirect'] = null;  // 失败时不跳转
    }

    return response()->json($response);
}
```

**关键特性**：
- 直接调用 `CreateBankingDocumentTransaction`，**不经由事件系统**
- `ajaxDispatch` 直接返回 `['success' => true/false, 'message' => ..., 'redirect' => ...]`
- 若抛出异常，异常由 `ajaxDispatch` 捕获并封装到 response 中
- 控制器根据 `$response['success']` 决定是否 flash 成功消息
- **控制流清晰**：交易创建失败时，用户看到错误信息

### 6.3 两条路径的关键差异

| 维度 | Online Payment 流 | Direct Document Transaction 流 |
|------|-------------------|-------------------------------|
| 入口 | 模块 `confirm()` → `finish()` | `DocumentTransactions::store()` |
| 交易创建方式 | `event(PaymentReceived)` → 监听器 | 直接 `dispatch(CreateBankingDocumentTransaction)` |
| 异常传播 | 监听器 catch 但返回值被丢弃 | `ajaxDispatch` 捕获并返回给控制器 |
| 失败感知 | `finish()` 无法感知，始终返回成功 | 控制器读取 `$response['success']`，失败时返回错误 |
| amount 来源 | `dispatchPaidEvent` 硬编码 `$invoice->amount` | 来自前端表单输入的实际支付金额 |
| partial invoice 支付 | 有全额 amount 导致的 over_payment 风险 | 前端提交实际金额，无此问题 |

---

## 7. CreateDocumentTransaction 异常捕获对 finish() 控制流的影响

### 7.1 异常捕获代码

[CreateDocumentTransaction.php#L20-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Document/CreateDocumentTransaction.php#L20-L49)：

```php
public function handle(Event $event)
{
    $document = $event->document;
    $request = $event->request;

    try {
        $this->dispatch(new CreateBankingDocumentTransaction($document, $request));
    } catch (\Exception $e) {
        $message = $e->getMessage();
        $user = user();
        $type = Str::plural($event->document->type);
        $signed = request()->isSigned($document->company_id);

        if (empty($user) || $signed) {
            flash($message)->error()->important();
            return $this->getResponse('signed.' . $type . '.show', $document, $message);
        }

        if ($user->isCustomer()) {
            flash($message)->error()->important();
            return $this->getResponse('portal.' . $type . '.show', $document, $message);
        }

        throw new \Exception($message);
    }
}
```

### 7.2 返回值丢失的机制

Laravel 的 `event()` 函数调用监听器时，**不收集也不返回监听器的返回值**。监听器返回的 `response()->json(...)` 或 `redirect(...)` 被静默丢弃。

### 7.3 控制流后果

```
PaymentController::finish()
    ↓
dispatchPaidEvent()
    ↓
event(PaymentReceived)
    ↓
CreateDocumentTransaction::handle()
    ↓
    ├─ 成功路径：CreateBankingDocumentTransaction 正常完成
    │   → handle() 返回 null（无显式 return）
    │   → finish() 继续，flash 成功，返回 redirect
    │
    └─ 异常路径：CreateBankingDocumentTransaction 抛出异常
        → catch 块 flash 错误消息
        → catch 块 return response()->json(['success' => false, ...])
        → 但该 return 值被事件系统丢弃
        → finish() 继续执行，flash 成功，返回 redirect
        → 用户看到"成功"但 flash 错误可能还在 session 中
        → 用户被重定向到 finish 页面，看到成功消息
```

**具体场景**：partial invoice 在线支付 $70，`dispatchPaidEvent` 设置 amount = $100 → over_payment 异常 → 监听器 catch → flash "Cannot overpay..." → return error response → 返回值丢弃 → `finish()` flash "Payment added successfully" → 用户被 redirect 到 finish 页。

**session 中的 flash**：catch 块的 `flash($message)->error()` 和 `finish()` 的 `flash($message)->success()` 都写入 session。最终用户看到的取决于哪个先被消费，或两者都可能显示。

---

## 8. 已 Paid 发票被查看时不退回 Viewed 的完整路径

### 8.1 代码执行追踪

```php
// 场景：invoice #42 状态为 'paid'，客户在 Portal 中点击查看

// Portal\Invoices::show() — L45-L76
public function show(Document $invoice, Request $request)
{
    // $invoice->status === 'paid'
    $invoice->load([...]);
    $payment_methods = Modules::getPaymentMethods();

    event(new \App\Events\Document\DocumentViewed($invoice));
    // ↓ 事件系统同步调用所有监听器

    // MarkDocumentViewed::handle() — L19-L49
    //   $document->status === 'paid'
    //   if ($document->status != 'sent') { return; }  ← 命中，直接返回
    //   → 无状态变更，无历史记录写入

    // SendDocumentViewNotification::handle()
    //   → 发送查看通知（不受状态 guard 影响）

    return view('portal.invoices.show', ...);
    // $invoice->status 仍为 'paid'
}
```

### 8.2 视图层的状态感知

[show.blade.php#L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/portal/invoices/show.blade.php#L23) 和 [signed.blade.php#L32](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/resources/views/portal/invoices/signed.blade.php#L32)：

```blade
@if (! empty($payment_methods) && ! in_array($invoice->status, ['paid', 'cancelled']))
    {{-- 支付选项卡 --}}
@endif
```

当状态为 `paid` 时，支付方式选项卡**不渲染**，客户无法再次发起支付。这进一步确保 `paid` 状态不会因后续操作被意外修改。

### 8.3 可观察的完整不变量

| 操作 | 前置状态 | 状态是否变更 | 原因 |
|------|----------|--------------|------|
| 客户在 Portal 查看 | `paid` | **否** | `MarkDocumentViewed` guard `!= sent` 阻止 |
| 客户在 Portal 查看 | `partial` | **否** | 同上 |
| 客户在 Portal 查看 | `viewed` | **否** | 同上（guard 阻止 viewed → viewed 重复写入） |
| 客户在 Portal 查看 | `sent` | **是，变为 viewed** | guard 通过 |
| 后台删除交易记录 | `paid` | **可能变为 partial/sent** | Transaction Observer deleted 无 guard |
| 后台手动新增交易 | `paid` | **否** | checkAmount 会抛 over_payment |

---

## 总结

| 问题 | 代码定位的答案 |
|------|---------------|
| Portal invoice show 使用哪些 route？ | `portal.invoices.show` (GET)、`signed.invoices.show` (GET)、`preview.invoices.show` (GET) |
| Portal payment 使用哪些 route？ | 模块自行注册的 `portal.{alias}.invoices.show` (GET) 和 `portal.{alias}.invoices.confirm` (POST)；signed 同理 |
| `signed.invoices.payment`/`confirm` 路由指向哪里？ | 指向 `Portal\Invoices@payment`/`@confirm`，但该控制器**没有这些方法**（占位路由） |
| DocumentViewed 何时触发？ | `show()` 无条件触发；`signed()` 仅访客或 contact 用户触发 |
| MarkDocumentViewed 如何改状态？ | 仅 `sent → viewed`，其他状态一律 return |
| viewed/partial/paid 会互相覆盖吗？ | viewed 不会覆盖 partial/paid（guard 阻止）；partial/paid 可通过 Transaction Observer 删除交易降级 |
| dispatchPaidEvent 对 partial invoice 的影响？ | 硬编码全额 amount → `checkAmount()` 抛 over_payment → 监听器 catch 但 finish() 无感 |
| Online payment 与 direct transaction 的区别？ | 前者经事件-监听器（返回值丢失），后者直接 dispatch（异常可感知）；前者 amount 被硬编码 |
| 已 paid 发票被查看会退回 viewed 吗？ | **不会**。`MarkDocumentViewed` 的 guard `!= sent` 阻止；视图层也不渲染支付选项 |
