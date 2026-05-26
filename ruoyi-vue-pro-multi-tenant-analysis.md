# Ruoyi-Vue-Pro 多租户实现深度分析

## 1. 概述

本文档深入分析 ruoyi-vue-pro 的多租户实现机制，覆盖从 HTTP 请求到 SQL 执行的完整链路，并构造一个真实的故障推演场景。

## 2. 核心组件架构

### 2.1 整体架构

ruoyi-vue-pro 采用基于 `行级租户` 的多租户方案（共享数据库、共享表），通过在每张业务表中增加 `tenant_id` 字段来区分不同租户的数据。

主要组件包括：
- **TenantContextHolder**: 租户上下文存储
- **TenantContextWebFilter**: 从请求头解析租户 ID
- **TenantSecurityWebFilter**: 安全校验和登录用户租户信息提取
- **TenantDatabaseInterceptor**: MyBatis Plus 租户拦截器
- **TenantIgnore 注解**: 忽略租户过滤机制
- **TenantJob 注解**: 定时任务租户处理

## 3. 租户上下文管理：TenantContextHolder

### 3.1 实现机制

位于 `yudao-spring-boot-starter-biz-tenant` 模块，核心实现：

```java
public class TenantContextHolder {
    private static final ThreadLocal<Long> TENANT_ID = new TransmittableThreadLocal<>();
    private static final ThreadLocal<Boolean> IGNORE = new TransmittableThreadLocal<>();
    
    public static Long getTenantId() { return TENANT_ID.get(); }
    public static void setTenantId(Long tenantId) { TENANT_ID.set(tenantId); }
    public static void setIgnore(Boolean ignore) { IGNORE.set(ignore); }
    public static boolean isIgnore() { return Boolean.TRUE.equals(IGNORE.get()); }
    public static void clear() { TENANT_ID.remove(); IGNORE.remove(); }
}
```

### 3.2 关键设计点

1. **使用 TransmittableThreadLocal**: 而非普通的 ThreadLocal，这是解决异步任务中 tenantId 传递的关键（阿里开源的线程上下文传递工具）
2. **双变量设计**: 
   - `TENANT_ID`: 存储当前租户编号
   - `IGNORE`: 标记是否忽略租户过滤
3. **手动清理机制**: 必须通过 `clear()` 方法手动清理，避免线程池复用导致的上下文污染

## 4. HTTP 请求中的租户 ID 传递链路

### 4.1 过滤器链顺序

```
HTTP 请求
    ↓
TenantContextWebFilter (优先级高)
    ↓
Security Filter (Spring Security)
    ↓
TenantSecurityWebFilter
    ↓
Controller
    ↓
Service
    ↓
MyBatis Plus 拦截器
    ↓
SQL 执行
```

### 4.2 第一步：TenantContextWebFilter

位置：`cn.iocoder.yudao.framework.tenant.core.web.TenantContextWebFilter`

```java
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
    Long tenantId = WebFrameworkUtils.getTenantId(request); // 从请求头获取
    if (tenantId != null) {
        TenantContextHolder.setTenantId(tenantId);
    }
    try {
        chain.doFilter(request, response);
    } finally {
        TenantContextHolder.clear(); // 重要：请求结束必须清理
    }
}
```

**作用**: 从 HTTP 请求头 `tenant-id` 中解析租户 ID 并设置到上下文

### 4.3 第二步：TenantSecurityWebFilter

位置：`cn.iocoder.yudao.framework.tenant.core.security.TenantSecurityWebFilter`

```java
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
    Long tenantId = TenantContextHolder.getTenantId();
    LoginUser user = SecurityFrameworkUtils.getLoginUser();
    
    // 1. 如果是已登录用户
    if (user != null) {
        if (tenantId == null) {
            // 从登录用户信息中获取租户 ID
            tenantId = user.getTenantId();
            TenantContextHolder.setTenantId(tenantId);
        } else if (!Objects.equals(user.getTenantId(), tenantId)) {
            // 校验：防止越权访问其他租户
            // 返回 403 错误
            return;
        }
    }
    
    // 2. 非忽略 URL 必须有租户 ID
    if (!isIgnoreUrl(request)) {
        if (tenantId == null) {
            // 返回 400 错误
            return;
        }
        // 3. 校验租户合法性（是否禁用、是否过期）
        tenantFrameworkService.validTenant(tenantId);
    } else {
        // 忽略 URL 且无租户 ID 时，设置忽略标记
        if (tenantId == null) {
            TenantContextHolder.setIgnore(true);
        }
    }
    
    chain.doFilter(request, response);
}
```

**关键逻辑**:
- 优先从请求头获取租户 ID
- 如果请求头没有，从登录用户信息中获取
- 越权校验：用户的租户 ID 必须和请求的租户 ID 一致
- 忽略 URL 处理：如短信回调、支付回调等无需租户 ID 的接口

### 4.4 租户信息来源总结

| 场景 | 租户 ID 来源 |
|-----|-------------|
| 前端发起请求 | 请求头 `tenant-id` |
| 已登录用户 | 从登录用户信息 `user.getTenantId()` 兜底 |
| 忽略 URL | 无，设置 `ignore=true` |

## 5. MyBatis Plus 租户拦截器

### 5.1 自动配置

位置：`YudaoTenantAutoConfiguration`

```java
@Bean
public TenantLineInnerInterceptor tenantLineInnerInterceptor(TenantProperties properties,
                                                             MybatisPlusInterceptor interceptor) {
    TenantLineInnerInterceptor inner = new TenantLineInnerInterceptor(
        new TenantDatabaseInterceptor(properties));
    // 添加到拦截器链首位
    MyBatisUtils.addInterceptor(interceptor, inner, 0);
    return inner;
}
```

### 5.2 TenantDatabaseInterceptor 实现

位置：`cn.iocoder.yudao.framework.tenant.core.db.TenantDatabaseInterceptor`

```java
public class TenantDatabaseInterceptor implements TenantLineHandler {
    
    @Override
    public Expression getTenantId() {
        return new LongValue(TenantContextHolder.getRequiredTenantId());
    }
    
    @Override
    public boolean ignoreTable(String tableName) {
        // 情况一：全局忽略多租户
        if (TenantContextHolder.isIgnore()) {
            return true;
        }
        // 情况二：配置中忽略的表
        // 情况三：DO 实体上有 @TenantIgnore 注解
        // 情况四：DO 实体没有继承 TenantBaseDO
    }
}
```

### 5.3 SQL 改写示例

原始 SQL:
```sql
SELECT id, order_no FROM trade_order WHERE status = 10;
```

改写后 SQL:
```sql
SELECT id, order_no FROM trade_order WHERE tenant_id = 100 AND status = 10;
```

## 6. @TenantIgnore 注解详解

### 6.1 注解定义

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface TenantIgnore {
    String enable() default "true"; // 支持 Spring EL
}
```

### 6.2 生效范围

| 标注位置 | 生效机制 | 说明 |
|---------|---------|------|
| **Controller 类/方法** | 自动加入忽略 URL 列表 | 启动时通过 `getTenantIgnoreUrls()` 扫描，在 `TenantSecurityWebFilter` 中跳过 |
| **Service 方法** | AOP 环绕拦截 | 执行期间设置 `ignore=true`，执行后恢复 |
| **DO 实体类** | MyBatis 表级忽略 | 在 `TenantDatabaseInterceptor.ignoreTable()` 中检查 |
| **不标注** | 默认启用租户过滤 | 所有查询自动附加 tenant_id 条件 |

### 6.3 Service 层 AOP 实现

```java
@Aspect
public class TenantIgnoreAspect {
    @Around("@annotation(tenantIgnore)")
    public Object around(ProceedingJoinPoint joinPoint, TenantIgnore tenantIgnore) {
        Boolean oldIgnore = TenantContextHolder.isIgnore();
        try {
            // 支持 Spring EL 表达式
            Object enable = SpringExpressionUtils.parseExpression(tenantIgnore.enable());
            if (Boolean.TRUE.equals(enable)) {
                TenantContextHolder.setIgnore(true);
            }
            return joinPoint.proceed();
        } finally {
            TenantContextHolder.setIgnore(oldIgnore); // 恢复原状态
        }
    }
}
```

### 6.4 启动时扫描 Controller

```java
private Set<String> getTenantIgnoreUrls() {
    Set<String> ignoreUrls = new HashSet<>();
    RequestMappingHandlerMapping handlerMapping = 
        applicationContext.getBean("requestMappingHandlerMapping");
    Map<RequestMappingInfo, HandlerMethod> handlerMethodMap = handlerMapping.getHandlerMethods();
    
    for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : handlerMethodMap.entrySet()) {
        HandlerMethod method = entry.getValue();
        if (method.hasMethodAnnotation(TenantIgnore.class) 
            || method.getBeanType().isAnnotationPresent(TenantIgnore.class)) {
            // 添加该 URL 到忽略列表
            ignoreUrls.addAll(entry.getKey().getPatternsCondition().getPatterns());
        }
    }
    return ignoreUrls;
}
```

## 7. 系统租户与默认租户处理

### 7.1 系统租户概念

系统租户是一个特殊租户，具有以下特征：

```java
public class TenantServiceImpl implements TenantService {
    private static boolean isSystemTenant(TenantDO tenant) {
        return Objects.equals(tenant.getPackageId(), TenantDO.PACKAGE_ID_SYSTEM);
    }
    
    @Override
    public void handleTenantMenu(TenantMenuHandler handler) {
        TenantDO tenant = getTenant(TenantContextHolder.getRequiredTenantId());
        Set<Long> menuIds;
        if (isSystemTenant(tenant)) {
            // 系统租户：菜单是全量的
            menuIds = CollectionUtils.convertSet(menuService.getMenuList(), MenuDO::getId);
        } else {
            // 普通租户：受套餐限制
            menuIds = tenantPackageService.getTenantPackage(tenant.getPackageId()).getMenuIds();
        }
        handler.handle(menuIds);
    }
}
```

### 7.2 系统租户限制

```java
private TenantDO validateUpdateTenant(Long id) {
    TenantDO tenant = tenantMapper.selectById(id);
    if (isSystemTenant(tenant)) {
        throw exception(TENANT_CAN_NOT_UPDATE_SYSTEM);
    }
    return tenant;
}
```

**系统租户不能**:
- 修改
- 删除
- 但可以正常使用所有功能

### 7.3 默认租户

在 Swagger 配置中可以看到默认租户为 `1`：

```java
.schema(new IntegerSchema()._default(1L).name(HEADER_TENANT_ID).description("租户编号"))
```

## 8. 异步任务与定时任务中的租户 ID 丢失问题

### 8.1 根本原因：ThreadLocal 与线程池

```
请求线程 (Thread-101)                    线程池线程 (pool-1-thread-1)
┌──────────────────────┐                ┌──────────────────────────┐
│ tenantId = 100       │                │ tenantId = null (默认)   │
│ (TransmittableThread │   @Async       │ (新线程，无上下文)        │
│  Local)              │ ─────────────▶ │                          │
└──────────────────────┘                └──────────────────────────┘
```

**问题分析**:
- 普通 `ThreadLocal`: 只能在当前线程访问，子线程无法继承
- `InheritableThreadLocal`: 子线程可以继承父线程的值，但线程池复用线程时不会更新
- `TransmittableThreadLocal`: 需要配合 TTL 的线程池包装才能正确传递

### 8.2 ruoyi-vue-pro 的解决方案

#### 8.2.1 定时任务：@TenantJob 注解

```java
@Aspect
public class TenantJobAspect {
    @Around("@annotation(tenantJob)")
    public String around(ProceedingJoinPoint joinPoint, TenantJob tenantJob) {
        // 1. 获取所有租户列表
        List<Long> tenantIds = tenantFrameworkService.getTenantIds();
        
        // 2. 逐个租户执行
        Map<Long, String> results = new ConcurrentHashMap<>();
        tenantIds.parallelStream().forEach(tenantId -> {
            TenantUtils.execute(tenantId, () -> {
                try {
                    Object result = joinPoint.proceed();
                    results.put(tenantId, StrUtil.toStringOrEmpty(result));
                } catch (Throwable e) {
                    log.error("[execute][租户({}) 执行 Job 发生异常", tenantId, e);
                }
            });
        });
        return JsonUtils.toJsonString(results);
    }
}
```

使用示例：

```java
@Component
public class TradeOrderAutoCancelJob implements JobHandler {
    @Override
    @TenantJob
    public String execute(String param) {
        int count = tradeOrderUpdateService.cancelOrderBySystem();
        return String.format("过期订单 %s 个", count);
    }
}
```

**执行流程**:
1. 定时任务触发（无租户上下文）
2. `@TenantJob` 切面拦截
3. 查询所有租户 ID 列表
4. 为每个租户创建上下文并执行任务
5. `TenantUtils.execute()` 临时设置租户 ID

#### 8.2.2 TenantUtils.execute 工具方法

```java
public static void execute(Long tenantId, Runnable runnable) {
    Long oldTenantId = TenantContextHolder.getTenantId();
    Boolean oldIgnore = TenantContextHolder.isIgnore();
    try {
        TenantContextHolder.setTenantId(tenantId);
        TenantContextHolder.setIgnore(false);
        runnable.run();
    } finally {
        TenantContextHolder.setTenantId(oldTenantId);
        TenantContextHolder.setIgnore(oldIgnore);
    }
}
```

#### 8.2.3 消息队列：Redis MQ 拦截器

```java
public class TenantRedisMessageInterceptor implements RedisMessageInterceptor {
    @Override
    public void sendMessageBefore(AbstractRedisMessage message) {
        Long tenantId = TenantContextHolder.getTenantId();
        if (tenantId != null) {
            message.addHeader(HEADER_TENANT_ID, tenantId.toString());
        }
    }
    
    @Override
    public void consumeMessageBefore(AbstractRedisMessage message) {
        String tenantIdStr = message.getHeader(HEADER_TENANT_ID);
        if (StrUtil.isNotEmpty(tenantIdStr)) {
            TenantContextHolder.setTenantId(Long.valueOf(tenantIdStr));
        }
    }
    
    @Override
    public void consumeMessageAfter(AbstractRedisMessage message) {
        TenantContextHolder.clear();
    }
}
```

**机制**: 发送消息时将 tenantId 放入消息头，消费时从消息头恢复

---

## 9. 故障推演：后台任务批量处理订单却串租户

### 9.1 场景描述

**背景**: 某 SaaS 电商平台使用 ruoyi-vue-pro 框架，有多个租户（商家）入驻。

**功能**: 每天凌晨 2:00 执行定时任务，批量结算所有"已完成"订单的佣金。

**现象**: 
- 租户 A（商家A）的订单被错误地结算到了租户 B（商家B）
- 日志显示：查询到的订单包含了其他租户的数据
- 数据库中 `tenant_id` 字段值混乱

### 9.2 错误代码复现

#### 9.2.1 错误实现

```java
// 错误的后台任务实现
@Component
public class OrderSettlementJob implements JobHandler {
    
    @Resource
    private TradeOrderMapper orderMapper;
    
    @Resource
    private SettlementService settlementService;
    
    @Override
    // ❌ 错误：没有使用 @TenantJob 注解
    public String execute(String param) {
        // 1. 查询所有"已完成"订单
        // ❌ 没有设置 tenantId，TenantContextHolder.getTenantId() 返回 null
        // ❌ 如果 MyBatis 拦截器没有严格校验，可能导致查询全表
        List<TradeOrderDO> orders = orderMapper.selectList(
            new LambdaQueryWrapper<TradeOrderDO>()
                .eq(TradeOrderDO::getStatus, OrderStatusEnum.COMPLETED.getStatus())
        );
        
        // 2. 批量结算
        for (TradeOrderDO order : orders) {
            // ❌ 结算时使用当前线程上下文的 tenantId（可能是 null 或随机值）
            settlementService.settleOrder(order);
        }
        
        return "结算完成";
    }
}
```

#### 9.2.2 更隐蔽的错误：异步调用

```java
@Service
public class SettlementService {
    
    @Async
    public void settleOrdersAsync(List<Long> orderIds) {
        // ❌ @Async 导致在新线程中执行
        // ❌ 如果线程池没有用 TTL 包装，TransmittableThreadLocal 也无法传递
        for (Long orderId : orderIds) {
            // 这里查询的是哪个租户的数据？不确定！
            TradeOrderDO order = orderMapper.selectById(orderId);
            // 结算...
        }
    }
}

// Controller 调用
@PostMapping("/settle/batch")
public void batchSettle(@RequestBody List<Long> orderIds) {
    // 当前请求线程：tenantId = 100 (租户A)
    settlementService.settleOrdersAsync(orderIds); 
    // 异步线程：tenantId = null (如果线程池没包装)
}
```

### 9.3 真实根因分析

#### 9.3.1 根本原因

```
定时任务线程                                MyBatis 执行
┌──────────────────────────┐              ┌──────────────────────┐
│ tenantId = null          │   SQL 查询   │ getRequiredTenantId()│
│ (没有调用 @TenantJob)    │ ──────────▶ │ 抛出 NullPointerException │
│ 或                        │              │  或                   │
│ tenantId = 随机值         │              │ 如果不严格，查询全表   │
│ (线程池上下文污染)         │              └──────────────────────┘
└──────────────────────────┘
```

**四种可能的根因**:

| 根因编号 | 原因 | 后果 |
|---------|------|------|
| 1 | 定时任务未使用 `@TenantJob` | `tenantId = null`，要么报错要么查询全表 |
| 2 | 线程池复用导致上下文污染 | 上一个任务的 `tenantId` 残留，影响当前任务 |
| 3 | `@Async` 调用未包装线程池 | 子线程无法继承父线程的 `TransmittableThreadLocal` |
| 4 | 手动清理机制缺失 | `finally` 块中忘记调用 `TenantContextHolder.clear()` |

#### 9.3.2 本案例的具体根因

**根因 2（上下文污染）+ 根因 3（异步未包装）的组合**:

1. 租户 A 的管理员先触发了一次手动结算（`tenantId = 100`）
2. 使用了线程池中的线程 `pool-1-thread-1`
3. 执行完成后**没有清理** `TenantContextHolder`
4. 凌晨定时任务触发，分配到同一线程 `pool-1-thread-1`
5. 此时 `TenantContextHolder.getTenantId()` 返回 `100`（污染值）
6. 定时任务查询的是租户 A 的订单，但结算时可能逻辑混乱

或者更严重的情况：

1. 定时任务没有 `@TenantJob`，也没有手动设置 `tenantId`
2. `TenantContextHolder.getTenantId()` 返回 `null`
3. 如果某个地方有 `if (tenantId == null) tenantId = 1` 的默认值逻辑
4. 所有订单都被当作租户 1 的数据处理

### 9.4 错误修复方式（无效或有缺陷）

#### 9.4.1 错误修复 1：硬编码租户 ID

```java
public String execute(String param) {
    // ❌ 错误：硬编码，只处理租户 1
    TenantContextHolder.setTenantId(1L);
    try {
        List<TradeOrderDO> orders = orderMapper.selectList(...);
        // ...
    } finally {
        TenantContextHolder.clear();
    }
}
```

**缺陷**: 只能处理一个租户，多租户架构失效

#### 9.4.2 错误修复 2：查询所有订单再过滤

```java
@TenantIgnore // 忽略租户过滤
public String execute(String param) {
    List<TradeOrderDO> orders = orderMapper.selectList(...); // 查询所有租户订单
    
    // 手动按租户分组
    Map<Long, List<TradeOrderDO>> ordersByTenant = orders.stream()
        .collect(Collectors.groupingBy(TradeOrderDO::getTenantId));
    
    // 逐个租户处理
    for (Map.Entry<Long, List<TradeOrderDO>> entry : ordersByTenant.entrySet()) {
        Long tenantId = entry.getKey();
        List<TradeOrderDO> tenantOrders = entry.getValue();
        // 处理...
    }
}
```

**缺陷**: 
- `@TenantIgnore` 导致 SQL 全表扫描，性能差
- 大数据量时内存溢出风险
- 需要手动管理租户上下文，容易出错

#### 9.4.3 错误修复 3：只加 finally 块

```java
public String execute(String param) {
    try {
        // 业务逻辑，tenantId 还是 null
    } finally {
        TenantContextHolder.clear(); // 只是清理，但之前已经错了
    }
}
```

**缺陷**: 根本问题没解决，`tenantId` 依然是 null

### 9.5 正确修复方式

#### 9.5.1 方案一：使用 @TenantJob 注解（推荐）

```java
@Component
public class OrderSettlementJob implements JobHandler {
    
    @Resource
    private TradeOrderMapper orderMapper;
    
    @Resource
    private SettlementService settlementService;
    
    @Override
    @TenantJob  // ✅ 正确：自动遍历所有租户
    public String execute(String param) {
        // 此时 TenantContextHolder 已经被 TenantJobAspect 设置好
        // 每个租户独立执行一次
        
        // 1. 查询当前租户的已完成订单
        // ✅ MyBatis 拦截器自动附加 tenant_id 条件
        List<TradeOrderDO> orders = orderMapper.selectList(
            new LambdaQueryWrapper<TradeOrderDO>()
                .eq(TradeOrderDO::getStatus, OrderStatusEnum.COMPLETED.getStatus())
        );
        
        // 2. 批量结算当前租户的订单
        for (TradeOrderDO order : orders) {
            // ✅ 安全：上下文已设置为当前租户
            settlementService.settleOrder(order);
        }
        
        return String.format("结算订单 %d 个", orders.size());
    }
}
```

**执行流程**:

```
@TenantJob 切面
    ↓
获取所有租户 [100, 101, 102, ...]
    ↓
并行/串行遍历
    ↓
tenantId = 100 ──▶ execute() ──▶ 查询 tenant_id=100 的订单
    ↓
tenantId = 101 ──▶ execute() ──▶ 查询 tenant_id=101 的订单
    ↓
tenantId = 102 ──▶ execute() ──▶ 查询 tenant_id=102 的订单
    ↓
...
```

#### 9.5.2 方案二：手动使用 TenantUtils.execute

```java
@Component
public class OrderSettlementJob implements JobHandler {
    
    @Resource
    private TenantFrameworkService tenantFrameworkService;
    
    @Override
    public String execute(String param) {
        // 1. 获取所有租户
        List<Long> tenantIds = tenantFrameworkService.getTenantIds();
        
        // 2. 逐个租户执行
        for (Long tenantId : tenantIds) {
            // ✅ 正确：手动设置上下文，自动恢复
            TenantUtils.execute(tenantId, () -> {
                // 这里执行的所有数据库操作都自动归属当前租户
                List<TradeOrderDO> orders = orderMapper.selectList(...);
                for (TradeOrderDO order : orders) {
                    settlementService.settleOrder(order);
                }
            });
        }
        
        return "完成";
    }
}
```

#### 9.5.3 方案三：@Async 线程池包装（使用 TTL）

```java
@Configuration
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        
        // ✅ 关键：使用 TTL 包装线程池
        return TtlExecutors.getTtlExecutor(executor);
    }
}

// 然后 @Async 就能正确传递 TransmittableThreadLocal 了
@Async
public void settleOrdersAsync(List<Long> orderIds) {
    // ✅ tenantId 正确传递
    for (Long orderId : orderIds) {
        TradeOrderDO order = orderMapper.selectById(orderId);
        // ...
    }
}
```

#### 9.5.4 方案四：自定义任务的完整实现模板

```java
public void executeSettlementForTenant(Long tenantId) {
    // 保存旧状态
    Long oldTenantId = TenantContextHolder.getTenantId();
    Boolean oldIgnore = TenantContextHolder.isIgnore();
    
    try {
        // 设置新状态
        TenantContextHolder.setTenantId(tenantId);
        TenantContextHolder.setIgnore(false);
        
        // 执行业务
        doSettlement();
        
    } finally {
        // ✅ 必须：恢复旧状态
        if (oldTenantId != null) {
            TenantContextHolder.setTenantId(oldTenantId);
        } else {
            TenantContextHolder.getTenantId(); // 防止 null
        }
        TenantContextHolder.setIgnore(oldIgnore);
    }
}
```

### 9.6 故障预防清单

| 检查项 | 说明 |
|-------|------|
| ✅ 定时任务 | 必须使用 `@TenantJob` 或手动遍历租户 |
| ✅ `@Async` 方法 | 线程池必须用 `TtlExecutors.getTtlExecutor()` 包装 |
| ✅ 手动设置上下文 | 必须使用 `try-finally` 模式，确保恢复 |
| ✅ 忽略过滤 | `@TenantIgnore` 只用于特殊场景，不能滥用 |
| ✅ 线程池 | 必须考虑线程复用导致的上下文污染问题 |
| ✅ 测试 | 必须在多租户环境下测试，验证数据隔离 |

### 9.7 故障排查方法

当出现串租户问题时，按以下步骤排查：

1. **检查日志**: 打印 `TenantContextHolder.getTenantId()` 的值
2. **检查 SQL**: 开启 MyBatis SQL 日志，查看实际执行的 SQL 是否有 `tenant_id` 条件
3. **检查调用链**: 使用链路追踪，查看 `tenantId` 在哪个环节丢失
4. **检查线程**: 打印线程名和 ID，查看是否是线程池复用导致
5. **检查注解**: 确认是否正确使用了 `@TenantJob`

## 10. 完整请求流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HTTP 请求处理流程                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 1. TenantContextWebFilter                                           │
│    从请求头 "tenant-id" 获取值                                       │
│    TenantContextHolder.setTenantId(tenantId)                        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Spring Security Filter                                          │
│    解析 Token，获取 LoginUser                                        │
│    LoginUser 中包含 tenantId                                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. TenantSecurityWebFilter                                          │
│    - 校验越权：用户.tenantId == 请求.tenantId                         │
│    - 兜底：请求头为空时用登录用户的 tenantId                           │
│    - 校验租户合法性：是否禁用、是否过期                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Controller → Service → Mapper                                    │
│    业务逻辑执行                                                       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. MyBatis Plus TenantLineInnerInterceptor                          │
│    调用 TenantDatabaseInterceptor                                    │
│    - getTenantId(): 返回 TenantContextHolder.getRequiredTenantId()   │
│    - ignoreTable(): 检查是否需要忽略                                 │
│    SQL 自动改写: WHERE tenant_id = ?                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. TenantContextWebFilter finally 块                                │
│    TenantContextHolder.clear()                                      │
│    防止线程池复用导致的上下文污染                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 11. 关键代码位置汇总

| 组件 | 文件路径 |
|-----|---------|
| TenantContextHolder | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/context/TenantContextHolder.java` |
| TenantContextWebFilter | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/web/TenantContextWebFilter.java` |
| TenantSecurityWebFilter | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/security/TenantSecurityWebFilter.java` |
| TenantDatabaseInterceptor | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/db/TenantDatabaseInterceptor.java` |
| TenantIgnore 注解 | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/aop/TenantIgnore.java` |
| TenantIgnoreAspect | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/aop/TenantIgnoreAspect.java` |
| TenantJob 注解 | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/job/TenantJob.java` |
| TenantJobAspect | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/job/TenantJobAspect.java` |
| TenantUtils | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/util/TenantUtils.java` |
| 自动配置 | `yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/config/YudaoTenantAutoConfiguration.java` |

---

**文档生成时间**: 2026-05-10  
**分析基础**: ruoyi-vue-pro 当前代码库版本
