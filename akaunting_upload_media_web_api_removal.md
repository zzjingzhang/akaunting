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

模型访问器中也对应使用相同的 tag：
```php
// Document 模型
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

Web 端采用**隐式删除**机制，当满足以下条件时会删除现有媒体：

```php
// [UpdateDocument.php#L45-L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L45-L46)
elseif ($this->request->isNotApi() && ! $this->request->file('attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

**三个必要条件：**
1. `$this->request->isNotApi()` - 请求来自 Web 表单而非 API
2. `! $this->request->file('attachment')` - 本次请求没有上传新文件
3. `$this->model->attachment` - 模型当前已有关联的媒体

**设计原因：**
- Web 表单提交时，如果用户清空了文件选择框（或保持为空），通常语义是"删除现有附件"
- 这是传统 Web 表单的交互习惯：空文件字段 = 删除
- 这种设计符合 Web 用户的操作预期

**同样的逻辑也适用于 picture：**
```php
// [UpdateItem.php#L28-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Common/UpdateItem.php#L28-L29)
elseif ($this->request->isNotApi() && ! $this->request->file('picture') && $this->model->picture) {
    $this->deleteMediaModel($this->model, 'picture', $this->request);
}
```

## 3. API 删除媒体为什么依赖 remove_attachment/remove_picture

API 端采用**显式删除**机制，必须传递专门的参数才能删除：

```php
// [UpdateDocument.php#L47-L48](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Document/UpdateDocument.php#L47-L48)
elseif ($this->request->isApi() && $this->request->has('remove_attachment') && $this->model->attachment) {
    $this->deleteMediaModel($this->model, 'attachment', $this->request);
}
```

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

## 4. deleteMediaModel 与 getMedia 的职责边界

两个方法都定义在 [Uploads.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php) Trait 中，但职责完全分离：

### getMedia - 媒体上传/创建
```php
// [Uploads.php#L12-L34](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L12-L34)
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
**职责**：接收上传文件，存储到磁盘，创建 `Media` 模型记录，返回该 Media 实例。

### deleteMediaModel - 媒体删除
```php
// [Uploads.php#L55-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Uploads.php#L55-L92)
public function deleteMediaModel($model, $parameter, $request = null)
{
    // 1. 获取模型关联的媒体
    $medias = $model->$parameter;
    
    // 2. 处理 uploaded_ 前缀的保留媒体（Web 端多图上传时保留已上传的）
    if ($request && isset($request['uploaded_' . $parameter])) {
        // 对比已上传媒体与当前媒体，保留匹配的，删除不匹配的
        foreach ($uploaded as $old_media) {
            $already_uploaded[] = $old_media['id'];
        }
    }
    
    // 3. 删除不在保留列表中的媒体
    foreach ((array) $medias as $media) {
        if (in_array($media->id, $already_uploaded)) {
            continue;
        }
        MediaModel::where('id', $media->id)->delete();
    }
}
```
**职责**：删除模型关联的媒体记录。支持通过 `uploaded_` 参数保留部分媒体（用于多图上传场景）。

### 职责边界对比表

| 维度 | getMedia | deleteMediaModel |
|------|----------|------------------|
| 核心职责 | 创建媒体、上传文件 | 删除媒体记录 |
| 输入 | UploadedFile 文件对象 | 模型实例 + 参数名 + 可选请求 |
| 输出 | Media 模型实例 | 无返回值 |
| 操作类型 | C（创建） | D（删除） |
| 磁盘操作 | 写入文件 | 无（仅删除 DB 记录，文件由观察者处理） |
| 调用时机 | 有新文件上传时 | 需要删除旧媒体时 |

## 5. API 客户端漏传 remove 标记导致旧附件保留的例子

### 场景描述
假设系统中有一张已上传附件的发票（ID = 123），API 客户端需要更新该发票的金额，但不需要保留原附件。

### 错误的 API 调用（漏传 remove_attachment）
```http
PUT /api/v1/invoices/123
Content-Type: application/json
Authorization: Bearer {token}

{
    "amount": 1500.00,
    "due_date": "2024-06-30"
}
```

**结果**：
- 发票金额和到期日更新成功 ✅
- 原有附件**被保留** ❌（因为没有传 `remove_attachment: true`）
- 不符合业务预期（用户可能以为更新就会清除旧附件）

### 正确的 API 调用（传 remove_attachment）
```http
PUT /api/v1/invoices/123
Content-Type: application/json
Authorization: Bearer {token}

{
    "amount": 1500.00,
    "due_date": "2024-06-30",
    "remove_attachment": true
}
```

**结果**：
- 发票金额和到期日更新成功 ✅
- 原有附件被删除 ✅
- 符合业务预期

### 另一个场景：替换附件时的隐式删除
如果 API 客户端上传了新文件，即使不传 `remove_attachment`，旧附件也会被删除：
```http
PUT /api/v1/invoices/123
Content-Type: multipart/form-data
Authorization: Bearer {token}

amount=1500.00&due_date=2024-06-30&attachment=@new_invoice.pdf
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

**总结**：API 端删除媒体有两种触发方式：
1. **显式删除**：传递 `remove_attachment: true` 或 `remove_picture: true`
2. **隐式替换**：上传新文件时自动删除旧文件
3. **遗漏风险**：既不上传新文件，也不传 remove 标记 → 旧媒体保留
