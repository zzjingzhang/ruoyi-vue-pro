# RuoYi-Vue-Pro 后端认证与前端路由完整流程分析

## 一、项目架构概述

RuoYi-Vue-Pro 采用 **Spring Security + OAuth2 + JWT-like Token + Redis** 的认证授权方案，前端使用 **Vue3 + Vue Router + Pinia** 实现动态路由。

### 核心组件关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                        前端 (Vue3 Admin)                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐     │
│  │ 登录页       │  │ permission  │  │ 动态路由 (Vue Router)   │     │
│  └──────┬──────┘  │   store     │  └──────────┬──────────────┘     │
│         │         └──────┬──────┘             │                     │
│         └────────────────┼────────────────────┘                     │
│                          │                                           │
│                    HTTP Request (Bearer Token)                      │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────────┐
│                       后端 (Spring Boot)                            │
│  ┌───────────────────────▼──────────────────────────────────────┐   │
│  │              Spring Security 过滤链                            │   │
│  │  ┌──────────────────────────────────────────────────────┐    │   │
│  │  │ 1. CorsFilter                                        │    │   │
│  │  │ 2. TokenAuthenticationFilter (核心)                   │    │   │
│  │  │ 3. FilterSecurityInterceptor                        │    │   │
│  │  │ 4. ExceptionTranslationFilter                       │    │   │
│  │  └──────────────────────────────────────────────────────┘    │   │
│  └───────────────────────────────┬──────────────────────────────┘   │
│                                  │                                   │
│  ┌───────────────────────────────▼──────────────────────────────┐   │
│  │                    OAuth2TokenService                        │   │
│  │  - 创建/刷新/验证 AccessToken/RefreshToken                   │   │
│  │  - Redis + MySQL 双重存储                                     │   │
│  └───────────────────────────────┬──────────────────────────────┘   │
│                                  │                                   │
│  ┌───────────────────────────────▼──────────────────────────────┐   │
│  │                    PermissionService                         │   │
│  │  - 用户-角色-菜单关联查询                                      │   │
│  │  - 权限标识校验 (@PreAuthorize)                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
                           │
         ┌─────────────────┴─────────────────┐
         ▼                                   ▼
┌──────────────────┐              ┌──────────────────┐
│     MySQL        │              │     Redis        │
│  - system_user   │              │  - oauth2_access │
│  - system_role   │              │    _token:{token}│
│  - system_menu   │              │  - user_role_ids │
│  - system_user_  │              │    :{userId}     │
│    role          │              │  - menu_role_ids │
│  - system_role_  │              │    :{menuId}     │
│    menu          │              │  - permission_   │
│                  │              │    menu_ids:     │
│                  │              │    {permission}  │
└──────────────────┘              └──────────────────┘
```

---

## 二、完整流程追踪

### 2.1 登录接口与 Token 生成

#### 2.1.1 登录入口

**关键类和方法：**

| 组件 | 位置 | 说明 |
|------|------|------|
| 登录 Controller | `AuthController.java:66-71` | `POST /system/auth/login` |
| 登录 Service | `AdminAuthServiceImpl.java:102-117` | `login()` 方法 |
| Token 生成 | `OAuth2TokenServiceImpl.java:63-69` | `createAccessToken()` |

**调用链：**

```
前端 POST /system/auth/login
    ↓
AuthController.login()
    ↓
AdminAuthServiceImpl.login()
    ├─ validateCaptcha()          // 图形验证码校验
    ├─ authenticate()             // 用户名密码校验
    └─ createTokenAfterLoginSuccess()
         └─ OAuth2TokenServiceImpl.createAccessToken()
              ├─ validOAuthClientFromCache()
              ├─ createOAuth2RefreshToken()   // 生成 refresh_token
              └─ createOAuth2AccessToken()    // 生成 access_token
                   ├─ 写入 MySQL: oauth2_access_token
                   └─ 写入 Redis: oauth2_access_token:{token}
```

#### 2.1.2 Token 存储位置

**后端存储 (双重存储机制)：**

1. **MySQL 持久化存储**
   - 表：`oauth2_access_token` 和 `oauth2_refresh_token`
   - 位置：`yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/mysql/oauth2/`
   - 包含字段：access_token, refresh_token, user_id, user_type, tenant_id, expires_time 等

2. **Redis 缓存 (高性能访问)**
   - Key 格式：`oauth2_access_token:{token}`
   - 类：`OAuth2AccessTokenRedisDAO.java`
   - 过期时间：与 token 本身的 expiresTime 一致
   - 数据结构：JSON 序列化的 `OAuth2AccessTokenDO`

**Token 生成逻辑 (OAuth2TokenServiceImpl.java:177-194)：**

```java
private OAuth2AccessTokenDO createOAuth2AccessToken(
    OAuth2RefreshTokenDO refreshTokenDO, OAuth2ClientDO clientDO) {
    
    // 1. 生成 UUID 格式的 Token
    String accessToken = IdUtil.fastSimpleUUID();  // 如: "a1b2c3d4e5f6..."
    
    // 2. 构建 Token 对象
    OAuth2AccessTokenDO accessTokenDO = new OAuth2AccessTokenDO()
        .setAccessToken(accessToken)
        .setUserId(refreshTokenDO.getUserId())
        .setUserType(refreshTokenDO.getUserType())
        .setUserInfo(buildUserInfo(userId, userType))  // {nickname, deptId}
        .setClientId(clientDO.getClientId())
        .setRefreshToken(refreshTokenDO.getRefreshToken())
        .setExpiresTime(LocalDateTime.now()
            .plusSeconds(clientDO.getAccessTokenValiditySeconds()));
        .setTenantId(tenantId);
    
    // 3. 双写：MySQL + Redis
    oauth2AccessTokenMapper.insert(accessTokenDO);
    oauth2AccessTokenRedisDAO.set(accessTokenDO);
    
    return accessTokenDO;
}
```

**前端存储：**

根据 RuoYi-Vue-Pro 架构，前端 Token 存储在：
- **localStorage**：持久化存储，刷新页面不丢失
- **内存**：Pinia/Vuex store 中，用于快速访问
- **HTTP Header**：每次请求通过 `Authorization: Bearer {token}` 发送

---

### 2.2 Spring Security 过滤链配置

#### 2.2.1 核心配置类

**关键配置类：**

| 类名 | 位置 | 作用 |
|------|------|------|
| `YudaoWebSecurityConfigurerAdapter` | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/.../config/` | 主 Security 配置 |
| `YudaoSecurityAutoConfiguration` | 同上 | 自动配置类，注册 Bean |
| `TokenAuthenticationFilter` | `core/filter/` | Token 认证过滤器 (核心) |

**Security 配置详解 (YudaoWebSecurityConfigurerAdapter.java:109-153)：**

```java
@Bean
protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    
    // 1. 基础配置
    httpSecurity
        .cors(Customizer.withDefaults())           // 开启跨域
        .csrf(AbstractHttpConfigurer::disable)     // 禁用 CSRF (Token 机制不需要)
        .sessionManagement(c -> 
            c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))  // 无状态，不使用 Session
        .headers(c -> c.frameOptions(
            HeadersConfigurer.FrameOptionsConfig::disable));
    
    // 2. 异常处理
    httpSecurity.exceptionHandling(c -> 
        c.authenticationEntryPoint(authenticationEntryPoint)  // 401 处理
         .accessDeniedHandler(accessDeniedHandler));          // 403 处理
    
    // 3. URL 权限规则
    httpSecurity.authorizeHttpRequests(c -> {
        // 3.1 静态资源放行
        c.requestMatchers(HttpMethod.GET, 
            "/*.html", "/*.css", "/*.js").permitAll();
        
        // 3.2 @PermitAll 注解的接口自动放行
        // 例如: /system/auth/login, /system/auth/refresh-token 等
        c.requestMatchers(HttpMethod.GET, 
            permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll();
        // ... 其他 HTTP 方法
        
        // 3.3 配置文件中的放行 URL
        c.requestMatchers(securityProperties
            .getPermitAllUrls().toArray(new String[0])).permitAll();
    });
    
    // 4. 各模块自定义规则 (AuthorizeRequestsCustomizer)
    httpSecurity.authorizeHttpRequests(c -> 
        authorizeRequestsCustomizers.forEach(customizer -> 
            customizer.customize(c)));
    
    // 5. 兜底规则：所有请求必须认证
    httpSecurity.authorizeHttpRequests(c -> 
        c.dispatcherTypeMatchers(DispatcherType.ASYNC).permitAll()
         .anyRequest().authenticated());
    
    // 6. 添加自定义 Token 过滤器
    // 位置: UsernamePasswordAuthenticationFilter 之前
    httpSecurity.addFilterBefore(
        authenticationTokenFilter, 
        UsernamePasswordAuthenticationFilter.class);
    
    return httpSecurity.build();
}
```

#### 2.2.2 过滤链执行顺序

```
请求进入
    ↓
┌─────────────────────────────────────────┐
│ 1. CorsFilter (跨域处理)                 │
└──────────────────┬──────────────────────┘
                   ↓
┌──────────────────▼──────────────────────┐
│ 2. TokenAuthenticationFilter (核心)      │
│    - 从 Header/Parameter 提取 Token      │
│    - 调用 OAuth2TokenService 校验        │
│    - 构建 LoginUser                      │
│    - 注入 SecurityContext                │
└──────────────────┬──────────────────────┘
                   ↓
┌──────────────────▼──────────────────────┐
│ 3. FilterSecurityInterceptor            │
│    - URL 级别的权限校验 (permitAll 等)   │
└──────────────────┬──────────────────────┘
                   ↓
┌──────────────────▼──────────────────────┐
│ 4. MethodSecurityInterceptor            │
│    - 方法级权限校验 (@PreAuthorize)      │
│    - 调用 SecurityFrameworkService       │
└──────────────────┬──────────────────────┘
                   ↓
              Controller 方法
```

---

### 2.3 Token 认证与 LoginUser 注入

#### 2.3.1 Token 认证过滤器

**核心类：** `TokenAuthenticationFilter.java` (继承 `OncePerRequestFilter`)

**执行流程 (TokenAuthenticationFilter.java:41-69)：**

```java
@Override
protected void doFilterInternal(HttpServletRequest request, 
    HttpServletResponse response, FilterChain chain)
    throws ServletException, IOException {
    
    // ========== 步骤 1: 提取 Token ==========
    // 优先级: Header > Parameter
    // Header: Authorization: Bearer {token}
    // Parameter: ?token={token}
    String token = SecurityFrameworkUtils.obtainAuthorization(
        request,
        securityProperties.getTokenHeader(),      // 默认: "Authorization"
        securityProperties.getTokenParameter()    // 默认: "token"
    );
    
    if (StrUtil.isNotEmpty(token)) {
        Integer userType = WebFrameworkUtils.getLoginUserType(request);
        
        try {
            // ========== 步骤 2: 构建 LoginUser ==========
            LoginUser loginUser = buildLoginUserByToken(token, userType);
            
            // 调试模式：支持 mock token (以 mockSecret 开头)
            if (loginUser == null) {
                loginUser = mockLoginUser(request, token, userType);
            }
            
            // ========== 步骤 3: 注入到 SecurityContext ==========
            if (loginUser != null) {
                SecurityFrameworkUtils.setLoginUser(loginUser, request);
            }
        } catch (Throwable ex) {
            // 异常处理：直接返回 JSON 错误
            CommonResult<?> result = globalExceptionHandler
                .allExceptionHandler(request, ex);
            ServletUtils.writeJSON(response, result);
            return;
        }
    }
    
    // ========== 步骤 4: 继续过滤链 ==========
    chain.doFilter(request, response);
}
```

#### 2.3.2 Token 校验与 LoginUser 构建

**Token 校验逻辑 (TokenAuthenticationFilter.java:71-93)：**

```java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    try {
        // 1. 调用 OAuth2TokenApi 校验 Token
        // 内部会先查 Redis，再查 MySQL
        OAuth2AccessTokenCheckRespDTO accessToken = 
            oauth2TokenApi.checkAccessToken(token);
        
        if (accessToken == null) {
            return null;
        }
        
        // 2. 用户类型校验 (admin-api vs app-api)
        if (userType != null 
            && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
            throw new AccessDeniedException("错误的用户类型");
        }
        
        // 3. 构建 LoginUser 对象
        return new LoginUser()
            .setId(accessToken.getUserId())
            .setUserType(accessToken.getUserType())
            .setInfo(accessToken.getUserInfo())           // {nickname, deptId}
            .setTenantId(accessToken.getTenantId())
            .setScopes(accessToken.getScopes())
            .setExpiresTime(accessToken.getExpiresTime());
            
    } catch (ServiceException serviceException) {
        // Token 无效时返回 null，后续由 Spring Security 处理
        return null;
    }
}
```

**Token 校验的实现 (OAuth2TokenServiceImpl.java:103-140)：**

```java
@Override
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // ========== 优先从 Redis 获取 ==========
    OAuth2AccessTokenDO accessTokenDO = 
        oauth2AccessTokenRedisDAO.get(accessToken);
    
    if (accessTokenDO != null) {
        return accessTokenDO;
    }
    
    // ========== Redis 未命中，从 MySQL 获取 ==========
    accessTokenDO = oauth2AccessTokenMapper
        .selectByAccessToken(accessToken);
    
    // 特殊兼容：某些场景传递的是 refresh_token
    if (accessTokenDO == null) {
        OAuth2RefreshTokenDO refreshTokenDO = 
            oauth2RefreshTokenMapper.selectByRefreshToken(accessToken);
        if (refreshTokenDO != null && 
            !DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
            accessTokenDO = convertToAccessToken(refreshTokenDO);
        }
    }
    
    // ========== 回写到 Redis (缓存预热) ==========
    if (accessTokenDO != null && 
        !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    
    return accessTokenDO;
}

@Override
public OAuth2AccessTokenDO checkAccessToken(String accessToken) {
    OAuth2AccessTokenDO accessTokenDO = getAccessToken(accessToken);
    
    // 校验 1: Token 是否存在
    if (accessTokenDO == null) {
        throw exception0(UNAUTHORIZED.getCode(), "访问令牌不存在");
    }
    
    // 校验 2: Token 是否已过期
    if (DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        throw exception0(UNAUTHORIZED.getCode(), "访问令牌已过期");
    }
    
    return accessTokenDO;
}
```

#### 2.3.3 LoginUser 注入 SecurityContext

**注入逻辑 (SecurityFrameworkUtils.java:122-133)：**

```java
public static void setLoginUser(LoginUser loginUser, 
    HttpServletRequest request) {
    
    // 1. 创建 Authentication 对象
    // 使用 UsernamePasswordAuthenticationToken (非用户名密码认证，只是借用)
    Authentication authentication = buildAuthentication(loginUser, request);
    
    // 2. 设置到 SecurityContextHolder
    // 策略: TransmittableThreadLocal (支持线程池传递)
    SecurityContextHolder.getContext().setAuthentication(authentication);
    
    // 3. 额外设置到 request 中
    // 用于 ApiAccessLogFilter 等在 Security Filter 之后的过滤器获取
    if (request != null) {
        WebFrameworkUtils.setLoginUserId(request, loginUser.getId());
        WebFrameworkUtils.setLoginUserType(request, loginUser.getUserType());
    }
}

private static Authentication buildAuthentication(
    LoginUser loginUser, HttpServletRequest request) {
    
    // Principal: LoginUser 对象
    // Credentials: null (Token 机制不需要)
    // Authorities: Collections.emptyList() (权限动态校验，不使用 GrantedAuthority)
    UsernamePasswordAuthenticationToken authenticationToken = 
        new UsernamePasswordAuthenticationToken(
            loginUser, null, Collections.emptyList());
    
    // 设置请求详情 (IP, SessionId 等)
    authenticationToken.setDetails(
        new WebAuthenticationDetailsSource().buildDetails(request));
    
    return authenticationToken;
}
```

**LoginUser 数据结构 (LoginUser.java)：**

```java
@Data
public class LoginUser {
    private Long id;                    // 用户ID
    private Integer userType;           // 用户类型 (ADMIN=1, MEMBER=2)
    private Map<String, String> info;   // 额外信息 {nickname, deptId}
    private Long tenantId;              // 租户ID
    private List<String> scopes;        // OAuth2 授权范围
    private LocalDateTime expiresTime;  // Token 过期时间
    
    // 上下文缓存 (不持久化)
    @JsonIgnore
    private Map<String, Object> context;
    private Long visitTenantId;         // 访问的租户ID (跨租户)
}
```

---

### 2.4 权限标识加载与后端接口鉴权

#### 2.4.1 数据表关系

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ system_user  │       │ system_role  │       │ system_menu  │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id           │       │ id           │       │ id           │
│ username     │       │ code         │       │ permission   │
│ password     │       │ status       │       │ type         │
│ status       │       │ data_scope   │       │ path         │
│ dept_id      │       │ ...          │       │ component    │
└──────┬───────┘       └──────┬───────┘       └──────┬───────┘
       │                      │                      │
       │  ┌──────────────┐    │  ┌──────────────┐    │
       └──┤system_user_  ├────┘  │system_role_  ├────┘
          │   role       │       │   menu       │
          ├──────────────┤       ├──────────────┤
          │ user_id      │       │ role_id      │
          │ role_id      │       │ menu_id      │
          └──────────────┘       └──────────────┘
```

**权限标识存储位置：**

| 表名 | 字段 | 说明 |
|------|------|------|
| `system_menu` | `permission` | 权限标识，如 `system:user:list` |
| `system_menu` | `type` | 菜单类型 (1=目录, 2=菜单, 3=按钮) |
| `system_role` | `code` | 角色标识，如 `super_admin` |

**权限标识示例：**

```sql
-- 查询用户的所有权限标识
SELECT DISTINCT m.permission
FROM system_user u
JOIN system_user_role ur ON u.id = ur.user_id
JOIN system_role r ON ur.role_id = r.id AND r.status = 0
JOIN system_role_menu rm ON r.id = rm.role_id
JOIN system_menu m ON rm.menu_id = m.id 
    AND m.status = 0 
    AND m.permission IS NOT NULL 
    AND m.permission != ''
WHERE u.id = {userId};
```

#### 2.4.2 权限服务实现

**核心类：** `PermissionServiceImpl.java`

**权限校验流程 (PermissionServiceImpl.java:63-84)：**

```java
@Override
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // 空权限标识 = 放行
    if (ArrayUtil.isEmpty(permissions)) {
        return true;
    }
    
    // ========== 步骤 1: 获取用户的启用角色 ==========
    // 带 Redis 缓存: user_role_ids:{userId}
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roles)) {
        return false;
    }
    
    // ========== 步骤 2: 逐个权限标识校验 ==========
    for (String permission : permissions) {
        if (hasAnyPermission(roles, permission)) {
            return true;
        }
    }
    
    // ========== 步骤 3: 超级管理员特殊处理 ==========
    // 角色 type = 2 (SYSTEM) 视为超管
    return roleService.hasAnySuperAdmin(
        convertSet(roles, RoleDO::getId));
}
```

**单个权限标识校验 (PermissionServiceImpl.java:93-111)：**

```java
private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    // ========== 步骤 1: 根据权限标识查菜单 ==========
    // 缓存: permission_menu_ids:{permission}
    List<Long> menuIds = menuService
        .getMenuIdListByPermissionFromCache(permission);
    
    // 严格模式：找不到菜单 = 无权限
    if (CollUtil.isEmpty(menuIds)) {
        return false;
    }
    
    // ========== 步骤 2: 查询每个菜单的拥有角色 ==========
    Set<Long> roleIds = convertSet(roles, RoleDO::getId);
    
    for (Long menuId : menuIds) {
        // 缓存: menu_role_ids:{menuId}
        Set<Long> menuRoleIds = getSelf()
            .getMenuRoleIdListByMenuIdFromCache(menuId);
        
        // 角色交集判断
        if (CollUtil.containsAny(menuRoleIds, roleIds)) {
            return true;
        }
    }
    
    return false;
}
```

#### 2.4.3 方法级权限校验

**权限服务封装 (SecurityFrameworkServiceImpl.java)：**

```java
@Service("ss")  // Spring EL 中使用 @PreAuthorize("@ss.hasPermission(...)")
public class SecurityFrameworkServiceImpl 
    implements SecurityFrameworkService {
    
    @Override
    public boolean hasPermission(String permission) {
        return hasAnyPermissions(permission);
    }
    
    @Override
    public boolean hasAnyPermissions(String... permissions) {
        // 特殊: 跨租户访问跳过权限校验
        if (skipPermissionCheck()) {
            return true;
        }
        
        // 获取当前登录用户ID
        Long userId = getLoginUserId();
        if (userId == null) {
            return false;
        }
        
        // 调用权限服务
        return permissionApi.hasAnyPermissions(userId, permissions);
    }
}
```

**使用示例：**

```java
@GetMapping("/list")
@Operation(summary = "获得用户分页列表")
@PreAuthorize("@ss.hasPermission('system:user:query')")
public CommonResult<PageResult<UserRespVO>> getUserPage(...) {
    // ...
}

@PostMapping("/create")
@Operation(summary = "创建用户")
@PreAuthorize("@ss.hasPermission('system:user:create')")
public CommonResult<Long> createUser(...) {
    // ...
}
```

#### 2.4.4 缓存机制

**Redis Key 设计 (RedisKeyConstants.java)：**

| Key 格式 | 缓存内容 | 更新时机 |
|----------|----------|----------|
| `user_role_ids:{userId}` | 用户的角色ID集合 | 用户角色变更时 |
| `menu_role_ids:{menuId}` | 拥有该菜单的角色ID集合 | 角色菜单变更时 |
| `permission_menu_ids:{permission}` | 权限标识对应的菜单ID集合 | 菜单变更时 |
| `role:{roleId}` | 角色详情 | 角色变更时 |
| `oauth2_access_token:{token}` | Token 详情 | Token 创建/刷新时 |

**缓存注解使用示例 (PermissionServiceImpl.java)：**

```java
// 读缓存
@Override
@Cacheable(value = RedisKeyConstants.USER_ROLE_ID_LIST, key = "#userId")
public Set<Long> getUserRoleIdListByUserIdFromCache(Long userId) {
    return getUserRoleIdListByUserId(userId);
}

// 清缓存 (角色菜单变更时)
@Override
@Caching(evict = {
    @CacheEvict(value = RedisKeyConstants.MENU_ROLE_ID_LIST, allEntries = true),
    @CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, allEntries = true)
})
public void assignRoleMenu(Long roleId, Set<Long> menuIds) {
    // ...
}
```

---

### 2.5 前端路由与 permission store

#### 2.5.1 前端权限信息获取

**后端接口：** `GET /system/auth/get-permission-info`

**实现位置：** `AuthController.java:93-118`

```java
@GetMapping("/get-permission-info")
@Operation(summary = "获取登录用户的权限信息")
public CommonResult<AuthPermissionInfoRespVO> getPermissionInfo() {
    // 1. 获取用户信息
    AdminUserDO user = userService.getUser(getLoginUserId());
    
    // 2. 获取用户角色
    Set<Long> roleIds = permissionService
        .getUserRoleIdListByUserId(getLoginUserId());
    List<RoleDO> roles = roleService.getRoleList(roleIds);
    roles.removeIf(role -> !CommonStatusEnum.ENABLE
        .getStatus().equals(role.getStatus()));
    
    // 3. 获取用户菜单 (根据角色)
    Set<Long> menuIds = permissionService
        .getRoleMenuListByRoleId(convertSet(roles, RoleDO::getId));
    List<MenuDO> menuList = menuService.getMenuList(menuIds);
    menuList = menuService.filterDisableMenus(menuList);
    
    // 4. 转换返回
    return success(AuthConvert.INSTANCE
        .convert(user, roles, menuList));
}
```

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
      "component": null,
      "icon": "system",
      "type": 1,
      "children": [...]
    }
  ]
}
```

#### 2.5.2 前端 permission store (Pinia/Vuex)

**基于 RuoYi-Vue-Pro 标准架构，permission store 的核心逻辑：**

```typescript
// stores/permission.ts
import { defineStore } from 'pinia'
import { router, routes } from '@/router'
import { getPermissionInfo } from '@/api/system/auth'
import { transformMenusToRoutes } from '@/utils/router'

export const usePermissionStore = defineStore('permission', {
  state: () => ({
    routes: [] as any[],           // 完整路由表
    addRoutes: [] as any[],        // 动态添加的路由
    menus: [] as any[],            // 菜单树
    roles: [] as string[],         // 角色标识
    permissions: [] as string[]    // 权限标识 (从菜单提取)
  }),

  actions: {
    /**
     * 生成动态路由
     */
    async generateRoutes() {
      // 1. 调用后端接口获取权限信息
      const { data } = await getPermissionInfo()
      
      // 2. 存储角色和菜单
      this.roles = data.roles
      this.menus = data.menus
      
      // 3. 提取权限标识 (从菜单的 permission 字段)
      this.permissions = this.extractPermissions(data.menus)
      
      // 4. 将菜单树转换为 Vue Router 路由表
      const accessRoutes = transformMenusToRoutes(data.menus)
      
      // 5. 动态添加路由
      this.addRoutes = accessRoutes
      this.routes = routes.concat(accessRoutes)
      
      accessRoutes.forEach(route => {
        router.addRoute(route)  // Vue Router 4.x API
      })
      
      return accessRoutes
    },

    /**
     * 递归提取权限标识
     */
    extractPermissions(menus: any[]): string[] {
      const permissions: string[] = []
      menus.forEach(menu => {
        if (menu.permission) {
          permissions.push(menu.permission)
        }
        if (menu.children) {
          permissions.push(...this.extractPermissions(menu.children))
        }
      })
      return permissions
    },

    /**
     * 重置路由 (登出时)
     */
    resetRoutes() {
      this.addRoutes.forEach(route => {
        const name = route.name
        if (name && router.hasRoute(name)) {
          router.removeRoute(name)
        }
      })
      this.routes = []
      this.addRoutes = []
      this.menus = []
      this.roles = []
      this.permissions = []
    }
  }
})
```

#### 2.5.3 路由守卫 (router/permission.ts)

```typescript
import { router } from './index'
import { useUserStore } from '@/stores/user'
import { usePermissionStore } from '@/stores/permission'
import { getToken } from '@/utils/auth'

const whiteList = ['/login', '/auth-redirect']

router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore()
  const permissionStore = usePermissionStore()
  const hasToken = getToken()  // 从 localStorage 读取
  
  if (hasToken) {
    if (to.path === '/login') {
      next({ path: '/' })
    } else {
      // 是否已获取用户信息
      const hasGetUserInfo = userStore.name
      
      if (hasGetUserInfo) {
        next()
      } else {
        try {
          // 1. 获取用户信息
          await userStore.getUserInfo()
          
          // 2. 生成动态路由
          const accessRoutes = await permissionStore.generateRoutes()
          
          // 3. hack: 确保 addRoute 生效
          next({ ...to, replace: true })
        } catch (error) {
          // 出错: 清除 Token 并跳转登录页
          await userStore.resetToken()
          next(`/login?redirect=${to.path}`)
        }
      }
    }
  } else {
    // 无 Token
    if (whiteList.indexOf(to.path) !== -1) {
      next()
    } else {
      next(`/login?redirect=${to.path}`)
    }
  }
})
```

#### 2.5.4 菜单转路由逻辑

```typescript
// utils/router.ts

/**
 * 后端菜单结构转 Vue Router 路由
 */
export function transformMenusToRoutes(menus: any[]): any[] {
  return menus
    .filter(menu => menu.type !== 3)  // 排除按钮类型
    .map(menu => {
      const route = {
        path: menu.path,
        name: menu.componentName || menu.name,
        component: resolveComponent(menu.component, menu.path),
        meta: {
          title: menu.name,
          icon: menu.icon,
          hidden: menu.visible === false,
          keepAlive: menu.keepAlive,
          permission: menu.permission
        },
        children: menu.children 
          ? transformMenusToRoutes(menu.children)
          : []
      }
      return route
    })
}

/**
 * 解析组件路径
 */
function resolveComponent(component: string, path: string) {
  if (component === 'Layout') {
    return () => import('@/layout/index.vue')
  }
  if (component === 'ParentView') {
    return () => import('@/layout/ParentView.vue')
  }
  // 动态导入组件
  return () => import(`@/views${component}.vue`)
}
```

---

### 2.6 刷新页面时的状态恢复

#### 2.6.1 恢复流程

```
用户刷新页面 (F5 / Command+R)
    ↓
Vue 应用重新初始化
    ↓
Router beforeEach 路由守卫触发
    ↓
┌─────────────────────────────────────────────┐
│ 1. 检查 Token (localStorage)                 │
│    - 有 Token: 继续                          │
│    - 无 Token: 重定向 /login                 │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────▼───────────────────────┐
│ 2. 检查是否已获取用户信息                     │
│    - 已获取 (Pinia store): 直接放行          │
│    - 未获取 (store 已清空): 重新获取         │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────▼───────────────────────┐
│ 3. 调用后端接口 (携带 Token)                 │
│    - GET /admin-api/system/auth/get-        │
│      permission-info                        │
│    - Token 在 Header 中: Authorization:     │
│      Bearer {token}                         │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────▼───────────────────────┐
│ 4. 后端 TokenAuthenticationFilter 校验      │
│    - 提取 Token                             │
│    - Redis → MySQL 校验                     │
│    - 注入 LoginUser                         │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────▼───────────────────────┐
│ 5. 后端返回权限信息                          │
│    - user, roles, menus                     │
└─────────────────────┬───────────────────────┘
                      ↓
┌─────────────────────▼───────────────────────┐
│ 6. 前端恢复状态                              │
│    - userStore 填充用户信息                 │
│    - permissionStore 生成动态路由            │
│    - router.addRoute() 添加路由              │
└─────────────────────┬───────────────────────┘
                      ↓
              页面正常渲染
```

#### 2.6.2 关键代码位置

| 功能 | 位置 | 说明 |
|------|------|------|
| Token 持久化 | `localStorage` | `getToken()` / `setToken()` |
| 状态检查 | `router.beforeEach` | 路由守卫 |
| 重新获取权限 | `permissionStore.generateRoutes()` | 动态路由生成 |
| 重新添加路由 | `router.addRoute()` | Vue Router API |

---

## 三、缓存失效分析

### 3.1 各环节缓存失效影响

#### 3.1.1 Token 缓存失效 (`oauth2_access_token:{token}`)

**缓存位置：** Redis

**失效场景：**
- Redis 重启
- Key 过期 (与 Token 过期时间一致)
- 手动删除 (踢下线)

**失效影响：**

| 影响点 | 表象 | 内部机制 |
|--------|------|----------|
| 首次请求 | 无感知，响应略慢 | Redis 未命中 → 查询 MySQL → 回写 Redis |
| 并发请求 | 可能出现短暂重复查库 | 缓存击穿 (无锁保护时) |
| MySQL 也失效 | 401 未授权 | 双重存储都失效 |

**代码层面的容错 (OAuth2TokenServiceImpl.java:103-128)：**

```java
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // 1. 先查 Redis
    OAuth2AccessTokenDO accessTokenDO = 
        oauth2AccessTokenRedisDAO.get(accessToken);
    
    if (accessTokenDO != null) {
        return accessTokenDO;
    }
    
    // 2. Redis 失效，查 MySQL
    accessTokenDO = oauth2AccessTokenMapper
        .selectByAccessToken(accessToken);
    
    // 3. 回写 Redis (缓存预热)
    if (accessTokenDO != null && !isExpired(...)) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    
    return accessTokenDO;
}
```

#### 3.1.2 用户角色缓存失效 (`user_role_ids:{userId}`)

**缓存位置：** Redis

**失效场景：**
- 用户角色变更
- Redis 重启
- 内存不足淘汰

**失效影响：**

| 影响点 | 表象 | 内部机制 |
|--------|------|----------|
| 权限查询 | 响应略慢 | 查库后回写 |
| 角色刚变更 | 可能短暂不生效 | 缓存未及时更新 |
| 权限判断错误 | 403 或越权 | 缓存与 DB 不一致 |

**缓存更新机制：**

```java
// 用户角色变更时清除缓存
@Override
@CacheEvict(value = RedisKeyConstants.USER_ROLE_ID_LIST, key = "#userId")
public void assignUserRole(Long userId, Set<Long> roleIds) {
    // ... 数据库操作
}
```

#### 3.1.3 菜单角色缓存失效 (`menu_role_ids:{menuId}`)

**失效场景：**
- 角色菜单配置变更
- 菜单删除

**失效影响：**

| 影响点 | 表象 |
|--------|------|
| 权限标识校验 | 403 (应该有权限却没有) 或越权 |
| 前端菜单显示 | 菜单不显示或显示不该显示的菜单 |

#### 3.1.4 权限-菜单缓存失效 (`permission_menu_ids:{permission}`)

**失效场景：**
- 菜单的 permission 字段修改
- 菜单新增/删除

**失效影响：**

| 权限标识 | 缓存状态 | 数据库状态 | 结果 |
|----------|----------|------------|------|
| `system:user:query` | 不存在 | 存在 | 返回 false (本应有权限) |
| `system:user:query` | 菜单ID=[100] | 菜单ID=[100,200] | 权限判断不完整 |
| `system:user:delete` | 菜单ID=[101] | 已删除 | 返回 true (本应无权限) |

### 3.2 缓存失效故障树

```
用户报告权限问题
    │
    ├─► 前端检查
    │    ├─ Token 是否存在 (localStorage)
    │    ├─ permissionStore 状态
    │    └─ 动态路由是否已加载
    │
    ├─► Token 层
    │    ├─ Redis: oauth2_access_token:{token}
    │    ├─ MySQL: oauth2_access_token 表
    │    └─ Token 是否过期
    │
    ├─► 权限层
    │    ├─ Redis: user_role_ids:{userId}
    │    ├─ Redis: menu_role_ids:{menuId}
    │    ├─ Redis: permission_menu_ids:{permission}
    │    └─ 数据库表: user_role / role_menu / menu
    │
    └─► 配置层
         ├─ 角色 status 是否启用
         ├─ 菜单 status 是否启用
         └─ 角色 type 是否超管
```

---

## 四、容易误判的边界场景

### 4.1 场景一：Token 未过期但 RefreshToken 已过期

**误判点：** 以为 access_token 有效就一定能刷新

**实际情况：**

```
时间线:
00:00 - 用户登录
       access_token 有效期: 2小时 (到 02:00)
       refresh_token 有效期: 7天 (到 第7天 00:00)

第8天 01:00 - 前端发起请求
            access_token 已过期 (2小时 < 8天)
            refresh_token 也已过期 (7天 < 8天)

第8天 01:30 - 前端尝试刷新 Token
            → 失败: 刷新令牌已过期
```

**关键代码 (OAuth2TokenServiceImpl.java:73-101)：**

```java
public OAuth2AccessTokenDO refreshAccessToken(
    String refreshToken, String clientId) {
    
    // 1. 查询 refresh_token
    OAuth2RefreshTokenDO refreshTokenDO = 
        oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    
    if (refreshTokenDO == null) {
        throw exception0(BAD_REQUEST, "无效的刷新令牌");
    }
    
    // 2. 检查 refresh_token 是否过期
    if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
        oauth2RefreshTokenMapper.deleteById(refreshTokenDO.getId());
        throw exception0(UNAUTHORIZED, "刷新令牌已过期");
    }
    
    // 3. 创建新的 access_token
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

**表象：**
- 前端自动刷新 Token 失败
- 突然跳转到登录页
- 后端日志："刷新令牌已过期"

**容易误判：** 以为是 access_token 问题，实际是 refresh_token 过期

---

### 4.2 场景二：超管角色的权限绕过

**误判点：** 以为超管也需要配置菜单权限

**实际情况：**

**超管判断逻辑 (PermissionServiceImpl.java:83)：**

```java
// 情况二：如果是超管，也说明有权限
return roleService.hasAnySuperAdmin(
    convertSet(roles, RoleDO::getId));
```

**RoleService 实现：**

```java
public boolean hasAnySuperAdmin(Collection<Long> ids) {
    if (CollUtil.isEmpty(ids)) {
        return false;
    }
    // 角色 type = 2 (SYSTEM) 视为超管
    List<RoleDO> roles = getRoleList(ids);
    return roles.stream()
        .anyMatch(role -> RoleTypeEnum.SYSTEM.getType()
            .equals(role.getType()));
}
```

**数据库字段：**

```sql
-- system_role 表
-- type 字段: 1=自定义, 2=系统内置(超管)

SELECT id, name, code, type FROM system_role;
+-----+-----------+-------------+------+
| id  | name      | code        | type |
+-----+-----------+-------------+------+
| 1   | 超级管理员 | super_admin | 2    |  ← 超管
| 2   | 普通角色   | common      | 1    |  ← 非超管
+-----+-----------+-------------+------+
```

**表象：**
- 超管用户即使不配置任何菜单，也能访问所有接口
- `@PreAuthorize("@ss.hasPermission('xxx')")` 对超管永远返回 true
- 前端菜单可能为空，但后端接口都能调通

**容易误判：** 
- 以为超管也需要配置菜单权限
- 测试时给超管配置了错误的菜单，以为是权限系统 bug
- 看到空菜单以为是登录问题

---

### 4.3 场景三：用户类型不匹配导致的 401

**误判点：** Token 有效但用户类型不匹配

**用户类型枚举：**

```java
public enum UserTypeEnum {
    ADMIN(1, "管理员"),      // 管理后台用户
    MEMBER(2, "会员用户");    // C端用户
}
```

**路由前缀与用户类型对应：**

| 路由前缀 | 用户类型 | 用途 |
|----------|----------|------|
| `/admin-api/**` | UserTypeEnum.ADMIN (1) | 管理后台 |
| `/app-api/**` | UserTypeEnum.MEMBER (2) | C端应用 |

**校验逻辑 (TokenAuthenticationFilter.java:80-83)：**

```java
// 用户类型不匹配，无权限
// 注意：只有 /admin-api/* 和 /app-api/* 有 userType
if (userType != null
    && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
    throw new AccessDeniedException("错误的用户类型");
}
```

**用户类型注入 (WebFrameworkUtils)：**

```java
// 从请求路径提取 userType
// /admin-api/... → userType = 1 (ADMIN)
// /app-api/... → userType = 2 (MEMBER)
public static Integer getLoginUserType(HttpServletRequest request) {
    // 根据请求 URL 前缀判断
}
```

**表象：**
- Token 本身有效 (Redis/MySQL 都存在且未过期)
- 调用 `/admin-api/xxx` 却返回 401
- 后端日志："错误的用户类型" 或 AccessDeniedException

**常见触发场景：**
1. C端用户 (userType=2) 的 Token 拿去调用管理后台接口
2. 管理后台用户 (userType=1) 的 Token 拿去调用 C端接口
3. 测试时复制了错误的 Token

**容易误判：**
- 以为 Token 过期
- 以为 Redis 缓存失效
- 反复检查 MySQL token 表却发现数据正常

**排查方法：**
```sql
-- 检查 Token 的用户类型
SELECT user_id, user_type, expires_time 
FROM oauth2_access_token 
WHERE access_token = 'xxx';

-- user_type = 1 只能调用 /admin-api/*
-- user_type = 2 只能调用 /app-api/*
```

---

### 4.4 场景四：菜单 permission 为空的按钮权限

**误判点：** 以为所有按钮都有 permission 字段

**菜单类型：**

| type | 类型 | 是否有 permission |
|------|------|-------------------|
| 1 | 目录 | 通常为空 |
| 2 | 菜单 | 通常有 (页面级) |
| 3 | 按钮 | 应该有 (操作级) |

**校验逻辑 (PermissionServiceImpl.java:94-98)：**

```java
private boolean hasAnyPermission(List<RoleDO> roles, String permission) {
    // 根据权限标识查菜单
    List<Long> menuIds = menuService
        .getMenuIdListByPermissionFromCache(permission);
    
    // 采用严格模式，如果权限找不到对应的 Menu 的话，
    // 也认为没有权限
    if (CollUtil.isEmpty(menuIds)) {
        return false;  // ← 关键：permission 不存在时返回 false
    }
    // ...
}
```

**表象：**
- 按钮在前端显示 (菜单配置了 type=3)
- 但点击按钮调用后端接口时返回 403
- 接口的 `@PreAuthorize` 权限标识和菜单配置的不一致

**容易误判：**
- 以为是角色菜单配置错误
- 反复检查 system_role_menu 关联表
- 以为是缓存问题

**实际原因：**
- 菜单的 `permission` 字段为空或拼写错误
- 或接口的 `@PreAuthorize` 与菜单配置不一致

**排查 SQL：**

```sql
-- 检查菜单的 permission 字段
SELECT id, name, permission, type 
FROM system_menu 
WHERE name LIKE '%用户%' AND type = 3;

-- 检查是否有接口使用了该权限
-- 代码中搜索: @PreAuthorize("@ss.hasPermission('xxx')")
```

---

## 五、关键类与配置汇总

### 5.1 后端核心类

| 类名 | 路径 | 职责 |
|------|------|------|
| `AuthController` | `yudao-module-system/.../controller/admin/auth/` | 登录、登出、获取权限信息接口 |
| `AdminAuthServiceImpl` | `yudao-module-system/.../service/auth/` | 认证业务逻辑 |
| `OAuth2TokenServiceImpl` | `yudao-module-system/.../service/oauth2/` | Token 创建/刷新/校验 |
| `TokenAuthenticationFilter` | `yudao-framework/.../security/core/filter/` | Token 认证过滤器 |
| `PermissionServiceImpl` | `yudao-module-system/.../service/permission/` | 权限校验 |
| `SecurityFrameworkServiceImpl` | `yudao-framework/.../security/core/service/` | SpEL 权限服务 (@ss) |
| `YudaoWebSecurityConfigurerAdapter` | `yudao-framework/.../security/config/` | Security 配置 |
| `SecurityFrameworkUtils` | `yudao-framework/.../security/core/util/` | 安全工具类 |

### 5.2 数据库表

| 表名 | 用途 |
|------|------|
| `system_user` | 用户信息 |
| `system_role` | 角色信息 (code, type, status) |
| `system_menu` | 菜单信息 (permission, path, component) |
| `system_user_role` | 用户-角色关联 |
| `system_role_menu` | 角色-菜单关联 |
| `oauth2_access_token` | 访问令牌 |
| `oauth2_refresh_token` | 刷新令牌 |
| `oauth2_client` | OAuth2 客户端 (有效期配置) |

### 5.3 Redis Key

| Key 格式 | 说明 |
|----------|------|
| `oauth2_access_token:{token}` | Token 缓存 |
| `user_role_ids:{userId}` | 用户角色ID集合 |
| `menu_role_ids:{menuId}` | 菜单对应的角色ID |
| `permission_menu_ids:{permission}` | 权限标识对应的菜单ID |
| `role:{roleId}` | 角色详情 |

### 5.4 配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `yudao.security.token-header` | `Authorization` | Token 请求头名 |
| `yudao.security.token-parameter` | `token` | Token 参数名 |
| `yudao.security.mock-enable` | `false` | 是否启用 mock token |
| `yudao.security.permit-all-urls` | `[]` | 放行 URL 列表 |
| `yudao.captcha.enable` | `true` | 是否启用验证码 |

---

## 六、排查问题的快速 checklist

当遇到认证/权限问题时，按以下顺序排查：

### 6.1 401 未授权

```
1. 前端是否携带了 Token?
   └─ 检查请求头: Authorization: Bearer {token}

2. Token 是否存在?
   ├─ Redis: GET oauth2_access_token:{token}
   └─ MySQL: SELECT * FROM oauth2_access_token WHERE access_token = ?

3. Token 是否已过期?
   └─ SELECT expires_time, NOW() FROM oauth2_access_token WHERE ...

4. 用户类型是否匹配?
   ├─ 接口路径: /admin-api/* → user_type=1
   └─ 接口路径: /app-api/* → user_type=2

5. 用户是否被禁用?
   └─ SELECT status FROM system_user WHERE id = ?
```

### 6.2 403 权限不足

```
1. 接口需要什么权限?
   └─ 代码: @PreAuthorize("@ss.hasPermission('xxx')")

2. 用户有哪些角色?
   ├─ Redis: GET user_role_ids:{userId}
   └─ MySQL: SELECT * FROM system_user_role WHERE user_id = ?

3. 角色是否启用?
   └─ SELECT status FROM system_role WHERE id IN (...)

4. 权限标识对应的菜单?
   ├─ Redis: GET permission_menu_ids:{permission}
   └─ MySQL: SELECT * FROM system_menu WHERE permission = ?

5. 角色是否有该菜单?
   └─ SELECT * FROM system_role_menu 
      WHERE role_id IN (?) AND menu_id IN (?)

6. 是否是超管?
   └─ SELECT type FROM system_role WHERE id IN (?)
      (type=2 为超管，跳过权限校验)
```

### 6.3 前端菜单不显示

```
1. 后端 get-permission-info 接口返回的数据是否包含该菜单?

2. 菜单是否可见?
   └─ system_menu.visible = 1

3. 菜单是否启用?
   └─ system_menu.status = 0

4. 前端路由转换逻辑是否正确?
   └─ transformMenusToRoutes()

5. 路由是否已动态添加?
   └─ router.getRoutes() 检查
```

---

## 七、总结

RuoYi-Vue-Pro 的认证授权体系是一个多层级、多缓存的复杂系统：

1. **认证层**：Spring Security 过滤链 + TokenAuthenticationFilter，支持 Redis+MySQL 双重存储
2. **权限层**：RBAC 模型 (用户-角色-菜单-权限标识)，多层 Redis 缓存提升性能
3. **前端层**：Pinia permission store + Vue Router 动态路由，基于菜单树自动生成

理解这个系统的关键在于：
- 区分 **认证** (你是谁) 和 **授权** (你能做什么)
- 理解 **双重存储** (Redis 高性能 + MySQL 持久化) 的容错机制
- 理解 **多层缓存** (token、角色、菜单、权限) 的一致性问题
- 注意 **边界场景** (用户类型、超管绕过、refresh_token 过期等)
