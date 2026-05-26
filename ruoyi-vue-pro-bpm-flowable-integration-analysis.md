# RuoYi-Vue-Pro BPM/Flowable 集成分析

## 1. 核心概念关系图

```
业务表单 (Form)
    ↓ 提交
业务表记录 (Business Record) ←→ 流程变量 (Process Variables)
    ↑ 双向绑定 ↓
BusinessKey + ProcessInstanceId
    ↓
流程实例 (Process Instance)
    ↓ 包含
任务 (Task)
    ↓ 分配
审批人/候选组 (Assignee/Candidate)
    ↓ 审批
流程状态变更 → 业务状态同步
```

## 2. 流程定义部署

### 2.1 部署流程

1. **模型创建 (Model)**：在 `BpmModelServiceImpl` 中创建流程模型，设计 BPMN 流程定义
2. **部署验证**：通过 `BpmTaskCandidateInvoker.validateBpmnConfig()` 校验所有 UserTask 的审批人配置
3. **流程定义生成**：部署后生成 `ProcessDefinition`，包含流程定义 ID 和 Key

### 2.2 关键代码位置

- 模型管理：`yudao-module-bpm/src/main/java/cn/iocoder/yudao/module/bpm/service/definition/BpmModelServiceImpl.java`
- 流程定义服务：`yudao-module-bpm/src/main/java/cn/iocoder/yudao/module/bpm/service/definition/BpmProcessDefinitionServiceImpl.java`

## 3. 发起流程

### 3.1 完整流程

```
1. 业务表记录创建
   └─→ 插入业务数据，设置初始状态为 RUNNING

2. 构建流程变量
   └─→ 将业务数据映射为流程变量 (如 day = 请假天数)

3. 发起流程实例
   └─→ 调用 processInstanceApi.createProcessInstance()
   └─→ 设置 processDefinitionKey (如 "oa_leave")
   └─→ 设置 businessKey = 业务记录 ID
   └─→ 返回 processInstanceId

4. 双向绑定
   └─→ 更新业务表的 processInstanceId 字段
```

### 3.2 业务表与 ProcessInstanceId 绑定机制

以 OA 请假为例 (`BpmOALeaveServiceImpl.java:47-64`)：

```java
@Transactional(rollbackFor = Exception.class)
public Long createLeave(Long userId, BpmOALeaveCreateReqVO createReqVO) {
    // 1. 插入 OA 请假单
    long day = LocalDateTimeUtil.between(createReqVO.getStartTime(), createReqVO.getEndTime()).toDays();
    BpmOALeaveDO leave = BeanUtils.toBean(createReqVO, BpmOALeaveDO.class)
            .setUserId(userId).setDay(day).setStatus(BpmTaskStatusEnum.RUNNING.getStatus());
    leaveMapper.insert(leave);

    // 2. 发起 BPM 流程
    Map<String, Object> processInstanceVariables = new HashMap<>();
    processInstanceVariables.put("day", day);
    String processInstanceId = processInstanceApi.createProcessInstance(userId,
            new BpmProcessInstanceCreateReqDTO().setProcessDefinitionKey(PROCESS_KEY)
                    .setVariables(processInstanceVariables).setBusinessKey(String.valueOf(leave.getId()))
                    .setStartUserSelectAssignees(createReqVO.getStartUserSelectAssignees()));

    // 3. 将工作流的编号，更新到 OA 请假单中
    leaveMapper.updateById(new BpmOALeaveDO().setId(leave.getId()).setProcessInstanceId(processInstanceId));
    return leave.getId();
}
```

### 3.3 绑定要点

| 方向 | 绑定方式 | 说明 |
|------|---------|------|
| 流程 → 业务 | `businessKey = 业务记录ID` | 流程实例通过 businessKey 关联业务 |
| 业务 → 流程 | `processInstanceId` 字段 | 业务表记录保存流程实例 ID |
| 数据传递 | 流程变量 `variables` | 业务数据以变量形式传入流程引擎 |

## 4. 审批人/候选组解析

### 4.1 解析机制

核心类：`BpmTaskCandidateInvoker.java`

```
1. 从 BpmnModel 中解析 UserTask 的配置
   ├─→ parseCandidateStrategy() - 获取分配策略
   └─→ parseCandidateParam() - 获取策略参数

2. 根据策略调用对应的 BpmTaskCandidateStrategy
   ├─→ USER (指定用户)
   ├─→ ROLE (角色)
   ├─→ DEPT_MEMBER (部门成员)
   ├─→ DEPT_LEADER (部门领导)
   ├─→ START_USER (发起人)
   ├─→ EXPRESSION (表达式)
   └─→ ... 其他策略

3. 候选人过滤
   ├─→ 移除被禁用用户
   ├─→ 处理"审批人为空"的情况
   └─→ 根据配置决定是否跳过发起人
```

### 4.2 关键策略

| 策略 | 枚举值 | 说明 |
|------|--------|------|
| 指定用户 | `USER` | 直接指定用户ID列表 |
| 角色 | `ROLE` | 根据角色编码获取用户 |
| 部门成员 | `DEPT_MEMBER` | 获取指定部门的所有成员 |
| 部门领导 | `DEPT_LEADER` | 获取指定部门的负责人 |
| 发起人 | `START_USER` | 流程发起人本人 |
| 表单选择 | `FORM_USER` | 从表单字段中选择审批人 |
| 表达式 | `EXPRESSION` | 执行 Spring EL 表达式动态计算 |

### 4.3 代码示例 (`BpmTaskCandidateInvoker.java:91-126`)

```java
@DataPermission(enable = false)
public Set<Long> calculateUsersByTask(DelegateExecution execution) {
    return FlowableUtils.execute(execution.getTenantId(), () -> {
        // 1. 获取流程元素
        FlowElement flowElement = execution.getCurrentFlowElement();
        // 2. 解析策略和参数
        Integer strategy = BpmnModelUtils.parseCandidateStrategy(flowElement);
        String param = BpmnModelUtils.parseCandidateParam(flowElement);
        // 3. 调用策略计算候选人
        Set<Long> userIds = getCandidateStrategy(strategy).calculateUsersByTask(execution, param);
        // 4. 过滤处理
        removeDisableUsers(userIds);
        // 5. 处理空候选人情况
        if (CollUtil.isEmpty(userIds)) {
            userIds = getCandidateStrategy(BpmTaskCandidateStrategyEnum.ASSIGN_EMPTY.getStrategy())
                    .calculateUsersByTask(execution, param);
        }
        return userIds;
    });
}
```

## 5. 任务查询

### 5.1 任务查询依赖 Flowable 引擎

**关键问题：为什么不能只依赖业务表状态查询任务？**

| 维度 | 业务表 | Flowable 引擎表 |
|------|--------|----------------|
| 数据时效性 | 可能存在延迟同步 | 实时反映流程状态 |
| 任务分配 | 不存储具体审批人 | 精确存储 assignee/candidate |
| 任务状态 | 仅记录整体流程状态 | 记录每个任务的详细状态 |
| 多实例任务 | 难以表达 | 原生支持 |
| 加签/减签/转办 | 不支持 | 完整支持 |
| 待办/已办区分 | 无法区分 | 可通过 Task/HistoricTask 区分 |

### 5.2 待办任务查询 (`BpmTaskServiceImpl.java:114-140`)

```java
public PageResult<Task> getTaskTodoPage(Long userId, BpmTaskPageReqVO pageVO) {
    TaskQuery taskQuery = taskService.createTaskQuery()
            .taskAssignee(String.valueOf(userId))  // 分配给自己
            .active()                              // 活跃任务
            .includeProcessVariables()
            .taskTenantId(FlowableUtils.getTenantId())
            .orderByTaskCreateTime().desc();
    // ... 过滤条件
    long count = taskQuery.count();
    List<Task> tasks = taskQuery.listPage(PageUtils.getStart(pageVO), pageVO.getPageSize());
    return new PageResult<>(tasks, count);
}
```

### 5.3 任务状态说明

定义在 `BpmTaskStatusEnum.java`：

| 状态 | 值 | 说明 |
|------|-----|------|
| SKIP | -2 | 跳过 |
| NOT_START | -1 | 未开始 |
| WAIT | 0 | 待审批 |
| RUNNING | 1 | 审批中 |
| APPROVE | 2 | 审批通过 |
| REJECT | 3 | 审批不通过 |
| CANCEL | 4 | 已取消 |
| RETURN | 5 | 已退回 |
| APPROVING | 7 | 审批通过中（加签场景） |

## 6. 流程变量与业务表单

### 6.1 流程变量的作用

```
流程变量 (Process Variables)
├── 业务数据：表单字段值 (如 day, amount)
├── 控制数据：审批结果、审批意见
├── 系统变量：发起人、流程状态
└── 表达式参数：用于条件网关判断
```

### 6.2 业务表单与流程变量的关系

1. **发起时**：表单数据 → 流程变量
2. **审批时**：审批结果 → 流程变量
3. **流转时**：流程变量 → 条件判断 (Expression)
4. **结束时**：流程状态 → 业务状态

### 6.3 关键流程变量

```java
// 定义在 BpmnVariableConstants
PROCESS_INSTANCE_VARIABLE_START_USER_ID  // 发起人ID
PROCESS_INSTANCE_VARIABLE_STATUS         // 流程实例状态
PROCESS_INSTANCE_VARIABLE_REASON         // 审批原因/意见
```

## 7. 流程状态与业务状态同步

### 7.1 流程实例状态枚举 (`BpmProcessInstanceStatusEnum.java`)

| 状态 | 值 | 说明 |
|------|-----|------|
| NOT_START | -1 | 未开始 |
| RUNNING | 1 | 审批中 |
| APPROVE | 2 | 审批通过 |
| REJECT | 3 | 审批不通过 |
| CANCEL | 4 | 已取消 |

### 7.2 状态同步机制

```
Flowable 事件
    ↓
BpmProcessInstanceEventListener 监听
    ↓
processProcessInstanceCreated/completed()
    ↓
BpmProcessInstanceEventPublisher 发布事件
    ↓
业务监听器 (如 BpmOALeaveStatusListener) 处理
    ↓
更新业务表状态
```

### 7.3 事件监听链

**1. Flowable 原生事件监听** (`BpmProcessInstanceEventListener.java`)

```java
public static final Set<FlowableEngineEventType> PROCESS_INSTANCE_EVENTS = 
    ImmutableSet.<FlowableEngineEventType>builder()
        .add(FlowableEngineEventType.PROCESS_CREATED)
        .add(FlowableEngineEventType.PROCESS_COMPLETED)
        .add(FlowableEngineEventType.PROCESS_CANCELLED)
        .build();

@Override
protected void processCompleted(FlowableEngineEntityEvent event) {
    ProcessInstance processInstance = (ProcessInstance) event.getEntity();
    FlowableUtils.execute(processInstance.getTenantId(),
            () -> processInstanceService.processProcessInstanceCompleted(processInstance));
}
```

**2. 流程完成处理** (`BpmProcessInstanceServiceImpl.java:967-1029`)

```java
public void processProcessInstanceCompleted(ProcessInstance instance) {
    // 1. 获取当前状态
    Integer status = (Integer) instance.getProcessVariables()
            .get(BpmnVariableConstants.PROCESS_INSTANCE_VARIABLE_STATUS);
    
    // 2. 如果还是 RUNNING，说明是正常结束（审批通过）
    if (Objects.equals(status, BpmProcessInstanceStatusEnum.RUNNING.getStatus())) {
        status = BpmProcessInstanceStatusEnum.APPROVE.getStatus();
        runtimeService.setVariable(instance.getId(), 
            BpmnVariableConstants.PROCESS_INSTANCE_VARIABLE_STATUS, status);
    }
    
    // 3. 发送流程实例状态事件
    processInstanceEventPublisher.sendProcessInstanceResultEvent(
            BpmProcessInstanceConvert.INSTANCE.buildProcessInstanceStatusEvent(
                this, instance, status, reason));
}
```

**3. 业务监听器处理** (`BpmOALeaveStatusListener.java`)

```java
@Component
public class BpmOALeaveStatusListener extends BpmProcessInstanceStatusEventListener {
    
    @Override
    protected String getProcessDefinitionKey() {
        return BpmOALeaveServiceImpl.PROCESS_KEY;  // "oa_leave"
    }
    
    @Override
    protected void onEvent(BpmProcessInstanceStatusEvent event) {
        // businessKey 就是业务记录 ID
        leaveService.updateLeaveStatus(
            Long.parseLong(event.getBusinessKey()), 
            event.getStatus()
        );
    }
}
```

### 7.4 不同场景的状态同步

#### 场景 1：流程正常结束（审批通过）

```
最后一个任务审批通过
    ↓
流程引擎触发 PROCESS_COMPLETED 事件
    ↓
status = RUNNING → 自动改为 APPROVE
    ↓
发布 BpmProcessInstanceStatusEvent (status=2)
    ↓
业务监听器更新业务表 status = 2 (审批通过)
```

#### 场景 2：流程驳回

```
某节点审批驳回
    ↓
设置流程变量 status = REJECT (3)
    ↓
触发 PROCESS_COMPLETED 事件
    ↓
status 已是 REJECT，不自动修改
    ↓
发布事件 (status=3)
    ↓
业务监听器更新业务表 status = 3 (审批不通过)
```

#### 场景 3：流程撤回/取消

```
发起人撤回流程
    ↓
触发 PROCESS_CANCELLED 事件
    ↓
发布事件 (status=4, CANCEL)
    ↓
业务监听器更新业务表 status = 4 (已取消)
```

#### 场景 4：流程驳回（回退）

驳回分为两种情况：
- **终止驳回**：流程直接结束，状态变为 REJECT
- **回退驳回**：流程回退到指定节点，流程仍在 RUNNING 状态

## 8. "业务单显示已通过但流程实例仍在运行" 原因链分析

### 8.1 可能原因链

#### 原因链 1：事务问题 - 状态更新失败

```
1. 最后一个任务审批通过，本地事务中更新业务状态为 APPROVE
   └─→ 业务表 status = 2 (APPROVE)

2. 但 Flowable 流程提交在另一个事务或异步执行
   └─→ 流程实例尚未完成，仍在 ACT_RU_EXECUTION 表中

3. 结果：业务表显示已通过，但 Flowable 中流程仍运行中
   └─→ 事务隔离或异步导致的状态不一致
```

**代码佐证**：`BpmProcessInstanceServiceImpl.java:1042` 使用了事务同步

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCommit() {
        // 在事务提交后执行
        runtimeService.setProcessInstanceName(instance.getProcessInstanceId(), name);
    }
});
```

#### 原因链 2：事件监听失败 - 业务状态提前更新

```
1. 业务代码在审批通过时直接更新了业务状态
   └─→ 绕过了事件机制，直接 UPDATE 业务表

2. 但由于异常或事务回滚，Flowable 流程未真正完成
   └─→ 或流程监听器执行失败，流程状态未同步

3. 结果：业务表 status=APPROVE，但流程实例 status=RUNNING
```

#### 原因链 3：多实例任务 - 最终条件未触发

```
1. 多实例任务 (Parallel Multi-instance)
   └─→ 配置了 completionCondition (如 nrOfCompletedInstances >= 1)

2. 第一个实例审批通过
   └─→ 业务监听器错误地将整体状态设为 APPROVE

3. 但流程还在等待其他并行实例
   └─→ 或 completionCondition 实际上是 nrOfCompletedInstances == nrOfInstances

4. 结果：业务显示通过，但流程仍在运行
```

#### 原因链 4：子流程/调用活动 - 父子流程不同步

```
1. 子流程审批完成，触发子流程的 PROCESS_COMPLETED
   └─→ 业务表绑定的是子流程的 businessKey

2. 业务监听器错误响应子流程事件，更新状态为 APPROVE
   └─→ 但父流程仍在继续执行

3. 父流程的 processInstanceId 仍在运行中
   └─→ 但业务表显示已通过
```

**代码佐证**：`BpmProcessInstanceServiceImpl.java:981-1002` 有子流程处理逻辑

```java
// 如果子流程拒绝，设置其父流程也为拒绝状态
if (Objects.equals(status, BpmProcessInstanceStatusEnum.REJECT.getStatus())
        && StrUtil.isNotBlank(instance.getSuperExecutionId())) {
    Execution execution = runtimeService.createExecutionQuery()
        .executionId(instance.getSuperExecutionId()).singleResult();
    ProcessInstance parentProcessInstance = getProcessInstance(execution.getProcessInstanceId());
    updateProcessInstanceReject(parentProcessInstance, ...);
}
```

#### 原因链 5：监听器异常 - 状态不一致

```
1. 流程正常完成，进入 processProcessInstanceCompleted
   └─→ status = APPROVE

2. 发布事件后，业务监听器执行异常
   └─→ 或事件被过滤，业务表未更新

3. 反之：业务代码提前更新状态，然后流程异常回滚
   └─→ 业务表是 APPROVE，但流程被回滚仍在运行
```

#### 原因链 6：加签/会签场景

```
1. 后加签 (Add Sign After)
   └─→ 原任务审批通过后，状态变为 APPROVING (7)

2. 业务状态被错误地映射为 APPROVE (2)
   └─→ 但实际上还有加签任务需要审批

3. 流程实例仍在运行
   └─→ ACT_RU_TASK 中还有待办任务
```

**代码佐证**：`BpmTaskStatusEnum.java:30-34` 定义了 APPROVING 状态

```java
/**
 * 使用场景：
 * 1. 任务被向后【加签】时，它在审批通过后，会变成 APPROVING 这个状态，
 *    然后等到【加签】出来的任务都被审批后，才会变成 APPROVE 审批通过
 */
APPROVING(7, "审批通过中"),
```

### 8.2 诊断方法

| 检查项 | SQL 示例 |
|--------|---------|
| 流程实例状态 | `SELECT * FROM ACT_RU_EXECUTION WHERE PROC_INST_ID_ = ?` |
| 运行中的任务 | `SELECT * FROM ACT_RU_TASK WHERE PROC_INST_ID_ = ?` |
| 历史流程实例 | `SELECT * FROM ACT_HI_PROCINST WHERE PROC_INST_ID_ = ?` |
| 业务表状态 | `SELECT status, process_instance_id FROM bpm_oa_leave WHERE id = ?` |
| 流程变量 | `SELECT * FROM ACT_RU_VARIABLE WHERE PROC_INST_ID_ = ?` |

## 9. 关键数据表对应关系

| 业务层 | Flowable 运行时表 | Flowable 历史表 |
|--------|------------------|----------------|
| 流程实例 | ACT_RU_EXECUTION | ACT_HI_PROCINST |
| 任务 | ACT_RU_TASK | ACT_HI_TASKINST |
| 流程变量 | ACT_RU_VARIABLE | ACT_HI_VARINST |
| 活动执行 | ACT_RU_ACTINST | ACT_HI_ACTINST |
| 身份链接 | ACT_RU_IDENTITYLINK | ACT_HI_IDENTITYLINK |

## 10. 总结

### 10.1 核心原则

1. **双向绑定**：BusinessKey 和 ProcessInstanceId 是业务与流程的桥梁
2. **事件驱动**：通过 Spring Event 机制实现业务状态与流程状态的解耦
3. **策略模式**：审批人分配采用可扩展的策略模式
4. **事务同步**：使用 TransactionSynchronization 确保状态一致性

### 10.2 常见问题预防

1. **不要直接更新业务状态**：应通过事件机制同步
2. **不要依赖业务表查任务**：任务查询必须使用 Flowable API
3. **注意加签/会签状态**：APPROVING ≠ APPROVE
4. **处理事务边界**：Flowable 事件与业务事务的同步问题
5. **子流程特殊处理**：父子流程的状态联动需要额外处理
