# ruoyi-vue-pro 支付模块状态机分析

## 1. 支付订单核心模型与单号区别

### 1.1 核心数据模型

支付模块采用**支付订单 + 支付拓展单**的双层架构：

- **PayOrderDO** (`pay_order` 表)：支付主订单，面向业务系统
- **PayOrderExtensionDO**：支付拓展单，面向支付渠道

### 1.2 单号区别

| 单号类型 | 字段 | 所属表 | 说明 | 示例 |
|---------|------|--------|------|------|
| **商户订单号** | `merchantOrderId` | `pay_order` | 业务系统自己的订单号，用于关联业务订单，每个 App 下唯一 | `1001` (Demo订单ID) |
| **支付订单号** | `no` (Extension) | `pay_order_extension` | 支付模块生成的单号，作为**外部交易号(OutTradeNo)**传给支付渠道 | `P20260510xxxxxx` |
| **渠道订单号** | `channelOrderNo` | `pay_order` | 支付渠道(微信/支付宝)返回的内部订单号 | 微信 `420000xxxxxx` |
| **支付单ID** | `id` | `pay_order` | 数据库主键自增 | `1024` |

**关键区别**：
- `merchantOrderId`：业务层视角，如订单系统、会员系统的业务单号
- `payOrderExtension.no`：支付渠道视角，传递给微信/支付宝的 `out_trade_no`

## 2. 状态定义与单向流转约束

### 2.1 支付订单状态 (`PayOrderStatusEnum`)

```java
WAITING(0, "未支付")
SUCCESS(10, "支付成功")
REFUND(20, "已退款")
CLOSED(30, "支付关闭")
```

### 2.2 状态机流转图

```
┌─────────────┐
│  WAITING    │◄──── 初始状态
└──────┬──────┘
       │
   ┌───┴───┐
   │       │
   ▼       ▼
┌──────┐  ┌───────┐
│SUCCESS│  │CLOSED │  ◄──── 支付失败/超时关闭
└───┬──┘  └───────┘
    │
    ▼
┌────────┐
│ REFUND │  ◄──── 全部退款后进入
└────────┘
```

### 2.3 单向流转约束实现

项目通过**CAS更新**保证状态只能单向流转，核心代码在 `PayOrderServiceImpl.java`：

```java
// PayOrderServiceImpl.java:361-371
int updateCounts = orderMapper.updateByIdAndStatus(
    order.getId(), 
    PayOrderStatusEnum.WAITING.getStatus(),  // WHERE条件：必须是WAITING
    PayOrderDO.builder()
        .status(PayOrderStatusEnum.SUCCESS.getStatus()) // SET：更新为SUCCESS
        .build()
);
if (updateCounts == 0) { // 0行受影响，说明状态不是WAITING
    throw exception(PAY_ORDER_STATUS_IS_NOT_WAITING);
}
```

**流转规则**：
- WAITING → SUCCESS ✅ 允许
- WAITING → CLOSED ✅ 允许  
- SUCCESS → REFUND ✅ 允许（全部退款）
- SUCCESS → WAITING ❌ 禁止（CAS更新失败）
- REFUND → WAITING ❌ 禁止
- CLOSED → SUCCESS ❌ 禁止

## 3. 完整支付流程

### 3.1 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        业务订单 (DemoOrder)                                  │
│  1. createDemoOrder() → 2. payOrderApi.createOrder() → 3. 保存payOrderId    │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        支付订单 (PayOrder)                                   │
│  4. createOrder() → 5. submitOrder() → 6. 调用支付渠道统一下单                 │
│     (WAITING)       (生成Extension)      (返回支付参数)                       │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼  用户完成支付
┌─────────────────────────────────────────────────────────────────────────────┐
│                   支付渠道回调 (微信/支付宝)                                   │
│  7. notifyOrder() → 8. 签名校验 → 9. 解析通知 → 10. notifyOrderSuccess()      │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                 支付订单更新 (PayOrder + Extension)                           │
│  11. updateOrderSuccess(Extension) → 12. updateOrderSuccess(Order)           │
│      (WAITING→SUCCESS)            (WAITING→SUCCESS)                          │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   业务回调通知 (PayNotifyService)                             │
│  13. createPayNotifyTask() → 14. executeNotify() → 15. HTTP通知业务系统       │
│      (事务提交后异步)           (Redis分布式锁)     (重试机制)                   │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     业务订单回写 (DemoOrder)                                  │
│  16. updateDemoOrderPaid() → 17. CAS校验状态 → 18. 更新库存/积分              │
│      (幂等校验)              (payed=false)       (业务处理)                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键代码位置

| 步骤 | 类名 | 方法 | 行号 |
|-----|------|------|------|
| 4. 创建支付单 | `PayOrderServiceImpl` | `createOrder()` | 116-139 |
| 5. 提交支付 | `PayOrderServiceImpl` | `submitOrder()` | 142-187 |
| 7. 回调入口 | `PayNotifyController` | `notifyOrder()` | 63-83 |
| 8. 签名校验 | `AbstractWxPayClient` | `doParseOrderNotifyV3()` | 184-194 |
| 10. 通知成功 | `PayOrderServiceImpl` | `notifyOrderSuccess()` | 292-304 |
| 11. 更新Extension | `PayOrderServiceImpl` | `updateOrderSuccess()`(Extension) | 312-334 |
| 12. 更新Order | `PayOrderServiceImpl` | `updateOrderSuccess()`(Order) | 344-374 |
| 13. 创建通知任务 | `PayNotifyServiceImpl` | `createPayNotifyTask()` | 96-129 |
| 15. 通知业务 | `PayNotifyServiceImpl` | `executeNotifyInvoke()` | 232-259 |
| 16. 业务回写 | `PayDemoOrderServiceImpl` | `updateDemoOrderPaid()` | 115-145 |

## 4. 支付渠道回调签名校验

### 4.1 微信支付V3签名校验流程

```java
// AbstractWxPayClient.java:184-194
private PayOrderRespDTO doParseOrderNotifyV3(String body, Map<String, String> headers) 
        throws WxPayException {
    // 1. 从请求头提取签名信息
    SignatureHeader signatureHeader = getRequestHeader(headers);
    // 包含：Wechatpay-Signature, Wechatpay-Nonce, Wechatpay-Serial, Wechatpay-Timestamp
    
    // 2. 调用WxJava SDK进行签名校验和结果解密
    WxPayNotifyV3Result response = client.parseOrderNotifyV3Result(body, signatureHeader);
    // SDK内部会：
    // - 使用微信平台公钥验证签名
    // - 使用商户APIv3密钥解密回调数据
    
    WxPayNotifyV3Result.DecryptNotifyResult result = response.getResult();
    // ...
}
```

### 4.2 签名校验失败的处理

如果签名校验失败（如伪造回调、参数篡改），`WxPayException` 会被抛出，直接中断流程，不会执行任何业务逻辑。

## 5. 重复通知幂等性保障

项目采用**多层幂等校验**机制：

### 5.1 第一层：Extension状态校验

```java
// PayOrderServiceImpl.java:312-334
private PayOrderExtensionDO updateOrderSuccess(PayOrderRespDTO notify) {
    // 1. 查询Extension
    PayOrderExtensionDO orderExtension = orderExtensionMapper.selectByNo(notify.getOutTradeNo());
    
    // 2. 已支付则直接返回（幂等）
    if (PayOrderStatusEnum.isSuccess(orderExtension.getStatus())) {
        log.info("[updateOrderExtensionSuccess][orderExtension({}) 已经是已支付，无需更新]", 
                 orderExtension.getId());
        return orderExtension;
    }
    
    // 3. CAS更新（并发安全）
    int updateCounts = orderExtensionMapper.updateByIdAndStatus(
        orderExtension.getId(), 
        orderExtension.getStatus(), // 必须是WAITING
        PayOrderExtensionDO.builder()
            .status(PayOrderStatusEnum.SUCCESS.getStatus())
            .build()
    );
    if (updateCounts == 0) {
        throw exception(PAY_ORDER_EXTENSION_STATUS_IS_NOT_WAITING);
    }
    return orderExtension;
}
```

### 5.2 第二层：Order状态校验

```java
// PayOrderServiceImpl.java:344-374
private Boolean updateOrderSuccess(PayChannelDO channel, 
                                   PayOrderExtensionDO orderExtension,
                                   PayOrderRespDTO notify) {
    // 1. 已支付且ExtensionId匹配，直接返回true（幂等）
    if (PayOrderStatusEnum.isSuccess(order.getStatus()) 
            && Objects.equals(order.getExtensionId(), orderExtension.getId())) {
        log.info("[updateOrderExtensionSuccess][order({}) 已经是已支付，无需更新]", order.getId());
        return true; // 返回true，不会重复创建通知任务
    }
    // ...
    return false;
}
```

### 5.3 第三层：通知任务Redis锁

```java
// PayNotifyServiceImpl.java:187-203
public void executeNotify(PayNotifyTaskDO task) {
    // 分布式锁，避免并发问题
    notifyLockCoreRedisDAO.lock(task.getId(), NOTIFY_TIMEOUT_MILLIS, () -> {
        // 二次校验：通知次数匹配
        PayNotifyTaskDO dbTask = notifyTaskMapper.selectById(task.getId());
        if (ObjectUtil.notEqual(task.getNotifyTimes(), dbTask.getNotifyTimes())) {
            log.warn("[executeNotifySync][task({}) 任务被忽略，原因是通知次数不匹配]", task);
            return;
        }
        // 执行通知...
    });
}
```

### 5.4 第四层：业务订单幂等校验

```java
// PayDemoOrderServiceImpl.java:115-145
public void updateDemoOrderPaid(Long id, Long payOrderId) {
    // 1. 校验订单是否存在
    PayDemoOrderDO order = payDemoOrderMapper.selectById(id);
    
    // 2. 已支付且支付单号相同，直接返回（幂等）
    if (order.getPayStatus()) {
        if (ObjectUtil.equals(order.getPayOrderId(), payOrderId)) {
            log.warn("[updateDemoOrderPaid][order({}) 已支付，且支付单号相同({})，直接返回]", 
                     order, payOrderId);
            return; // 幂等返回
        }
        // 支付单号不同，异常
        throw exception(DEMO_ORDER_UPDATE_PAID_FAIL_PAY_ORDER_ID_ERROR);
    }
    
    // 3. CAS更新
    int updateCount = payDemoOrderMapper.updateByIdAndPayed(id, false,
            new PayDemoOrderDO().setPayStatus(true)...);
    if (updateCount == 0) {
        throw exception(DEMO_ORDER_UPDATE_PAID_STATUS_NOT_UNPAID);
    }
}
```

## 6. 事务边界分析

### 6.1 支付订单更新事务

```java
// PayOrderServiceImpl.java:276-290
@Transactional(rollbackFor = Exception.class)
public void notifyOrder(PayChannelDO channel, PayOrderRespDTO notify) {
    if (PayOrderStatusEnum.isSuccess(notify.getStatus())) {
        notifyOrderSuccess(channel, notify); // 包含：更新Extension、更新Order、创建通知任务
        return;
    }
    // ...
}
```

**事务范围**：
- Extension状态更新（CAS）
- Order状态更新（CAS）
- 创建PayNotifyTask记录

**全部成功或全部回滚**。

### 6.2 通知任务创建与执行分离

```java
// PayNotifyServiceImpl.java:96-129
@Transactional(rollbackFor = Exception.class)
public void createPayNotifyTask(Integer type, Long dataId) {
    // 1. 插入任务记录
    notifyTaskMapper.insert(task);
    
    // 2. 事务提交后异步执行
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            getSelf().executeNotifyAsync(task); // 异步，不阻塞当前事务
        }
    });
}
```

**关键设计**：
- 任务创建在事务内（保证任务一定被记录）
- 任务执行在事务提交后（异步）
- 即使业务通知失败，任务记录已存在，可通过定时任务重试

### 6.3 事务边界图

```
┌─────────────────────────────────────────────────────────────┐
│               事务T1: notifyOrder()                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────────────────┐   │
│  │更新Extension│→│ 更新Order  │→│ 插入PayNotifyTask记录 │   │
│  │(CAS)      │  │ (CAS)     │  │                       │   │
│  └───────────┘  └───────────┘  └───────────┬───────────┘   │
└─────────────────────────────────────────────┼───────────────┘
                                              │ 事务提交后
                                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    异步执行 (无事务)                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  executeNotify()                                      │   │
│  │  ┌─────────────┐  ┌────────────┐  ┌──────────────┐  │   │
│  │  │Redis分布式锁 │→│HTTP通知业务 │→│更新任务状态   │  │   │
│  │  │             │  │系统        │  │(SUCCESS/FAIL)│  │   │
│  │  └─────────────┘  └─────┬──────┘  └──────────────┘  │   │
│  │                         │失败                        │   │
│  │                         ▼                            │   │
│  │                  ┌────────────┐                      │   │
│  │                  │写入日志    │                      │   │
│  │                  │计算下次重试│                      │   │
│  │                  └────────────┘                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 7. 回调成功但业务更新失败时的系统表现

### 7.1 场景描述

支付渠道回调成功 → 支付订单状态更新为SUCCESS → 但业务系统通知失败（如网络超时、业务异常）

### 7.2 系统行为

```
时间线：
T1: 微信回调 → notifyOrder() → 支付订单更新为SUCCESS ✅
T2: createPayNotifyTask() → 任务记录插入 ✅
T3: 事务提交 ✅
T4: 异步executeNotify() → HTTP调用业务系统 ❌ (超时)
T5: processNotifyResult() → 任务状态=REQUEST_FAILURE，计算下次重试时间 ✅
T6: 定时任务PayNotifyJob执行 → 重试通知
... (最多重试NOTIFY_FREQUENCY次)
Tn: 超过最大次数 → 任务状态=FAILURE (需人工介入)
```

### 7.3 关键机制

**1. 重试频率** (`PayNotifyTaskDO.NOTIFY_FREQUENCY`)
```
第1次：立即
第2次：+5秒
第3次：+10秒
第4次：+30秒
第5次：+1分钟
第6次：+5分钟
第7次：+10分钟
第8次：+30分钟
第9次：+1小时
第10次：+2小时
第11次：+6小时
第12次：+12小时 (最后一次)
```

**2. 最终一致性保障**
- 支付订单状态=SUCCESS（用户已付款）
- 业务订单状态=待处理（未收到通知）
- 通知任务状态=FAILURE（可查看日志）
- **人工可通过管理后台查看并手动触发**

## 8. 错误实现示例：微信支付重复回调导致库存重复增加

### 8.1 错误代码场景

假设一个电商系统，支付成功后需要扣减库存。下面是一个**有问题**的实现：

```java
// ========== 错误实现 ==========
@Service
@Slf4j
public class BadOrderServiceImpl {

    @Resource
    private StockMapper stockMapper;
    @Resource
    private OrderMapper orderMapper;

    /**
     * ❌ 错误实现：没有幂等校验，直接扣减库存
     */
    public void updateOrderPaid(Long orderId, Long payOrderId) {
        // 1. 查询订单
        OrderDO order = orderMapper.selectById(orderId);
        if (order == null) {
            throw new RuntimeException("订单不存在");
        }
        
        // ❌ 问题1：没有检查订单是否已支付
        // if (order.getPayStatus()) { return; }
        
        // 2. 扣减库存
        // ❌ 问题2：没有使用CAS，直接更新
        stockMapper.decreaseStock(order.getProductId(), order.getQuantity());
        // UPDATE stock SET quantity = quantity - #{quantity} WHERE product_id = #{productId}
        
        // 3. 更新订单状态
        // ❌ 问题3：没有使用CAS
        order.setPayStatus(true);
        order.setPayTime(LocalDateTime.now());
        orderMapper.updateById(order);
    }
}
```

### 8.2 问题分析

**场景**：微信支付同一笔订单连续发送2次回调

```
时序：
回调1: T0 进入updateOrderPaid() → T1 查询订单(payed=false) → T2 扣减库存(-1) → T3 更新订单(payed=true)
回调2: T0 进入updateOrderPaid() → T1 查询订单(payed=false) → T2 扣减库存(-1) → T3 更新订单(payed=true)
                                 (因为回调1还没提交事务，读到的是旧数据)

结果：
- 库存扣减了2次
- 订单状态只更新一次（后写覆盖）
- 用户支付1次，获得2件商品
```

### 8.3 正确实现（参考项目中的做法）

```java
// ========== 正确实现（参考PayDemoOrderServiceImpl） ==========
@Service
@Slf4j
public class GoodOrderServiceImpl {

    @Resource
    private StockMapper stockMapper;
    @Resource
    private OrderMapper orderMapper;

    /**
     * ✅ 正确实现：多层幂等校验 + CAS更新
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateOrderPaid(Long orderId, Long payOrderId) {
        // 1. 查询订单
        OrderDO order = orderMapper.selectById(orderId);
        if (order == null) {
            throw new RuntimeException("订单不存在");
        }
        
        // ✅ 幂等校验1：已支付且支付单号相同，直接返回
        if (order.getPayStatus()) {
            if (Objects.equals(order.getPayOrderId(), payOrderId)) {
                log.warn("[updateOrderPaid][order({}) 已支付，直接返回]", orderId);
                return;
            }
            throw new RuntimeException("支付单号不匹配");
        }
        
        // ✅ 幂等校验2：CAS更新订单状态
        // UPDATE order SET pay_status = true, pay_time = NOW(), pay_order_id = #{payOrderId} 
        // WHERE id = #{orderId} AND pay_status = false
        int updateCount = orderMapper.updateByIdAndPayStatus(
            orderId, 
            false, // 条件：必须是未支付
            new OrderDO().setPayStatus(true).setPayTime(LocalDateTime.now()).setPayOrderId(payOrderId)
        );
        if (updateCount == 0) {
            // 并发情况下，另一个线程已更新成功
            log.warn("[updateOrderPaid][order({}) 并发更新，已有线程处理成功]", orderId);
            return;
        }
        
        // ✅ 只有CAS成功后才执行业务逻辑（扣减库存、增加积分等）
        // 此时其他重复回调已被CAS拦截
        stockMapper.decreaseStockWithVersion(
            order.getProductId(), 
            order.getQuantity()
        );
        // UPDATE stock SET quantity = quantity - #{quantity}, version = version + 1 
        // WHERE product_id = #{productId} AND version = #{version}
    }
}
```

## 9. 项目中的防重机制总结

| 层级 | 机制 | 代码位置 | 作用 |
|-----|------|----------|------|
| **支付渠道层** | Extension CAS更新 | `PayOrderServiceImpl.updateOrderSuccess()` Extension版 | 防止重复更新支付拓展单 |
| **支付订单层** | Order CAS更新 | `PayOrderServiceImpl.updateOrderSuccess()` Order版 | 防止重复更新主订单 |
| **通知层** | Redis分布式锁 + notifyTimes校验 | `PayNotifyServiceImpl.executeNotify()` | 防止并发重复通知 |
| **通知层** | 通知任务状态机 | `PayNotifyServiceImpl.processNotifyResult()` | SUCCESS后不再通知 |
| **业务层** | 业务订单CAS更新 | `PayDemoOrderServiceImpl.updateDemoOrderPaid()` | 防止重复更新业务订单 |
| **业务层** | 支付单号匹配校验 | `PayDemoOrderServiceImpl.updateDemoOrderPaid()` | 防止错单 |

## 10. 最佳实践建议

### 10.1 业务系统接入支付时必须做到

1. **使用CAS更新订单状态**：`UPDATE xxx SET status='PAID' WHERE id=? AND status='UNPAID'`

2. **先更新状态，后执行业务**：CAS成功后再扣库存/加积分

3. **幂等校验**：
   ```java
   if (已支付 && 支付单号相同) {
       return; // 幂等返回
   }
   if (已支付 && 支付单号不同) {
       throw; // 异常
   }
   ```

4. **使用事务**：订单状态更新 + 业务处理在同一事务

### 10.2 监控与告警

- 关注 `pay_notify_task` 表中 `status=FAILURE` 的记录
- 超过最大重试次数的通知应触发人工告警
- 定期对账：支付订单SUCCESS vs 业务订单PAID，不一致需人工介入

## 11. 总结

ruoyi-vue-pro 支付模块设计了完善的状态机和防重机制：

- **状态单向流转**：通过CAS更新保证WAITING→SUCCESS→REFUND的单向性
- **多层幂等**：从支付渠道到业务订单共5层幂等校验
- **最终一致性**：通知任务+重试机制保证业务最终能收到通知
- **错误隔离**：支付订单状态更新与业务通知分离，支付成功即不可逆转

项目中的 `PayDemoOrderServiceImpl` 是一个很好的业务接入参考实现，实际开发中应严格遵循其模式。
