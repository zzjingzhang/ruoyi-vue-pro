# ruoyi-vue-pro 缓存机制深度分析

## 概述

ruoyi-vue-pro 项目采用了两种主要的缓存策略：
1. **Redis 分布式缓存**：通过 Spring Cache 注解和 RedisTemplate 手动操作实现，适合跨服务共享的数据
2. **Guava LoadingCache 本地缓存**：适合单机高频访问且允许短暂不一致的数据

本文将详细分析以下三类核心缓存的实现机制：

---

## 一、用户权限缓存（Redis + Spring Cache）

### 1.1 缓存 Key 组成

权限相关的缓存 Key 定义在 `RedisKeyConstants.java` 中：

| 缓存类型 | Key 格式 | 存储内容 | 定义位置 |
|---------|---------|---------|---------|
| 用户角色列表 | `user_role_ids:{userId}` | 用户拥有的角色编号集合 Set<Long> | `RedisKeyConstants.USER_ROLE_ID_LIST` |
| 菜单角色列表 | `menu_role_ids:{menuId}` | 拥有某菜单的角色编号集合 Set<Long> | `RedisKeyConstants.MENU_ROLE_ID_LIST` |
| 权限菜单列表 | `permission_menu_ids:{permission}` | 某权限标识对应的菜单编号数组 List<Long> | `RedisKeyConstants.PERMISSION_MENU_ID_LIST` |
| 角色信息 | `role:{id}` | 角色完整信息 RoleDO | `RedisKeyConstants.ROLE` |

在多租户场景下，通过 `TenantRedisCacheManager` 会自动在缓存名后拼接租户 ID：
```
cacheName + ":" + tenantId
```

**完整 Redis Key 示例**：
- 非多租户：`user_role_ids:1` （用户 ID 为 1 的角色列表）
- 多租户：`user_role_ids:100:1` （租户 100 下用户 ID 为 1 的角色列表）

### 1.2 读写路径分析

#### 读取路径 - 权限校验流程

当用户访问需要权限的接口时，调用链如下：

```
PermissionServiceImpl.hasAnyPermissions(userId, permissions)
    └── getEnableUserRoleListByUserIdFromCache(userId)
         ├── getUserRoleIdListByUserIdFromCache(userId)  // 缓存读取：@Cacheable
         │    └── [Redis Key: user_role_ids:{userId}]
         └── roleService.getRoleListFromCache(roleIds)
              └── getRoleFromCache(roleId)  // 缓存读取：@Cacheable
                   └── [Redis Key: role:{roleId}]
```

**关键代码**（`PermissionServiceImpl.java:63-84`）：
```java
@Override
public boolean hasAnyPermissions(Long userId, String... permissions) {
    if (ArrayUtil.isEmpty(permissions)) {
        return true;
    }
    
    // 从缓存获取用户角色
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roles)) {
        return false;
    }
    
    for (String permission : permissions) {
        if (hasAnyPermission(roles, permission)) {
            return true;
        }
    }
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}
```

**单权限校验内部流程**（`PermissionServiceImpl.java:93-111`）：
```java
private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    // 步骤1：从缓存获取权限对应的菜单列表
    List<Long> menuIds = menuService.getMenuIdListByPermissionFromCache(permission);
    // 缓存 Key: permission_menu_ids:{permission}
    
    if (CollUtil.isEmpty(menuIds)) {
        return false;
    }
    
    Set<Long> roleIds = convertSet(roles, RoleDO::getId);
    for (Long menuId : menuIds) {
        // 步骤2：从缓存获取拥有该菜单的角色列表
        Set<Long> menuRoleIds = getSelf().getMenuRoleIdListByMenuIdFromCache(menuId);
        // 缓存 Key: menu_role_ids:{menuId}
        
        if (CollUtil.containsAny(menuRoleIds, roleIds)) {
            return true;
        }
    }
    return false;
}
```

#### 写入路径

**场景1：分配用户角色**
- 方法：`PermissionServiceImpl.assignUserRole(Long userId, Set<Long> roleIds)`
- 位置：`PermissionServiceImpl.java:206-228`
- 缓存处理：`@CacheEvict(value = USER_ROLE_ID_LIST, key = "#userId")`

**场景2：分配角色菜单**
- 方法：`PermissionServiceImpl.assignRoleMenu(Long roleId, Set<Long> menuIds)`
- 位置：`PermissionServiceImpl.java:134-160`
- 缓存处理：
  ```java
  @Caching(evict = {
      @CacheEvict(value = MENU_ROLE_ID_LIST, allEntries = true),
      @CacheEvict(value = PERMISSION_MENU_ID_LIST, allEntries = true)
  })
  ```

### 1.3 缓存失效入口（更新数据库后清缓存）

| 操作 | 方法 | 清除的缓存 | 注解位置 |
|-----|------|-----------|---------|
| 分配用户角色 | `assignUserRole(userId, roleIds)` | `USER_ROLE_ID_LIST:{userId}` | `PermissionServiceImpl.java:207` |
| 删除用户 | `processUserDeleted(userId)` | `USER_ROLE_ID_LIST:{userId}` | `PermissionServiceImpl.java:231` |
| 分配角色菜单 | `assignRoleMenu(roleId, menuIds)` | `MENU_ROLE_ID_LIST:*` (全部) + `PERMISSION_MENU_ID_LIST:*` (全部) | `PermissionServiceImpl.java:135-139` |
| 删除角色 | `processRoleDeleted(roleId)` | `MENU_ROLE_ID_LIST:*` (全部) + `USER_ROLE_ID_LIST:*` (全部) | `PermissionServiceImpl.java:164-169` |
| 删除菜单 | `processMenuDeleted(menuId)` | `MENU_ROLE_ID_LIST:{menuId}` | `PermissionServiceImpl.java:178` |
| 更新角色 | `updateRole(updateReqVO)` | `ROLE:{id}` | `RoleServiceImpl.java:75` |
| 更新角色数据范围 | `updateRoleDataScope(id, ...)` | `ROLE:{id}` | `RoleServiceImpl.java:94` |
| 删除角色 | `deleteRole(id)` | `ROLE:{id}` | `RoleServiceImpl.java:109` |
| 创建菜单 | `createMenu(createReqVO)` | `PERMISSION_MENU_ID_LIST:{permission}`（仅当设置了权限标识时） | `MenuServiceImpl.java:51` |
| 更新菜单 | `updateMenu(updateReqVO)` | `PERMISSION_MENU_ID_LIST:*` (全部) | `MenuServiceImpl.java:69-70` |
| 删除菜单 | `deleteMenu(id)` | `PERMISSION_MENU_ID_LIST:*` (全部) | `MenuServiceImpl.java:90-91` |

### 1.4 集群部署下本地缓存不一致的风险

**权限缓存使用的是 Redis 分布式缓存，理论上不存在本地缓存不一致问题。** 但需要注意以下潜在风险：

#### 风险点 1：批量清除缓存的粒度问题

在 `assignRoleMenu` 方法中使用了 `allEntries = true` 批量清除：
```java
@CacheEvict(value = RedisKeyConstants.MENU_ROLE_ID_LIST, allEntries = true)
@CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, allEntries = true)
```

**风险**：虽然 Redis 是分布式的，但频繁的全量缓存清除可能导致缓存雪崩，所有请求同时打到数据库。

#### 风险点 2：Spring Cache 默认行为

Spring Cache 对于 Redis 的 `@CacheEvict(allEntries = true)` 是通过 `keys` 命令匹配模式删除，在 Redis 集群模式下性能较差。

#### 风险点 3：事务与缓存的一致性问题

看以下代码（`PermissionServiceImpl.java:134-160`）：
```java
@DSTransactional
@Caching(evict = {
    @CacheEvict(value = MENU_ROLE_ID_LIST, allEntries = true),
    @CacheEvict(value = PERMISSION_MENU_ID_LIST, allEntries = true)
})
public void assignRoleMenu(Long roleId, Set<Long> menuIds) {
    // 先清除缓存，再更新数据库
    // ... 数据库操作
}
```

**问题**：Spring Cache 默认在方法执行**前**清除缓存。如果数据库事务回滚，但缓存已经被清除了，这不会有问题（缓存为空下次会从数据库重新加载）。但如果方法执行时间很长，在清除缓存后、数据库更新前，其他请求可能会把旧数据重新加载到缓存中。

**正确做法**：应该使用 `beforeInvocation = false`（默认值就是 false，在方法执行后清除）。让我再确认一下...

查看代码，Spring 的 `@CacheEvict` 默认是在方法**成功执行后**才清除缓存的（`beforeInvocation=false`），所以这个问题在正常情况下不会发生。但如果是手动操作 RedisTemplate 清除缓存，就需要特别注意事务顺序。

---

## 二、字典缓存（Guava LoadingCache 本地缓存）

### 2.1 缓存 Key 组成

字典缓存使用的是 Guava 的 `LoadingCache`，Key 就是字典类型（dictType）字符串。

**缓存定义**（`DictFrameworkUtils.java:28-40`）：
```java
private static final LoadingCache<String, List<DictDataRespDTO>> GET_DICT_DATA_CACHE = 
    CacheUtils.buildAsyncReloadingCache(
        Duration.ofMinutes(1L), // 过期时间 1 分钟
        new CacheLoader<String, List<DictDataRespDTO>>() {
            @Override
            public List<DictDataRespDTO> load(String dictType) {
                return dictDataApi.getDictDataList(dictType);
            }
        });
```

**Key 格式**：直接使用 `dictType` 字符串作为 Key，例如：
- `"user_status"`（用户状态字典）
- `"common_status"`（通用状态字典）

### 2.2 读写路径分析

#### 读取路径

**入口方法**（`DictFrameworkUtils.java:52-83`）：
```java
// 场景1：解析字典值对应的标签
public static String parseDictDataLabel(String dictType, String value) {
    List<DictDataRespDTO> dictDatas = GET_DICT_DATA_CACHE.get(dictType);
    DictDataRespDTO dictData = CollUtil.findOne(dictDatas, 
        data -> Objects.equals(data.getValue(), value));
    return dictData != null ? dictData.getLabel() : null;
}

// 场景2：获取字典的所有标签列表
public static List<String> getDictDataLabelList(String dictType) {
    List<DictDataRespDTO> dictDatas = GET_DICT_DATA_CACHE.get(dictType);
    return convertList(dictDatas, DictDataRespDTO::getLabel);
}
```

**调用链**：
```
DictFrameworkUtils.parseDictDataLabel(dictType, value)
    └── GET_DICT_DATA_CACHE.get(dictType)
         └── [缓存命中] → 直接返回
         └── [缓存未命中] → dictDataApi.getDictDataList(dictType)
              └── DictDataApiImpl.getDictDataList(dictType)
                   └── dictDataService.getDictDataListByDictType(dictType)
                        └── dictDataMapper.selectList(DictDataDO::getDictType, dictType)
```

#### 写入路径

字典数据的增删改操作在 `DictDataServiceImpl.java` 中：

**创建字典数据**（`DictDataServiceImpl.java:66-76`）：
```java
@Override
public Long createDictData(DictDataSaveReqVO createReqVO) {
    validateDictTypeExists(createReqVO.getDictType());
    validateDictDataValueUnique(null, createReqVO.getDictType(), createReqVO.getValue());
    
    DictDataDO dictData = BeanUtils.toBean(createReqVO, DictDataDO.class);
    dictDataMapper.insert(dictData);
    return dictData.getId();
}
```

**更新字典数据**（`DictDataServiceImpl.java:79-90`）：
```java
@Override
public void updateDictData(DictDataSaveReqVO updateReqVO) {
    validateDictDataExists(updateReqVO.getId());
    validateDictTypeExists(updateReqVO.getDictType());
    validateDictDataValueUnique(updateReqVO.getId(), updateReqVO.getDictType(), updateReqVO.getValue());
    
    DictDataDO updateObj = BeanUtils.toBean(updateReqVO, DictDataDO.class);
    dictDataMapper.updateById(updateObj);
}
```

### 2.3 缓存失效入口

**⚠️ 重要发现**：`DictDataServiceImpl` 的增删改方法**没有调用任何缓存清除逻辑**！

唯一的缓存清除方法在 `DictFrameworkUtils.java:47-49`：
```java
public static void clearCache() {
    GET_DICT_DATA_CACHE.invalidateAll();
}
```

但这个方法只在**测试代码**中被调用：
```
DictFrameworkUtilsTest.java:30: DictFrameworkUtils.clearCache();
```

**字典缓存的失效完全依赖于 Guava Cache 的过期时间：1 分钟。**

### 2.4 集群部署下本地缓存不一致的风险

这是一个**典型的本地缓存一致性问题**。

#### 问题场景

1. 应用集群有 3 个节点：Node-A、Node-B、Node-C
2. 管理员在后台修改了某个字典数据（例如把"性别"字典的"男"改为"男性"）
3. 数据库更新成功，但**没有任何缓存清除操作**

#### 一致性问题表现

| 时间点 | Node-A | Node-B | Node-C |
|-------|--------|--------|--------|
| T0（修改前） | 缓存：男、女 | 缓存：男、女 | 缓存：男、女 |
| T1（修改数据库） | - | - | - |
| T2（修改后第 10 秒） | 请求到达 Node-A，命中缓存 → 显示"男" | 无请求 | 无请求 |
| T3（修改后第 30 秒） | 新请求到达 Node-B，缓存未过期 → 显示"男" | - | - |
| T4（修改后第 61 秒） | Node-A 缓存过期 → 重新加载，显示"男性" | 缓存未过期 → "男" | 缓存未过期 → "男" |
| T5（修改后第 90 秒） | "男性" | Node-B 缓存过期 → "男性" | 缓存未过期 → "男" |
| T6（修改后第 120 秒） | "男性" | "男性" | Node-C 缓存过期 → "男性" |

**最坏情况下，可能存在长达 2 分钟的数据不一致窗口**（假设三个节点的缓存加载时间点分布均匀）。

#### 风险影响

1. **前端展示不一致**：同一个用户刷新页面，可能看到不同的字典值（因为负载均衡到不同节点）
2. **数据导出问题**：Excel 导出使用了 `DictFrameworkUtils`，不同节点导出的文件可能有不同的字典标签
3. **表单验证问题**：如果字典用于表单验证，可能出现部分节点验证通过、部分节点验证失败

---

## 三、OAuth2 Access Token 缓存（Redis 手动操作）

### 3.1 缓存 Key 组成

Token 缓存 Key 定义在 `RedisKeyConstants.java:68`：
```java
String OAUTH2_ACCESS_TOKEN = "oauth2_access_token:%s";
```

**Key 格式**：`oauth2_access_token:{accessTokenValue}`

**示例**：`oauth2_access_token:a1b2c3d4e5f6...`

**Key 构造**（`OAuth2AccessTokenRedisDAO.java:55-57`）：
```java
private static String formatKey(String accessToken) {
    return String.format(OAUTH2_ACCESS_TOKEN, accessToken);
}
```

### 3.2 读写路径分析

#### 读取路径

**Token 校验入口**（`OAuth2TokenServiceImpl.java:104-128`）：
```java
@Override
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // 步骤1：优先从 Redis 中获取
    OAuth2AccessTokenDO accessTokenDO = oauth2AccessTokenRedisDAO.get(accessToken);
    if (accessTokenDO != null) {
        return accessTokenDO;
    }
    
    // 步骤2：获取不到，从 MySQL 中获取
    accessTokenDO = oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    
    // 步骤3：兼容处理：特殊场景下可能传递的是 refreshToken
    if (accessTokenDO == null) {
        OAuth2RefreshTokenDO refreshTokenDO = 
            oauth2RefreshTokenMapper.selectByRefreshToken(accessToken);
        if (refreshTokenDO != null && !DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
            accessTokenDO = convertToAccessToken(refreshTokenDO);
        }
    }
    
    // 步骤4：如果在 MySQL 存在且未过期，则回写到 Redis
    if (accessTokenDO != null && !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    return accessTokenDO;
}
```

**Redis DAO 实现**（`OAuth2AccessTokenRedisDAO.java:30-33`）：
```java
public OAuth2AccessTokenDO get(String accessToken) {
    String redisKey = formatKey(accessToken);
    return JsonUtils.parseObject(
        stringRedisTemplate.opsForValue().get(redisKey), 
        OAuth2AccessTokenDO.class);
}
```

#### 写入路径

**创建 Token**（`OAuth2TokenServiceImpl.java:177-195`）：
```java
private OAuth2AccessTokenDO createOAuth2AccessToken(
    OAuth2RefreshTokenDO refreshTokenDO, OAuth2ClientDO clientDO) {
    
    OAuth2AccessTokenDO accessTokenDO = new OAuth2AccessTokenDO()
        .setAccessToken(generateAccessToken())
        .setUserId(refreshTokenDO.getUserId())
        .setUserType(refreshTokenDO.getUserType())
        .setClientId(clientDO.getClientId())
        .setScopes(refreshTokenDO.getScopes())
        .setRefreshToken(refreshTokenDO.getRefreshToken())
        .setExpiresTime(LocalDateTime.now().plusSeconds(
            clientDO.getAccessTokenValiditySeconds()));
    
    // 步骤1：写入数据库
    oauth2AccessTokenMapper.insert(accessTokenDO);
    
    // 步骤2：写入 Redis（设置过期时间）
    oauth2AccessTokenRedisDAO.set(accessTokenDO);
    
    return accessTokenDO;
}
```

**Redis 写入带过期时间**（`OAuth2AccessTokenRedisDAO.java:35-43`）：
```java
public void set(OAuth2AccessTokenDO accessTokenDO) {
    String redisKey = formatKey(accessTokenDO.getAccessToken());
    // 清理多余字段
    accessTokenDO.setUpdater(null).setUpdateTime(null)
        .setCreateTime(null).setCreator(null).setDeleted(null);
    
    // 计算剩余过期时间
    long time = LocalDateTimeUtil.between(
        LocalDateTime.now(), 
        accessTokenDO.getExpiresTime(), 
        ChronoUnit.SECONDS);
    
    if (time > 0) {
        stringRedisTemplate.opsForValue().set(
            redisKey, 
            JsonUtils.toJsonString(accessTokenDO), 
            time, 
            TimeUnit.SECONDS);
    }
}
```

### 3.3 缓存失效入口

| 场景 | 方法 | 失效操作 | 位置 |
|-----|------|---------|-----|
| 用户登出（主动注销） | `removeAccessToken(accessToken)` | 数据库删除 + Redis 删除 | `OAuth2TokenServiceImpl.java:144-155` |
| 强制下线（管理员操作） | `removeAccessToken(userId, userType)` | 批量数据库删除 + 批量 Redis 删除 | `OAuth2TokenServiceImpl.java:158-170` |
| 刷新 Token | `refreshAccessToken(refreshToken, clientId)` | 删除旧 access_token 的 Redis 缓存 | `OAuth2TokenServiceImpl.java:87-91` |
| 自然过期 | - | Redis TTL 自动过期 | 数据库中也有 expires_time 字段 |

**主动失效代码**（`OAuth2TokenServiceImpl.java:144-155`）：
```java
@Override
@Transactional(rollbackFor = Exception.class)
public OAuth2AccessTokenDO removeAccessToken(String accessToken) {
    OAuth2AccessTokenDO accessTokenDO = 
        oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    if (accessTokenDO == null) {
        return null;
    }
    // 删除数据库
    oauth2AccessTokenMapper.deleteById(accessTokenDO.getId());
    // 删除 Redis 缓存
    oauth2AccessTokenRedisDAO.delete(accessToken);
    // 删除刷新令牌
    oauth2RefreshTokenMapper.deleteByRefreshToken(accessTokenDO.getRefreshToken());
    return accessTokenDO;
}
```

**Redis 删除实现**（`OAuth2AccessTokenRedisDAO.java:45-53`）：
```java
public void delete(String accessToken) {
    String redisKey = formatKey(accessToken);
    stringRedisTemplate.delete(redisKey);
}

public void deleteList(Collection<String> accessTokens) {
    List<String> redisKeys = CollectionUtils.convertList(
        accessTokens, OAuth2AccessTokenRedisDAO::formatKey);
    stringRedisTemplate.delete(redisKeys);
}
```

### 3.4 集群部署下本地缓存不一致的风险

**Token 缓存使用的是 Redis 分布式缓存，理论上不存在本地缓存不一致问题。** 

但需要注意以下问题：

#### 风险点 1：数据库与 Redis 的双写一致性

看创建 Token 的代码：
```java
oauth2AccessTokenMapper.insert(accessTokenDO);  // 先写数据库
oauth2AccessTokenRedisDAO.set(accessTokenDO);    // 再写 Redis
```

**问题**：如果数据库写入成功但 Redis 写入失败（例如 Redis 临时不可用），会出现什么情况？

**答案**：不会有大问题。因为 `getAccessToken` 方法有回写机制：
```java
// 从 Redis 获取不到，会从 MySQL 查
// 查到后会回写到 Redis
if (accessTokenDO != null && !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
    oauth2AccessTokenRedisDAO.set(accessTokenDO);
}
```

这是一个**最终一致性**的设计。

#### 风险点 2：删除操作的顺序

看删除 Token 的代码：
```java
oauth2AccessTokenMapper.deleteById(accessTokenDO.getId());  // 先删数据库
oauth2AccessTokenRedisDAO.delete(accessToken);               // 再删 Redis
```

**问题**：如果数据库删除成功但 Redis 删除失败怎么办？

**答案**：Redis 的 Key 本身有 TTL，会自动过期。而且下次请求时：
1. 先查 Redis → 可能命中旧数据
2. 但 Token 的 `expiresTime` 字段也在 Redis 中存储
3. 如果 Token 已过期，会被拒绝

但如果 Token 还没过期，就会出现**已被删除但仍能使用**的窗口。

---

## 四、其他缓存类型补充分析

### 4.1 文件配置缓存（Guava LoadingCache）

**定义位置**：`FileConfigServiceImpl.java:53-67`

```java
private final LoadingCache<Long, FileClient> clientCache = 
    buildAsyncReloadingCache(Duration.ofSeconds(10L),
        new CacheLoader<Long, FileClient>() {
            @Override
            public FileClient load(Long id) {
                FileConfigDO config = Objects.equals(CACHE_MASTER_ID, id) ?
                    fileConfigMapper.selectByMaster() : fileConfigMapper.selectById(id);
                if (config != null) {
                    fileClientFactory.createOrUpdateFileClient(
                        config.getId(), config.getStorage(), config.getConfig());
                }
                return fileClientFactory.getFileClient(
                    null == config ? id : config.getId());
            }
        });
```

**失效处理**：有手动清除（`FileConfigServiceImpl.java:162-169`）
```java
private void clearCache(Long id, Boolean master) {
    if (id != null) {
        clientCache.invalidate(id);
    }
    if (Boolean.TRUE.equals(master)) {
        clientCache.invalidate(CACHE_MASTER_ID);
    }
}
```

**但同样存在集群问题**：清除的只是当前节点的本地缓存。

### 4.2 租户 ID 列表缓存（Guava LoadingCache）

**定义位置**：`TenantFrameworkServiceImpl.java:26-57`

```java
private final LoadingCache<Object, List<Long>> getTenantIdsCache = 
    CacheUtils.buildAsyncReloadingCache(
        Duration.ofMinutes(1L),
        new CacheLoader<Object, List<Long>>() {
            @Override
            public List<Long> load(Object key) {
                return tenantApi.getTenantIdList();
            }
        });

private final LoadingCache<Long, ServiceException> validTenantCache = 
    CacheUtils.buildAsyncReloadingCache(
        Duration.ofMinutes(1L),
        new CacheLoader<Long, ServiceException>() {
            @Override
            public ServiceException load(Long id) {
                try {
                    tenantApi.validateTenant(id);
                    return SERVICE_EXCEPTION_NULL;
                } catch (ServiceException ex) {
                    return ex;
                }
            }
        });
```

**失效处理**：**无任何清除逻辑**，完全依赖 1 分钟过期。

---

## 五、排障路径：后台修改角色权限但用户仍能访问旧接口

### 5.1 问题场景复现

**现象**：
1. 管理员在后台修改了角色 A 的权限（例如：移除了"用户删除"权限）
2. 用户张三拥有角色 A
3. 但张三仍然可以调用"用户删除"接口

### 5.2 完整排障路径

#### 步骤 1：确认问题是否与 Token 有关

**检查点**：用户是否重新登录了？

**分析**：
- 如果用户的 Access Token 还没过期，Token 中的信息（如用户信息）可能已经缓存
- 但**权限校验是实时的**，不是存在 Token 里的

**验证方法**：
```bash
# 查看用户当前的 Token 信息
redis-cli KEYS "oauth2_access_token:*"
redis-cli GET "oauth2_access_token:{实际token值}"
```

**代码位置**：`OAuth2TokenServiceImpl.getAccessToken()` 方法

---

#### 步骤 2：检查权限校验的缓存链路

权限校验涉及多层缓存，让我们逐层排查：

##### 2.1 第一层：用户角色缓存

**检查 Redis 中用户的角色列表**：
```bash
# 格式：user_role_ids:{userId}
redis-cli GET "user_role_ids:100"  # 假设用户 ID 是 100
```

**对应代码**：`PermissionServiceImpl.getUserRoleIdListByUserIdFromCache()` 
- 注解：`@Cacheable(value = USER_ROLE_ID_LIST, key = "#userId")`

**如果缓存中角色列表未更新**：
- 可能原因：修改角色时清除的是 `MENU_ROLE_ID_LIST` 和 `PERMISSION_MENU_ID_LIST`，不是 `USER_ROLE_ID_LIST`
- 看 `assignRoleMenu` 方法：
  ```java
  @Caching(evict = {
      @CacheEvict(value = MENU_ROLE_ID_LIST, allEntries = true),
      @CacheEvict(value = PERMISSION_MENU_ID_LIST, allEntries = true)
  })
  ```
  **注意**：这里没有清除 `USER_ROLE_ID_LIST`！

---

##### 2.2 第二层：角色信息缓存

**检查 Redis 中的角色信息**：
```bash
# 格式：role:{roleId}
redis-cli GET "role:1"  # 假设角色 ID 是 1
```

**对应代码**：`RoleServiceImpl.getRoleFromCache()`
- 注解：`@Cacheable(value = ROLE, key = "#id")`

**如果角色状态被禁用但缓存未更新**：
- 检查 `updateRole` 方法是否清除了缓存：
  ```java
  @CacheEvict(value = ROLE, key = "#updateReqVO.id")
  public void updateRole(RoleSaveReqVO updateReqVO) { ... }
  ```
  这个是有清除的。

---

##### 2.3 第三层：权限标识 → 菜单映射缓存

**检查 Redis 中的权限菜单映射**：
```bash
# 格式：permission_menu_ids:{permission}
redis-cli GET "permission_menu_ids:system:user:delete"
```

**对应代码**：`MenuServiceImpl.getMenuIdListByPermissionFromCache()`
- 注解：`@Cacheable(value = PERMISSION_MENU_ID_LIST, key = "#permission")`

---

##### 2.4 第四层：菜单 → 角色映射缓存

**检查 Redis 中的菜单角色映射**：
```bash
# 格式：menu_role_ids:{menuId}
redis-cli GET "menu_role_ids:100"  # 假设"用户删除"菜单 ID 是 100
```

**对应代码**：`PermissionServiceImpl.getMenuRoleIdListByMenuIdFromCache()`
- 注解：`@Cacheable(value = MENU_ROLE_ID_LIST, key = "#menuId")`

---

#### 步骤 3：定位缓存失效的具体原因

根据 `assignRoleMenu` 方法的缓存清除逻辑：

```java
@Caching(evict = {
    @CacheEvict(value = MENU_ROLE_ID_LIST, allEntries = true),
    @CacheEvict(value = PERMISSION_MENU_ID_LIST, allEntries = true)
})
public void assignRoleMenu(Long roleId, Set<Long> menuIds) { ... }
```

**应该被清除的缓存**：
1. ✅ `MENU_ROLE_ID_LIST`（全部）
2. ✅ `PERMISSION_MENU_ID_LIST`（全部）

**不应该影响但可能有问题的**：
1. `USER_ROLE_ID_LIST`：没有被清除，但这是"用户拥有哪些角色"，修改角色的菜单不应该影响这个
2. `ROLE`：没有被清除，但角色信息本身没有变化

---

#### 步骤 4：检查是否是多租户问题

**如果是多租户环境**：

检查 `TenantRedisCacheManager` 的实现：
```java
public Cache getCache(String name) {
    String[] names = StrUtil.splitToArray(name, SPLIT);
    if (!TenantContextHolder.isIgnore()
            && TenantContextHolder.getTenantId() != null
            && !CollUtil.contains(ignoreCaches, names[0])) {
        name = name + ":" + TenantContextHolder.getTenantId();
    }
    return super.getCache(name);
}
```

**可能的问题**：
1. 管理员操作时的租户 ID 与用户请求时的租户 ID 不一致
2. 检查清除缓存时的 `TenantContextHolder.getTenantId()` 是否正确

**验证方法**：
```bash
# 多租户环境下，Key 会带有租户 ID
# 格式：{cacheName}:{tenantId}:{key}
redis-cli KEYS "menu_role_ids:*"
# 可能看到：menu_role_ids:100:1 (租户100的菜单1)
#          menu_role_ids:200:1 (租户200的菜单1)
```

---

#### 步骤 5：检查是否是 Spring Cache 的 beforeInvocation 问题

**Spring `@CacheEvict` 默认行为**：
- `beforeInvocation = false`（默认）：方法成功执行后才清除缓存
- `beforeInvocation = true`：方法执行前就清除缓存

**可能的问题场景**：
1. 方法执行过程中抛出异常（但被捕获了），缓存没有被清除
2. 事务回滚了，但缓存清除注解的执行顺序问题

**验证方法**：查看日志，确认 `assignRoleMenu` 方法是否正常完成，没有抛出异常。

---

#### 步骤 6：检查是否有其他地方重写了缓存

**场景**：在清除缓存后、数据库更新完成前，有其他请求把旧数据重新加载进缓存了。

**时间线问题**：
```
T0: 管理员调用 assignRoleMenu(roleId=1, newMenuIds=[...])
T1: Spring Cache 清除 MENU_ROLE_ID_LIST 所有缓存
T2: 用户张三发起请求，缓存未命中
T3: 张三的请求从数据库加载旧数据（事务还没提交），写入缓存
T4: 管理员的事务提交，数据库更新为新数据
T5: 但缓存中已经是旧数据了！
```

**但这应该不会发生**，因为：
1. Spring Cache 默认 `beforeInvocation = false`
2. 数据库事务隔离级别（默认是可重复读）

**除非**：使用了 `beforeInvocation = true`，或者清除缓存和数据库操作不在同一个事务中。

---

#### 步骤 7：最可能的原因 - 用户未重新登录 + Token 缓存的用户信息

等等，让我再仔细看一下权限校验的流程...

**权限校验是实时的**，每次请求都会重新计算。但有一个地方可能有问题：

看 `hasAnyPermission` 方法：
```java
private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    List<Long> menuIds = menuService.getMenuIdListByPermissionFromCache(permission);
    // ...
    for (Long menuId : menuIds) {
        Set<Long> menuRoleIds = getSelf().getMenuRoleIdListByMenuIdFromCache(menuId);
        if (CollUtil.containsAny(menuRoleIds, roleIds)) {
            return true;
        }
    }
    return false;
}
```

**关键点**：
- `menuIds = 权限标识 → 菜单列表` 的映射
- `menuRoleIds = 菜单 → 角色列表` 的映射

**如果移除了角色对菜单的授权**：
- 应该清除 `MENU_ROLE_ID_LIST`（菜单→角色映射）
- `assignRoleMenu` 确实清除了这个（`allEntries = true`）

---

#### 步骤 8：终极排查 - 直接对比数据库和缓存

**数据库查询**：
```sql
-- 查看角色实际拥有的菜单
SELECT menu_id FROM system_role_menu WHERE role_id = {角色ID};

-- 查看菜单实际被哪些角色拥有
SELECT role_id FROM system_role_menu WHERE menu_id = {菜单ID};
```

**Redis 查询**：
```bash
# 查看缓存中的菜单角色映射
redis-cli GET "menu_role_ids:{菜单ID}"

# 如果是多租户
redis-cli GET "menu_role_ids:{租户ID}:{菜单ID}"
```

**对比结果**：
- 如果数据库和缓存不一致 → 缓存清除失败
- 如果一致 → 问题在其他地方（可能是用户没有这个角色了，或者前端问题）

---

### 5.3 常见原因总结与解决方案

| 可能原因 | 排查方法 | 解决方案 |
|---------|---------|---------|
| 缓存清除失败（Redis 连接问题） | 检查 Redis 日志，看 `DEL` 命令是否执行成功 | 确保 Redis 高可用配置 |
| 多租户 Key 不匹配 | 对比管理员操作和用户请求时的 `TenantContextHolder.getTenantId()` | 确保清除缓存时租户上下文正确 |
| 清除的缓存 Key 不对 | 检查 `assignRoleMenu` 方法的 `@CacheEvict` 注解 | 确认是否遗漏了需要清除的缓存 |
| 用户角色没有更新 | 检查用户是否真的拥有该角色 | 可能需要检查 `assignUserRole` 是否被调用 |
| 前端缓存问题 | 清除浏览器缓存或刷新页面 | 前端可能缓存了菜单数据 |
| 权限标识拼写错误 | 检查接口上的 `@PreAuthorize` 注解 | 确保权限标识和菜单配置一致 |

---

## 六、缓存类型对比总结

| 特性 | 用户权限缓存 | 字典缓存 | OAuth2 Token 缓存 |
|-----|------------|---------|------------------|
| **缓存类型** | Redis + Spring Cache | Guava LoadingCache（本地） | Redis（手动操作） |
| **Key 组成** | `{cacheName}:{key}` | `dictType` 字符串 | `oauth2_access_token:{token}` |
| **多租户支持** | ✅ `TenantRedisCacheManager` 自动拼接 | ❌ 无特殊处理 | ❌ 无特殊处理（Token 本身带租户信息） |
| **失效策略** | 主动清除（`@CacheEvict`） | 被动过期（1 分钟） | 主动删除 + TTL 自动过期 |
| **集群一致性** | ✅ Redis 分布式 | ❌ 本地缓存，最多 2 分钟不一致 | ✅ Redis 分布式 |
| **主要风险** | 批量清除可能导致缓存雪崩 | 本地缓存不一致窗口 | 双写一致性（数据库和 Redis） |
| **典型场景** | 权限校验、菜单管理 | 前端字典下拉框、Excel 导出 | 用户登录态管理 |

---

## 七、集群部署缓存一致性建议

### 7.1 针对本地缓存（Guava LoadingCache）的改进方案

**方案 1：改用 Redis 缓存**
- 优点：彻底解决一致性问题
- 缺点：增加网络开销，字典读取频率很高

**方案 2：引入缓存失效广播机制**
- 使用 Redis Pub/Sub 或 MQ
- 当字典更新时，发布消息通知所有节点清除本地缓存
- 参考实现：
  ```java
  // 伪代码
  public void updateDictData(DictDataSaveReqVO updateReqVO) {
      dictDataMapper.updateById(updateObj);
      // 广播缓存失效消息
      redisTemplate.convertAndSend("cache:invalidation:dict", updateReqVO.getDictType());
  }
  
  // 订阅消息
  @RedisListener(topic = "cache:invalidation:dict")
  public void onDictInvalidation(String dictType) {
      DictFrameworkUtils.clearCache(dictType);  // 需要实现按类型清除
  }
  ```

**方案 3：缩短过期时间**
- 把 1 分钟缩短到 10-30 秒
- 优点：简单
- 缺点：增加数据库压力

### 7.2 针对 Spring Cache 注解的注意事项

1. **避免使用 `allEntries = true`**：尽量精确清除，减少缓存失效范围
2. **注意清除顺序**：先更新数据库，再清除缓存（Spring 默认行为）
3. **多租户环境**：确保清除缓存时的租户上下文正确
4. **异常处理**：考虑 Redis 不可用时的降级策略

### 7.3 针对 Token 缓存的注意事项

1. **删除操作的幂等性**：即使 Redis 删除失败，数据库删除成功，下次 Token 校验也会失败（因为数据库查不到）
2. **Redis TTL 与数据库 expires_time 保持一致**：代码中已经处理
3. **考虑使用 Redis Lua 脚本**：保证数据库和 Redis 操作的原子性（但 MySQL 和 Redis 无法真正原子）
