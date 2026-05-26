# RuoYi-Vue-Pro 事务边界分析报告

## 一、典型业务流程选择

本文选择**交易订单创建与支付流程**作为分析对象，该流程涉及：
- Controller 层：接收创建订单请求
- Service 层：订单创建、支付单创建、积分扣减等业务逻辑
- Mapper 层：数据库操作
- 跨模块调用：支付模块、会员模块

---

## 二、完整调用链路分析

### 2.1 整体调用层级

```
AppTradeOrderController (Controller 层 - 无事务)
    ↓
TradeOrderUpdateServiceImpl.createOrder (Service 层 - @Transactional)
    ├── buildTradeOrder (私有方法 - 无事务)
    ├── buildTradeOrderItems (私有方法 - 无事务)
    ├── tradeOrderHandlers.forEach (beforeOrderCreate)
    │   └── TradeMemberPointOrderHandler.beforeOrderCreate (无 @Transactional)
    ├── tradeOrderMapper.insert (Mapper 层 - 无事务)
    ├── tradeOrderItemMapper.insertBatch (Mapper 层 - 无事务)
    ├── afterCreateTradeOrder (私有方法 - 无事务)
    │   ├── tradeOrderHandlers.forEach (afterOrderCreate)
    │   │   └── TradeMemberPointOrderHandler.afterOrderCreate
    │   │       └── MemberPointApiImpl.reducePoint
    │   │           └── MemberPointRecordServiceImpl.createPointRecord (@Transactional)
    │   ├── cartService.deleteCart (无 @Transactional)
    │   └── createPayOrder (私有方法)
    │       └── PayOrderApiImpl.createOrder
    │           └── PayOrderServiceImpl.createOrder (无 @Transactional)
    └── return order
```

### 2.2 支付回调链路

```
PayOrderServiceImpl.notifyOrder (channelId, notify) - 无 @Transactional
    ↓
TenantUtils.execute → getSelf().notifyOrder (channel, notify) - @Transactional
    ├── notifyOrderSuccess (私有方法)
    │   ├── updateOrderSuccess (私有方法)
    │   ├── updateOrderSuccess (私有方法)
    │   └── notifyService.createPayNotifyTask (@Transactional)
    │       ├── notifyTaskMapper.insert
    │       └── TransactionSynchronization.afterCommit
    │           └── getSelf().executeNotifyAsync (@Async)
    │               └── executeNotify → HTTP 回调 AppTradeOrderController.update-paid
    │                   └── TradeOrderUpdateServiceImpl.updateOrderPaid (@Transactional)
    │                       ├── tradeOrderMapper.updateByIdAndStatus
    │                       └── tradeOrderHandlers.forEach (afterPayOrder)
    │                           └── TradeMemberPointOrderHandler.afterPayOrder
    │                               └── MemberPointRecordServiceImpl.createPointRecord (@Transactional)
```

---

## 三、@Transactional 真实边界分析

### 3.1 有 @Transactional 注解的方法

| 类名 | 方法 | 注解位置 | 说明 |
|------|------|---------|------|
| TradeOrderUpdateServiceImpl | createOrder | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| TradeOrderUpdateServiceImpl | updateOrderPaid | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| TradeOrderUpdateServiceImpl | receiveOrderBySystem | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| TradeOrderUpdateServiceImpl | cancelOrderBySystem | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| TradeOrderUpdateServiceImpl | pickUpOrder | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| PayOrderServiceImpl | notifyOrder (重载) | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| PayNotifyServiceImpl | createPayNotifyTask | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| PayNotifyServiceImpl | executeNotify0 | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| MemberPointRecordServiceImpl | createPointRecord | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |
| MemberUserServiceImpl | createUser | 方法级别 | `@Transactional(rollbackFor = Exception.class)` |

### 3.2 关键发现：类级别无 @Transactional

**重要发现**：
- `TradeOrderUpdateServiceImpl` 类本身**没有** `@Transactional` 注解
- `PayOrderServiceImpl` 类本身**没有** `@Transactional` 注解
- 只有具体的**方法**上才有 `@Transactional` 注解

这意味着：
- 没有 `@Transactional` 注解的方法调用时，**不会创建事务**
- 同一个 Service 内的自调用（this.method()）**不会触发事务**

---

## 四、代理机制与自调用分析

### 4.1 实际进入代理的方法

以下方法会通过 Spring AOP 代理执行，事务生效：

```java
// AppTradeOrderController 调用 - 通过代理
tradeOrderUpdateService.createOrder(userId, createReqVO);

// 外部调用 - 通过代理
tradeOrderUpdateService.updateOrderPaid(id, payOrderId);

// 外部调用 - 通过代理
payOrderService.notifyOrder(channel, notify);
```

### 4.2 不会进入代理的自调用

**关键代码位置 1**：TradeOrderUpdateServiceImpl 内部私有方法调用

```java
// TradeOrderUpdateServiceImpl.java:186-204
@Override
@Transactional(rollbackFor = Exception.class)
@TradeOrderLog(operateType = TradeOrderOperateTypeEnum.MEMBER_CREATE)
public TradeOrderDO createOrder(Long userId, AppTradeOrderCreateReqVO createReqVO) {
    // 1.1 价格计算 - 私有方法，this 调用，不通过代理
    TradePriceCalculateRespBO calculateRespBO = calculatePrice(userId, createReqVO);
    // 1.2 构建订单 - 私有方法，this 调用，不通过代理
    TradeOrderDO order = buildTradeOrder(userId, createReqVO, calculateRespBO);
    // ...
    // 4. 订单创建后的逻辑 - 私有方法，this 调用，不通过代理
    afterCreateTradeOrder(order, orderItems, createReqVO);
    return order;
}

// 私有方法 - 没有 @Transactional，通过 this 调用
private void afterCreateTradeOrder(...) {
    // ...
    createPayOrder(order, orderItems); // 私有方法调用
}

private void createPayOrder(...) {
    // 这里的代码不会触发新事务
    // 如果抛出异常，会回滚外层事务
}
```

**分析**：
- `createOrder()` 通过代理进入，创建了事务
- 内部调用的 `calculatePrice()`、`buildTradeOrder()`、`afterCreateTradeOrder()`、`createPayOrder()` 都是通过 `this` 调用的
- 这些私有方法**不会**通过代理执行，即使它们有 `@Transactional`（这里实际上没有）
- 但它们**会参与**外层 `createOrder()` 方法的事务（因为在同一个物理连接中）

### 4.3 正确的自调用处理方式

**关键代码位置 2**：使用 `getSelf()` 获取代理对象

```java
// TradeOrderUpdateServiceImpl.java:321-334
@Override
public void syncOrderPayStatusQuietly(Long id, Long payOrderId) {
    // ...
    try {
        // 关键点：通过 getSelf() 获取代理对象，再调用事务方法
        getSelf().updateOrderPaid(id, payOrderId);
    } catch (Throwable e) {
        log.warn("[syncOrderPayStatusQuietly][...]", id, payOrderId, e);
    }
}

// TradeOrderUpdateServiceImpl.java:475-495
@Override
public int receiveOrderBySystem() {
    // ...
    for (TradeOrderDO order : orders) {
        try {
            // 通过 getSelf() 获取代理对象
            getSelf().receiveOrderBySystem(order);
            count++;
        } catch (Throwable e) {
            log.error("[receiveOrderBySystem][...]", order.getId(), e);
        }
    }
    return count;
}

// TradeOrderUpdateServiceImpl.java:1048-1050
private TradeOrderUpdateServiceImpl getSelf() {
    return SpringUtil.getBean(getClass());
}
```

**分析**：
- `getSelf()` 通过 `SpringUtil.getBean(getClass())` 从 Spring 容器获取代理对象
- 使用 `getSelf().updateOrderPaid()` 而不是 `this.updateOrderPaid()`
- 这样 `@Transactional` 注解才能生效，创建新的事务

---

## 五、事务内事件/MQ 发布风险分析

### 5.1 风险场景：事务内直接发送消息

**错误做法示例**（如果这样写会有问题）：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(...) {
    // 1. 插入订单数据
    tradeOrderMapper.insert(order);
    
    // 2. 直接发送 MQ 消息
    mqProducer.sendOrderCreatedEvent(order.getId());
    
    // 3. 后续操作抛出异常
    if (someCondition) {
        throw new RuntimeException("业务异常");
    }
}
```

**问题**：
1. MQ 消息在事务提交前就发送了
2. 如果后续操作抛出异常，数据库事务会回滚
3. 但 MQ 消息已经发出，消费者可能已经处理
4. 导致**脏消息**：消息对应的数据实际不存在

### 5.2 项目中的正确做法 1：TransactionSynchronization

**关键代码位置 3**：PayNotifyServiceImpl 中的事务后通知

```java
// PayNotifyServiceImpl.java:96-129
@Override
@Transactional(rollbackFor = Exception.class)
public void createPayNotifyTask(Integer type, Long dataId) {
    // 1. 构建通知任务
    PayNotifyTaskDO task = new PayNotifyTaskDO().setType(type).setDataId(dataId);
    // ...
    
    // 2. 插入通知任务到数据库
    notifyTaskMapper.insert(task);
    
    // 3. 关键点：注册事务同步回调，在事务提交后再执行
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            // 异步执行通知，避免阻塞当前事务
            getSelf().executeNotifyAsync(task);
        }
    });
}
```

**分析**：
1. 先将通知任务持久化到数据库（在事务内）
2. 注册 `TransactionSynchronization` 回调
3. `afterCommit()` 方法在**事务提交后**才执行
4. 确保：只有数据库事务提交成功，才会发送 HTTP 回调
5. 即使回调失败，任务已在数据库中，后续定时任务可以重试

### 5.3 项目中的正确做法 2：MQ 消息发送

**关键代码位置 4**：MemberUserServiceImpl 中的事务后 MQ 发送

```java
// MemberUserServiceImpl.java:95-122
private MemberUserDO createUser(String mobile, String nickname, String avtar,
                                String registerIp, Integer terminal) {
    // 1. 插入用户数据
    MemberUserDO user = new MemberUserDO();
    // ...
    memberUserMapper.insert(user);
    
    // 2. 注册事务同步回调
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            // 事务提交后才发送 MQ 消息
            memberUserProducer.sendUserCreateMessage(user.getId());
        }
    });
    return user;
}
```

**分析**：
1. 先插入用户数据到数据库
2. 注册事务同步回调
3. `afterCommit()` 在事务提交后才执行
4. 确保：用户数据真正持久化后，才发送 MQ 消息
5. 避免了"消息已发送但数据不存在"的脏消息问题

---

## 六、异常捕获与事务回滚分析

### 6.1 正常回滚场景

```java
// TradeOrderUpdateServiceImpl.java:184-204
@Override
@Transactional(rollbackFor = Exception.class)
public TradeOrderDO createOrder(...) {
    // ...
    tradeOrderMapper.insert(order);  // 操作 1
    // ...
    throw new ServiceException("业务异常");  // 抛出异常
    // ...
}
```

**结果**：
- 异常抛出且未被捕获
- 事务回滚
- `tradeOrderMapper.insert(order)` 操作被撤销

### 6.2 异常被捕获的场景

**关键代码位置 5**：TradeOrderUpdateServiceImpl 中的异常捕获

```java
// TradeOrderUpdateServiceImpl.java:321-334
@Override
public void syncOrderPayStatusQuietly(Long id, Long payOrderId) {
    // ... 校验逻辑 ...
    try {
        // 通过代理调用，创建新事务
        getSelf().updateOrderPaid(id, payOrderId);
    } catch (Throwable e) {
        // 异常被捕获
        log.warn("[syncOrderPayStatusQuietly][id({}) payOrderId({}) 同步支付状态失败]", id, payOrderId, e);
        // 没有重新抛出异常
    }
}
```

**分析**：
1. `syncOrderPayStatusQuietly()` 方法**没有** `@Transactional`
2. `getSelf().updateOrderPaid()` 通过代理调用，创建了**新的独立事务**
3. 如果 `updateOrderPaid()` 内部抛出异常，那个独立事务会回滚
4. 异常被 `try-catch` 捕获后，`syncOrderPayStatusQuietly()` 继续执行
5. 外层（如果有）的事务**不会**受到影响（因为是独立事务）

### 6.3 内部私有方法异常

```java
// TradeOrderUpdateServiceImpl.java:248-269
private void afterCreateTradeOrder(...) {
    // 1. 执行订单创建后置处理器
    tradeOrderHandlers.forEach(handler -> handler.afterOrderCreate(order, orderItems));
    // ...
    // 3. 生成预支付
    createPayOrder(order, orderItems);
}

private void createPayOrder(...) {
    // 创建支付单
    Long payOrderId = payOrderApi.createOrder(payOrderCreateReqDTO);
    // 更新交易单
    tradeOrderMapper.updateById(...);
}
```

**分析**：
- `afterCreateTradeOrder()` 和 `createPayOrder()` 是私有方法
- 它们在 `createOrder()` 的事务范围内执行
- 如果这些方法抛出异常，**会被外层事务的 AOP 拦截**
- 事务会回滚

**注意**：如果这些方法内部捕获了异常且不重新抛出，事务**不会**回滚。

---

## 七、5 个模型容易给出错误结论的代码位置

### 位置 1：PayOrderServiceImpl.submitOrder 方法的注释

```java
// PayOrderServiceImpl.java:141-142
@Override // 注意，这里不能添加事务注解，避免调用支付渠道失败时，将 PayOrderExtensionDO 回滚了
public PayOrderSubmitRespVO submitOrder(PayOrderSubmitReqVO reqVO, String userIp) {
    // ...
    // 2. 插入 PayOrderExtensionDO
    orderExtensionMapper.insert(orderExtension);
    
    // 3. 调用三方接口（可能失败）
    PayOrderRespDTO unifiedOrderResp = client.unifiedOrder(unifiedOrderReqDTO);
    // ...
}
```

**模型容易错误认为**：
- "这个方法需要事务保证数据一致性"

**实际情况**：
- 注释明确说明"不能添加事务注解"
- 原因：调用三方支付接口可能失败
- 如果有事务，三方调用失败会回滚 `PayOrderExtensionDO`
- 但实际上需要保留 `PayOrderExtensionDO` 记录，用于后续查询和对账
- 这是**业务设计决策**，不是技术疏忽

---

### 位置 2：TradeOrderUpdateServiceImpl.getSelf() 的使用

```java
// TradeOrderUpdateServiceImpl.java:321-334
@Override
public void syncOrderPayStatusQuietly(Long id, Long payOrderId) {
    // ...
    try {
        getSelf().updateOrderPaid(id, payOrderId);
    } catch (Throwable e) {
        log.warn("[...]", id, payOrderId, e);
    }
}

// TradeOrderUpdateServiceImpl.java:475-495
@Override
public int receiveOrderBySystem() {
    // ...
    for (TradeOrderDO order : orders) {
        try {
            getSelf().receiveOrderBySystem(order);
            count++;
        } catch (Throwable e) {
            log.error("[...]", order.getId(), e);
        }
    }
    return count;
}
```

**模型容易错误认为**：
- "这是冗余代码，直接 `this.updateOrderPaid()` 就行"
- 或者 "这些方法应该在类级别加 @Transactional"

**实际情况**：
- `getSelf()` 是**故意设计**的，用于获取 Spring 代理对象
- `this.updateOrderPaid()` 不会触发事务（自调用问题）
- `getSelf().updateOrderPaid()` 才会通过代理，`@Transactional` 才生效
- 循环中逐个调用，每个订单在独立事务中执行，一个失败不影响其他

---

### 位置 3：MemberPointRecordServiceImpl.createPointRecord 中的 early return

```java
// MemberPointRecordServiceImpl.java:67-94
@Override
@Transactional(rollbackFor = Exception.class)
public void createPointRecord(Long userId, Integer point, MemberPointBizTypeEnum bizType, String bizId) {
    if (point == 0) {
        return;  // 直接返回
    }
    // 1. 校验用户积分余额
    MemberUserDO user = memberUserService.getUser(userId);
    Integer userPoint = ObjectUtil.defaultIfNull(user.getPoint(), 0);
    int totalPoint = userPoint + point;
    if (totalPoint < 0) {
        log.error("[createPointRecord][...]");
        return;  // 直接返回，不抛出异常
    }
    // 2. 更新用户积分
    boolean success = memberUserService.updateUserPoint(userId, point);
    if (!success) {
        throw exception(USER_POINT_NOT_ENOUGH);
    }
    // 3. 增加积分记录
    memberPointRecordMapper.insert(record);
}
```

**模型容易错误认为**：
- "第 79 行应该抛出异常而不是直接 return"
- "early return 会导致事务部分提交"

**实际情况**：
1. `point == 0` 时直接 return：这是正常情况（无需变更），不是异常
2. `totalPoint < 0` 时 return 且记录错误日志：
   - 这是**业务决策**：积分不足时静默失败，不影响主流程
   - 此时事务还没有进行任何写操作（只读取了 user）
   - 所以不存在"部分提交"问题
3. 只有 `updateUserPoint` 返回 false 时才抛出异常，触发回滚

---

### 位置 4：PayOrderServiceImpl.notifyOrder 的重载和租户切换

```java
// PayOrderServiceImpl.java:262-290
@Override
public void notifyOrder(Long channelId, PayOrderRespDTO notify) {
    // 校验支付渠道
    PayChannelDO channel = channelService.validPayChannel(channelId);
    // 关键点：租户切换后，通过 getSelf() 调用事务方法
    TenantUtils.execute(channel.getTenantId(), () -> getSelf().notifyOrder(channel, notify));
}

@Transactional(rollbackFor = Exception.class)
// 注意，如果是方法内调用该方法，需要通过 getSelf().notifyPayOrder(...) 调用，否则事务不生效
public void notifyOrder(PayChannelDO channel, PayOrderRespDTO notify) {
    // 实际处理逻辑
}
```

**模型容易错误认为**：
- "两个 `notifyOrder` 方法造成了混淆，是代码坏味道"
- "租户切换应该在事务方法内部进行"

**实际情况**：
1. **方法重载是故意设计**：
   - `notifyOrder(Long channelId, PayOrderRespDTO notify)`：无事务，负责租户切换
   - `notifyOrder(PayChannelDO channel, PayOrderRespDTO notify)`：有事务，执行业务逻辑
2. **租户切换必须在事务外**：
   - Spring 的事务同步管理器是基于租户的
   - 如果先开启事务再切换租户，会导致混乱
   - 正确顺序：切换租户 → 获取代理 → 调用事务方法
3. **注释还提醒了自调用问题**，说明开发者已经考虑到了

---

### 位置 5：TradeOrderUpdateServiceImpl.cancelOrderByAfterSale 方法

```java
// TradeOrderUpdateServiceImpl.java:651-664
@TradeOrderLog(operateType = TradeOrderOperateTypeEnum.ADMIN_CANCEL_AFTER_SALE)
public void cancelOrderByAfterSale(TradeOrderDO order, Integer refundPrice) {
    // 1. 更新订单
    if (refundPrice < order.getPayPrice()) {
        return;  // 部分退款，直接返回
    }
    tradeOrderMapper.updateById(new TradeOrderDO().setId(order.getId())
            .setStatus(TradeOrderStatusEnum.CANCELED.getStatus())
            .setCancelType(TradeOrderCancelTypeEnum.AFTER_SALE_CLOSE.getType())
            .setCancelTime(LocalDateTime.now()));
    
    // 2. 执行 TradeOrderHandler 的后置处理
    List<TradeOrderItemDO> orderItems = tradeOrderItemMapper.selectListByOrderId(order.getId());
    tradeOrderHandlers.forEach(handler -> handler.afterCancelOrder(order, orderItems));
}

// TradeOrderUpdateServiceImpl.java:833
// 在事务方法 updateOrderItemWhenAfterSaleSuccess 中调用
getSelf().cancelOrderByAfterSale(order, orderRefundPrice);
```

**模型容易错误认为**：
- "这个方法没有 @Transactional，数据库操作不安全"
- "getSelf() 调用是多余的，因为方法没有 @Transactional"

**实际情况**：
1. **方法本身没有 @Transactional 是故意的**：
   - 它被设计为在已有事务中执行
   - 调用者 `updateOrderItemWhenAfterSaleSuccess` 有 `@Transactional`
2. **但仍使用 `getSelf()` 调用**：
   - 不是为了事务（外层已有事务）
   - 而是为了 `@TradeOrderLog` 注解！
   - AOP 日志切面也需要通过代理才能生效
3. **`return` 逻辑是业务决策**：
   - 部分退款不取消订单
   - 只有全部退款才取消订单

---

## 八、总结

### 8.1 关键设计模式

| 设计模式 | 代码位置 | 作用 |
|---------|---------|------|
| `getSelf()` 代理获取 | TradeOrderUpdateServiceImpl:1048 | 解决自调用事务失效问题 |
| TransactionSynchronization | PayNotifyServiceImpl:120 | 事务提交后再执行异步操作 |
| 事务方法与非事务方法分离 | PayOrderServiceImpl.notifyOrder | 租户切换与事务边界清晰 |
| 循环中独立事务 | receiveOrderBySystem | 一个失败不影响其他 |

### 8.2 模型常见误区

1. **认为所有 Service 方法都应该有事务**：实际上很多方法故意没有事务（如 `submitOrder`）
2. **忽视自调用问题**：`this.method()` 不会触发代理，需要 `getSelf().method()`
3. **认为 early return 一定是 bug**：很多是业务决策（如积分不足时静默失败）
4. **不理解事务传播和租户的关系**：租户切换必须在事务边界外
5. **忽视其他 AOP 切面**：`getSelf()` 不仅用于事务，还用于日志等其他切面

### 8.3 最佳实践

1. **事务边界明确**：只在需要事务的方法上加 `@Transactional`
2. **自调用使用代理**：`getSelf()` 模式或注入自身
3. **异步操作延后**：使用 `TransactionSynchronization.afterCommit()`
4. **异常处理谨慎**：捕获异常时要考虑是否影响事务
5. **文档化设计决策**：如 `submitOrder` 方法的注释
