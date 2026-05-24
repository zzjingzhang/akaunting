# Akaunting Web 与 API 媒体上传/替换/删除机制对比分析

## 1. attachment 与 picture 分别使用哪些 collection

在 Akaunting 中，媒体（Media）通过 Mediable 包的 tag 机制进行分类管理，`attachment` 和 `picture` 是两种不同的 tag（也可理解为 collection）：

| 类型 | Tag/Collection 名称 | 使用场景 | 代码位置 |
|------|---------------------|----------|----------|
| attachment | `'attachment'` | 文档（发票、账单等）、转账、交易的附件 | [UpdateDocument.php#L43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L43-L43)、[UpdateTransfer.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransfer.php)、[UpdateTransaction.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Banking/UpdateTransaction.php) |
| picture | `'picture'` | 商品图片、用户头像 | [UpdateItem.php#L27](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/UpdateItem.php#L27-L27)、[UpdateUser.php#L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L36-L36) |

**关键代码：**
```php
// attachment 使用示例
$this->model->attachMedia($media, 'attachment');

// picture 使用示例
$this->model->attachMedia($media, 'picture');
```

模型访问器中也对应使用相同的 tag 读取媒体：
```php
// Document 模型 - getMedia() 来自 Mediable 包，按 tag 读取
public function getAttachmentAttribute($value = null)
{
    return $this->getMedia('attachment')->all();  // [Document.php#L312](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Document/Document.php#L312-L312)
}

// Item 模型
public function getPictureAttribute($value)
{
    return $this->getMedia('picture')->last();  // [Item.php#L182](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Item.php#L182-L182)
}
```

## 2. Web 没传文件但模型已有媒体时为什么可能删除

Web 端采用**隐式删除**机制，当满足以下条件时会进入删除流程：

```php
// [UpdateDocument.php#L45-L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L45-L46)
elseif ($this->request->isNotApi() && ! $this->request->file('attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

**三个进入条件：**
1. `$this->request->isNotApi()` - 请求来自 Web 表单而非 API
2. `! $this->request->file('attachment')` - 本次请求没有上传新文件
3. `$this->model->attachment` - 模型当前已有关联的媒体

**但进入 `deleteMediaModel` 后，实际删除还受 `uploaded_attachment` / `uploaded_picture` 保留列表影响：**

```php
// [Uploads.php#L73-L83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L73-L83)
if ($request && isset($request['uploaded_' . $parameter])) {
    $uploaded = ($multiple) ? $request['uploaded_' . $parameter] : [$request['uploaded_' . $parameter]];

    if (count($medias) == count($uploaded)) {
        return;  // 数量一致则跳过，不删除任何媒体
    }

    foreach ($uploaded as $old_media) {
        $already_uploaded[] = $old_media['id'];  // 收集要保留的媒体 ID
    }
}
```

**Dropzone 中间件的作用**：[Dropzone.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/Dropzone.php) 会解析前端 Dropzone 组件提交的数据，将已上传但本次未重新上传的媒体信息提取出来，设置为 `uploaded_attachment` / `uploaded_picture` 字段。这些媒体会被保留，不在列表中的才会被删除。

**设计原因：**
- Web 表单提交时，如果用户清空了文件选择框（或 Dropzone 中移除了所有已上传文件），则不会有 `uploaded_` 前缀的保留字段 → 所有旧媒体被删除
- 如果 Dropzone 中仍有已上传文件（用户未移除），则这些媒体的 ID 会出现在 `uploaded_` 列表中 → 被保留
- 这种设计允许用户在 Dropzone 中部分删除附件，而不必每次都重新上传所有文件

**同样的逻辑也适用于 picture：**
```php
// [UpdateItem.php#L28-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/UpdateItem.php#L28-L29)
elseif ($this->request->isNotApi() && ! $this->request->file('picture') && $this->model->picture) {
    $this->deleteMediaModel($this->model, 'picture', $this->request);
}
```

## 3. API 删除媒体为什么依赖 remove_attachment/remove_picture

API 端采用**显式删除**机制，只要请求中存在 `remove_attachment` 或 `remove_picture` 字段（不要求值为 `true`），就会触发删除：

```php
// [UpdateDocument.php#L47-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L47-L48)
elseif ($this->request->isApi() && $this->request->has('remove_attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

**注意**：这里使用的是 `$this->request->has('remove_attachment')`，仅检查字段是否存在，不检查其真值。即使传 `"remove_attachment": "0"` 或 `"remove_attachment": false`，只要字段存在就会进入删除分支。

**设计原因：**

1. **RESTful API 语义**：API 通常支持 PATCH 部分更新，未传递的字段应保持不变，而非被删除
2. **无状态特性**：API 无法像 Web 表单那样判断"空字段 = 删除"，因为 API 调用可能只更新部分字段
3. **安全性**：避免因客户端疏漏导致意外删除数据
4. **幂等性**：显式标记使 API 调用更可预测、更安全

**同样的逻辑也适用于 picture：**
```php
// [UpdateUser.php#L39-L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/UpdateUser.php#L39-L40)
elseif ($this->request->isApi() && $this->request->has('remove_picture') && $this->model->picture) {
    $this->deleteMediaModel($this->model, 'picture', $this->request);
}
```

## 4. deleteMediaModel 与两个 getMedia 的职责边界

系统中存在三个相关但职责不同的方法，需要明确区分：

### (A) Uploads::getMedia($file, $folder, ...) — 上传创建媒体

定义在 [Uploads.php#L12-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L12-L34)：

```php
public function getMedia($file, $folder = 'settings', $company_id = null)
{
    // 1. 验证文件有效性
    if (! $file || ! $file->isValid()) {
        return $path;
    }

    // 2. 确定存储路径
    $path = $this->getMediaFolder($folder, $company_id);
    $file_name = $this->getMediaFileName($file);

    // 3. 上传并创建 Media 记录
    return MediaUploader::makePrivate()
                        ->beforeSave(function(MediaModel $media) {
                            $media->company_id = company_id();
                            $media->created_from = source_name();
                            $media->created_by = user_id();
                        })
                        ->fromSource($file)
                        ->toDirectory($path)
                        ->useFilename($file_name)
                        ->upload();
}
```

**职责**：接收 `UploadedFile` 文件对象，将文件存储到磁盘，创建新的 `Media` 模型记录并返回。这是**创建**操作。

### (B) 模型侧 getMedia($tag) — 按 tag 读取媒体

来自 `Plank\Mediable\Mediable` trait，在 [Media.php#L18](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Media.php#L18-L18) 中通过 `use Mediable` 引入：

```php
// 在模型访问器中使用
public function getAttachmentAttribute($value = null)
{
    return $this->getMedia('attachment')->all();  // 读取 tag='attachment' 的所有媒体
}

public function getPictureAttribute($value)
{
    return $this->getMedia('picture')->last();  // 读取 tag='picture' 的最后一张图片
}
```

**职责**：查询并返回模型已关联的、指定 tag 的媒体集合。这是**读取**操作。

### (C) deleteMediaModel — 删除媒体

定义在 [Uploads.php#L55-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L55-L92)：

```php
public function deleteMediaModel($model, $parameter, $request = null)
{
    // 1. 获取模型关联的媒体（通过访问器间接调用 (B) 的 getMedia）
    $medias = $model->$parameter;

    // 2. 处理 uploaded_ 前缀的保留列表（来自 Dropzone 中间件）
    if ($request && isset($request['uploaded_' . $parameter])) {
        $uploaded = ($multiple) ? $request['uploaded_' . $parameter] : [$request['uploaded_' . $parameter]];

        if (count($medias) == count($uploaded)) {
            return;  // 数量完全匹配，不删除任何媒体
        }

        foreach ($uploaded as $old_media) {
            $already_uploaded[] = $old_media['id'];
        }
    }

    // 3. 删除不在保留列表中的媒体（仅删除 DB 记录，触发软删除）
    foreach ((array) $medias as $media) {
        if (in_array($media->id, $already_uploaded)) {
            continue;
        }
        MediaModel::where('id', $media->id)->delete();
    }
}
```

**职责**：删除模型关联的媒体记录。支持通过 `uploaded_` 参数保留部分媒体（Web 端 Dropzone 场景）。这是**删除**操作。

### Media 模型软删除的影响

[Media.php#L14](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Common/Media.php#L14-L14) 使用了 `SoftDeletes` trait，`deleteMediaModel` 调用的 `MediaModel::where('id', $media->id)->delete()` 实际上执行的是**软删除**（设置 `deleted_at` 字段）。

[Media.php#L24-L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Media.php#L24-L36) 的 `media()` 关系在 `detach_on_soft_delete` 为 false 时会过滤已软删除的媒体：

```php
public function media()
{
    $media = $this->morphToMany(config('mediable.model'), 'mediable')
        ->withPivot('tag', 'order')
        ->orderBy('order');

    // Skip deleted media if not detached
    if (config('mediable.detach_on_soft_delete') == false) {
        $media->whereNull('deleted_at');
    }

    return $media;
}
```

因此当 `detach_on_soft_delete = false` 时，被软删除的媒体在通过 `media()` 关系或 `getMedia()` 查询时不可见，但数据库中记录仍存在。

### 三个方法职责边界对比表

| 维度 | (A) Uploads::getMedia($file,...) | (B) 模型::getMedia($tag) | (C) deleteMediaModel |
|------|----------------------------------|--------------------------|----------------------|
| 核心职责 | 上传文件、创建 Media 记录 | 按 tag 读取已关联媒体 | 删除媒体记录（软删除） |
| 输入 | UploadedFile 文件对象 + 存储目录 | tag 名称（字符串） | 模型实例 + 参数名 + 可选请求 |
| 输出 | 新创建的 Media 模型实例 | 媒体集合（Media 实例数组） | 无返回值 |
| 操作类型 | C（创建） | R（读取） | D（删除） |
| 磁盘操作 | 写入文件到存储 | 无 | 无（仅操作 DB） |
| 来源 | Uploads trait | Mediable trait（Plank 包） | Uploads trait |
| 调用时机 | 有新文件上传时 | 访问模型附件/图片属性时 | 需要删除旧媒体时 |

## 5. API 客户端漏传 remove 标记导致旧附件保留的例子

### 场景描述

假设系统中已有一张发票（ID = 123），其 type 为 `invoice`，已关联了一份 PDF 附件。现在 API 客户端需要更新该发票的到期日 `due_at` 及其他字段，但不需要保留原有附件。

路由定义在 [api.php#L43](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/api.php#L43-L43)：
```php
Route::apiResource('documents', 'Document\Documents', ['middleware' => ['date.format', 'money', 'dropzone']]);
```

对应的更新路径为 `PUT /api/documents/{document}`，控制器方法为 [Documents.php#L95-L99](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Document/Documents.php#L95-L99)：
```php
public function update(Document $document, Request $request)
{
    $document = $this->dispatch(new UpdateDocument($document, $request));
    return new Resource($document->fresh());
}
```

### 错误的 API 调用（漏传 remove_attachment）

请求体包含满足 [Document.php 请求类](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L48-L67) 验证规则的完整字段，但未包含 `remove_attachment`：

```http
PUT /api/documents/123
Content-Type: application/json
Authorization: Bearer {token}

{
    "type": "invoice",
    "document_number": "INV-2024-0042",
    "status": "draft",
    "issued_at": "2024-05-15 00:00:00",
    "due_at": "2024-06-30 00:00:00",
    "amount": "1500.00",
    "currency_code": "USD",
    "currency_rate": "1.00000000",
    "contact_id": 5,
    "contact_name": "Acme Corp",
    "category_id": 3,
    "items": [
        {
            "name": "Consulting Service",
            "price": "1500.00",
            "quantity": "1"
        }
    ]
}
```

**执行路径分析**：

1. 请求通过 `dropzone` 中间件 → 无 `dropzone` 标记数据，`uploaded_attachment` 不会被设置
2. 进入 `UpdateDocument::handle()` → 检查 [第 37 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L37-L37) `$this->request->file('attachment')` → 无文件，跳过
3. 检查 [第 45 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L45-L45) `$this->request->isNotApi()` → false，跳过
4. 检查 [第 47 行](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L47-L47) `$this->request->isApi() && $this->request->has('remove_attachment')` → `has('remove_attachment')` 为 **false** → 跳过

**结果**：
- 发票字段更新成功 ✅
- 原有附件**被保留** ❌（因为 `remove_attachment` 字段不存在，删除分支未触发）
- 业务预期不符

### 正确的 API 调用（传 remove_attachment）

只需在请求体中加入 `remove_attachment` 字段（值可以是任意内容，仅需字段存在）：

```http
PUT /api/documents/123
Content-Type: application/json
Authorization: Bearer {token}

{
    "type": "invoice",
    "document_number": "INV-2024-0042",
    "status": "draft",
    "issued_at": "2024-05-15 00:00:00",
    "due_at": "2024-06-30 00:00:00",
    "amount": "1500.00",
    "currency_code": "USD",
    "currency_rate": "1.00000000",
    "contact_id": 5,
    "contact_name": "Acme Corp",
    "category_id": 3,
    "items": [
        {
            "name": "Consulting Service",
            "price": "1500.00",
            "quantity": "1"
        }
    ],
    "remove_attachment": "1"
}
```

**执行路径分析**：

1. 第 3 步中 `$this->request->has('remove_attachment')` 为 **true** → 进入删除分支
2. 调用 `deleteMediaModel($this->model, 'attachment', $this->request)`
3. 因 `uploaded_attachment` 不存在，`$already_uploaded` 为空数组 → 所有旧媒体被软删除

**结果**：
- 发票字段更新成功 ✅
- 原有附件被软删除 ✅
- 符合业务预期

### 另一个场景：上传新文件时的隐式替换

如果 API 客户端上传了新文件（`multipart/form-data`），即使不传 `remove_attachment`，旧附件也会被删除。注意 `attachment` 字段在代码中按数组处理（[UpdateDocument.php#L40](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L40-L40) `foreach ($this->request->file('attachment') as $attachment)`，以及 [Document.php#L64](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Requests/Document/Document.php#L64-L64) `'attachment.*'` 验证规则），因此 multipart 请求中需使用数组索引：

```http
PUT /api/documents/123
Content-Type: multipart/form-data
Authorization: Bearer {token}

type=invoice&document_number=INV-2024-0042&status=draft&issued_at=2024-05-15%2000:00:00&due_at=2024-06-30%2000:00:00&amount=1500.00&currency_code=USD&currency_rate=1.00000000&contact_id=5&contact_name=Acme+Corp&category_id=3&items[0][name]=Consulting+Service&items[0][price]=1500.00&items[0][quantity]=1&attachment[0]=@new_invoice.pdf
```

**代码逻辑**：
```php
// [UpdateDocument.php#L37-L44](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L37-L44)
if ($this->request->file('attachment')) {
    // 有新文件上传，先删除旧的
    $this->deleteMediaModel($this->model, 'attachment', $this->request);

    // 再上传新的
    foreach ($this->request->file('attachment') as $attachment) {
        $media = $this->getMedia($attachment, Str::plural($this->model->type));
        $this->model->attachMedia($media, 'attachment');
    }
}
```

**总结**：API 端删除媒体有三种触发路径：

| 路径 | 触发条件 | 说明 |
|------|----------|------|
| 显式删除 | 请求包含 `remove_attachment` / `remove_picture` 字段 | 字段存在即可，不要求值为 `true` |
| 隐式替换 | 请求上传了新的 `attachment` / `picture` 文件 | 自动删除旧媒体后上传新的 |
| 遗漏风险 | 既不上传新文件，也不传 `remove_*` 字段 | 旧媒体被保留 |
