# Akaunting 权限与菜单显示联动机制分析（修订版）

> 所有结论均以代码为准，所有链接指向对应源码行号。

---

## 1. 权限 key 的形态与不同来源的生成规则

Akaunting 不存在"统一的 `{action}-{module}-{controller}` 规则"。实际代码中出现的 key 形态至少来自 5 条独立来源，各自有各自的命名约定。

### 1.1 来源一：Seeder 静态清单（核心权限）

**代码证据**：[Permissions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/Permissions.php#L28-L173)

Seeder 直接手写"角色 → 页面 → 动作缩写"的对照表，调用 `attachPermissionsByRoleNames($rows)`：

```php
// [database/seeds/Permissions.php#L29-L32]
'admin' => [
    'admin-panel' => 'r',
    'api'         => 'r',
    'auth-profile' => 'r,u',
    ...
]
```

最终由 [Permissions::applyPermissionsByAction](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L359-L374) 按动作展开：

```php
// [app/Traits/Permissions.php#L363-L373]
$actions_map = collect($this->getActionsMap());   // c=create, r=read, u=update, d=delete
$actions = explode(',', $action_list);
foreach ($actions as $short_action) {
    $action = $actions_map->get($short_action);
    $name   = $action . '-' . $page;              // 拼接最终 key
    $this->$function($role, $name);
}
```

**最终 key 形态**：`{action}-{page}`，其中 `{page}` 是 seeder 里的字符串：

- `admin-panel`  → `read-admin-panel`（无模块层）
- `auth-profile`  → `read-auth-profile`、`update-auth-profile`
- `reports-profit-loss` → `read-reports-profit-loss`
- `widgets-account-balance` → `read-widgets-account-balance`

### 1.2 来源二：`createModuleControllerPermission`（模块控制器权限）

**代码证据**：[createModuleControllerPermission](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L237-L247)

```php
// [app/Traits/Permissions.php#L243-L246]
$name         = $action . '-' . $module->getAlias() . '-' . $controller;
$display_name = Str::title($action) . ' ' . $module->getName() . ' ' . Str::title($controller);
return $this->createPermission($name, $display_name);
```

**最终 key 形态**：`{action}-{module_alias}-{controller}`

示例：`read-offline-payments-settings`、`update-offline-payments-settings`（由模块别名 `offline-payments` 的 settings 控制器生成）。

### 1.3 来源三：`createModuleReportPermission`（模块报表权限）

**代码证据**：[createModuleReportPermission](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L200-L214)

```php
// [app/Traits/Permissions.php#L210-L213]
$name         = Reports::getPermission($class);
$display_name = 'Read ' . $module->getName() . ' Reports ' . Reports::getDefaultName($class);
return $this->createPermission($name, $display_name);
```

key 完全由 `Reports::getPermission($class)` 决定。示例（来自 seeder）：`read-reports-income-summary`。

### 1.4 来源四：`createModuleWidgetPermission`（模块小部件权限）

**代码证据**：[createModuleWidgetPermission](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L216-L230)

```php
// [app/Traits/Permissions.php#L226-L229]
$name         = Widgets::getPermission($class);
$display_name = 'Read ' . $module->getName() . ' Widgets ' . Widgets::getDefaultName($class);
return $this->createPermission($name, $display_name);
```

key 完全由 `Widgets::getPermission($class)` 决定。示例（来自 seeder）：`read-widgets-bank-feeds`。

### 1.5 来源五：`assignPermissionsToController`（控制器运行时生成）

**代码证据**：[assignPermissionsToController](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L425-L500)

根据当前路由反解析出 controller 字符串，再注册 4 条中间件：

```php
// [app/Traits/Permissions.php#L459-L499]
$route = app(Route::class);
$arr   = array_reverse(explode('\\', explode('@', $route->getAction()['uses'])[0]));

// 注释给出的 4 种解析：
// App\Http\Controllers\FooBar                   -> foo-bar
// App\Http\Controllers\FooBar\Main              -> foo-bar-main
// Modules\Blog\Http\Controllers\Posts           -> blog-posts
// Modules\Blog\Http\Controllers\Portal\Posts    -> blog-portal-posts

$this->middleware('permission:create-' . $controller)->only('create', 'store', 'duplicate', 'import');
$this->middleware('permission:read-'   . $controller)->only('index', 'show', 'edit', 'export');
$this->middleware('permission:update-' . $controller)->only('update', 'enable', 'disable');
$this->middleware('permission:delete-' . $controller)->only('destroy');
```

**最终 key 形态**：`{action}-{namespace-derived-controller}`，可能携带模块前缀。示例：`read-blog-portal-posts`。

**关键陷阱**：`$skip = ['portal-dashboard']` 直接 `return`，不注册任何权限中间件。

### 1.6 权限 key 的创建/查找/删除方法

- **创建**：`createPermission($name)` [Permissions.php#L285-L295](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L285-L295) → `Permission::firstOrCreate(['name' => $name], ...)`

- **查找**：`Permission::where('name', 'xxx')->first()`、`Permission::action('read')->get()` [Permission.php#L57-L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Permission.php#L57-L60)

- **删除**：`detachPermission($role, $permission, $delete = true)` [Permissions.php#L310-L337](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L310-L337) — 当 `$delete = true` 时，`$role->detachPermission($permission)` 之后还会 `$permission->delete()` 物理删除 `permissions` 表记录。

- **改名**：`updatePermissionNames` [Permissions.php#L96-L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L96-L119) — 同时更新 `create/read/update/delete` 四种前缀的 name 与 display_name。

---

## 2. Role、Permission、User 与 `user()->can()` 的两条授权路径

### 2.1 表结构（`config/laratrust.php` 声明）

**代码证据**：[config/laratrust.php#L121-L151](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/laratrust.php#L121-L151)

```php
// [config/laratrust.php#L121-L151]
'tables' => [
    'roles'          => 'roles',
    'permissions'    => 'permissions',
    'role_user'      => 'user_roles',        // ← 注意不是 role_user
    'permission_user' => 'user_permissions', // ← 注意不是 permission_user
    'permission_role' => 'role_permissions', // ← 注意不是 permission_role
],
```

**迁移创建表**：[2017_09_14_000000_core_v1.php#L367-L421](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2017_09_14_000000_core_v1.php#L367-L421)

```
permissions       — id, name, display_name, description
roles             — id, name, display_name, description
role_permissions  — role_id (FK→roles), permission_id (FK→permissions)，联合主键
user_permissions  — user_id, permission_id (FK→permissions), user_type，联合主键
user_roles        — user_id, role_id (FK→roles), user_type，联合主键
```

`user_type` 是 Laratrust morphMap 多态关联（`use_morph_map => true`）：[laratrust.php#L21](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/laratrust.php#L21)

### 2.2 两条授权路径

**代码证据**：[app/Models/Auth/User.php#L24](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/User.php#L24)

```php
class User extends Authenticatable
{
    use LaratrustUserTrait;  // 提供 permissions()、roles()、can() 等方法
}
```

`can()` 生效的前提：`permissions_as_gates => true` [laratrust.php#L276](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/laratrust.php#L276)。

**路径一（间接授权）**：通过角色

```
user → user_roles(user_id, role_id, user_type)
     → role_permissions(role_id, permission_id)
     → permissions
```

**路径二（直接授权）**：用户直接挂 Permission

```
user → user_permissions(user_id, permission_id, user_type)
     → permissions
```

`user()->can('key')` 会合并两条路径的权限并集，再按 `config('laratrust.checker')` 决定使用 `default` 或 `query` 检查器执行：[laratrust.php#L36](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/laratrust.php#L36)

### 2.3 直接授权入口：`CreateUser`

**代码证据**：[app/Jobs/Auth/CreateUser.php#L43-L49](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php#L43-L49)

```php
if ($this->request->has('permissions')) {
    $this->model->permissions()->attach($this->request->get('permissions'));
}
if ($this->request->has('roles')) {
    $this->model->roles()->attach($this->request->get('roles'));
}
```

创建用户时可同时 `attach` 角色与直接权限。`permissions()` 与 `roles()` 由 `LaratrustUserTrait` 提供，分别写入 `user_permissions` 与 `user_roles` 表。

### 2.4 排查 SQL（修正表名）

```sql
-- 查看用户的直接权限（路径二）
SELECT p.name
FROM permissions p
JOIN user_permissions up ON up.permission_id = p.id
WHERE up.user_id = ? AND up.user_type = 'App\\Models\\Auth\\User';

-- 查看用户通过角色获得的权限（路径一）
SELECT p.name
FROM permissions p
JOIN role_permissions rp ON rp.permission_id = p.id
JOIN user_roles ur ON ur.role_id = rp.role_id
WHERE ur.user_id = ? AND ur.user_type = 'App\\Models\\Auth\\User';

-- 合并两条路径的全部权限
SELECT p.name FROM permissions p
JOIN user_permissions up ON up.permission_id = p.id WHERE up.user_id = ?
UNION
SELECT p.name FROM permissions p
JOIN role_permissions rp ON rp.permission_id = p.id
JOIN user_roles ur ON ur.role_id = rp.role_id WHERE ur.user_id = ?;
```

---

## 3. Menu Listeners 与菜单区域的事件映射

### 3.1 事件-监听器注册表

**代码证据**：[app/Providers/Event.php#L93-L110](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php#L93-L110)

```php
\App\Events\Menu\AdminCreated::class         => [\App\Listeners\Menu\ShowInAdmin::class],
\App\Events\Menu\SettingsCreated::class      => [\App\Listeners\Menu\ShowInSettings::class],
\App\Events\Menu\ProfileCreated::class       => [\App\Listeners\Menu\ShowInProfile::class],
\App\Events\Menu\NewwCreated::class          => [\App\Listeners\Menu\ShowInNeww::class],
\App\Events\Menu\PortalCreated::class        => [\App\Listeners\Menu\ShowInPortal::class],
\App\Events\Menu\NotificationsCreated::class => [\App\Listeners\Menu\ShowInNotifications::class],
```

### 3.2 各区域菜单构建触发点

- **Admin 菜单**：由 `menu.admin` 中间件触发 → [app/Http/Middleware/AdminMenu.php#L25-L31](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Middleware/AdminMenu.php#L25-L31)，注册在 `admin` 路由组 [Kernel.php#L76-L88](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L76-L88) 和 [routes/common.php#L19-L25](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/common.php#L19-L25)

- **Portal 菜单**：由 `menu.portal` 中间件触发，注册在 `portal` 路由组 [Kernel.php#L100-L109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L100-L109) 和 [routes/common.php#L27-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/common.php#L27-L29)

- **Settings 菜单**：Livewire 组件 `render()` 时构建 → [app/Http/Livewire/Menu/Settings.php#L28-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L28-L50)，无中间件。

### 3.3 各 Listener 的权限检查方式

| Listener | 菜单区域 | 权限检查方式 |
|-----------|---------|-------------|
| [ShowInAdmin](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInAdmin.php) | 主导航栏 | `$this->canAccessMenuItem($title, $permissions)` |
| [ShowInSettings](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInSettings.php) | 设置页面 | `$this->canAccessMenuItem($title, $permissions)` |
| [ShowInProfile](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInProfile.php) | 用户下拉 | `$this->canAccessMenuItem($title, $permissions)` |
| [ShowInNeww](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInNeww.php) | 快捷新建 | `$this->canAccessMenuItem($title, $permissions)` |
| [ShowInPortal](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInPortal.php) | 客户门户 | **直接 `$user->can('key')`**（不走 `canAccessMenuItem`） |

`ShowInPortal` 直接 `$user->can()` 不走 `canAccessMenuItem`，绕过 `ItemAuthorizing` 事件。

---

## 4. 菜单显示 ≠ 路由可访问：核心原因拆解

### 4.1 `canAccessMenuItem` 的 OR 语义与 ItemAuthorizing 扩展点

**代码证据**：[app/Traits/Permissions.php#L502-L513](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L502-L513)

```php
public function canAccessMenuItem($title, $permissions)
{
    $permissions = Arr::wrap($permissions);          // 字符串/数组统一成数组

    $item = new \stdClass();
    $item->title       = $title;
    $item->permissions = $permissions;

    event(new \App\Events\Menu\ItemAuthorizing($item)); // 扩展点：模块可修改 $item->permissions

    return user()->canAny($item->permissions);          // 最终用 OR 语义检查
}
```

**关键点**：

1. `Arr::wrap` 把字符串包成数组，后续 `canAny` 天然是 **OR** 语义。
2. `canAny(['A', 'B'])`：用户具备其中任一项即返回 `true`。
3. `ItemAuthorizing` 事件在核心代码中**没有任何监听器**。搜索 `ItemAuthorizing` 仅命中 [app/Traits/Permissions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php) 和 [app/Events/Menu/ItemAuthorizing.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Menu/ItemAuthorizing.php)；[Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php) 的 `$listen` 数组中没有 `ItemAuthorizing::class => [...]` 注册。因此它在核心里只是扩展点，由模块/插件在安装时动态挂载自己的监听器来替换/加严 `$item->permissions`。

### 4.2 路由权限检查的三种绑定方式

**路径一：控制器自动（`assignPermissionsToController`）**

```php
// [app/Traits/Permissions.php#L496-L499]
$this->middleware('permission:create-' . $controller)->only('create', 'store', ...);
$this->middleware('permission:read-'   . $controller)->only('index', ...);
```

`LaratrustPermission` 中间件支持 `|` 分隔的 **OR** 语义，即 `permission:A|B` 表示用户具备 A 或 B 任一即可通过。

**路径二：控制器手工覆盖（组合 OR 权限）**

**代码证据**：[app/Http/Controllers/Auth/Users.php#L22-L29](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Users.php#L22-L29)

```php
$this->middleware('permission:create-auth-users')->only('create', 'store', 'duplicate', 'import');
$this->middleware('permission:read-auth-users')->only('index', 'show', 'export');
$this->middleware('permission:update-auth-users')->only('enable', 'disable');
$this->middleware('permission:delete-auth-users')->only('destroy');

$this->middleware('permission:read-auth-users|read-auth-profile')->only('edit');    // 组合 OR
$this->middleware('permission:update-auth-users|update-auth-profile')->only('update'); // 组合 OR
```

**路径三：路由组级权限**

**代码证据**：[routes/common.php#L19](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/common.php#L19) 和 [Kernel.php#L50-L60](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L50-L60)

```php
// routes/common.php#L19
Route::group(['middleware' => ['permission:read-admin-panel']], function () { ... });

// Kernel.php#L50-L60 的 `api` 路由组
'api' => [
    ...
    'permission:read-api',
    ...
],
```

### 4.3 核心控制器使用组合 OR 权限的证据

**代码证据**：[app/Http/Controllers/Api/Settings/Settings.php#L21-L23](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Settings/Settings.php#L21-L23)

```php
$this->middleware('permission:create-settings-company|create-settings-defaults|create-settings-localisation')
     ->only('create', 'store', 'duplicate', 'import');
$this->middleware('permission:read-settings-company|read-settings-defaults|read-settings-localisation')
     ->only('index', 'show', 'edit', 'export');
$this->middleware('permission:update-settings-company|update-settings-defaults|update-settings-localisation')
     ->only('update', 'enable', 'disable', 'destroy');
```

**代码证据**：[app/Http/Controllers/Common/Contacts.php#L16](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Common/Contacts.php#L16)

```php
$this->middleware('permission:read-sales-customers|read-purchases-vendors')->only('index');
```

### 4.4 菜单显示与路由可访问不等价的具体场景

**场景 A：有路由权限但菜单不显示**

| 可达条件 | 代码证据 |
|-----------|-----------|
| 菜单用 `canAny(['A', 'B'])`，用户只拥有 `A`，路由只需要 `A` 或 `B` 中之一（菜单 OR 权限 ≥ 路由权限） | [ShowInAdmin.php#L41](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInAdmin.php#L41) `$this->canAccessMenuItem($title, ['read-sales-invoices', 'read-sales-customers'])` — 用户只有 `read-sales-invoices` 能进 `invoices.index` 路由（路由中间件只需要 `read-sales-invoices`），菜单 `canAny` 也显示 |
| 模块/插件挂载了 `ItemAuthorizing` 监听器，把 `$item->permissions` 加严 | [Permissions.php#L510](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php#L510) `event(new ItemAuthorizing($item))` — 模块在运行时动态添加额外权限 |
| 设置菜单搜索关键词不匹配 | [Settings.php#L81-L88](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L81-L88) `availableInSearch()` 返回 false 时 `removeByTitle` 移除菜单项 |
| 模块 JSON 配置中 `settings` 键为空 | [Settings.php#L63-L66](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L63-L66) `if (empty($module->get('settings'))) { continue; }` |

**场景 B：菜单显示但路由不可访问**

| 可达条件 | 代码证据 |
|-----------|-----------|
| ShowInPortal 直接 `$user->can('read-portal-invoices')`，不经过 `ItemAuthorizing`，但 `portal` 路由组额外要求 `permission:read-client-portal` [Kernel.php#L100-L109](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L100-L109) | [ShowInPortal.php#L28](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInPortal.php#L28) |
| 菜单检查的权限 key 与路由中间件实际需要的 key 不一致（菜单 `read-A`，路由中间件 `read-B`，用户只有 `read-A`） | 需要逐菜单核对 |

**场景 C：API 路由可达但设置菜单不可见**

| 可达条件 | 代码证据 |
|-----------|-----------|
| 设置菜单搜索关键词不匹配时被 `removeByTitle` 移除 | [Settings.php#L41-L46](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L41-L46) |
| API 路由组 `permission:read-api` 在 [Kernel.php#L54](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php#L54) 但 UI 菜单检查 `read-common-items` 等更细粒度的 key | 用户有 `read-api` 但没有 `read-common-items`，API 可达但 UI 菜单不可见 |

---

## 5. 设置模块菜单的排查路径

### 5.1 设置菜单构建完整流程

**代码证据**：[app/Http/Livewire/Menu/Settings.php#L28-L50](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L28-L50)

```php
menu()->create('settings', function ($menu) {
    $menu->style('tailwind');

    event(new SettingsCreating($menu));   // 扩展点：模块可在创建前修改
    event(new SettingsCreated($menu));    // ShowInSettings 添加核心设置项

    $this->addSettingsOfModulesFromJsonFile($menu);  // 模块设置项

    foreach ($menu->getItems() as $item) {
        if ($item->isActive()) {
            $this->active_menu = 1;
        }
        if ($this->availableInSearch($item)) {
            continue;
        }
        $menu->removeByTitle($item->title);  // 搜索不匹配 → 移除
    }

    event(new SettingsFinished($menu));  // 扩展点：最终可修改
});
```

### 5.2 模块设置菜单项添加条件

**代码证据**：[addSettingsOfModulesFromJsonFile](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L55-L79)

```php
// [Settings.php#L55-L79]
public function addSettingsOfModulesFromJsonFile($menu): void
{
    $enabled_modules = module()->allEnabled();
    $order = 110;

    foreach ($enabled_modules as $module) {
        // 条件 1：模块 JSON 配置里 settings 键非空
        if (empty($module->get('settings'))) {
            continue;
        }
        // 条件 2：当前用户有 read-{alias}-settings 权限
        if ($this->user->cannot('read-' . $module->getAlias() . '-settings')) {
            continue;
        }
        $menu->route('settings.module.edit', $module->getName(),
            ['alias' => $module->getAlias()],
            $module->get('setting_order', $order),
            [
                'icon' => $module->get('icon', 'custom-akaunting'),
                'search_keywords' => $module->getDescription(),
            ]
        );
        $order += 10;
    }
}
```

**模块设置菜单显示需同时满足**：

1. 模块已启用 (`module()->allEnabled()`)
2. 模块 JSON 配置里 `settings` 键非空（来自 `module.json` 或 `module.php` 配置）
3. 当前用户拥有 `read-{module_alias}-settings` 权限

### 5.3 搜索过滤逻辑

**代码证据**：[availableInSearch](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L81-L88) 和 [search](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php#L90-L114)

```php
// [Settings.php#L81-L88]
public function availableInSearch($item): bool
{
    if (empty($this->keyword)) {
        return true;  // 关键词为空 → 全部可见
    }
    return $this->search($item);
}

// [Settings.php#L90-L114]
public function search($item): bool
{
    $status = false;
    $keywords = explode(' ', $this->keyword);
    foreach ($keywords as $keyword) {
        if (Str::contains(Str::lower($item->title), Str::lower($keyword))) {
            $status = true;
            break;
        }
        if (!empty($item->attributes['search_keywords'])
            && Str::contains(Str::lower($item->attributes['search_keywords']), Str::lower($keyword))) {
            $status = true;
            break;
        }
    }
    return $status;
}
```

关键词为空 → 全部保留；关键词非空 → 对每个关键词匹配 `item->title` 或 `item->attributes['search_keywords']`，任一关键词匹配即保留；不匹配则 `removeByTitle` 移除。

### 5.4 排查路径（修正）

**有路由权限但设置菜单不显示：**

```
1. 区分核心设置项 vs 模块设置项
   ├─ 核心设置项 → ShowInSettings.php 检查 canAccessMenuItem 权限
   └─ 模块设置项 → Settings.php addSettingsOfModulesFromJsonFile

2. 核心设置项排查
   ├─ 检查用户是否有 read-settings-company 等权限（两条路径 SQL）
   └─ 检查 ItemAuthorizing 扩展监听器（仅模块/插件会挂载）

3. 模块设置项排查
   ├─ module()->allEnabled() 确认模块启用
   ├─ $module->get('settings') 非空
   ├─ $user->can('read-{alias}-settings')
   └─ 当前搜索关键词是否过滤掉该项

4. 确认路由可达性
   └─ 直接访问 settings.module.edit/{alias} 路由看中间件检查
```

**菜单显示但设置路由不可访问：**

```
1. 确认路由定义
   ├─ 核心设置路由 → settings.company.edit 等
   └─ 模块设置路由 → settings.module.edit

2. 路由权限检查
   ├─ admin 路由组的 permission:read-admin-panel [Kernel.php#L85]
   └─ 控制器中间件 permission:read-{module_alias}-settings 等

3. 权限 key 对比
   ├─ 菜单检查的 key vs 路由中间件实际需要的 key
   └─ 两条授权路径（user_permissions、user_roles+role_permissions）是否都缺失
```

---

## 核心文件索引

| 文件 | 用途 |
|------|------|
| [app/Traits/Permissions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Traits/Permissions.php) | 权限创建、分配、菜单权限检查核心 trait |
| [app/Models/Auth/Permission.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Permission.php) | 权限模型 |
| [app/Models/Auth/Role.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/Role.php) | 角色模型 |
| [app/Models/Auth/User.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Models/Auth/User.php) | 用户模型（LaratrustUserTrait） |
| [app/Events/Menu/ItemAuthorizing.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Events/Menu/ItemAuthorizing.php) | 菜单项授权事件（核心无监听器，仅扩展点） |
| [app/Listeners/Menu/ShowInAdmin.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInAdmin.php) | 后台主菜单监听器 |
| [app/Listeners/Menu/ShowInSettings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInSettings.php) | 设置菜单监听器 |
| [app/Listeners/Menu/ShowInPortal.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Listeners/Menu/ShowInPortal.php) | 门户菜单监听器（直接 `$user->can()`） |
| [app/Providers/Event.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Providers/Event.php) | 事件-监听器注册 |
| [app/Http/Kernel.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Kernel.php) | 路由组中间件（含 `permission:read-admin-panel` / `read-client-portal` / `read-api`） |
| [app/Http/Livewire/Menu/Settings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Livewire/Menu/Settings.php) | 设置菜单 Livewire 组件（含搜索过滤） |
| [config/laratrust.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/config/laratrust.php) | Laratrust 配置（表名映射、permissions_as_gates） |
| [database/seeds/Permissions.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/seeds/Permissions.php) | 核心权限种子数据 |
| [app/Jobs/Auth/CreateUser.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Jobs/Auth/CreateUser.php) | 创建用户（直接权限 & 角色同时 attach） |
| [app/Http/Controllers/Auth/Users.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Auth/Users.php) | 用户控制器（edit/update 使用组合 OR 权限） |
| [app/Http/Controllers/Common/Contacts.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Common/Contacts.php) | Contacts 控制器（组合 OR 权限） |
| [app/Http/Controllers/Api/Settings/Settings.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/app/Http/Controllers/Api/Settings/Settings.php) | API 设置控制器（三项组合 OR 权限） |
| [routes/common.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/routes/common.php) | 通用路由（路由组权限） |
| [database/migrations/2017_09_14_000000_core_v1.php](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/akaunting/database/migrations/2017_09_14_000000_core_v1.php) | 核心表迁移（permissions、user_roles、user_permissions、role_permissions） |
