# RuoYi-Vue-Pro Maven 多模块依赖边界分析

## 1. 整体架构分层

RuoYi-Vue-Pro 采用了清晰的 Maven 多模块架构设计，从功能上可以划分为以下几类模块：

```
┌──────────────────────────────────────────────────────────────┐
│                     yudao-server (容器)                      │
│                   聚合所有业务模块，统一启动                     │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  业务模块层   │    │  通用业务层   │    │  基础设施层   │
│ trade/pay/...│    │  system      │    │  infra       │
└──────────────┘    └──────────────┘    └──────────────┘
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  framework 层   │
                    │ starter 组件    │
                    └─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  dependencies   │
                    │  统一版本管理    │
                    └─────────────────┘
```

## 2. 模块职责与依赖边界

### 2.1 各模块职责说明

| 模块 | 职责 | 依赖方向 |
|------|------|----------|
| **yudao-dependencies** | 统一管理所有第三方依赖的版本号（BOM） | 无业务依赖 |
| **yudao-framework** | 技术组件层，提供各种 Spring Boot Starter | 依赖 yudao-dependencies |
| **yudao-module-infra** | 基础设施运维与研发工具 | 依赖 framework |
| **yudao-module-system** | 通用业务能力（用户、权限、字典等） | 依赖 infra + framework |
| **yudao-module-member** | 会员中心业务 | 依赖 system + infra + framework |
| **yudao-module-pay** | 支付能力（支付单、退款单、钱包等） | 依赖 system + framework |
| **yudao-module-trade** | 交易/订单业务 | 依赖 pay + member + product + system |
| **yudao-module-bpm** | 工作流引擎（Flowable） | 依赖 system + framework |
| **yudao-server** | 主启动容器 | 依赖需要启用的业务模块 |

### 2.2 依赖方向原则

1. **单向依赖**：上层业务模块依赖下层基础模块，不允许反向依赖
2. **接口隔离**：跨模块调用通过 `api` 包接口，而非直接依赖实现类
3. **同层不互依**：同级业务模块之间应尽量避免直接依赖（trade 是特殊的上层聚合模块）

## 3. 为什么跨模块依赖 API 包而非 Implementation

### 3.1 设计模式：Facade / API 层

从代码结构中可以看到以下模式：

```java
// 在 yudao-module-pay 中定义的 API 接口
// 文件：yudao-module-pay/src/main/java/cn/iocoder/yudao/module/pay/api/order/PayOrderApi.java
public interface PayOrderApi {
    Long createOrder(@Valid PayOrderCreateReqDTO reqDTO);
    PayOrderRespDTO getOrder(Long id);
    void updatePayOrderPrice(Long id, Integer payPrice);
}

// API 实现类
// 文件：yudao-module-pay/src/main/java/cn/iocoder/yudao/module/pay/api/order/PayOrderApiImpl.java
@Service
public class PayOrderApiImpl implements PayOrderApi {
    
    @Resource
    private PayOrderService payOrderService;  // 内部 Service
    
    @Override
    public Long createOrder(PayOrderCreateReqDTO reqDTO) {
        return payOrderService.createOrder(reqDTO);  // 委托给内部 Service
    }
}

// 内部的业务 Service 接口（不应被外部模块直接依赖）
// 文件：yudao-module-pay/src/main/java/cn/iocoder/yudao/module/pay/service/order/PayOrderService.java
public interface PayOrderService {
    PayOrderDO getOrder(Long id);  // 返回 DO，包含数据库细节
    List<PayOrderDO> getOrderList(Collection<Long> ids);
    PageResult<PayOrderDO> getOrderPage(PayOrderPageReqVO pageReqVO);
    // ... 更多内部方法
}
```

### 3.2 选择依赖 API 的原因

| 对比维度 | 依赖 API 接口 | 直接依赖 Service 实现类 |
|---------|-------------|----------------------|
| **耦合度** | 低，依赖抽象 | 高，依赖具体实现 |
| **编译可见性** | 仅暴露必要接口 | 暴露所有内部方法和数据结构 |
| **数据安全** | DTO 经过转换，隐藏 DO 细节 | DO 直接暴露，包含敏感字段 |
| **变更影响** | API 稳定时对调用方无影响 | Service 重构会影响所有调用方 |
| **测试隔离** | 可 mock API 进行单元测试 | 需要启动整个模块才能测试 |
| **循环依赖风险** | 低，API 层可独立 | 高，容易形成互相依赖 |

### 3.3 trade-api 模块的设计

对于 `trade` 模块，项目采用了更清晰的拆分：

```
yudao-module-mall/
├── yudao-module-trade-api/       ← 仅包含 API 接口和 DTO
│   └── src/main/java/cn/iocoder/yudao/module/trade/
│       ├── api/                  ← API 接口
│       └── enums/                ← 公共枚举
└── yudao-module-trade/           ← 实现模块
    └── src/main/java/cn/iocoder/yudao/module/trade/
        ├── api/                  ← API 实现类
        ├── controller/           ← Controller
        ├── service/              ← 内部 Service
        ├── dal/                  ← 数据访问层
        └── convert/              ← 类型转换
```

`trade-api` 的 pom.xml 依赖非常轻量：

```xml
<!-- yudao-module-trade-api/pom.xml -->
<dependencies>
    <dependency>
        <groupId>cn.iocoder.boot</groupId>
        <artifactId>yudao-common</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

这种设计允许其他模块只依赖 `trade-api`，而不需要依赖整个 `trade` 模块。

## 4. Starter 和 Framework 模块提供的能力

### 4.1 yudao-framework 模块分类

`yudao-framework` 模块是技术组件层，包含两类组件：

```
yudao-framework/
├── 框架组件（Framework Components）
│   ├── yudao-spring-boot-starter-mybatis      # MyBatis Plus 增强
│   ├── yudao-spring-boot-starter-redis        # Redis 缓存封装
│   ├── yudao-spring-boot-starter-web          # Web 层统一配置
│   ├── yudao-spring-boot-starter-security     # 安全认证
│   ├── yudao-spring-boot-starter-websocket    # WebSocket 支持
│   └── yudao-common                            # 公共工具类
│
└── 业务组件（Biz Components，名称包含 biz）
    ├── yudao-spring-boot-starter-biz-tenant          # 多租户
    ├── yudao-spring-boot-starter-biz-data-permission # 数据权限
    └── yudao-spring-boot-starter-biz-ip              # IP 解析
```

### 4.2 各类 Starter 的能力

#### 框架组件

| Starter | 核心能力 |
|---------|----------|
| **yudao-spring-boot-starter-mybatis** | 自动分页、字段自动填充（创建/更新时间、创建人）、多表关联查询封装、逻辑删除 |
| **yudao-spring-boot-starter-redis** | Redis 序列化配置、分布式锁、缓存管理、Key 前缀管理 |
| **yudao-spring-boot-starter-web** | 全局异常处理、统一响应包装、参数校验、CORS 配置 |
| **yudao-spring-boot-starter-security** | JWT 认证、权限注解、登录拦截、方法级安全 |
| **yudao-spring-boot-starter-excel** | Excel 导入导出（基于 EasyExcel 封装） |
| **yudao-spring-boot-starter-job** | 定时任务（XXL-Job 集成） |
| **yudao-spring-boot-starter-mq** | 消息队列封装 |
| **yudao-spring-boot-starter-test** | 单元测试基类、测试配置 |

#### 业务组件

| Starter | 核心能力 |
|---------|----------|
| **yudao-spring-boot-starter-biz-tenant** | 租户隔离、自动过滤租户条件、租户上下文管理 |
| **yudao-spring-boot-starter-biz-data-permission** | 基于部门/用户的数据权限过滤 |
| **yudao-spring-boot-starter-biz-ip** | IP 地址解析、IP 区域划分 |

### 4.3 业务模块如何使用 Starter

以 `yudao-module-system` 的依赖为例：

```xml
<!-- yudao-module-system/pom.xml -->
<dependencies>
    <!-- 业务组件：数据权限、多租户、IP -->
    <dependency>
        <groupId>cn.iocoder.boot</groupId>
        <artifactId>yudao-spring-boot-starter-biz-data-permission</artifactId>
    </dependency>
    <dependency>
        <groupId>cn.iocoder.boot</groupId>
        <artifactId>yudao-spring-boot-starter-biz-tenant</artifactId>
    </dependency>
    
    <!-- 框架组件：安全、MyBatis、Redis、Job、MQ -->
    <dependency>
        <groupId>cn.iocoder.boot</groupId>
        <artifactId>yudao-spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>cn.iocoder.boot</groupId>
        <artifactId>yudao-spring-boot-starter-mybatis</artifactId>
    </dependency>
    <!-- ... -->
</dependencies>
```

**设计理念**：
- 业务模块按需引入 Starter，避免依赖冗余
- Starter 遵循 Spring Boot 自动配置约定，减少手动配置
- 业务模块聚焦业务逻辑，技术细节由 Starter 封装

## 5. 循环依赖的暴露机制

### 5.1 Maven 级别的循环依赖

Maven 在构建阶段会检测模块间的循环依赖。例如：

```
Module A --> Module B
Module B --> Module A
```

**暴露方式**：
- 执行 `mvn compile` 时 Maven 直接报错
- 错误信息：`The projects in the reactor contain a cyclic reference`

### 5.2 Spring 容器中的循环依赖

即使 Maven 层面没有循环依赖，Spring 容器中也可能出现 Bean 循环依赖：

```java
// Module A
@Service
public class ServiceA {
    @Resource
    private ServiceB serviceB;
}

// Module B
@Service
public class ServiceB {
    @Resource
    private ServiceA serviceA;
}
```

**暴露方式**：
- 应用启动时抛出 `BeanCurrentlyInCreationException`
- 错误信息包含循环链路：`Requested bean is currently in creation: Is there an unresolvable circular reference?`

### 5.3 预防循环依赖的设计原则

1. **模块分层清晰**：严格遵循上层依赖下层的原则
2. **通过 API 解耦**：如果 A 和 B 需要互相调用，考虑提取公共的 API 层
3. **事件驱动**：使用消息队列或 Spring Event 替代直接调用
4. **中介者模式**：引入中间模块 C，A 和 B 都依赖 C

## 6. 新增业务模块：VO、DO、Api、Service 的放置位置

### 6.1 标准包结构

新增一个业务模块（例如 `yudao-module-demo`）时，建议的包结构：

```
yudao-module-demo/
├── pom.xml
└── src/main/java/cn/iocoder/yudao/module/demo/
    ├── api/                          ← 对外暴露的 API 接口 + 实现
    │   ├── DemoApi.java              ← API 接口（其他模块依赖这个）
    │   ├── DemoApiImpl.java          ← API 实现类
    │   └── dto/                      ← API 传输对象（DTO）
    │       ├── DemoCreateReqDTO.java
    │       └── DemoRespDTO.java
    │
    ├── controller/                   ← Controller 层
    │   ├── admin/                    ← 管理后台接口
    │   │   ├── DemoController.java
    │   │   └── vo/                   ← VO（View Object）
    │   │       ├── DemoSaveReqVO.java
    │   │       ├── DemoPageReqVO.java
    │   │       └── DemoRespVO.java
    │   └── app/                      ← 用户端接口（如有）
    │       ├── AppDemoController.java
    │       └── vo/
    │
    ├── service/                      ← 业务逻辑层
    │   ├── DemoService.java          ← Service 接口
    │   └── DemoServiceImpl.java      ← Service 实现
    │
    ├── dal/                          ← 数据访问层
    │   ├── dataobject/               ← DO（Data Object）
    │   │   └── DemoDO.java
    │   └── mysql/
    │       └── DemoMapper.java
    │
    ├── convert/                      ← 对象转换（MapStruct）
    │   └── DemoConvert.java
    │
    ├── enums/                        ← 模块内枚举
    │   ├── DemoStatusEnum.java
    │   └── ErrorCodeConstants.java
    │
    └── framework/                    ← 框架扩展（如有）
        └── config/
            └── DemoConfiguration.java
```

### 6.2 各层对象的职责边界

| 对象类型 | 放置位置 | 职责 | 是否跨模块可见 |
|---------|---------|------|---------------|
| **DO (Data Object)** | `dal/dataobject/` | 与数据库表一一对应 | **不可见**，仅模块内使用 |
| **DTO (Data Transfer Object)** | `api/dto/` | API 层的数据传输 | **可见**，其他模块使用 |
| **VO (View Object)** | `controller/.../vo/` | 前后端交互的数据 | **不可见**，仅 Controller 层 |
| **BO (Business Object)** | `service/.../bo/` | 业务层内部数据结构 | **不可见** |
| **API 接口** | `api/` | 对外暴露的服务契约 | **可见** |
| **Service 接口** | `service/` | 模块内部业务接口 | **不可见** |

### 6.3 对象转换示例

```java
// DO - 数据库实体
// dal/dataobject/DemoDO.java
@Data
@TableName("demo")
public class DemoDO {
    private Long id;
    private String name;
    private Integer status;
    private String secretField;  // 敏感字段，不应暴露给外部
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}

// DTO - API 传输对象
// api/dto/DemoRespDTO.java
@Data
public class DemoRespDTO {
    private Long id;
    private String name;
    private Integer status;
    // 不包含 secretField，且时间格式可自定义
}

// VO - 前后端交互对象
// controller/admin/vo/DemoRespVO.java
@Data
public class DemoRespVO {
    private Long id;
    private String name;
    private String statusName;  // 状态码转中文，前端展示用
    private String createTime;  // 格式化后的字符串
}

// Convert - 使用 MapStruct 进行转换
// convert/DemoConvert.java
@Mapper
public interface DemoConvert {
    DemoConvert INSTANCE = Mappers.getMapper(DemoConvert.class);
    
    // DO -> DTO（给其他模块）
    DemoRespDTO convert2(DemoDO bean);
    
    // DO -> VO（给前端）
    DemoRespVO convert(DemoDO bean);
}
```

## 7. 错误设计：Trade 直接依赖 Pay 的 Service 实现类

### 7.1 问题场景

假设在 `yudao-module-trade` 中，开发者没有使用 `PayOrderApi`，而是直接依赖了 `PayOrderServiceImpl`：

```java
// 错误的写法：Trade 模块直接依赖 Pay 模块的实现类
// yudao-module-trade/src/main/java/cn/iocoder/yudao/module/trade/service/order/TradeOrderServiceImpl.java

import cn.iocoder.yudao.module.pay.service.order.PayOrderServiceImpl;  // 直接导入实现类
import cn.iocoder.yudao.module.pay.dal.dataobject.order.PayOrderDO;    // 直接依赖 DO

@Service
public class TradeOrderServiceImpl implements TradeOrderService {
    
    @Resource
    private PayOrderServiceImpl payOrderService;  // 注入实现类，而非接口
    
    @Override
    public void createTradeOrder(TradeOrderCreateReqVO reqVO) {
        // ... 创建交易订单
        
        // 直接调用 Pay 模块的内部方法
        PayOrderDO payOrder = payOrderService.getOrder(payOrderId);
        if (payOrder.getSecretField() != null) {  // 访问了敏感字段
            // 业务逻辑
        }
    }
}
```

### 7.2 构建阶段的问题

#### 问题 1：编译耦合过深

```
Trade 模块编译时需要：
- 编译 Pay 模块的全部代码（包括所有内部类）
- 扫描 Pay 模块的所有依赖
```

**影响**：
- 编译变慢，每次修改 Pay 模块的任意代码都可能触发 Trade 重新编译
- Maven 依赖树变得复杂，排除依赖困难

#### 问题 2：循环依赖风险

```
如果未来 Pay 模块也需要调用 Trade 的功能：
Pay --> Trade (需要 Trade 的订单状态)
Trade --> Pay (需要支付能力)

形成循环依赖 → Maven 构建失败
```

### 7.3 测试阶段的问题

#### 问题 1：单元测试难以隔离

```java
// 正确写法（依赖 API）：可以 mock
@Test
public void testCreateOrder() {
    PayOrderApi mockApi = mock(PayOrderApi.class);
    when(mockApi.getOrder(any())).thenReturn(new PayOrderRespDTO());
    // 注入 mock 的 API 进行测试
}

// 错误写法（依赖实现类）：难以 mock
@Test
public void testCreateOrder() {
    // 需要启动 Pay 模块的整个 Spring 上下文
    // 需要配置数据库、Redis 等基础设施
    // 测试变慢且不稳定
}
```

#### 问题 2：测试数据污染

```
Trade 的测试直接操作 Pay 的数据库表
→ 两个模块的测试数据互相影响
→ 测试失败时难以定位问题根源
```

### 7.4 运行时的问题

#### 问题 1：内部实现变更导致故障

假设 Pay 模块重构：

```java
// Pay 模块重构前
public class PayOrderServiceImpl implements PayOrderService {
    public PayOrderDO getOrder(Long id) { ... }
}

// Pay 模块重构后（方法签名不变，但内部行为变化）
public class PayOrderServiceImpl implements PayOrderService {
    public PayOrderDO getOrder(Long id) {
        // 现在返回的 DO 中某些字段为 null 了
        // 但 API 层的 DTO 仍然保持兼容
    }
}
```

**影响**：
- 依赖 API 的模块：不受影响，因为 `PayOrderRespDTO` 由 API 层保证兼容
- 直接依赖实现类的 Trade 模块：可能出现 `NullPointerException`

#### 问题 2：敏感数据暴露

```java
// PayOrderDO 包含的字段：
- id, orderNo, amount          ← 业务字段
- createTime, updateTime       ← 审计字段
- channelSecret, callbackUrl   ← 支付渠道敏感配置
- rawResponse                  ← 支付渠道原始响应（可能含敏感信息）

// 如果 Trade 直接使用 DO：
PayOrderDO order = payOrderService.getOrder(id);
log.info("Order details: {}", order);  // 日志泄露敏感信息
```

#### 问题 3：难以做灰度发布

```
场景：需要灰度升级 Pay 模块的支付逻辑

正确设计（依赖 API）：
- 可以同时运行两个版本的 PayOrderApiImpl
- 通过配置决定使用哪个实现
- Trade 模块无感知

错误设计（依赖实现类）：
- 必须同时升级 Trade 和 Pay 模块
- 无法独立灰度
- 回滚时需要同时回滚两个模块
```

#### 问题 4：难以替换实现

```
场景：需要从自研支付切换到第三方支付网关

正确设计（依赖 API）：
- 编写新的 ThirdPartyPayOrderApiImpl 实现 PayOrderApi
- 配置 Spring 使用新实现
- Trade 模块完全无感知

错误设计（依赖实现类）：
- 需要修改 Trade 中所有引用 PayOrderServiceImpl 的代码
- 修改 import 语句
- 修改注入的 Bean 名称/类型
- 大量代码改动，容易引入 Bug
```

### 7.5 正确的依赖方式

```java
// 正确的写法：依赖 API 接口
// yudao-module-trade/src/main/java/cn/iocoder/yudao/module/trade/service/order/TradeOrderServiceImpl.java

import cn.iocoder.yudao.module.pay.api.order.PayOrderApi;        // 依赖接口
import cn.iocoder.yudao.module.pay.api.order.dto.PayOrderRespDTO; // 依赖 DTO

@Service
public class TradeOrderServiceImpl implements TradeOrderService {
    
    @Resource
    private PayOrderApi payOrderApi;  // 注入接口，Spring 自动找到实现类
    
    @Override
    public void createTradeOrder(TradeOrderCreateReqVO reqVO) {
        // ... 创建交易订单
        
        // 通过 API 调用，只获取需要的数据
        PayOrderRespDTO payOrder = payOrderApi.getOrder(payOrderId);
        // PayOrderRespDTO 中不包含敏感字段
        // API 层保证向后兼容
    }
}
```

## 8. 总结：依赖设计原则

| 原则 | 说明 |
|------|------|
| **依赖抽象而非具体** | 依赖 API 接口，不依赖 Service 实现类 |
| **依赖稳定而非易变** | 依赖稳定的 API 层，不依赖经常重构的内部实现 |
| **最小可见性原则** | 只暴露必要的接口和 DTO，隐藏内部实现细节 |
| **单向依赖原则** | 上层依赖下层，基础模块不依赖业务模块 |
| **接口隔离原则** | 为不同调用方提供专门的 API，而非大而全的 Service |
| **分层架构原则** | Controller → Service → DAL，跨层必须通过 API |

遵循这些原则，可以构建出**低耦合、高内聚、易测试、易维护**的多模块 Maven 项目。
