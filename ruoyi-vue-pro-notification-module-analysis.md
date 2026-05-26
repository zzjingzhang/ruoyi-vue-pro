# ruoyi-vue-pro 站内信/通知模块深度分析

## 1. 模块架构概览

ruoyi-vue-pro 的站内信/通知模块主要包含两大消息体系：

| 消息类型 | 数据对象 | 核心服务 | 推送机制 |
|---------|---------|---------|---------|
| 站内信（Notify） | `NotifyMessageDO` | `NotifySendService`、`NotifyMessageService` | 数据库持久化为主 |
| 通知公告（Notice） | `NoticeDO` | `NoticeService` | 支持 WebSocket 推送在线用户 |
| 客服消息（KeFu） | `KeFuMessageDO` | `KeFuMessageService` | 持久化 + 异步 WebSocket 推送 |

---

## 2. 核心组件关系图

```
业务事件（如 BPM 审批）
        ↓
业务模块（BpmMessageService/BpmProcessInstanceServiceImpl
        ↓
NotifyMessageSendApiImpl（系统模块 API 层）
        ↓
NotifySendServiceImpl（模板校验 + 参数格式化）
        ↓
NotifyMessageServiceImpl（数据库持久化）
        ↓
NotifyMessageDO → system_notify_message 表
```

---

## 3. 模板配置与参数填充

### 3.1 模板数据结构

模板定义在 `NotifyTemplateDO` 中，关键字段：

```java
// yudao-module-system/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/notify/NotifyTemplateDO.java

private Long id;
private String name;           // 模板名称
private String code;           // 模板编码（唯一标识）
private Integer type;        // 模板类型
private String nickname;    // 发送人名称
private String content;       // 模板内容（含占位符）
private List<String> params;    // 解析出的参数名列表
private Integer status;      // 状态（启用/禁用）
```

### 3.2 模板参数解析

模板参数在创建/更新模板时自动解析参数：

```java
// NotifyTemplateServiceImpl.java:74-76
private static final Pattern PATTERN_PARAMS = Pattern.compile("\\{(.*?)}");

public List<String> parseTemplateContentParams(String content) {
    return ReUtil.findAllGroup1(PATTERN_PARAMS, content);
}
```

**示例：模板内容 `"您的审批{processInstanceName}已通过，点击查看：{detailUrl}"`

解析出的参数列表：`["processInstanceName", "detailUrl"]`

### 3.3 参数填充机制

```java
// NotifyTemplateServiceImpl.java:141-142
@Override
public String formatNotifyTemplateContent(String content, Map<String, Object> params) {
    return StrUtil.format(content, params);
}
```

使用 Hutool 的 `StrUtil.format()` 实现 `{key}` 占位符替换。

### 3.4 发送流程中的完整调用链

```
NotifySendServiceImpl.sendSingleNotify()
    ↓
1. validateNotifyTemplate() — 从缓存获取模板并校验状态
    ↓
2. validateTemplateParams() — 校验所有必需参数是否存在
    ↓
3. formatNotifyTemplateContent() — 填充参数生成最终内容
    ↓
4. createNotifyMessage() — 持久化到数据库
```

---

## 4. 消息持久化与 WebSocket 推送

### 4.1 当前实现的关键发现

**站内信（NotifyMessage）**：
- **当前实现**：只做**数据库持久化**
- **未实现**：没有触发 WebSocket 推送
- **核心代码**：`NotifyMessageServiceImpl.java:30-38

```java
@Override
public Long createNotifyMessage(Long userId, Integer userType,
                                NotifyTemplateDO template, String templateContent,
                                Map<String, Object> templateParams) {
    NotifyMessageDO message = new NotifyMessageDO()
        .setUserId(userId).setUserType(userType)
        .setTemplateId(template.getId())
        .setTemplateCode(template.getCode())
        .setTemplateContent(templateContent)
        .setTemplateParams(templateParams)
        .setReadStatus(false);
    notifyMessageMapper.insert(message);
    return message.getId();
}
```

**通知公告（Notice）**：
- **单独的推送入口**：`NoticeController.push()` 手动触发推送

```java
// NoticeController.java:98
webSocketSenderApi.sendObject(UserTypeEnum.ADMIN.getValue(), "notice-push", notice);
```

**客服消息（KeFuMessage）**：
- **模式**：先持久化 → 再异步推送

```java
// KeFuMessageServiceImpl.java:57-77
@Transactional(rollbackFor = Exception.class)
public Long sendKefuMessage(KeFuMessageSendReqVO sendReqVO) {
    // 2.1 保存消息
    keFuMessageMapper.insert(kefuMessage);
    // 2.2 更新会话消息冗余
    conversationService.updateConversationLastMessage(kefuMessage);

    // 3.1 发送消息给会员（@Async 异步）
    getSelf().sendAsyncMessageToMember(..., KEFU_MESSAGE_TYPE, message);
    // 3.2 通知所有管理员更新对话
    getSelf().sendAsyncMessageToAdmin(KEFU_MESSAGE_TYPE, message);
    return kefuMessage.getId();
}
```

### 4.2 WebSocket 推送架构

```
业务代码调用 webSocketSenderApi.sendObject()
        ↓
WebSocketSenderApiImpl
        ↓
RedisWebSocketMessageSender（如果用 Redis 实现
        ↓
redisMQTemplate.send(RedisWebSocketMessage)
        ↓
Redis Pub/Sub 广播
        ↓
RedisWebSocketMessageConsumer（各实例消费
        ↓
AbstractWebSocketMessageSender.send()
        ↓
WebSocketSessionManager 获取会话列表
        ↓
遍历所有匹配的 WebSocketSession.sendMessage()
```

---

## 5. 已读/未读状态管理

### 5.1 数据模型

```java
// NotifyMessageDO.java:92-99
private Boolean readStatus;   // 是否已读
private LocalDateTime readTime; // 阅读时间
```

### 5.2 未读数获取

未读数**没有缓存层**，每次直接查询数据库：

```java
// NotifyMessageServiceImpl.java:61-63
@Override
public Long getUnreadNotifyMessageCount(Long userId, Integer userType) {
    return notifyMessageMapper.selectUnreadCountByUserIdAndUserType(userId, userType);
}
```

前端通过**轮询** `GET /admin-api/system/notify-message/get-unread-count` 接口获取未读数。

### 5.3 标记已读

```java
// 单条/批量标记
int updateNotifyMessageRead(Collection<Long> ids, Long userId, Integer userType);

// 全部标记已读
int updateAllNotifyMessageRead(Long userId, Integer userType);
```

---

## 6. 多端登录推送处理

### 6.1 会话管理结构

```java
// WebSocketSessionManagerImpl.java:29-38
private final ConcurrentMap<String, WebSocketSession> idSessions
    = new ConcurrentHashMap<>();  // sessionId → WebSocketSession

private final ConcurrentMap<Integer, ConcurrentMap<Long, CopyOnWriteArrayList<WebSocketSession>>> userSessions
    = new ConcurrentHashMap<>();
// userType → (userId → List<WebSocketSession>)
```

### 6.2 多端登录支持：

当同一用户在多个设备登录时，每个设备建立独立的 WebSocketSession，全部保存在同一个 `userSessions[userType][userId]` 的 List 中。

### 6.3 推送时的会话匹配

```java
// AbstractWebSocketMessageSender.java:53-74
public void send(String sessionId, Integer userType, Long userId, ...) {
    List<WebSocketSession> sessions = Collections.emptyList();
    if (StrUtil.isNotEmpty(sessionId)) {
        // 推送到指定 Session
    } else if (userType != null && userId != null) {
        // 推送到该用户的所有 Session（多端登录全覆盖
        sessions = sessionManager.getSessionList(userType, userId);
    } else if (userType != null) {
        // 推送到该类型所有用户
    }
    doSend(sessions, messageType, messageContent);
}
```

---

## 7. BPM 审批与消息发送

### 7.1 审批流程中的消息发送位置

```java
// BpmProcessInstanceServiceImpl.java:1004-1015

// 2. 发送对应的消息通知
if (Objects.equals(status, BpmProcessInstanceStatusEnum.APPROVE.getStatus())) {
    messageService.sendMessageWhenProcessInstanceApprove(...);
} else if (...) {
    messageService.sendMessageWhenProcessInstanceReject(...);
}

// 3. 发送流程实例的状态事件
processInstanceEventPublisher.sendProcessInstanceResultEvent(...);
```

### 7.2 关键问题：消息服务类型

**BpmMessageServiceImpl 目前只发送**短信**，**不发送站内信：

```java
// BpmMessageServiceImpl.java:36-41
public void sendMessageWhenProcessInstanceApprove(...) {
    Map<String, Object> templateParams = new HashMap<>();
    templateParams.put("processInstanceName", reqDTO.getProcessInstanceName());
    templateParams.put("detailUrl", getProcessInstanceDetailUrl(reqDTO.getProcessInstanceId()));
    // 只调用短信 API
    smsSendApi.sendSingleSmsToAdmin(...);
}
```

**注意**：如果需要站内信通知，需要额外调用 `NotifyMessageSendApi`。

---

## 8. 事务回滚风险分析

### 8.1 风险场景一：站内信消息发送在事务内

当前站内信发送**没有** `@Transactional` 注解保护：

```java
// NotifySendServiceImpl.java - 没有 @Transactional
public Long sendSingleNotify(...) {
    // 1. 校验模板
    NotifyTemplateDO template = validateNotifyTemplate(templateCode);
    // 2. 校验参数
    validateTemplateParams(template, templateParams);
    // 3. 格式化内容
    String content = notifyTemplateService.formatNotifyTemplateContent(...);
    // 4. 持久化（在调用方事务内提交）
    return notifyMessageService.createNotifyMessage(...);
}
```

### 8.2 风险场景二：BPM 审批事务边界

```java
// BpmTaskServiceImpl.java:552
@Transactional(rollbackFor = Exception.class)
public void approveTask(...) {
    // ... 审批逻辑 ...
    taskService.complete(task.getId(), variables, true);
    // ← 此时可能触发消息发送
    // ← 如果后续逻辑抛异常，事务回滚
}
```

### 8.3 如果消息推送与持久化的时序问题

**对于客服消息模式**（参考实现）：

```
T1: 事务开始
    ↓
T2: keFuMessageMapper.insert() — 消息写入数据库（事务内）
    ↓
T3: @Async sendAsyncMessageToMember() — 异步线程启动（已脱离事务上下文
    ↓
T4: 事务内其他业务操作
    ↓
T5: 事务回滚（T2 的插入回滚）
    ↓
T6: 异步线程推送 WebSocket — 消息推送到前端
    ↓
结果：用户收到了推送，但数据库中消息不存在
```

---

## 9. 故障推演：审批被回滚但用户收到了审批通过通知

### 9.1 场景背景

假设某业务场景：采购订单审批通过后需要执行以下步骤：

1. 审批任务完成
2. 发送"审批通过"站内信通知
3. 更新订单状态为"已审批"
4. 扣减库存

### 9.2 故障触发条件

**假设一**：业务代码中在审批通过后调用了**自定义扩展逻辑：

```java
@Service
public class OrderApprovalService {

    @Transactional(rollbackFor = Exception.class)
    public void approveOrder(String processInstanceId, Long userId) {
        // 1. 完成审批任务
        bpmTaskService.approveTask(userId, approveReqVO);

        // 2. 发送站内信通知（假设这里假设集成了 Notify API）
        notifyMessageSendApi.sendSingleMessageToAdmin(
            new NotifySendSingleToUserReqDTO()
                .setUserId(startUserId)
                .setTemplateCode("order_approved")
                .setTemplateParams(Map.of(
                    "processInstanceName", "采购订单-2024001",
                    "detailUrl", "/order/detail?id=1001"
                ))
        );
        // ↑ 消息已持久化到数据库（事务内）

        // 3. 更新订单状态
        orderService.updateStatus(orderId, OrderStatus.APPROVED);

        // 4. 扣减库存
        inventoryService.deductStock(productId, quantity);
        // ← 这里抛异常！库存不足！
    }
}
```

### 9.3 故障时序（如果实现了 WebSocket 推送）

**假设二**：项目已扩展 `NotifySendServiceImpl 中添加了 WebSocket 推送：

```java
// 假设的扩展实现
@Service
public class NotifySendServiceImpl {

    @Resource
    private WebSocketSenderApi webSocketSenderApi;

    public Long sendSingleNotify(...) {
        // ... 原有逻辑 ...
        
        // 新增：持久化后推送
        Long messageId = notifyMessageService.createNotifyMessage(...);
        
        // 异步推送
        webSocketSenderApi.sendObject(
            userType, userId,
            "notify-message",
            message
        );
        
        return messageId;
    }
}
```

### 9.4 故障详细时序

```
时间轴
 │
 T0: 事务开始
 │
 T1: approveTask() 完成 → BPM 任务
 │
 T2: sendSingleMessageToAdmin() 调用
 │     ↓
 T2.1: 校验模板
 T2.2: 校验参数
 T2.3: formatNotifyTemplateContent()
 T2.4: createNotifyMessage() → INSERT INTO system_notify_message
 │        (事务内，未提交）
 │
 T2.5: webSocketSenderApi.sendObject() 调用
 │     ↓
 T2.5.1: RedisMQTemplate.send(RedisWebSocketMessage)
 │        → Redis Pub/Sub 广播
 │
 T3: RedisWebSocketMessageConsumer 消费消息
 │     ↓
 T3.1: 遍历 WebSocketSession
 T3.2: session.sendMessage()
 │        → 前端收到"采购订单-2024001已通过"通知
 │
 T4: 用户张三的浏览器收到通知并显示
 │
 T5: orderService.updateStatus() 执行成功
 │
 T6: inventoryService.deductStock() 执行
 │     ↓
 T6.1: 检查库存 → 库存不足！
 T6.2: 抛出 InsufficientStockException
 │
 T7: 事务回滚！
 │     ↓
 T7.1: T2.4 的 INSERT 被回滚
 T7.2: 数据库中没有这条消息
 │
 T8: 结果
 │     ↓
 用户张三看到通知，点击查看
 │     ↓
 详情页查询消息 → 404 或空
 未读数轮询 → 不包含这条消息
 │
```

### 9.5 故障表现

| 1. **用户看到**：顶部通知栏显示"采购订单-2024001已通过
 点击查看详情 → 消息不存在或 404
 未读数与实际看到的不一致

2. **数据库状态**：
   - `system_notify_message` 表中**没有**这条审批通过的消息
   - 订单状态**还是**待审批
   - 库存**未扣减**

3. **用户困惑**：
   - 明明收到了通过通知
   - 但订单还是待审批状态
   - 消息详情打不开

### 9.6 根本原因分析

**根本原因**：WebSocket 推送**异步且脱离事务上下文**

```
问题 1：Redis Pub/Sub 是**即时**广播，不参与事务
问题 2：WebSocket 推送发生在**事务提交前**
问题 3：推送与持久化**缺乏最终一致性保证**
问题 4：没有**事务提交后再推送**的机制
```

### 9.7 对比客服消息实现的隐患

```java
// KeFuMessageServiceImpl.java:57-77
@Transactional(rollbackFor = Exception.class)
public Long sendKefuMessage(...) {
    keFuMessageMapper.insert(kefuMessage);  // T1: 事务内插入
    
    // T2: @Async 异步线程
    getSelf().sendAsyncMessageToMember(...);
    // 异步线程可能在事务提交前就执行了！
    
    // T3: 如果这里抛异常
    someOtherService.doSomething();  // 抛异常！
}
```

**客服消息同样存在相同问题**：`@Async` 异步推送可能在事务提交前执行。

---

## 10. 修复方案建议

### 10.1 方案一：事务提交后推送（推荐）

使用 `TransactionSynchronizationManager`：

```java
public Long sendSingleNotify(...) {
    // 1. 持久化
    Long messageId = notifyMessageService.createNotifyMessage(...);
    
    // 2. 注册事务同步，提交后再推送
    if (TransactionSynchronizationManager.isActualTransactionActive())) {
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    // 事务提交后才推送
                    webSocketSenderApi.sendObject(...);
                }
            }
        );
    } else {
        // 无事务时直接推送
        webSocketSenderApi.sendObject(...);
    }
    
    return messageId;
}
```

### 10.2 方案二：使用 @TransactionalEventListener

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void sendAfterCommit(NotifyMessageDO message) {
    webSocketSenderApi.sendObject(...);
}
```

### 10.3 方案三：消息队列解耦

持久化 → 发消息到 MQ → 消费后再推送

---

## 11. 总结

| 问题点 | 当前状态 | 风险等级 |
|-------|---------|---------|
| 模板参数填充 | 使用 `{param}` + `StrUtil.format()` | 低 |
| 消息持久化 | 数据库 INSERT | 低 |
| 站内信 WebSocket 推送 | **未实现** | - |
| 客服消息推送 | 异步推送，事务内调用 | **高** |
| 事务回滚风险 | 推送脱离事务 | **高** |
| 多端登录 | 支持，userSessions 映射 | 低 |
| 未读数缓存 | 无缓存，直接查库 | 中（性能） |
| BPM 审批通知 | 只发短信，未发站内信 | - |

**核心风险点**：如果项目扩展了站内信的 WebSocket 推送，必须使用**事务提交后再推送**的机制，否则会出现"幽灵消息"问题。