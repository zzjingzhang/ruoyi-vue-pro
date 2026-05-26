# RuoYi-Vue-Pro RBAC 数据模型深度分析

## 一、RBAC 数据模型整体架构

### 1.1 核心数据表关系

```
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│   system_user    │       │   system_role    │       │   system_menu    │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ id (PK)          │       │ id (PK)          │       │ id (PK)          │
│ username         │       │ name             │       │ name             │
│ password         │       │ code             │       │ permission       │
│ status           │       │ type             │       │ type             │
│ dept_id          │       │ status           │       │ path             │
│ ...              │       │ data_scope       │       │ component        │
└────────┬─────────┘       │ ...              │       │ icon             │
         │                 └────────┬─────────┘       │ visible          │
         │  ┌──────────────────────┐ │                 │ keepAlive        │
         └──┤  system_user_role   ├─┘                 │ alwaysShow       │
            ├──────────────────────┤                   │ parent_id        │
            │ user_id (FK)        │                   └────────┬─────────┘
            │ role_id (FK)        │                            │
            └──────────────────────┘                            │
                     ┌──────────────────────┐                  │
                     │  system_role_menu   ├──────────────────┘
                     ├──────────────────────┤
                     │ role_id (FK)        │
                     │ menu_id (FK)        │
                     └──────────────────────┘
```

### 1.2 数据模型定义文件

| 数据对象 | 文件路径 | 说明 |
|---------|---------|------|
| `MenuDO` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/permission/MenuDO.java` | 菜单表结构 |
| `RoleDO` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/permission/RoleDO.java` | 角色表结构 |
| `RoleMenuDO` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/permission/RoleMenuDO.java` | 角色-菜单关联表 |
| `UserRoleDO` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/permission/UserRoleDO.java` | 用户-角色关联表 |

---

## 二、菜单类型详解

### 2.1 菜单类型枚举

菜单类型在 `MenuTypeEnum.java` 中定义：

```java
public enum MenuTypeEnum {
    DIR(1),    // 目录
    MENU(2),   // 菜单
    BUTTON(3)  // 按钮
}
```

### 2.2 不同菜单类型的影响

#### 2.2.1 目录 (DIR, type=1)

| 影响维度 | 具体表现 |
|---------|---------|
| **前端路由** | 作为路由容器，通常使用 `Layout` 或 `ParentView` 组件 |
| **侧边栏** | 在侧边栏中作为可展开的菜单项显示 |
| **按钮显隐** | 不直接影响按钮显隐，作为菜单分组容器 |
| **后端鉴权** | 通常不设置 `permission` 字段，不参与接口权限校验 |
| **特殊属性** | 支持 `visible` (是否可见)、`alwaysShow` (是否总是显示) |

**前端路由转换示例：**
```typescript
// 目录类型菜单转换为路由
{
  path: '/system',
  component: () => import('@/layout/index.vue'),
  meta: {
    title: '系统管理',
    icon: 'system',
    hidden: menu.visible === false
  },
  children: [...] // 子菜单
}
```

#### 2.2.2 菜单 (MENU, type=2)

| 影响维度 | 具体表现 |
|---------|---------|
| **前端路由** | 作为具体页面路由，使用 `component` 字段指定页面组件 |
| **侧边栏** | 在侧边栏中显示为可点击的菜单项 |
| **按钮显隐** | 不直接控制按钮，但 `permission` 字段可作为页面级权限 |
| **后端鉴权** | `permission` 字段可用于接口权限校验 (@PreAuthorize) |
| **特殊属性** | 支持 `keepAlive` (是否缓存)、`componentName` (组件名) |

**前端路由转换示例：**
```typescript
// 菜单类型转换为路由
{
  path: '/system/user',
  name: 'SystemUser',
  component: () => import('@/views/system/user/index.vue'),
  meta: {
    title: '用户管理',
    icon: 'user',
    keepAlive: menu.keepAlive,
    permission: menu.permission
  }
}
```

#### 2.2.3 按钮 (BUTTON, type=3)

| 影响维度 | 具体表现 |
|---------|---------|
| **前端路由** | **不参与** 路由生成，前端转换时会过滤掉 type=3 的菜单 |
| **侧边栏** | **不显示** 在侧边栏中 |
| **按钮显隐** | `permission` 字段用于前端按钮 `v-hasPermi` 或 `v-if` 条件判断 |
| **后端鉴权** | `permission` 字段用于后端接口 @PreAuthorize 权限校验 |
| **特殊属性** | 保存时会自动清空 `component`、`icon`、`path` 等字段 |

**按钮显隐前端逻辑 (示意)：**
```typescript
// 从菜单中提取所有权限标识
function extractPermissions(menus) {
  const permissions = []
  menus.forEach(menu => {
    if (menu.permission) {
      permissions.push(menu.permission)
    }
    if (menu.children) {
      permissions.push(...extractPermissions(menu.children))
    }
  })
  return permissions
}

// 按钮显隐判断
function hasPermission(permission) {
  return permissions.includes(permission)
}
```

**后端保存逻辑 (MenuServiceImpl.java:297-305)：**
```java
private void initMenuProperty(MenuDO menu) {
    // 菜单为按钮类型时，无需 component、icon、path 属性，进行置空
    if (MenuTypeEnum.BUTTON.getType().equals(menu.getType())) {
        menu.setComponent("");
        menu.setComponentName("");
        menu.setIcon("");
        menu.setPath("");
    }
}
```

---

## 三、权限校验流程

### 3.1 权限标识格式

权限标识存储在 `system_menu.permission` 字段中，格式约定：

```
${系统}:${模块}:${操作}
```

**示例：**
- `system:user:query` - 用户查询
- `system:user:create` - 用户创建
- `system:user:update` - 用户更新
- `system:user:delete` - 用户删除
- `system:user:export` - 用户导出

### 3.2 后端权限校验流程

#### 3.2.1 权限校验入口

权限校验通过 `@PreAuthorize` 注解触发：

```java
@GetMapping("/list")
@PreAuthorize("@ss.hasPermission('system:user:query')")
public CommonResult<PageResult<UserRespVO>> getUserPage(...) {
    // ...
}
```

#### 3.2.2 完整校验链路

```
@PreAuthorize("@ss.hasPermission('xxx')")
    ↓
SecurityFrameworkServiceImpl.hasPermission()
    ↓
PermissionServiceImpl.hasAnyPermissions()
    ├─ 步骤1: 获取用户的启用角色 (带缓存)
    │    └─ getEnableUserRoleListByUserIdFromCache(userId)
    │         ├─ 查 Redis: user_role_ids:{userId}
    │         └─ 查 Redis: role:{roleId} (逐个角色)
    ├─ 步骤2: 逐个权限标识校验
    │    └─ hasAnyPermission(roles, permission)
    │         ├─ 查 Redis: permission_menu_ids:{permission}
    │         │    (获取权限标识对应的菜单ID列表)
    │         ├─ 查 Redis: menu_role_ids:{menuId}
    │         │    (获取每个菜单对应的角色ID列表)
    │         └─ 角色交集判断 (CollUtil.containsAny)
    └─ 步骤3: 超级管理员特殊处理
         └─ roleService.hasAnySuperAdmin(roleIds)
              └─ 检查角色 code 是否为 "super_admin"
```

#### 3.2.3 核心校验代码 (PermissionServiceImpl.java:63-111)

```java
@Override
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // 如果为空，说明已经有权限
    if (ArrayUtil.isEmpty(permissions)) {
        return true;
    }

    // 获得当前登录的角色。如果为空，说明没有权限
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roles)) {
        return false;
    }

    // 情况一：遍历判断每个权限，如果有一满足，说明有权限
    for (String permission : permissions) {
        if (hasAnyPermission(roles, permission)) {
            return true;
        }
    }

    // 情况二：如果是超管，也说明有权限
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}

private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    List<Long> menuIds = menuService.getMenuIdListByPermissionFromCache(permission);
    // 采用严格模式，如果权限找不到对应的 Menu 的话，也认为没有权限
    if (CollUtil.isEmpty(menuIds)) {
        return false;  // ← 关键：权限标识不存在时返回 false
    }

    // 判断是否有权限
    Set<Long> roleIds = convertSet(roles, RoleDO::getId);
    for (Long menuId : menuIds) {
        Set<Long> menuRoleIds = getSelf().getMenuRoleIdListByMenuIdFromCache(menuId);
        if (CollUtil.containsAny(menuRoleIds, roleIds)) {
            return true;
        }
    }
    return false;
}
```

---

## 四、前端权限控制

### 4.1 权限信息获取接口

**接口：** `GET /admin-api/system/auth/get-permission-info`

**返回数据结构：**
```json
{
  "user": {
    "id": 1,
    "username": "admin",
    "nickname": "芋道源码",
    "deptId": 103
  },
  "roles": ["super_admin", "hr"],
  "menus": [
    {
      "id": 1,
      "name": "系统管理",
      "path": "/system",
      "type": 1,
      "children": [...]
    }
  ]
}
```

**后端实现 (AuthController.java)：**
```java
@GetMapping("/get-permission-info")
public CommonResult<AuthPermissionInfoRespVO> getPermissionInfo() {
    // 1. 获取用户信息
    AdminUserDO user = userService.getUser(getLoginUserId());
    
    // 2. 获取用户角色
    Set<Long> roleIds = permissionService.getUserRoleIdListByUserId(getLoginUserId());
    List<RoleDO> roles = roleService.getRoleList(roleIds);
    roles.removeIf(role -> !CommonStatusEnum.ENABLE.getStatus().equals(role.getStatus()));
    
    // 3. 获取用户菜单 (根据角色)
    Set<Long> menuIds = permissionService.getRoleMenuListByRoleId(convertSet(roles, RoleDO::getId));
    List<MenuDO> menuList = menuService.getMenuList(menuIds);
    menuList = menuService.filterDisableMenus(menuList);
    
    return success(AuthConvert.INSTANCE.convert(user, roles, menuList));
}
```

### 4.2 菜单转路由逻辑

前端将菜单树转换为 Vue Router 路由时，会过滤掉按钮类型：

```typescript
function transformMenusToRoutes(menus) {
  return menus
    .filter(menu => menu.type !== 3)  // ← 关键：排除按钮类型 (type=3)
    .map(menu => {
      return {
        path: menu.path,
        name: menu.componentName || menu.name,
        component: resolveComponent(menu.component),
        meta: {
          title: menu.name,
          icon: menu.icon,
          hidden: menu.visible === false,
          keepAlive: menu.keepAlive,
          permission: menu.permission
        },
        children: menu.children ? transformMenusToRoutes(menu.children) : []
      }
    })
}
```

### 4.3 按钮显隐判断

前端按钮显隐基于从菜单中提取的 `permission` 字段：

```typescript
// 递归提取所有权限标识
function extractPermissions(menus) {
  const permissions = []
  menus.forEach(menu => {
    if (menu.permission) {
      permissions.push(menu.permission)
    }
    if (menu.children) {
      permissions.push(...extractPermissions(menu.children))
    }
  })
  return permissions
}

// 使用方式 (Vue 模板)
// 方式1: 自定义指令
<button v-hasPermi="'system:user:create'">新增用户</button>

// 方式2: v-if 判断
<button v-if="hasPermission('system:user:create')">新增用户</button>
```

---

## 五、超级管理员权限绕过

### 5.1 超级管理员判定逻辑

超级管理员通过角色 `code` 字段判断，在 `RoleCodeEnum.java` 中定义：

```java
public enum RoleCodeEnum {
    SUPER_ADMIN("super_admin", "超级管理员"),
    // ...
    
    public static boolean isSuperAdmin(String code) {
        return ObjectUtils.equalsAny(code, SUPER_ADMIN.getCode());
    }
}
```

**判定实现 (RoleServiceImpl.java:234-243)：**
```java
@Override
public boolean hasAnySuperAdmin(Collection<Long> ids) {
    if (CollectionUtil.isEmpty(ids)) {
        return false;
    }
    RoleServiceImpl self = getSelf();
    return ids.stream().anyMatch(id -> {
        RoleDO role = self.getRoleFromCache(id);
        return role != null && RoleCodeEnum.isSuperAdmin(role.getCode());
    });
}
```

### 5.2 权限绕过机制

在 `PermissionServiceImpl.hasAnyPermissions()` 方法中：

```java
@Override
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // ... 普通权限校验 ...
    
    // 情况二：如果是超管，也说明有权限
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}
```

**执行流程：**
```
1. 先尝试普通权限校验 (通过角色-菜单关联)
2. 如果普通校验不通过，再检查是否是超管
3. 如果是超管 → 返回 true (绕过权限校验)
4. 如果不是超管 → 返回 false
```

### 5.3 超级管理员的特殊表现

| 表现 | 说明 |
|-----|------|
| **后端接口** | 所有 `@PreAuthorize` 校验都通过 |
| **前端菜单** | 仍需配置菜单才能在侧边栏显示 |
| **数据权限** | 数据权限仍需配置 (`data_scope` 字段) |
| **缓存** | 角色信息同样使用 Redis 缓存 |

---

## 六、缓存机制

### 6.1 Redis Key 设计

| Key 格式 | 缓存内容 | 数据类型 | 说明 |
|---------|---------|---------|------|
| `user_role_ids:{userId}` | 用户的角色ID集合 | Set | 用户-角色关联 |
| `role:{roleId}` | 角色详情 | String (JSON) | 角色信息 |
| `menu_role_ids:{menuId}` | 拥有该菜单的角色ID集合 | Set | 菜单-角色关联 |
| `permission_menu_ids:{permission}` | 权限标识对应的菜单ID列表 | List | 权限-菜单映射 |

**定义位置：** `RedisKeyConstants.java`

```java
public interface RedisKeyConstants {
    String ROLE = "role";
    String USER_ROLE_ID_LIST = "user_role_ids";
    String MENU_ROLE_ID_LIST = "menu_role_ids";
    String PERMISSION_MENU_ID_LIST = "permission_menu_ids";
}
```

### 6.2 缓存读取流程

#### 6.2.1 用户角色缓存

```java
// PermissionServiceImpl.java:242-245
@Override
@Cacheable(value = RedisKeyConstants.USER_ROLE_ID_LIST, key = "#userId")
public Set<Long> getUserRoleIdListByUserIdFromCache(Long userId) {
    return getUserRoleIdListByUserId(userId);
}
```

#### 6.2.2 权限-菜单缓存

```java
// MenuServiceImpl.java:191-195
@Override
@Cacheable(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, key = "#permission")
public List<Long> getMenuIdListByPermissionFromCache(String permission) {
    List<MenuDO> menus = menuMapper.selectListByPermission(permission);
    return convertList(menus, MenuDO::getId);
}
```

#### 6.2.3 菜单-角色缓存

```java
// PermissionServiceImpl.java:198-201
@Override
@Cacheable(value = RedisKeyConstants.MENU_ROLE_ID_LIST, key = "#menuId")
public Set<Long> getMenuRoleIdListByMenuIdFromCache(Long menuId) {
    return convertSet(roleMenuMapper.selectListByMenuId(menuId), RoleMenuDO::getRoleId);
}
```

### 6.3 缓存失效时机

#### 6.3.1 角色菜单变更时

```java
// PermissionServiceImpl.java:134-160
@Override
@Caching(evict = {
    @CacheEvict(value = RedisKeyConstants.MENU_ROLE_ID_LIST, allEntries = true),
    @CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, allEntries = true)
})
public void assignRoleMenu(Long roleId, Set<Long> menuIds) {
    // ... 数据库操作 ...
}
```

**说明：** 使用 `allEntries = true` 清空整个缓存命名空间，因为一次更新涉及多个 menuIds。

#### 6.3.2 用户角色变更时

```java
// PermissionServiceImpl.java:206-208
@Override
@CacheEvict(value = RedisKeyConstants.USER_ROLE_ID_LIST, key = "#userId")
public void assignUserRole(Long userId, Set<Long> roleIds) {
    // ... 数据库操作 ...
}
```

#### 6.3.3 菜单变更时

```java
// MenuServiceImpl.java:51-53
@Override
@CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, key = "#createReqVO.permission",
        condition = "#createReqVO.permission != null")
public Long createMenu(MenuSaveVO createReqVO) {
    // ...
}
```

---

## 七、角色权限变更后在线用户生效机制

### 7.1 生效时机分析

角色权限变更后，不同层级的生效时间不同：

#### 7.1.1 后端缓存 (Redis)

| 操作 | 缓存状态 | 生效时间 |
|-----|---------|---------|
| 分配角色菜单 | `@CacheEvict` 清除 | **立即生效** |
| 分配用户角色 | `@CacheEvict` 清除 | **立即生效** |
| 修改菜单权限 | `@CacheEvict` 清除 | **立即生效** |

#### 7.1.2 前端状态

| 状态 | 存储位置 | 生效时间 |
|-----|---------|---------|
| 路由配置 | Vue Router (内存) | **需刷新页面** 或重新调用 `generateRoutes()` |
| 权限列表 | Pinia/Vuex Store (内存) | **需刷新页面** 或重新获取权限信息 |
| 菜单显示 | 侧边栏组件 | **需刷新页面** 或重新获取菜单数据 |

### 7.2 完整生效流程图

```
管理员修改角色权限
    ↓
┌─────────────────────────────────────────┐
│ 后端层面 (立即生效)                      │
│  ├─ @CacheEvict 清除 Redis 缓存          │
│  └─ 后续请求直接查库并回写缓存            │
└─────────────────────────┬───────────────┘
                          ↓
┌─────────────────────────┴───────────────┐
│ 在线用户的后续请求                        │
│  ├─ 后端接口权限校验: 立即生效 (查最新)   │
│  └─ 前端按钮显隐: 仍使用旧权限列表         │
│     (前端权限列表在登录/刷新时获取)        │
└─────────────────────────┬───────────────┘
                          ↓
┌─────────────────────────┴───────────────┐
│ 前端层面生效方式                          │
│  ├─ 方式1: 用户手动刷新页面 (F5)          │
│  │   └─ 路由守卫触发 → 重新获取权限信息    │
│  ├─ 方式2: 退出重新登录                   │
│  │   └─ 登录成功后获取最新权限            │
│  └─ 方式3: 前端实现主动刷新机制           │
│      └─ 定时器/WebSocket 主动刷新         │
└─────────────────────────────────────────┘
```

### 7.3 路由守卫恢复流程

```typescript
// 刷新页面时，路由守卫会触发
router.beforeEach(async (to, from, next) => {
  const hasToken = getToken()
  
  if (hasToken) {
    // 检查是否已获取用户信息
    const hasGetUserInfo = userStore.name
    
    if (hasGetUserInfo) {
      next()
    } else {
      try {
        // 1. 获取用户信息
        await userStore.getUserInfo()
        
        // 2. 生成动态路由 (从后端获取最新菜单和权限)
        const accessRoutes = await permissionStore.generateRoutes()
        
        // 3. 重新导航
        next({ ...to, replace: true })
      } catch (error) {
        // 出错: 清除 Token 并跳转登录页
        await userStore.resetToken()
        next(`/login?redirect=${to.path}`)
      }
    }
  }
})
```

### 7.4 总结

| 层级 | 变更后是否立即生效 | 生效条件 |
|-----|-----------------|---------|
| **后端 Redis 缓存** | ✅ 是 | `@CacheEvict` 立即清除 |
| **后端接口鉴权** | ✅ 是 | 下次请求查最新数据 |
| **前端路由配置** | ❌ 否 | 需刷新页面或重新登录 |
| **前端按钮显隐** | ❌ 否 | 需刷新页面或重新登录 |
| **前端侧边栏菜单** | ❌ 否 | 需刷新页面或重新登录 |

---

## 八、权限字符串命名不一致问题

### 8.1 问题场景

权限字符串需要在两处保持一致：

1. **菜单配置**：`system_menu.permission` 字段
2. **接口注解**：`@PreAuthorize("@ss.hasPermission('xxx')")`

### 8.2 不一致的症状

#### 8.2.1 场景一：菜单有但接口无

**情况：** 菜单配置了 `permission = "system:user:create"`，但接口注解写错了。

```java
// 菜单配置: system:user:create
// 接口注解: @PreAuthorize("@ss.hasPermission('system:user:add')")
// 不一致!
```

**症状：**
| 检查项 | 结果 |
|-------|------|
| 前端按钮显示 | ✅ 显示 (菜单 permission 正确) |
| 点击按钮调用接口 | ❌ 403 权限不足 |
| 后端日志 | 权限校验失败 |

**原因分析：**
```
1. 前端: 从菜单提取权限 → 有 "system:user:create" → 按钮显示
2. 后端: @PreAuthorize 校验 "system:user:add"
   → 查 permission_menu_ids:system:user:add
   → 找不到对应菜单 (因为菜单配置的是 create)
   → 返回 false → 403
```

#### 8.2.2 场景二：接口有但菜单无

**情况：** 接口注解是 `system:user:create`，但菜单 permission 为空或写错。

```java
// 菜单配置: permission = "" 或 "system:user:add"
// 接口注解: @PreAuthorize("@ss.hasPermission('system:user:create')")
```

**症状：**
| 检查项 | 结果 |
|-------|------|
| 前端按钮显示 | ❌ 不显示 (菜单 permission 为空/错误) |
| 直接调用接口 | 取决于角色是否超管 |

**原因分析：**
```
1. 前端: 从菜单提取权限 → 没有 "system:user:create" → 按钮隐藏
2. 后端: 如果直接用工具调用接口
   → 普通用户: 查 permission_menu_ids:system:user:create
      → 找不到菜单或角色无关联 → 403
   → 超管: 绕过校验 → 200
```

#### 8.2.3 场景三：大小写或拼写错误

```
正确: system:user:create
错误: System:user:create (首字母大写)
错误: system:user:creat (少一个 e)
错误: system_user_create (下划线代替冒号)
```

**症状：** 与场景一相同，前端显示但后端 403。

### 8.3 严格模式说明

权限校验采用**严格模式**：

```java
// PermissionServiceImpl.java:94-98
private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    List<Long> menuIds = menuService.getMenuIdListByPermissionFromCache(permission);
    // 采用严格模式，如果权限找不到对应的 Menu 的话，也认为没有权限
    if (CollUtil.isEmpty(menuIds)) {
        return false;  // ← 权限标识不存在时直接返回 false
    }
    // ...
}
```

**这意味着：**
- 接口注解的权限标识必须在 `system_menu` 表中存在对应的记录
- 否则即使角色配置了菜单，也会被判定为无权限

---

## 九、前端按钮可见但点击接口 403 原因树

### 9.1 完整原因树

```
前端按钮可见但点击接口返回 403
    │
    ├─ A. 权限标识不一致问题
    │   ├─ A1. 菜单 permission 与接口 @PreAuthorize 不一致
    │   │   ├─ 菜单: system:user:create
    │   │   └─ 接口: @PreAuthorize("system:user:add")
    │   │
    │   ├─ A2. 菜单 permission 为空或未配置
    │   │   ├─ 前端: 按钮可能因其他条件显示 (如 v-if 硬编码)
    │   │   └─ 后端: @PreAuthorize 校验失败
    │   │
    │   └─ A3. 大小写或拼写错误
    │       ├─ System:user:create (首字母大写)
    │       ├─ system:user:creat (拼写错误)
    │       └─ system_user_create (分隔符错误)
    │
    ├─ B. 缓存不一致问题
    │   ├─ B1. 前端权限列表未刷新
    │   │   ├─ 前端: 使用旧权限列表 (含该权限) → 按钮显示
    │   │   └─ 后端: Redis 缓存已清除，查最新数据 (无该权限) → 403
    │   │   └─ 触发场景: 管理员刚撤销了该用户的权限
    │   │
    │   ├─ B2. Redis 缓存与数据库不一致 (异常情况)
    │   │   ├─ 数据库: 用户角色已变更
    │   │   ├─ Redis: user_role_ids 缓存未清除 (如手动改库未走 Service)
    │   │   └─ 前端: 登录时获取的是旧权限
    │   │
    │   └─ B3. 权限-菜单缓存问题
    │       ├─ permission_menu_ids 缓存脏数据
    │       └─ 菜单 permission 已修改但缓存未更新
    │
    ├─ C. 角色配置问题
    │   ├─ C1. 角色被禁用
    │   │   ├─ 前端: 权限列表是登录时获取的 (旧数据)
    │   │   └─ 后端: 校验时过滤禁用角色 → 角色为空 → 无权限
    │   │   └─ 关键代码: roles.removeIf(role -> !ENABLE.equals(role.getStatus()))
    │   │
    │   ├─ C2. 角色菜单配置未生效
    │   │   ├─ 前端: 按钮权限来自用户自己的菜单权限
    │   │   └─ 后端: 校验的是按钮对应的菜单是否分配给角色
    │   │   └─ 可能: 角色只分配了父菜单，没分配按钮菜单
    │   │
    │   └─ C3. 用户角色变更但前端未刷新
    │       ├─ 管理员移除了用户的某角色
    │       ├─ 后端: user_role_ids 缓存已清除 → 无该角色
    │       └─ 前端: 仍使用登录时获取的旧权限列表
    │
    ├─ D. 菜单配置问题
    │   ├─ D1. 按钮菜单未分配给角色
    │   │   ├─ 前端: 按钮权限可能来自其他菜单 (如父菜单的 permission)
    │   │   └─ 后端: 校验按钮对应的 permission → 角色无该菜单 → 403
    │   │
    │   ├─ D2. 按钮菜单被禁用
    │   │   ├─ 前端: 登录时获取的菜单列表包含该按钮 (禁用前登录)
    │   │   └─ 后端: filterDisableMenus 或校验时排除禁用菜单
    │   │
    │   └─ D3. 存在多个相同 permission 的菜单
    │       ├─ 菜单A: permission = "system:user:create" (分配给角色)
    │       ├─ 菜单B: permission = "system:user:create" (未分配)
    │       └─ 前端: 提取到权限 → 按钮显示
    │       └─ 后端: 可能命中未分配的菜单B → 403
    │
    └─ E. 其他特殊情况
        ├─ E1. 多租户菜单过滤
        │   ├─ 前端: 菜单权限已提取
        │   └─ 后端: 租户套餐未包含该菜单 → 过滤掉 → 403
        │
        ├─ E2. 数据权限干扰 (特殊场景)
        │   └─ 数据权限注解导致的误判 (较少见)
        │
        └─ E3. 跨模块权限校验
            └─ 权限 API 调用异常或超时
```

### 9.2 排查流程图

```
用户报告: 按钮可见但点击 403
    │
    ▼
┌─────────────────────────────────────────────┐
│ 步骤1: 确认权限标识一致性                      │
├─────────────────────────────────────────────┤
│ 1. 查看按钮对应菜单的 permission 字段         │
│    SELECT permission FROM system_menu        │
│    WHERE name LIKE '%按钮名%' AND type = 3    │
│                                             │
│ 2. 查看接口的 @PreAuthorize 注解              │
│    搜索代码: @PreAuthorize("@ss.hasPermission│
│                                             │
│ 3. 比较两者是否完全一致 (大小写、分隔符)       │
└───────────────────┬─────────────────────────┘
                    │ 不一致
                    ▼
              ┌────────────┐
              │ 修正不一致 │
              └─────┬──────┘
                    │ 一致
                    ▼
┌─────────────────────────────────────────────┐
│ 步骤2: 检查角色和菜单配置                      │
├─────────────────────────────────────────────┤
│ 1. 用户有哪些角色?                            │
│    SELECT r.* FROM system_role r             │
│    JOIN system_user_role ur ON r.id = ur.role_id│
│    WHERE ur.user_id = {userId}               │
│                                             │
│ 2. 角色是否启用?                              │
│    SELECT status FROM system_role            │
│    WHERE id IN ({roleIds})                   │
│                                             │
│ 3. 角色是否分配了该按钮菜单?                   │
│    SELECT m.* FROM system_menu m             │
│    JOIN system_role_menu rm ON m.id = rm.menu_id│
│    WHERE rm.role_id IN ({roleIds})           │
│      AND m.permission = '{permission}'       │
└───────────────────┬─────────────────────────┘
                    │ 配置有问题
                    ▼
              ┌────────────┐
              │ 修正配置   │
              └─────┬──────┘
                    │ 配置正确
                    ▼
┌─────────────────────────────────────────────┐
│ 步骤3: 检查缓存状态                            │
├─────────────────────────────────────────────┤
│ 1. 清除相关 Redis 缓存后重试                   │
│    DEL user_role_ids:{userId}               │
│    DEL permission_menu_ids:{permission}     │
│    DEL menu_role_ids:{menuId}               │
│                                             │
│ 2. 建议用户刷新页面或重新登录                  │
└───────────────────┬─────────────────────────┘
                    │ 仍有问题
                    ▼
┌─────────────────────────────────────────────┐
│ 步骤4: 检查特殊情况                            │
├─────────────────────────────────────────────┤
│ 1. 是否多租户? 检查租户套餐菜单权限            │
│ 2. 是否超管角色? 检查角色 code                │
│ 3. 是否跨模块调用? 检查权限 API 连通性         │
└─────────────────────────────────────────────┘
```

### 9.3 关键排查 SQL

```sql
-- 1. 查看用户的角色
SELECT 
    u.id AS user_id,
    u.username,
    r.id AS role_id,
    r.name AS role_name,
    r.code AS role_code,
    r.status AS role_status
FROM system_user u
JOIN system_user_role ur ON u.id = ur.user_id
JOIN system_role r ON ur.role_id = r.id
WHERE u.username = '{用户名}';

-- 2. 查看角色的菜单权限
SELECT 
    r.name AS role_name,
    m.id AS menu_id,
    m.name AS menu_name,
    m.permission,
    m.type AS menu_type
FROM system_role r
JOIN system_role_menu rm ON r.id = rm.role_id
JOIN system_menu m ON rm.menu_id = m.id
WHERE r.id = {角色ID}
  AND m.permission IS NOT NULL
ORDER BY r.name, m.id;

-- 3. 查看特定权限标识的菜单
SELECT 
    id,
    name,
    permission,
    type,
    status
FROM system_menu
WHERE permission = '{权限标识}';

-- 4. 查看哪些角色拥有该权限
SELECT DISTINCT
    r.id AS role_id,
    r.name AS role_name,
    r.code AS role_code
FROM system_role r
JOIN system_role_menu rm ON r.id = rm.role_id
JOIN system_menu m ON rm.menu_id = m.id
WHERE m.permission = '{权限标识}';

-- 5. 查看某用户是否有某权限 (完整链路)
SELECT DISTINCT
    u.username,
    r.name AS role_name,
    m.name AS menu_name,
    m.permission
FROM system_user u
JOIN system_user_role ur ON u.id = ur.user_id
JOIN system_role r ON ur.role_id = r.id
JOIN system_role_menu rm ON r.id = rm.role_id
JOIN system_menu m ON rm.menu_id = m.id
WHERE u.username = '{用户名}'
  AND m.permission = '{权限标识}'
  AND r.status = 0  -- 角色启用
  AND m.status = 0; -- 菜单启用
```

### 9.4 常见问题快速定位

| 现象 | 可能原因 | 排查方法 |
|-----|---------|---------|
| 按钮显示、接口403、刷新页面后按钮消失 | 前端权限列表未刷新 | 检查是否刚变更权限，建议刷新 |
| 按钮显示、接口403、刷新后问题依旧 | 权限标识不一致 | 比较菜单 permission 和接口注解 |
| 其他用户正常、某用户异常 | 该用户角色配置问题 | 检查该用户的角色和角色菜单 |
| 所有用户都异常 | 菜单配置或缓存问题 | 检查菜单 permission 是否配置，清缓存 |
| 超管正常、普通用户异常 | 角色菜单分配问题 | 检查角色是否分配了该按钮菜单 |

---

## 十、关键类与文件汇总

### 10.1 后端核心类

| 类名 | 文件路径 | 职责 |
|-----|---------|------|
| `PermissionServiceImpl` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/service/permission/PermissionServiceImpl.java` | 权限校验核心实现 |
| `RoleServiceImpl` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/service/permission/RoleServiceImpl.java` | 角色管理、超管判断 |
| `MenuServiceImpl` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/service/permission/MenuServiceImpl.java` | 菜单管理、权限-菜单缓存 |
| `SecurityFrameworkServiceImpl` | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/service/SecurityFrameworkServiceImpl.java` | SpEL 权限服务 (@ss) |
| `AuthController` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/controller/admin/auth/AuthController.java` | 获取权限信息接口 |

### 10.2 枚举类

| 枚举名 | 文件路径 | 说明 |
|-------|---------|------|
| `MenuTypeEnum` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/enums/permission/MenuTypeEnum.java` | 菜单类型枚举 |
| `RoleCodeEnum` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/enums/permission/RoleCodeEnum.java` | 角色编码枚举 (含超管) |

### 10.3 Redis 常量

| 常量类 | 文件路径 |
|-------|---------|
| `RedisKeyConstants` | `yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/redis/RedisKeyConstants.java` |

---

## 十一、最佳实践建议

1. **权限标识命名规范**
   - 统一使用小写：`system:user:create`
   - 统一使用冒号分隔
   - 命名要语义化：`${系统}:${模块}:${操作}`

2. **菜单配置规范**
   - 按钮类型 (type=3) 必须配置 `permission` 字段
   - 确保 `permission` 与接口 `@PreAuthorize` 完全一致
   - 修改 `permission` 后要测试前后端是否正常

3. **权限变更操作**
   - 必须通过 Service 层操作，确保 `@CacheEvict` 生效
   - 不要直接修改数据库
   - 变更后建议通知相关用户刷新页面

4. **问题排查流程**
   - 先检查权限标识一致性
   - 再检查角色菜单配置
   - 最后检查缓存状态
   - 建议用户刷新页面测试

---

*文档生成时间: 2026-05-10*
