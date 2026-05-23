# Akaunting 权限与菜单显示联动机制分析

## 1. 权限 key 的创建、查找与删除机制

### 1.1 权限 key 的命名规范

Akaunting 的权限 key 遵循统一的命名模式：`{action}-{module}-{controller}`

- **action**: `create` / `read` / `update` / `delete`（对应 CRUD 操作）
- **module**: 模块别名（如 `sales`、`purchases`、`banking` 等）
- **controller**: 控制器名称（如 `invoices`、`customers`、`bills` 等）

示例：
- `read-sales-invoices` - 读取销售发票
- `create-purchases-bills` - 创建采购账单
- `update-settings-company` - 更新公司设置

### 1.2 权限 key 的创建

权限 key 的创建主要通过 [Permissions](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php) trait 实现：

**核心方法：**

```php
// [Permissions.php#L285-L295] 创建单个权限
public function createPermission($name, $display_name = null, $description = null)
{
    $display_name = $display_name ?? $this->getPermissionDisplayName($name);
    return Permission::firstOrCreate([
        'name' => $name,
    ], [
        'display_name' => $display_name,
        'description' => $description ?? $display_name,
    ]);
}
```

**批量创建场景：**

1. **按角色批量附加权限** [Permissions.php#L28-L37]
   ```php
   public function attachPermissionsByRoleNames($roles)
   {
       foreach ($roles as $role_name => $permissions) {
           $role = $this->createRole($role_name);
           foreach ($permissions as $id => $permission) {
               $this->attachPermissionsByAction($role, $id, $permission);
           }
       }
   }
   ```

2. **按操作缩写批量创建** [Permissions.php#L349-L374]
   - 支持简写操作符：`c`=create, `r`=read, `u`=update, `d`=delete
   - 例如：`attachPermissionsByAction($role, 'sales-invoices', 'c,r,u,d')` 会创建四个权限

3. **模块权限自动创建**
   - 报表权限：`createModuleReportPermission()` [Permissions.php#L200-L214]
   - 小部件权限：`createModuleWidgetPermission()` [Permissions.php#L216-L230]
   - 设置权限：`createModuleSettingPermission()` [Permissions.php#L232-L235]

### 1.3 权限 key 的查找

权限查找主要通过 [Permission](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Permission.php) 模型：

```php
// 按名称查找
$permission = Permission::where('name', 'read-sales-invoices')->first();

// 按操作类型筛选 [Permission.php#L57-L60]
public function scopeAction($query, $action = 'read')
{
    return $query->where('name', 'like', $action . '-%');
}
```

### 1.4 权限 key 的删除

权限删除通过 `detachPermission` 方法 [Permissions.php#L310-L337]：

```php
public function detachPermission($role, $permission, $delete = true)
{
    // 1. 查找角色
    // 2. 查找权限
    // 3. 从角色中分离权限
    // 4. 如果 $delete = true，则删除权限记录本身
    if ($delete === false) {
        return;
    }
    $permission->delete();
}
```

**批量删除：**
- `detachPermissionsFromAdminRoles()` - 从所有管理角色中分离
- `detachPermissionsFromPortalRoles()` - 从门户角色中分离
- `detachPermissionsFromAllRoles()` - 从所有符合条件的角色中分离

---

## 2. Role 与 Permission 的关系对 user can/cannot 的影响

### 2.1 数据模型关系

Akaunting 基于 Laratrust 权限包实现多对多关系：

- [Role](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Role.php) 与 Permission：多对多（通过 `permission_role` 表）
- [User](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/User.php) 与 Role：多对多（通过 `role_user` 表）

```php
// User.php 使用 LaratrustUserTrait
use Laratrust\Traits\LaratrustUserTrait;

class User extends Authenticatable
{
    use LaratrustUserTrait;
    
    public function roles()
    {
        return $this->belongsToMany(role_model_class(), 'App\Models\Auth\UserRole');
    }
}
```

### 2.2 权限检查流程

当调用 `user()->can('permission-key')` 时，Laratrust 内部执行以下检查：

1. **获取用户所有角色**：通过 `role_user` 关联表
2. **获取角色所有权限**：通过 `permission_role` 关联表
3. **权限匹配**：检查目标权限是否在用户权限集合中

### 2.3 内置角色与默认权限

| 角色 | 标识 | 典型权限 | 用途 |
|------|------|---------|------|
| 管理员 | `admin` | 所有权限 | 系统管理员 |
| 经理 | `manager` | 大部分业务权限 | 业务经理 |
| 员工 | `employee` | 有限的业务权限 | 普通员工 |
| 客户 | `customer` | `read-client-portal` 等 | 前台门户用户 |

**权限判定示例：**
```php
// User.php#L300-L303
public function isCustomer()
{
    return (bool) $this->can('read-client-portal');
}

// User.php#L310-L313
public function isNotCustomer()
{
    return (bool) $this->can('read-admin-panel');
}
```

### 2.4 权限在控制器中的应用

通过 `assignPermissionsToController()` 方法自动绑定中间件 [Permissions.php#L425-L500]：

```php
public function assignPermissionsToController()
{
    // 解析控制器名称为权限前缀
    // App\Http\Controllers\Sales\Invoices -> sales-invoices
    
    // 自动绑定 CRUD 权限中间件
    $this->middleware('permission:create-' . $controller)->only('create', 'store', 'duplicate', 'import');
    $this->middleware('permission:read-' . $controller)->only('index', 'show', 'edit', 'export');
    $this->middleware('permission:update-' . $controller)->only('update', 'enable', 'disable');
    $this->middleware('permission:delete-' . $controller)->only('destroy');
}
```

---

## 3. Menu Listeners 与菜单区域的事件映射

### 3.1 菜单事件体系

Akaunting 采用事件驱动的菜单构建机制，在 [Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php) 中注册了完整的事件-监听器映射：

| 事件 | 触发时机 | 监听器 | 菜单区域 |
|------|---------|--------|---------|
| `AdminCreated` | 后台菜单创建完成 | [ShowInAdmin](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInAdmin.php) | 主导航栏 |
| `SettingsCreated` | 设置菜单创建完成 | [ShowInSettings](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInSettings.php) | 设置页面 |
| `ProfileCreated` | 用户菜单创建完成 | [ShowInProfile](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInProfile.php) | 用户下拉菜单 |
| `NewwCreated` | 新建菜单创建完成 | [ShowInNeww](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInNeww.php) | 快捷新建菜单 |
| `PortalCreated` | 门户菜单创建完成 | [ShowInPortal](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInPortal.php) | 客户门户 |
| `NotificationsCreated` | 通知菜单创建完成 | `ShowInNotifications` | 通知中心 |

### 3.2 菜单构建流程

**后台菜单构建流程 [AdminMenu.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/AdminMenu.php)：**

```php
public function handle($request, Closure $next)
{
    menu()->create('admin', function ($menu) {
        $menu->style('tailwind');
        event(new AdminCreating($menu));   // 构建前事件
        event(new AdminCreated($menu));    // 构建事件 - 监听器添加菜单项
    });
    return $next($request);
}
```

**设置菜单构建流程 [Settings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php)：**

```php
menu()->create('settings', function ($menu) {
    $menu->style('tailwind');
    event(new SettingsCreating($menu));
    event(new SettingsCreated($menu));    // ShowInSettings 监听器在此添加项
    
    // 额外添加模块设置（从 JSON 配置读取）
    $this->addSettingsOfModulesFromJsonFile($menu);
    
    event(new SettingsFinished($menu));
});
```

### 3.3 各区域菜单项示例

**ShowInAdmin - 主导航栏：**
```php
// 仪表盘
if ($this->canAccessMenuItem($title, 'read-common-dashboards')) {
    $menu->route('dashboard', $title, [], 10, ['icon' => 'speed']);
}

// 销售下拉菜单
if ($this->canAccessMenuItem($title, ['read-sales-invoices', 'read-sales-customers'])) {
    $menu->dropdown($title, function ($sub) {
        // 子菜单项...
    }, 30, ['icon' => 'payments']);
}
```

**ShowInSettings - 设置页面：**
```php
// 公司设置
if ($this->canAccessMenuItem($title, 'read-settings-company')) {
    $menu->route('settings.company.edit', $title, [], 10, ['icon' => 'business']);
}

// 本地化设置
if ($this->canAccessMenuItem($title, 'read-settings-localisation')) {
    $menu->route('settings.localisation.edit', $title, [], 20, ['icon' => 'flag']);
}
```

---

## 4. ItemAuthorizing 事件与菜单/路由权限的不等价性

### 4.1 ItemAuthorizing 事件机制

[ItemAuthorizing](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Menu/ItemAuthorizing.php) 是菜单显示前的授权事件，在 `canAccessMenuItem()` 方法中触发 [Permissions.php#L502-L513]：

```php
public function canAccessMenuItem($title, $permissions)
{
    $permissions = Arr::wrap($permissions);
    
    $item = new \stdClass();
    $item->title = $title;
    $item->permissions = $permissions;
    
    // 触发 ItemAuthorizing 事件，允许其他监听器修改权限要求
    event(new \App\Events\Menu\ItemAuthorizing($item));
    
    // 使用可能被修改后的权限进行检查
    return user()->canAny($item->permissions);
}
```

### 4.2 为什么菜单显示 ≠ 路由可访问

| 维度 | 菜单显示检查 | 路由访问检查 |
|------|-------------|-------------|
| **触发时机** | 菜单渲染时（事件驱动） | 路由中间件执行时 |
| **检查位置** | `canAccessMenuItem()` 方法 | Laratrust `permission` 中间件 |
| **可扩展性** | 通过 `ItemAuthorizing` 事件可动态修改权限 | 静态绑定在路由/控制器 |
| **权限粒度** | 可组合多个权限（OR 逻辑） | 单一权限匹配 |
| **修改点** | 事件监听器可在运行时调整 | 需修改路由/控制器代码 |

### 4.3 不等价性的具体表现

**场景 1：有路由权限但菜单不显示**

原因：
- `ItemAuthorizing` 事件监听器动态添加了额外的权限要求
- 菜单权限使用 `canAny(['A', 'B'])` 组合检查，而路由只需要 `A`
- 菜单被其他逻辑条件隐藏（如功能开关、用户层级判断）

**场景 2：菜单显示但无路由权限**

原因：
- 菜单项没有权限检查（直接添加，未经过 `canAccessMenuItem`）
- 路由权限被中间件单独加强，但菜单检查用了更宽松的权限
- 模块/插件注册的菜单项权限与实际路由权限不匹配

**示例 - ShowInPortal 中的直接权限检查：**
```php
// ShowInPortal.php#L28-L30 - 不经过 ItemAuthorizing 事件
if ($user->can('read-portal-invoices')) {
    $menu->route('portal.invoices.index', $title, [], 20, ['icon' => 'description']);
}
```

---

## 5. 权限与菜单不一致的排查路径

### 5.1 用户有路由权限但菜单不展示

**排查步骤：**

```
1. 确认用户确实有路由权限
   ├─ 直接访问 URL，看是否能进入（排除中间件问题）
   ├─ 检查用户角色：SELECT * FROM role_user WHERE user_id = ?
   ├─ 检查角色权限：SELECT * FROM permission_role WHERE role_id IN (?)
   └─ 确认权限 key：SELECT * FROM permissions WHERE name = 'read-xxx-yyy'

2. 定位菜单定义位置
   ├─ 查找菜单项属于哪个 listener：ShowInAdmin / ShowInSettings / ShowInNeww 等
   └─ 检查该菜单项的权限检查逻辑

3. 检查 canAccessMenuItem 调用
   ├─ 查看权限参数是字符串还是数组
   ├─ 如果是数组，确认用户是否拥有其中任意一个权限
   └─ 检查 ItemAuthorizing 事件是否有监听器修改了权限

4. 检查 ItemAuthorizing 事件监听器
   ├─ 搜索项目中所有监听 ItemAuthorizing 的类
   ├─ 检查是否有逻辑动态添加/移除了权限
   └─ 确认是否有基于业务逻辑的动态隐藏

5. 检查其他隐藏条件
   ├─ 是否有功能开关配置
   ├─ 是否有公司/租户级别的限制
   └─ 是否有用户层级/部门的限制
```

**关键代码检查点：**
- [Permissions.php#L502-L513] - `canAccessMenuItem` 方法
- 各 Menu Listener 中的 `if ($this->canAccessMenuItem(...))` 判断

### 5.2 菜单展示但用户无路由权限

**排查步骤：**

```
1. 确认菜单项确实显示
   ├─ 登录目标用户账号查看菜单
   └─ 检查菜单构建时的权限检查逻辑

2. 检查菜单项是否有正确的权限检查
   ├─ 是否使用了 canAccessMenuItem()
   ├─ 是否直接使用了 user()->can() 但权限 key 错误
   └─ 是否完全没有权限检查（直接添加菜单项）

3. 对比菜单权限与路由权限
   ├─ 菜单项检查的权限 key：例如 'read-sales-invoices'
   ├─ 路由实际需要的权限：查看控制器中间件
   └─ 确认两者是否一致

4. 检查路由权限中间件
   ├─ 查看路由定义：route:list | grep 目标路由
   ├─ 检查控制器是否调用了 assignPermissionsToController()
   └─ 确认权限 key 生成是否正确（控制器解析逻辑）

5. 检查模块/插件注册
   ├─ 是否是模块通过 JSON 配置注册的菜单项
   ├─ 检查模块 setting_order、icon、permission 配置
   └─ 查看 Settings.php 中 addSettingsOfModulesFromJsonFile() 逻辑
```

**关键代码检查点：**
- [Settings.php#L55-L79] - 模块设置菜单项添加逻辑
- [Permissions.php#L425-L500] - 控制器权限自动分配逻辑
- 各 Menu Listener 中是否遗漏权限检查

### 5.3 快速诊断命令

```bash
# 查看用户拥有的所有权限
php artisan tinker
>>> $user = App\Models\Auth\User::find(USER_ID);
>>> $user->allPermissions()->pluck('name');

# 查看用户角色
>>> $user->roles()->pluck('name');

# 检查权限是否存在
>>> App\Models\Auth\Permission::where('name', 'LIKE', '%invoices%')->pluck('name');

# 测试权限检查
>>> $user->can('read-sales-invoices');
>>> $user->canAny(['read-sales-invoices', 'read-sales-customers']);
```

---

## 核心文件索引

| 文件 | 用途 |
|------|------|
| [app/Traits/Permissions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php) | 权限创建、分配、菜单权限检查核心 trait |
| [app/Models/Auth/Permission.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Permission.php) | 权限模型 |
| [app/Models/Auth/Role.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Role.php) | 角色模型 |
| [app/Models/Auth/User.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/User.php) | 用户模型（使用 LaratrustUserTrait） |
| [app/Events/Menu/ItemAuthorizing.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Menu/ItemAuthorizing.php) | 菜单项授权事件 |
| [app/Listeners/Menu/ShowInAdmin.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInAdmin.php) | 后台主菜单监听器 |
| [app/Listeners/Menu/ShowInSettings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInSettings.php) | 设置菜单监听器 |
| [app/Providers/Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php) | 事件-监听器注册 |
| [app/Http/Middleware/AdminMenu.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/AdminMenu.php) | 后台菜单构建中间件 |
| [app/Http/Livewire/Menu/Settings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php) | 设置菜单 Livewire 组件 |
